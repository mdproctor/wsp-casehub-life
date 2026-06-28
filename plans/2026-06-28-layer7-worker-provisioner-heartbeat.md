# Layer 7 WorkerProvisioner Heartbeat Mode — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add persistent heartbeat-mode sentinel monitoring to all 7 active case plans via a life-owned `ReactiveWorkerProvisioner` implementation with Quartz-scheduled periodic LLM invocations.

**Architecture:** `LifeReactiveWorkerProvisioner` implements `ReactiveWorkerProvisioner` SPI. When a sentinel binding fires (no inline worker), the provisioner registers the sentinel in `LifeSentinelRegistry` and schedules a Quartz `LifeHeartbeatJob`. Each heartbeat queries case context via `CaseHubRuntime.query()`, invokes `Agent.execute()` via the existing DirectCallBridge path, and signals the result back via `CaseHubRuntime.signal()`. `LifeProvisionerCleanupObserver` terminates sentinels on case completion.

**Tech Stack:** Java 21, Quarkus 3.32.2, Quartz (via quarkus-quartz), casehub-engine-api (`CaseHubRuntime`, `ReactiveWorkerProvisioner`)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app`
- Single test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClassName --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
- All commits must reference `Refs #37`
- `casehub.life.tenancy-id` is required (no default). Test value: `278776f9-e1b0-46fb-9032-8bddebdcf9ce`
- Sentinel capabilities must NEVER have inline workers registered in CaseHub `augment()` — they are reserved for the provisioner path
- `NoOpReactiveWorkerProvisioner` is `@DefaultBean` — automatically displaced by `LifeReactiveWorkerProvisioner`, no exclusion needed
- All openclaw-casehub CDI exclusions remain unchanged — sentinels use no openclaw-casehub beans
- `CaseHubRuntime` is a public API in `casehub-engine-api`, injectable via `@Inject CaseHubRuntime`
- `LifeAgent` enum and `LifeOpenClawChatModelFactory` from life#46 are available
- Quartz `Scheduler` is auto-injected by quarkus-quartz (`@Inject Scheduler`)

---

### Task 1: LifeSentinelRegistry + unit tests

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeSentinelRegistry.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeSentinelRegistryTest.java`

**Interfaces:**
- Consumes: `LifeAgent` (existing), `org.quartz.JobKey` (Quartz)
- Produces: `LifeSentinelRegistry.isProvisioned(UUID, String)` → `boolean`, `register(SentinelEntry)`, `findByCaseId(UUID)` → `List<SentinelEntry>`, `removeByCaseId(UUID)`, `SentinelEntry` record

- [ ] **Step 1: Write the failing test**

```java
// app/src/test/java/io/casehub/life/app/engine/LifeSentinelRegistryTest.java
package io.casehub.life.app.engine;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.quartz.JobKey;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class LifeSentinelRegistryTest {

    private LifeSentinelRegistry registry;

    @BeforeEach
    void setup() {
        registry = new LifeSentinelRegistry();
    }

    @Test
    void notProvisionedByDefault() {
        assertThat(registry.isProvisioned(UUID.randomUUID(), "contractor-sentinel")).isFalse();
    }

    @Test
    void provisionedAfterRegister() {
        UUID caseId = UUID.randomUUID();
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.HOME, caseId, "contractor-sentinel", new JobKey("test", "sentinels")));
        assertThat(registry.isProvisioned(caseId, "contractor-sentinel")).isTrue();
    }

    @Test
    void notProvisionedForDifferentCapability() {
        UUID caseId = UUID.randomUUID();
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.HOME, caseId, "contractor-sentinel", new JobKey("test", "sentinels")));
        assertThat(registry.isProvisioned(caseId, "maintenance-sentinel")).isFalse();
    }

    @Test
    void notProvisionedForDifferentCase() {
        UUID case1 = UUID.randomUUID();
        UUID case2 = UUID.randomUUID();
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.HOME, case1, "contractor-sentinel", new JobKey("test", "sentinels")));
        assertThat(registry.isProvisioned(case2, "contractor-sentinel")).isFalse();
    }

    @Test
    void concurrentCasesSameAgentType() {
        UUID case1 = UUID.randomUUID();
        UUID case2 = UUID.randomUUID();
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.HEALTH, case1, "follow-up-sentinel", new JobKey("j1", "sentinels")));
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.HEALTH, case2, "follow-up-sentinel", new JobKey("j2", "sentinels")));
        assertThat(registry.isProvisioned(case1, "follow-up-sentinel")).isTrue();
        assertThat(registry.isProvisioned(case2, "follow-up-sentinel")).isTrue();
    }

    @Test
    void findByCaseIdReturnsAllEntries() {
        UUID caseId = UUID.randomUUID();
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.HOME, caseId, "contractor-sentinel", new JobKey("j1", "sentinels")));
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.FINANCE, caseId, "anomaly-sentinel", new JobKey("j2", "sentinels")));
        assertThat(registry.findByCaseId(caseId)).hasSize(2);
    }

    @Test
    void findByCaseIdReturnsEmptyForUnknownCase() {
        assertThat(registry.findByCaseId(UUID.randomUUID())).isEmpty();
    }

    @Test
    void removeByCaseIdClearsAll() {
        UUID caseId = UUID.randomUUID();
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.HOME, caseId, "contractor-sentinel", new JobKey("j1", "sentinels")));
        registry.removeByCaseId(caseId);
        assertThat(registry.isProvisioned(caseId, "contractor-sentinel")).isFalse();
        assertThat(registry.findByCaseId(caseId)).isEmpty();
    }

    @Test
    void removeByCaseIdDoesNotAffectOtherCases() {
        UUID case1 = UUID.randomUUID();
        UUID case2 = UUID.randomUUID();
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.HOME, case1, "contractor-sentinel", new JobKey("j1", "sentinels")));
        registry.register(new LifeSentinelRegistry.SentinelEntry(
                LifeAgent.HOME, case2, "contractor-sentinel", new JobKey("j2", "sentinels")));
        registry.removeByCaseId(case1);
        assertThat(registry.isProvisioned(case1, "contractor-sentinel")).isFalse();
        assertThat(registry.isProvisioned(case2, "contractor-sentinel")).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeSentinelRegistryTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `LifeSentinelRegistry` does not exist

- [ ] **Step 3: Write the implementation**

