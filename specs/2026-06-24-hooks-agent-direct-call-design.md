# Design: /hooks/agent direct-call integration — first real OpenClaw workers

**Issue:** casehubio/life#38
**Date:** 2026-06-24
**Status:** Approved

---

## Problem

Workers using `WorkerFunction.AgentExec(Agent)` call OpenClaw's `/v1/chat/completions` endpoint, which is synchronous but has no tool access — agents hallucinate results rather than executing real skills (calendar, banking, Home Assistant). OpenClaw's `/hooks/agent` endpoint routes to real skills but delivers results asynchronously via webhook. These are incompatible at the API level.

31 of 32 workers across 7 YamlCaseHubs are stubs returning hardcoded maps. The one real worker (`book-appointment-agent`) uses `/v1/chat/completions` without tools.

## Architecture: two coexisting execution modes

After life#38 (direct-call) and life#37 (heartbeat) both land, life supports two modes:

**Direct-call (life#38):** Worker defined in CaseDefinition → `Agent.execute()` → `DirectCallChatModel.chat()` → `OpenClawHookClient.invoke()` → `POST /hooks/agent` (fire-and-forget) → block virtual thread on `CompletableFuture` → webhook fires → `DirectCallDeliveryResource` completes future → `ChatResponse` → `WorkerResult`.

**Heartbeat (life#37):** No matching worker → `tryProvision()` → `OpenClawWorkerProvisioner.provision()` → agent registered → Qhorus COMMAND on case channel → `OpenClawChannelBackend.post()` → `POST /hooks/agent` → webhook fires → `OpenClawDeliveryResource` → Qhorus message → context change → next binding fires.

Workers defined in the case definition use direct-call. Capabilities with no matching worker fall through to the provisioner. Both use `OpenClawHookClient` and `/hooks/agent`. They differ in result delivery: direct-call completes a pending future; heartbeat posts to a Qhorus channel.

## Phase 1: Bridge (casehub-openclaw scope)

**Blocks life#38.** Filed as casehub-openclaw issue.

Three new components in `casehub-openclaw-casehub`:

### DirectCallBridge (`@ApplicationScoped`)

`ConcurrentHashMap<String, CompletableFuture<String>>` keyed by correlation ID.

- `submit(String correlationId)` — creates future, registers, returns it
- `complete(String correlationId, String responseText)` — completes future, removes from map
- `cancel(String correlationId)` — cancels if present, removes (cleanup on timeout/error)

No scheduled cleanup — the caller's virtual thread timeout (`DefaultWorkerExecutor.executeSync`) handles expiry; cancelled futures are removed by the calling ChatModel in a `finally` block.

### DirectCallDeliveryResource (JAX-RS)

`POST /openclaw/direct-call/{correlationId}` — receives webhook payload from OpenClaw.

- Extracts agent response text from `OpenClawDeliveryPayload`
- Calls `bridge.complete(correlationId, responseText)`
- Returns 200 always (OpenClaw must not retry)
- `@PermitAll` — webhook callbacks carry no OIDC token

### DirectCallChatModel (implements langchain4j ChatModel)

Constructor: `(DirectCallBridge, OpenClawHookClient, String openClawAgentId, String callbackBaseUrl, int timeoutSeconds)`

`chat(ChatRequest)`:
1. Extract user message text from ChatRequest
2. Generate correlationId (UUID)
3. `future = bridge.submit(correlationId)`
4. `deliveryUrl = callbackBaseUrl + "/openclaw/direct-call/" + correlationId`
5. `hookClient.invoke(openClawAgentId, userText, null, timeoutSeconds, deliveryUrl)`
6. `responseText = future.get(timeoutSeconds, SECONDS)` — blocks virtual thread
7. `finally: bridge.cancel(correlationId)` — idempotent cleanup
8. Return `ChatResponse` wrapping response text as `AiMessage`

The CaseHub `Agent` class sends a `ChatRequest` containing both a `SystemMessage` and a `UserMessage`. `DirectCallChatModel` extracts only the user message text for the `/hooks/agent` `message` field — the system message is not forwarded because OpenClaw agents have their own system prompts configured server-side. The CaseHub system prompt still shapes the user message content: `Agent.execute()` applies the `userMessageTemplate` with input variables before calling `chat()`, so the rendered user text carries the instructions the system prompt would have provided.

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
1. System prompt — persona + task instructions + output format
2. User message template — `{{variable}}` placeholders from case context
3. Response schema — Java record defining structured output
4. AgentDescriptor — `{model-family}:{persona}@{major}` identity
5. Factory call — `factory.forAgent("<openClawAgentId>")`

### Worker inventory

**AppointmentCycleCaseHub** (5 workers, health-agent):
- `book-appointment-agent` — existing AgentExec, convert to factory
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
- `DirectCallBridgeTest` — submit/complete/cancel lifecycle, concurrency, timeout
- `DirectCallDeliveryResourceTest` — receives payload, completes bridge, always 200
- `DirectCallChatModelTest` — mock bridge + hookClient; deliveryUrl construction, future await, ChatResponse mapping

**Life tests:**
- `LifeOpenClawChatModelFactoryTest` — `forAgent()` returns correct ChatModelProvider
- 32 response schema records — Jackson serialization roundtrip tests
- Existing integration tests unchanged — `TestLifeOpenClawChatModelFactory` replaces test provider; serves canned responses for all 32 workers keyed by user message text
- No integration tests for the bridge itself — bridge unit tests in casehub-openclaw cover correctness; life integration tests use the test factory

## Deferred issues

- **casehubio/engine: Make `AgentBuilder.model(ChatModel)` public** — currently package-private, forcing the `ChatModelProvider` workaround. Non-blocking; the factory pattern works.
- **life#37: WorkerProvisioner heartbeat mode** — Phase 2 of the two-mode architecture. Uses the existing `OpenClawWorkerProvisioner` in casehub-openclaw-casehub.
