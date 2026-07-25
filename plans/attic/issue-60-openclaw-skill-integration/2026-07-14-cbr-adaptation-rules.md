# CBR Domain Adaptation Rules Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #55 — feat: CBR domain adaptation rules
**Issue group:** #55

**Goal:** Implement `PlanAdapter` SPI from casehub-neocortex-memory-api
with per-domain adaptation rules that adjust retrieved past case plans
to the current context.

**Architecture:** Composite `LifePlanAdapter` (`@Alternative`) dispatches
to 6 per-domain `LifeAdaptationRule` implementations. Each rule is a pure
function operating on feature deltas. Integration via
`LifeCbrSuggestionService.retrieveForAdaptation()` at case start, with
adapted plan written to case context and formatted for workers.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-neocortex-memory-api
0.2-SNAPSHOT (PlanAdapter SPI)

## Global Constraints

- `api/` module: pure Java, zero framework imports, no JPA, no neocortex types
- `app/` module: Quarkus, CDI, JPA — neocortex types allowed here
- `LifeAdaptationRule` is app-internal (not in `api/`)
- Adaptation rules are pure functions — no injected dependencies
- `@Alternative @Priority(10)` for `LifePlanAdapter` to displace `NoOpPlanAdapter`
- Use `mvn test -pl app -Dtest=ClassName` for single-class test runs
- Run `mvn install -pl api` before app tests if api/ classes change

---

### Task 1: LifeAdaptationRule SPI + LifePlanAdapter dispatcher

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifeAdaptationRule.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifePlanAdapter.java`
- Create: `app/src/test/java/io/casehub/life/app/cbr/LifePlanAdapterTest.java`

**Interfaces:**
- Consumes: `PlanAdapter` (neocortex SPI), `ScoredCbrCase<PlanCbrCase>`, `AdaptedPlan`, `AdaptedStep`, `AdaptationAction` from neocortex-memory-api
- Produces: `LifeAdaptationRule` interface (consumed by Task 2 rules), `LifePlanAdapter` CDI bean

- [ ] **Step 1: Write LifeAdaptationRule interface**

```java
package io.casehub.life.app.cbr;

import io.casehub.neocortex.memory.cbr.AdaptedStep;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import java.util.List;
import java.util.Map;

public interface LifeAdaptationRule {
    String caseType();
    List<AdaptedStep> adapt(ScoredCbrCase<PlanCbrCase> retrieved,
                            Map<String, FeatureValue> currentFeatures);
}
```

- [ ] **Step 2: Write failing tests for LifePlanAdapter**

Test cases:
- `dispatch_knownCaseType_delegatesToRule` — known type dispatches correctly
- `dispatch_unknownCaseType_retainsAll` — unknown type retains all steps
- `spiMethod_infersFromCapabilities` — SPI path infers caseType
- `spiMethod_noMatch_retainsAll` — no capability match → retain all
- `duplicateCaseType_throwsAtConstruction` — two rules with same caseType → fail fast
- `emptyPlanTrace_returnsEmptySteps` — empty trace → empty adaptation

```java
package io.casehub.life.app.cbr;

