# Design: OpenClaw LLM Backend Integration — AgentExec Pattern Validation (life#25)

**Date:** 2026-06-17
**Issue:** casehubio/life#25
**Branch:** `issue-25-openclaw-worker-provisioner`
**Status:** Approved (rev 5)

---

## 1. What This Is and Is Not

### What it delivers

`WorkerFunction.AgentExec(Agent)` wired end-to-end in casehub-life: one stub worker replaced
with a real LLM call through OpenClaw's OpenAI-compatible API (`POST /v1/chat/completions`,
`model="openclaw"`). Proves the AgentExec execution path from `Agent.execute()` through
`DefaultWorkerExecutor` → `WorkflowExecutionCompletedHandler` → case context update → binding
fires. Establishes `Worker.agentDescriptor()` for the trust system. Sets the agent identity
convention for all future AI workers.

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
                                   AgentDescriptor(agentId="openclaw:health-agent@1", ...))
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
  binding fires → AgentRoutingStrategy selects worker for "book-appointment" capability
                  (by capability name match; "book-appointment-agent" is incidental;
                   AgentDescriptor carries identity for trust scoring — not involved in selection)
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

**Temporary placement.** `casehub-openclaw-casehub` already exists at
`/Users/mdproctor/claude/casehub/openclaw/casehub` and is the natural home for OpenClaw
integration infrastructure. However, adding it as a dependency brings in
`OpenClawWorkerProvisioner`, `ReactiveOpenClawWorkerProvisioner`, `OpenClawWorkerStatusListener`,
and `OpenClawCaseChannelProvider` — all `@ApplicationScoped`, all requiring engine no-op
exclusions via `quarkus.arc.exclude-types`, plus mandatory `OpenClawCasehubConfig` agent
registration. The cleaner path is casehubio/engine#527: add optional `baseUrl` to
`OpenAiChatModelProvider` in engine-api, making it general-purpose for any OpenAI-compatible
server (Ollama, vLLM, OpenClaw). Delete this class and replace callers with
`OpenAiChatModelProvider.builder().baseUrl(...).modelName("openclaw").build()` once that lands.

Uses reflection to set `baseUrl` on `OpenAiChatModel.builder()`, mirroring the existing
pattern in `OpenAiChatModelProvider`. `get()` is called exactly once during `Agent.build()` —
the resulting `ChatModel` is stored in the `Agent` for the JVM lifetime. Config changes
(`api-url`, `timeout-seconds`) require a restart.

```java
@ApplicationScoped
public class LifeOpenClawChatModelProvider implements ChatModelProvider {

    // Temporary — pending casehubio/engine#527 (add baseUrl to OpenAiChatModelProvider).
    // Delete and replace with: OpenAiChatModelProvider.builder().baseUrl(url).modelName("openclaw").build()

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
# Runtime dependency: OpenClaw must be accessible at this URL.
# Failure is NOT detected at startup — deferred silently to first agent.execute() call.
# See OpenClawHealthProbe for the startup warning.
casehub.life.openclaw.api-url=http://localhost:3000/v1
casehub.life.openclaw.api-key=no-key-required
casehub.life.openclaw.timeout-seconds=120

# casehub.life.tenancy-id is required with NO default and must NOT appear in this file with
# an empty value. Quarkus injects "" for a present-but-empty key — AgentDescriptorValidator
# would then throw on the first getDefinition() call, not at startup.
# Omitting the key entirely forces a ConfigException at Quarkus startup — true fail-fast.
# Set this in the deployment environment (env var, Kubernetes secret, etc.).
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

Inject `LifeOpenClawChatModelProvider` and `casehub.life.tenancy-id`. Replace `bookAppointmentWorker()`
with an `AgentExec`-backed worker carrying an `AgentDescriptor`. All other workers remain stub lambdas.

**Agent identity convention** (`docs/specs/life-actor-model.md`):
`{model-family}:{persona}@{major}` — e.g. `claude:health-agent@3`, `openclaw:health-agent@1`.
The `health-agent` persona is the documented name for the health domain AI agent.

**Cross-repo consistency note:** When `WorkerProvisioner` is wired in Full Layer 7 (life#37),
the config map key in `OpenClawCasehubConfig` for the health agent **must be**
`openclaw:health-agent@1` (Quarkus escaping: `casehub.openclaw.agents.openclaw\:health-agent@1`).
`OpenClawWorkerProvisioner.resolveAgentId()` returns the config map key as the agentId and
registers it in `OpenClawAgentRegistry`. `WorkerStatusListener.onWorkerCompleted(agentId, ...)`
uses that same value. The trust system looks up agent scores by `AgentDescriptor.agentId`.
If the provisioner config key differs from the `AgentDescriptor.agentId` set here, trust scoring
cannot correlate executing agents with their descriptors. Establish the full format convention
in the provisioner config from day one — do not use the short form `"health-agent"`.

**`AppointmentCycleCaseDefinitions` note:** The DSL companion defines static structure
(capabilities, bindings, goals) only. Workers are runtime behaviour in `CaseHub.augment()`,
not in the DSL companion. No changes needed there.

```java
@ApplicationScoped
public class AppointmentCycleCaseHub extends YamlCaseHub {

