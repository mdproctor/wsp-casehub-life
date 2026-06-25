# Phase 1: OpenClaw Direct-Call Bridge — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add synchronous webhook bridge infrastructure to casehub-openclaw so that casehub-life (and other harnesses) can call `/hooks/agent` from `WorkerFunction.AgentExec` workers and get structured results back.

**Architecture:** Implements `AgentProvider` (platform SPI) with a thin langchain4j `ChatModel` bridge, following the `ClaudeAgentProvider` / `ClaudeAgentChatModel` pattern. `OpenClawAgentProvider.invoke()` fires `/hooks/agent` via `OpenClawHookClient.invokeDirect()`, registers a `CompletableFuture` in `DirectCallBridge`, and emits `Multi<AgentEvent>` when the webhook callback completes the future. `OpenClawChatModel.doChat()` bridges from langchain4j to `AgentProvider`, extracting system prompt, user text, and JSON schema from `ChatRequest`.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny (Multi/Uni), langchain4j-core 1.14.1, casehub-platform-agent-api

**Issue:** casehubio/openclaw#49
**Design spec:** casehub-life workspace `specs/2026-06-24-hooks-agent-direct-call-design.md` (rev 5)

## Global Constraints

- Quarkus 3.32.2, Java 21 (on Java 26 JVM)
- `casehub-platform-agent-api` 0.2-SNAPSHOT — `AgentProvider`, `AgentSessionConfig`, `AgentEvent`
- `OpenClawHookClient` is in `casehub-openclaw-core` (`io.casehub.openclaw.client`)
- New classes go in `casehub-openclaw-casehub` (`io.casehub.openclaw.casehub`) unless noted
- `@PermitAll` on webhook endpoints — OpenClaw callbacks carry no OIDC token
- All delivery endpoints return 200 always — OpenClaw must not retry
- Tests use `mvn test -pl casehub` or `mvn test -pl core` scoped to the changed module

---

### Task 1: invokeDirect() sessionless overload on OpenClawHookClient

**Files:**
- Modify: `core/src/main/java/io/casehub/openclaw/client/OpenClawHookClient.java`
- Test: `core/src/test/java/io/casehub/openclaw/client/OpenClawHookClientTest.java`

**Interfaces:**
- Consumes: `invokeInternal(agentId, message, model, timeoutSeconds, deliveryUrl, sessionKey)` — existing private method
- Produces: `invokeDirect(String agentId, String message, String model, int timeoutSeconds, String deliveryUrl)` — new public method, called by `OpenClawAgentProvider`

- [ ] **Step 1: Write the failing test**

```java
@Test
void invokeDirect_noSessionRequired() {
    // Setup: no registerSession() call — invoking with no session
    // The mock gateway should receive the request with sessionName=null
    OpenClawHookClient client = new OpenClawHookClient(mockGateway, config);

    client.invokeDirect("health-agent", "Book appointment", null, 30,
            "https://casehub.internal/openclaw/direct-call/abc-123");

    // Verify gateway received request with sessionName=null
    ArgumentCaptor<AgentInvocationRequest> captor =
            ArgumentCaptor.forClass(AgentInvocationRequest.class);
    verify(mockGateway).invokeAgent(captor.capture());
    AgentInvocationRequest req = captor.getValue();
    assertEquals("health-agent", req.agentId());
    assertEquals("Book appointment", req.message());
    assertEquals("webhook", req.deliver());
    assertEquals("https://casehub.internal/openclaw/direct-call/abc-123", req.to());
    assertNull(req.sessionName());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=OpenClawHookClientTest#invokeDirect_noSessionRequired --batch-mode`
Expected: FAIL — `invokeDirect` method doesn't exist

- [ ] **Step 3: Write minimal implementation**

Add to `OpenClawHookClient.java`:

