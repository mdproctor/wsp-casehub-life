# Design: OpenClaw LLM Backend Integration — AgentExec Pattern Validation (life#25)

**Date:** 2026-06-17
**Issue:** casehubio/life#25
**Branch:** `issue-25-openclaw-worker-provisioner`
**Status:** Approved (rev 2)

---

## 1. What This Is and Is Not

### What it delivers

`WorkerFunction.AgentExec(Agent)` wired end-to-end in casehub-life: one stub worker replaced
with a real LLM call through OpenClaw's OpenAI-compatible API (`POST /v1/chat/completions`,
`model="openclaw"`). Proves the AgentExec execution path from `Agent.execute()` through
`DefaultWorkerExecutor` → `WorkflowExecutionCompletedHandler` → case context update → binding
fires. Establishes `Worker.agentDescriptor()` for the trust system.

### What it explicitly is NOT

**This does not start Layer 7.** ARC42STORIES §9.4 defines Layer 7 as:
> "casehub-openclaw as the WorkerProvisioner. OpenClaw instances execute household skills:
> banking API aggregation, Google Calendar integration, Home Assistant smart home control,
> WhatsApp/SMS follow-up."

Workers using `/v1/chat/completions` have no tool access, no skill routing, no ChannelContextWindow.
They will hallucinate appointment IDs and booking confirmations. That is not a real worker in any
meaningful sense — it is an LLM inference call with domain-shaped JSON output. Layer 7
(casehubio/life#8) begins when workers actually call appointment systems, calendars, and home
automation devices. The path to that is via `POST /hooks/agent` (real direct-call mode) or
`WorkerProvisioner` heartbeat mode — both requiring separate design work tracked in life#38
and life#37 respectively.

### Why AgentExec, not WorkerProvisioner or /hooks/agent

`WorkerFunction.AgentExec(Agent)` is the engine#463 settled abstraction for LLM agent workers.
`Agent.execute()` is synchronous — it calls `model.chat(ChatRequest)` and blocks on a virtual
thread. This is compatible with the inline worker execution path in `DefaultWorkerExecutor`.

`POST /hooks/agent` (real OpenClaw direct-call) delivers results asynchronously via
`deliver:webhook`. Bridging that to the synchronous `Agent.execute()` API requires a pending
future registry, a new delivery endpoint, and new casehub-openclaw infrastructure — filed as
life#38.

`WorkerProvisioner` heartbeat mode requires `ReactiveWorkerProvisioner` wiring and the reactive
qhorus stack — filed as life#37.

---

## 2. Architecture

```
AppointmentCycleCaseHub
  └── augment(yaml) → adds Worker("book-appointment-agent",
                                   AgentExec(bookingAgent),
                                   AgentDescriptor(agentId="life:openclaw:health-agent", ...))
                           stub lambdas for all other workers (unchanged)

bookingAgent = Agent.builder()
    .model(lifeOpenClawChatModelProvider)  ← LifeOpenClawChatModelProvider
    .systemPrompt(...)
    .userMessage(...)
    .responseSchema(BookingResult.class)
    .build()   ← chatModelProvider.get() called ONCE here; ChatModel stored in Agent for JVM lifetime

At runtime (first getDefinition() call):
  augment() runs (double-checked lock), bookingAgent is built, OpenAiChatModel is created

At case execution:
  binding fires → AgentRoutingStrategy selects "book-appointment-agent" (via AgentDescriptor)
  → WorkerScheduleEvent → DefaultWorkerExecutor.executeSync(agent::execute, ...) on virtual thread
  → LifeOpenClawChatModelProvider-backed OpenAiChatModel.chat(ChatRequest)
  → POST /v1/chat/completions → OpenClaw LLM → structured JSON response
  → Agent parses response via responseSchema → WorkerResult
  → WorkflowExecutionCompletedHandler applies output → .booking != null → next binding fires
```

---

## 3. New Components

### 3.1 `LifeOpenClawChatModelProvider`

**Package:** `io.casehub.life.app.engine.agent`
**Type:** `@ApplicationScoped` CDI bean
**Implements:** `io.casehub.api.model.ai.ChatModelProvider`

Temporary placement pending casehubio/engine#527 (add optional `baseUrl` to
`OpenAiChatModelProvider` in engine-api). Once that lands, this class is deleted and callers
use `OpenAiChatModelProvider.builder().baseUrl(...).modelName("openclaw").build()` directly.

