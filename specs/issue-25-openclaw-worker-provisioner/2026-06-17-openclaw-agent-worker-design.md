# Design: OpenClaw Agent Workers — First Real Workers (life#25)

**Date:** 2026-06-17  
**Issue:** casehubio/life#25  
**Branch:** `issue-25-openclaw-worker-provisioner`  
**Status:** Approved

---

## 1. Context and Scope

### What this is

Replace the first stub `WorkerFunction.Sync(hardcoded-lambda)` with `WorkerFunction.AgentExec(Agent)`
where the Agent calls OpenClaw's OpenAI-compatible API (`POST /v1/chat/completions`,
`model="openclaw"`). This is the settled abstraction from casehubio/engine#463.

### What this is NOT

- Not `WorkerProvisioner` / heartbeat mode (that is Full Layer 7 — see §7)
- Not removal of all stub workers (remaining 7 case types stay as stubs)
- Not migration to the descriptor+handler pattern (separate refactor)
- Not enabling `casehub.qhorus.reactive.enabled=true` (not needed for this pattern)

### Why AgentExec, not WorkerProvisioner

The engine's `CaseContextChangedEventHandler` uses `ReactiveWorkerProvisioner` (not blocking
`WorkerProvisioner`) for binding-based flows. Enabling reactive provisioner requires the full
qhorus reactive stack (`casehub.qhorus.reactive.enabled=true`) and a properly wired COMMAND
dispatch path — both are out of scope here. The `WorkerFunction.AgentExec(Agent)` pattern uses
the existing inline-worker path unchanged: Worker in CaseDefinition → `AgentRoutingStrategy.select()`
→ `WorkerScheduleEvent` → `DefaultWorkerExecutor.executeSync(agent::execute, ...)` on a virtual
thread → `WorkflowExecutionCompletedHandler` applies output to context → binding fires.

OpenClaw exposes an OpenAI-compatible API (`/v1/chat/completions`, `model="openclaw"`). The
`Agent.execute()` call dispatches synchronously to OpenClaw via LangChain4J's `OpenAiChatModel`
and blocks on a virtual thread until the response arrives. This is the direct-call mode.

---

## 2. Architecture Overview

```
AppointmentCycleCaseHub
  └── augment(yaml) → adds Worker("book-appointment-agent", AgentExec(bookingAgent))
                           stub lambdas for all other workers (unchanged)

bookingAgent = Agent.builder()
    .model(openClawProvider)       ← OpenClawChatModelProvider (new @ApplicationScoped bean)
    .systemPrompt(...)
    .userMessage(...)
    .responseSchema(BookingResult.class)
    .build()

At runtime:
  binding fires → AgentRoutingStrategy selects "book-appointment-agent"
  → WorkerScheduleEvent → DefaultWorkerExecutor.executeSync(agent::execute, inputData, ...)
  → OpenClawChatModelProvider.get() → OpenAiChatModel(baseUrl=openclaw, model="openclaw")
  → POST /v1/chat/completions → OpenClaw runs → JSON response
  → Agent parses response via responseSchema → WorkerResult
  → WorkflowExecutionCompletedHandler applies output → .booking != null → next binding fires
```

---

## 3. New Components

### 3.1 `OpenClawChatModelProvider`

**Package:** `io.casehub.life.app.engine.agent`  
**Type:** `@ApplicationScoped` CDI bean  
**Implements:** `io.casehub.api.model.ai.ChatModelProvider`

Reads three Quarkus config properties and builds `OpenAiChatModel` via reflection, mirroring
the pattern in `OpenAiChatModelProvider` from `casehub-engine-api`. The `baseUrl` parameter
points at OpenClaw's endpoint instead of OpenAI's.

```java
@ApplicationScoped
public class OpenClawChatModelProvider implements ChatModelProvider {

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
            invoke(bc, builder, "baseUrl", String.class, apiUrl);
            invoke(bc, builder, "apiKey", String.class, apiKey);
            invoke(bc, builder, "modelName", String.class, "openclaw");
            invoke(bc, builder, "timeout", Duration.class, Duration.ofSeconds(timeoutSeconds));
            return (ChatModel) bc.getMethod("build").invoke(builder);
        } catch (InvocationTargetException e) {
            Throwable cause = e.getCause() != null ? e.getCause() : e;
            throw new AgentException("Failed to build OpenClawChatModel: " + cause.getMessage(), cause);
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
# OpenClaw AI backend — direct-call mode
casehub.life.openclaw.api-url=http://localhost:3000/v1
casehub.life.openclaw.api-key=no-key-required
casehub.life.openclaw.timeout-seconds=120
```

