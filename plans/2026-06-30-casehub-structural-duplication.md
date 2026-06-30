# CaseHub Structural Duplication Extraction — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract structural duplication across 7 CaseHub classes by creating a `LifeTypedCaseHub` base class with template method pattern, migrating 6 CaseHubs to it, and fixing CareEpisodeCaseHub separately.

**Architecture:** `LifeTypedCaseHub extends YamlCaseHub` holds shared CDI injections, makes `augment()` final (template method calling `configureCase()` then registering descriptors), and provides `agentWorker()` convenience. Subclasses override `configureCase()` to add workers. CareEpisodeCaseHub stays on `YamlCaseHub` (no `LifeCaseType`, spawned only as sub-case).

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-api 0.2-SNAPSHOT (engine#591)

**Spec:** `specs/2026-06-30-casehub-structural-duplication-design.md` (design-reviewed, 4 rounds, 12/12 issues resolved)

## Global Constraints

- Engine#591 made `YamlCaseHub.getDefinition()` final — subclasses MUST NOT override it
- `Worker.Builder` uses `capabilityName(String)` — `capabilities(List<Capability>)` no longer exists
- Worker name convention: `{capabilityName}-agent` — enforced by `agentWorker()` helper
- `FamilyVoteCaseHub` — unchanged (no augmentation, no agent)
- `LifeCaseService` — NOT changed this issue (switch elimination deferred to life#27)
- All files in `app/src/main/java/io/casehub/life/app/engine/`

---

### Task 1: Create `LifeTypedCaseHub` base class + unit test

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeTypedCaseHub.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeTypedCaseHubTest.java`

**Interfaces:**
- Consumes: `YamlCaseHub.augment(CaseDefinition)` from engine-api, `LifeOpenClawChatModelFactory.forAgent(LifeAgent)`, `LifeAgentDescriptorFactory.descriptorFor(LifeAgent)`, `LifeCaseType` enum from api/
- Produces: `LifeTypedCaseHub` abstract class — Task 2 subclasses extend it; `agentWorker(String, String, Class<?>)` returns `Worker`; `configureCase(CaseDefinition)` hook; `lifeCaseType()` abstract method; `agent()` protected getter

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.life.app.engine;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.life.api.LifeCaseType;
import io.casehub.life.app.engine.agent.LifeAgentDescriptorFactory;
import io.casehub.life.app.engine.agent.LifeOpenClawChatModelFactory;
import io.casehub.worker.api.Worker;
import java.util.Map;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class LifeTypedCaseHubTest {

    private LifeOpenClawChatModelFactory mockFactory;
    private LifeAgentDescriptorFactory mockDescriptorFactory;

    @BeforeEach
    void setUp() {
        mockFactory = mock(LifeOpenClawChatModelFactory.class);
        when(mockFactory.forAgent(any())).thenReturn(mock(ChatModelProvider.class));
        mockDescriptorFactory = mock(LifeAgentDescriptorFactory.class);
        when(mockDescriptorFactory.descriptorFor(any()))
                .thenReturn(mock(io.casehub.eidos.api.AgentDescriptor.class));
    }

    @Test
    void agentWorkerProducesCorrectNameAndCapability() {
        var hub = createHub();
        Worker worker = hub.agentWorker("test-cap", "Test system prompt", Map.class);
        assertThat(worker.name()).isEqualTo("test-cap-agent");
        assertThat(worker.capabilityNames()).containsExactly("test-cap");
        assertThat(worker.function()).isInstanceOf(AgentWorkerFunction.class);
    }

    @Test
    void lifeCaseTypeReturnsExpectedValue() {
        var hub = createHub();
        assertThat(hub.lifeCaseType()).isEqualTo(LifeCaseType.APPOINTMENT_CYCLE);
    }

    @Test
    void agentGetterReturnsConstructorValue() {
        var hub = createHub();
        assertThat(hub.agent()).isEqualTo(LifeAgent.HEALTH);
    }

    private TestCaseHub createHub() {
        var hub = new TestCaseHub();
        hub.openClawFactory = mockFactory;
        hub.descriptorFactory = mockDescriptorFactory;
        return hub;
    }

    static class TestCaseHub extends LifeTypedCaseHub {
        TestCaseHub() {
            super("life/appointment-cycle.yaml", LifeAgent.HEALTH);
        }

        @Override
        public LifeCaseType lifeCaseType() {
            return LifeCaseType.APPOINTMENT_CYCLE;
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeTypedCaseHubTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: FAIL — `LifeTypedCaseHub` does not exist

- [ ] **Step 3: Write LifeTypedCaseHub implementation**

```java
package io.casehub.life.app.engine;

import io.casehub.api.engine.YamlCaseHub;
import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ai.Agent;
import io.casehub.life.api.LifeCaseType;
import io.casehub.life.app.engine.agent.LifeAgentDescriptorFactory;
import io.casehub.life.app.engine.agent.LifeOpenClawChatModelFactory;
import io.casehub.worker.api.Worker;
import jakarta.inject.Inject;
import java.util.Map;

public abstract class LifeTypedCaseHub extends YamlCaseHub {

    @Inject
    LifeOpenClawChatModelFactory openClawFactory;

    @Inject
    LifeAgentDescriptorFactory descriptorFactory;

    private final LifeAgent agent;

    protected LifeTypedCaseHub(String path, LifeAgent agent) {
        super(path);
        this.agent = agent;
    }

    public abstract LifeCaseType lifeCaseType();

    protected LifeAgent agent() {
        return agent;
    }

    @Override
    protected final void augment(CaseDefinition definition) {
        configureCase(definition);
        definition.setAgentDescriptors(Map.of(
                agent.agentId(), descriptorFactory.descriptorFor(agent)));
    }

    protected void configureCase(CaseDefinition definition) {
    }

    protected Worker agentWorker(String capabilityName, String systemPrompt,
                                  Class<?> responseSchema) {
        Agent a = Agent.builder()
                .model(openClawFactory.forAgent(agent))
                .systemPrompt(systemPrompt)
                .responseSchema(responseSchema)
                .build();
        return Worker.builder()
                .name(capabilityName + "-agent")
                .capabilityName(capabilityName)
                .function(new AgentWorkerFunction(a))
                .build();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeTypedCaseHubTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`

Note: This may fail if the full app module can't compile (existing CaseHubs still override `final getDefinition()`). If so, proceed directly to Task 2 — all tests run after migration.

---

### Task 2: Migrate all CaseHubs + full build

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/AppointmentCycleCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/CareCoordinationCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/ContractorCoordinationCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/FinancialReviewCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/HomeMaintenanceCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/TravelPlanCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/CareEpisodeCaseHub.java`
- Modify: 6 existing `*CaseHubTest.java` files (add `lifeCaseType()` assertion)

**Interfaces:**
- Consumes: `LifeTypedCaseHub` from Task 1
- Produces: All CaseHubs compile against engine 0.2-SNAPSHOT; behavior unchanged

**Migration pattern (applies to 6 CaseHubs extending LifeTypedCaseHub):**

Each migrated CaseHub:
1. Extends `LifeTypedCaseHub` instead of `YamlCaseHub`
2. Removes: `volatile CaseDefinition augmentedDefinition`, `getDefinition()` override, `cap()` helper, `@Inject openClawFactory`, `@Inject descriptorFactory`, `setAgentDescriptors(...)` call, all individual worker methods
3. Adds: `lifeCaseType()` returning the matching `LifeCaseType` enum value
4. Adds: `configureCase(CaseDefinition)` override with `agentWorker()` calls replacing per-worker methods
5. Constructor passes YAML path + `LifeAgent` to super

- [ ] **Step 1: Migrate AppointmentCycleCaseHub**

Replace the entire class body. One worker (`book-appointment-agent`) needs manual construction due to `userMessage`. All others use `agentWorker()`. Preserve existing system prompts and response schema classes exactly.

Key: `super("life/appointment-cycle.yaml", LifeAgent.HEALTH)`, `lifeCaseType() → APPOINTMENT_CYCLE`

Workers: `book-appointment` (manual — has userMessage), `find-alternative`, `confirm-appointment`, `pre-visit-prep`, `record-health-decision`

- [ ] **Step 2: Migrate CareCoordinationCaseHub**

`super("life/care-coordination.yaml", LifeAgent.HEALTH)`, `lifeCaseType() → CARE_COORDINATION`

Workers: `needs-assessment`, `care-plan`, `health-check`

- [ ] **Step 3: Migrate ContractorCoordinationCaseHub**

`super("life/contractor-coordination.yaml", LifeAgent.HOME)`, `lifeCaseType() → CONTRACTOR_COORDINATION`

Workers: `request-quote`, `watchdog-escalation`, `quote-received`, `job-monitoring`, `record-payment`

- [ ] **Step 4: Migrate FinancialReviewCaseHub**

`super("life/financial-review.yaml", LifeAgent.FINANCE)`, `lifeCaseType() → FINANCIAL_REVIEW`

Workers: `gather-data`, `analyse-anomalies`, `escalate-anomalies`, `oversight-response`, `produce-report`

- [ ] **Step 5: Migrate HomeMaintenanceCaseHub**

`super("life/home-maintenance.yaml", LifeAgent.HOME)`, `lifeCaseType() → HOME_MAINTENANCE`

Workers: `schedule-inspection`, `get-quotes`, `issue-commitment`, `monitor-job`, `record-completion`

- [ ] **Step 6: Migrate TravelPlanCaseHub**

`super("life/travel-plan.yaml", LifeAgent.TRAVEL)`, `lifeCaseType() → TRAVEL_PLAN`

Workers: `destination-research`, `flight-search`, `hotel-search`, `budget-assessment`, `booking`, `rebooking`, `confirmation`

Extra: `configureCase()` also adds SubCase bindings via `familyVoteBinding()` (private method stays).

- [ ] **Step 7: Fix CareEpisodeCaseHub (stays on YamlCaseHub)**

This CaseHub has no `LifeCaseType` (spawned as sub-case only). Fix compilation breaks:
1. Remove `volatile CaseDefinition augmentedDefinition` field
2. Remove `getDefinition()` override
3. Replace `private augment()` with `@Override protected void augment(CaseDefinition)`
4. Remove `cap()` helper — use `Worker.builder().capabilityName(name)` directly
5. Keep its own `@Inject openClawFactory` and `@Inject descriptorFactory`
6. Build workers + set descriptors directly in `augment()`

- [ ] **Step 8: Add lifeCaseType() assertions to existing CaseHub tests**

Add to each of the 6 `LifeTypedCaseHub` subclass test classes:
```java
@Test
void lifeCaseType() {
    assertEquals(LifeCaseType.FINANCIAL_REVIEW, caseHub.lifeCaseType());
}
```

With the appropriate `LifeCaseType` value for each test class:
- `AppointmentCycleCaseHubTest` → `APPOINTMENT_CYCLE`
- `CareCoordinationCaseHubTest` → `CARE_COORDINATION`
- `ContractorCoordinationCaseHubTest` → `CONTRACTOR_COORDINATION`
- `FinancialReviewCaseHubTest` → `FINANCIAL_REVIEW`
- `HomeMaintenanceCaseHubTest` → `HOME_MAINTENANCE`
- `TravelPlanCaseHubTest` → `TRAVEL_PLAN`

- [ ] **Step 9: Install api, full build, all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app`

Expected: BUILD SUCCESS, all tests pass (401+)

- [ ] **Step 10: Commit Tasks 1 + 2**

```
refactor(#47): extract LifeTypedCaseHub base — template method, agentWorker(), lifeCaseType()
```

Stage: `LifeTypedCaseHub.java`, `LifeTypedCaseHubTest.java`, all 7 migrated CaseHub files, 6 modified test files.

---

### Task 3: Update protocol + commit

**Files:**
- Modify: `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`

**Interfaces:**
- Consumes: None (documentation only)
- Produces: Updated protocol reflecting the new patterns

- [ ] **Step 1: Update openclaw-agent-worker-pattern.md**

Four changes:
1. Replace per-worker construction example with `LifeTypedCaseHub.agentWorker()` reference
2. Remove `cap()` helper — document `capabilityName(String)` as the Worker.Builder API
3. Document template method contract: subclasses override `configureCase()`, not `augment()`
4. Document manual Agent construction for the `userMessage` exception case

- [ ] **Step 2: Commit**

```
docs(#47): update openclaw-agent-worker-pattern for LifeTypedCaseHub migration
```

---

## Verification

1. `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install` — full build passes
2. All existing CaseHub tests pass (definition loads, workers present, capabilities correct, bindings correct)
3. New `LifeTypedCaseHubTest` passes (agentWorker name/capability, lifeCaseType, agent getter)
4. `FamilyVoteCaseHub` unchanged and its tests still pass
5. No references to `cap()`, `capabilities(List<Capability>)`, or `volatile CaseDefinition augmentedDefinition` remain in any CaseHub
6. No CaseHub overrides `getDefinition()` (compiler enforces — it's `final`)
