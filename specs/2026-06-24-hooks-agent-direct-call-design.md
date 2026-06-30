# Design: /hooks/agent direct-call integration — first real OpenClaw workers

**Issue:** casehubio/life#38
**Date:** 2026-06-24 (rev 5: 2026-06-25)
**Status:** Approved

---

## Problem

Workers using `WorkerFunction.AgentExec(Agent)` call OpenClaw's `/v1/chat/completions` endpoint, which is synchronous but has no tool access — agents hallucinate results rather than executing real skills (calendar, banking, Home Assistant). OpenClaw's `/hooks/agent` endpoint routes to real skills but delivers results asynchronously via webhook. These are incompatible at the API level.

31 of 32 workers across 7 YamlCaseHubs are stubs returning hardcoded maps. The one real worker (`book-appointment-agent`) uses `/v1/chat/completions` without tools.

## Architecture: two coexisting execution modes

After life#38 (direct-call) and life#37 (heartbeat) both land, life supports two modes:

**Direct-call (life#38):** Worker defined in CaseDefinition → `Agent.execute()` → `OpenClawChatModel.doChat()` → `OpenClawAgentProvider.invoke()` → `DirectCallBridge` → `OpenClawHookClient.invokeDirect()` → `POST /hooks/agent` (fire-and-forget) → webhook fires → `DirectCallDeliveryResource` completes future → `Multi<AgentEvent>` emits `TextDelta` → `ChatResponse` → `WorkerResult`.

**Heartbeat (life#37):** No matching worker → `tryProvision()` → `OpenClawWorkerProvisioner.provision()` → agent registered → Qhorus COMMAND on case channel → `OpenClawChannelBackend.post()` → `POST /hooks/agent` → webhook fires → `OpenClawDeliveryResource` → Qhorus message → context change → next binding fires.

Workers defined in the case definition use direct-call. Capabilities with no matching worker fall through to the provisioner. Both use `OpenClawHookClient` and `/hooks/agent`. They differ in result delivery: direct-call completes a pending future via `AgentProvider.invoke()`; heartbeat posts to a Qhorus channel.

## Abstraction layer: AgentProvider, not ChatModel

The platform already has `AgentProvider` (`casehub-platform-agent-api`) as the agent SPI, with `ClaudeAgentProvider` (`casehub-platform-agent-claude`) as the established implementation and `ClaudeAgentChatModel` / `AgentSessionChatModel` (`casehub-platform-agent-claude-langchain4j`) as thin langchain4j bridges.

OpenClaw follows the same pattern — two classes instead of one `DirectCallChatModel`:

```
Engine Agent → OpenClawChatModel (thin bridge, doChat()) → OpenClawAgentProvider (AgentProvider impl)
                                                            → DirectCallBridge → /hooks/agent → webhook
```

This is the platform-coherent architecture. `ClaudeAgentChatModel` demonstrates the pattern:
- Constructor takes `AgentProvider` + config
- `doChat(ChatRequest)` extracts system prompt and user text from messages, builds `AgentSessionConfig`, calls `agentProvider.invoke(config)`, collects `TextDelta` events into response text
- `validateNoJsonFormat()` — Claude rejects JSON response format

OpenClaw diverges on one point: `OpenClawChatModel` does NOT reject `ResponseFormat.JSON`. Instead, it extracts the `JsonSchema` from the response format and serializes it into the user prompt as schema instructions. This adapts structured output enforcement from model-level (`response_format` parameter) to prompt-level (schema in message) — the correct adaptation for an agent endpoint that doesn't support `response_format`.

**What this gives over the rev 2 design (direct ChatModel):**
- `correlationId` is a first-class `AgentSessionConfig` field, not an internal UUID
- `timeout` is a `Duration`, not a raw int
- System prompt / user prompt separation is structural — `AgentSessionConfig` fields, not string concatenation at the ChatModel level
- `Multi<AgentEvent>` is natively reactive — the async-to-reactive conversion belongs in the provider, not in the ChatModel
- CDI priority ladder works for single-backend deployments (e.g., Claude uses `ClaudeAgentProvider @ApplicationScoped` replacing `NoOpAgentProvider @DefaultBean`); OpenClaw's per-agent instances are created via factory, not CDI discovery
- Forward-compatible: when the engine migrates from ChatModel to AgentProvider, OpenClaw already works
- One timeout layer — `Multi.await().atMost()` in the ChatModel bridge; the provider's emitter `onTermination()` handles cleanup. No double-timeout race.

## Phase 1: Bridge (casehub-openclaw scope)

**Blocks life#38.** Filed as casehubio/openclaw#49.

### Phase 1 dependency

casehub-openclaw-casehub's pom.xml needs `casehub-platform-agent-api` added — `OpenClawAgentProvider` implements `AgentProvider`, and `OpenClawChatModel` uses `AgentSessionConfig` and `AgentEvent`, all from this module:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-api</artifactId>
</dependency>
```

### OpenClawHookClient.invokeDirect() — new sessionless overload

The existing `invoke()` overloads require a registered session (`sessions.get(agentId)` → `OpenClawSession` → `sessionKey`). This is correct for the heartbeat/provisioner path where agents have long-lived sessions with stable webhook URLs.

Direct-call mode is per-invocation — the delivery URL changes on every call (includes correlationId). Using a persistent session for this is a category mismatch. The `sessionName` field is `@JsonInclude(NON_NULL)` in `AgentInvocationRequest`, meaning OpenClaw accepts requests without it. Auth is handled by `BearerTokenRequestFilter` (gateway bearer token from `OpenClawClientConfig`), decoupled from the session registry.

New overload:
```java
public void invokeDirect(String agentId, String message, String model,
                          int timeoutSeconds, String deliveryUrl)
```
Calls `invokeInternal()` with `sessionKey=null`. No session lookup. The gateway bearer token provides authentication.

### DirectCallBridge (`@ApplicationScoped`)

`ConcurrentHashMap<String, CompletableFuture<String>>` keyed by correlation ID.

- `submit(String correlationId)` — creates future, registers, returns it
- `complete(String correlationId, String responseText)` — completes future, removes from map
- `cancel(String correlationId)` — cancels if present, removes (idempotent cleanup)

No scheduled cleanup — the emitter's `onTermination()` callback in `OpenClawAgentProvider.invoke()` handles cleanup when the `Multi` subscription is cancelled (by timeout or consumer cancellation).

### DirectCallDeliveryResource (JAX-RS)

`POST /openclaw/direct-call/{correlationId}` — receives webhook payload from OpenClaw.

- Extracts agent response text from `OpenClawDeliveryPayload`
- Calls `bridge.complete(correlationId, responseText)`
- Returns 200 always (OpenClaw must not retry)
- `@PermitAll` — webhook callbacks carry no OIDC token
- Production deployments must use HTTPS for the callback base URL and restrict network access to the OpenClaw Gateway (firewall rules, VPC, or service mesh)

### OpenClawAgentProvider (implements AgentProvider)

Follows `ClaudeAgentProvider` pattern — implements the platform's agent SPI.

```java
public Multi<AgentEvent> invoke(AgentSessionConfig config) {
    return Multi.createFrom().emitter(emitter -> {
        String correlationId = config.correlationId() != null
                ? config.correlationId() : UUID.randomUUID().toString();
        CompletableFuture<String> future = bridge.submit(correlationId);
        String deliveryUrl = deliveryBaseUrl + "/openclaw/direct-call/" + correlationId;

        String message = config.systemPrompt() + "\n\n" + config.userPrompt();

        emitter.onTermination(() -> bridge.cancel(correlationId));

        try {
            hookClient.invokeDirect(agentId, message, null,
                    (int) config.timeout().toSeconds(), deliveryUrl);
        } catch (OpenClawInvocationException e) {
            emitter.fail(e);
            return;
        }

        future.whenComplete((text, error) -> {
            if (error != null) {
                emitter.fail(error);
            } else {
                emitter.emit(new AgentEvent.TextDelta(text));
                emitter.complete();
            }
        });
    });
}

public AgentSession openSession(AgentSessionInit init) {
    throw new UnsupportedOperationException(
            "OpenClaw direct-call is single-shot — use invoke()");
}
```

Constructor: `(DirectCallBridge bridge, OpenClawHookClient hookClient, String agentId, String deliveryBaseUrl)`

Note: `OpenClawAgentProvider` is NOT a CDI `@ApplicationScoped` singleton like `ClaudeAgentProvider`. Each `OpenClawChatModel` creates its own provider instance configured for a specific OpenClaw agentId. The CDI priority ladder (`NoOpAgentProvider @DefaultBean` → `OpenClawAgentProvider`) applies when OpenClaw is the ONLY agent provider; for direct-call, the provider is wired per-worker through the factory.

### OpenClawChatModel (thin langchain4j bridge, follows ClaudeAgentChatModel pattern)

Implements `ChatModel`. Bridges from langchain4j to `AgentProvider`.

`doChat(ChatRequest)`:
1. Extract system prompt via `extractSystemPrompt(request.messages())` (same helper pattern as `ClaudeAgentChatModel`)
2. Extract user text via `extractLastUserText(request.messages())`
3. If `request.responseFormat()` contains a `JsonSchema`: extract and serialize as a human-readable schema block, prepend to user text
4. Build `AgentSessionConfig(systemPrompt, userPromptWithSchema, List.of(), timeout, correlationId)`
5. Call `agentProvider.invoke(config)` → `Multi<AgentEvent>`
6. Collect `TextDelta` events: `.filter(TextDelta.class::isInstance).map(TextDelta::text).collect().asList().await().atMost(timeout)` — blocks virtual thread via reactive timeout
7. Join collected text fragments
8. Validate response is valid JSON (`MAPPER.readTree(responseText)`) — throw `AgentException("OpenClaw agent returned invalid JSON")` if parsing fails
9. Return `ChatResponse` wrapping response text as `AiMessage`

Does NOT call `validateNoJsonFormat()` — unlike `ClaudeAgentChatModel`, OpenClaw handles JSON format via schema-in-prompt.

Constructor: `(OpenClawAgentProvider provider, Duration timeout)`

### Schema serialization format

The `JsonSchema` from `ChatRequest.responseFormat()` is serialized as a human-readable schema block. Example for `BookingResult`:

```
Respond with JSON matching schema "BookingResult":
{
  "appointmentId": string (required),
  "provider": string (required),
  "confirmed": boolean (required),
  "declined": boolean (required),
  "reason": string (required)
}
```

This block is prepended to the user text. The OpenClaw agent receives both the schema requirements and the task-specific content in one message.

### Timeout design

One timeout layer — in `OpenClawChatModel.doChat()` via `Multi.await().atMost(timeout)`. When the reactive timeout fires:
1. The `Multi` subscription is cancelled
2. The emitter's `onTermination()` callback fires → `bridge.cancel(correlationId)` — removes the pending future
3. `await()` throws `TimeoutException` → caught and wrapped as `AgentException`

The `DefaultWorkerExecutor.executeSync()` Mutiny timeout is a backstop only. The reactive timeout inside `doChat()` fires first because it's set to the worker-configured timeout, while the executor uses the same value. If the reactive timeout fires, the virtual thread unblocks with an `AgentException` before the executor timeout can act.

### Failure modes

**(a) `hookClient.invokeDirect()` throws `OpenClawInvocationException`** — network error or non-2xx. The emitter calls `emitter.fail(e)`. The `onTermination()` callback fires → `bridge.cancel(correlationId)`. The exception propagates through `Multi.await()` → caught and wrapped as `AgentException` in `doChat()`.

**(b) Webhook delivers an error response** — OpenClaw agent failed internally. `OpenClawDeliveryPayload.output()` contains an error message instead of expected JSON. `bridge.complete()` completes the future with the error text. The emitter emits a `TextDelta` with the error text. `doChat()` validates JSON (step 8) and throws `AgentException` if parsing fails.

**(c) Webhook arrives after timeout** — reactive timeout cancels the Multi subscription → `onTermination()` fires → `bridge.cancel(correlationId)` → future cancelled and removed from map. The late webhook calls `bridge.complete(correlationId, responseText)` → no future in map → silently ignored. No leak, no side effects.

## Phase 2: Life consumption

### New components (life `app/`)

**`LifeOpenClawChatModelFactory`** (`@ApplicationScoped`)
- Injects `DirectCallBridge`, `OpenClawHookClient`, config (`casehub.openclaw.delivery.base-url`, `casehub.openclaw.agent.default-timeout-seconds`)
- `ChatModelProvider forAgent(String openClawAgentId)`:
  ```java
  public ChatModelProvider forAgent(String openClawAgentId) {
      var provider = new OpenClawAgentProvider(
              bridge, hookClient, openClawAgentId, deliveryBaseUrl);
      var chatModel = new OpenClawChatModel(provider,
              Duration.ofSeconds(timeoutSeconds));
      return () -> chatModel;
  }
  ```
- Each worker calls `factory.forAgent("health-agent")` and passes to `Agent.builder().model(...)`

**`TestLifeOpenClawChatModelFactory`** (`@Alternative @Priority(10)`)
- Same `forAgent(String)` API
- Returns mock `ChatModel`s keyed by rendered user message text
- No bridge, no HTTP — pure in-memory

### Deleted

- `LifeOpenClawChatModelProvider` — replaced by factory
- `TestLifeOpenClawChatModelProvider` — replaced by test factory
- `OpenClawHealthProbe` — TCP probe for `/v1/chat/completions` no longer relevant

### Retained

- `BookingResult` — response schema for `book-appointment-agent`, unchanged

### New response schema records

31 new Java records (one per stub worker being converted). Each defines the structured output fields for that worker. `BookingResult` (existing) is the template.

### Dependencies

- Add `casehub-openclaw-core` (OpenClawHookClient, OpenClawGatewayClient, OpenClawClientConfig)
- Add `casehub-openclaw-casehub` (DirectCallBridge, DirectCallDeliveryResource, OpenClawAgentProvider, OpenClawChatModel)
- Add `casehub-platform-agent-api` (AgentProvider, AgentSessionConfig, AgentEvent — transitive via openclaw-casehub but listed for clarity)
- Remove `langchain4j-open-ai` runtime dep (no longer calling `/v1/chat/completions` directly)

### Configuration

Remove life-specific config:
- `casehub.life.openclaw.api-url`
- `casehub.life.openclaw.api-key`
- `casehub.life.openclaw.timeout-seconds`

Use standard casehub-openclaw config (`OpenClawClientConfig`):
- `casehub.openclaw.gateway.url`
- `casehub.openclaw.gateway.bearer-token`
- `casehub.openclaw.delivery.base-url`
- `casehub.openclaw.agent.default-timeout-seconds`

### Protocol update

PP-20260618-openclaw-agent must be updated when this spec is implemented. It currently references:
- `LifeOpenClawChatModelProvider` (being deleted)
- `OpenClawHealthProbe` (being deleted)
- `casehub.life.openclaw.api-url` config (being removed)
- "chatModelProvider.get() is called once in Agent.build()" behavior (changing to factory pattern)

## Worker conversions

### OpenClaw agent mapping

| OpenClaw agentId | CaseHub persona | Workers |
|---|---|---|
| `health-agent` | `openclaw:health-agent@1` | AppointmentCycle (5) + CareCoordination (3) + CareEpisode (2) = 10 |
| `home-agent` | `openclaw:home-agent@1` | HomeMaintenance (5) + ContractorCoordination (5) = 10 |
| `finance-agent` | `openclaw:finance-agent@1` | FinancialReview (5) = 5 |
| `travel-agent` | `openclaw:travel-agent@1` | TravelPlan (7) = 7 |

FamilyVoteCaseHub has no workers (pure humanTask). Total: 32 workers converted.

### Per-worker requirements

Each converted worker needs:
1. System prompt — persona + task instructions (JSON format instructions are no longer needed — the auto-extracted schema handles format enforcement)
2. User message template — `{{variable}}` placeholders from case context
3. Response schema — Java record defining structured output (auto-serialized into the message by `OpenClawChatModel`)
4. AgentDescriptor — `{model-family}:{persona}@{major}` identity
5. Factory call — `factory.forAgent("<openClawAgentId>")`

### Worker inventory

**AppointmentCycleCaseHub** (5 workers, health-agent):
- `book-appointment-agent` — existing AgentExec, convert to factory; BookingResult retained
- `find-alternative-agent` — find alternative provider after decline
- `confirm-appointment-agent` — send confirmation + reminder
- `pre-visit-prep-agent` — send checklist and prep instructions
- `record-health-decision-agent` — record to tamper-evident ledger

**CareCoordinationCaseHub** (3 workers, health-agent):
- `needs-assessment-agent` — assess care level and frequency
- `care-plan-agent` — build care schedule
- `health-check-agent` — periodic health review

**CareEpisodeCaseHub** (2 workers, health-agent):
- `assess-patient-agent` — vital signs, mobility, cognition
- `provide-care-agent` — execute care tasks, record observations

**HomeMaintenanceCaseHub** (5 workers, home-agent):
- `schedule-inspection-agent` — schedule and report inspection
- `get-quotes-agent` — gather contractor quotes
- `issue-commitment-agent` — issue commitment to contractor
- `monitor-job-agent` — track job progress
- `record-completion-agent` — record completion to ledger

**ContractorCoordinationCaseHub** (5 workers, home-agent):
- `request-quote-agent` — request quote via messaging
- `watchdog-escalation-agent` — escalate overdue commitment
- `quote-received-agent` — process received quote
- `job-monitoring-agent` — monitor active job
- `record-payment-agent` — record payment to ledger

**FinancialReviewCaseHub** (5 workers, finance-agent):
- `gather-data-agent` — aggregate multi-account transactions
- `analyse-anomalies-agent` — flag spending anomalies
- `escalate-anomalies-agent` — route anomalies to oversight
- `oversight-response-agent` — process human approval
- `produce-report-agent` — generate monthly report

**TravelPlanCaseHub** (7 workers, travel-agent):
- `destination-research-agent` — research destination options
- `flight-search-agent` — search flights
- `hotel-search-agent` — search hotels
- `budget-assessment-agent` — assess total cost vs threshold
- `booking-agent` — book flights + hotels
- `rebooking-agent` — rebook after decline
- `confirmation-agent` — confirm itinerary

## Testing

**Bridge tests (casehub-openclaw):**
- `DirectCallBridgeTest` — submit/complete/cancel lifecycle, concurrency, complete-after-cancel is no-op
- `DirectCallDeliveryResourceTest` — receives payload, completes bridge, always 200; unknown correlationId returns 200 (idempotent)
- `OpenClawAgentProviderTest` — mock bridge + hookClient; verifies `invokeDirect()` called with correct deliveryUrl; Multi emits TextDelta on future completion; emitter onTermination cancels bridge; failure paths (invocation exception, late webhook)
- `OpenClawChatModelTest` — system prompt + schema extraction from ChatRequest; schema serialized as human-readable block; JSON validation on response; timeout throws AgentException; ResponseFormat.JSON handled (not rejected)

**Life tests:**
- `LifeOpenClawChatModelFactoryTest` — `forAgent()` creates provider + chatModel with correct agentId
- 31 new response schema records + BookingResult (existing) — Jackson serialization roundtrip tests
- Existing integration tests unchanged — `TestLifeOpenClawChatModelFactory` replaces test provider; serves canned responses for all 32 workers keyed by user message text
- No integration tests for the bridge itself — bridge unit tests in casehub-openclaw cover correctness; life integration tests use the test factory

## Deferred issues

- **casehubio/engine#569: Convenience — make `AgentBuilder.model(ChatModel)` public** — currently package-private. `ChatModelProvider` is the designed public API; using it through a factory is the intended pattern. Making `model(ChatModel)` public would avoid needing a `ChatModelProvider` wrapper when the `ChatModel` instance is already in hand. Non-blocking; the factory pattern works correctly.
- **life#37: WorkerProvisioner heartbeat mode** — Phase 2 of the two-mode architecture. Uses the existing `OpenClawWorkerProvisioner` in casehub-openclaw-casehub.