# Playbook Failure Recovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #141 — feat: playbook failure detection, triage, and adaptive recovery
**Issue group:** #141

**Goal:** Detect playbook execution failures (full and partial), triage them via the planning worker, and feed outcomes into CBR for adaptive learning.

**Architecture:** Life implements the engine's `RecoveryCoordinator` SPI. When a playbook worker exhausts retries, the coordinator re-invokes the planning worker with failure context. The planner returns a structured `TriageDecision` (one of four actions). Per-failure CBR records capture playbook outcomes for future adaptation.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine RecoveryCoordinator SPI, casehub-neocortex CBR

## Global Constraints

- All new classes in `app/src/main/java/io/casehub/life/app/coordination/recovery/`
- Response schema records follow existing pattern in `app/engine/agent/` — Java records, no framework imports
- CDI beans are `@ApplicationScoped`; `RecoveryCoordinator` impl must be `@Alternative @Priority(10)` to displace the engine's `@DefaultBean` no-op
- Tests use `@QuarkusTest` with H2 where JPA is needed; unit tests with Mockito where not
- All commits reference `Refs #141`
- Engine changes tracked as a separate issue on casehubio/engine (created in Batch 1)

---

## Batch 1: Triage Foundation — TriageDecision model + TriageRoundTracker

After this batch: the triage data model exists and is tested. The round tracker enforces the per-worker cap. No integration yet — all unit-testable in isolation.

### Task 1: TriageAction enum + TriageDecision response schema + TriageRequest

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/coordination/recovery/TriageAction.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/recovery/TriageDecision.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/recovery/TriageRequest.java`
- Test: `app/src/test/java/io/casehub/life/app/coordination/recovery/TriageDecisionTest.java`

**Interfaces:**
- Consumes: `io.casehub.api.model.PlaybookExecutionResult`, `io.casehub.api.model.StepResult` (engine API)
- Produces: `TriageAction` enum (used by Task 3, Task 4, Task 5), `TriageDecision` record (used by Task 3, Task 4), `TriageRequest` record (used by Task 3)

- [ ] **Step 1: Create TriageAction enum**

```java
package io.casehub.life.app.coordination.recovery;

public enum TriageAction {
    RETRY_MODIFIED,
    SWITCH_PLAYBOOK,
    GENERATE_FIX,
    FALLBACK_MCP
}
```

- [ ] **Step 2: Create TriageDecision response schema**

```java
package io.casehub.life.app.coordination.recovery;

import java.util.Map;

public record TriageDecision(
    TriageAction action,
    String reasoning,
    Map<String, Object> modifiedParams,
    String alternateTemplateName,
    String fixedPlaybookYaml,
    String fixedPlaybookName,
    String fallbackReason) {}
```

- [ ] **Step 3: Create TriageRequest context record**

```java
package io.casehub.life.app.coordination.recovery;

import io.casehub.api.model.PlaybookExecutionResult;
import io.casehub.api.model.StepResult;
import java.util.List;
import java.util.Map;

public record TriageRequest(
    String workerName,
    String capabilityName,
    String templateName,
    PlaybookExecutionResult executionResult,
    List<StepResult> steps,
    int triageRound,
    List<String> catalogTemplates,
    Map<String, Object> caseContext) {}