import io.casehub.neocortex.memory.cbr.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class LifePlanAdapterTest {

    @Test
    void dispatch_knownCaseType_delegatesToRule() {
        var rule = testRule("contractor-coordination",
            (r, f) -> List.of(new AdaptedStep("b1", "request-quote", "w1",
                "ok", 8, Map.of("slaHours", 24),
                AdaptationAction.BOOSTED, "winter urgency")));
        var adapter = new LifePlanAdapter(List.of(rule));
        var scored = scoredCase("request-quote");
        var result = adapter.adapt("contractor-coordination", scored, Map.of());
        assertEquals(1, result.steps().size());
        assertEquals(AdaptationAction.BOOSTED, result.steps().getFirst().action());
    }

    @Test
    void dispatch_unknownCaseType_retainsAll() {
        var adapter = new LifePlanAdapter(List.of());
        var scored = scoredCase("some-capability");
        var result = adapter.adapt("unknown-type", scored, Map.of());
        assertEquals(1, result.steps().size());
        assertEquals(AdaptationAction.RETAINED, result.steps().getFirst().action());
    }

    @Test
    void spiMethod_infersFromCapabilities() {
        var rule = testRule("contractor-coordination",
            (r, f) -> List.of(new AdaptedStep("b1", "request-quote", "w1",
                "ok", 8, Map.of(), AdaptationAction.BOOSTED, "test")));
        var adapter = new LifePlanAdapter(List.of(rule));
        var scored = scoredCase("request-quote");
        var result = adapter.adapt(scored, Map.of());
        assertEquals(AdaptationAction.BOOSTED, result.steps().getFirst().action());
    }

    @Test
    void spiMethod_noMatch_retainsAll() {
        var rule = testRule("contractor-coordination",
            (r, f) -> List.of());
        var adapter = new LifePlanAdapter(List.of(rule));
        var scored = scoredCase("unknown-capability");
        var result = adapter.adapt(scored, Map.of());
        assertEquals(1, result.steps().size());
        assertEquals(AdaptationAction.RETAINED, result.steps().getFirst().action());
    }

    @Test
    void duplicateCaseType_throwsAtConstruction() {
        var rule1 = testRule("same-type", (r, f) -> List.of());
        var rule2 = testRule("same-type", (r, f) -> List.of());
        assertThrows(IllegalStateException.class,
            () -> new LifePlanAdapter(List.of(rule1, rule2)));
    }

    @Test
    void emptyPlanTrace_returnsEmptySteps() {
        var adapter = new LifePlanAdapter(List.of());
        var scored = new ScoredCbrCase<>(
            new PlanCbrCase("p", "s", "COMPLETED", 0.9,
                Map.of(), List.of()),
            "case-1", 0.85);
        var result = adapter.adapt("any", scored, Map.of());
        assertTrue(result.steps().isEmpty());
    }

    // --- helpers ---

    static ScoredCbrCase<PlanCbrCase> scoredCase(String capabilityName) {
        return new ScoredCbrCase<>(
            new PlanCbrCase("problem", "solution", "COMPLETED", 0.9,
                Map.of("budget", FeatureValue.number(1000)),
                List.of(new PlanTrace("b1", capabilityName, "w1", "ok", 5, Map.of()))),
            "case-1", 0.85);
    }

    static LifeAdaptationRule testRule(String caseType,
            java.util.function.BiFunction<ScoredCbrCase<PlanCbrCase>,
                Map<String, FeatureValue>, List<AdaptedStep>> fn) {
        return new LifeAdaptationRule() {
            @Override public String caseType() { return caseType; }
            @Override public List<AdaptedStep> adapt(
                    ScoredCbrCase<PlanCbrCase> r, Map<String, FeatureValue> f) {
                return fn.apply(r, f);
            }
        };
    }
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifePlanAdapterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `LifePlanAdapter` does not exist

- [ ] **Step 4: Implement LifePlanAdapter**

```java
package io.casehub.life.app.cbr;

import io.casehub.neocortex.memory.cbr.*;
import io.quarkus.arc.All;
import jakarta.alternative.Alternative;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.*;

@ApplicationScoped
@Alternative
@Priority(10)
public class LifePlanAdapter implements PlanAdapter {

    private final Map<String, LifeAdaptationRule> rulesByType;
    private final Map<String, LifeAdaptationRule> rulesByCapability;

    @Inject
    public LifePlanAdapter(@All List<LifeAdaptationRule> rules) {
        this.rulesByType = new LinkedHashMap<>();
        this.rulesByCapability = new LinkedHashMap<>();
        for (var rule : rules) {
            var prev = rulesByType.put(rule.caseType(), rule);
            if (prev != null) {
                throw new IllegalStateException(
                    "Duplicate LifeAdaptationRule for caseType '" + rule.caseType()
                    + "': " + prev.getClass().getName() + " and " + rule.getClass().getName());
            }
        }
        // Build capability→rule index for SPI inference
        // Rules must expose their known capabilities for this to work
    }

    @Override
    public AdaptedPlan adapt(ScoredCbrCase<PlanCbrCase> retrieved,
                             Map<String, FeatureValue> currentFeatures) {
        // SPI path — infer caseType from plan trace capabilities
        String inferred = inferCaseType(retrieved.cbrCase().planTrace());
        return adapt(inferred, retrieved, currentFeatures);
    }

    public AdaptedPlan adapt(String caseType,
                             ScoredCbrCase<PlanCbrCase> retrieved,
                             Map<String, FeatureValue> currentFeatures) {
        LifeAdaptationRule rule = rulesByType.get(caseType);
        if (rule == null) {
            return retainAll(retrieved);
        }
        List<AdaptedStep> steps = rule.adapt(retrieved, currentFeatures);
        return new AdaptedPlan(steps);
    }

    private String inferCaseType(List<PlanTrace> traces) {
        for (var trace : traces) {
            for (var entry : rulesByType.entrySet()) {
                // Check if rule's caseType matches based on capability names
                // Each life case definition has unique capability names
                if (matchesRule(entry.getValue(), trace.capabilityName())) {
                    return entry.getKey();
                }
            }
        }
        return "";
    }

    private boolean matchesRule(LifeAdaptationRule rule, String capabilityName) {
        // Convention: capability names in life are prefixed/unique per case type
        // Each rule knows its own capabilities
        return rule instanceof CapabilityAware ca
            && ca.knownCapabilities().contains(capabilityName);
    }

    private AdaptedPlan retainAll(ScoredCbrCase<PlanCbrCase> retrieved) {
        return new AdaptedPlan(
            retrieved.cbrCase().planTrace().stream()
                .map(t -> new AdaptedStep(t.bindingName(), t.capabilityName(),
                    t.workerName(), t.stepOutcome(), t.priority(),
                    t.parameters(), AdaptationAction.RETAINED, null))
                .toList());
    }

    interface CapabilityAware {
        Set<String> knownCapabilities();
    }
}
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifePlanAdapterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 6 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/LifeAdaptationRule.java app/src/main/java/io/casehub/life/app/cbr/LifePlanAdapter.java app/src/test/java/io/casehub/life/app/cbr/LifePlanAdapterTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#55): LifeAdaptationRule SPI + LifePlanAdapter dispatcher with tests"
```

---

### Task 2: SeverityScaling helper + 6 domain adaptation rules

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/adapt/SeverityScaling.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/adapt/ContractorAdaptationRule.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/adapt/HomeMaintenanceAdaptationRule.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/adapt/HealthAdaptationRule.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/adapt/FinancialAdaptationRule.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/adapt/AppointmentCycleAdaptationRule.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/adapt/TravelPlanAdaptationRule.java`
- Create: `app/src/test/java/io/casehub/life/app/cbr/adapt/ContractorAdaptationRuleTest.java`
- Create: `app/src/test/java/io/casehub/life/app/cbr/adapt/HomeMaintenanceAdaptationRuleTest.java`
- Create: `app/src/test/java/io/casehub/life/app/cbr/adapt/HealthAdaptationRuleTest.java`
- Create: `app/src/test/java/io/casehub/life/app/cbr/adapt/FinancialAdaptationRuleTest.java`
- Create: `app/src/test/java/io/casehub/life/app/cbr/adapt/AppointmentCycleAdaptationRuleTest.java`
- Create: `app/src/test/java/io/casehub/life/app/cbr/adapt/TravelPlanAdaptationRuleTest.java`
- Create: `app/src/test/java/io/casehub/life/app/cbr/adapt/SeverityScalingTest.java`

**Interfaces:**
- Consumes: `LifeAdaptationRule` (from Task 1), `AdaptedStep`, `AdaptationAction`, `FeatureValue`, `ScoredCbrCase<PlanCbrCase>`, `PlanTrace`
- Produces: 6 `@ApplicationScoped` CDI beans implementing `LifeAdaptationRule`, each returning its `caseType()` for dispatcher routing

Each rule is a pure function. Test pattern for each:
1. Feature delta fires correct action (BOOSTED/SUPPRESSED)
2. No delta → all RETAINED
3. Missing features → graceful (RETAINED)
4. Empty plan trace → empty list
5. Failed outcome → SUPPRESS

- [ ] **Step 1: Write SeverityScaling + test**

`SeverityScaling.scale(double retrievedSeverity, double currentSeverity, int basePriority) → int`
Maps severity delta to priority adjustment. Shared by Health and Appointment rules.

```java
package io.casehub.life.app.cbr.adapt;

public final class SeverityScaling {
    private SeverityScaling() {}

    public static int scale(double retrievedSeverity, double currentSeverity, int basePriority) {
        if (currentSeverity <= 0 || retrievedSeverity <= 0) return basePriority;
        double ratio = currentSeverity / retrievedSeverity;
        if (ratio > 1.5) return Math.min(basePriority + 3, 10);
        if (ratio > 1.0) return Math.min(basePriority + 1, 10);
        return basePriority;
    }
}
```

Test: ratios >1.5, >1.0, ≤1.0, zero/negative inputs.

- [ ] **Step 2: Write ContractorAdaptationRule + test**

Rule logic per spec:
- Season delta (winter + heating/plumbing): BOOST request-quote, tighten slaHours
- Budget delta: scale quotedCost by ratio
- Failed outcome: SUPPRESS

Must implement `LifePlanAdapter.CapabilityAware` for SPI inference. Known capabilities: `request-quote`, `watchdog-escalation`, `quote-received`, `job-monitoring`, `record-payment`, `contractor-sentinel`.

Test: `ContractorAdaptationRuleTest` — season=winter+heating boosts, budget delta scales, failed suppresses, no delta retains, missing features retains, empty trace empty list.

- [ ] **Step 3: Write HomeMaintenanceAdaptationRule + test**

Rule logic: seasonal SLA halving, cost delta, failed outcome suppression.
Known capabilities: from `home-maintenance.yaml`.
Test: winter halves SLA, cost delta, failed suppresses.

- [ ] **Step 4: Write HealthAdaptationRule + test**

Rule logic: severity scaling (via SeverityScaling), provider change annotation, SLA breach boost.
Known capabilities: from `care-coordination.yaml`.
Test: severity increase boosts, provider change annotates, SLA breach boosts.

- [ ] **Step 5: Write AppointmentCycleAdaptationRule + test**

Rule logic: severity scaling (shared), provider change, prep-time adjustment.
Known capabilities: from `appointment-cycle.yaml`.
Test: severity increase boosts, provider change annotates.

- [ ] **Step 6: Write FinancialAdaptationRule + test**

Rule logic: amount delta boost, escalation pattern detection.
Known capabilities: from `financial-review.yaml`.
Test: higher amount boosts, escalation flags.

- [ ] **Step 7: Write TravelPlanAdaptationRule + test**

Rule logic: budget scaling, seasonal pricing, rejected booking suppression.
Known capabilities: from `travel-plan.yaml`.
Test: budget delta scales, seasonal annotates, rejected suppresses.

- [ ] **Step 8: Run all adaptation rule tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="SeverityScalingTest,ContractorAdaptationRuleTest,HomeMaintenanceAdaptationRuleTest,HealthAdaptationRuleTest,AppointmentCycleAdaptationRuleTest,FinancialAdaptationRuleTest,TravelPlanAdaptationRuleTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/adapt/ app/src/test/java/io/casehub/life/app/cbr/adapt/
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#55): 6 domain adaptation rules + SeverityScaling helper with tests"
```

---

### Task 3: LifeCbrRetrievalResult + retrieveForAdaptation() + integration

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifeCbrRetrievalResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/cbr/LifeCbrSuggestionService.java` — add `retrieveForAdaptation()`
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java` — wire adaptation into `startCase()`
- Modify: `app/src/main/java/io/casehub/life/app/cbr/LifeCbrExperienceFormatter.java` — add `formatAdaptedPlan()`
- Modify: `app/src/main/java/io/casehub/life/app/cbr/CbrInputTransformer.java` — include adapted plan
- Modify: `app/src/test/java/io/casehub/life/app/cbr/LifeCbrSuggestionServiceTest.java` — add retrieveForAdaptation tests
- Modify: `app/src/test/java/io/casehub/life/app/cbr/LifeCbrExperienceFormatterTest.java` — add formatAdaptedPlan tests
- Modify: `app/src/test/java/io/casehub/life/app/cbr/CbrInputTransformerTest.java` — add adapted plan tests

**Interfaces:**
- Consumes: `LifePlanAdapter` (Task 1), `LifeCbrSuggestionService`, `CbrInputTransformer`, `LifeCbrExperienceFormatter`
- Produces: Full integration — `startCase()` retrieves, adapts, and writes `adaptedPlan` to context; workers see formatted adapted plan in `_cbrContext`

- [ ] **Step 1: Create LifeCbrRetrievalResult record**

```java
package io.casehub.life.app.cbr;

import io.casehub.life.api.CbrSuggestions;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import java.util.List;
import java.util.Map;

public record LifeCbrRetrievalResult(
    CbrSuggestions suggestions,
    List<ScoredCbrCase<PlanCbrCase>> cases,
    Map<String, FeatureValue> currentFeatures) {

    public static final LifeCbrRetrievalResult EMPTY =
        new LifeCbrRetrievalResult(CbrSuggestions.EMPTY, List.of(), Map.of());
}
```

- [ ] **Step 2: Write failing test for retrieveForAdaptation()**

Add to `LifeCbrSuggestionServiceTest`:
- `retrieveForAdaptation_singleCase_returnsWithEmptySuggestions` — ≥1 case, <2 means suggestions EMPTY but cases populated
- `retrieveForAdaptation_multipleCases_returnsBoth` — ≥2 cases, both suggestions and cases populated
- `retrieveForAdaptation_noExtraction_returnsEmpty` — no features → EMPTY

- [ ] **Step 3: Implement retrieveForAdaptation()**

New method on `LifeCbrSuggestionService` that shares retrieval logic with `suggest()` but uses ≥1 threshold for cases (≥2 for statistics). Returns `LifeCbrRetrievalResult`.

- [ ] **Step 4: Write failing test for formatAdaptedPlan()**

Add to `LifeCbrExperienceFormatterTest`:
- `formatAdaptedPlan_producesStructuredOutput` — BOOSTED/RETAINED/SUPPRESSED steps formatted correctly
- `formatAdaptedPlan_emptyPlan_returnsNull` — empty plan → null

- [ ] **Step 5: Implement formatAdaptedPlan()**

New method on `LifeCbrExperienceFormatter`:
```java
public @Nullable String formatAdaptedPlan(AdaptedPlan plan)
```

- [ ] **Step 6: Write failing test for CbrInputTransformer adapted plan**

Add to `CbrInputTransformerTest`:
- `apply_withAdaptedPlan_includesBoth` — input has `adaptedPlan` key → `_cbrContext` includes adapted plan section

- [ ] **Step 7: Update CbrInputTransformer**

Modify `apply()` to check input `JsonNode` for `adaptedPlan`, deserialize to `AdaptedPlan`, format via `formatAdaptedPlan()`, append to `_cbrContext`.

- [ ] **Step 8: Wire adaptation into LifeCaseService.startCase()**

Modify `startCase()` to call `retrieveForAdaptation()`, then `planAdapter.adapt()` on best match, write `adaptedPlan` to context per the threshold table (0 cases → nothing, 1 case → adaptedPlan only, ≥2 → both). Inject `LifePlanAdapter` into `LifeCaseService`. Fire `CbrAdaptationRecorded` CDI event after adaptation.

- [ ] **Step 9: Run all modified tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="LifeCbrSuggestionServiceTest,LifeCbrExperienceFormatterTest,CbrInputTransformerTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/ app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java app/src/test/java/io/casehub/life/app/cbr/
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#55): CBR adaptation integration — retrieveForAdaptation, formatAdaptedPlan, case start wiring"
```

---

### Task 4: CDI wiring + quarkus.arc config + integration test

**Files:**
- Modify: `app/src/main/resources/application.properties` — add `LifePlanAdapter` to `selected-alternatives`
- Create: `app/src/test/java/io/casehub/life/app/cbr/LifePlanAdapterQuarkusTest.java`

**Interfaces:**
- Consumes: All from Tasks 1-3
- Produces: Verified CDI wiring — `PlanAdapter` resolves to `LifePlanAdapter`

- [ ] **Step 1: Add LifePlanAdapter to selected-alternatives**

Add `io.casehub.life.app.cbr.LifePlanAdapter` to `quarkus.arc.selected-alternatives` in `application.properties`.

- [ ] **Step 2: Write integration test**

```java
@QuarkusTest
class LifePlanAdapterQuarkusTest {

    @Inject PlanAdapter planAdapter;

    @Test
    void planAdapter_resolvesToLifePlanAdapter() {
        assertInstanceOf(LifePlanAdapter.class, planAdapter);
    }
}
```

- [ ] **Step 3: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifePlanAdapterQuarkusTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode -am`
Expected: all existing tests still pass, no regressions

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/resources/application.properties app/src/test/java/io/casehub/life/app/cbr/LifePlanAdapterQuarkusTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#55): CDI wiring — LifePlanAdapter as @Alternative + integration test"
```

---

### Task 5: File engine issue + update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` — update Layer 8, What This Project Owns, capability tags

**Interfaces:**
- Consumes: None (documentation task)
- Produces: Engine issue for PlanAdapter wiring, updated project docs

- [ ] **Step 1: File engine issue**

Create issue on `casehubio/engine` for PlanAdapter wiring into CbrRetrievalService pipeline.

- [ ] **Step 2: Update CLAUDE.md**

Add Layer 8 adaptation entries:
- `LifeAdaptationRule` SPI + 6 impls in Layer 8 additions
- `LifePlanAdapter` in Layer 8 additions
- `LifeCbrRetrievalResult` in Layer 8 additions
- `SeverityScaling` utility in Layer 8 additions
- Update Foundation Gates table for CBR adaptation

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/life commit -m "docs(#55): update CLAUDE.md — Layer 8 adaptation rules, engine issue filed"
```
