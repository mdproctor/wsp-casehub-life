# Design: /hooks/agent direct-call integration — first real OpenClaw workers

**Issue:** casehubio/life#38
**Date:** 2026-06-24 (rev 2: 2026-06-25)
**Status:** Approved

---

## Problem

Workers using `WorkerFunction.AgentExec(Agent)` call OpenClaw's `/v1/chat/completions` endpoint, which is synchronous but has no tool access — agents hallucinate results rather than executing real skills (calendar, banking, Home Assistant). OpenClaw's `/hooks/agent` endpoint routes to real skills but delivers results asynchronously via webhook. These are incompatible at the API level.

31 of 32 workers across 7 YamlCaseHubs are stubs returning hardcoded maps. The one real worker (`book-appointment-agent`) uses `/v1/chat/completions` without tools.

## Architecture: two coexisting execution modes

After life#38 (direct-call) and life#37 (heartbeat) both land, life supports two modes:

**Direct-call (life#38):** Worker defined in CaseDefinition → `Agent.execute()` → `DirectCallChatModel.chat()` → `OpenClawHookClient.invokeDirect()` → `POST /hooks/agent` (fire-and-forget) → block virtual thread on `CompletableFuture` → webhook fires → `DirectCallDeliveryResource` completes future → `ChatResponse` → `WorkerResult`.

**Heartbeat (life#37):** No matching worker → `tryProvision()` → `OpenClawWorkerProvisioner.provision()` → agent registered → Qhorus COMMAND on case channel → `OpenClawChannelBackend.post()` → `POST /hooks/agent` → webhook fires → `OpenClawDeliveryResource` → Qhorus message → context change → next binding fires.

Workers defined in the case definition use direct-call. Capabilities with no matching worker fall through to the provisioner. Both use `OpenClawHookClient` and `/hooks/agent`. They differ in result delivery: direct-call completes a pending future; heartbeat posts to a Qhorus channel.

## Phase 1: Bridge (casehub-openclaw scope)

**Blocks life#38.** Filed as casehubio/openclaw#49.

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

No scheduled cleanup — the caller's virtual thread timeout handles expiry; cancelled futures are removed by the calling ChatModel in a `finally` block.

### DirectCallDeliveryResource (JAX-RS)

`POST /openclaw/direct-call/{correlationId}` — receives webhook payload from OpenClaw.

- Extracts agent response text from `OpenClawDeliveryPayload`
- Calls `bridge.complete(correlationId, responseText)`
- Returns 200 always (OpenClaw must not retry)
- `@PermitAll` — webhook callbacks carry no OIDC token
- Production deployments must use HTTPS for the callback base URL and restrict network access to the OpenClaw Gateway (firewall rules, VPC, or service mesh)

### DirectCallChatModel (implements langchain4j ChatModel)

Constructor: `(DirectCallBridge, OpenClawHookClient, String openClawAgentId, String callbackBaseUrl, int timeoutSeconds)`

`chat(ChatRequest)`:
1. Extract system message text and user message text from ChatRequest
2. Extract `JsonSchema` from `ChatRequest.responseFormat()` and serialize as a human-readable schema block (field names, types, required fields)
3. Build the combined message: `systemPrompt + "\n\n" + schemaBlock + "\n\n" + userText`
4. Generate correlationId (UUID)
5. `future = bridge.submit(correlationId)`
6. `deliveryUrl = callbackBaseUrl + "/openclaw/direct-call/" + correlationId`
7. `int effectiveTimeout = Math.max(timeoutSeconds - 5, 5)` — shorter than executor timeout to ensure consistent error handling
8. `hookClient.invokeDirect(openClawAgentId, combinedMessage, null, timeoutSeconds, deliveryUrl)`
9. `responseText = future.get(effectiveTimeout, SECONDS)` — blocks virtual thread
10. Validate `responseText` is valid JSON (`MAPPER.readTree(responseText)`) — fail fast with `AgentException("OpenClaw agent returned invalid JSON")` rather than propagating malformed responses
11. `finally: bridge.cancel(correlationId)` — idempotent cleanup
12. Return `ChatResponse` wrapping response text as `AiMessage`

**System prompt forwarding:** The CaseHub `Agent` class sends a `ChatRequest` containing both a `SystemMessage` and a `UserMessage`. `DirectCallChatModel` combines them into the single `message` field for `/hooks/agent`. The system prompt carries persona instructions and task guidance. OpenClaw agents have their own system prompts configured server-side (persona-level: skills, capabilities); CaseHub's system prompt is task-level (what to do, how to format output). They are complementary, not conflicting.

**Response schema forwarding:** The `JsonSchema` from `ChatRequest.responseFormat()` is automatically extracted and serialized into the message. This makes `responseSchema(BookingResult.class)` meaningful regardless of backend — the worker author doesn't need to manually describe JSON format in the system prompt. Enforcement shifts from model-level (response_format parameter) to prompt-level (schema instructions in message), which is the correct adaptation for an agent endpoint that doesn't support response_format.

### Timeout design

Two independent timeouts apply:
1. `DirectCallChatModel.chat()` — `future.get(effectiveTimeout, SECONDS)` → `java.util.concurrent.TimeoutException`
2. `DefaultWorkerExecutor.executeSync()` — Mutiny `.ifNoItem().after(Duration)` → `io.smallrye.mutiny.TimeoutException`

These are different exception types with different handling. The ChatModel timeout must be shorter (by 5 seconds, minimum 5s floor) to ensure it fires first. `DirectCallChatModel` catches `java.util.concurrent.TimeoutException` and throws `AgentException("OpenClaw agent timed out after " + effectiveTimeout + "s")`, giving `Agent.execute()` a consistent error type. If the ChatModel timeout somehow doesn't fire, the executor timeout produces `WorkerResult.expired()` as a backstop.

### Failure modes

**(a) `hookClient.invokeDirect()` throws `OpenClawInvocationException`** — network error or non-2xx. Happens after `bridge.submit()` has registered the future. The `finally` block calls `bridge.cancel(correlationId)`, removing the orphaned future. The exception propagates through `Agent.execute()` as an unrecoverable worker failure.

**(b) Webhook delivers an error response** — OpenClaw agent failed internally. `OpenClawDeliveryPayload.output()` contains an error message instead of expected JSON. `bridge.complete()` completes the future with the error text. `DirectCallChatModel` validates JSON (step 10) and throws `AgentException` if parsing fails. This gives a clean error rather than downstream field-mismatch failures.

**(c) Webhook arrives after timeout** — ChatModel timeout fires, `future.get()` throws, `finally` cancels and removes the future from the map. The late webhook calls `bridge.complete(correlationId, responseText)` → `futures.get(correlationId)` returns null → silently ignored. No leak, no side effects. The cancelled `CompletableFuture` is already removed from the map.

## Phase 2: Life consumption

### New components (life `app/`)

**`LifeOpenClawChatModelFactory`** (`@ApplicationScoped`)
- Injects `DirectCallBridge`, `OpenClawHookClient`, config
- `ChatModelProvider forAgent(String openClawAgentId)` — returns a `ChatModelProvider` whose `get()` creates a `DirectCallChatModel` for that agent
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
- Add `casehub-openclaw-casehub` (DirectCallBridge, DirectCallDeliveryResource, DirectCallChatModel)
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
1. System prompt — persona + task instructions (JSON format instructions are no longer needed here — the auto-extracted schema handles format enforcement)
2. User message template — `{{variable}}` placeholders from case context
3. Response schema — Java record defining structured output (auto-serialized into the message by `DirectCallChatModel`)
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
- `DirectCallChatModelTest` — mock bridge + hookClient; deliveryUrl construction, system prompt + schema prepended to message, JSON validation on response, timeout throws AgentException; verify `invokeDirect()` called (not `invoke()`)

**Life tests:**
- `LifeOpenClawChatModelFactoryTest` — `forAgent()` returns correct ChatModelProvider
- 31 new response schema records + BookingResult (existing) — Jackson serialization roundtrip tests
- Existing integration tests unchanged — `TestLifeOpenClawChatModelFactory` replaces test provider; serves canned responses for all 32 workers keyed by user message text
- No integration tests for the bridge itself — bridge unit tests in casehub-openclaw cover correctness; life integration tests use the test factory

## Deferred issues

- **casehubio/engine#569: Convenience — make `AgentBuilder.model(ChatModel)` public** — currently package-private. `ChatModelProvider` is the designed public API; using it through a factory is the intended pattern, not a workaround. Making `model(ChatModel)` public would avoid needing a `ChatModelProvider` wrapper when the `ChatModel` instance is already in hand. Non-blocking; the factory pattern works correctly.
- **life#37: WorkerProvisioner heartbeat mode** — Phase 2 of the two-mode architecture. Uses the existing `OpenClawWorkerProvisioner` in casehub-openclaw-casehub.