```

- [ ] **Step 4: Write TriageDecision serialization test**

```java
package io.casehub.life.app.coordination.recovery;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class TriageDecisionTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Test
    void roundTripsViaJson() throws Exception {
        var decision = new TriageDecision(
            TriageAction.RETRY_MODIFIED,
            "connector timeout — increase timeout param",
            Map.of("timeout", 30),
            null, null, null, null);

        String json = MAPPER.writeValueAsString(decision);
        TriageDecision deserialized = MAPPER.readValue(json, TriageDecision.class);

        assertThat(deserialized.action()).isEqualTo(TriageAction.RETRY_MODIFIED);
        assertThat(deserialized.modifiedParams()).containsEntry("timeout", 30);
    }

    @Test
    void eachActionHasCorrectFields() {
        var switchDecision = new TriageDecision(
            TriageAction.SWITCH_PLAYBOOK, "try alternate",
            null, "contractor-onboarding-v2", null, null, null);
        assertThat(switchDecision.alternateTemplateName()).isEqualTo("contractor-onboarding-v2");

        var fixDecision = new TriageDecision(
            TriageAction.GENERATE_FIX, "add pre-check step",
            null, null, "steps:\n  - name: pre-check", "contractor-onboarding-fixed", null);
        assertThat(fixDecision.fixedPlaybookYaml()).isNotEmpty();

        var fallbackDecision = new TriageDecision(
            TriageAction.FALLBACK_MCP, "no viable playbook",
            null, null, null, null, "all templates failed for this contractor");
        assertThat(fallbackDecision.fallbackReason()).contains("all templates");
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=TriageDecisionTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/coordination/recovery/TriageAction.java app/src/main/java/io/casehub/life/app/coordination/recovery/TriageDecision.java app/src/main/java/io/casehub/life/app/coordination/recovery/TriageRequest.java app/src/test/java/io/casehub/life/app/coordination/recovery/TriageDecisionTest.java
git commit -m "feat(#141): add TriageAction, TriageDecision, and TriageRequest records

Refs #141"
```

### Task 2: TriageRoundTracker — per-worker round counting with cap

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/coordination/recovery/TriageRoundTracker.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/recovery/PlaybookTriageEvent.java`
- Test: `app/src/test/java/io/casehub/life/app/coordination/recovery/TriageRoundTrackerTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `TriageRoundTracker.roundCount(UUID, String): int`, `TriageRoundTracker.increment(UUID, String): void`, `TriageRoundTracker.clearForCase(UUID): void` (used by Task 3); `PlaybookTriageEvent` CDI event record (used by Task 5)

- [ ] **Step 1: Write failing test — round counting and cap**

```java
package io.casehub.life.app.coordination.recovery;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class TriageRoundTrackerTest {

    private final TriageRoundTracker tracker = new TriageRoundTracker();

    @Test
    void startsAtZero() {
        UUID caseId = UUID.randomUUID();
        assertThat(tracker.roundCount(caseId, "pre-process")).isEqualTo(0);
    }

    @Test
    void incrementsPerWorker() {
        UUID caseId = UUID.randomUUID();
        tracker.increment(caseId, "pre-process");
        assertThat(tracker.roundCount(caseId, "pre-process")).isEqualTo(1);
        tracker.increment(caseId, "pre-process");
        assertThat(tracker.roundCount(caseId, "pre-process")).isEqualTo(2);
    }

    @Test
    void independentPerWorker() {
        UUID caseId = UUID.randomUUID();
        tracker.increment(caseId, "pre-process");
        tracker.increment(caseId, "pre-process");
        assertThat(tracker.roundCount(caseId, "post-process")).isEqualTo(0);
    }

    @Test
    void independentPerCase() {
        UUID case1 = UUID.randomUUID();
        UUID case2 = UUID.randomUUID();
        tracker.increment(case1, "worker-a");
        assertThat(tracker.roundCount(case2, "worker-a")).isEqualTo(0);
    }

    @Test
    void clearRemovesAllWorkersForCase() {
        UUID caseId = UUID.randomUUID();
        tracker.increment(caseId, "worker-a");
        tracker.increment(caseId, "worker-b");
        tracker.clearForCase(caseId);
        assertThat(tracker.roundCount(caseId, "worker-a")).isEqualTo(0);
        assertThat(tracker.roundCount(caseId, "worker-b")).isEqualTo(0);
    }

    @Test
    void isExhaustedAfterTwoRounds() {
        UUID caseId = UUID.randomUUID();
        tracker.increment(caseId, "worker-a");
        assertThat(tracker.isExhausted(caseId, "worker-a")).isFalse();
        tracker.increment(caseId, "worker-a");
        assertThat(tracker.isExhausted(caseId, "worker-a")).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=TriageRoundTrackerTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement TriageRoundTracker**

```java
package io.casehub.life.app.coordination.recovery;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

@ApplicationScoped
public class TriageRoundTracker {

    static final int MAX_TRIAGE_ROUNDS = 2;

    private final ConcurrentHashMap<String, AtomicInteger> rounds = new ConcurrentHashMap<>();

    public int roundCount(UUID caseId, String workerName) {
        AtomicInteger count = rounds.get(key(caseId, workerName));
        return count != null ? count.get() : 0;
    }

    public void increment(UUID caseId, String workerName) {
        rounds.computeIfAbsent(key(caseId, workerName), k -> new AtomicInteger()).incrementAndGet();
    }

    public boolean isExhausted(UUID caseId, String workerName) {
        return roundCount(caseId, workerName) >= MAX_TRIAGE_ROUNDS;
    }

    public void clearForCase(UUID caseId) {
        String prefix = caseId.toString() + ":";
        rounds.keySet().removeIf(k -> k.startsWith(prefix));
    }

    private static String key(UUID caseId, String workerName) {
        return caseId.toString() + ":" + workerName;
    }
}
```

- [ ] **Step 4: Create PlaybookTriageEvent CDI event record**

```java
package io.casehub.life.app.coordination.recovery;

import io.casehub.api.model.PlaybookExecutionResult;
import java.util.Map;
import java.util.UUID;

public record PlaybookTriageEvent(
    UUID caseId,
    String tenancyId,
    String workerName,
    String capabilityName,
    String templateName,
    TriageAction triageAction,
    String reasoning,
    int triageRound,
    PlaybookExecutionResult executionResult,
    Map<String, Object> caseContext) {}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=TriageRoundTrackerTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/coordination/recovery/TriageRoundTracker.java app/src/main/java/io/casehub/life/app/coordination/recovery/PlaybookTriageEvent.java app/src/test/java/io/casehub/life/app/coordination/recovery/TriageRoundTrackerTest.java
git commit -m "feat(#141): add TriageRoundTracker with per-worker cap and PlaybookTriageEvent

Refs #141"
```

---

## Batch 2: Triage Executor — applies decisions to case definitions

After this batch: the executor can apply all four triage actions. Tested with mocked catalog and validator. No coordinator wiring yet.

### Task 3: TriageExecutor — applies TriageDecision to case state

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/coordination/recovery/TriageExecutor.java`
- Test: `app/src/test/java/io/casehub/life/app/coordination/recovery/TriageExecutorTest.java`

**Interfaces:**
- Consumes: `TriageAction`, `TriageDecision` (Task 1), `CoordinationTemplateCatalog.findByName()`, `CoordinationTemplateCatalog.save()`, `CoordinationTemplateCatalog.resolveToPlaybookWorker()`, `CoordinationTemplateValidator.validate()`, `LifeTypedCaseHub.buildFreshDefinition()`
- Produces: `TriageExecutor.apply(TriageDecision, UUID caseId, CaseDefinition currentDef): TriageOutcome` (used by Task 4)

- [ ] **Step 1: Write failing test — RETRY_MODIFIED applies parameter injection**

```java
package io.casehub.life.app.coordination.recovery;

import io.casehub.life.app.coordination.CoordinationTemplateCatalog;
import io.casehub.life.app.coordination.CoordinationTemplateValidator;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class TriageExecutorTest {

    private CoordinationTemplateCatalog catalog;
    private CoordinationTemplateValidator validator;
    private TriageExecutor executor;

    @BeforeEach
    void setUp() {
        catalog = Mockito.mock(CoordinationTemplateCatalog.class);
        validator = Mockito.mock(CoordinationTemplateValidator.class);
        executor = new TriageExecutor(catalog, validator);
    }

    @Test
    void retryModifiedReturnsParamsToInject() {
        var decision = new TriageDecision(
            TriageAction.RETRY_MODIFIED, "increase timeout",
            Map.of("timeout", 30), null, null, null, null);

        TriageOutcome outcome = executor.apply(decision, UUID.randomUUID());
        assertThat(outcome.action()).isEqualTo(TriageAction.RETRY_MODIFIED);
        assertThat(outcome.modifiedParams()).containsEntry("timeout", 30);
        assertThat(outcome.shouldReschedule()).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=TriageExecutorTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Create TriageOutcome record**

```java
package io.casehub.life.app.coordination.recovery;

import java.util.Map;

public record TriageOutcome(
    TriageAction action,
    boolean shouldReschedule,
    Map<String, Object> modifiedParams,
    String appliedTemplateName) {

    public static TriageOutcome reschedule(TriageAction action, Map<String, Object> params, String template) {
        return new TriageOutcome(action, true, params, template);
    }

    public static TriageOutcome fallback() {
        return new TriageOutcome(TriageAction.FALLBACK_MCP, false, Map.of(), null);
    }
}
```

- [ ] **Step 4: Implement TriageExecutor**

```java
package io.casehub.life.app.coordination.recovery;

import io.casehub.life.app.coordination.CoordinationTemplate;
import io.casehub.life.app.coordination.CoordinationTemplateCatalog;
import io.casehub.life.app.coordination.CoordinationTemplateValidator;
import io.casehub.life.app.coordination.CoordinationTemplateValidator.ValidationResult;
import jakarta.enterprise.context.ApplicationScoped;
import org.jboss.logging.Logger;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class TriageExecutor {

    private static final Logger LOG = Logger.getLogger(TriageExecutor.class);

    private final CoordinationTemplateCatalog catalog;
    private final CoordinationTemplateValidator validator;

    public TriageExecutor(CoordinationTemplateCatalog catalog,
                          CoordinationTemplateValidator validator) {
        this.catalog = catalog;
        this.validator = validator;
    }

    public TriageOutcome apply(TriageDecision decision, UUID caseId) {
        return switch (decision.action()) {
            case RETRY_MODIFIED -> applyRetryModified(decision);
            case SWITCH_PLAYBOOK -> applySwitchPlaybook(decision);
            case GENERATE_FIX -> applyGenerateFix(decision);
            case FALLBACK_MCP -> TriageOutcome.fallback();
        };
    }

    private TriageOutcome applyRetryModified(TriageDecision decision) {
        return TriageOutcome.reschedule(
            TriageAction.RETRY_MODIFIED, decision.modifiedParams(), null);
    }

    private TriageOutcome applySwitchPlaybook(TriageDecision decision) {
        String altName = decision.alternateTemplateName();
        var template = catalog.findByName(altName);
        if (template.isEmpty()) {
            LOG.warnf("Triage SWITCH_PLAYBOOK: template '%s' not found, falling back to MCP", altName);
            return TriageOutcome.fallback();
        }
        return TriageOutcome.reschedule(TriageAction.SWITCH_PLAYBOOK, Map.of(), altName);
    }

    private TriageOutcome applyGenerateFix(TriageDecision decision) {
        String yaml = decision.fixedPlaybookYaml();
        String name = decision.fixedPlaybookName();
        if (yaml == null || name == null) {
            LOG.warn("Triage GENERATE_FIX: missing YAML or name, falling back to MCP");
            return TriageOutcome.fallback();
        }

        ValidationResult validation = validator.validate(yaml);
        if (!validation.valid()) {
            LOG.warnf("Triage GENERATE_FIX: invalid YAML — %s, falling back to MCP", validation.errors());
            return TriageOutcome.fallback();
        }

        var existing = catalog.findByName(name);
        if (existing.isPresent() && existing.get().isCurated()) {
            String fixedName = name + "-fixed";
            LOG.infof("Triage: curated template '%s' — saving fix as '%s'", name, fixedName);
            name = fixedName;
        }

        CoordinationTemplate template = new CoordinationTemplate();
        template.setName(name);
        template.setTemplateYaml(yaml);
        template.setCurated(false);
        template.setDescription("Auto-generated fix via triage round");
        catalog.save(template);

        return TriageOutcome.reschedule(TriageAction.GENERATE_FIX, Map.of(), name);
    }
}
```

- [ ] **Step 5: Add remaining test cases — SWITCH_PLAYBOOK, GENERATE_FIX, FALLBACK_MCP**

Add to `TriageExecutorTest.java`:

```java
    @Test
    void switchPlaybookResolvesFromCatalog() {
        var template = Mockito.mock(CoordinationTemplate.class);
        Mockito.when(catalog.findByName("alt-template")).thenReturn(java.util.Optional.of(template));

        var decision = new TriageDecision(
            TriageAction.SWITCH_PLAYBOOK, "try alternate",
            null, "alt-template", null, null, null);

        TriageOutcome outcome = executor.apply(decision, UUID.randomUUID());
        assertThat(outcome.action()).isEqualTo(TriageAction.SWITCH_PLAYBOOK);
        assertThat(outcome.appliedTemplateName()).isEqualTo("alt-template");
        assertThat(outcome.shouldReschedule()).isTrue();
    }

    @Test
    void switchPlaybookFallsBackWhenNotFound() {
        Mockito.when(catalog.findByName("missing")).thenReturn(java.util.Optional.empty());

        var decision = new TriageDecision(
            TriageAction.SWITCH_PLAYBOOK, "try alternate",
            null, "missing", null, null, null);

        TriageOutcome outcome = executor.apply(decision, UUID.randomUUID());
        assertThat(outcome.action()).isEqualTo(TriageAction.FALLBACK_MCP);
        assertThat(outcome.shouldReschedule()).isFalse();
    }

    @Test
    void generateFixValidatesAndPersists() {
        Mockito.when(validator.validate(Mockito.anyString()))
            .thenReturn(new CoordinationTemplateValidator.ValidationResult(true, java.util.List.of()));
        Mockito.when(catalog.findByName("fixed-template")).thenReturn(java.util.Optional.empty());

        var decision = new TriageDecision(
            TriageAction.GENERATE_FIX, "add pre-check",
            null, null, "steps:\n  - name: pre-check", "fixed-template", null);

        TriageOutcome outcome = executor.apply(decision, UUID.randomUUID());
        assertThat(outcome.action()).isEqualTo(TriageAction.GENERATE_FIX);
        assertThat(outcome.appliedTemplateName()).isEqualTo("fixed-template");
        Mockito.verify(catalog).save(Mockito.any(CoordinationTemplate.class));
    }

    @Test
    void generateFixFallsBackOnInvalidYaml() {
        Mockito.when(validator.validate(Mockito.anyString()))
            .thenReturn(new CoordinationTemplateValidator.ValidationResult(false, java.util.List.of("missing steps")));

        var decision = new TriageDecision(
            TriageAction.GENERATE_FIX, "bad fix",
            null, null, "invalid yaml", "bad-template", null);

        TriageOutcome outcome = executor.apply(decision, UUID.randomUUID());
        assertThat(outcome.action()).isEqualTo(TriageAction.FALLBACK_MCP);
    }

    @Test
    void generateFixRenamesWhenCurated() {
        Mockito.when(validator.validate(Mockito.anyString()))
            .thenReturn(new CoordinationTemplateValidator.ValidationResult(true, java.util.List.of()));
        var curatedTemplate = Mockito.mock(CoordinationTemplate.class);
        Mockito.when(curatedTemplate.isCurated()).thenReturn(true);
        Mockito.when(catalog.findByName("contractor-onboarding")).thenReturn(java.util.Optional.of(curatedTemplate));

        var decision = new TriageDecision(
            TriageAction.GENERATE_FIX, "fix curated",
            null, null, "steps:\n  - name: fixed", "contractor-onboarding", null);

        TriageOutcome outcome = executor.apply(decision, UUID.randomUUID());
        assertThat(outcome.appliedTemplateName()).isEqualTo("contractor-onboarding-fixed");
    }

    @Test
    void fallbackMcpReturnsNoReschedule() {
        var decision = new TriageDecision(
            TriageAction.FALLBACK_MCP, "give up",
            null, null, null, null, "all attempts exhausted");

        TriageOutcome outcome = executor.apply(decision, UUID.randomUUID());
        assertThat(outcome.action()).isEqualTo(TriageAction.FALLBACK_MCP);
        assertThat(outcome.shouldReschedule()).isFalse();
    }
```

- [ ] **Step 6: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=TriageExecutorTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/coordination/recovery/TriageExecutor.java app/src/main/java/io/casehub/life/app/coordination/recovery/TriageOutcome.java app/src/test/java/io/casehub/life/app/coordination/recovery/TriageExecutorTest.java
git commit -m "feat(#141): add TriageExecutor with all four action handlers

Validates GENERATE_FIX YAML, protects curated templates from replacement,
falls back to MCP on invalid state.

Refs #141"
```

---

## Batch 3: RecoveryCoordinator — the core integration

After this batch: full triage pipeline works end-to-end in unit tests. The coordinator receives failure context from the engine, invokes the planning worker, and dispatches the result.

### Task 4: LifePlaybookRecoveryCoordinator — RecoveryCoordinator SPI implementation

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/coordination/recovery/LifePlaybookRecoveryCoordinator.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java` — harden `invokePlanningWorker()` (add CBR record on failure)
- Test: `app/src/test/java/io/casehub/life/app/coordination/recovery/LifePlaybookRecoveryCoordinatorTest.java`

**Interfaces:**
- Consumes: `RecoveryCoordinator` (engine SPI), `RecoveryContext` (engine), `TriageRoundTracker` (Task 2), `TriageExecutor` (Task 3), `TriageDecision` (Task 1), `LifeAgentProviderFactory.invokerFor()`, `WorkerAgentInvoker.execute()`
- Produces: `LifePlaybookRecoveryCoordinator` — CDI bean implementing `RecoveryCoordinator.handleFailure(RecoveryContext): boolean`; fires `PlaybookTriageEvent` CDI event (consumed by Task 5)

- [ ] **Step 1: Write failing test — coordinator returns false for non-playbook failures**

```java
package io.casehub.life.app.coordination.recovery;

import io.casehub.engine.common.spi.recovery.RecoveryContext;
import io.casehub.worker.api.WorkerOutcome;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.lenient;

class LifePlaybookRecoveryCoordinatorTest {

    private TriageRoundTracker roundTracker;
    private TriageExecutor triageExecutor;
    private LifeAgentProviderFactory agentFactory;
    private Event<PlaybookTriageEvent> triageEvent;
    private CoordinationTemplateCatalog catalog;
    private LifePlaybookRecoveryCoordinator coordinator;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        roundTracker = new TriageRoundTracker();
        triageExecutor = Mockito.mock(TriageExecutor.class);
        agentFactory = Mockito.mock(LifeAgentProviderFactory.class);
        triageEvent = Mockito.mock(Event.class);
        catalog = Mockito.mock(CoordinationTemplateCatalog.class);
        coordinator = new LifePlaybookRecoveryCoordinator(
            roundTracker, triageExecutor, agentFactory, triageEvent, catalog);
    }

    @Test
    void returnsFalseForNonPlaybookFailure() {
        var ctx = new RecoveryContext(
            UUID.randomUUID(), "tenant-1", "binding-1",
            "some-worker", "some-capability",
            null, 3, null, null);

        assertThat(coordinator.handleFailure(ctx)).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifePlaybookRecoveryCoordinatorTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement LifePlaybookRecoveryCoordinator**

```java
package io.casehub.life.app.coordination.recovery;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.PlaybookExecutionResult;
import io.casehub.api.model.StepResult;
import io.casehub.api.model.ai.WorkerAgentInvoker;
import io.casehub.engine.common.spi.recovery.RecoveryContext;
import io.casehub.engine.common.spi.recovery.RecoveryCoordinator;
import io.casehub.life.app.coordination.CoordinationTemplateCatalog;
import io.casehub.life.app.engine.LifeAgent;
import io.casehub.life.app.engine.agent.LifeAgentProviderFactory;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Alternative;
import jakarta.annotation.Priority;
import org.jboss.logging.Logger;
import java.util.List;
import java.util.Map;

@ApplicationScoped
@Alternative
@Priority(10)
public class LifePlaybookRecoveryCoordinator implements RecoveryCoordinator {

    private static final Logger LOG = Logger.getLogger(LifePlaybookRecoveryCoordinator.class);
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final TriageRoundTracker roundTracker;
    private final TriageExecutor triageExecutor;
    private final LifeAgentProviderFactory agentFactory;
    private final Event<PlaybookTriageEvent> triageEvent;
    private final CoordinationTemplateCatalog catalog;

    public LifePlaybookRecoveryCoordinator(
            TriageRoundTracker roundTracker,
            TriageExecutor triageExecutor,
            LifeAgentProviderFactory agentFactory,
            Event<PlaybookTriageEvent> triageEvent,
            CoordinationTemplateCatalog catalog) {
        this.roundTracker = roundTracker;
        this.triageExecutor = triageExecutor;
        this.agentFactory = agentFactory;
        this.triageEvent = triageEvent;
        this.catalog = catalog;
    }

    @Override
    public boolean handleFailure(RecoveryContext context) {
        if (context.executionResult() == null) {
            return false;
        }

        if (roundTracker.isExhausted(context.caseId(), context.workerName())) {
            LOG.infof("Triage budget exhausted for %s in case %s",
                context.workerName(), context.caseId());
            return false;
        }

        int round = roundTracker.roundCount(context.caseId(), context.workerName()) + 1;

        TriageDecision decision;
        try {
            decision = invokeTriagePlanner(context, round);
        } catch (Exception e) {
            LOG.warn("Triage planner invocation failed", e);
            return false;
        }

        TriageOutcome outcome = triageExecutor.apply(decision, context.caseId());
        roundTracker.increment(context.caseId(), context.workerName());

        fireTriageEvent(context, decision, round);

        return outcome.shouldReschedule();
    }

    private TriageDecision invokeTriagePlanner(RecoveryContext context, int round) {
        List<StepResult> steps = context.executionResult().steps();
        String templateListing = catalog.all().stream()
            .map(t -> t.getName() + " (" + t.getDomain() + ")")
            .collect(java.util.stream.Collectors.joining(", "));

        String systemPrompt = buildTriagePrompt(context, steps, round, templateListing);
        WorkerAgentInvoker invoker = agentFactory.invokerFor(
            LifeAgent.HOME, systemPrompt, null,
            TriageDecision.class, input -> input);
        var result = invoker.execute(Map.of(
            "failedWorker", context.workerName(),
            "executionResult", context.executionResult(),
            "triageRound", round));
        return MAPPER.convertValue(result.output(), TriageDecision.class);
    }

    private String buildTriagePrompt(RecoveryContext context, List<StepResult> steps,
                                      int round, String templateListing) {
        long failedCount = steps.stream()
            .filter(s -> s.status() == io.casehub.api.model.StepStatus.FAILED).count();
        String failedSteps = steps.stream()
            .filter(s -> s.status() == io.casehub.api.model.StepStatus.FAILED)
            .map(s -> s.stepName() + ": " + s.error())
            .collect(java.util.stream.Collectors.joining("\n  "));

        return "You are triaging a playbook failure. Triage round " + round + " of 2.\n\n"
            + "Worker: " + context.workerName() + "\n"
            + "Failed steps (" + failedCount + "/" + steps.size() + "):\n  " + failedSteps + "\n\n"
            + "Available templates: " + templateListing + "\n\n"
            + "Return a TriageDecision JSON with one of:\n"
            + "- RETRY_MODIFIED: same playbook, adjust params (round 1 preferred)\n"
            + "- SWITCH_PLAYBOOK: select a different template by name\n"
            + "- GENERATE_FIX: produce fixed YAML (round 2 preferred)\n"
            + "- FALLBACK_MCP: abandon playbooks, use MCP tools directly\n";
    }

    private void fireTriageEvent(RecoveryContext context, TriageDecision decision, int round) {
        triageEvent.fire(new PlaybookTriageEvent(
            context.caseId(),
            context.tenancyId(),
            context.workerName(),
            context.capabilityName(),
            null,
            decision.action(),
            decision.reasoning(),
            round,
            context.executionResult(),
            Map.of()));
    }
}
```

- [ ] **Step 4: Add test — returns false when triage budget exhausted**

```java
    @Test
    void returnsFalseWhenTriageBudgetExhausted() {
        UUID caseId = UUID.randomUUID();
        roundTracker.increment(caseId, "pre-process");
        roundTracker.increment(caseId, "pre-process");

        var execResult = new PlaybookExecutionResult.Failure("timeout", java.util.List.of());
        var ctx = new RecoveryContext(
            caseId, "tenant-1", "binding-1",
            "pre-process", "household-management",
            null, 3, null, execResult);

        assertThat(coordinator.handleFailure(ctx)).isFalse();
    }
```

- [ ] **Step 5: Add test — invokes triage planner and applies decision (mocked)**

```java
    @Test
    void invokesTriagePlannerAndAppliesDecision() {
        var execResult = new PlaybookExecutionResult.Failure("timeout", java.util.List.of());
        UUID caseId = UUID.randomUUID();
        var ctx = new RecoveryContext(
            caseId, "tenant-1", "binding-1",
            "pre-process", "household-management",
            null, 3, null, execResult);

        var decision = new TriageDecision(
            TriageAction.FALLBACK_MCP, "no fix possible",
            null, null, null, null, "all failed");
        var outcome = TriageOutcome.fallback();

        var invoker = Mockito.mock(io.casehub.api.model.ai.WorkerAgentInvoker.class);
        Mockito.when(agentFactory.invokerFor(
            Mockito.any(), Mockito.anyString(), Mockito.isNull(),
            Mockito.eq(TriageDecision.class), Mockito.any()))
            .thenReturn(invoker);
        Mockito.when(invoker.execute(Mockito.anyMap()))
            .thenReturn(io.casehub.worker.api.WorkerResult.of(
                new com.fasterxml.jackson.databind.ObjectMapper().convertValue(
                    decision, new com.fasterxml.jackson.core.type.TypeReference<java.util.Map<String, Object>>() {})));
        Mockito.when(triageExecutor.apply(Mockito.any(), Mockito.eq(caseId)))
            .thenReturn(outcome);

        boolean handled = coordinator.handleFailure(ctx);

        assertThat(handled).isFalse();
        assertThat(roundTracker.roundCount(caseId, "pre-process")).isEqualTo(1);
        Mockito.verify(triageEvent).fire(Mockito.any(PlaybookTriageEvent.class));
    }
```

- [ ] **Step 6: Harden LifeCaseService.invokePlanningWorker() — log CBR-trackable event on failure**

In `LifeCaseService.java`, modify the catch block at line 241-243:

```java
    } catch (Exception e) {
        LOG.warn("Planning worker failed, proceeding without coordination", e);
        triageEvent.fire(new PlaybookTriageEvent(
            null, null, "planning-worker", null, null,
            TriageAction.FALLBACK_MCP, "planning worker exception: " + e.getMessage(),
            0, null, Map.of()));
        return null;
    }
```

Add field injection: `@Inject Event<PlaybookTriageEvent> triageEvent;`

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifePlaybookRecoveryCoordinatorTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Add `LifePlaybookRecoveryCoordinator` to `quarkus.arc.selected-alternatives` in test and production `application.properties`**

Append `io.casehub.life.app.coordination.recovery.LifePlaybookRecoveryCoordinator` to the `quarkus.arc.selected-alternatives` list.

- [ ] **Step 9: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/coordination/recovery/LifePlaybookRecoveryCoordinator.java app/src/test/java/io/casehub/life/app/coordination/recovery/LifePlaybookRecoveryCoordinatorTest.java app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java app/src/main/resources/application.properties app/src/test/resources/application.properties
git commit -m "feat(#141): implement RecoveryCoordinator SPI for playbook triage

LifePlaybookRecoveryCoordinator intercepts exhausted-retry failures,
re-invokes planning worker for triage, applies structured TriageDecision.
Hardens LifeCaseService.invokePlanningWorker() with failure event.

Refs #141"
```

---

## Batch 4: CBR Integration + Cleanup Observer

After this batch: playbook failures are retained in CBR. Triage round state is cleaned up on case termination. Full feature complete.

### Task 5: LifePlaybookCbrWriter — per-failure CBR retention

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/coordination/recovery/LifePlaybookCbrWriter.java`
- Modify: `app/src/main/java/io/casehub/life/app/cbr/LifeCbrFeatureSchemaRegistrar.java` — add playbook failure feature schema
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeProvisionerCleanupObserver.java` — piggyback `TriageRoundTracker.clearForCase()`
- Test: `app/src/test/java/io/casehub/life/app/coordination/recovery/LifePlaybookCbrWriterTest.java`

**Interfaces:**
- Consumes: `PlaybookTriageEvent` (Task 2), `CbrRecordStore`, `TriageAction`, `StepResult`, `StepStatus`
- Produces: CDI observer method `onTriageEvent(@Observes PlaybookTriageEvent event)` — writes `CbrPlanRecord` to `CbrRecordStore`

- [ ] **Step 1: Write failing test — CBR record written on triage event**

```java
package io.casehub.life.app.coordination.recovery;

import io.casehub.api.model.PlaybookExecutionResult;
import io.casehub.api.model.StepResult;
import io.casehub.api.model.StepStatus;
import io.casehub.neocortex.memory.api.CbrRecordStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.mockito.Mockito;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class LifePlaybookCbrWriterTest {

    private CbrRecordStore cbrStore;
    private LifePlaybookCbrWriter writer;

    @BeforeEach
    void setUp() {
        cbrStore = Mockito.mock(CbrRecordStore.class);
        writer = new LifePlaybookCbrWriter(cbrStore);
    }

    @Test
    void writesRecordWithPlaybookFeatures() {
        var steps = List.of(
            new StepResult("s1", "fetch-contacts", StepStatus.SUCCEEDED, Duration.ofMillis(200), null),
            new StepResult("s2", "send-email", StepStatus.FAILED, Duration.ofMillis(5000), "timeout"),
            new StepResult("s3", "log-result", StepStatus.SKIPPED, Duration.ZERO, null));

        var execResult = new PlaybookExecutionResult.PartialSuccess(Map.of("partial", true), steps);

        var event = new PlaybookTriageEvent(
            UUID.randomUUID(), "tenant-1", "pre-process",
            "contractor-coordination", "contractor-onboarding",
            TriageAction.RETRY_MODIFIED, "increase timeout",
            1, execResult, Map.of("domain", "CONTRACTOR_COORDINATION"));

        writer.onTriageEvent(event);

        Mockito.verify(cbrStore).store(Mockito.any());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifePlaybookCbrWriterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement LifePlaybookCbrWriter**

```java
package io.casehub.life.app.coordination.recovery;

import io.casehub.api.model.StepResult;
import io.casehub.api.model.StepStatus;
import io.casehub.neocortex.memory.api.CbrRecordStore;
import io.casehub.neocortex.memory.api.model.CbrPlanRecord;
import io.casehub.neocortex.memory.api.model.FeatureValue;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import org.jboss.logging.Logger;
import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.stream.Collectors;

@ApplicationScoped
public class LifePlaybookCbrWriter {

    private static final Logger LOG = Logger.getLogger(LifePlaybookCbrWriter.class);
    static final String TENANT_ID = "life-personal";

    private final CbrRecordStore cbrStore;

    public LifePlaybookCbrWriter(CbrRecordStore cbrStore) {
        this.cbrStore = cbrStore;
    }

    void onTriageEvent(@Observes PlaybookTriageEvent event) {
        Map<String, FeatureValue> features = new LinkedHashMap<>();
        features.put("templateName",
            new FeatureValue.CategoricalVal(event.templateName() != null ? event.templateName() : "unknown"));
        features.put("triageAction",
            new FeatureValue.CategoricalVal(event.triageAction().name()));
        features.put("triageRound",
            new FeatureValue.NumberVal(event.triageRound()));

        if (event.executionResult() != null) {
            var steps = event.executionResult().steps();
            long failedCount = steps.stream()
                .filter(s -> s.status() == StepStatus.FAILED).count();
            features.put("failedStepCount", new FeatureValue.NumberVal(failedCount));

            String failedNames = steps.stream()
                .filter(s -> s.status() == StepStatus.FAILED)
                .map(StepResult::stepName)
                .collect(Collectors.joining(","));
            if (!failedNames.isEmpty()) {
                features.put("failedStepNames", new FeatureValue.CategoricalVal(failedNames));
            }

            String firstError = steps.stream()
                .filter(s -> s.status() == StepStatus.FAILED && s.error() != null)
                .map(StepResult::error)
                .findFirst().orElse("unknown");
            features.put("connectorErrorType", new FeatureValue.CategoricalVal(categorizeError(firstError)));
        }

        String domain = event.caseContext() != null
            ? String.valueOf(event.caseContext().getOrDefault("domain", "unknown"))
            : "unknown";

        CbrPlanRecord record = new CbrPlanRecord(
            "casehubio/life/playbook-triage/" + domain.toLowerCase(),
            TENANT_ID,
            "playbook-triage:" + event.workerName(),
            "Playbook triage for " + event.workerName() + ": " + event.triageAction(),
            event.triageAction() != TriageAction.FALLBACK_MCP ? "resolved" : "unresolved",
            features,
            Instant.now());

        cbrStore.store(record);
        LOG.debugf("CBR record stored for playbook triage: worker=%s action=%s",
            event.workerName(), event.triageAction());
    }

    private static String categorizeError(String error) {
        if (error == null) return "unknown";
        String lower = error.toLowerCase();
        if (lower.contains("timeout")) return "timeout";
        if (lower.contains("401") || lower.contains("403") || lower.contains("unauthorized")) return "auth";
        if (lower.contains("404") || lower.contains("not found")) return "not-found";
        if (lower.contains("500") || lower.contains("internal server")) return "server-error";
        return "other";
    }
}
```

- [ ] **Step 4: Add playbook failure feature schema to LifeCbrFeatureSchemaRegistrar**

Add a new method and register it in `onStartup()`:

```java
    private static CbrRecordSchema playbookTriage() {
        return CbrRecordSchema.builder("playbook-triage")
            .feature("templateName", SEASON_TABLE)
            .feature("triageAction", SEASON_TABLE)
            .feature("failedStepCount", COST_DECAY)
            .feature("connectorErrorType", SEASON_TABLE)
            .feature("triageRound", COST_DECAY)
            .build();
    }
```

In `onStartup()`, add: `cbrStore.registerSchema(playbookTriage());`

- [ ] **Step 5: Add TriageRoundTracker cleanup to LifeProvisionerCleanupObserver**

Inject `TriageRoundTracker` and call `clearForCase(event.caseId())` alongside existing `provisioner.terminateAllForCase()`.

- [ ] **Step 6: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifePlaybookCbrWriterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 7: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/coordination/recovery/LifePlaybookCbrWriter.java app/src/test/java/io/casehub/life/app/coordination/recovery/LifePlaybookCbrWriterTest.java app/src/main/java/io/casehub/life/app/cbr/LifeCbrFeatureSchemaRegistrar.java app/src/main/java/io/casehub/life/app/engine/LifeProvisionerCleanupObserver.java
git commit -m "feat(#141): add playbook CBR writer and cleanup integration

LifePlaybookCbrWriter observes PlaybookTriageEvent, writes per-failure
CBR records with playbook-specific features. LifeProvisionerCleanupObserver
clears TriageRoundTracker on case termination. Playbook failure feature
schema registered in LifeCbrFeatureSchemaRegistrar.

Refs #141"
```

### Task 6: Engine issue — PartialSuccess detection in FlowWorkerFunctionHandler

**Files:**
- This task creates a GitHub issue on casehubio/engine — no code changes in life

**Interfaces:**
- Consumes: nothing
- Produces: Engine issue tracking `PartialSuccess` detection

- [ ] **Step 1: Create engine issue**

```bash
gh issue create --repo casehubio/engine --title "feat: FlowWorkerFunctionHandler — produce PartialSuccess for mixed step results" --body "## Summary

FlowWorkerFunctionHandler currently produces PlaybookExecutionResult.Success even when some steps have StepStatus.FAILED. It should check the step results after workflow completion and produce PartialSuccess when any step failed but the workflow didn't throw.

## Change

In FlowWorkerFunctionHandler.execute(), after collecting steps from the registry (line 96), check for failed steps:

\`\`\`java
boolean hasFailedSteps = steps.stream()
    .anyMatch(s -> s.status() == StepStatus.FAILED);
\`\`\`

If any step failed, throw PlaybookExecutionException wrapping PartialSuccess (not Success), so the RetryOrchestrator path fires and RecoveryCoordinator can handle it.

## Context
- Depends on: casehubio/life#141 (playbook failure recovery)
- PlaybookExecutionResult.PartialSuccess already exists in the API — never produced
- ~5 lines of code + test"
```

- [ ] **Step 2: Record the engine issue number in the spec for tracking**

- [ ] **Step 3: Commit** (no code changes — just tracking)

---

## References

- [2026-10-29-playbook-failure-recovery-design.md] — design spec this plan implements
- `engine/common-core/.../RecoveryCoordinator.java` — SPI interface being implemented
- `engine/common-core/.../RetryOrchestrator.java:112-128` — RecoveryCoordinator invocation point
- `engine/common-core/.../RecoveryContext.java` — context record carrying PlaybookExecutionResult
- `engine/flow/FlowWorkerFunctionHandler.java:88-129` — playbook execution and result production
- `life/app/coordination/CoordinationTemplateCatalog.java` — template storage (classpath + DB)
- `life/app/coordination/CoordinationTemplateValidator.java` — YAML validation
- `life/app/engine/LifeCaseService.java:230-245` — planning worker invocation (silent-catch path)
- `life/app/engine/agent/LifeAgentProviderFactory.java:41-51` — invokerFor() for triage planner
- `life/app/cbr/LifeCbrFeatureSchemaRegistrar.java` — domain feature schema registration
- `life/app/cbr/LifeCaseOutcomeCbrWriter.java` — existing CBR retention pattern
- PP-20260618-openclaw-agent — agent worker protocol
- GitHub casehubio/life#141
