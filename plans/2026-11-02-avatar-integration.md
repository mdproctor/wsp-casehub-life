# Avatar Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #143 — feat: Avatar integration — neocortex cognitive agent into life-ui
**Issue group:** #143

**Goal:** Build a conversational avatar surface in casehub-life — POST+SSE
endpoint backed by AgentProvider multi-turn sessions, domain context
injection, MCP tool access, and a Lit conversation panel replacing
life-comms-mock. Phase A only (without CognitionCore — deferred until
neocortex artifacts publish).

**Architecture:** LifeCognitiveAgentService manages per-user AgentSessions
via RoutingAgentProvider (which delegates to casehub-platform-agent-claude
backend). System prompt composed from eidos AgentDescriptor + domain
context summary. Response streamed as SSE events from a POST endpoint.
Frontend Lit panel parses SSE stream and renders message bubbles + action
cards.

**Tech Stack:** Java 21, Quarkus 3.32.2, Lit 3.x, SSE (no WebSocket),
casehub-platform-agent-api, casehub-platform-agent-claude,
casehub-eidos-api

## Global Constraints

- Java 21 on Java 26 JVM
- Quarkus 3.32.2
- All REST resources: `@Blocking @ApplicationScoped`, class-level
  `@Produces(APPLICATION_JSON)` and `@Consumes(APPLICATION_JSON)`
  (except SSE endpoints which use `@Produces(SERVER_SENT_EVENTS)`)
- `@Transactional` on service methods only, never resources
- All commits reference issue: `Refs #143`
- `@RolesAllowed` uses `HouseholdGroups.ADMIN`, `.MEMBER`, `.JUNIOR`
- No new JPA entities or Flyway migrations
- No WebSocket dependencies

---

## Batch 1: API types and domain context