Uses reflection to set `baseUrl` on `OpenAiChatModel.builder()`, mirroring the pattern in
`OpenAiChatModelProvider`. `get()` is called once during `Agent.build()` — config changes
(`api-url`, `timeout-seconds`) require a restart.

```java
@ApplicationScoped
public class LifeOpenClawChatModelProvider implements ChatModelProvider {

    // Pending casehubio/engine#527 — move baseUrl support to OpenAiChatModelProvider in engine-api.
    // Delete this class and replace callers with OpenAiChatModelProvider.builder().baseUrl(...) once landed.

    @ConfigProperty(name = "casehub.life.openclaw.api-url")
    String apiUrl;

    @ConfigProperty(name = "casehub.life.openclaw.api-key", defaultValue = "no-key")
    String apiKey;

    @ConfigProperty(name = "casehub.life.openclaw.timeout-seconds", defaultValue = "120")
    int timeoutSeconds;

    @Override
    public ModelType type() { return ModelType.OPENAI; }

    @Override
    public ChatModel get() {
        try {
            Class<?> clazz = Class.forName("dev.langchain4j.model.openai.OpenAiChatModel");
            Object builder = clazz.getMethod("builder").invoke(null);
            Class<?> bc = builder.getClass();
            invoke(bc, builder, "baseUrl",   String.class, apiUrl);
            invoke(bc, builder, "apiKey",    String.class, apiKey);
            invoke(bc, builder, "modelName", String.class, "openclaw");  // GE-20260614-328420
            invoke(bc, builder, "timeout",   Duration.class, Duration.ofSeconds(timeoutSeconds));
            return (ChatModel) bc.getMethod("build").invoke(builder);
        } catch (InvocationTargetException e) {
            throw new AgentException("Failed to build OpenClawChatModel: "
                + (e.getCause() != null ? e.getCause() : e).getMessage(), e);
        } catch (Exception e) {
            throw new AgentException("Failed to build OpenClawChatModel: " + e.getMessage(), e);
        }
    }

    private static void invoke(Class<?> bc, Object b, String method, Class<?> type, Object value)
            throws Exception {
        bc.getMethod(method, type).invoke(b, value);
    }
}
```

**Config (`application.properties`):**
```properties
# casehubio/life#25 — OpenClaw LLM backend (direct /v1/chat/completions, not /hooks/agent)
# Runtime dependency: OpenClaw must be accessible at this URL; failure deferred to first agent.execute()
casehub.life.openclaw.api-url=http://localhost:3000/v1
casehub.life.openclaw.api-key=no-key-required
casehub.life.openclaw.timeout-seconds=120
```

### 3.2 `BookingResult`

**Package:** `io.casehub.life.app.engine.agent`
**Type:** Java record (no framework deps)

```java
public record BookingResult(
    String appointmentId,
    String provider,
    boolean confirmed,
    Boolean declined,
    String reason
) {}
```

`AgentBuilder.responseSchema(BookingResult.class)` derives the JSON schema. Structured output
enforcement means OpenClaw must return conforming JSON.

### 3.3 `AppointmentCycleCaseHub` — modified

Inject `LifeOpenClawChatModelProvider`. Replace `bookAppointmentWorker()` with an
`AgentExec`-backed worker carrying an `AgentDescriptor`. All other workers remain stub lambdas.

`AgentDescriptor` is required: without it, `AgentCandidateFactory` builds a candidate with null
identity, `AgentRoutingStrategy` has nothing to score against trust dimensions, and the
attestation pipeline (Layer 6) cannot attribute outcomes from this worker.