**GE-20260614-328420:** `model="openclaw"` is required — the upstream provider model ID is
rejected by OpenClaw's `/v1/chat/completions` endpoint.

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

`AgentBuilder.responseSchema(BookingResult.class)` derives the JSON schema automatically.
`Agent.execute()` enforces structured output — OpenClaw returns conforming JSON.

### 3.3 `AppointmentCycleCaseHub` — modified

Inject `OpenClawChatModelProvider`. Replace the `bookAppointmentWorker()` lambda with an
`Agent`-backed worker. All other workers remain as stub lambdas.

```java
@ApplicationScoped
public class AppointmentCycleCaseHub extends YamlCaseHub {

    @Inject OpenClawChatModelProvider openClaw;   // new injection

    // getDefinition() lazy-init unchanged

    private Worker bookAppointmentWorker() {
        return Worker.builder()
            .name("book-appointment-agent")
            .capabilities(List.of(cap("book-appointment")))
            .function(Agent.builder()
                .model(openClaw)
                .systemPrompt("""
                    You are a healthcare appointment booking agent for a UK household.
                    Your task: book medical appointments with the requested provider.
                    If the provider is unavailable, set declined=true and provide a reason.
                    Always respond with valid JSON only — no prose, no explanation.
                    """)
                .userMessage(
                    "Book a {{appointmentType}} appointment with provider {{provider}}.")
                .responseSchema(BookingResult.class)
                .build())
            .build();
    }

    // findAlternativeWorker, confirmAppointmentWorker, preVisitPrepWorker,
    // recordHealthDecisionWorker — all unchanged stub lambdas
}
```

### 3.4 Maven dependency addition

In `app/pom.xml`, add `langchain4j-open-ai` at `runtime` scope. The base `langchain4j`
API (containing `ChatModel`) is already on the classpath transitively via `casehub-engine-api`
(compile scope). The `OpenAiChatModel` implementation is loaded reflectively at runtime.

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
    <scope>runtime</scope>
</dependency>
```

No version needed — managed by `casehub-life-parent` BOM (which inherits from `casehub-parent`
which manages LangChain4J versions).

---

## 4. CDI Wiring

No changes to `quarkus.arc.exclude-types` or `quarkus.arc.selected-alternatives`. The
`OpenClawChatModelProvider` is a new `@ApplicationScoped` bean with no conflicts. No engine
no-op beans are affected because we are adding an inline worker (the AgentExec path), not
replacing a `WorkerProvisioner` no-op.

---

## 5. Test Strategy

### 5.1 Unit test — `LifeOpenClawAgentTest`

Pure JUnit 5, no Quarkus. Tests `Agent.execute()` in isolation via a `ChatModelProvider` stub.

```java
class LifeOpenClawAgentTest {

    ChatModel mockModel;
    ChatModelProvider stubProvider;

    // Agent.execute() calls model.chat(ChatRequest) → ChatResponse → aiMessage().text()

    private ChatResponse stubResponse(String json) {
        AiMessage msg = mock(AiMessage.class);
        when(msg.text()).thenReturn(json);
        ChatResponse resp = mock(ChatResponse.class);
        when(resp.aiMessage()).thenReturn(msg);
        return resp;
    }

    @BeforeEach
    void setup() {
        mockModel = mock(ChatModel.class);
        stubProvider = new ChatModelProvider() {
            public ModelType type() { return ModelType.OPENAI; }
            public ChatModel get() { return mockModel; }
        };
    }

    @Test
    void execute_confirmedBooking_returnsWorkerResult() {
        when(mockModel.chat(any(ChatRequest.class))).thenReturn(stubResponse(
            "{\"appointmentId\":\"APT-123\",\"provider\":\"Dr Smith\","
            + "\"confirmed\":false,\"declined\":null,\"reason\":null}"));

        Agent agent = Agent.builder()
            .model(stubProvider)
            .systemPrompt("You are a booking agent...")
            .userMessage("Book a {{appointmentType}} with {{provider}}.")
            .responseSchema(BookingResult.class)
            .build();

        WorkerResult result = agent.execute(
            Map.of("appointmentType", "GP checkup", "provider", "Dr Smith"));

        assertThat(result.output()).containsKey("appointmentId");
        assertThat(result.output().get("confirmed")).isEqualTo(false);
    }