### Task 1: AvatarEvent sealed interface

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/avatar/AvatarEvent.java`
- Test: `api/src/test/java/io/casehub/life/api/avatar/AvatarEventTest.java`

**Interfaces:**
- Consumes: nothing (leaf type)
- Produces: `AvatarEvent` sealed interface with 8 record variants —
  used by LifeCognitiveAgentService and LifeAvatarResource

- [ ] **Step 1: Write the test**

```java
package io.casehub.life.api.avatar;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AvatarEventTest {

    @Test
    void textDeltaCarriesText() {
        var event = new AvatarEvent.TextDelta("hello");
        assertEquals("hello", event.text());
        assertInstanceOf(AvatarEvent.class, event);
    }

    @Test
    void thinkingDeltaCarriesText() {
        var event = new AvatarEvent.ThinkingDelta("reasoning");
        assertEquals("reasoning", event.text());
    }

    @Test
    void toolCallStartedCarriesName() {
        var event = new AvatarEvent.ToolCallStarted("life/tasks/create");
        assertEquals("life/tasks/create", event.toolName());
    }

    @Test
    void toolCallCompletedCarriesArguments() {
        var event = new AvatarEvent.ToolCallCompleted("life/tasks/create", "{\"title\":\"Buy milk\"}");
        assertEquals("life/tasks/create", event.toolName());
        assertEquals("{\"title\":\"Buy milk\"}", event.arguments());
    }

    @Test
    void toolResultCarriesErrorFlag() {
        var event = new AvatarEvent.ToolResult("life/tasks/create", "{\"id\":1}", false);
        assertFalse(event.isError());
        var error = new AvatarEvent.ToolResult("life/tasks/create", "not found", true);
        assertTrue(error.isError());
    }

    @Test
    void actionCardCarriesTypeAndData() {
        var event = new AvatarEvent.ActionCard("task-created", "{\"id\":1,\"title\":\"Buy milk\"}");
        assertEquals("task-created", event.type());
    }

    @Test
    void confirmationRequiredCarriesDetails() {
        var event = new AvatarEvent.ConfirmationRequired("action-1", "Start care-coordination case", "Will create WorkItems with SLA deadlines");
        assertEquals("action-1", event.actionId());
        assertEquals("Start care-coordination case", event.description());
    }

    @Test
    void sessionCompleteCarriesTokenUsage() {
        var event = new AvatarEvent.SessionComplete(1500, 800);
        assertEquals(1500, event.inputTokens());
        assertEquals(800, event.outputTokens());
    }

    @Test
    void sealedInterfacePermitsAllVariants() {
        Class<?>[] permitted = AvatarEvent.class.getPermittedSubclasses();
        assertEquals(8, permitted.length);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AvatarEventTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement AvatarEvent**

```java
package io.casehub.life.api.avatar;

public sealed interface AvatarEvent permits
        AvatarEvent.TextDelta,
        AvatarEvent.ThinkingDelta,
        AvatarEvent.ToolCallStarted,
        AvatarEvent.ToolCallCompleted,
        AvatarEvent.ToolResult,
        AvatarEvent.ActionCard,
        AvatarEvent.ConfirmationRequired,
        AvatarEvent.SessionComplete {

    record TextDelta(String text) implements AvatarEvent {}

    record ThinkingDelta(String text) implements AvatarEvent {}

    record ToolCallStarted(String toolName) implements AvatarEvent {}

    record ToolCallCompleted(String toolName, String arguments) implements AvatarEvent {}

    record ToolResult(String toolName, String result, boolean isError) implements AvatarEvent {}

    record ActionCard(String type, String data) implements AvatarEvent {}

    record ConfirmationRequired(String actionId, String description, String consequences) implements AvatarEvent {}

    record SessionComplete(long inputTokens, long outputTokens) implements AvatarEvent {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AvatarEventTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Install api and commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode -DskipTests
git add api/src/main/java/io/casehub/life/api/avatar/AvatarEvent.java api/src/test/java/io/casehub/life/api/avatar/AvatarEventTest.java
git commit -m "feat(#143): add AvatarEvent sealed interface in api

8 event variants mapping from platform AgentEvent to life-specific
avatar events: TextDelta, ThinkingDelta, ToolCallStarted,
ToolCallCompleted, ToolResult, ActionCard, ConfirmationRequired,
SessionComplete.

Refs #143"
```

### Task 2: LifeDomainContextProvider

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/LifeDomainContextProvider.java`
- Test: `app/src/test/java/io/casehub/life/app/service/LifeDomainContextProviderTest.java`

**Interfaces:**
- Consumes: `DashboardService.buildBriefing()` → `BriefingResponse`,
  `PendingActionsService.findPendingActions(...)` → `PagedResponse<PendingActionResponse>`,
  `LifeCaseQueryService` (active case count)
- Produces: `String buildContextSummary(String tenantId)` — structured
  text for system prompt injection. `void invalidate()` — dirty flag.
  `boolean isDirty()`.

- [ ] **Step 1: Write the test**

```java
package io.casehub.life.app.service;

import io.casehub.life.api.PagedResponse;
import io.casehub.life.api.PendingActionResponse;
import io.casehub.life.api.Urgency;
import io.casehub.life.app.service.DashboardService;
import io.casehub.life.app.service.DashboardService.BriefingResponse;
import io.casehub.life.app.service.DashboardService.BriefingItem;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class LifeDomainContextProviderTest {

    private DashboardService dashboardService;
    private LifeDomainContextProvider provider;

    @BeforeEach
    void setUp() {
        dashboardService = mock(DashboardService.class);
        provider = new LifeDomainContextProvider(dashboardService);
    }

    @Test
    void buildContextSummaryIncludesBriefingItems() {
        when(dashboardService.buildBriefing()).thenReturn(
            new BriefingResponse("Good morning", 2, List.of(
                new BriefingItem("GP appointment overdue", "HEALTH", "health-coordination"),
                new BriefingItem("Grocery delivery due today", "HOUSEHOLD", "household-management")
            ))
        );

        String summary = provider.buildContextSummary();
        assertTrue(summary.contains("GP appointment overdue"));
        assertTrue(summary.contains("Grocery delivery due today"));
        assertTrue(summary.contains("2"));
    }

    @Test
    void invalidateSetsFlag() {
        assertFalse(provider.isDirty());
        provider.invalidate();
        assertTrue(provider.isDirty());
    }

    @Test
    void buildContextSummaryClearsDirtyFlag() {
        when(dashboardService.buildBriefing()).thenReturn(
            new BriefingResponse("Hello", 0, List.of()));

        provider.invalidate();
        assertTrue(provider.isDirty());
        provider.buildContextSummary();
        assertFalse(provider.isDirty());
    }

    @Test
    void summaryIsCapped() {
        when(dashboardService.buildBriefing()).thenReturn(
            new BriefingResponse("Hello", 0, List.of()));

        String summary = provider.buildContextSummary();
        assertTrue(summary.length() < 8000);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeDomainContextProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement LifeDomainContextProvider**

```java
package io.casehub.life.app.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.concurrent.atomic.AtomicBoolean;

@ApplicationScoped
public class LifeDomainContextProvider {

    private final DashboardService dashboardService;
    private final AtomicBoolean dirty = new AtomicBoolean(false);

    @Inject
    public LifeDomainContextProvider(DashboardService dashboardService) {
        this.dashboardService = dashboardService;
    }

    public String buildContextSummary() {
        dirty.set(false);

        var briefing = dashboardService.buildBriefing();
        var sb = new StringBuilder();
        sb.append("## Current Life State\n\n");
        sb.append("Active items: ").append(briefing.itemCount()).append("\n\n");

        if (!briefing.items().isEmpty()) {
            sb.append("### Attention Items\n");
            var items = briefing.items();
            int limit = Math.min(items.size(), 10);
            for (int i = 0; i < limit; i++) {
                var item = items.get(i);
                sb.append("- [").append(item.domain()).append("] ").append(item.description()).append("\n");
            }
            if (items.size() > limit) {
                sb.append("- ... and ").append(items.size() - limit).append(" more (use tools to query)\n");
            }
        }

        return sb.toString();
    }

    public void invalidate() {
        dirty.set(true);
    }

    public boolean isDirty() {
        return dirty.get();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeDomainContextProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/service/LifeDomainContextProvider.java app/src/test/java/io/casehub/life/app/service/LifeDomainContextProviderTest.java
git commit -m "feat(#143): add LifeDomainContextProvider for avatar system prompt

Assembles compact life state summary from DashboardService briefing.
Dirty flag for SSE-event-driven invalidation. Capped output to stay
within prompt token budget.

Refs #143"
```

---

## Batch 2: Agent backend wiring and avatar service

### Task 3: Maven dependencies and agent backend configuration

**Files:**
- Modify: `app/pom.xml` — add casehub-platform-agent-claude dependency
- Modify: `app/src/main/resources/application.properties` — add avatar config
- Modify: `app/src/test/resources/application.properties` — add test avatar config

**Interfaces:**
- Consumes: nothing
- Produces: `RoutingAgentProvider` CDI bean with Claude backend available.
  Config properties: `casehub.platform.agent.claude.*`,
  `casehub.life.avatar.*`

- [ ] **Step 1: Add Maven dependency**

Add to `app/pom.xml` `<dependencies>` section:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-claude</artifactId>
</dependency>
```

Verify it resolves: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn dependency:resolve -pl app --batch-mode`

Check if `casehub-eidos-api` is already transitive. If not, add it too.
Check if `casehub-platform-agent-api` is already transitive. If not, add.

- [ ] **Step 2: Add configuration**

Add to `app/src/main/resources/application.properties`:
```properties
# Avatar agent backend
casehub.life.avatar.name=Life
casehub.life.avatar.model=claude-sonnet-4-5
casehub.life.avatar.session.idle-timeout=PT30M
```

Add to `app/src/test/resources/application.properties`:
```properties
casehub.life.avatar.name=Life
casehub.life.avatar.model=test
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add app/pom.xml app/src/main/resources/application.properties app/src/test/resources/application.properties
git commit -m "feat(#143): add agent-claude dependency and avatar config

Wires casehub-platform-agent-claude for AgentProvider.openSession()
support. Avatar configuration: name, model, session idle timeout.

Refs #143"
```

### Task 4: LifeAvatarDescriptorProvider

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/LifeAvatarDescriptorProvider.java`
- Test: `app/src/test/java/io/casehub/life/app/engine/agent/LifeAvatarDescriptorProviderTest.java`

**Interfaces:**
- Consumes: `casehub.life.avatar.name` config property,
  `casehub.life.tenancy-id` config property
- Produces: `AgentDescriptor getDescriptor()` — full avatar identity
  for system prompt assembly

- [ ] **Step 1: Write the test**

```java
package io.casehub.life.app.engine.agent;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class LifeAvatarDescriptorProviderTest {

    @Test
    void descriptorHasAvatarIdentity() {
        var provider = new LifeAvatarDescriptorProvider(
            "Life", "278776f9-e1b0-46fb-9032-8bddebdcf9ce", "GB");

        var descriptor = provider.getDescriptor();
        assertEquals("Life", descriptor.name());
        assertNotNull(descriptor.agentId());
        assertTrue(descriptor.agentId().contains("avatar"));
    }

    @Test
    void briefingContainsPersonalityAndConstraints() {
        var provider = new LifeAvatarDescriptorProvider(
            "Life", "278776f9-e1b0-46fb-9032-8bddebdcf9ce", "GB");

        var descriptor = provider.getDescriptor();
        String briefing = descriptor.briefing();
        assertNotNull(briefing);
        assertTrue(briefing.contains("household"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeAvatarDescriptorProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL

- [ ] **Step 3: Implement LifeAvatarDescriptorProvider**

```java
package io.casehub.life.app.engine.agent;

import io.casehub.eidos.api.AgentDescriptor;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class LifeAvatarDescriptorProvider {

    private final String name;
    private final String tenancyId;
    private final String jurisdiction;

    @Inject
    public LifeAvatarDescriptorProvider(
            @ConfigProperty(name = "casehub.life.avatar.name", defaultValue = "Life") String name,
            @ConfigProperty(name = "casehub.life.tenancy-id") String tenancyId,
            @ConfigProperty(name = "casehub.life.jurisdiction", defaultValue = "GB") String jurisdiction) {
        this.name = name;
        this.tenancyId = tenancyId;
        this.jurisdiction = jurisdiction;
    }

    public AgentDescriptor getDescriptor() {
        return AgentDescriptor.builder()
                .agentId("life:avatar@1")
                .name(name)
                .version("1.0.0")
                .provider("casehub-life")
                .modelFamily("anthropic")
                .slot("avatar")
                .jurisdiction(jurisdiction)
                .tenancyId(tenancyId)
                .briefing(buildBriefing())
                .build();
    }

    private String buildBriefing() {
        return """
            You are the household's digital life assistant. You help manage \
            daily life: tasks, appointments, contractors, finances, and family \
            coordination. You have access to MCP tools for creating tasks, \
            starting cases, looking up people, and checking the calendar.

            Rules:
            - Always confirm before starting a case, creating a financial \
              commitment, or approving an oversight gate
            - Direct lookups, status queries, and low-risk task creation can \
              proceed without confirmation
            - If the user lacks permission for an action, explain what role \
              is needed
            - Be proactive: if you notice overdue items or upcoming deadlines \
              in the context, mention them
            - Be concise but warm — this is a household assistant, not a \
              corporate chatbot
            """;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeAvatarDescriptorProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/engine/agent/LifeAvatarDescriptorProvider.java app/src/test/java/io/casehub/life/app/engine/agent/LifeAvatarDescriptorProviderTest.java
git commit -m "feat(#143): add LifeAvatarDescriptorProvider for avatar identity

Eidos AgentDescriptor with life avatar personality, constraints, and
household management briefing. Config-driven name and tenancy.

Refs #143"
```

### Task 5: LifeCognitiveAgentService

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/LifeCognitiveAgentService.java`
- Create: `app/src/main/java/io/casehub/life/app/service/AvatarEventMapper.java`
- Test: `app/src/test/java/io/casehub/life/app/service/LifeCognitiveAgentServiceTest.java`
- Test: `app/src/test/java/io/casehub/life/app/service/AvatarEventMapperTest.java`

**Interfaces:**
- Consumes: `AgentProvider.openSession(AgentSessionInit)` → `AgentSession`,
  `AgentSession.query(String)` → `Multi<AgentEvent>`,
  `LifeAvatarDescriptorProvider.getDescriptor()` → `AgentDescriptor`,
  `LifeDomainContextProvider.buildContextSummary()` → `String`
- Produces: `Multi<AvatarEvent> sendMessage(String principalId, String message)`,
  `void closeSession(String principalId)`

- [ ] **Step 1: Write AvatarEventMapper test**

```java
package io.casehub.life.app.service;

import io.casehub.life.api.avatar.AvatarEvent;
import io.casehub.platform.agent.AgentEvent;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AvatarEventMapperTest {

    @Test
    void mapsTextDelta() {
        var event = new AgentEvent.TextDelta("hello");
        var result = AvatarEventMapper.map(event);
        assertInstanceOf(AvatarEvent.TextDelta.class, result);
        assertEquals("hello", ((AvatarEvent.TextDelta) result).text());
    }

    @Test
    void mapsThinkingDelta() {
        var event = new AgentEvent.ThinkingDelta("reasoning about...");
        var result = AvatarEventMapper.map(event);
        assertInstanceOf(AvatarEvent.ThinkingDelta.class, result);
    }

    @Test
    void mapsToolCallComplete() {
        var event = new AgentEvent.ToolCallComplete(0, "call-1", "life/tasks/create", "{\"title\":\"Buy milk\"}");
        var result = AvatarEventMapper.map(event);
        assertInstanceOf(AvatarEvent.ToolCallCompleted.class, result);
        assertEquals("life/tasks/create", ((AvatarEvent.ToolCallCompleted) result).toolName());
    }

    @Test
    void mapsToolResult() {
        var event = new AgentEvent.ToolResult("call-1", "{\"id\":1}", false);
        var result = AvatarEventMapper.map(event);
        assertInstanceOf(AvatarEvent.ToolResult.class, result);
        assertFalse(((AvatarEvent.ToolResult) result).isError());
    }

    @Test
    void mapsToolCallDeltaToStarted() {
        var event = new AgentEvent.ToolCallDelta(0, "call-1", "life/tasks/create", "{\"ti");
        var result = AvatarEventMapper.map(event);
        assertInstanceOf(AvatarEvent.ToolCallStarted.class, result);
        assertEquals("life/tasks/create", ((AvatarEvent.ToolCallStarted) result).toolName());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=AvatarEventMapperTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL

- [ ] **Step 3: Implement AvatarEventMapper**

```java
package io.casehub.life.app.service;

import io.casehub.life.api.avatar.AvatarEvent;
import io.casehub.platform.agent.AgentEvent;

public final class AvatarEventMapper {

    private AvatarEventMapper() {}

    public static AvatarEvent map(AgentEvent event) {
        return switch (event) {
            case AgentEvent.TextDelta td -> new AvatarEvent.TextDelta(td.text());
            case AgentEvent.ThinkingDelta td -> new AvatarEvent.ThinkingDelta(td.text());
            case AgentEvent.ToolCallDelta tcd -> new AvatarEvent.ToolCallStarted(tcd.name());
            case AgentEvent.ToolCallComplete tcc -> new AvatarEvent.ToolCallCompleted(tcc.name(), tcc.arguments());
            case AgentEvent.ToolResult tr -> new AvatarEvent.ToolResult(tr.toolCallId(), tr.content(), tr.isError());
            case AgentEvent.InvocationComplete ic -> new AvatarEvent.SessionComplete(
                    ic.usage() != null ? ic.usage().inputTokens() : 0,
                    ic.usage() != null ? ic.usage().outputTokens() : 0);
        };
    }
}
```

Note: the exact field names on `InvocationComplete.usage()` need to be
verified against the platform API. Adjust `inputTokens()`/`outputTokens()`
accessor names to match the actual `AgentEvent.InvocationComplete` record
fields.

- [ ] **Step 4: Run mapper test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=AvatarEventMapperTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Write LifeCognitiveAgentService test**

```java
package io.casehub.life.app.service;

import io.casehub.life.api.avatar.AvatarEvent;
import io.casehub.life.app.engine.agent.LifeAvatarDescriptorProvider;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionInit;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class LifeCognitiveAgentServiceTest {

    private AgentProvider agentProvider;
    private AgentSession agentSession;
    private LifeAvatarDescriptorProvider descriptorProvider;
    private LifeDomainContextProvider contextProvider;
    private LifeCognitiveAgentService service;

    @BeforeEach
    void setUp() {
        agentProvider = mock(AgentProvider.class);
        agentSession = mock(AgentSession.class);
        descriptorProvider = new LifeAvatarDescriptorProvider(
            "Life", "278776f9-e1b0-46fb-9032-8bddebdcf9ce", "GB");
        contextProvider = mock(LifeDomainContextProvider.class);

        when(agentProvider.openSession(any(AgentSessionInit.class))).thenReturn(agentSession);
        when(contextProvider.buildContextSummary()).thenReturn("## Current Life State\nActive items: 2");
        when(contextProvider.isDirty()).thenReturn(false);

        service = new LifeCognitiveAgentService(agentProvider, descriptorProvider, contextProvider, "claude-sonnet-4-5");
    }

    @Test
    void sendMessageStreamsAvatarEvents() {
        when(agentSession.query(anyString())).thenReturn(
            Multi.createFrom().items(
                new AgentEvent.TextDelta("Hello! "),
                new AgentEvent.TextDelta("How can I help?")
            )
        );

        List<AvatarEvent> events = service.sendMessage("user-1", "Hi there")
            .collect().asList()
            .await().atMost(Duration.ofSeconds(5));

        assertEquals(2, events.size());
        assertInstanceOf(AvatarEvent.TextDelta.class, events.get(0));
        assertEquals("Hello! ", ((AvatarEvent.TextDelta) events.get(0)).text());
    }

    @Test
    void sendMessageCreatesSessionOnFirstCall() {
        when(agentSession.query(anyString())).thenReturn(
            Multi.createFrom().items(new AgentEvent.TextDelta("Hi")));

        service.sendMessage("user-1", "Hello").collect().asList()
            .await().atMost(Duration.ofSeconds(5));

        verify(agentProvider).openSession(any(AgentSessionInit.class));
    }

    @Test
    void sendMessageReusesExistingSession() {
        when(agentSession.query(anyString())).thenReturn(
            Multi.createFrom().items(new AgentEvent.TextDelta("Hi")));

        service.sendMessage("user-1", "Hello").collect().asList()
            .await().atMost(Duration.ofSeconds(5));
        service.sendMessage("user-1", "What's next?").collect().asList()
            .await().atMost(Duration.ofSeconds(5));

        verify(agentProvider, times(1)).openSession(any(AgentSessionInit.class));
    }

    @Test
    void closeSessionRemovesSession() {
        when(agentSession.query(anyString())).thenReturn(
            Multi.createFrom().items(new AgentEvent.TextDelta("Hi")));

        service.sendMessage("user-1", "Hello").collect().asList()
            .await().atMost(Duration.ofSeconds(5));
        service.closeSession("user-1");

        verify(agentSession).close(any(Duration.class));
    }
}
```

- [ ] **Step 6: Implement LifeCognitiveAgentService**

```java
package io.casehub.life.app.service;

import io.casehub.life.api.avatar.AvatarEvent;
import io.casehub.life.app.engine.agent.LifeAvatarDescriptorProvider;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionInit;
import io.smallrye.mutiny.Multi;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.time.Duration;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class LifeCognitiveAgentService {

    private final AgentProvider agentProvider;
    private final LifeAvatarDescriptorProvider descriptorProvider;
    private final LifeDomainContextProvider contextProvider;
    private final String model;
    private final ConcurrentHashMap<String, AgentSession> sessions = new ConcurrentHashMap<>();

    @Inject
    public LifeCognitiveAgentService(
            AgentProvider agentProvider,
            LifeAvatarDescriptorProvider descriptorProvider,
            LifeDomainContextProvider contextProvider,
            @ConfigProperty(name = "casehub.life.avatar.model", defaultValue = "claude-sonnet-4-5") String model) {
        this.agentProvider = agentProvider;
        this.descriptorProvider = descriptorProvider;
        this.contextProvider = contextProvider;
        this.model = model;
    }

    public Multi<AvatarEvent> sendMessage(String principalId, String userMessage) {
        var session = sessions.computeIfAbsent(principalId, this::createSession);
        return session.query(userMessage).map(AvatarEventMapper::map);
    }

    public void closeSession(String principalId) {
        var session = sessions.remove(principalId);
        if (session != null) {
            session.close(Duration.ofSeconds(5));
        }
    }

    private AgentSession createSession(String principalId) {
        var descriptor = descriptorProvider.getDescriptor();
        var context = contextProvider.buildContextSummary();
        var systemPrompt = descriptor.briefing() + "\n\n" + context;

        var init = AgentSessionInit.of(systemPrompt)
                .withModel(model);
        return agentProvider.openSession(init);
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeCognitiveAgentServiceTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/service/LifeCognitiveAgentService.java app/src/main/java/io/casehub/life/app/service/AvatarEventMapper.java app/src/test/java/io/casehub/life/app/service/LifeCognitiveAgentServiceTest.java app/src/test/java/io/casehub/life/app/service/AvatarEventMapperTest.java
git commit -m "feat(#143): add LifeCognitiveAgentService and AvatarEventMapper

Core avatar service managing per-user AgentSessions via
RoutingAgentProvider. Maps platform AgentEvent to AvatarEvent.
System prompt assembled from eidos descriptor + domain context.

Refs #143"
```

---

## Batch 3: REST endpoint

### Task 6: LifeAvatarResource

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/resource/LifeAvatarResource.java`
- Test: `app/src/test/java/io/casehub/life/app/resource/LifeAvatarResourceTest.java`

**Interfaces:**
- Consumes: `LifeCognitiveAgentService.sendMessage(principalId, message)` →
  `Multi<AvatarEvent>`, `CurrentPrincipal.actorId()` → `String`
- Produces: `POST /avatar/message` (SSE stream), `DELETE /avatar/session`

- [ ] **Step 1: Write the test**

```java
package io.casehub.life.app.resource;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;

@QuarkusTest
class LifeAvatarResourceTest {

    @Test
    @TestSecurity(user = "test-user", roles = "household-member")
    void postMessageReturns200WithSseContentType() {
        given()
            .contentType("text/plain")
            .body("Hello, what tasks do I have today?")
        .when()
            .post("/avatar/message")
        .then()
            .statusCode(200)
            .contentType("text/event-stream");
    }

    @Test
    void postMessageWithoutAuthReturns401() {
        given()
            .contentType("text/plain")
            .body("Hello")
        .when()
            .post("/avatar/message")
        .then()
            .statusCode(401);
    }

    @Test
    @TestSecurity(user = "test-admin", roles = "household-admin")
    void adminCanAccessAvatar() {
        given()
            .contentType("text/plain")
            .body("Hello")
        .when()
            .post("/avatar/message")
        .then()
            .statusCode(200);
    }

    @Test
    @TestSecurity(user = "test-junior", roles = "household-junior")
    void juniorCanAccessAvatar() {
        given()
            .contentType("text/plain")
            .body("Hello")
        .when()
            .post("/avatar/message")
        .then()
            .statusCode(200);
    }

    @Test
    @TestSecurity(user = "test-user", roles = "household-member")
    void deleteSessionReturns204() {
        given()
        .when()
            .delete("/avatar/session")
        .then()
            .statusCode(204);
    }
}
```

Note: these tests require a test AgentProvider that returns canned
responses. Wire a `@Alternative @Priority(10)` test AgentProvider
that returns `Multi.createFrom().items(new AgentEvent.TextDelta("Hello!"))`
for any query. Register in test `application.properties`
`quarkus.arc.selected-alternatives`.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeAvatarResourceTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — resource not found

- [ ] **Step 3: Create test AgentProvider alternative**

```java
package io.casehub.life.app.resource;

import io.casehub.platform.agent.*;
import io.smallrye.mutiny.Multi;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

@Alternative
@Priority(10)
@ApplicationScoped
public class TestAvatarAgentProvider implements AgentProvider {

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig config) {
        return Multi.createFrom().items(new AgentEvent.TextDelta("Test response"));
    }

    @Override
    public AgentSession openSession(AgentSessionInit init) {
        return new AgentSession() {
            @Override
            public Multi<AgentEvent> query(String prompt) {
                return Multi.createFrom().items(new AgentEvent.TextDelta("Hello from avatar!"));
            }

            @Override
            public void close(java.time.Duration drain) {}

            @Override
            public void interrupt() {}
        };
    }
}
```

Add to test `application.properties`:
```properties
quarkus.arc.selected-alternatives=\
  ...,\
  io.casehub.life.app.resource.TestAvatarAgentProvider
```

- [ ] **Step 4: Implement LifeAvatarResource**

```java
package io.casehub.life.app.resource;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.avatar.AvatarEvent;
import io.casehub.life.app.service.LifeCognitiveAgentService;
import io.casehub.platform.identity.CurrentPrincipal;
import io.smallrye.mutiny.Multi;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import org.jboss.resteasy.reactive.RestStreamElementType;

@ApplicationScoped
@Path("/avatar")
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
public class LifeAvatarResource {

    @Inject
    LifeCognitiveAgentService agentService;

    @Inject
    CurrentPrincipal currentPrincipal;

    @POST
    @Path("/message")
    @Consumes(MediaType.TEXT_PLAIN)
    @Produces(MediaType.SERVER_SENT_EVENTS)
    @RestStreamElementType(MediaType.APPLICATION_JSON)
    public Multi<AvatarEvent> sendMessage(String message) {
        return agentService.sendMessage(currentPrincipal.actorId(), message);
    }

    @DELETE
    @Path("/session")
    public void closeSession() {
        agentService.closeSession(currentPrincipal.actorId());
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeAvatarResourceTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/resource/LifeAvatarResource.java app/src/test/java/io/casehub/life/app/resource/LifeAvatarResourceTest.java app/src/test/java/io/casehub/life/app/resource/TestAvatarAgentProvider.java
git commit -m "feat(#143): add LifeAvatarResource — POST+SSE conversation endpoint

POST /avatar/message accepts text, returns SSE stream of AvatarEvent.
DELETE /avatar/session closes the current session. All three household
roles admitted. Test AgentProvider alternative for canned responses.

Refs #143"
```

---

## Batch 4: Frontend conversation panel

### Task 7: life-conversation-panel Lit component

**Files:**
- Create: `life-ui/src/panels/conversation-panel.ts`
- Delete: `life-ui/src/panels/comms-mock.ts` (use `ide_refactor_safe_delete`)
- Modify: `life-ui/src/panels/register.ts` — swap import

**Interfaces:**
- Consumes: `POST /avatar/message` (text body), SSE response stream,
  `DELETE /avatar/session`
- Produces: `<life-conversation-panel>` custom element

- [ ] **Step 1: Create conversation-panel.ts**

```typescript
import { LitElement, html, css, PropertyValues } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import { unsafeHTML } from 'lit/directives/unsafe-html.js';
import { marked } from 'marked';
import DOMPurify from 'dompurify';

interface ChatMessage {
  role: 'user' | 'avatar';
  content: string;
  timestamp: Date;
  thinking?: boolean;
  toolCalls?: Array<{ name: string; result?: string; isError?: boolean }>;
}

@customElement('life-conversation-panel')
export class LifeConversationPanel extends LitElement {

  @state() private messages: ChatMessage[] = [];
  @state() private inputText = '';
  @state() private isStreaming = false;
  @state() private isThinking = false;
  @state() private expanded = false;

  static styles = css`
    :host {
      display: flex;
      flex-direction: column;
      height: 100%;
      font-family: var(--font-family, system-ui);
      background: var(--surface-1, #fff);
      color: var(--text-1, #1a1a1a);
    }

    .header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 8px 12px;
      border-bottom: 1px solid var(--border-1, #e0e0e0);
      font-weight: 600;
      font-size: 14px;
    }

    .expand-btn {
      background: none;
      border: none;
      cursor: pointer;
      font-size: 16px;
      color: var(--text-2, #666);
    }

    .messages {
      flex: 1;
      overflow-y: auto;
      padding: 8px;
      display: flex;
      flex-direction: column;
      gap: 8px;
    }

    .message {
      max-width: 85%;
      padding: 8px 12px;
      border-radius: 12px;
      font-size: 13px;
      line-height: 1.4;
      word-wrap: break-word;
    }

    .message.user {
      align-self: flex-end;
      background: var(--accent-1, #0066cc);
      color: white;
      border-bottom-right-radius: 4px;
    }

    .message.avatar {
      align-self: flex-start;
      background: var(--surface-2, #f0f0f0);
      border-bottom-left-radius: 4px;
    }

    .message.avatar :first-child { margin-top: 0; }
    .message.avatar :last-child { margin-bottom: 0; }
    .message.avatar p { margin: 4px 0; }

    .thinking {
      align-self: flex-start;
      padding: 8px 12px;
      font-size: 12px;
      color: var(--text-3, #999);
      font-style: italic;
    }

    .thinking::after {
      content: '';
      animation: dots 1.5s infinite;
    }

    @keyframes dots {
      0%, 20% { content: '.'; }
      40% { content: '..'; }
      60%, 100% { content: '...'; }
    }

    .tool-call {
      align-self: flex-start;
      padding: 6px 10px;
      font-size: 11px;
      background: var(--surface-3, #e8e8e8);
      border-radius: 6px;
      color: var(--text-2, #666);
      font-family: monospace;
    }

    .tool-call.error { color: var(--error, #cc0000); }

    .input-area {
      display: flex;
      padding: 8px;
      border-top: 1px solid var(--border-1, #e0e0e0);
      gap: 8px;
    }

    .input-area input {
      flex: 1;
      padding: 8px 12px;
      border: 1px solid var(--border-1, #e0e0e0);
      border-radius: 20px;
      font-size: 13px;
      outline: none;
      background: var(--surface-1, #fff);
      color: var(--text-1, #1a1a1a);
    }

    .input-area input:focus {
      border-color: var(--accent-1, #0066cc);
    }

    .input-area button {
      padding: 8px 16px;
      border: none;
      border-radius: 20px;
      background: var(--accent-1, #0066cc);
      color: white;
      cursor: pointer;
      font-size: 13px;
    }

    .input-area button:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
  `;

  render() {
    return html`
      <div class="header">
        <span>Avatar</span>
        <button class="expand-btn" @click=${this._toggleExpand}>
          ${this.expanded ? '▼' : '▲'}
        </button>
      </div>
      <div class="messages">
        ${this.messages.map(msg => this._renderMessage(msg))}
        ${this.isThinking ? html`<div class="thinking">Thinking</div>` : ''}
      </div>
      <div class="input-area">
        <input
          type="text"
          placeholder="Ask your assistant..."
          .value=${this.inputText}
          @input=${(e: InputEvent) => this.inputText = (e.target as HTMLInputElement).value}
          @keydown=${(e: KeyboardEvent) => { if (e.key === 'Enter' && !this.isStreaming) this._send(); }}
          ?disabled=${this.isStreaming}
        />
        <button @click=${this._send} ?disabled=${this.isStreaming || !this.inputText.trim()}>
          Send
        </button>
      </div>
    `;
  }

  private _renderMessage(msg: ChatMessage) {
    if (msg.role === 'user') {
      return html`<div class="message user">${msg.content}</div>`;
    }
    const rendered = DOMPurify.sanitize(marked.parse(msg.content) as string);
    return html`<div class="message avatar">${unsafeHTML(rendered)}</div>`;
  }

  private _toggleExpand() {
    this.expanded = !this.expanded;
    this.dispatchEvent(new CustomEvent('expand-toggle', {
      detail: { expanded: this.expanded },
      bubbles: true, composed: true
    }));
  }

  private async _send() {
    const text = this.inputText.trim();
    if (!text || this.isStreaming) return;

    this.messages = [...this.messages, { role: 'user', content: text, timestamp: new Date() }];
    this.inputText = '';
    this.isStreaming = true;
    this.isThinking = true;

    let avatarContent = '';

    try {
      const response = await fetch('/avatar/message', {
        method: 'POST',
        headers: { 'Content-Type': 'text/plain' },
        body: text,
      });

      if (!response.ok || !response.body) {
        throw new Error(`Avatar request failed: ${response.status}`);
      }

      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let buffer = '';

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop() || '';

        for (const line of lines) {
          if (line.startsWith('data:')) {
            const data = line.slice(5).trim();
            if (!data) continue;
            try {
              const event = JSON.parse(data);
              this._handleEvent(event, () => {
                this.isThinking = false;
                avatarContent += event.text || '';
                this.messages = [
                  ...this.messages.slice(0, -1),
                  { role: 'avatar' as const, content: avatarContent, timestamp: new Date() }
                ];
                if (this.messages[this.messages.length - 1]?.role !== 'avatar') {
                  this.messages = [...this.messages, { role: 'avatar', content: avatarContent, timestamp: new Date() }];
                }
              });
            } catch { /* skip malformed */ }
          }
        }
      }
    } catch (err) {
      avatarContent = 'Sorry, something went wrong. Please try again.';
      this.messages = [...this.messages, { role: 'avatar', content: avatarContent, timestamp: new Date() }];
    } finally {
      this.isStreaming = false;
      this.isThinking = false;
    }
  }

  private _handleEvent(event: Record<string, unknown>, appendText: () => void) {
    const type = Object.keys(event)[0];
    switch (type) {
      case 'text':
        appendText();
        break;
      case 'toolName':
        break;
      case 'isError':
        break;
    }
  }

  protected updated(changedProperties: PropertyValues) {
    super.updated(changedProperties);
    const messagesDiv = this.shadowRoot?.querySelector('.messages');
    if (messagesDiv) {
      messagesDiv.scrollTop = messagesDiv.scrollHeight;
    }
  }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `npm run build --prefix life-ui`
Expected: BUILD SUCCESS

- [ ] **Step 3: Update register.ts — swap comms-mock for conversation-panel**

In `life-ui/src/panels/register.ts`, replace the comms-mock import and
registration with:
```typescript
import './conversation-panel.js';
```

Remove:
```typescript
import './comms-mock.js';
```

- [ ] **Step 4: Commit**

```bash
git add life-ui/src/panels/conversation-panel.ts life-ui/src/panels/register.ts
git rm life-ui/src/panels/comms-mock.ts
git commit -m "feat(#143): add life-conversation-panel replacing comms-mock

Lit 3.x conversation panel with message bubbles, streaming SSE
response rendering, markdown support, thinking indicator, and
expand/collapse toggle. POST+SSE transport to /avatar/message.

Refs #143"
```

### Task 8: Home-view integration

**Files:**
- Modify: `life-ui/src/views/home-view.ts`

**Interfaces:**
- Consumes: `<life-conversation-panel>` custom element from Task 7
- Produces: Avatar panel in right dock replacing Comms

- [ ] **Step 1: Update RIGHT_PANELS in home-view.ts**

In `home-view.ts`, find the `RIGHT_PANELS` array and change the comms
entry:

Replace:
```typescript
{key:'comms', label:'Comms', tag:'life-comms-mock'}
```

With:
```typescript
{key:'avatar', label:'Avatar', tag:'life-conversation-panel'}
```

- [ ] **Step 2: Verify it builds**

Run: `npm run build --prefix life-ui`
Expected: BUILD SUCCESS

- [ ] **Step 3: Manual test in browser**

Run the dev server: `npm run dev --prefix life-ui`
Open http://localhost:5173/
Verify: right dock shows "Avatar" instead of "Comms". Click to toggle
the panel. Input field visible. Send a message (will fail without
backend running — verify the request is made to `/avatar/message`).

- [ ] **Step 4: Commit**

```bash
git add life-ui/src/views/home-view.ts
git commit -m "feat(#143): wire conversation panel into home-view right dock

Replace Comms mock with Avatar panel in RIGHT_PANELS. Toggle and
expand/collapse work via existing dock infrastructure.

Refs #143"
```

---

## References

- [2026-11-02-avatar-integration-design.md] — design spec this plan implements
- [LifeEventSseResource.java] — SSE streaming pattern (Multi<String>, heartbeat)
- [AgentProvider.java] — platform agent SPI (invoke, openSession)
- [AgentSessionInit.java] — session init record (systemPrompt, mcpServers, model)
- [RoutingAgentProvider.java] — routes openSession() to AgentBackend
- [LifeAgentDescriptorFactory.java] — existing descriptor pattern
- [DashboardService.java] — buildBriefing() for domain context
- [life-comms-mock.ts] — placeholder being replaced
- [home-view.ts] — RIGHT_PANELS dock registration
- [register.ts] — panel registration imports
- [LifeEventController.ts] — SSEManager frontend pattern
- [HouseholdGroups.java] — ADMIN/MEMBER/JUNIOR constants
- [GitHub #143] — focal issue
- [GitHub #139] — parent epic (digital life assistant)