```java
@ApplicationScoped
public class AppointmentCycleCaseHub extends YamlCaseHub {

    @Inject LifeOpenClawChatModelProvider openClaw;

    @Inject
    @ConfigProperty(name = "quarkus.uuid", defaultValue = "278776f9-e1b0-46fb-9032-8bddebdcf9ce")
    String tenancyId;

    // Note: AppointmentCycleCaseDefinitions (DSL companion) defines capabilities, bindings,
    // goals — pure structure, no workers. It does not need updating; workers are runtime
    // behaviour augmented here in CaseHub.augment(), not in the DSL companion.

    // getDefinition() double-checked lazy init unchanged.
    // augment() runs exactly once on first getDefinition() call.
    // bookingAgent is built once: chatModelProvider.get() is called in Agent.build(), not per invocation.

    private Worker bookAppointmentWorker() {
        Agent bookingAgent = Agent.builder()
            .model(openClaw)
            .systemPrompt("""
                You are a healthcare appointment booking agent for a UK household.
                Book medical appointments with the requested provider.
                If the provider is unavailable, set declined=true and provide a reason.
                Respond with valid JSON only — no prose, no explanation.
                """)
            .userMessage("Book a {{appointmentType}} appointment with provider {{provider}}.")
            .responseSchema(BookingResult.class)
            .build();

        AgentDescriptor descriptor = new AgentDescriptor(
            "life:openclaw:health-agent",         // agentId
            "OpenClaw Health Agent",              // name
            "1.0",                                // version
            "openclaw",                           // provider
            "openclaw",                           // modelFamily
            "openclaw",                           // modelVersion (same — no sub-version known)
            null,                                 // weightsFingerprint
            null,                                 // domainVocabulary
            null,                                 // slotVocabulary
            null,                                 // dispositionVocabulary
            null,                                 // axisVocabularies
            "casehubio/life/health",              // slot — matches scope path convention
            List.of(),                            // capabilities (populated when skill manifest available)
            null,                                 // disposition
            "GB",                                 // jurisdiction
            null,                                 // dataHandlingPolicy
            tenancyId,                            // tenancyId (required)
            "Booking agent for health domain"     // briefing
        );

        return Worker.builder()
            .name("book-appointment-agent")
            .capabilities(List.of(cap("book-appointment")))
            .function(bookingAgent)
            .agentDescriptor(descriptor)
            .build();
    }

    // findAlternativeWorker, confirmAppointmentWorker, preVisitPrepWorker,
    // recordHealthDecisionWorker — unchanged stub lambdas
}
```

### 3.4 Startup health probe

```java
@ApplicationScoped
public class OpenClawHealthProbe {

    @ConfigProperty(name = "casehub.life.openclaw.api-url")
    String apiUrl;

    void onStart(@Observes StartupEvent event) {
        try {
            HttpURLConnection conn = (HttpURLConnection)
                new URL(apiUrl.replace("/v1", "/health")).openConnection();
            conn.setConnectTimeout(3000);
            conn.setReadTimeout(3000);
            int code = conn.getResponseCode();
            if (code < 200 || code >= 300) {
                Log.warnf("OpenClaw health probe returned %d — agent workers will fail on first invocation", code);
            }
        } catch (Exception e) {
            Log.warnf("OpenClaw not reachable at %s — agent workers will fail on first invocation: %s",
                apiUrl, e.getMessage());
        }
    }
}
```

### 3.5 Maven dependency addition

In `app/pom.xml`, add `langchain4j-open-ai` at `runtime` scope. Base `langchain4j` API
(`ChatModel`) is already available transitively via `casehub-engine-api` (compile scope).
`OpenAiChatModel` is loaded via reflection at runtime.

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

## 4. CDI Wiring