```java
// app/src/main/java/io/casehub/life/app/engine/LifeSentinelRegistry.java
package io.casehub.life.app.engine;

import jakarta.enterprise.context.ApplicationScoped;
import org.quartz.JobKey;

import java.util.List;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;

@ApplicationScoped
public class LifeSentinelRegistry {

    public record SentinelEntry(LifeAgent agent, UUID caseId,
                                String capabilityName, JobKey heartbeatJobKey) {}

    private final ConcurrentHashMap<UUID, CopyOnWriteArrayList<SentinelEntry>> byCaseId =
            new ConcurrentHashMap<>();

    public boolean isProvisioned(UUID caseId, String capabilityName) {
        var entries = byCaseId.get(caseId);
        if (entries == null) return false;
        return entries.stream().anyMatch(e -> e.capabilityName().equals(capabilityName));
    }

    public void register(SentinelEntry entry) {
        byCaseId.computeIfAbsent(entry.caseId(), k -> new CopyOnWriteArrayList<>()).add(entry);
    }

    public List<SentinelEntry> findByCaseId(UUID caseId) {
        var entries = byCaseId.get(caseId);
        return entries != null ? List.copyOf(entries) : List.of();
    }

    public void removeByCaseId(UUID caseId) {
        byCaseId.remove(caseId);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeSentinelRegistryTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 9 tests PASS

- [ ] **Step 5: Commit**

```
git add app/src/main/java/io/casehub/life/app/engine/LifeSentinelRegistry.java app/src/test/java/io/casehub/life/app/engine/LifeSentinelRegistryTest.java
git commit -m "feat(#37): add LifeSentinelRegistry — sentinel tracking with (caseId, capability) keying

Refs #37"
```

---

### Task 2: LifeSentinelConfig + 7 response schemas

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeSentinelConfig.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/ContractorSentinelReport.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/MaintenanceSentinelReport.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/FollowUpSentinelReport.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/CareQualitySentinelReport.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/PatientStatusSentinelReport.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/AnomalySentinelReport.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/BookingSentinelReport.java`

**Interfaces:**
- Consumes: nothing
- Produces: `LifeSentinelConfig.capabilities()` → `Map<String, SentinelCapabilityEntry>`, `SentinelCapabilityEntry.agent()` → `String`, `SentinelCapabilityEntry.heartbeatInterval()` → `Duration`. Seven response schema records (all have `boolean escalationRequired`).

- [ ] **Step 1: Create LifeSentinelConfig**

```java
// app/src/main/java/io/casehub/life/app/engine/LifeSentinelConfig.java
package io.casehub.life.app.engine;

import io.smallrye.config.ConfigMapping;

import java.time.Duration;
import java.util.Map;

@ConfigMapping(prefix = "casehub.life.sentinel")
public interface LifeSentinelConfig {
    Map<String, SentinelCapabilityEntry> capabilities();

    interface SentinelCapabilityEntry {
        String agent();
        Duration heartbeatInterval();
    }
}
```

- [ ] **Step 2: Create 7 response schema records**

```java
// app/src/main/java/io/casehub/life/app/engine/agent/ContractorSentinelReport.java
package io.casehub.life.app.engine.agent;

public record ContractorSentinelReport(
        int progressPercent, String status, String concerns,
        String recommendedAction, boolean escalationRequired) {}
```

```java
// app/src/main/java/io/casehub/life/app/engine/agent/MaintenanceSentinelReport.java
package io.casehub.life.app.engine.agent;

public record MaintenanceSentinelReport(
        int progressPercent, String status, String concerns,
        String recommendedAction, boolean escalationRequired) {}
```

```java
// app/src/main/java/io/casehub/life/app/engine/agent/FollowUpSentinelReport.java
package io.casehub.life.app.engine.agent;

import java.util.List;

public record FollowUpSentinelReport(
        List<String> pendingActions, int daysOverdue,
        String concerns, boolean escalationRequired) {}
```

```java
// app/src/main/java/io/casehub/life/app/engine/agent/CareQualitySentinelReport.java
package io.casehub.life.app.engine.agent;

import java.util.List;

public record CareQualitySentinelReport(
        int sessionsScheduled, int sessionsCompleted, List<String> missedSessions,
        String concerns, boolean escalationRequired) {}
```

```java
// app/src/main/java/io/casehub/life/app/engine/agent/PatientStatusSentinelReport.java
package io.casehub.life.app.engine.agent;

import java.util.List;

public record PatientStatusSentinelReport(
        String conditionSummary, String trend,
        List<String> alerts, boolean escalationRequired) {}
```

```java
// app/src/main/java/io/casehub/life/app/engine/agent/AnomalySentinelReport.java
package io.casehub.life.app.engine.agent;

import java.util.List;

public record AnomalySentinelReport(
        List<String> anomalies, String severity,
        String concerns, boolean escalationRequired) {}
```

```java
// app/src/main/java/io/casehub/life/app/engine/agent/BookingSentinelReport.java
package io.casehub.life.app.engine.agent;

import java.util.List;

public record BookingSentinelReport(
        String bookingStatus, boolean priceChanged, String priceChangeDetail,
        List<String> alerts, boolean escalationRequired) {}
```

- [ ] **Step 3: Compile check**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add app/src/main/java/io/casehub/life/app/engine/LifeSentinelConfig.java app/src/main/java/io/casehub/life/app/engine/agent/ContractorSentinelReport.java app/src/main/java/io/casehub/life/app/engine/agent/MaintenanceSentinelReport.java app/src/main/java/io/casehub/life/app/engine/agent/FollowUpSentinelReport.java app/src/main/java/io/casehub/life/app/engine/agent/CareQualitySentinelReport.java app/src/main/java/io/casehub/life/app/engine/agent/PatientStatusSentinelReport.java app/src/main/java/io/casehub/life/app/engine/agent/AnomalySentinelReport.java app/src/main/java/io/casehub/life/app/engine/agent/BookingSentinelReport.java
git commit -m "feat(#37): add LifeSentinelConfig + 7 per-sentinel response schemas

Refs #37"
```

---

### Task 3: LifeHeartbeatJob + unit test

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeHeartbeatJobTest.java`

**Interfaces:**
- Consumes: `LifeOpenClawChatModelFactory.forAgent(LifeAgent)` (existing), `CaseHubRuntime.query(UUID, String)` and `CaseHubRuntime.signal(UUID, String, Object)` (engine-api), `LifeAgent` (existing), 7 response schemas (Task 2)
- Produces: `LifeHeartbeatJob` (Quartz `Job` implementation). JobDataMap keys: `"agent"` (LifeAgent name), `"caseId"` (UUID string), `"capabilityName"` (sentinel capability).

- [ ] **Step 1: Write the failing test**