    @Inject LifeOpenClawChatModelProvider openClaw;

    @ConfigProperty(name = "casehub.life.tenancy-id")
    String tenancyId;

    // getDefinition() double-checked lazy init unchanged.
    // augment() runs exactly ONCE on first getDefinition() call.
    // chatModelProvider.get() is called in Agent.build() — once, not per invocation.
    // Config changes to api-url or timeout-seconds require a restart.

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

        // AgentDescriptor.agentId MUST match the casehub-openclaw-casehub config key
        // used by OpenClawWorkerProvisioner when Full Layer 7 lands (life#37).
        // Format: {model-family}:{persona}@{major} per docs/specs/life-actor-model.md.
        AgentDescriptor descriptor = new AgentDescriptor(
            "openclaw:health-agent@1",            // agentId — MATCHES provisioner config key
            "OpenClaw Health Agent",              // name
            "1",                                  // version
            "openclaw",                           // provider
            "openclaw",                           // modelFamily
            null,                                 // modelVersion — unknown; null is honest
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
            tenancyId,                            // tenancyId — required, injected from config
            "Health domain booking and follow-up agent"  // briefing
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

### 3.4 `OpenClawHealthProbe`

OpenClaw has no documented `/health` endpoint. Known endpoints: `POST /hooks/agent`,
`POST /hooks/wake`, `POST /v1/chat/completions`. A TCP connectivity probe confirms the
server is reachable without hitting an undocumented path. Failure is logged as a warning —
startup proceeds regardless (this is a housekeeping signal, not a fatal condition).

`@IfBuildProfile("prod")` suppresses the probe during tests (which run under the default or
`test` profile) and during `quarkus:dev` (which runs under the `dev` profile). In dev mode a
developer won't see a reachability signal at startup; the first agent execution surfaces it.
That is acceptable. Without this gate, test runs log a spurious TCP-connect warning to
`localhost:9999` — noise with no diagnostic value.

```java
@IfBuildProfile("prod")   // io.quarkus.arc.profile.IfBuildProfile
@ApplicationScoped
public class OpenClawHealthProbe {

    @ConfigProperty(name = "casehub.life.openclaw.api-url")
    String apiUrl;

    void onStart(@Observes StartupEvent event) {
        try {
            URI uri = URI.create(apiUrl);
            int port = uri.getPort() > 0 ? uri.getPort()
                : ("https".equals(uri.getScheme()) ? 443 : 80);
            try (var socket = new java.net.Socket()) {
                socket.connect(new java.net.InetSocketAddress(uri.getHost(), port), 3000);
            }
            Log.infof("OpenClaw reachable at %s (TCP)", apiUrl);
        } catch (Exception e) {
            Log.warnf("OpenClaw not reachable at %s — agent workers will fail on first "
                + "invocation: %s", apiUrl, e.getMessage());
        }
    }
}
```

This uses `URI.create()` (no deprecated `new URL()`), extracts host+port without string
manipulation, and does not depend on any specific HTTP endpoint.

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
JSON string. (Verified from `Agent.execute()` source: calls `model.chat(request)` where request
is a `ChatRequest`, response is `ChatResponse`.)

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
        // confirmed=false is correct: booking step returns a PENDING booking.
        // confirmed=true is set by the later confirm-appointment binding, not this worker.
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
        ChatModel mockModel = mock(ChatModel.class);
        when(mockModel.chat(any(ChatRequest.class))).thenReturn(stubResponse(
            "{\"appointmentId\":null,\"provider\":\"Dr Gone\","
            + "\"confirmed\":false,\"declined\":true,\"reason\":\"Not accepting new patients\"}"));

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
            Map.of("appointmentType", "GP checkup", "provider", "Dr Gone"));