No changes to `quarkus.arc.exclude-types` or `quarkus.arc.selected-alternatives`. We are adding
an inline `WorkerFunction.AgentExec` worker — not replacing a `WorkerProvisioner` no-op, not
adding casehub-openclaw-casehub as a dependency. The `LifeOpenClawChatModelProvider` is a
new `@ApplicationScoped` bean with no engine no-op conflicts.

---

## 5. Test Strategy

### 5.1 Unit test — `LifeOpenClawAgentTest`

Pure JUnit 5, no Quarkus. Constructs `Agent` via a `ChatModelProvider` stub backed by a mock
`ChatModel`. Verifies `Agent.execute()` → correct `WorkerResult` output.

`ChatModel` mock: `model.chat(ChatRequest)` returns `ChatResponse` → `aiMessage().text()` →
JSON string (as verified from `Agent.execute()` source).

```java
class LifeOpenClawAgentTest {

    private ChatResponse stubResponse(String json) {
        AiMessage msg = mock(AiMessage.class);
        when(msg.text()).thenReturn(json);
        ChatResponse resp = mock(ChatResponse.class);
        when(resp.aiMessage()).thenReturn(msg);
        return resp;
    }

    @Test
    void execute_booking_returnsPendingAppointment() {
        // "confirmed=false" is correct here: the booking step returns a PENDING booking.
        // confirmed=true is set by the later confirm-appointment binding — not this worker.
        ChatModel mockModel = mock(ChatModel.class);
        when(mockModel.chat(any(ChatRequest.class))).thenReturn(stubResponse(
            "{\"appointmentId\":\"APT-123\",\"provider\":\"Dr Smith\","
            + "\"confirmed\":false,\"declined\":null,\"reason\":null}"));

        ChatModelProvider stub = new ChatModelProvider() {
            public ModelType type() { return ModelType.OPENAI; }
            public ChatModel get() { return mockModel; }
        };

        Agent agent = Agent.builder()
            .model(stub)
            .systemPrompt("You are a booking agent.")
            .userMessage("Book a {{appointmentType}} with {{provider}}.")
            .responseSchema(BookingResult.class)
            .build();

        WorkerResult result = agent.execute(
            Map.of("appointmentType", "GP checkup", "provider", "Dr Smith"));

        assertThat(result.output().get("appointmentId")).isEqualTo("APT-123");
        assertThat(result.output().get("confirmed")).isEqualTo(false);
    }

    @Test
    void execute_unavailableProvider_returnsDeclined() {
        // ...mock returns declined=true...
        assertThat(result.output().get("declined")).isEqualTo(true);
    }
}
```

### 5.2 Integration test — `AppointmentCycleIntegrationTest`

`@QuarkusTest` with `@InjectMock LifeOpenClawChatModelProvider`.

**Critical timing:** `@InjectMock` must be configured before first `getDefinition()` access
because `chatModelProvider.get()` is called exactly once in `Agent.build()` during `augment()`.
The double-checked lock in `getDefinition()` ensures `augment()` only runs once per JVM lifecycle.
Since `@QuarkusTest` bean reset occurs before the test method, `@InjectMock` is established
before any case start triggers `getDefinition()` — timing is safe, but this dependency is
non-obvious and should not be changed (e.g. by moving augmentation to `@PostConstruct`).

`@InjectMock` stubs `openClaw.get()` to return a `ChatModel` that returns a fixed
`BookingResult` JSON. The full case flow is exercised: case starts → `book-appointment-agent`
executes via `Agent.execute(mock)` → `WorkerResult` applied to context → `.booking != null`
is true → `confirm-appointment` binding fires → humanTask created.

All other workers remain stub lambdas — `CaseIntegrationTestSupport` is unchanged.

### 5.3 Test config

`test/resources/application.properties`:
```properties
casehub.life.openclaw.api-url=http://localhost:9999/v1
casehub.life.openclaw.api-key=test-key
casehub.life.openclaw.timeout-seconds=5
```

---

## 6. Protocol Updates

### New: `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`

```
id: PP-20260617-XXXXXX
title: WorkerFunction.AgentExec(Agent) pattern for LLM-backed life workers
```