```java
// app/src/test/java/io/casehub/life/app/engine/LifeHeartbeatJobTest.java
package io.casehub.life.app.engine;

import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.api.model.ai.ModelType;
import io.casehub.life.app.engine.agent.ContractorSentinelReport;
import io.casehub.life.app.engine.agent.LifeOpenClawChatModelFactory;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.quartz.JobDataMap;
import org.quartz.JobDetail;
import org.quartz.JobExecutionContext;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class LifeHeartbeatJobTest {

    private LifeOpenClawChatModelFactory mockFactory;
    private CaseHubRuntime mockRuntime;
    private LifeHeartbeatJob job;
    private final UUID caseId = UUID.randomUUID();

    @BeforeEach
    void setup() {
        mockFactory = mock(LifeOpenClawChatModelFactory.class);
        mockRuntime = mock(CaseHubRuntime.class);

        ChatModel mockModel = mock(ChatModel.class);
        AiMessage aiMessage = AiMessage.from(
                "{\"progressPercent\":75,\"status\":\"on-track\","
                + "\"concerns\":null,\"recommendedAction\":null,\"escalationRequired\":false}");
        ChatResponse response = ChatResponse.builder().aiMessage(aiMessage).build();
        when(mockModel.chat(any(ChatRequest.class))).thenReturn(response);

        ChatModelProvider provider = new ChatModelProvider() {
            @Override public ModelType type() { return ModelType.OPENAI; }
            @Override public ChatModel get() { return mockModel; }
        };
        when(mockFactory.forAgent(any(LifeAgent.class))).thenReturn(provider);

        when(mockRuntime.query(any(UUID.class), eq(".")))
                .thenReturn(CompletableFuture.completedFuture(
                        Map.of("contractorRequest", Map.of("contractor", "AcmeBuild"))));
        when(mockRuntime.signal(any(UUID.class), any(String.class), any()))
                .thenReturn(CompletableFuture.completedFuture(null));

        job = new LifeHeartbeatJob();
        job.openClawFactory = mockFactory;
        job.caseHubRuntime = mockRuntime;
    }

    @Test
    void executesAgentAndSignalsResult() throws Exception {
        JobExecutionContext ctx = mock(JobExecutionContext.class);
        JobDataMap data = new JobDataMap();
        data.put("agent", "HOME");
        data.put("caseId", caseId.toString());
        data.put("capabilityName", "contractor-sentinel");
        when(ctx.getMergedJobDataMap()).thenReturn(data);

        job.execute(ctx);

        verify(mockRuntime).query(caseId, ".");
        verify(mockFactory).forAgent(LifeAgent.HOME);
        verify(mockRuntime).signal(eq(caseId), eq("sentinelReport"), any(Map.class));
    }

    @Test
    void signalContainsSentinelReportData() throws Exception {
        JobExecutionContext ctx = mock(JobExecutionContext.class);
        JobDataMap data = new JobDataMap();
        data.put("agent", "HOME");
        data.put("caseId", caseId.toString());
        data.put("capabilityName", "contractor-sentinel");
        when(ctx.getMergedJobDataMap()).thenReturn(data);

        job.execute(ctx);

        var captor = org.mockito.ArgumentCaptor.forClass(Object.class);
        verify(mockRuntime).signal(eq(caseId), eq("sentinelReport"), captor.capture());
        @SuppressWarnings("unchecked")
        Map<String, Object> report = (Map<String, Object>) captor.getValue();
        assertThat(report).containsKey("progressPercent");
        assertThat(report.get("escalationRequired")).isEqualTo(false);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeHeartbeatJobTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `LifeHeartbeatJob` does not exist

- [ ] **Step 3: Write the implementation**

```java
// app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java
package io.casehub.life.app.engine;