        assertThat(result.output().get("declined")).isEqualTo(true);
        assertThat(result.output().get("reason")).isEqualTo("Not accepting new patients");
    }
}
```

### 5.2 Integration test migration — `AppointmentCycleIntegrationTest` and `AppointmentCycleCaseHubTest`

Both `AppointmentCycleIntegrationTest` (3 case-execution tests) and `AppointmentCycleCaseHubTest`
(structural definition tests) are existing `@QuarkusTest` classes that will break after this
spec's change unless migrated.

**Why both classes need migration:**

All `@QuarkusTest` classes share the same Quarkus instance by default. `augmentedDefinition` is
a `volatile` field on the `@ApplicationScoped` `AppointmentCycleCaseHub` singleton — it is baked
exactly once per JVM lifetime (double-checked lock in `getDefinition()`). `AppointmentCycleCaseHubTest`
calls `caseHub.getDefinition()` in `hasFiveWorkers()` and other tests — this triggers `augment()`
and calls `openClaw.get()`. Alphabetical ordering means `AppointmentCycleCaseHubTest` runs first.
Without `@InjectMock`, the REAL `LifeOpenClawChatModelProvider.get()` builds a real `OpenAiChatModel`
with `baseUrl=localhost:9999` and bakes it permanently. When `AppointmentCycleIntegrationTest` then
sets up `@InjectMock`, `get()` is never called again — `agent.execute()` uses the real model,
connects to `localhost:9999/v1/chat/completions`, gets connection refused, and all three tests fail.

**Fix: add the same `@InjectMock` and request-aware `@BeforeEach` to BOTH classes.**

Whichever class runs first has its `@BeforeEach` execute before the first `getDefinition()` call,
stubs `openClaw.get()` with a request-aware `ChatModel`, and bakes it. The other class's
`@BeforeEach` then stubs `get()` again — but `get()` is never called again. The baked
request-aware `ChatModel` serves all test methods across both classes.

**Request-aware mock is required** because `declinePath_findsAlternativeAndContinues()` passes
`"provider": "unavailable"`. The user message template is
`"Book a {{appointmentType}} appointment with provider {{provider}}."` which renders to
`"Book a GP appointment with provider unavailable."`. The mock must detect this string in the
rendered user message and return the decline response. A single-response stub would break
the decline-path test.

**Shared mock setup** (identical `@InjectMock` + `@BeforeEach` in both classes):

```java
@InjectMock
LifeOpenClawChatModelProvider openClaw;

@BeforeEach
void setupOpenClaw() {
    // openClaw.get() is called AT MOST ONCE across all test classes —
    // during the first getDefinition() call that triggers augment().
    // The ChatModel returned here is baked permanently for the JVM lifetime.
    // Must be request-aware: declinePath sends "provider=unavailable",
    // golden-path and sequential tests send real provider names.
    ChatModel requestAware = mock(ChatModel.class);
    when(requestAware.chat(any(ChatRequest.class))).thenAnswer(invocation -> {
        ChatRequest req = invocation.getArgument(0);
        String userText = req.messages().stream()
            .filter(m -> m instanceof UserMessage)
            .map(m -> ((UserMessage) m).singleText())
            .findFirst().orElse("");
        if (userText.toLowerCase().contains("unavailable")) {
            return stubChatResponse(
                "{\"appointmentId\":null,\"provider\":\"Dr Gone\","
                + "\"confirmed\":false,\"declined\":true,\"reason\":\"Provider unavailable\"}");
        }
        return stubChatResponse(
            "{\"appointmentId\":\"APT-" + System.currentTimeMillis()
            + "\",\"provider\":\"Dr Smith\",\"confirmed\":false,"
            + "\"declined\":null,\"reason\":null}");
    });
    when(openClaw.get()).thenReturn(requestAware);
    when(openClaw.type()).thenReturn(ModelType.OPENAI);
}

