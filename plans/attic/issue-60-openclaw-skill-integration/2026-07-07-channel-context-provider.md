# ChannelContextProvider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #61 — ChannelContextProvider — heartbeat channel context enrichment
**Issue group:** #61 (contributes to #8 but does not close it)

**Goal:** Enrich heartbeat sentinel agents with recent qhorus channel messages
so they can make cross-agent-aware decisions.

**Architecture:** A new `@ApplicationScoped` CDI bean `LifeChannelContextProvider`
queries recent messages from relevant qhorus channels (delegation, oversight,
per-actor) and returns a structured map. `LifeHeartbeatJob` injects the provider,
calls `gatherContext(caseId)` after querying case context, merges the result, and
passes enriched context to the sentinel agent. Channel context gathering is
fault-isolated — failures log a warning and the heartbeat proceeds with case
context only.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-qhorus (MessageStore, ChannelService),
casehub-engine (CaseHubRuntime, PlanItemCallerRef), Panache (WorkItem, LifeTaskContext)

## Global Constraints

- Java source level 21 (running on Java 26 JVM)
- All new CDI beans: `@ApplicationScoped`
- All `@Transactional` on service methods only, never on Quartz jobs
- Channel names use `ext-{uuid}` prefix convention (PP-20260609-982617)
- `callerRef` format: `case:{caseId}/pi:{planItemId}` (PlanItemCallerRef.encode())
- Test templates seeded with `LifeTestFixtures.seedStandardTemplates()` in `@BeforeEach @Transactional`
- Test tenancyId: `"278776f9-e1b0-46fb-9032-8bddebdcf9ce"`
- Use `mvn test -pl app -Dtest=ClassName --batch-mode` with `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Install api first: `mvn install -pl api` before app tests

---

### Task 1: Ledger Import Migration

Mechanical prerequisite. `LedgerAttestation` moved from `io.casehub.ledger.runtime.model`
to `io.casehub.ledger.api.model` in an upstream SNAPSHOT. Two files need the import
path updated.

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriter.java:5`
- Modify: `app/src/test/java/io/casehub/life/app/service/LifeOutcomeAttestationWriterTest.java:5`

**Interfaces:**
- Consumes: nothing
- Produces: compilable codebase (prerequisite for all subsequent tasks)

- [ ] **Step 1: Fix production import**

In `LifeOutcomeAttestationWriter.java`, change line 5:
```java
// OLD:
import io.casehub.ledger.runtime.model.LedgerAttestation;
// NEW:
import io.casehub.ledger.api.model.LedgerAttestation;
```

- [ ] **Step 2: Fix test import**

In `LifeOutcomeAttestationWriterTest.java`, change line 5:
```java
// OLD:
import io.casehub.ledger.runtime.model.LedgerAttestation;
// NEW:
import io.casehub.ledger.api.model.LedgerAttestation;
```

- [ ] **Step 3: Verify compilation**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Run existing attestation tests**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeOutcomeAttestationWriterTest --batch-mode
```
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriter.java app/src/test/java/io/casehub/life/app/service/LifeOutcomeAttestationWriterTest.java
git commit -m "chore(#61): migrate LedgerAttestation import runtime.model → api.model

Refs #61"
```

---

### Task 2: LifeChannelContextProvider — Unit Tests + Implementation

The core CDI bean. Queries recent messages from delegation, oversight, and
per-actor channels. Returns a structured `Map<String, Object>` keyed by
`"channelContext"`.