import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.ai.Agent;
import io.casehub.life.app.engine.agent.AnomalySentinelReport;
import io.casehub.life.app.engine.agent.BookingSentinelReport;
import io.casehub.life.app.engine.agent.CareQualitySentinelReport;
import io.casehub.life.app.engine.agent.ContractorSentinelReport;
import io.casehub.life.app.engine.agent.FollowUpSentinelReport;
import io.casehub.life.app.engine.agent.LifeOpenClawChatModelFactory;
import io.casehub.life.app.engine.agent.MaintenanceSentinelReport;
import io.casehub.life.app.engine.agent.PatientStatusSentinelReport;
import io.casehub.worker.api.WorkerResult;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.quartz.Job;
import org.quartz.JobDataMap;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class LifeHeartbeatJob implements Job {

    @Inject
    LifeOpenClawChatModelFactory openClawFactory;

    @Inject
    CaseHubRuntime caseHubRuntime;

    @Override
    @SuppressWarnings("unchecked")
    public void execute(JobExecutionContext ctx) throws JobExecutionException {
        JobDataMap data = ctx.getMergedJobDataMap();
        LifeAgent agent = LifeAgent.valueOf(data.getString("agent"));
        UUID caseId = UUID.fromString(data.getString("caseId"));
        String capabilityName = data.getString("capabilityName");

        Map<String, Object> caseContext = (Map<String, Object>)
                caseHubRuntime.query(caseId, ".").toCompletableFuture().join();

        Agent sentinelAgent = Agent.builder()
                .model(openClawFactory.forAgent(agent))
                .systemPrompt(sentinelSystemPrompt(capabilityName))
                .responseSchema(sentinelResponseSchema(capabilityName))
                .build();

        WorkerResult result = sentinelAgent.execute(caseContext);

        caseHubRuntime.signal(caseId, "sentinelReport", result.output())
                .toCompletableFuture().join();
    }

    static Class<?> sentinelResponseSchema(String capabilityName) {
        return switch (capabilityName) {
            case "contractor-sentinel" -> ContractorSentinelReport.class;
            case "maintenance-sentinel" -> MaintenanceSentinelReport.class;
            case "follow-up-sentinel" -> FollowUpSentinelReport.class;
            case "care-quality-sentinel" -> CareQualitySentinelReport.class;
            case "patient-status-sentinel" -> PatientStatusSentinelReport.class;
            case "anomaly-sentinel" -> AnomalySentinelReport.class;
            case "booking-sentinel" -> BookingSentinelReport.class;
            default -> throw new IllegalArgumentException("Unknown sentinel: " + capabilityName);
        };
    }

    static String sentinelSystemPrompt(String capabilityName) {
        return switch (capabilityName) {
            case "contractor-sentinel" -> """
                    You are a contractor progress monitoring agent for a UK household.
                    Check on the status of the active contractor job for this case.
                    Report current progress, status (on-track/delayed/stalled),
                    any concerns, and recommended actions.""";
            case "maintenance-sentinel" -> """
                    You are a home maintenance progress monitoring agent for a UK household.
                    Check on the status of the active maintenance job for this case.
                    Report current progress, status (on-track/delayed/stalled),
                    any concerns, and recommended actions.""";
            case "follow-up-sentinel" -> """
                    You are a health appointment follow-up agent for a UK household.
                    Check whether post-appointment actions have been completed:
                    prescriptions collected, referrals booked, test results received.
                    Report pending actions, days overdue, and whether escalation is needed.""";
            case "care-quality-sentinel" -> """
                    You are a care quality monitoring agent for a UK household.
                    Check whether scheduled care sessions have been delivered.
                    Report sessions scheduled vs completed, any missed sessions,
                    concerns, and whether escalation is needed.""";
            case "patient-status-sentinel" -> """
                    You are a patient status monitoring agent for a UK household.
                    Assess the patient's current condition between care episodes.
                    Report condition summary, trend (improving/stable/declining),
                    any alerts, and whether escalation is needed.""";
            case "anomaly-sentinel" -> """
                    You are a financial anomaly detection agent for a UK household.
                    Scan recent transactions for unusual patterns, budget overruns,
                    or suspicious activity. Report anomalies found, severity, and
                    whether escalation is needed.""";
            case "booking-sentinel" -> """
                    You are a travel booking monitoring agent for a UK household.
                    Check booking confirmations, price changes, and availability.
                    Report booking status, any price changes, alerts, and whether
                    escalation is needed.""";
            default -> throw new IllegalArgumentException("Unknown sentinel: " + capabilityName);
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeHeartbeatJobTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 2 tests PASS

- [ ] **Step 5: Commit**

```
git add app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java app/src/test/java/io/casehub/life/app/engine/LifeHeartbeatJobTest.java
git commit -m "feat(#37): add LifeHeartbeatJob — Quartz job for sentinel heartbeat

Queries case context, invokes Agent.execute() via DirectCallBridge,
signals result via CaseHubRuntime.signal().

Refs #37"
```

---

### Task 4: LifeReactiveWorkerProvisioner + unit test

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeReactiveWorkerProvisioner.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeReactiveWorkerProvisionerTest.java`

**Interfaces:**
- Consumes: `LifeSentinelRegistry` (Task 1), `LifeSentinelConfig` (Task 2), `LifeHeartbeatJob` (Task 3 — class reference for Quartz JobDetail), `org.quartz.Scheduler`, `LifeAgent`
- Produces: `ReactiveWorkerProvisioner` SPI implementation. `provision(Set<String>, ProvisionContext)` → `Uni<ProvisionResult>`, `terminate(String, String)` → `Uni<Void>`, `terminateAllForCase(UUID)`, `getCapabilities()` → `Uni<Set<String>>`

- [ ] **Step 1: Write the failing test**

```java
// app/src/test/java/io/casehub/life/app/engine/LifeReactiveWorkerProvisionerTest.java
package io.casehub.life.app.engine;

import io.casehub.api.model.ProvisionContext;
import io.casehub.api.spi.ProvisionResult;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.quartz.JobKey;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;

import java.time.Duration;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class LifeReactiveWorkerProvisionerTest {

    private LifeReactiveWorkerProvisioner provisioner;
    private LifeSentinelRegistry registry;
    private Scheduler mockScheduler;

    @BeforeEach
    void setup() throws SchedulerException {
        registry = new LifeSentinelRegistry();
        mockScheduler = mock(Scheduler.class);
        when(mockScheduler.scheduleJob(any(), any())).thenReturn(null);

        LifeSentinelConfig config = mock(LifeSentinelConfig.class);
        LifeSentinelConfig.SentinelCapabilityEntry entry = mock(LifeSentinelConfig.SentinelCapabilityEntry.class);
        when(entry.agent()).thenReturn("HOME");
        when(entry.heartbeatInterval()).thenReturn(Duration.ofHours(4));
        when(config.capabilities()).thenReturn(Map.of("contractor-sentinel", entry));

        provisioner = new LifeReactiveWorkerProvisioner();
        provisioner.sentinelRegistry = registry;
        provisioner.sentinelConfig = config;
        provisioner.scheduler = mockScheduler;
    }

    @Test
    void firstProvisionRegistersAndSchedules() throws SchedulerException {
        UUID caseId = UUID.randomUUID();
        ProvisionContext ctx = new ProvisionContext(
                caseId, "test-tenant", "contractor-sentinel",
                null, null, null, null);

        ProvisionResult result = provisioner.provision(Set.of("contractor-sentinel"), ctx)
                .await().indefinitely();

        assertThat(result).isNotNull();
        assertThat(registry.isProvisioned(caseId, "contractor-sentinel")).isTrue();
        verify(mockScheduler).scheduleJob(any(), any());
    }

    @Test
    void secondProvisionIsIdempotent() throws SchedulerException {
        UUID caseId = UUID.randomUUID();
        ProvisionContext ctx = new ProvisionContext(
                caseId, "test-tenant", "contractor-sentinel",
                null, null, null, null);

        provisioner.provision(Set.of("contractor-sentinel"), ctx).await().indefinitely();
        provisioner.provision(Set.of("contractor-sentinel"), ctx).await().indefinitely();

        assertThat(registry.findByCaseId(caseId)).hasSize(1);
        verify(mockScheduler, times(1)).scheduleJob(any(), any());
    }

    @Test
    void terminateAllForCaseCancelsHeartbeatAndDeregisters() throws SchedulerException {
        UUID caseId = UUID.randomUUID();
        ProvisionContext ctx = new ProvisionContext(
                caseId, "test-tenant", "contractor-sentinel",
                null, null, null, null);

        provisioner.provision(Set.of("contractor-sentinel"), ctx).await().indefinitely();
        provisioner.terminateAllForCase(caseId);

        assertThat(registry.findByCaseId(caseId)).isEmpty();
        verify(mockScheduler).deleteJob(any(JobKey.class));
    }

    @Test
    void getCapabilitiesReturnsConfigKeys() {
        Set<String> capabilities = provisioner.getCapabilities().await().indefinitely();
        assertThat(capabilities).containsExactly("contractor-sentinel");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeReactiveWorkerProvisionerTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `LifeReactiveWorkerProvisioner` does not exist

- [ ] **Step 3: Write the implementation**

```java
// app/src/main/java/io/casehub/life/app/engine/LifeReactiveWorkerProvisioner.java
package io.casehub.life.app.engine;

import io.casehub.api.model.ProvisionContext;
import io.casehub.api.spi.ProvisionResult;
import io.casehub.api.spi.ReactiveWorkerProvisioner;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.quartz.JobBuilder;
import org.quartz.JobKey;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.SimpleScheduleBuilder;
import org.quartz.TriggerBuilder;
import io.quarkus.logging.Log;

import java.time.Duration;
import java.util.Set;
import java.util.UUID;

@ApplicationScoped
public class LifeReactiveWorkerProvisioner implements ReactiveWorkerProvisioner {

    @Inject
    LifeSentinelRegistry sentinelRegistry;

    @Inject
    LifeSentinelConfig sentinelConfig;

    @Inject
    Scheduler scheduler;

    @Override
    public Uni<ProvisionResult> provision(Set<String> capabilities, ProvisionContext context) {
        return Uni.createFrom().item(() -> {
            String capabilityName = context.taskType();

            if (sentinelRegistry.isProvisioned(context.caseId(), capabilityName)) {
                return ProvisionResult.empty();
            }

            LifeAgent agent = resolveAgent(capabilityName);
            Duration interval = sentinelConfig.capabilities()
                    .get(capabilityName).heartbeatInterval();
            JobKey jobKey = scheduleHeartbeat(agent, context.caseId(), capabilityName, interval);

            sentinelRegistry.register(new LifeSentinelRegistry.SentinelEntry(
                    agent, context.caseId(), capabilityName, jobKey));

            Log.infof("Provisioned sentinel: capability=%s agent=%s caseId=%s interval=%s",
                    capabilityName, agent.agentId(), context.caseId(), interval);

            return ProvisionResult.empty();
        });
    }

    @Override
    public Uni<Void> terminate(String workerId, String tenancyId) {
        return Uni.createFrom().voidItem();
    }

    public void terminateAllForCase(UUID caseId) {
        sentinelRegistry.findByCaseId(caseId).forEach(entry -> {
            cancelHeartbeat(entry.heartbeatJobKey());
            Log.infof("Terminated sentinel: capability=%s caseId=%s",
                    entry.capabilityName(), caseId);
        });
        sentinelRegistry.removeByCaseId(caseId);
    }

    @Override
    public Uni<Set<String>> getCapabilities() {
        return Uni.createFrom().item(() ->
                Set.copyOf(sentinelConfig.capabilities().keySet()));
    }

    LifeAgent resolveAgent(String capabilityName) {
        var entry = sentinelConfig.capabilities().get(capabilityName);
        if (entry == null) {
            throw new io.casehub.api.spi.ProvisioningException(
                    "No sentinel configuration for capability: " + capabilityName);
        }
        return LifeAgent.valueOf(entry.agent());
    }

    private JobKey scheduleHeartbeat(LifeAgent agent, UUID caseId,
                                     String capabilityName, Duration interval) {
        String jobName = capabilityName + "-" + caseId;
        String group = "life-sentinels";
        JobKey jobKey = new JobKey(jobName, group);

        try {
            var jobDetail = JobBuilder.newJob(LifeHeartbeatJob.class)
                    .withIdentity(jobKey)
                    .usingJobData("agent", agent.name())
                    .usingJobData("caseId", caseId.toString())
                    .usingJobData("capabilityName", capabilityName)
                    .build();

            var trigger = TriggerBuilder.newTrigger()
                    .withIdentity(jobName + "-trigger", group)
                    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                            .withIntervalInMilliseconds(interval.toMillis())
                            .repeatForever())
                    .startNow()
                    .build();

            scheduler.scheduleJob(jobDetail, trigger);
        } catch (SchedulerException e) {
            throw new io.casehub.api.spi.ProvisioningException(
                    "Failed to schedule heartbeat for " + capabilityName + ": " + e.getMessage(), e);
        }

        return jobKey;
    }

    private void cancelHeartbeat(JobKey jobKey) {
        try {
            scheduler.deleteJob(jobKey);
        } catch (SchedulerException e) {
            Log.warnf("Failed to cancel heartbeat job %s: %s", jobKey, e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeReactiveWorkerProvisionerTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 4 tests PASS

- [ ] **Step 5: Commit**

```
git add app/src/main/java/io/casehub/life/app/engine/LifeReactiveWorkerProvisioner.java app/src/test/java/io/casehub/life/app/engine/LifeReactiveWorkerProvisionerTest.java
git commit -m "feat(#37): add LifeReactiveWorkerProvisioner — idempotent provisioner with Quartz heartbeat

Refs #37"
```

---

### Task 5: LifeProvisionerCleanupObserver + unit test

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeProvisionerCleanupObserver.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeProvisionerCleanupObserverTest.java`

**Interfaces:**
- Consumes: `LifeReactiveWorkerProvisioner.terminateAllForCase(UUID)` (Task 4), `CaseLifecycleEvent` (engine-common)
- Produces: CDI observer that calls `terminateAllForCase()` on terminal case events

- [ ] **Step 1: Write the failing test**

```java
// app/src/test/java/io/casehub/life/app/engine/LifeProvisionerCleanupObserverTest.java
package io.casehub.life.app.engine;

import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import java.util.UUID;

import static org.mockito.Mockito.*;

class LifeProvisionerCleanupObserverTest {

    private LifeReactiveWorkerProvisioner mockProvisioner;
    private LifeProvisionerCleanupObserver observer;

    @BeforeEach
    void setup() {
        mockProvisioner = mock(LifeReactiveWorkerProvisioner.class);
        observer = new LifeProvisionerCleanupObserver();
        observer.provisioner = mockProvisioner;
    }

    @ParameterizedTest
    @ValueSource(strings = {"CaseCompleted", "CaseFaulted", "CaseCancelled"})
    void terminatesOnTerminalEvents(String eventType) {
        UUID caseId = UUID.randomUUID();
        CaseLifecycleEvent event = new CaseLifecycleEvent(
                caseId, "test-tenant", "command", eventType,
                "COMPLETED", null, "System", null);

        observer.onCaseTerminal(event);

        verify(mockProvisioner).terminateAllForCase(caseId);
    }

    @ParameterizedTest
    @ValueSource(strings = {"CaseStarted", "CaseSuspended", "CaseResumed", "WorkSubmitted"})
    void ignoresNonTerminalEvents(String eventType) {
        UUID caseId = UUID.randomUUID();
        CaseLifecycleEvent event = new CaseLifecycleEvent(
                caseId, "test-tenant", "command", eventType,
                "ACTIVE", null, "System", null);

        observer.onCaseTerminal(event);

        verify(mockProvisioner, never()).terminateAllForCase(any());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeProvisionerCleanupObserverTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `LifeProvisionerCleanupObserver` does not exist

- [ ] **Step 3: Write the implementation**

```java
// app/src/main/java/io/casehub/life/app/engine/LifeProvisionerCleanupObserver.java
package io.casehub.life.app.engine;

import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

@ApplicationScoped
public class LifeProvisionerCleanupObserver {

    @Inject
    LifeReactiveWorkerProvisioner provisioner;

    public void onCaseTerminal(@ObservesAsync CaseLifecycleEvent event) {
        if (isTerminal(event.eventType())) {
            provisioner.terminateAllForCase(event.caseId());
        }
    }

    private static boolean isTerminal(String eventType) {
        return "CaseCompleted".equals(eventType)
                || "CaseFaulted".equals(eventType)
                || "CaseCancelled".equals(eventType);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeProvisionerCleanupObserverTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 7 tests PASS (3 terminal + 4 non-terminal parameterized)

- [ ] **Step 5: Commit**

```
git add app/src/main/java/io/casehub/life/app/engine/LifeProvisionerCleanupObserver.java app/src/test/java/io/casehub/life/app/engine/LifeProvisionerCleanupObserverTest.java
git commit -m "feat(#37): add LifeProvisionerCleanupObserver — terminate sentinels on case completion

Engine never calls terminate(). Life handles via CaseLifecycleEvent.

Refs #37"
```

---

### Task 6: YAML changes to 7 case plans

**Files:**
- Modify: `app/src/main/resources/life/appointment-cycle.yaml`
- Modify: `app/src/main/resources/life/care-coordination.yaml`
- Modify: `app/src/main/resources/life/care-episode.yaml`
- Modify: `app/src/main/resources/life/home-maintenance.yaml`
- Modify: `app/src/main/resources/life/contractor-coordination.yaml`
- Modify: `app/src/main/resources/life/financial-review.yaml`
- Modify: `app/src/main/resources/life/travel-plan.yaml`

**Interfaces:**
- Consumes: nothing (declarative YAML)
- Produces: 7 sentinel capabilities + bindings + escalation bindings in YAML

Each case plan gets three additions: a sentinel capability, a sentinel binding, and an escalation binding. The sentinel capability has no inline worker — the engine falls through to the provisioner.

- [ ] **Step 1: Add sentinel to appointment-cycle.yaml**

Append to `capabilities:` section:
```yaml
    - name: follow-up-sentinel
      description: "Persistent heartbeat monitor for appointment follow-up"
      inputSchema: "."
      outputSchema: "."
```

Append to `bindings:` section:
```yaml
    - name: follow-up-sentinel
      on: { contextChange: {} }
      when: ".appointmentType != null"
      capability: follow-up-sentinel

    - name: sentinel-escalation
      on: { contextChange: {} }
      when: ".sentinelReport != null and .sentinelReport.escalationRequired == true and (.sentinelEscalation == null or .sentinelEscalation.resolved == true)"
      humanTask:
        title: "Sentinel detected follow-up issue"
        expiresIn: PT24H
        candidateGroups: [household-admin]
        scope: "casehubio/life/health"
        inputMapping: "{ sentinelReport: .sentinelReport }"
        outputMapping: "{ sentinelEscalation: . }"
```

- [ ] **Step 2: Add sentinel to care-coordination.yaml**

Append to `capabilities:` section:
```yaml
    - name: care-quality-sentinel
      description: "Persistent heartbeat monitor for care quality"
      inputSchema: "."
      outputSchema: "."
```

Append to `bindings:` section:
```yaml
    - name: care-quality-sentinel
      on: { contextChange: {} }
      when: ".careRequest != null"
      capability: care-quality-sentinel

    - name: sentinel-escalation
      on: { contextChange: {} }
      when: ".sentinelReport != null and .sentinelReport.escalationRequired == true and (.sentinelEscalation == null or .sentinelEscalation.resolved == true)"
      humanTask:
        title: "Sentinel detected care quality issue"
        expiresIn: PT12H
        candidateGroups: [household-admin]
        scope: "casehubio/life/health"
        inputMapping: "{ sentinelReport: .sentinelReport }"
        outputMapping: "{ sentinelEscalation: . }"
```

- [ ] **Step 3: Add sentinel to care-episode.yaml**

Append to `capabilities:` section:
```yaml
    - name: patient-status-sentinel
      description: "Persistent heartbeat monitor for patient status"
      inputSchema: "."
      outputSchema: "."
```

Append to `bindings:` section:
```yaml
    - name: patient-status-sentinel
      on: { contextChange: {} }
      when: ".careRequest != null"
      capability: patient-status-sentinel

    - name: sentinel-escalation
      on: { contextChange: {} }
      when: ".sentinelReport != null and .sentinelReport.escalationRequired == true and (.sentinelEscalation == null or .sentinelEscalation.resolved == true)"
      humanTask:
        title: "Sentinel detected patient status concern"
        expiresIn: PT12H
        candidateGroups: [household-admin]
        scope: "casehubio/life/health"
        inputMapping: "{ sentinelReport: .sentinelReport }"
        outputMapping: "{ sentinelEscalation: . }"
```

- [ ] **Step 4: Add sentinel to home-maintenance.yaml**

Append to `capabilities:` section:
```yaml
    - name: maintenance-sentinel
      description: "Persistent heartbeat monitor for maintenance progress"
      inputSchema: "."
      outputSchema: "."
```

Append to `bindings:` section:
```yaml
    - name: maintenance-sentinel
      on: { contextChange: {} }
      when: ".request != null"
      capability: maintenance-sentinel

    - name: sentinel-escalation
      on: { contextChange: {} }
      when: ".sentinelReport != null and .sentinelReport.escalationRequired == true and (.sentinelEscalation == null or .sentinelEscalation.resolved == true)"
      humanTask:
        title: "Sentinel detected maintenance issue"
        expiresIn: PT24H
        candidateGroups: [household-admin]
        scope: "casehubio/life/household"
        inputMapping: "{ sentinelReport: .sentinelReport }"
        outputMapping: "{ sentinelEscalation: . }"
```

- [ ] **Step 5: Add sentinel to contractor-coordination.yaml**

Append to `capabilities:` section:
```yaml
    - name: contractor-sentinel
      description: "Persistent heartbeat monitor for contractor progress and follow-up"
      inputSchema: "."
      outputSchema: "."
```

Append to `bindings:` section:
```yaml
    - name: contractor-sentinel
      on: { contextChange: {} }
      when: ".contractorRequest != null"
      capability: contractor-sentinel

    - name: sentinel-escalation
      on: { contextChange: {} }
      when: ".sentinelReport != null and .sentinelReport.escalationRequired == true and (.sentinelEscalation == null or .sentinelEscalation.resolved == true)"
      humanTask:
        title: "Sentinel detected contractor issue"
        expiresIn: PT24H
        candidateGroups: [household-admin]
        scope: "casehubio/life/household"
        inputMapping: "{ sentinelReport: .sentinelReport }"
        outputMapping: "{ sentinelEscalation: . }"
```

- [ ] **Step 6: Add sentinel to financial-review.yaml**

Append to `capabilities:` section:
```yaml
    - name: anomaly-sentinel
      description: "Persistent heartbeat monitor for financial anomalies"
      inputSchema: "."
      outputSchema: "."
```

Append to `bindings:` section:
```yaml
    - name: anomaly-sentinel
      on: { contextChange: {} }
      when: ".reviewPeriod != null"
      capability: anomaly-sentinel

    - name: sentinel-escalation
      on: { contextChange: {} }
      when: ".sentinelReport != null and .sentinelReport.escalationRequired == true and (.sentinelEscalation == null or .sentinelEscalation.resolved == true)"
      humanTask:
        title: "Sentinel detected financial anomaly"
        expiresIn: PT24H
        candidateGroups: [household-admin]
        scope: "casehubio/life/finance"
        inputMapping: "{ sentinelReport: .sentinelReport }"
        outputMapping: "{ sentinelEscalation: . }"
```

- [ ] **Step 7: Add sentinel to travel-plan.yaml**

Append to `capabilities:` section:
```yaml
    - name: booking-sentinel
      description: "Persistent heartbeat monitor for booking confirmations and price changes"
      inputSchema: "."
      outputSchema: "."
```

Append to `bindings:` section:
```yaml
    - name: booking-sentinel
      on: { contextChange: {} }
      when: ".request != null"
      capability: booking-sentinel

    - name: sentinel-escalation
      on: { contextChange: {} }
      when: ".sentinelReport != null and .sentinelReport.escalationRequired == true and (.sentinelEscalation == null or .sentinelEscalation.resolved == true)"
      humanTask:
        title: "Sentinel detected booking issue"
        expiresIn: PT24H
        candidateGroups: [household-admin]
        scope: "casehubio/life/travel"
        inputMapping: "{ sentinelReport: .sentinelReport }"
        outputMapping: "{ sentinelEscalation: . }"
```

- [ ] **Step 8: Compile check**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```
git add app/src/main/resources/life/
git commit -m "feat(#37): add sentinel capabilities and bindings to all 7 case plans

Each case plan gets a sentinel capability (no inline worker → provisioner
fallback), a sentinel binding, and an escalation humanTask binding.

Refs #37"
```

---

### Task 7: Config + test infrastructure + integration test

**Files:**
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/resources/application.properties`
- Create: `app/src/test/java/io/casehub/life/app/engine/TestLifeReactiveWorkerProvisioner.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeProvisionerIntegrationTest.java`

**Interfaces:**
- Consumes: all Tasks 1-6
- Produces: wired configuration, test alternative, integration test

- [ ] **Step 1: Add sentinel config to production application.properties**

Append to `app/src/main/resources/application.properties`:
```properties

# ============================================================
# Sentinel heartbeat configuration (life#37)
# ============================================================
casehub.life.sentinel.capabilities.contractor-sentinel.agent=HOME
casehub.life.sentinel.capabilities.contractor-sentinel.heartbeat-interval=PT4H
casehub.life.sentinel.capabilities.maintenance-sentinel.agent=HOME
casehub.life.sentinel.capabilities.maintenance-sentinel.heartbeat-interval=PT4H
casehub.life.sentinel.capabilities.follow-up-sentinel.agent=HEALTH
casehub.life.sentinel.capabilities.follow-up-sentinel.heartbeat-interval=PT12H
casehub.life.sentinel.capabilities.care-quality-sentinel.agent=HEALTH
casehub.life.sentinel.capabilities.care-quality-sentinel.heartbeat-interval=PT12H
casehub.life.sentinel.capabilities.patient-status-sentinel.agent=HEALTH
casehub.life.sentinel.capabilities.patient-status-sentinel.heartbeat-interval=PT24H
casehub.life.sentinel.capabilities.anomaly-sentinel.agent=FINANCE
casehub.life.sentinel.capabilities.anomaly-sentinel.heartbeat-interval=PT24H
casehub.life.sentinel.capabilities.booking-sentinel.agent=TRAVEL
casehub.life.sentinel.capabilities.booking-sentinel.heartbeat-interval=PT6H
quarkus.quartz.thread-pool-size=15
```

- [ ] **Step 2: Add sentinel config to test application.properties**

Append to `app/src/test/resources/application.properties`:
```properties

# ============================================================
# Sentinel test config (life#37)
# Short heartbeat intervals for test speed.
# TestLifeReactiveWorkerProvisioner replaces production provisioner.
# ============================================================
casehub.life.sentinel.capabilities.contractor-sentinel.agent=HOME
casehub.life.sentinel.capabilities.contractor-sentinel.heartbeat-interval=PT1S
casehub.life.sentinel.capabilities.maintenance-sentinel.agent=HOME
casehub.life.sentinel.capabilities.maintenance-sentinel.heartbeat-interval=PT1S
casehub.life.sentinel.capabilities.follow-up-sentinel.agent=HEALTH
casehub.life.sentinel.capabilities.follow-up-sentinel.heartbeat-interval=PT1S
casehub.life.sentinel.capabilities.care-quality-sentinel.agent=HEALTH
casehub.life.sentinel.capabilities.care-quality-sentinel.heartbeat-interval=PT1S
casehub.life.sentinel.capabilities.patient-status-sentinel.agent=HEALTH
casehub.life.sentinel.capabilities.patient-status-sentinel.heartbeat-interval=PT1S
casehub.life.sentinel.capabilities.anomaly-sentinel.agent=FINANCE
casehub.life.sentinel.capabilities.anomaly-sentinel.heartbeat-interval=PT1S
casehub.life.sentinel.capabilities.booking-sentinel.agent=TRAVEL
casehub.life.sentinel.capabilities.booking-sentinel.heartbeat-interval=PT1S
```

Add `TestLifeReactiveWorkerProvisioner` to `quarkus.arc.selected-alternatives` (append to existing value):
```properties
  io.casehub.life.app.engine.TestLifeReactiveWorkerProvisioner
```

- [ ] **Step 3: Create TestLifeReactiveWorkerProvisioner**

```java
// app/src/test/java/io/casehub/life/app/engine/TestLifeReactiveWorkerProvisioner.java
package io.casehub.life.app.engine;

import io.casehub.api.model.ProvisionContext;
import io.casehub.api.spi.ProvisionResult;
import io.casehub.api.spi.ReactiveWorkerProvisioner;
import io.smallrye.mutiny.Uni;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import java.util.Set;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;

@Alternative
@Priority(10)
@ApplicationScoped
public class TestLifeReactiveWorkerProvisioner implements ReactiveWorkerProvisioner {

    @Inject
    LifeSentinelRegistry sentinelRegistry;

    @Inject
    LifeSentinelConfig sentinelConfig;

    private final CopyOnWriteArrayList<ProvisionRecord> provisionCalls = new CopyOnWriteArrayList<>();

    public record ProvisionRecord(UUID caseId, String capabilityName) {}

    @Override
    public Uni<ProvisionResult> provision(Set<String> capabilities, ProvisionContext context) {
        return Uni.createFrom().item(() -> {
            String capabilityName = context.taskType();
            if (sentinelRegistry.isProvisioned(context.caseId(), capabilityName)) {
                return ProvisionResult.empty();
            }
            LifeAgent agent = LifeAgent.valueOf(
                    sentinelConfig.capabilities().get(capabilityName).agent());
            sentinelRegistry.register(new LifeSentinelRegistry.SentinelEntry(
                    agent, context.caseId(), capabilityName, null));
            provisionCalls.add(new ProvisionRecord(context.caseId(), capabilityName));
            return ProvisionResult.empty();
        });
    }

    @Override
    public Uni<Void> terminate(String workerId, String tenancyId) {
        return Uni.createFrom().voidItem();
    }

    @Override
    public Uni<Set<String>> getCapabilities() {
        return Uni.createFrom().item(() ->
                Set.copyOf(sentinelConfig.capabilities().keySet()));
    }

    public void terminateAllForCase(UUID caseId) {
        sentinelRegistry.removeByCaseId(caseId);
    }

    public CopyOnWriteArrayList<ProvisionRecord> getProvisionCalls() {
        return provisionCalls;
    }
}
```

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode`
Expected: all tests PASS — existing tests unchanged, new sentinel bindings fire but test provisioner is idempotent

- [ ] **Step 5: Commit**

```
git add app/src/main/resources/application.properties app/src/test/resources/application.properties app/src/test/java/io/casehub/life/app/engine/TestLifeReactiveWorkerProvisioner.java
git commit -m "feat(#37): add sentinel config + TestLifeReactiveWorkerProvisioner

Production config: 7 sentinel capabilities with heartbeat intervals.
Test alternative: tracks provisioning calls, skips Quartz and LLM.

Refs #37"
```

---

### Task 8: Protocol + CLAUDE.md updates

**Files:**
- Modify: `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: Tasks 1-7 (complete implementation)
- Produces: updated protocol, updated CLAUDE.md Layer 7 section

- [ ] **Step 1: Update PP-20260618-openclaw-agent**

Add life#37 to the refs list. Add a new section after the existing content:

```markdown
**Provisioner mode (life#37):** Sentinel capabilities use the provisioner path — no inline
worker exists, so the engine falls through to `LifeReactiveWorkerProvisioner`. The provisioner
registers the sentinel in `LifeSentinelRegistry` and schedules a Quartz `LifeHeartbeatJob`.
Each heartbeat tick: `CaseHubRuntime.query()` for fresh case context, `Agent.execute()` via
DirectCallBridge for structured result, `CaseHubRuntime.signal()` to deliver the result.
`LifeProvisionerCleanupObserver` terminates sentinels on `CaseLifecycleEvent` terminal states.
Sentinel capabilities must NEVER have inline workers registered — they are reserved for the
provisioner path. Uses life's own `LifeSentinelRegistry` (not `OpenClawAgentRegistry`).
```

- [ ] **Step 2: Update CLAUDE.md Layer 7 section**

Update the Layer 7 (full) entry from "PENDING (WorkerProvisioner heartbeat)" to "COMPLETE":

```
Layer 7 (full): + casehub-openclaw — OpenClaw as WorkerProvisioner; skill ecosystem.
         LifeReactiveWorkerProvisioner implements ReactiveWorkerProvisioner SPI.
         7 sentinel capabilities across all case plans. LifeSentinelRegistry tracks
         provisioned sentinels (supports concurrent same-agent cases). LifeHeartbeatJob
         (Quartz) invokes Agent.execute() periodically, signals results via
         CaseHubRuntime.signal(). LifeProvisionerCleanupObserver handles termination.
         LifeSentinelConfig maps capabilities to LifeAgent + heartbeat interval.
         ✅ COMPLETE
```

Add to the "Layer 7 additions" section in "What This Project Owns":

```
- `LifeSentinelRegistry` — `app/engine/` `@ApplicationScoped`; tracks provisioned sentinels
  by (caseId, capabilityName). CopyOnWriteArrayList per case. Supports concurrent same-agent
  cases (no 1:1 constraint). `isProvisioned()` provides idempotency guard for the provisioner.
- `LifeReactiveWorkerProvisioner` — `app/engine/` `@ApplicationScoped`; implements
  `ReactiveWorkerProvisioner`. Idempotent via `LifeSentinelRegistry`. Resolves `LifeAgent`
  from `LifeSentinelConfig`. Schedules Quartz `LifeHeartbeatJob` per sentinel. Displaces
  `NoOpReactiveWorkerProvisioner` (`@DefaultBean`).
- `LifeHeartbeatJob` — `app/engine/` `@ApplicationScoped` Quartz job. Each tick:
  `CaseHubRuntime.query()` → `Agent.execute()` → `CaseHubRuntime.signal("sentinelReport")`.
  Per-sentinel response schemas (7 records in `app/engine/agent/`).
- `LifeProvisionerCleanupObserver` — `app/engine/` `@ApplicationScoped`; observes
  `CaseLifecycleEvent` for terminal states (CaseCompleted, CaseFaulted, CaseCancelled).
  Calls `provisioner.terminateAllForCase()`.
- `LifeSentinelConfig` — `app/engine/` `@ConfigMapping(prefix="casehub.life.sentinel")`;
  maps capability name → LifeAgent + heartbeat interval.
- 7 sentinel response schemas — `app/engine/agent/` records: ContractorSentinelReport,
  MaintenanceSentinelReport, FollowUpSentinelReport, CareQualitySentinelReport,
  PatientStatusSentinelReport, AnomalySentinelReport, BookingSentinelReport.
```

- [ ] **Step 3: Commit protocol update**

```
git add docs/protocols/casehub-life/openclaw-agent-worker-pattern.md
git commit -m "docs(#37): update PP-20260618-openclaw-agent for provisioner mode

Refs #37"
```

- [ ] **Step 4: Commit CLAUDE.md update**

```
git add CLAUDE.md
git commit -m "docs(#37): update CLAUDE.md — Layer 7 complete

Refs #37"
```