// helper — place in each class or extract to CaseIntegrationTestSupport
private static ChatResponse stubChatResponse(String json) {
    AiMessage msg = mock(AiMessage.class);
    when(msg.text()).thenReturn(json);
    ChatResponse resp = mock(ChatResponse.class);
    when(resp.aiMessage()).thenReturn(msg);
    return resp;
}
```

`AppointmentCycleCaseHubTest` structural tests are otherwise unchanged — no test bodies need
updating. `AppointmentCycleIntegrationTest` test bodies are unchanged — all three tests pass
with the baked request-aware `ChatModel`.

**Optional new test in `AppointmentCycleCaseHubTest`** (validates the descriptor is wired):

```java
@Test
void bookAppointmentWorkerHasAgentDescriptor() {
    var worker = caseHub.getDefinition().getWorkers().stream()
        .filter(w -> "book-appointment-agent".equals(w.getName()))
        .findFirst().orElseThrow();
    assertNotNull(worker.agentDescriptor(), "AgentDescriptor must be set — not build-time enforced");
    assertEquals("openclaw:health-agent@1", worker.agentDescriptor().agentId());
}
```

Test config (`test/resources/application.properties`):
```properties
casehub.life.openclaw.api-url=http://localhost:9999/v1
casehub.life.openclaw.api-key=test-key
casehub.life.openclaw.timeout-seconds=5
# Required property — must appear in test config so Quarkus doesn't throw ConfigException at startup.
# OpenClawHealthProbe is suppressed via @IfBuildProfile("prod"), so no TCP connect is attempted.
casehub.life.tenancy-id=278776f9-e1b0-46fb-9032-8bddebdcf9ce
```

---

## 6. Protocol Updates

### New: `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`

Rules:
- Use `Worker.builder().function(Agent.builder().model(provider)...build())` for LLM-backed workers
- **`Worker.agentDescriptor(AgentDescriptor)` is architecturally required (not build-time enforced)**
  — `Worker.Builder.build()` does not validate the field; omitting it compiles cleanly and
  silently produces a worker with null descriptor. Trust routing has no identity to score and
  the attestation pipeline cannot attribute outcomes. Treat as required by convention.
- **Agent identity format:** `{model-family}:{persona}@{major}` per `docs/specs/life-actor-model.md`
  (e.g. `"openclaw:health-agent@1"`, `"claude:health-agent@3"`). This value must match the
  config map key used in `casehub-openclaw-casehub` when `WorkerProvisioner` is wired
- **`responseSchema(Record.class)` is required** — typed structured output prevents hallucinated
  field names and parse failures
- **Runtime dependency:** OpenClaw must be deployed and accessible at `casehub.life.openclaw.api-url`
  at runtime. Startup succeeds regardless — failure is deferred silently to first
  `agent.execute()` invocation. `@Observes StartupEvent` TCP health probe is required on every
  deployment that registers LLM-backed workers
- **Config changes require restart:** `chatModelProvider.get()` is called exactly once in
  `Agent.build()`, which runs during `augment()` on first `getDefinition()` access (double-checked
  lock). Changing `api-url` or `timeout-seconds` without restart has no effect
- `model="openclaw"` is hardcoded in `LifeOpenClawChatModelProvider` — do not use an upstream
  provider model ID (GE-20260614-328420)
- `casehub.life.tenancy-id` is required with no default — fail fast on startup if not configured
- `LifeOpenClawChatModelProvider` is temporary — delete once casehubio/engine#527 lands

### Update: `docs/protocols/casehub-life/PP-20260531-worker-func-exec.md`

| Worker type | Use | When |
|---|---|---|
| Stub / in-process | `Worker.builder().function(lambda)` → `WorkerFunction.Sync` | Temporary stubs and pure CDI service calls |
| LLM-backed (OpenClaw, any LLM API) | `Worker.builder().function(Agent.builder()...build())` → `WorkerFunction.AgentExec` | Real agent calls — `agentDescriptor` required |
| Multi-step durable | `FuncWorkflowBuilder` or YAML workflow → `WorkerFunction.Flow` | Sequential steps with retry/branching |

---

## 7. Follow-On Issues

| Issue | Title | Purpose |
|---|---|---|
| casehubio/engine#527 | Add optional `baseUrl` to `OpenAiChatModelProvider` | Removes `LifeOpenClawChatModelProvider` from life |
| casehubio/life#37 | Layer 7 (full): wire `OpenClawWorkerProvisioner` — heartbeat mode | Full Layer 7 path 1 |
| casehubio/life#38 | Layer 7: `/hooks/agent` direct-call integration | Full Layer 7 path 2 — real skills |

---

## 8. Platform Coherence

- **Right repo:** life owns domain agent configuration (system prompts, response schemas,
  AgentDescriptor identity). Foundation remains domain-agnostic.
- **Right abstraction:** `WorkerFunction.AgentExec(Agent)` is the engine#463 settled abstraction.
- **Module placement:** `LifeOpenClawChatModelProvider` is temporarily in life pending
  casehubio/engine#527. `casehub-openclaw-casehub` exists but adding it as a dependency
  introduces CDI-conflicting beans requiring no-op exclusions and mandatory agent config —
  avoided here; the engine#527 path is cleaner for a ChatModelProvider that is general to any
  OpenAI-compatible endpoint, not specific to casehub-openclaw's provisioner infrastructure.
- **agentId consistency:** `"openclaw:health-agent@1"` matches the `{model-family}:{persona}@{major}`
  convention from `docs/specs/life-actor-model.md`. Must be used as the casehub-openclaw-casehub
  config key when WorkerProvisioner is wired.
- **GE-20260614-328420 applied:** `model="openclaw"` hardcoded.
- **Build eagerness documented:** `get()` called once at `Agent.build()` time; restart required
  for config changes.
- **AgentDescriptor present:** attestation and trust routing have agent identity.
- **AppointmentCycleCaseDefinitions unaffected:** DSL companion is structure only; no changes needed.
- **OpenClawHealthProbe:** TCP probe, no string manipulation, no undocumented HTTP endpoints.