```java
/**
 * Invokes an OpenClaw agent via the hook API without requiring a registered session.
 * Uses the gateway bearer token from OpenClawClientConfig for authentication.
 * sessionName is null — OpenClaw accepts this (field is @JsonInclude(NON_NULL)).
 *
 * <p>Designed for the direct-call pattern where each invocation uses a unique
 * delivery URL (including a correlationId). Persistent sessions are a category
 * mismatch for per-invocation URLs.
 *
 * @param agentId        OpenClaw agent identifier
 * @param message        prompt to deliver to the agent
 * @param model          model to use; null or blank uses the configured default
 * @param timeoutSeconds invocation timeout; 0 uses the configured default
 * @param deliveryUrl    the webhook URL OpenClaw will POST the result to
 * @throws OpenClawInvocationException if the gateway returns non-2xx
 */
public void invokeDirect(String agentId, String message, String model,
                          int timeoutSeconds, String deliveryUrl) {
    invokeInternal(agentId, message, model, timeoutSeconds, deliveryUrl, null);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=OpenClawHookClientTest#invokeDirect_noSessionRequired --batch-mode`
Expected: PASS

- [ ] **Step 5: Write additional test — JSON omits sessionName when null**

```java
@Test
void invokeDirect_sessionNameOmittedFromJson() throws Exception {
    OpenClawHookClient client = new OpenClawHookClient(mockGateway, config);
    client.invokeDirect("test-agent", "hello", null, 30, "https://example.com/cb");

    ArgumentCaptor<AgentInvocationRequest> captor =
            ArgumentCaptor.forClass(AgentInvocationRequest.class);
    verify(mockGateway).invokeAgent(captor.capture());

    String json = new ObjectMapper().writeValueAsString(captor.getValue());
    assertFalse(json.contains("sessionName"),
            "sessionName should be omitted from JSON when null");
}
```

