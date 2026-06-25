# Phase 2: Life Worker Conversions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert all 32 workers across 7 YamlCaseHubs from stubs/direct-LLM to real OpenClaw agents via the direct-call bridge. Delete the old `LifeOpenClawChatModelProvider` infrastructure. Update protocol.

**Architecture:** Each worker uses `Agent.builder().model(factory.forAgent("<agentId>"))` to get a `ChatModel` backed by `OpenClawAgentProvider` → `DirectCallBridge` → `/hooks/agent`. Response schemas enforce structured output via prompt-level schema injection. The test factory serves canned JSON responses keyed by system prompt content.

**Tech Stack:** Java 21, Quarkus 3.32.2, langchain4j-core 1.14.1, casehub-openclaw-core, casehub-openclaw-casehub

**Prerequisite:** Phase 1 (casehubio/openclaw#49) must be complete and SNAPSHOTs installed to local Maven repo.

**Design spec:** `specs/2026-06-24-hooks-agent-direct-call-design.md` (rev 5)

## Global Constraints

- Quarkus 3.32.2, Java 21 (on Java 26 JVM)
- `JAVA_HOME=$(/usr/libexec/java_home -v 26)` before all Maven commands
- `mvn install -pl api` before `mvn test -pl app` (api must be in local repo)
- Response schema records go in `io.casehub.life.app.engine.agent`
- System prompts describe persona + task; JSON format handled automatically by `OpenClawChatModel` schema injection
- AgentDescriptor follows `{model-family}:{persona}@{major}` convention
- Test factory matches on system prompt key phrases (not user message text)
- `casehub.life.tenancy-id=278776f9-e1b0-46fb-9032-8bddebdcf9ce` in test config
- All commits reference `Refs casehubio/life#38`

---

### Task 1: Infrastructure + AppointmentCycle migration

**Files:**
- Modify: `app/pom.xml` — add openclaw deps, remove langchain4j-open-ai
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/LifeOpenClawChatModelFactory.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/agent/TestLifeOpenClawChatModelFactory.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/AppointmentCycleCaseHub.java` — use factory
- Modify: `app/src/main/resources/application.properties` — update config
- Modify: `app/src/test/resources/application.properties` — update config + selected-alternatives
- Delete: `app/src/main/java/io/casehub/life/app/engine/agent/LifeOpenClawChatModelProvider.java`
- Delete: `app/src/test/java/io/casehub/life/app/engine/agent/TestLifeOpenClawChatModelProvider.java`
- Delete: `app/src/main/java/io/casehub/life/app/engine/agent/OpenClawHealthProbe.java`

**Interfaces:**
- Consumes: `DirectCallBridge`, `OpenClawHookClient`, `OpenClawAgentProvider`, `OpenClawChatModel` from casehub-openclaw-casehub (Phase 1)
- Produces: `LifeOpenClawChatModelFactory.forAgent(String openClawAgentId) → ChatModelProvider` — used by all subsequent tasks

- [ ] **Step 1: Add dependencies to app/pom.xml**

Add these dependencies (version managed by parent):
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-openclaw-core</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-openclaw-casehub</artifactId>
</dependency>
```

Remove this dependency:
```xml
<!-- DELETE: no longer calling /v1/chat/completions directly -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
    <scope>runtime</scope>
</dependency>
```

- [ ] **Step 2: Create LifeOpenClawChatModelFactory**

```java
package io.casehub.life.app.engine.agent;

import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.api.model.ai.ModelType;
import io.casehub.openclaw.casehub.DirectCallBridge;
import io.casehub.openclaw.casehub.OpenClawAgentProvider;
import io.casehub.openclaw.casehub.OpenClawChatModel;
import io.casehub.openclaw.client.OpenClawClientConfig;
import io.casehub.openclaw.client.OpenClawHookClient;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Duration;

@ApplicationScoped
public class LifeOpenClawChatModelFactory {

    private final DirectCallBridge bridge;
    private final OpenClawHookClient hookClient;
    private final String deliveryBaseUrl;
    private final int timeoutSeconds;

    @Inject
    public LifeOpenClawChatModelFactory(DirectCallBridge bridge,
                                         OpenClawHookClient hookClient,
                                         OpenClawClientConfig config) {
        this.bridge = bridge;
        this.hookClient = hookClient;
        this.deliveryBaseUrl = config.delivery().baseUrl();
        this.timeoutSeconds = config.agent().defaultTimeoutSeconds();
    }

    public ChatModelProvider forAgent(String openClawAgentId) {
        var provider = new OpenClawAgentProvider(
                bridge, hookClient, openClawAgentId, deliveryBaseUrl);
        var chatModel = new OpenClawChatModel(
                provider, Duration.ofSeconds(timeoutSeconds));
        return new ChatModelProvider() {
            @Override
            public ModelType type() {
                return ModelType.OPENAI;
            }

            @Override
            public dev.langchain4j.model.chat.ChatModel get() {
                return chatModel;
            }
        };
    }
}
```

- [ ] **Step 3: Create TestLifeOpenClawChatModelFactory**

```java
package io.casehub.life.app.engine.agent;

import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.api.model.ai.ModelType;
import io.casehub.openclaw.casehub.DirectCallBridge;
import io.casehub.openclaw.client.OpenClawClientConfig;
import io.casehub.openclaw.client.OpenClawHookClient;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import java.util.Map;

@Alternative
@Priority(10)
@ApplicationScoped
public class TestLifeOpenClawChatModelFactory extends LifeOpenClawChatModelFactory {

    private static final Map<String, String> RESPONSES = Map.ofEntries(
            // --- Health domain (health-agent) ---
            Map.entry("healthcare appointment booking",
                    "{\"appointmentId\":\"APT-MOCK\",\"provider\":\"Dr Smith\","
                    + "\"confirmed\":false,\"declined\":null,\"reason\":null}"),
            Map.entry("find an alternative",
                    "{\"alternativeFound\":true,\"appointmentId\":\"APT-ALT-MOCK\","
                    + "\"provider\":\"Dr Alternative\",\"confirmed\":false}"),
            Map.entry("send appointment confirmation",
                    "{\"confirmed\":true,\"reminderSent\":true}"),
            Map.entry("pre-visit preparation",
                    "{\"checklistSent\":true,\"instructions\":\"Bring ID, insurance card\"}"),
            Map.entry("record health decision",
                    "{\"recorded\":true,\"ledgerEntryId\":\"LEDGER-MOCK\"}"),
            Map.entry("assess care needs",
                    "{\"careLevel\":\"moderate\",\"recommendedFrequency\":\"weekly\","
                    + "\"specialRequirements\":[\"mobility support\"]}"),
            Map.entry("create a care plan",
                    "{\"schedule\":[\"Mon 9am\",\"Wed 2pm\"],\"duration\":\"2 hours\","
                    + "\"tasks\":[\"medication\",\"mobility exercises\"]}"),
            Map.entry("periodic health check",
                    "{\"reviewed\":true,\"healthConcern\":false,\"notes\":\"Stable condition\"}"),
            Map.entry("assess patient condition",
                    "{\"vitalSigns\":{\"bp\":\"120/80\",\"hr\":72,\"temp\":36.6},"
                    + "\"mobility\":\"assisted\",\"cognition\":\"alert\"}"),
            Map.entry("provide care",
                    "{\"tasksCompleted\":[\"medication\",\"mobility\"],\"duration\":\"90 min\","
                    + "\"observations\":\"Patient cooperative\"}"),

            // --- Home domain (home-agent) ---
            Map.entry("schedule a property inspection",
                    "{\"inspected\":true,\"condition\":\"good\",\"inspectionDate\":\"2026-07-01\"}"),
            Map.entry("gather contractor quotes",
                    "{\"quoteCount\":2,\"quotes\":[{\"contractor\":\"ABC\",\"amount\":500,"
                    + "\"available\":true},{\"contractor\":\"DEF\",\"amount\":650,\"available\":true}]}"),
            Map.entry("issue a commitment to the selected contractor",
                    "{\"commitmentIssued\":true,\"channel\":\"life/contractor/mock\"}"),
            Map.entry("monitor job progress",
                    "{\"progress\":\"75% complete\",\"estimatedCompletion\":\"2026-07-15\"}"),
            Map.entry("record job completion",
                    "{\"recorded\":true,\"ledgerEntryId\":\"LEDGER-MOCK\"}"),
            Map.entry("request a quote",
                    "{\"quoteRequested\":true,\"channel\":\"life/contractor/mock\","
                    + "\"deadlinePassed\":false}"),
            Map.entry("escalate an overdue",
                    "{\"escalated\":true,\"reminderSent\":true}"),
            Map.entry("process a received quote",
                    "{\"quoteAmount\":500,\"contractor\":\"ABC Plumbing\","
                    + "\"validUntil\":\"2026-07-30\"}"),
            Map.entry("monitor an active contractor job",
                    "{\"progress\":\"50% complete\",\"estimatedCompletion\":\"2026-07-20\"}"),
            Map.entry("record a contractor payment",
                    "{\"paymentRecorded\":true,\"amount\":500,\"ledgerEntryId\":\"LEDGER-MOCK\","
                    + "\"crossCaseSignal\":\"payment-complete\"}"),

            // --- Finance domain (finance-agent) ---
            Map.entry("gather financial data",
                    "{\"totalSpend\":2500,\"budgetLimit\":3000,"
                    + "\"categories\":[\"groceries\",\"utilities\",\"entertainment\"]}"),
            Map.entry("analyse spending anomalies",
                    "{\"hasAnomalies\":false,\"anomalyDetails\":\"No anomalies detected\"}"),
            Map.entry("escalate anomalies",
                    "{\"escalationSent\":true,\"channel\":\"life/oversight\"}"),
            Map.entry("process oversight response",
                    "{\"approved\":true,\"comments\":\"Approved by household admin\"}"),
            Map.entry("produce a monthly financial report",
                    "{\"reportGenerated\":true,\"summary\":\"Within budget\","
                    + "\"ledgerEntryId\":\"LEDGER-MOCK\"}"),

            // --- Travel domain (travel-agent) ---
            Map.entry("research destination options",
                    "{\"options\":[{\"name\":\"Paris\",\"cost\":1200,\"rating\":\"4.5\"},"
                    + "{\"name\":\"Barcelona\",\"cost\":900,\"rating\":\"4.3\"}]}"),
            Map.entry("search for flights",
                    "{\"flights\":[{\"airline\":\"BA\",\"price\":450,\"stops\":0},"
                    + "{\"airline\":\"RY\",\"price\":280,\"stops\":1}]}"),
            Map.entry("search for hotels",
                    "{\"hotels\":[{\"name\":\"Grand Hotel\",\"price\":120,\"rating\":4.5},"
                    + "{\"name\":\"Budget Inn\",\"price\":60,\"rating\":3.0}]}"),
            Map.entry("assess the total travel budget",
                    "{\"totalCost\":1500,\"requiresApproval\":false,\"isHighValue\":false}"),
            Map.entry("book the selected flights and hotels",
                    "{\"bookingRef\":\"BK-MOCK\",\"status\":\"confirmed\","
                    + "\"declined\":null,\"reason\":null}"),
            Map.entry("rebook after a declined",
                    "{\"bookingRef\":\"BK-REBK-MOCK\",\"status\":\"confirmed\","
                    + "\"alternativeDates\":true}"),
            Map.entry("confirm the travel itinerary",
                    "{\"confirmed\":true,\"itinerarySent\":true,"
                    + "\"confirmationRef\":\"CONF-MOCK\"}")
    );

    @SuppressWarnings("unused")
    protected TestLifeOpenClawChatModelFactory() {
        super(null, null, null);
    }

    @Override
    public ChatModelProvider forAgent(String openClawAgentId) {
        return new ChatModelProvider() {
            @Override
            public ModelType type() {
                return ModelType.OPENAI;
            }

            @Override
            public ChatModel get() {
                return new TestChatModel();
            }
        };
    }

    private static final class TestChatModel implements ChatModel {
        @Override
        public ChatResponse doChat(ChatRequest request) {
            String sysPrompt = request.messages().stream()
                    .filter(m -> m instanceof SystemMessage)
                    .map(m -> ((SystemMessage) m).text().toLowerCase())
                    .findFirst()
                    .orElse("");

            // Match decline path for appointment booking
            boolean decline = request.messages().stream()
                    .filter(m -> m instanceof dev.langchain4j.data.message.UserMessage)
                    .map(m -> ((dev.langchain4j.data.message.UserMessage) m).singleText())
                    .findFirst()
                    .map(t -> t.toLowerCase().contains("unavailable"))
                    .orElse(false);
            if (decline && sysPrompt.contains("appointment booking")) {
                return respond("{\"appointmentId\":null,\"provider\":\"Dr Gone\","
                        + "\"confirmed\":false,\"declined\":true,"
                        + "\"reason\":\"Provider unavailable\"}");
            }

            for (var entry : RESPONSES.entrySet()) {
                if (sysPrompt.contains(entry.getKey())) {
                    return respond(entry.getValue());
                }
            }

            return respond("{\"ok\":true}");
        }

        private static ChatResponse respond(String json) {
            return ChatResponse.builder()
                    .aiMessage(new AiMessage(json))
                    .build();
        }
    }
}
```

- [ ] **Step 4: Update production application.properties**

Replace the OpenClaw config section. Find:
```
#   casehub.life.openclaw.api-url   — OpenClaw /v1/chat/completions base URL
```
and the surrounding comment block, replace with:
```properties
# ============================================================
# OpenClaw configuration (casehub-openclaw-core/casehub)
# OpenClawClientConfig (@ConfigMapping prefix = "casehub.openclaw")
# casehub.openclaw.gateway.url — OpenClaw gateway base URL (required, no default)
# casehub.openclaw.gateway.bearer-token — API key for /hooks/agent auth (required)
# casehub.openclaw.delivery.base-url — webhook callback URL reachable from OpenClaw (required)
# casehub.openclaw.agent.default-timeout-seconds — default 120
# ============================================================
```

Also add the Quarkus REST client config for the OpenClaw gateway:
```properties
quarkus.rest-client.openclaw-gateway.url=${casehub.openclaw.gateway.url}
```

- [ ] **Step 5: Update test application.properties**

Replace the test OpenClaw config. Change:
```properties
casehub.life.openclaw.api-url=http://localhost:9999/v1
casehub.life.openclaw.api-key=test-key
casehub.life.openclaw.timeout-seconds=5
```
to:
```properties
casehub.openclaw.gateway.url=http://localhost:9999
casehub.openclaw.gateway.bearer-token=test-token
casehub.openclaw.delivery.base-url=http://localhost:8081
casehub.openclaw.agent.default-timeout-seconds=5
quarkus.rest-client.openclaw-gateway.url=http://localhost:9999
```

Update selected-alternatives — replace `TestLifeOpenClawChatModelProvider` with `TestLifeOpenClawChatModelFactory`:
```properties
quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaActorTrustScoreRepository,\
  io.casehub.persistence.memory.MemorySubCaseGroupRepository,\
  io.casehub.persistence.memory.MemoryPlanItemStore,\
  io.casehub.persistence.memory.MemoryReactivePlanItemStore,\
  io.casehub.life.app.engine.agent.TestLifeOpenClawChatModelFactory
```

Update the comment block above the config:
```properties
# ============================================================
# OpenClaw test config — TestLifeOpenClawChatModelFactory
# (@Alternative @Priority(10), registered in selected-alternatives above).
# Returns canned JSON responses keyed by system prompt content.
# casehub.life.tenancy-id is required (no default).
# ============================================================
```

- [ ] **Step 6: Convert AppointmentCycleCaseHub to use factory**

Replace `@Inject LifeOpenClawChatModelProvider openClaw` with `@Inject LifeOpenClawChatModelFactory openClawFactory`.

Rewrite `bookAppointmentWorker()` to use the factory:

```java
private Worker bookAppointmentWorker() {
    final Agent bookingAgent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a healthcare appointment booking agent for a UK household.
                    Book medical appointments with the requested provider.
                    If the provider is unavailable, set declined=true and provide a reason.""")
            .userMessage("Book a {{appointmentType}} appointment with provider {{provider}}.")
            .responseSchema(BookingResult.class)
            .build();

    final AgentDescriptor descriptor = new AgentDescriptor(
            "openclaw:health-agent@1", "OpenClaw Health Agent", "1",
            "openclaw", "openclaw", null, null, null, null, null,
            "casehubio/life/health", List.of(), null, "GB", null,
            tenancyId, "Health domain booking and follow-up agent");

    return Worker.builder()
            .name("book-appointment-agent")
            .capabilities(List.of(cap("book-appointment")))
            .function(bookingAgent)
            .agentDescriptor(descriptor)
            .build();
}
```

The import changes: remove `LifeOpenClawChatModelProvider`, add `LifeOpenClawChatModelFactory`.

- [ ] **Step 7: Delete old classes**

Delete these three files:
- `app/src/main/java/io/casehub/life/app/engine/agent/LifeOpenClawChatModelProvider.java`
- `app/src/test/java/io/casehub/life/app/engine/agent/TestLifeOpenClawChatModelProvider.java`
- `app/src/main/java/io/casehub/life/app/engine/agent/OpenClawHealthProbe.java`

- [ ] **Step 8: Build and test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=AppointmentCycleCaseHubTest --batch-mode
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=AppointmentCycleIntegrationTest --batch-mode
```

Expected: All PASS — existing behavior preserved with new factory

- [ ] **Step 9: Commit**

```
feat(#38): migrate to LifeOpenClawChatModelFactory — delete old provider infrastructure

Replace LifeOpenClawChatModelProvider with LifeOpenClawChatModelFactory backed
by OpenClawAgentProvider → DirectCallBridge → /hooks/agent. Delete
LifeOpenClawChatModelProvider, TestLifeOpenClawChatModelProvider, OpenClawHealthProbe.

Refs casehubio/life#38
```

---

### Task 2: Health domain — CareCoordination + CareEpisode workers

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/FindAlternativeResult.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/ConfirmAppointmentResult.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/PreVisitPrepResult.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/RecordHealthDecisionResult.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/NeedsAssessmentResult.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/CarePlanResult.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/HealthCheckResult.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/AssessPatientResult.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/ProvideCareResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/AppointmentCycleCaseHub.java` — convert 4 stub workers
- Modify: `app/src/main/java/io/casehub/life/app/engine/CareCoordinationCaseHub.java` — convert 3 workers
- Modify: `app/src/main/java/io/casehub/life/app/engine/CareEpisodeCaseHub.java` — convert 2 workers

**Interfaces:**
- Consumes: `LifeOpenClawChatModelFactory.forAgent("health-agent")` (Task 1)
- Produces: 9 response schema records + 9 AgentExec workers (AppointmentCycle stubs + CareCoordination + CareEpisode)

- [ ] **Step 1: Create response schema records**

```java
// FindAlternativeResult.java
package io.casehub.life.app.engine.agent;
public record FindAlternativeResult(boolean alternativeFound, String appointmentId,
                                     String provider, boolean confirmed) {}

// ConfirmAppointmentResult.java
package io.casehub.life.app.engine.agent;
public record ConfirmAppointmentResult(boolean confirmed, boolean reminderSent) {}

// PreVisitPrepResult.java
package io.casehub.life.app.engine.agent;
public record PreVisitPrepResult(boolean checklistSent, String instructions) {}

// RecordHealthDecisionResult.java
package io.casehub.life.app.engine.agent;
public record RecordHealthDecisionResult(boolean recorded, String ledgerEntryId) {}

// NeedsAssessmentResult.java
package io.casehub.life.app.engine.agent;
import java.util.List;
public record NeedsAssessmentResult(String careLevel, String recommendedFrequency,
                                     List<String> specialRequirements) {}

// CarePlanResult.java
package io.casehub.life.app.engine.agent;
import java.util.List;
public record CarePlanResult(List<String> schedule, String duration, List<String> tasks) {}

// HealthCheckResult.java
package io.casehub.life.app.engine.agent;
public record HealthCheckResult(boolean reviewed, boolean healthConcern, String notes) {}

// AssessPatientResult.java
package io.casehub.life.app.engine.agent;
public record AssessPatientResult(VitalSigns vitalSigns, String mobility, String cognition) {
    public record VitalSigns(String bp, int hr, double temp) {}
}

// ProvideCareResult.java
package io.casehub.life.app.engine.agent;
import java.util.List;
public record ProvideCareResult(List<String> tasksCompleted, String duration,
                                 String observations) {}
```

- [ ] **Step 2: Convert AppointmentCycleCaseHub remaining 4 stub workers**

Replace the 4 stub methods with AgentExec workers. Each injects `openClawFactory` (already done in Task 1) and uses `factory.forAgent("health-agent")`.

```java
private Worker findAlternativeWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a healthcare appointment agent. Find an alternative provider
                    after a booking was declined. Search available providers and propose
                    an alternative appointment.""")
            .responseSchema(FindAlternativeResult.class)
            .build();

    return Worker.builder()
            .name("find-alternative-agent")
            .capabilities(List.of(cap("find-alternative")))
            .function(agent)
            .agentDescriptor(healthDescriptor())
            .build();
}

private Worker confirmAppointmentWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a healthcare appointment agent. Send appointment confirmation
                    to the patient and schedule a reminder for 24 hours before.""")
            .responseSchema(ConfirmAppointmentResult.class)
            .build();

    return Worker.builder()
            .name("confirm-appointment-agent")
            .capabilities(List.of(cap("confirm-appointment")))
            .function(agent)
            .agentDescriptor(healthDescriptor())
            .build();
}

private Worker preVisitPrepWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a healthcare appointment agent. Send pre-visit preparation
                    checklist and instructions to the patient.""")
            .responseSchema(PreVisitPrepResult.class)
            .build();

    return Worker.builder()
            .name("pre-visit-prep-agent")
            .capabilities(List.of(cap("pre-visit-prep")))
            .function(agent)
            .agentDescriptor(healthDescriptor())
            .build();
}

private Worker recordHealthDecisionWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a healthcare records agent. Record health decision outcomes
                    to the tamper-evident ledger.""")
            .responseSchema(RecordHealthDecisionResult.class)
            .build();

    return Worker.builder()
            .name("record-health-decision-agent")
            .capabilities(List.of(cap("record-health-decision")))
            .function(agent)
            .agentDescriptor(healthDescriptor())
            .build();
}
```

Extract a shared `healthDescriptor()` method to avoid repeating the AgentDescriptor for all health workers:

```java
private AgentDescriptor healthDescriptor() {
    return new AgentDescriptor(
            "openclaw:health-agent@1", "OpenClaw Health Agent", "1",
            "openclaw", "openclaw", null, null, null, null, null,
            "casehubio/life/health", List.of(), null, "GB", null,
            tenancyId, "Health domain agent");
}
```

Update imports: add all new result types.

- [ ] **Step 3: Convert CareCoordinationCaseHub**

Add `@Inject LifeOpenClawChatModelFactory openClawFactory` and `@ConfigProperty(name = "casehub.life.tenancy-id") String tenancyId`.

Replace all 3 stub workers:

```java
private Worker needsAssessmentWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a care coordination agent. Assess care needs for the patient,
                    determining care level, recommended frequency, and any special requirements.""")
            .responseSchema(NeedsAssessmentResult.class)
            .build();

    return Worker.builder()
            .name("needs-assessment-agent")
            .capabilities(List.of(cap("needs-assessment")))
            .function(agent)
            .agentDescriptor(healthDescriptor())
            .build();
}

private Worker carePlanWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a care coordination agent. Create a care plan with schedule,
                    duration, and task list based on the needs assessment.""")
            .responseSchema(CarePlanResult.class)
            .build();

    return Worker.builder()
            .name("care-plan-agent")
            .capabilities(List.of(cap("care-plan")))
            .function(agent)
            .agentDescriptor(healthDescriptor())
            .build();
}

private Worker healthCheckWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a care coordination agent. Perform a periodic health check,
                    reviewing the patient's condition and flagging any concerns.""")
            .responseSchema(HealthCheckResult.class)
            .build();

    return Worker.builder()
            .name("health-check-agent")
            .capabilities(List.of(cap("health-check")))
            .function(agent)
            .agentDescriptor(healthDescriptor())
            .build();
}

private AgentDescriptor healthDescriptor() {
    return new AgentDescriptor(
            "openclaw:health-agent@1", "OpenClaw Health Agent", "1",
            "openclaw", "openclaw", null, null, null, null, null,
            "casehubio/life/health", List.of(), null, "GB", null,
            tenancyId, "Health domain agent");
}
```

Add double-checked lock `getDefinition()` override (same pattern as AppointmentCycleCaseHub).

- [ ] **Step 4: Convert CareEpisodeCaseHub**

Same pattern. Add factory + tenancyId injection. Replace 2 stub workers:

```java
private Worker assessPatientWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a care episode agent. Assess patient condition including
                    vital signs, mobility status, and cognitive state.""")
            .responseSchema(AssessPatientResult.class)
            .build();

    return Worker.builder()
            .name("assess-patient-agent")
            .capabilities(List.of(cap("assess-patient")))
            .function(agent)
            .agentDescriptor(healthDescriptor())
            .build();
}

private Worker provideCareWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("health-agent"))
            .systemPrompt("""
                    You are a care episode agent. Provide care to the patient, completing
                    assigned tasks and recording observations.""")
            .responseSchema(ProvideCareResult.class)
            .build();

    return Worker.builder()
            .name("provide-care-agent")
            .capabilities(List.of(cap("provide-care")))
            .function(agent)
            .agentDescriptor(healthDescriptor())
            .build();
}
```

- [ ] **Step 5: Run health domain tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="AppointmentCycleCaseHubTest,AppointmentCycleIntegrationTest,CareCoordinationCaseHubTest,CareCoordinationIntegrationTest,CareEpisodeCaseHubTest,CareEpisodeIntegrationTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All PASS

- [ ] **Step 6: Commit**

```
feat(#38): convert health domain workers to AgentExec — 9 stubs replaced

AppointmentCycle (4 stubs), CareCoordination (3), CareEpisode (2) now use
OpenClawAgentProvider via LifeOpenClawChatModelFactory.forAgent("health-agent").

Refs casehubio/life#38
```

---

### Task 3: Home domain — HomeMaintenance + ContractorCoordination

**Files:**
- Create: 10 response schema records in `app/src/main/java/io/casehub/life/app/engine/agent/`
- Modify: `HomeMaintenanceCaseHub.java` — convert 5 workers
- Modify: `ContractorCoordinationCaseHub.java` — convert 5 workers

**Interfaces:**
- Consumes: `LifeOpenClawChatModelFactory.forAgent("home-agent")` (Task 1)
- Produces: 10 response schema records + 10 AgentExec workers

- [ ] **Step 1: Create response schema records**

```java
// ScheduleInspectionResult.java
package io.casehub.life.app.engine.agent;
public record ScheduleInspectionResult(boolean inspected, String condition,
                                        String inspectionDate) {}

// GetQuotesResult.java
package io.casehub.life.app.engine.agent;
import java.util.List;
public record GetQuotesResult(int quoteCount, List<QuoteItem> quotes) {
    public record QuoteItem(String contractor, int amount, boolean available) {}
}

// IssueCommitmentResult.java
package io.casehub.life.app.engine.agent;
public record IssueCommitmentResult(boolean commitmentIssued, String channel) {}

// MonitorJobResult.java
package io.casehub.life.app.engine.agent;
public record MonitorJobResult(String progress, String estimatedCompletion) {}

// RecordCompletionResult.java
package io.casehub.life.app.engine.agent;
public record RecordCompletionResult(boolean recorded, String ledgerEntryId) {}

// RequestQuoteResult.java
package io.casehub.life.app.engine.agent;
public record RequestQuoteResult(boolean quoteRequested, String channel,
                                  boolean deadlinePassed) {}

// WatchdogEscalationResult.java
package io.casehub.life.app.engine.agent;
public record WatchdogEscalationResult(boolean escalated, boolean reminderSent) {}

// QuoteReceivedResult.java
package io.casehub.life.app.engine.agent;
public record QuoteReceivedResult(int quoteAmount, String contractor,
                                   String validUntil) {}

// JobMonitoringResult.java
package io.casehub.life.app.engine.agent;
public record JobMonitoringResult(String progress, String estimatedCompletion) {}

// RecordPaymentResult.java
package io.casehub.life.app.engine.agent;
public record RecordPaymentResult(boolean paymentRecorded, int amount,
                                   String ledgerEntryId, String crossCaseSignal) {}
```

- [ ] **Step 2: Convert HomeMaintenanceCaseHub**

Add `@Inject LifeOpenClawChatModelFactory openClawFactory` and `@ConfigProperty(name = "casehub.life.tenancy-id") String tenancyId`. Add double-checked lock `getDefinition()`. Replace 5 stub workers:

```java
private Worker scheduleInspectionWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a home maintenance agent. Schedule a property inspection,
                    assess the condition, and report findings.""")
            .responseSchema(ScheduleInspectionResult.class)
            .build();
    return Worker.builder().name("schedule-inspection-agent")
            .capabilities(List.of(cap("schedule-inspection")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}

private Worker getQuotesWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a home maintenance agent. Gather contractor quotes for the
                    required maintenance work.""")
            .responseSchema(GetQuotesResult.class)
            .build();
    return Worker.builder().name("get-quotes-agent")
            .capabilities(List.of(cap("get-quotes")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}

private Worker issueCommitmentWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a home maintenance agent. Issue a commitment to the selected
                    contractor for the approved work.""")
            .responseSchema(IssueCommitmentResult.class)
            .build();
    return Worker.builder().name("issue-commitment-agent")
            .capabilities(List.of(cap("issue-commitment")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}

private Worker monitorJobWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a home maintenance agent. Monitor job progress and report
                    estimated completion.""")
            .responseSchema(MonitorJobResult.class)
            .build();
    return Worker.builder().name("monitor-job-agent")
            .capabilities(List.of(cap("monitor-job")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}

private Worker recordCompletionWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a home maintenance agent. Record job completion to the
                    tamper-evident ledger.""")
            .responseSchema(RecordCompletionResult.class)
            .build();
    return Worker.builder().name("record-completion-agent")
            .capabilities(List.of(cap("record-completion")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}

private AgentDescriptor homeDescriptor() {
    return new AgentDescriptor(
            "openclaw:home-agent@1", "OpenClaw Home Agent", "1",
            "openclaw", "openclaw", null, null, null, null, null,
            "casehubio/life/household", List.of(), null, "GB", null,
            tenancyId, "Household maintenance agent");
}
```

- [ ] **Step 3: Convert ContractorCoordinationCaseHub**

Same pattern with `home-agent`. Replace 5 stub workers:

```java
private Worker requestQuoteWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a contractor coordination agent. Request a quote from the
                    contractor via the appropriate messaging channel.""")
            .responseSchema(RequestQuoteResult.class)
            .build();
    return Worker.builder().name("request-quote-agent")
            .capabilities(List.of(cap("request-quote")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}

private Worker watchdogEscalationWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a contractor coordination agent. Escalate an overdue
                    contractor commitment by sending a reminder.""")
            .responseSchema(WatchdogEscalationResult.class)
            .build();
    return Worker.builder().name("watchdog-escalation-agent")
            .capabilities(List.of(cap("watchdog-escalation")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}

private Worker quoteReceivedWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a contractor coordination agent. Process a received quote,
                    extracting amount, contractor details, and validity period.""")
            .responseSchema(QuoteReceivedResult.class)
            .build();
    return Worker.builder().name("quote-received-agent")
            .capabilities(List.of(cap("quote-received")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}

private Worker jobMonitoringWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a contractor coordination agent. Monitor an active contractor
                    job and report progress.""")
            .responseSchema(JobMonitoringResult.class)
            .build();
    return Worker.builder().name("job-monitoring-agent")
            .capabilities(List.of(cap("job-monitoring")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}

private Worker recordPaymentWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("home-agent"))
            .systemPrompt("""
                    You are a contractor coordination agent. Record a contractor payment
                    to the tamper-evident ledger and emit a cross-case signal.""")
            .responseSchema(RecordPaymentResult.class)
            .build();
    return Worker.builder().name("record-payment-agent")
            .capabilities(List.of(cap("record-payment")))
            .function(agent).agentDescriptor(homeDescriptor()).build();
}
```

- [ ] **Step 4: Run home domain tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="HomeMaintenanceCaseHubTest,HomeMaintenanceIntegrationTest,ContractorCoordinationCaseHubTest,ContractorCoordinationIntegrationTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All PASS

- [ ] **Step 5: Commit**

```
feat(#38): convert home domain workers to AgentExec — 10 stubs replaced

HomeMaintenance (5), ContractorCoordination (5) now use
OpenClawAgentProvider via LifeOpenClawChatModelFactory.forAgent("home-agent").

Refs casehubio/life#38
```

---

### Task 4: Finance domain — FinancialReview

**Files:**
- Create: 5 response schema records
- Modify: `FinancialReviewCaseHub.java` — convert 5 workers

**Interfaces:**
- Consumes: `LifeOpenClawChatModelFactory.forAgent("finance-agent")` (Task 1)
- Produces: 5 response schema records + 5 AgentExec workers

- [ ] **Step 1: Create response schema records**

```java
// GatherDataResult.java
package io.casehub.life.app.engine.agent;
import java.util.List;
public record GatherDataResult(int totalSpend, int budgetLimit,
                                List<String> categories) {}

// AnalyseAnomaliesResult.java
package io.casehub.life.app.engine.agent;
public record AnalyseAnomaliesResult(boolean hasAnomalies, String anomalyDetails) {}

// EscalateAnomaliesResult.java
package io.casehub.life.app.engine.agent;
public record EscalateAnomaliesResult(boolean escalationSent, String channel) {}

// OversightResponseResult.java
package io.casehub.life.app.engine.agent;
public record OversightResponseResult(boolean approved, String comments) {}

// ProduceReportResult.java
package io.casehub.life.app.engine.agent;
public record ProduceReportResult(boolean reportGenerated, String summary,
                                   String ledgerEntryId) {}
```

- [ ] **Step 2: Convert FinancialReviewCaseHub**

Add factory + tenancyId injection. Add double-checked lock. Replace 5 stub workers:

```java
private Worker gatherDataWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("finance-agent"))
            .systemPrompt("""
                    You are a financial review agent. Gather financial data by aggregating
                    transactions across all linked accounts.""")
            .responseSchema(GatherDataResult.class)
            .build();
    return Worker.builder().name("gather-data-agent")
            .capabilities(List.of(cap("gather-data")))
            .function(agent).agentDescriptor(financeDescriptor()).build();
}

private Worker analyseAnomaliesWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("finance-agent"))
            .systemPrompt("""
                    You are a financial review agent. Analyse spending anomalies by
                    comparing current spending patterns against budget limits.""")
            .responseSchema(AnalyseAnomaliesResult.class)
            .build();
    return Worker.builder().name("analyse-anomalies-agent")
            .capabilities(List.of(cap("analyse-anomalies")))
            .function(agent).agentDescriptor(financeDescriptor()).build();
}

private Worker escalateAnomaliesWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("finance-agent"))
            .systemPrompt("""
                    You are a financial review agent. Escalate anomalies to the oversight
                    channel for human review.""")
            .responseSchema(EscalateAnomaliesResult.class)
            .build();
    return Worker.builder().name("escalate-anomalies-agent")
            .capabilities(List.of(cap("escalate-anomalies")))
            .function(agent).agentDescriptor(financeDescriptor()).build();
}

private Worker oversightResponseWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("finance-agent"))
            .systemPrompt("""
                    You are a financial review agent. Process oversight response from
                    the household admin regarding flagged anomalies.""")
            .responseSchema(OversightResponseResult.class)
            .build();
    return Worker.builder().name("oversight-response-agent")
            .capabilities(List.of(cap("oversight-response")))
            .function(agent).agentDescriptor(financeDescriptor()).build();
}

private Worker produceReportWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("finance-agent"))
            .systemPrompt("""
                    You are a financial review agent. Produce a monthly financial report
                    summarising spending and recording it to the ledger.""")
            .responseSchema(ProduceReportResult.class)
            .build();
    return Worker.builder().name("produce-report-agent")
            .capabilities(List.of(cap("produce-report")))
            .function(agent).agentDescriptor(financeDescriptor()).build();
}

private AgentDescriptor financeDescriptor() {
    return new AgentDescriptor(
            "openclaw:finance-agent@1", "OpenClaw Finance Agent", "1",
            "openclaw", "openclaw", null, null, null, null, null,
            "casehubio/life/finance", List.of(), null, "GB", null,
            tenancyId, "Financial review and governance agent");
}
```

- [ ] **Step 3: Run finance domain tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="FinancialReviewCaseHubTest,FinancialReviewIntegrationTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All PASS

- [ ] **Step 4: Commit**

```
feat(#38): convert finance domain workers to AgentExec — 5 stubs replaced

FinancialReview (5) now uses OpenClawAgentProvider via
LifeOpenClawChatModelFactory.forAgent("finance-agent").

Refs casehubio/life#38
```

---

### Task 5: Travel domain — TravelPlan

**Files:**
- Create: 7 response schema records
- Modify: `TravelPlanCaseHub.java` — convert 7 workers

**Interfaces:**
- Consumes: `LifeOpenClawChatModelFactory.forAgent("travel-agent")` (Task 1)
- Produces: 7 response schema records + 7 AgentExec workers

- [ ] **Step 1: Create response schema records**

```java
// DestinationResearchResult.java
package io.casehub.life.app.engine.agent;
import java.util.List;
public record DestinationResearchResult(List<DestinationOption> options) {
    public record DestinationOption(String name, int cost, String rating) {}
}

// FlightSearchResult.java
package io.casehub.life.app.engine.agent;
import java.util.List;
public record FlightSearchResult(List<FlightOption> flights) {
    public record FlightOption(String airline, int price, int stops) {}
}

// HotelSearchResult.java
package io.casehub.life.app.engine.agent;
import java.util.List;
public record HotelSearchResult(List<HotelOption> hotels) {
    public record HotelOption(String name, int price, double rating) {}
}

// BudgetAssessmentResult.java
package io.casehub.life.app.engine.agent;
public record BudgetAssessmentResult(int totalCost, boolean requiresApproval,
                                      boolean isHighValue) {}

// TravelBookingResult.java
package io.casehub.life.app.engine.agent;
public record TravelBookingResult(String bookingRef, String status,
                                   Boolean declined, String reason) {}

// RebookingResult.java
package io.casehub.life.app.engine.agent;
public record RebookingResult(String bookingRef, String status,
                               boolean alternativeDates) {}

// ConfirmationResult.java
package io.casehub.life.app.engine.agent;
public record ConfirmationResult(boolean confirmed, boolean itinerarySent,
                                  String confirmationRef) {}
```

- [ ] **Step 2: Convert TravelPlanCaseHub**

Add factory + tenancyId injection. Add double-checked lock. Replace 7 stub workers:

```java
private Worker destinationResearchWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("travel-agent"))
            .systemPrompt("""
                    You are a travel planning agent. Research destination options with
                    costs and ratings.""")
            .responseSchema(DestinationResearchResult.class)
            .build();
    return Worker.builder().name("destination-research-agent")
            .capabilities(List.of(cap("destination-research")))
            .function(agent).agentDescriptor(travelDescriptor()).build();
}

private Worker flightSearchWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("travel-agent"))
            .systemPrompt("""
                    You are a travel planning agent. Search for flights with airline,
                    price, and number of stops.""")
            .responseSchema(FlightSearchResult.class)
            .build();
    return Worker.builder().name("flight-search-agent")
            .capabilities(List.of(cap("flight-search")))
            .function(agent).agentDescriptor(travelDescriptor()).build();
}

private Worker hotelSearchWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("travel-agent"))
            .systemPrompt("""
                    You are a travel planning agent. Search for hotels with name,
                    price, and rating.""")
            .responseSchema(HotelSearchResult.class)
            .build();
    return Worker.builder().name("hotel-search-agent")
            .capabilities(List.of(cap("hotel-search")))
            .function(agent).agentDescriptor(travelDescriptor()).build();
}

private Worker budgetAssessmentWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("travel-agent"))
            .systemPrompt("""
                    You are a travel planning agent. Assess the total travel budget
                    and determine if approval is required.""")
            .responseSchema(BudgetAssessmentResult.class)
            .build();
    return Worker.builder().name("budget-assessment-agent")
            .capabilities(List.of(cap("budget-assessment")))
            .function(agent).agentDescriptor(travelDescriptor()).build();
}

private Worker bookingWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("travel-agent"))
            .systemPrompt("""
                    You are a travel planning agent. Book the selected flights and hotels.
                    If booking fails, set declined=true with a reason.""")
            .responseSchema(TravelBookingResult.class)
            .build();
    return Worker.builder().name("booking-agent")
            .capabilities(List.of(cap("booking")))
            .function(agent).agentDescriptor(travelDescriptor()).build();
}

private Worker rebookingWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("travel-agent"))
            .systemPrompt("""
                    You are a travel planning agent. Rebook after a declined booking,
                    finding alternative dates.""")
            .responseSchema(RebookingResult.class)
            .build();
    return Worker.builder().name("rebooking-agent")
            .capabilities(List.of(cap("rebooking")))
            .function(agent).agentDescriptor(travelDescriptor()).build();
}

private Worker confirmationWorker() {
    final Agent agent = Agent.builder()
            .model(openClawFactory.forAgent("travel-agent"))
            .systemPrompt("""
                    You are a travel planning agent. Confirm the travel itinerary and
                    send confirmation details.""")
            .responseSchema(ConfirmationResult.class)
            .build();
    return Worker.builder().name("confirmation-agent")
            .capabilities(List.of(cap("confirmation")))
            .function(agent).agentDescriptor(travelDescriptor()).build();
}

private AgentDescriptor travelDescriptor() {
    return new AgentDescriptor(
            "openclaw:travel-agent@1", "OpenClaw Travel Agent", "1",
            "openclaw", "openclaw", null, null, null, null, null,
            "casehubio/life/travel", List.of(), null, "GB", null,
            tenancyId, "Travel planning and booking agent");
}
```

- [ ] **Step 3: Run travel domain tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="TravelPlanCaseHubTest,TravelPlanIntegrationTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All PASS

- [ ] **Step 4: Commit**

```
feat(#38): convert travel domain workers to AgentExec — 7 stubs replaced

TravelPlan (7) now uses OpenClawAgentProvider via
LifeOpenClawChatModelFactory.forAgent("travel-agent").

Refs casehubio/life#38
```

---

### Task 6: Protocol update + full verification

**Files:**
- Modify: `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`

**Interfaces:**
- Consumes: All previous tasks
- Produces: Updated protocol, verified green test suite

- [ ] **Step 1: Update PP-20260618-openclaw-agent protocol**

Update the protocol to reflect the new architecture:
- Replace references to `LifeOpenClawChatModelProvider` with `LifeOpenClawChatModelFactory`
- Replace `OpenClawHealthProbe` documentation with note that health probes are no longer needed (gateway client handles connectivity)
- Replace `casehub.life.openclaw.api-url` config references with `casehub.openclaw.gateway.url` (standard config)
- Update "chatModelProvider.get() is called once in Agent.build()" to describe the factory pattern: `factory.forAgent("<agentId>")` creates a per-agent `ChatModelProvider` backed by `OpenClawAgentProvider` → `DirectCallBridge` → `/hooks/agent`
- Add note: "Config changes require restart" still applies — the factory creates the ChatModel once per `forAgent()` call, which happens during `augment()` (double-checked lock, once per JVM lifetime)

- [ ] **Step 2: Run the full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install --batch-mode
```

Expected: BUILD SUCCESS — all tests pass across api and app modules

- [ ] **Step 3: Verify test count**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode 2>&1 | tail -5
```

Expected: 286+ tests, 0 failures, 0 errors, 0 skipped

- [ ] **Step 4: Commit**

```
docs(#38): update PP-20260618-openclaw-agent for factory pattern

References to deleted LifeOpenClawChatModelProvider, OpenClawHealthProbe,
and casehub.life.openclaw.* config replaced with current architecture.

Refs casehubio/life#38
```