**Files:**
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeChannelContextProviderTest.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeChannelContextProvider.java`

**Interfaces:**
- Consumes: `ChannelService.findByName(String) → Optional<Channel>`,
  `MessageStore.scan(MessageQuery) → List<Message>`,
  `WorkItem.list(String, Object...) → List<WorkItem>` (Panache),
  `LifeTaskContext.findByIdOptional(UUID) → Optional<LifeTaskContext>`,
  `LifeChannelInitializer.DELEGATION_CHANNEL`, `LifeChannelInitializer.OVERSIGHT_CHANNEL`
- Produces: `LifeChannelContextProvider.gatherContext(UUID caseId) → Map<String, Object>`
  returning `{"channelContext": {"delegation": [...], "oversight": [...], "actor/ext-{id}": [...]}}`
  where each message is `Map.of("sender", ..., "type", ..., "content", ..., "createdAt", ...)`

- [ ] **Step 1: Write unit test — delegation and oversight channels always queried**

Create `app/src/test/java/io/casehub/life/app/engine/LifeChannelContextProviderTest.java`:

```java
package io.casehub.life.app.engine;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class LifeChannelContextProviderTest {

    private ChannelService channelService;
    private MessageStore messageStore;
    private LifeChannelContextProvider provider;

    private final UUID delegationChannelId = UUID.randomUUID();
    private final UUID oversightChannelId = UUID.randomUUID();

    @BeforeEach
    void setup() {
        channelService = mock(ChannelService.class);
        messageStore = mock(MessageStore.class);

        when(channelService.findByName("life/delegation"))
                .thenReturn(Optional.of(channelWithId(delegationChannelId)));
        when(channelService.findByName("life/oversight"))
                .thenReturn(Optional.of(channelWithId(oversightChannelId)));

        when(messageStore.scan(any(MessageQuery.class))).thenReturn(List.of());

        provider = new LifeChannelContextProvider(channelService, messageStore, 10);
    }

    @Test
    void gatherContext_alwaysQueriesDelegationAndOversight() {
        UUID caseId = UUID.randomUUID();

        Map<String, Object> result = provider.gatherContext(caseId);

        @SuppressWarnings("unchecked")
        var ctx = (Map<String, Object>) result.get("channelContext");
        assertThat(ctx).containsKeys("delegation", "oversight");
    }

    @Test
    void gatherContext_serializesMessagesInChronologicalOrder() {
        UUID caseId = UUID.randomUUID();
        Instant t1 = Instant.parse("2026-07-07T10:00:00Z");
        Instant t2 = Instant.parse("2026-07-07T11:00:00Z");

        Message older = message("finance-agent", MessageType.STATUS, "Budget OK", t1);
        Message newer = message("home-agent", MessageType.STATUS, "Task done", t2);

        // descending query returns newer first
        when(messageStore.scan(argThat(q -> delegationChannelId.equals(q.channelId()))))
                .thenReturn(List.of(newer, older));

        Map<String, Object> result = provider.gatherContext(caseId);

        @SuppressWarnings("unchecked")
        var ctx = (Map<String, Object>) result.get("channelContext");
        @SuppressWarnings("unchecked")
        var messages = (List<Map<String, Object>>) ctx.get("delegation");
        assertThat(messages).hasSize(2);
        // chronological: older first
        assertThat(messages.get(0).get("sender")).isEqualTo("finance-agent");
        assertThat(messages.get(1).get("sender")).isEqualTo("home-agent");
    }

    @Test
    void gatherContext_respectsMessageLimit() {
        UUID caseId = UUID.randomUUID();
        provider = new LifeChannelContextProvider(channelService, messageStore, 5);

        provider.gatherContext(caseId);

        verify(messageStore, atLeastOnce()).scan(argThat(q -> q.limit() != null && q.limit() == 5));
    }

    @Test
    void gatherContext_serializationFormat() {
        UUID caseId = UUID.randomUUID();
        Instant t = Instant.parse("2026-07-07T12:00:00Z");
        Message msg = message("health-agent", MessageType.COMMAND, "Book appointment", t);

        when(messageStore.scan(argThat(q -> oversightChannelId.equals(q.channelId()))))
                .thenReturn(List.of(msg));

        Map<String, Object> result = provider.gatherContext(caseId);

        @SuppressWarnings("unchecked")
        var ctx = (Map<String, Object>) result.get("channelContext");
        @SuppressWarnings("unchecked")
        var messages = (List<Map<String, Object>>) ctx.get("oversight");
        assertThat(messages).hasSize(1);
        var m = messages.get(0);
        assertThat(m).containsKeys("sender", "type", "content", "createdAt");
        assertThat(m.get("sender")).isEqualTo("health-agent");
        assertThat(m.get("type")).isEqualTo("COMMAND");
        assertThat(m.get("content")).isEqualTo("Book appointment");
        assertThat(m.get("createdAt")).isEqualTo(t.toString());
    }

    // ── Helpers ──

    private static Channel channelWithId(UUID id) {
        return new Channel(id, "test", null, null, null, null, null,
                null, null, null, null, false, false, null, null, null);
    }

    private static Message message(String sender, MessageType type, String content, Instant createdAt) {
        return new Message(1L, UUID.randomUUID(), sender, type, ActorType.AGENT,
                "test-tenant", content, null, null, 0, null, null, null, null, null, 0, createdAt);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeChannelContextProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am
```
Expected: FAIL — `LifeChannelContextProvider` does not exist

- [ ] **Step 3: Write LifeChannelContextProvider implementation**

Create `app/src/main/java/io/casehub/life/app/engine/LifeChannelContextProvider.java`:

```java
package io.casehub.life.app.engine;

import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.app.infrastructure.LifeChannelInitializer;
import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.message.Message;
import io.casehub.qhorus.api.store.MessageStore;
import io.casehub.qhorus.api.store.query.MessageQuery;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
public class LifeChannelContextProvider {

    private static final Logger LOG = Logger.getLogger(LifeChannelContextProvider.class);

    private final ChannelService channelService;
    private final MessageStore messageStore;
    private final int messageLimit;

    @Inject
    public LifeChannelContextProvider(
            ChannelService channelService,
            MessageStore messageStore,
            @ConfigProperty(name = "casehub.life.channel-context.message-limit", defaultValue = "10")
            int messageLimit) {
        this.channelService = channelService;
        this.messageStore = messageStore;
        this.messageLimit = messageLimit;
    }

    public Map<String, Object> gatherContext(UUID caseId) {
        Map<String, Object> channelContext = new LinkedHashMap<>();

        queryChannel(LifeChannelInitializer.DELEGATION_CHANNEL, "delegation")
                .ifPresent(msgs -> channelContext.put("delegation", msgs));
        queryChannel(LifeChannelInitializer.OVERSIGHT_CHANNEL, "oversight")
                .ifPresent(msgs -> channelContext.put("oversight", msgs));

        resolveActorChannels(caseId).forEach((channelName, label) ->
                queryChannel(channelName, label)
                        .ifPresent(msgs -> channelContext.put(label, msgs)));

        return Map.of("channelContext", channelContext);
    }

    private Optional<List<Map<String, Object>>> queryChannel(String channelName, String label) {
        return channelService.findByName(channelName)
                .map(channel -> {
                    List<Message> messages = messageStore.scan(
                            MessageQuery.builder()
                                    .channelId(channel.id())
                                    .limit(messageLimit)
                                    .descending(true)
                                    .build());
                    List<Message> chronological = new ArrayList<>(messages);
                    Collections.reverse(chronological);
                    return chronological.stream()
                            .map(this::serializeMessage)
                            .toList();
                });
    }

    private Map<String, String> serializeMessage(Message msg) {
        return Map.of(
                "sender", msg.sender() != null ? msg.sender() : "",
                "type", msg.messageType() != null ? msg.messageType().name() : "",
                "content", msg.content() != null ? msg.content() : "",
                "createdAt", msg.createdAt() != null ? msg.createdAt().toString() : "");
    }

    private Map<String, String> resolveActorChannels(UUID caseId) {
        String callerRefPrefix = "case:" + caseId + "/";
        List<WorkItem> workItems = WorkItem.list("callerRef LIKE ?1", callerRefPrefix + "%");

        Map<String, String> actorChannels = new LinkedHashMap<>();
        for (WorkItem wi : workItems) {
            LifeTaskContext.findByIdOptional(wi.id)
                    .map(obj -> (LifeTaskContext) obj)
                    .filter(ctx -> ctx.externalActorId != null)
                    .ifPresent(ctx -> {
                        String channelName = "life/actor/ext-" + ctx.externalActorId;
                        actorChannels.put(channelName, "actor/ext-" + ctx.externalActorId);
                    });
        }
        return actorChannels;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeChannelContextProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am
```
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/engine/LifeChannelContextProvider.java app/src/test/java/io/casehub/life/app/engine/LifeChannelContextProviderTest.java
git commit -m "feat(#61): LifeChannelContextProvider — query channel messages for heartbeat enrichment

Refs #61"
```

---

### Task 3: LifeHeartbeatJob — Inject Provider + Fault Isolation

Modify the heartbeat job to inject `LifeChannelContextProvider`, call
`gatherContext(caseId)`, merge the result into the case context map, and
wrap channel context gathering in try-catch for fault isolation.

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java`
- Modify: `app/src/test/java/io/casehub/life/app/engine/LifeHeartbeatJobTest.java`
- Modify: `app/src/main/resources/application.properties` (add config property)
- Modify: `app/src/test/resources/application.properties` (add config property)

**Interfaces:**
- Consumes: `LifeChannelContextProvider.gatherContext(UUID) → Map<String, Object>`
- Produces: enriched case context map passed to `sentinelAgent.execute()`

- [ ] **Step 1: Write failing test — heartbeat merges channel context**

Add to `LifeHeartbeatJobTest.java`:

```java
@Test
void executeMergesChannelContextIntoCaseContext() throws Exception {
    LifeChannelContextProvider mockProvider = mock(LifeChannelContextProvider.class);
    when(mockProvider.gatherContext(caseId)).thenReturn(
            Map.of("channelContext", Map.of("delegation", List.of(
                    Map.of("sender", "finance-agent", "type", "STATUS",
                            "content", "Budget warning", "createdAt", "2026-07-07T10:00:00Z")))));
    job.channelContextProvider = mockProvider;

    JobExecutionContext ctx = mock(JobExecutionContext.class);
    JobDataMap data = new JobDataMap();
    data.put("agent", "HOME");
    data.put("caseId", caseId.toString());
    data.put("capabilityName", "contractor-sentinel");
    when(ctx.getMergedJobDataMap()).thenReturn(data);

    job.execute(ctx);

    verify(mockProvider).gatherContext(caseId);
    verify(mockRuntime).signal(eq(caseId), eq("sentinelReport"), any(Map.class));
}
```

- [ ] **Step 2: Write failing test — fault isolation**

Add to `LifeHeartbeatJobTest.java`:

```java
@Test
void executeCompletesWhenChannelContextFails() throws Exception {
    LifeChannelContextProvider mockProvider = mock(LifeChannelContextProvider.class);
    when(mockProvider.gatherContext(any(UUID.class)))
            .thenThrow(new RuntimeException("Channel DB unavailable"));
    job.channelContextProvider = mockProvider;

    JobExecutionContext ctx = mock(JobExecutionContext.class);
    JobDataMap data = new JobDataMap();
    data.put("agent", "HOME");
    data.put("caseId", caseId.toString());
    data.put("capabilityName", "contractor-sentinel");
    when(ctx.getMergedJobDataMap()).thenReturn(data);

    // Should not throw — fault isolation catches the exception
    job.execute(ctx);

    // Heartbeat still executes agent and signals result (with case context only)
    verify(mockRuntime).signal(eq(caseId), eq("sentinelReport"), any(Map.class));
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeHeartbeatJobTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am
```
Expected: FAIL — `channelContextProvider` field does not exist

- [ ] **Step 4: Modify LifeHeartbeatJob**

Update `app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java`:

Add field after `caseHubRuntime`:
```java
@Inject
LifeChannelContextProvider channelContextProvider;
```

Replace the `execute()` method body with:
```java
@Override
@SuppressWarnings("unchecked")
public void execute(JobExecutionContext ctx) throws JobExecutionException {
    JobDataMap data = ctx.getMergedJobDataMap();
    LifeAgent agent = LifeAgent.valueOf(data.getString("agent"));
    UUID caseId = UUID.fromString(data.getString("caseId"));
    String capabilityName = data.getString("capabilityName");

    Map<String, Object> caseContext = (Map<String, Object>)
            caseHubRuntime.query(caseId, ".").toCompletableFuture().join();

    // Enrich with channel context — fault-isolated
    Map<String, Object> enrichedContext = new java.util.HashMap<>(caseContext);
    try {
        enrichedContext.putAll(channelContextProvider.gatherContext(caseId));
    } catch (Exception e) {
        LOG.warnf(e, "Channel context gathering failed for case %s — proceeding with case context only", caseId);
    }

    Agent sentinelAgent = Agent.builder()
            .model(openClawFactory.forAgent(agent))
            .systemPrompt(sentinelSystemPrompt(capabilityName))
            .responseSchema(sentinelResponseSchema(capabilityName))
            .build();

    WorkerResult result = sentinelAgent.execute(enrichedContext);

    caseHubRuntime.signal(caseId, "sentinelReport", result.output())
            .toCompletableFuture().join();
}
```

Add logger field at top of class:
```java
private static final Logger LOG = Logger.getLogger(LifeHeartbeatJob.class);
```

Add import:
```java
import org.jboss.logging.Logger;
```

- [ ] **Step 5: Add config property**

Append to `app/src/main/resources/application.properties` after the sentinel config block:
```properties
# Channel context enrichment for heartbeat sentinels (life#61)
casehub.life.channel-context.message-limit=10
```

Append to `app/src/test/resources/application.properties`:
```properties
# Channel context enrichment
casehub.life.channel-context.message-limit=5
```

- [ ] **Step 6: Run tests to verify they pass**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeHeartbeatJobTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am
```
Expected: All tests PASS (including existing tests + 2 new tests)

- [ ] **Step 7: Run full test suite**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode
```
Expected: All 417+ tests PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java app/src/test/java/io/casehub/life/app/engine/LifeHeartbeatJobTest.java app/src/main/resources/application.properties app/src/test/resources/application.properties
git commit -m "feat(#61): wire LifeChannelContextProvider into LifeHeartbeatJob with fault isolation

Heartbeat sentinels now receive recent qhorus channel messages
alongside case context. Channel context failures log a warning
and the heartbeat proceeds with case context only.

Refs #61"
```

---

### Task 4: Integration Test — Actor Channel Resolution + Fault Isolation

The most complex test scenario: verify that `gatherContext()` correctly
resolves actor channels via the `WorkItem.callerRef` → `LifeTaskContext.externalActorId`
chain, and that fault isolation works end-to-end.

**Files:**
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeChannelContextProviderIntegrationTest.java`

**Interfaces:**
- Consumes: `LifeChannelContextProvider.gatherContext(UUID) → Map<String, Object>`,
  `LifeChannelInitializer.ensureActorChannel(UUID)`,
  `MessageService.dispatch(MessageDispatch) → DispatchResult`,
  `WorkItemService.create(WorkItemCreateRequest) → WorkItem`

- [ ] **Step 1: Write integration test**

Create `app/src/test/java/io/casehub/life/app/engine/LifeChannelContextProviderIntegrationTest.java`:

```java
package io.casehub.life.app.engine;

import io.casehub.life.app.LifeTestFixtures;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.infrastructure.LifeChannelInitializer;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.work.api.WorkItemCreateRequest;
import io.casehub.work.api.WorkItemPriority;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class LifeChannelContextProviderIntegrationTest {

    @Inject LifeChannelContextProvider provider;
    @Inject LifeChannelInitializer channelInitializer;
    @Inject MessageService messageService;
    @Inject WorkItemService workItemService;

    @BeforeEach
    @Transactional
    void setUp() {
        LifeTestFixtures.seedStandardTemplates();
    }

    @Test
    void gatherContext_returnsDelegationMessages() {
        UUID caseId = UUID.randomUUID();
        UUID delegationChannelId = channelInitializer.channelIdFor(
                LifeChannelInitializer.DELEGATION_CHANNEL);

        messageService.dispatch(new MessageDispatch.Builder()
                .channelId(delegationChannelId)
                .sender("finance-agent")
                .type(MessageType.STATUS)
                .content("Budget warning: grocery spend at 90%")
                .actorType(ActorType.AGENT)
                .tenancyId("278776f9-e1b0-46fb-9032-8bddebdcf9ce")
                .build());

        Map<String, Object> result = provider.gatherContext(caseId);

        @SuppressWarnings("unchecked")
        var ctx = (Map<String, Object>) result.get("channelContext");
        @SuppressWarnings("unchecked")
        var messages = (List<Map<String, Object>>) ctx.get("delegation");
        assertThat(messages).isNotEmpty();
        assertThat(messages).anyMatch(m ->
                "finance-agent".equals(m.get("sender"))
                && "Budget warning: grocery spend at 90%".equals(m.get("content")));
    }

    @Test
    @Transactional
    void gatherContext_includesActorChannelWhenWorkItemHasExternalActor() {
        UUID caseId = UUID.randomUUID();
        UUID externalActorId = UUID.randomUUID();

        // Create a WorkItem with callerRef matching our caseId
        var req = WorkItemCreateRequest.builder()
                .title("Contractor job")
                .types(List.of("contractor"))
                .priority(WorkItemPriority.MEDIUM)
                .candidateGroups("household-member")
                .createdBy("life-system")
                .callerRef("case:" + caseId + "/pi:test-plan-item")
                .scope("casehubio/life/contractor_coordination")
                .expiresAt(Instant.now().plusSeconds(3600))
                .build();
        WorkItem wi = workItemService.create(req);

        // Create LifeTaskContext linking WorkItem to external actor
        var ctx = new LifeTaskContext();
        ctx.workItemId = wi.id;
        ctx.domain = LifeDomain.CONTRACTOR_COORDINATION;
        ctx.externalActorId = externalActorId;
        ctx.persist();

        // Ensure the actor channel exists and post a message
        UUID actorChannelId = channelInitializer.ensureActorChannel(externalActorId);
        messageService.dispatch(new MessageDispatch.Builder()
                .channelId(actorChannelId)
                .sender("contractor-agent")
                .type(MessageType.STATUS)
                .content("Quote received: £2500")
                .actorType(ActorType.AGENT)
                .tenancyId("278776f9-e1b0-46fb-9032-8bddebdcf9ce")
                .build());

        Map<String, Object> result = provider.gatherContext(caseId);

        @SuppressWarnings("unchecked")
        var channelCtx = (Map<String, Object>) result.get("channelContext");
        String actorKey = "actor/ext-" + externalActorId;
        assertThat(channelCtx).containsKey(actorKey);
        @SuppressWarnings("unchecked")
        var actorMessages = (List<Map<String, Object>>) channelCtx.get(actorKey);
        assertThat(actorMessages).anyMatch(m ->
                "Quote received: £2500".equals(m.get("content")));
    }

    @Test
    void gatherContext_omitsActorChannelWhenNoCaseWorkItems() {
        UUID caseId = UUID.randomUUID();

        Map<String, Object> result = provider.gatherContext(caseId);

        @SuppressWarnings("unchecked")
        var ctx = (Map<String, Object>) result.get("channelContext");
        assertThat(ctx).containsOnlyKeys("delegation", "oversight");
    }
}
```

- [ ] **Step 2: Run integration tests**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeChannelContextProviderIntegrationTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am
```
Expected: All 3 tests PASS

- [ ] **Step 3: Run full test suite**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode
```
Expected: All tests PASS

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/life/app/engine/LifeChannelContextProviderIntegrationTest.java
git commit -m "test(#61): integration tests for LifeChannelContextProvider — actor channel resolution + delegation

Refs #61"
```