Rules:
- Use `Worker.builder().function(Agent.builder().model(provider)...build())` for LLM-backed workers
- `Worker.agentDescriptor(AgentDescriptor)` is REQUIRED on every LLM-backed worker —
  without it, trust routing has no identity to score and the attestation pipeline cannot
  attribute outcomes
- `responseSchema(Record.class)` is required — typed structured output prevents hallucinated
  field names and parsing failures
- **Runtime dependency:** OpenClaw must be deployed and accessible at `casehub.life.openclaw.api-url`
  at runtime. Startup succeeds regardless — failure is deferred silently to first
  `agent.execute()` invocation. A `@Observes StartupEvent` health probe is required on every
  service that registers LLM-backed workers
- Config changes to `casehub.life.openclaw.api-url` or `casehub.life.openclaw.timeout-seconds`
  require a restart — `chatModelProvider.get()` is called exactly once in `Agent.build()`,
  which runs during `augment()` on first `getDefinition()` access (double-checked lock)
- `model="openclaw"` is hardcoded in `LifeOpenClawChatModelProvider` — do not use an upstream
  provider model ID (GE-20260614-328420)
- `LifeOpenClawChatModelProvider` is temporary — replace with
  `OpenAiChatModelProvider.builder().baseUrl(...).modelName("openclaw").build()` once
  casehubio/engine#527 lands

### Update: `docs/protocols/casehub-life/PP-20260531-worker-func-exec.md`

| Worker type | Use | When |
|---|---|---|
| Stub / in-process | `Worker.builder().function(lambda)` → `WorkerFunction.Sync` | Temporary stubs and pure CDI service calls |
| LLM-backed (OpenClaw, any LLM API) | `Worker.builder().function(Agent.builder()...build())` → `WorkerFunction.AgentExec` | Real agent calls — requires `agentDescriptor` |
| Multi-step durable | `FuncWorkflowBuilder` or YAML workflow → `WorkerFunction.Flow` | Sequential steps with retry/branching |

---

## 7. Follow-On Issues

Filed before leaving brainstorming:

| Issue | Title | Purpose |
|---|---|---|
| casehubio/engine#527 | Add optional `baseUrl` to `OpenAiChatModelProvider` | Removes `LifeOpenClawChatModelProvider` from life; makes engine-api the general-purpose OpenAI-compatible provider |
| casehubio/life#37 | Layer 7 (full): wire `OpenClawWorkerProvisioner` — heartbeat mode | Full Layer 7 path 1 |
| casehubio/life#38 | Layer 7: `/hooks/agent` direct-call integration | Full Layer 7 path 2 — real skills, calendar, Home Assistant |

---

## 8. Platform Coherence Check

- **Right repo:** life application layer owns domain agent configuration (system prompts,
  response schemas, AgentDescriptor identity). Foundation remains domain-agnostic.
- **Right abstraction:** `WorkerFunction.AgentExec(Agent)` is the engine#463 settled abstraction
  for LLM workers. `ChatModelProvider` SPI in `casehub-engine-api` is the established interface.
- **Module placement:** `LifeOpenClawChatModelProvider` is temporarily in life pending
  casehubio/engine#527. It does NOT belong in `casehub-openclaw-casehub` (would pull in
  CDI-conflicting `WorkerProvisioner`/`WorkerStatusListener`/`CaseChannelProvider` beans) and
  NOT as a permanent life class (generic OpenAI-compatible URL redirect is an engine-api concern).
- **GE-20260614-328420 applied:** `model="openclaw"` hardcoded.
- **Issue 4 build-time behaviour documented:** `get()` called once at `Agent.build()` time.
- **`AgentDescriptor` present:** attestation pipeline and trust routing have agent identity.
- **`AppointmentCycleCaseDefinitions` unaffected:** DSL companion defines static structure
  (capabilities, bindings, goals); workers are runtime behaviour in `augment()`, not in the
  DSL companion. No changes needed there.