- [ ] **Step 6: Run all OpenClawHookClient tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=OpenClawHookClientTest --batch-mode`
Expected: All PASS

- [ ] **Step 7: Commit**

```
feat(#49): add invokeDirect() sessionless overload to OpenClawHookClient

Direct-call mode uses per-invocation delivery URLs (including correlationId).
Persistent sessions are a category mismatch. sessionName=null — OpenClaw
accepts this (@JsonInclude(NON_NULL)).

Refs casehubio/openclaw#49
```

---

### Task 2: DirectCallBridge + DirectCallDeliveryResource

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/DirectCallBridge.java`
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/DirectCallDeliveryResource.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/DirectCallBridgeTest.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/DirectCallDeliveryResourceTest.java`

**Interfaces:**
- Consumes: `OpenClawDeliveryPayload` — existing record in `io.casehub.openclaw.app`
- Produces: `DirectCallBridge.submit(String correlationId) → CompletableFuture<String>`, `DirectCallBridge.complete(String correlationId, String responseText)`, `DirectCallBridge.cancel(String correlationId)`

Note: `OpenClawDeliveryPayload` is in the `app` module. Check if it needs to be moved to `casehub` module or if a new delivery payload record is needed for direct-call. If it's app-only, create a `DirectCallDeliveryPayload` record in `casehub`.

- [ ] **Step 1: Write DirectCallBridge tests**

```java
@Test
void submit_createsAndReturnsFuture() {
    DirectCallBridge bridge = new DirectCallBridge();
    CompletableFuture<String> future = bridge.submit("corr-1");
    assertNotNull(future);
    assertFalse(future.isDone());
}

@Test
void complete_resolvesFuture() throws Exception {
    DirectCallBridge bridge = new DirectCallBridge();
    CompletableFuture<String> future = bridge.submit("corr-1");
    bridge.complete("corr-1", "{\"result\":\"ok\"}");
    assertTrue(future.isDone());
    assertEquals("{\"result\":\"ok\"}", future.get());
}

@Test
void cancel_cancelsFuture() {
    DirectCallBridge bridge = new DirectCallBridge();
    CompletableFuture<String> future = bridge.submit("corr-1");
    bridge.cancel("corr-1");
    assertTrue(future.isCancelled());
}

@Test
void complete_afterCancel_isNoOp() {
    DirectCallBridge bridge = new DirectCallBridge();
    CompletableFuture<String> future = bridge.submit("corr-1");
    bridge.cancel("corr-1");
    bridge.complete("corr-1", "late response");
    assertTrue(future.isCancelled());
}

@Test
void complete_unknownCorrelationId_isNoOp() {
    DirectCallBridge bridge = new DirectCallBridge();
    bridge.complete("unknown", "response");
    // No exception, no side effect
}

@Test
void cancel_unknownCorrelationId_isNoOp() {
    DirectCallBridge bridge = new DirectCallBridge();
    bridge.cancel("unknown");
    // No exception, no side effect
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=DirectCallBridgeTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement DirectCallBridge**

```java
package io.casehub.openclaw.casehub;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class DirectCallBridge {

    private final ConcurrentHashMap<String, CompletableFuture<String>> futures =
            new ConcurrentHashMap<>();

    public CompletableFuture<String> submit(String correlationId) {
        CompletableFuture<String> future = new CompletableFuture<>();
        futures.put(correlationId, future);
        return future;
    }

    public void complete(String correlationId, String responseText) {
        CompletableFuture<String> future = futures.remove(correlationId);
        if (future != null) {
            future.complete(responseText);
        }
    }

    public void cancel(String correlationId) {
        CompletableFuture<String> future = futures.remove(correlationId);
        if (future != null) {
            future.cancel(true);
        }
    }
}
```

- [ ] **Step 4: Run DirectCallBridge tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=DirectCallBridgeTest --batch-mode`
Expected: All PASS

- [ ] **Step 5: Write DirectCallDeliveryResource tests**

```java
@Test
void deliver_completesTheBridge() {
    DirectCallBridge bridge = new DirectCallBridge();
    DirectCallDeliveryResource resource = new DirectCallDeliveryResource(bridge);
    CompletableFuture<String> future = bridge.submit("corr-1");

    Response response = resource.deliver("corr-1",
            new DirectCallDeliveryPayload("health-agent", "result text"));

    assertEquals(200, response.getStatus());
    assertTrue(future.isDone());
    assertEquals("result text", future.getNow(null));
}

@Test
void deliver_unknownCorrelationId_returns200() {
    DirectCallBridge bridge = new DirectCallBridge();
    DirectCallDeliveryResource resource = new DirectCallDeliveryResource(bridge);

    Response response = resource.deliver("unknown",
            new DirectCallDeliveryPayload("agent", "text"));

    assertEquals(200, response.getStatus());
}

@Test
void deliver_nullPayload_returns200WithEmptyOutput() {
    DirectCallBridge bridge = new DirectCallBridge();
    DirectCallDeliveryResource resource = new DirectCallDeliveryResource(bridge);
    CompletableFuture<String> future = bridge.submit("corr-1");

    Response response = resource.deliver("corr-1", null);

    assertEquals(200, response.getStatus());
    assertEquals("", future.getNow(null));
}
```

- [ ] **Step 6: Implement DirectCallDeliveryPayload + DirectCallDeliveryResource**

```java
package io.casehub.openclaw.casehub;

public record DirectCallDeliveryPayload(String agentId, String output) {}
```

```java
package io.casehub.openclaw.casehub;

import jakarta.annotation.security.PermitAll;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@PermitAll
@Path("/openclaw/direct-call")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public class DirectCallDeliveryResource {

    private final DirectCallBridge bridge;

    @Inject
    public DirectCallDeliveryResource(DirectCallBridge bridge) {
        this.bridge = bridge;
    }

    @POST
    @Path("/{correlationId}")
    public Response deliver(@PathParam("correlationId") String correlationId,
                             DirectCallDeliveryPayload payload) {
        String output = payload != null && payload.output() != null
                ? payload.output() : "";
        bridge.complete(correlationId, output);
        return Response.ok().build();
    }
}
```

- [ ] **Step 7: Run all bridge + delivery tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest="DirectCallBridgeTest,DirectCallDeliveryResourceTest" --batch-mode`
Expected: All PASS

- [ ] **Step 8: Commit**

```
feat(#49): add DirectCallBridge and DirectCallDeliveryResource

DirectCallBridge manages pending CompletableFutures keyed by correlationId.
DirectCallDeliveryResource receives OpenClaw webhook callbacks at
POST /openclaw/direct-call/{correlationId} and completes the matching future.

Refs casehubio/openclaw#49
```

---

### Task 3: OpenClawAgentProvider + OpenClawChatModel

**Files:**
- Modify: `casehub/pom.xml` — add `casehub-platform-agent-api` dependency
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentProvider.java`
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChatModel.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentProviderTest.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawChatModelTest.java`

**Interfaces:**
- Consumes: `DirectCallBridge` (Task 2), `OpenClawHookClient.invokeDirect()` (Task 1), `AgentProvider` / `AgentSessionConfig` / `AgentEvent` (casehub-platform-agent-api)
- Produces: `OpenClawAgentProvider(DirectCallBridge, OpenClawHookClient, String agentId, String deliveryBaseUrl)`, `OpenClawChatModel(OpenClawAgentProvider, Duration timeout)`

- [ ] **Step 1: Add casehub-platform-agent-api dependency to casehub/pom.xml**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-api</artifactId>
</dependency>
```

- [ ] **Step 2: Write OpenClawAgentProvider tests**

```java
@Test
void invoke_callsInvokeDirectWithCorrectDeliveryUrl() {
    DirectCallBridge bridge = new DirectCallBridge();
    OpenClawHookClient hookClient = mock(OpenClawHookClient.class);
    OpenClawAgentProvider provider = new OpenClawAgentProvider(
            bridge, hookClient, "health-agent", "https://casehub.internal");

    AgentSessionConfig config = new AgentSessionConfig(
            "You are a health agent", "Book appointment with Dr Smith",
            List.of(), Duration.ofSeconds(30), "test-corr-id");

    // Subscribe to trigger invocation (don't await — just verify the call)
    provider.invoke(config).subscribe().with(e -> {}, e -> {});

    ArgumentCaptor<String> urlCaptor = ArgumentCaptor.forClass(String.class);
    verify(hookClient).invokeDirect(eq("health-agent"), anyString(),
            isNull(), eq(30), urlCaptor.capture());
    assertTrue(urlCaptor.getValue().startsWith(
            "https://casehub.internal/openclaw/direct-call/"));
}

@Test
void invoke_combinesSystemPromptAndUserPrompt() {
    DirectCallBridge bridge = new DirectCallBridge();
    OpenClawHookClient hookClient = mock(OpenClawHookClient.class);
    OpenClawAgentProvider provider = new OpenClawAgentProvider(
            bridge, hookClient, "health-agent", "https://casehub.internal");

    AgentSessionConfig config = new AgentSessionConfig(
            "System prompt here", "User prompt here",
            List.of(), Duration.ofSeconds(30), null);

    provider.invoke(config).subscribe().with(e -> {}, e -> {});

    ArgumentCaptor<String> msgCaptor = ArgumentCaptor.forClass(String.class);
    verify(hookClient).invokeDirect(anyString(), msgCaptor.capture(),
            isNull(), anyInt(), anyString());
    String message = msgCaptor.getValue();
    assertTrue(message.contains("System prompt here"));
    assertTrue(message.contains("User prompt here"));
}

@Test
void invoke_emitsTextDeltaOnFutureCompletion() {
    DirectCallBridge bridge = new DirectCallBridge();
    OpenClawHookClient hookClient = mock(OpenClawHookClient.class);
    OpenClawAgentProvider provider = new OpenClawAgentProvider(
            bridge, hookClient, "health-agent", "https://casehub.internal");

    // Capture the correlationId from the delivery URL to complete the future
    doAnswer(inv -> {
        String url = inv.getArgument(4);
        String corrId = url.substring(url.lastIndexOf('/') + 1);
        bridge.complete(corrId, "{\"result\":\"ok\"}");
        return null;
    }).when(hookClient).invokeDirect(anyString(), anyString(),
            isNull(), anyInt(), anyString());

    AgentSessionConfig config = AgentSessionConfig.of("sys", "user", Duration.ofSeconds(5));

    List<AgentEvent> events = provider.invoke(config)
            .collect().asList()
            .await().atMost(Duration.ofSeconds(5));

    assertEquals(1, events.size());
    assertInstanceOf(AgentEvent.TextDelta.class, events.get(0));
    assertEquals("{\"result\":\"ok\"}", ((AgentEvent.TextDelta) events.get(0)).text());
}

@Test
void invoke_invocationException_failsMulti() {
    DirectCallBridge bridge = new DirectCallBridge();
    OpenClawHookClient hookClient = mock(OpenClawHookClient.class);
    doThrow(new OpenClawInvocationException("HTTP 503"))
            .when(hookClient).invokeDirect(anyString(), anyString(),
                    isNull(), anyInt(), anyString());

    OpenClawAgentProvider provider = new OpenClawAgentProvider(
            bridge, hookClient, "health-agent", "https://casehub.internal");
    AgentSessionConfig config = AgentSessionConfig.of("sys", "user", Duration.ofSeconds(5));

    assertThrows(OpenClawInvocationException.class, () ->
            provider.invoke(config).collect().asList()
                    .await().atMost(Duration.ofSeconds(5)));
}

@Test
void openSession_throwsUnsupported() {
    DirectCallBridge bridge = new DirectCallBridge();
    OpenClawHookClient hookClient = mock(OpenClawHookClient.class);
    OpenClawAgentProvider provider = new OpenClawAgentProvider(
            bridge, hookClient, "health-agent", "https://casehub.internal");

    assertThrows(UnsupportedOperationException.class,
            () -> provider.openSession(AgentSessionInit.of("sys")));
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawAgentProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 4: Implement OpenClawAgentProvider**

```java
package io.casehub.openclaw.casehub;

import io.casehub.openclaw.client.OpenClawInvocationException;
import io.casehub.openclaw.client.OpenClawHookClient;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.platform.agent.AgentSessionInit;
import io.smallrye.mutiny.Multi;

import java.util.UUID;

public class OpenClawAgentProvider implements AgentProvider {

    private final DirectCallBridge bridge;
    private final OpenClawHookClient hookClient;
    private final String agentId;
    private final String deliveryBaseUrl;

    public OpenClawAgentProvider(DirectCallBridge bridge,
                                  OpenClawHookClient hookClient,
                                  String agentId,
                                  String deliveryBaseUrl) {
        this.bridge = bridge;
        this.hookClient = hookClient;
        this.agentId = agentId;
        this.deliveryBaseUrl = deliveryBaseUrl;
    }

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig config) {
        return Multi.createFrom().emitter(emitter -> {
            String correlationId = config.correlationId() != null
                    ? config.correlationId() : UUID.randomUUID().toString();
            var future = bridge.submit(correlationId);
            String deliveryUrl = deliveryBaseUrl
                    + "/openclaw/direct-call/" + correlationId;

            String message = config.systemPrompt() + "\n\n" + config.userPrompt();

            emitter.onTermination(() -> bridge.cancel(correlationId));

            try {
                int timeout = config.timeout() != null
                        ? (int) config.timeout().toSeconds() : 120;
                hookClient.invokeDirect(agentId, message, null, timeout, deliveryUrl);
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

    @Override
    public AgentSession openSession(AgentSessionInit init) {
        throw new UnsupportedOperationException(
                "OpenClaw direct-call is single-shot — use invoke()");
    }
}
```

- [ ] **Step 5: Run OpenClawAgentProvider tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawAgentProviderTest --batch-mode`
Expected: All PASS

- [ ] **Step 6: Write OpenClawChatModel tests**

```java
@Test
void doChat_extractsSystemPromptAndUserText() {
    // Mock provider that captures the AgentSessionConfig
    AtomicReference<AgentSessionConfig> captured = new AtomicReference<>();
    AgentProvider provider = config -> {
        captured.set(config);
        return Multi.createFrom().item(new AgentEvent.TextDelta("{\"ok\":true}"));
    };

    OpenClawChatModel chatModel = new OpenClawChatModel(provider, Duration.ofSeconds(5));
    ChatRequest request = ChatRequest.builder()
            .messages(List.of(
                    SystemMessage.from("You are a health agent"),
                    UserMessage.from("Book appointment")))
            .build();

    ChatResponse response = chatModel.doChat(request);

    assertEquals("You are a health agent", captured.get().systemPrompt());
    assertTrue(captured.get().userPrompt().contains("Book appointment"));
    assertEquals("{\"ok\":true}", response.aiMessage().text());
}

@Test
void doChat_extractsJsonSchemaIntoUserPrompt() {
    AtomicReference<AgentSessionConfig> captured = new AtomicReference<>();
    AgentProvider provider = config -> {
        captured.set(config);
        return Multi.createFrom().item(
                new AgentEvent.TextDelta("{\"appointmentId\":\"A1\",\"confirmed\":true}"));
    };

    JsonObjectSchema schema = JsonObjectSchema.builder()
            .addStringProperty("appointmentId")
            .addBooleanProperty("confirmed")
            .required("appointmentId", "confirmed")
            .build();
    ResponseFormat format = ResponseFormat.builder()
            .type(ResponseFormatType.JSON)
            .jsonSchema(JsonSchema.builder()
                    .name("BookingResult")
                    .rootElement(schema)
                    .build())
            .build();

    OpenClawChatModel chatModel = new OpenClawChatModel(provider, Duration.ofSeconds(5));
    ChatRequest request = ChatRequest.builder()
            .messages(List.of(
                    SystemMessage.from("sys"),
                    UserMessage.from("book it")))
            .responseFormat(format)
            .build();

    chatModel.doChat(request);

    String userPrompt = captured.get().userPrompt();
    assertTrue(userPrompt.contains("BookingResult"),
            "Schema name should appear in user prompt");
    assertTrue(userPrompt.contains("appointmentId"),
            "Schema fields should appear in user prompt");
    assertTrue(userPrompt.contains("book it"),
            "Original user text should appear after schema");
}

@Test
void doChat_invalidJson_throwsAgentException() {
    AgentProvider provider = config ->
            Multi.createFrom().item(new AgentEvent.TextDelta("not valid json"));

    OpenClawChatModel chatModel = new OpenClawChatModel(provider, Duration.ofSeconds(5));
    ChatRequest request = ChatRequest.builder()
            .messages(List.of(SystemMessage.from("sys"), UserMessage.from("go")))
            .build();

    assertThrows(AgentException.class, () -> chatModel.doChat(request));
}

@Test
void doChat_noSystemMessage_usesEmptyString() {
    AtomicReference<AgentSessionConfig> captured = new AtomicReference<>();
    AgentProvider provider = config -> {
        captured.set(config);
        return Multi.createFrom().item(new AgentEvent.TextDelta("{\"ok\":true}"));
    };

    OpenClawChatModel chatModel = new OpenClawChatModel(provider, Duration.ofSeconds(5));
    ChatRequest request = ChatRequest.builder()
            .messages(List.of(UserMessage.from("just user text")))
            .build();

    chatModel.doChat(request);

    assertEquals("", captured.get().systemPrompt());
}
```

- [ ] **Step 7: Implement OpenClawChatModel**

```java
package io.casehub.openclaw.casehub;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.data.message.ChatMessage;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.request.ResponseFormat;
import dev.langchain4j.model.chat.request.json.JsonObjectSchema;
import dev.langchain4j.model.chat.request.json.JsonSchema;
import dev.langchain4j.model.chat.request.json.JsonSchemaElement;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.api.model.ai.AgentException;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;

import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class OpenClawChatModel implements ChatModel {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final AgentProvider agentProvider;
    private final Duration timeout;

    public OpenClawChatModel(AgentProvider agentProvider, Duration timeout) {
        this.agentProvider = agentProvider;
        this.timeout = timeout;
    }

    @Override
    public ChatResponse doChat(ChatRequest request) {
        String systemPrompt = extractSystemPrompt(request.messages());
        String userText = extractLastUserText(request.messages());
        String userPromptWithSchema = prependSchema(request, userText);

        AgentSessionConfig config = new AgentSessionConfig(
                systemPrompt, userPromptWithSchema, List.of(), timeout, null);

        String responseText = agentProvider.invoke(config)
                .filter(AgentEvent.TextDelta.class::isInstance)
                .map(e -> ((AgentEvent.TextDelta) e).text())
                .collect().asList()
                .await().atMost(timeout)
                .stream()
                .collect(Collectors.joining());

        validateJson(responseText);

        return ChatResponse.builder()
                .aiMessage(new AiMessage(responseText))
                .build();
    }

    private static String extractSystemPrompt(List<ChatMessage> messages) {
        return messages.stream()
                .filter(SystemMessage.class::isInstance)
                .map(m -> ((SystemMessage) m).text())
                .findFirst()
                .orElse("");
    }

    private static String extractLastUserText(List<ChatMessage> messages) {
        return messages.stream()
                .filter(UserMessage.class::isInstance)
                .map(m -> ((UserMessage) m).singleText())
                .reduce((first, second) -> second)
                .orElse("");
    }

    private static String prependSchema(ChatRequest request, String userText) {
        ResponseFormat format = request.responseFormat();
        if (format == null || format.jsonSchema() == null) {
            return userText;
        }
        JsonSchema schema = format.jsonSchema();
        String schemaBlock = serializeSchema(schema);
        return schemaBlock + "\n\n" + userText;
    }

    static String serializeSchema(JsonSchema schema) {
        StringBuilder sb = new StringBuilder();
        sb.append("Respond with JSON matching schema \"")
                .append(schema.name()).append("\":\n{\n");
        JsonSchemaElement root = schema.rootElement();
        if (root instanceof JsonObjectSchema obj) {
            Map<String, JsonSchemaElement> props = obj.properties();
            List<String> required = obj.required() != null ? obj.required() : List.of();
            props.forEach((name, element) -> {
                String typeName = element.getClass().getSimpleName()
                        .replace("Json", "").replace("Schema", "").toLowerCase();
                String reqLabel = required.contains(name) ? " (required)" : "";
                sb.append("  \"").append(name).append("\": ")
                        .append(typeName).append(reqLabel).append(",\n");
            });
            if (!props.isEmpty()) {
                sb.setLength(sb.length() - 2);
                sb.append('\n');
            }
        }
        sb.append('}');
        return sb.toString();
    }

    private static void validateJson(String text) {
        if (text == null || text.isBlank()) {
            throw new AgentException("OpenClaw agent returned empty response");
        }
        try {
            MAPPER.readTree(text);
        } catch (Exception e) {
            throw new AgentException(
                    "OpenClaw agent returned invalid JSON: " + text, e);
        }
    }
}
```

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest="OpenClawAgentProviderTest,OpenClawChatModelTest" --batch-mode`
Expected: All PASS

- [ ] **Step 9: Run the full casehub module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub --batch-mode`
Expected: All existing tests + new tests PASS

- [ ] **Step 10: Commit**

```
feat(#49): add OpenClawAgentProvider and OpenClawChatModel

OpenClawAgentProvider implements AgentProvider (platform SPI), following the
ClaudeAgentProvider pattern. OpenClawChatModel is a thin langchain4j bridge
implementing doChat() — extracts system prompt, user text, and JSON schema
from ChatRequest, delegates to AgentProvider.invoke().

Refs casehubio/openclaw#49
```

- [ ] **Step 11: Install SNAPSHOT to local Maven repo**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install --batch-mode`
Expected: BUILD SUCCESS — SNAPSHOT jars available for casehub-life consumption