    @Test
    void execute_unavailableProvider_returnsDeclined() {
        when(mockModel.chat(any(ChatRequest.class))).thenReturn(stubResponse(
            "{\"appointmentId\":null,\"provider\":\"Dr Gone\","
            + "\"confirmed\":false,\"declined\":true,\"reason\":\"Not accepting patients\"}"));

        // ... agent.execute() ...
        assertThat(result.output().get("declined")).isEqualTo(true);
    }
}
```

### 5.2 Integration test — `AppointmentCycleIntegrationTest`

`@QuarkusTest` with `@InjectMock OpenClawChatModelProvider`. The mock provider returns a stub
`ChatModel` that returns a fixed JSON response. The full case flow is exercised unchanged:
start case → `book-appointment-agent` executes via `Agent.execute(mock)` → `WorkerResult` applied
→ `confirm-appointment` binding fires (`.booking != null`) → confirms humanTask creation.

`@InjectMock` replaces the `@ApplicationScoped` `OpenClawChatModelProvider` bean for the test.
Stub lambdas for all other workers remain — no changes to `CaseIntegrationTestSupport`.

### 5.3 Test config

`test/resources/application.properties`:
```properties
# Placeholder — overridden by @InjectMock in integration tests
casehub.life.openclaw.api-url=http://localhost:9999/v1
casehub.life.openclaw.api-key=test-key
casehub.life.openclaw.timeout-seconds=5
```

---

## 6. Protocol Updates

### New: `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`

Establishes the `WorkerFunction.AgentExec(Agent)` pattern for life domain workers:
- Use `Agent.builder().model(openClawProvider)...build()` for workers that delegate to OpenClaw
- `OpenClawChatModelProvider` is the single shared CDI provider for all life OpenClaw workers
- `responseSchema(Record.class)` is required — typed structured output prevents hallucinated field names
- Timeout configured via `casehub.life.openclaw.timeout-seconds` (default 120s)
- System prompt must instruct agent to return JSON only — no prose
- Worker name convention: `{capability-name}-agent` (matches existing stub names)

### Update: `docs/protocols/casehub-life/PP-20260531-worker-func-exec.md`

Reflect the engine#463 settled design:

| Worker type | Use | When |
|---|---|---|
| Stub / in-process utility | `Worker.builder().function(lambda)` → `WorkerFunction.Sync` | Temporary stubs, CDI service calls that return immediately |
| OpenClaw / LLM agent | `Worker.builder().function(Agent.builder()...build())` → `WorkerFunction.AgentExec` | Real agents, LLM-backed workers, OpenClaw direct-call |
| Multi-step durable | `FuncWorkflowBuilder` or YAML workflow → `WorkerFunction.Flow` | Sequential steps with retry, branching, or error recovery per step |

---

## 7. Out of Scope — Follow-on Issues to File

**Before leaving brainstorm**, file:

1. **Full Layer 7 — `WorkerProvisioner` heartbeat mode**: wire `casehub-openclaw-casehub`,
   implement `LifeReactiveWorkerProvisioner implements ReactiveWorkerProvisioner` (wraps blocking
   `OpenClawWorkerProvisioner`), `OpenClawChannelBackend` wiring, COMMAND dispatch path from
   `WorkerScheduleEventHandler.dispatchCommand()` to OpenClaw. Requires decision on whether to
   enable `casehub.qhorus.reactive.enabled=true` (and implications for qhorus datasource).

2. **Migrate remaining 7 case type stub workers** to `WorkerFunction.AgentExec` once the
   OpenClaw integration is validated end-to-end with the first real worker.

3. **Descriptor+handler pattern migration**: move worker lambdas/agents from `*CaseHub.augment()`
   into `*CaseDescriptor` POJOs per the PLATFORM.md protocol (FuncDSL companions superseded).

---

## 8. Platform Coherence Check

- **Right repo:** life application layer owns domain agent configuration (system prompts,
  response schemas, domain-specific model routing). Foundation remains domain-agnostic.
- **Right abstraction:** `WorkerFunction.AgentExec(Agent)` is the settled engine#463 abstraction
  for LLM/agent workers. `ChatModelProvider` SPI in `casehub-engine-api` is the established
  interface. No new platform abstractions needed.
- **Consolidation:** `OpenClawChatModelProvider` is life-specific (OpenClaw base URL config).
  If other harnesses adopt OpenClaw, a shared `OpenClawChatModelProvider` could move to
  `casehub-openclaw-casehub` — not premature to place it in life now.
- **Pattern consistency:** mirrors how `OpenAiChatModelProvider` works in `casehub-engine-api`
  (reflection-based builder, no compile-time dependency on the concrete model class).
- **GE-20260614-328420 applied:** `model="openclaw"` hard-coded in provider.
