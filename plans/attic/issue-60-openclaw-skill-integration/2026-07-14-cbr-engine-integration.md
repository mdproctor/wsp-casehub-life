# CBR Engine Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #56 — feat: CBR engine integration — wire suggestions into case plan execution
**Issue group:** #56

**Goal:** Wire CBR suggestions into case plan execution so that past case outcomes influence future case parameters through two complementary paths: automatic prompt enrichment and deterministic structured calibration.

**Architecture:** Path 1 uses `Agent.inputTransformer` to read `WorkerExecutionContext.current().experiences()` at worker execution time and merge formatted CBR context into the LLM user message. Path 2 queries `CbrCaseMemoryStore` at case start via `LifeCbrSuggestionService`, computes feature statistics, and writes `cbrCalibration` to the case context for deterministic parameter calibration.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-api 0.2-SNAPSHOT, casehub-neocortex-memory-api 0.2-SNAPSHOT

## Global Constraints

- `Agent` is `final`; `systemPrompt` is immutable at construction — the only dynamic hook is `inputTransformer`
- `WorkerExecutionContext.set(context)` is called before `agent.execute(inputData)` — `inputTransformer` runs in scope
- CBR is advisory — failures must never prevent case starts or worker execution
- Minimum 2 similar cases required for `FeatureStatistics`; below that → `CbrSuggestions.EMPTY`
- Percentile computation uses **nearest-rank** interpolation: `Math.ceil(rank * sampleCount) - 1`
- `LifeCaseType.caseName()` returns the YAML case type name (not `yamlName()`)
- `TENANT_ID = "life-personal"` — interim constant shared with retention writers
- `FeatureValue.NumberVal` identifies numeric features for statistical extraction
- Pre-release: breaking changes to existing signatures cost nothing

---

### Task 1: API Records — `FeatureStatistics` and `CbrSuggestions`

Pure value records in `api/` with no framework dependencies. Foundation for Tasks 2–4.

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/FeatureStatistics.java`
- Create: `api/src/main/java/io/casehub/life/api/CbrSuggestions.java`
- Test: `api/src/test/java/io/casehub/life/api/FeatureStatisticsTest.java`
- Test: `api/src/test/java/io/casehub/life/api/CbrSuggestionsTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `FeatureStatistics(double min, double max, double median, double p75, int sampleCount)` — used by Task 3. `FeatureStatistics.compute(double[] values)` — static factory computing all fields from a sorted array. `CbrSuggestions(Map<String, FeatureStatistics> featureStats, double historicalSuccessRate, int experienceCount, double averageSimilarity)` — used by Tasks 3, 4. `CbrSuggestions.EMPTY` — static constant. `CbrSuggestions.isEmpty()` — returns `experienceCount == 0`.

- [ ] **Step 1: Write FeatureStatisticsTest**

```java
package io.casehub.life.api;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class FeatureStatisticsTest {

    @Test
    void compute_oddSampleCount_nearestRankPercentiles() {
        // [100, 200, 300, 400, 500] → min=100, max=500
        // median: ceil(0.5*5)-1 = 2 → 300
        // p75: ceil(0.75*5)-1 = 3 → 400
        var stats = FeatureStatistics.compute(new double[]{100, 200, 300, 400, 500});
        assertEquals(100.0, stats.min());
        assertEquals(500.0, stats.max());
        assertEquals(300.0, stats.median());
        assertEquals(400.0, stats.p75());
        assertEquals(5, stats.sampleCount());
    }

    @Test
    void compute_evenSampleCount_nearestRankPercentiles() {
        // [100, 200, 300, 400] → min=100, max=400
        // median: ceil(0.5*4)-1 = 1 → 200
        // p75: ceil(0.75*4)-1 = 2 → 300
        var stats = FeatureStatistics.compute(new double[]{100, 200, 300, 400});
        assertEquals(100.0, stats.min());
        assertEquals(400.0, stats.max());
        assertEquals(200.0, stats.median());
        assertEquals(300.0, stats.p75());
        assertEquals(4, stats.sampleCount());
    }

    @Test
    void compute_singleElement() {
        var stats = FeatureStatistics.compute(new double[]{42.0});
        assertEquals(42.0, stats.min());
        assertEquals(42.0, stats.max());
        assertEquals(42.0, stats.median());
        assertEquals(42.0, stats.p75());
        assertEquals(1, stats.sampleCount());
    }

    @Test
    void compute_allEqualValues() {
        var stats = FeatureStatistics.compute(new double[]{7.0, 7.0, 7.0});
        assertEquals(7.0, stats.min());
        assertEquals(7.0, stats.max());
        assertEquals(7.0, stats.median());
        assertEquals(7.0, stats.p75());
    }

    @Test
    void compute_twoElements() {
        // [10, 20] → median: ceil(0.5*2)-1 = 0 → 10; p75: ceil(0.75*2)-1 = 1 → 20
        var stats = FeatureStatistics.compute(new double[]{10, 20});
        assertEquals(10.0, stats.min());
        assertEquals(20.0, stats.max());
        assertEquals(10.0, stats.median());
        assertEquals(20.0, stats.p75());
    }

    @Test
    void compute_unsortedInput_sortsInternally() {
        var stats = FeatureStatistics.compute(new double[]{500, 100, 300, 200, 400});
        assertEquals(100.0, stats.min());
        assertEquals(500.0, stats.max());
        assertEquals(300.0, stats.median());
        assertEquals(400.0, stats.p75());
    }
}
```

- [ ] **Step 2: Write CbrSuggestionsTest**

```java
package io.casehub.life.api;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class CbrSuggestionsTest {

    @Test
    void empty_constant_isCorrect() {
        assertTrue(CbrSuggestions.EMPTY.isEmpty());
        assertEquals(0, CbrSuggestions.EMPTY.experienceCount());
        assertEquals(0.0, CbrSuggestions.EMPTY.historicalSuccessRate());
        assertEquals(0.0, CbrSuggestions.EMPTY.averageSimilarity());
        assertTrue(CbrSuggestions.EMPTY.featureStats().isEmpty());
    }

    @Test
    void isEmpty_withExperiences_returnsFalse() {
        var stats = new FeatureStatistics(10, 20, 15, 18, 3);
        var suggestions = new CbrSuggestions(Map.of("cost", stats), 0.8, 3, 0.75);
        assertFalse(suggestions.isEmpty());
    }

    @Test
    void isEmpty_zeroCount_returnsTrue() {
        var suggestions = new CbrSuggestions(Map.of(), 0.0, 0, 0.0);
        assertTrue(suggestions.isEmpty());
    }
}
```

- [ ] **Step 3: Run tests — expect FAIL (classes don't exist)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="FeatureStatisticsTest,CbrSuggestionsTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 4: Implement FeatureStatistics**

Create `api/src/main/java/io/casehub/life/api/FeatureStatistics.java`:

```java
package io.casehub.life.api;

import java.util.Arrays;

public record FeatureStatistics(double min, double max, double median, double p75, int sampleCount) {

    public static FeatureStatistics compute(double[] values) {
        double[] sorted = values.clone();
        Arrays.sort(sorted);
        int n = sorted.length;
        return new FeatureStatistics(
                sorted[0],
                sorted[n - 1],
                nearestRank(sorted, 0.5),
                nearestRank(sorted, 0.75),
                n);
    }

    private static double nearestRank(double[] sorted, double rank) {
        int index = (int) Math.ceil(rank * sorted.length) - 1;
        return sorted[Math.max(0, index)];
    }
}
```

- [ ] **Step 5: Implement CbrSuggestions**

Create `api/src/main/java/io/casehub/life/api/CbrSuggestions.java`:

```java
package io.casehub.life.api;

import java.util.Map;

public record CbrSuggestions(
        Map<String, FeatureStatistics> featureStats,
        double historicalSuccessRate,
        int experienceCount,
        double averageSimilarity) {

    public static final CbrSuggestions EMPTY = new CbrSuggestions(Map.of(), 0.0, 0, 0.0);

    public boolean isEmpty() {
        return experienceCount == 0;
    }
}
```

- [ ] **Step 6: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="FeatureStatisticsTest,CbrSuggestionsTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 7: Install api and commit**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode -DskipTests
git -C /Users/mdproctor/claude/casehub/life add api/src/main/java/io/casehub/life/api/FeatureStatistics.java api/src/main/java/io/casehub/life/api/CbrSuggestions.java api/src/test/java/io/casehub/life/api/FeatureStatisticsTest.java api/src/test/java/io/casehub/life/api/CbrSuggestionsTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#56): FeatureStatistics + CbrSuggestions — API records for CBR calibration

Refs #56"
```

---

### Task 2: LifeCbrFeatureExtractor — Shared Feature Extraction

Consolidates the duplicated feature extraction pipeline from `LifeCaseOutcomeCbrWriter` and `LifeRoutingOutcomeRecorder`. Refactors both callers to delegate.

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifeCbrFeatureExtractor.java`
- Modify: `app/src/main/java/io/casehub/life/app/cbr/LifeCaseOutcomeCbrWriter.java` — remove `extractFeatures()` and `unwrap()`, inject `LifeCbrFeatureExtractor`
- Modify: `app/src/main/java/io/casehub/life/app/cbr/LifeRoutingOutcomeRecorder.java` — remove `extractFeatures()`, inject `LifeCbrFeatureExtractor`
- Create: `app/src/test/java/io/casehub/life/app/cbr/LifeCbrFeatureExtractorTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/cbr/LifeCaseOutcomeCbrWriterTest.java` — mock `LifeCbrFeatureExtractor` instead of testing inline extraction
- Modify: `app/src/test/java/io/casehub/life/app/cbr/LifeRoutingOutcomeRecorderTest.java` — mock `LifeCbrFeatureExtractor`

**Interfaces:**
- Consumes: `CaseDefinitionRegistry.findByName(String)`, `JQEvaluator.eval(String, JsonNode)`, `FeatureValue.toFeatureMap(Map)`
- Produces: `LifeCbrFeatureExtractor.extract(String caseType, JsonNode context)` → `Optional<ExtractionResult>`. `ExtractionResult(CbrConfig config, Map<String, FeatureValue> features)` — used by Tasks 3, and refactored callers in `LifeCaseOutcomeCbrWriter`/`LifeRoutingOutcomeRecorder`.

- [ ] **Step 1: Write LifeCbrFeatureExtractorTest**

```java
package io.casehub.life.app.cbr;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.model.cbr.JqFeatureExtractor;
import io.casehub.api.model.cbr.LambdaFeatureExtractor;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.platform.expression.JQEvaluator;
import io.casehub.platform.expression.ValidationResult;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class LifeCbrFeatureExtractorTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private CaseDefinitionRegistry registry;
    private JQEvaluator jqEvaluator;
    private LifeCbrFeatureExtractor extractor;

    @BeforeEach
    void setup() {
        registry = mock(CaseDefinitionRegistry.class);
        jqEvaluator = mock(JQEvaluator.class);
        extractor = new LifeCbrFeatureExtractor(registry, jqEvaluator);
    }

    @Test
    void extract_noDefinition_returnsEmpty() {
        when(registry.findByName("travel-plan")).thenReturn(Optional.empty());
        assertTrue(extractor.extract("travel-plan", MAPPER.createObjectNode()).isEmpty());
    }

    @Test
    void extract_noCbrConfig_returnsEmpty() {
        var def = mock(CaseDefinition.class);
        when(def.getCbrConfig()).thenReturn(null);
        when(registry.findByName("travel-plan")).thenReturn(Optional.of(def));
        assertTrue(extractor.extract("travel-plan", MAPPER.createObjectNode()).isEmpty());
    }

    @Test
    void extract_lambdaExtractor_returnsEmpty() {
        var def = mock(CaseDefinition.class);
        var config = mock(CbrConfig.class);
        when(config.featureExtractor()).thenReturn(mock(LambdaFeatureExtractor.class));
        when(def.getCbrConfig()).thenReturn(config);
        when(registry.findByName("travel-plan")).thenReturn(Optional.of(def));
        assertTrue(extractor.extract("travel-plan", MAPPER.createObjectNode()).isEmpty());
    }

    @Test
    void extract_jqExtractor_extractsFeatures() {
        var def = mock(CaseDefinition.class);
        var config = mock(CbrConfig.class);
        var jq = mock(JqFeatureExtractor.class);
        when(jq.featureExpressions()).thenReturn(Map.of(
                "budget", ".request.budget",
                "destination", ".request.destination"));
        when(config.featureExtractor()).thenReturn(jq);
        when(def.getCbrConfig()).thenReturn(config);
        when(registry.findByName("travel-plan")).thenReturn(Optional.of(def));

        ObjectNode context = MAPPER.createObjectNode();
        ObjectNode request = context.putObject("request");
        request.put("budget", 2000);
        request.put("destination", "Barcelona");

        JsonNode budgetNode = MAPPER.valueToTree(2000);
        JsonNode destNode = MAPPER.valueToTree("Barcelona");
        when(jqEvaluator.eval(".request.budget", context))
                .thenReturn(new ValidationResult(true, List.of(budgetNode), List.of()));
        when(jqEvaluator.eval(".request.destination", context))
                .thenReturn(new ValidationResult(true, List.of(destNode), List.of()));

        var result = extractor.extract("travel-plan", context);
        assertTrue(result.isPresent());
        assertEquals(config, result.get().config());
        assertEquals(2, result.get().features().size());
    }

    @Test
    void extract_nullJqResult_skipsFeature() {
        var def = mock(CaseDefinition.class);
        var config = mock(CbrConfig.class);
        var jq = mock(JqFeatureExtractor.class);
        when(jq.featureExpressions()).thenReturn(Map.of("budget", ".request.budget"));
        when(config.featureExtractor()).thenReturn(jq);
        when(def.getCbrConfig()).thenReturn(config);
        when(registry.findByName("travel-plan")).thenReturn(Optional.of(def));

        ObjectNode context = MAPPER.createObjectNode();
        JsonNode nullNode = MAPPER.nullNode();
        when(jqEvaluator.eval(".request.budget", context))
                .thenReturn(new ValidationResult(true, List.of(nullNode), List.of()));

        var result = extractor.extract("travel-plan", context);
        assertTrue(result.isPresent());
        assertTrue(result.get().features().isEmpty());
    }

    @Test
    void extract_failedJqEval_skipsFeature() {
        var def = mock(CaseDefinition.class);
        var config = mock(CbrConfig.class);
        var jq = mock(JqFeatureExtractor.class);
        when(jq.featureExpressions()).thenReturn(Map.of("budget", ".bad.path"));
        when(config.featureExtractor()).thenReturn(jq);
        when(def.getCbrConfig()).thenReturn(config);
        when(registry.findByName("travel-plan")).thenReturn(Optional.of(def));

        ObjectNode context = MAPPER.createObjectNode();
        when(jqEvaluator.eval(".bad.path", context))
                .thenReturn(new ValidationResult(false, List.of(), List.of("error")));

        var result = extractor.extract("travel-plan", context);
        assertTrue(result.isPresent());
        assertTrue(result.get().features().isEmpty());
    }
}
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeCbrFeatureExtractorTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 3: Implement LifeCbrFeatureExtractor**

Create `app/src/main/java/io/casehub/life/app/cbr/LifeCbrFeatureExtractor.java`:

```java
package io.casehub.life.app.cbr;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.model.cbr.JqFeatureExtractor;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.platform.expression.JQEvaluator;
import io.casehub.platform.expression.ValidationResult;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Optional;

@ApplicationScoped
public class LifeCbrFeatureExtractor {

    public record ExtractionResult(CbrConfig config, Map<String, FeatureValue> features) {}

    private final CaseDefinitionRegistry registry;
    private final JQEvaluator jqEvaluator;

    @Inject
    public LifeCbrFeatureExtractor(CaseDefinitionRegistry registry, JQEvaluator jqEvaluator) {
        this.registry = registry;
        this.jqEvaluator = jqEvaluator;
    }

    public Optional<ExtractionResult> extract(String caseType, JsonNode context) {
        var definition = registry.findByName(caseType).orElse(null);
        if (definition == null || definition.getCbrConfig() == null) return Optional.empty();

        CbrConfig config = definition.getCbrConfig();
        if (!(config.featureExtractor() instanceof JqFeatureExtractor jq)) {
            return Optional.empty();
        }

        Map<String, Object> rawFeatures = new LinkedHashMap<>();
        for (var entry : jq.featureExpressions().entrySet()) {
            ValidationResult result = jqEvaluator.eval(entry.getValue(), context);
            if (!result.ok() || result.output().isEmpty()) continue;
            JsonNode node = result.output().get(0);
            if (node.isNull()) continue;
            rawFeatures.put(entry.getKey(), unwrap(node));
        }

        return Optional.of(new ExtractionResult(config, FeatureValue.toFeatureMap(rawFeatures)));
    }

    static Object unwrap(JsonNode node) {
        if (node.isTextual()) return node.asText();
        if (node.isInt()) return node.asInt();
        if (node.isLong()) return node.asLong();
        if (node.isDouble() || node.isFloat()) return node.asDouble();
        if (node.isBoolean()) return node.asBoolean();
        return node.asText();
    }
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeCbrFeatureExtractorTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 5: Refactor LifeCaseOutcomeCbrWriter — replace inline extraction with LifeCbrFeatureExtractor**

Use `ide_edit_member` to:
1. Replace the constructor to inject `LifeCbrFeatureExtractor` instead of `CaseDefinitionRegistry` and `JQEvaluator`
2. Replace `onOutcome()` body to call `featureExtractor.extract()` instead of inline logic
3. Remove `extractFeatures()` method via `ide_refactor_safe_delete`
4. Remove `unwrap()` method via `ide_refactor_safe_delete` (now lives on `LifeCbrFeatureExtractor`)

Key changes in `onOutcome()`:
```java
// Before: inline registry lookup, CbrConfig check, JQ evaluation
// After:
var extraction = featureExtractor.extract(event.caseType(),
        MAPPER.valueToTree(event.caseFileSnapshot()));
if (extraction.isEmpty()) return;
var result = extraction.get();
CbrConfig config = result.config();
// ... use result.features() directly
```

- [ ] **Step 6: Refactor LifeRoutingOutcomeRecorder — same pattern**

Use `ide_edit_member` to:
1. Replace constructor to inject `LifeCbrFeatureExtractor` instead of `CaseDefinitionRegistry` and `JQEvaluator`
2. Replace `record()` body to use `featureExtractor.extract()`
3. Remove `extractFeatures()` method

Key changes in `record()`:
```java
// Before: inline registry lookup, CbrConfig check, JQ evaluation
// After:
var extraction = featureExtractor.extract(caseType, context.caseContext());
if (extraction.isEmpty()) return null;
var result = extraction.get();
CbrConfig config = result.config();
// ... use result.features() and config.domain()
```

- [ ] **Step 7: Update existing tests**

Update `LifeCaseOutcomeCbrWriterTest` and `LifeRoutingOutcomeRecorderTest`:
- Replace mocks for `CaseDefinitionRegistry` and `JQEvaluator` with mock for `LifeCbrFeatureExtractor`
- Constructor calls updated to match new signature
- Verify `featureExtractor.extract()` is called with correct arguments
- Feature extraction edge cases now covered by `LifeCbrFeatureExtractorTest`

- [ ] **Step 8: Run all CBR tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest="LifeCbrFeatureExtractorTest,LifeCaseOutcomeCbrWriterTest,LifeRoutingOutcomeRecorderTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/LifeCbrFeatureExtractor.java app/src/main/java/io/casehub/life/app/cbr/LifeCaseOutcomeCbrWriter.java app/src/main/java/io/casehub/life/app/cbr/LifeRoutingOutcomeRecorder.java app/src/test/java/io/casehub/life/app/cbr/LifeCbrFeatureExtractorTest.java app/src/test/java/io/casehub/life/app/cbr/LifeCaseOutcomeCbrWriterTest.java app/src/test/java/io/casehub/life/app/cbr/LifeRoutingOutcomeRecorderTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "refactor(#56): LifeCbrFeatureExtractor — consolidate feature extraction from 3 sites

Replaces duplicated extractFeatures()/unwrap() in LifeCaseOutcomeCbrWriter
and LifeRoutingOutcomeRecorder with shared extractor. Returns
ExtractionResult(CbrConfig, features) so callers get both config and
features from a single call.

Refs #56"
```

---

### Task 3: LifeCbrSuggestionService — Structured Calibration

Queries the CBR store and computes feature statistics from similar past cases.

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifeCbrSuggestionService.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/LifeCbrSuggestionServiceTest.java`

**Interfaces:**
- Consumes: `LifeCbrFeatureExtractor.extract(String, JsonNode)` from Task 2. `CbrCaseMemoryStore.retrieveSimilar(CbrQuery, Class)`. `CbrQuery.of()` + `.withWeights()` + `.withMinSimilarity()` + `.withVectorWeight()`. `FeatureStatistics.compute(double[])` from Task 1. `FeatureValue.NumberVal` for numeric detection.
- Produces: `LifeCbrSuggestionService.suggest(LifeCaseType, Map<String, Object>)` → `CbrSuggestions` — used by Task 4.

- [ ] **Step 1: Write LifeCbrSuggestionServiceTest**

```java
package io.casehub.life.app.cbr;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.life.api.CbrSuggestions;
import io.casehub.life.api.LifeCaseType;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class LifeCbrSuggestionServiceTest {

    private LifeCbrFeatureExtractor featureExtractor;
    private CbrCaseMemoryStore cbrStore;
    private LifeCbrSuggestionService service;

    @BeforeEach
    void setup() {
        featureExtractor = mock(LifeCbrFeatureExtractor.class);
        cbrStore = mock(CbrCaseMemoryStore.class);
        service = new LifeCbrSuggestionService(featureExtractor, cbrStore, new ObjectMapper());
    }

    @Test
    void suggest_noExtraction_returnsEmpty() {
        when(featureExtractor.extract(eq("travel-plan"), any())).thenReturn(Optional.empty());
        var result = service.suggest(LifeCaseType.TRAVEL_PLAN, Map.of());
        assertTrue(result.isEmpty());
    }

    @Test
    void suggest_fewerThanTwoResults_returnsEmpty() {
        var config = mockConfig();
        when(featureExtractor.extract(eq("travel-plan"), any()))
                .thenReturn(Optional.of(new LifeCbrFeatureExtractor.ExtractionResult(
                        config, Map.of("budget", FeatureValue.number(2000)))));
        when(cbrStore.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of(scoredCase(2000, "COMPLETED", 0.8)));
        var result = service.suggest(LifeCaseType.TRAVEL_PLAN, Map.of());
        assertTrue(result.isEmpty());
    }

    @Test
    void suggest_multipleResults_computesStatistics() {
        var config = mockConfig();
        when(featureExtractor.extract(eq("travel-plan"), any()))
                .thenReturn(Optional.of(new LifeCbrFeatureExtractor.ExtractionResult(
                        config, Map.of("budget", FeatureValue.number(2000)))));
        when(cbrStore.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of(
                        scoredCase(1500, "COMPLETED", 0.9),
                        scoredCase(2000, "COMPLETED", 0.85),
                        scoredCase(2500, "FAULTED", 0.7)));

        var result = service.suggest(LifeCaseType.TRAVEL_PLAN, Map.of());
        assertFalse(result.isEmpty());
        assertEquals(3, result.experienceCount());
        assertEquals(2.0 / 3.0, result.historicalSuccessRate(), 0.001);

        var budgetStats = result.featureStats().get("budget");
        assertNotNull(budgetStats);
        assertEquals(1500.0, budgetStats.min());
        assertEquals(2500.0, budgetStats.max());
        assertEquals(3, budgetStats.sampleCount());
    }

    @Test
    void suggest_categoricalFeaturesExcluded() {
        var config = mockConfig();
        when(featureExtractor.extract(eq("travel-plan"), any()))
                .thenReturn(Optional.of(new LifeCbrFeatureExtractor.ExtractionResult(
                        config, Map.of("budget", FeatureValue.number(2000)))));
        when(cbrStore.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of(
                        scoredCaseWithStringFeature("Barcelona", 1500, 0.9),
                        scoredCaseWithStringFeature("Madrid", 2000, 0.8)));

        var result = service.suggest(LifeCaseType.TRAVEL_PLAN, Map.of());
        assertNull(result.featureStats().get("destination"));
        assertNotNull(result.featureStats().get("budget"));
    }

    @Test
    void suggest_exceptionReturnsEmpty() {
        when(featureExtractor.extract(any(), any())).thenThrow(new RuntimeException("Qdrant down"));
        var result = service.suggest(LifeCaseType.TRAVEL_PLAN, Map.of());
        assertTrue(result.isEmpty());
    }

    @Test
    void suggest_averageSimilarity_computed() {
        var config = mockConfig();
        when(featureExtractor.extract(eq("travel-plan"), any()))
                .thenReturn(Optional.of(new LifeCbrFeatureExtractor.ExtractionResult(
                        config, Map.of("budget", FeatureValue.number(2000)))));
        when(cbrStore.retrieveSimilar(any(), eq(PlanCbrCase.class)))
                .thenReturn(List.of(
                        scoredCase(1500, "COMPLETED", 0.9),
                        scoredCase(2000, "COMPLETED", 0.7)));

        var result = service.suggest(LifeCaseType.TRAVEL_PLAN, Map.of());
        assertEquals(0.8, result.averageSimilarity(), 0.001);
    }

    private CbrConfig mockConfig() {
        var config = mock(CbrConfig.class);
        when(config.topK()).thenReturn(5);
        when(config.minSimilarity()).thenReturn(0.3);
        when(config.vectorWeight()).thenReturn(0.0);
        when(config.domain()).thenReturn("casehubio/life/travel");
        when(config.weights()).thenReturn(Map.of("budget", 1.5));
        return config;
    }

    private ScoredCbrCase<PlanCbrCase> scoredCase(double budget, String outcome, double score) {
        var cbrCase = new PlanCbrCase("problem", "solution", outcome, null,
                Map.of("budget", FeatureValue.number(budget)), List.of());
        return new ScoredCbrCase<>(cbrCase, score);
    }

    private ScoredCbrCase<PlanCbrCase> scoredCaseWithStringFeature(String dest, double budget, double score) {
        var cbrCase = new PlanCbrCase("problem", "solution", "COMPLETED", null,
                Map.of("destination", FeatureValue.string(dest),
                       "budget", FeatureValue.number(budget)),
                List.of());
        return new ScoredCbrCase<>(cbrCase, score);
    }
}
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeCbrSuggestionServiceTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 3: Implement LifeCbrSuggestionService**

Create `app/src/main/java/io/casehub/life/app/cbr/LifeCbrSuggestionService.java`:

```java
package io.casehub.life.app.cbr;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.life.api.CbrSuggestions;
import io.casehub.life.api.FeatureStatistics;
import io.casehub.life.api.LifeCaseType;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.CbrQuery;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class LifeCbrSuggestionService {

    private static final Logger LOG = Logger.getLogger(LifeCbrSuggestionService.class);
    private static final String TENANT_ID = "life-personal";

    private final LifeCbrFeatureExtractor featureExtractor;
    private final CbrCaseMemoryStore cbrStore;
    private final ObjectMapper objectMapper;

    @Inject
    public LifeCbrSuggestionService(LifeCbrFeatureExtractor featureExtractor,
                                     CbrCaseMemoryStore cbrStore,
                                     ObjectMapper objectMapper) {
        this.featureExtractor = featureExtractor;
        this.cbrStore = cbrStore;
        this.objectMapper = objectMapper;
    }

    public CbrSuggestions suggest(LifeCaseType caseType, Map<String, Object> initialContext) {
        try {
            JsonNode contextNode = objectMapper.valueToTree(initialContext);
            var extraction = featureExtractor.extract(caseType.caseName(), contextNode);
            if (extraction.isEmpty()) return CbrSuggestions.EMPTY;

            var result = extraction.get();
            CbrConfig config = result.config();

            CbrQuery query = CbrQuery.of(
                    TENANT_ID,
                    new MemoryDomain(config.domain()),
                    caseType.caseName(),
                    result.features(),
                    config.topK())
                .withWeights(config.weights())
                .withMinSimilarity(config.minSimilarity())
                .withVectorWeight(config.vectorWeight());

            List<ScoredCbrCase<PlanCbrCase>> cases = cbrStore.retrieveSimilar(query, PlanCbrCase.class);
            if (cases.size() < 2) return CbrSuggestions.EMPTY;

            Map<String, FeatureStatistics> featureStats = computeFeatureStats(cases);
            double successRate = computeSuccessRate(cases);
            double avgSimilarity = cases.stream().mapToDouble(ScoredCbrCase::score).average().orElse(0.0);

            return new CbrSuggestions(featureStats, successRate, cases.size(), avgSimilarity);
        } catch (Exception e) {
            LOG.warnf(e, "CBR suggestion failed for %s — returning empty", caseType);
            return CbrSuggestions.EMPTY;
        }
    }

    private Map<String, FeatureStatistics> computeFeatureStats(List<ScoredCbrCase<PlanCbrCase>> cases) {
        Map<String, List<Double>> numericValues = new LinkedHashMap<>();
        for (var scored : cases) {
            for (var entry : scored.cbrCase().features().entrySet()) {
                if (entry.getValue() instanceof FeatureValue.NumberVal num) {
                    numericValues.computeIfAbsent(entry.getKey(), k -> new ArrayList<>())
                            .add(num.value());
                }
            }
        }

        Map<String, FeatureStatistics> stats = new LinkedHashMap<>();
        for (var entry : numericValues.entrySet()) {
            double[] values = entry.getValue().stream().mapToDouble(Double::doubleValue).toArray();
            stats.put(entry.getKey(), FeatureStatistics.compute(values));
        }
        return stats;
    }

    private double computeSuccessRate(List<ScoredCbrCase<PlanCbrCase>> cases) {
        long completed = cases.stream()
                .filter(c -> "COMPLETED".equals(c.cbrCase().outcome()))
                .count();
        return (double) completed / cases.size();
    }
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeCbrSuggestionServiceTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/LifeCbrSuggestionService.java app/src/test/java/io/casehub/life/app/cbr/LifeCbrSuggestionServiceTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#56): LifeCbrSuggestionService — structured CBR calibration at case start

Queries CbrCaseMemoryStore via shared LifeCbrFeatureExtractor, computes
FeatureStatistics (nearest-rank percentiles) for numeric features, returns
CbrSuggestions with historicalSuccessRate and averageSimilarity. Wraps
entire pipeline in try/catch — CBR failure never prevents case start.

Refs #56"
```

---

### Task 4: Prompt Enrichment — LifeCbrExperienceFormatter, CbrInputTransformer, and LifeTypedCaseHub Integration

Builds Path 1 (automatic per-worker-execution CBR enrichment) and wires both paths into the case hub infrastructure.

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifeCbrExperienceFormatter.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/CbrInputTransformer.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeTypedCaseHub.java` — inject formatter, create transformer, update `agentWorker()`
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java` — inject `LifeCbrSuggestionService`, call `suggest()` between Phase 1 and Phase 2
- Test: `app/src/test/java/io/casehub/life/app/cbr/LifeCbrExperienceFormatterTest.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/CbrInputTransformerTest.java`

**Interfaces:**
- Consumes: `LifeCbrSuggestionService.suggest()` from Task 3. `WorkerExecutionContext.current()` → `WorkerContext.experiences()`. `RetrievedExperience` fields: `problem`, `solution`, `outcome`, `similarityScore`, `features`, `featureSimilarities`, `planTrace`.
- Produces: `LifeCbrExperienceFormatter.format(List<RetrievedExperience>)` → `@Nullable String`. `CbrInputTransformer` — `UnaryOperator<JsonNode>`. `LifeTypedCaseHub.agentWorker()` — updated method with `inputTransformer` and `CBR_SYSTEM_PROMPT_SUFFIX`. `LifeCaseService.startCase()` — enriched with `cbrCalibration`.

- [ ] **Step 1: Write LifeCbrExperienceFormatterTest**

```java
package io.casehub.life.app.cbr;

import io.casehub.api.spi.routing.RetrievedExperience;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class LifeCbrExperienceFormatterTest {

    private final LifeCbrExperienceFormatter formatter = new LifeCbrExperienceFormatter();

    @Test
    void format_emptyList_returnsNull() {
        assertNull(formatter.format(List.of()));
    }

    @Test
    void format_singleExperience_includesProblemSolutionOutcome() {
        var exp = new RetrievedExperience("A problem", "A solution", "COMPLETED",
                0.9, 0.85, Map.of(), List.of(), Map.of());
        String result = formatter.format(List.of(exp));
        assertNotNull(result);
        assertTrue(result.contains("A problem"));
        assertTrue(result.contains("A solution"));
        assertTrue(result.contains("COMPLETED"));
        assertTrue(result.contains("0.85"));
    }

    @Test
    void format_multipleExperiences_sortedBySimilarityDescending() {
        var low = new RetrievedExperience("Low", "sol", "COMPLETED", null,
                0.5, Map.of(), List.of(), Map.of());
        var high = new RetrievedExperience("High", "sol", "COMPLETED", null,
                0.9, Map.of(), List.of(), Map.of());
        String result = formatter.format(List.of(low, high));
        assertTrue(result.indexOf("High") < result.indexOf("Low"));
    }

    @Test
    void format_featureSimilarities_rendered() {
        var exp = new RetrievedExperience("prob", "sol", "COMPLETED", null,
                0.8, Map.of("cost", 500), List.of(),
                Map.of("cost", 0.95, "type", 1.0));
        String result = formatter.format(List.of(exp));
        assertTrue(result.contains("cost"));
        assertTrue(result.contains("0.95"));
    }

    @Test
    void format_emptyFeatureSimilarities_lineOmitted() {
        var exp = new RetrievedExperience("prob", "sol", "COMPLETED", null,
                0.8, Map.of("cost", 500), List.of(), Map.of());
        String result = formatter.format(List.of(exp));
        assertFalse(result.contains("Most similar on"));
    }

    @Test
    void format_cappedAtMaxExperiences() {
        var experiences = java.util.stream.IntStream.range(0, 10)
                .mapToObj(i -> new RetrievedExperience("p" + i, "s", "COMPLETED", null,
                        0.5 + i * 0.01, Map.of(), List.of(), Map.of()))
                .toList();
        String result = formatter.format(experiences);
        // Default max is 5 — count occurrences of "## Similar Case"
        long count = result.lines().filter(l -> l.startsWith("## Similar Case")).count();
        assertEquals(5, count);
    }
}
```

- [ ] **Step 2: Write CbrInputTransformerTest**

```java
package io.casehub.life.app.cbr;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerExecutionContext;
import io.casehub.api.spi.routing.RetrievedExperience;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class CbrInputTransformerTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @AfterEach
    void cleanup() {
        WorkerExecutionContext.clear();
    }

    @Test
    void apply_noWorkerContext_passThrough() {
        var transformer = new CbrInputTransformer(new LifeCbrExperienceFormatter());
        ObjectNode input = MAPPER.createObjectNode().put("key", "value");
        JsonNode result = transformer.apply(input);
        assertEquals("value", result.get("key").asText());
        assertFalse(result.has("_cbrContext"));
    }

    @Test
    void apply_emptyExperiences_passThrough() {
        var transformer = new CbrInputTransformer(new LifeCbrExperienceFormatter());
        WorkerExecutionContext.set(new WorkerContext("task", null, List.of(), List.of(), null, Map.of(), List.of()));
        ObjectNode input = MAPPER.createObjectNode().put("key", "value");
        JsonNode result = transformer.apply(input);
        assertFalse(result.has("_cbrContext"));
    }

    @Test
    void apply_withExperiences_mergesCbrContext() {
        var transformer = new CbrInputTransformer(new LifeCbrExperienceFormatter());
        var exp = new RetrievedExperience("problem", "solution", "COMPLETED", 0.9,
                0.85, Map.of(), List.of(), Map.of());
        WorkerExecutionContext.set(new WorkerContext("task", null, List.of(), List.of(), null, Map.of(), List.of(exp)));
        ObjectNode input = MAPPER.createObjectNode().put("key", "value");
        JsonNode result = transformer.apply(input);
        assertTrue(result.has("_cbrContext"));
        assertTrue(result.get("_cbrContext").asText().contains("problem"));
        assertEquals("value", result.get("key").asText());
    }

    @Test
    void apply_doesNotMutateOriginalInput() {
        var transformer = new CbrInputTransformer(new LifeCbrExperienceFormatter());
        var exp = new RetrievedExperience("problem", "solution", "COMPLETED", 0.9,
                0.85, Map.of(), List.of(), Map.of());
        WorkerExecutionContext.set(new WorkerContext("task", null, List.of(), List.of(), null, Map.of(), List.of(exp)));
        ObjectNode input = MAPPER.createObjectNode().put("key", "value");
        transformer.apply(input);
        assertFalse(input.has("_cbrContext"));
    }
}
```

- [ ] **Step 3: Run tests — expect FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest="LifeCbrExperienceFormatterTest,CbrInputTransformerTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 4: Implement LifeCbrExperienceFormatter**

Create `app/src/main/java/io/casehub/life/app/cbr/LifeCbrExperienceFormatter.java`:

```java
package io.casehub.life.app.cbr;

import io.casehub.api.spi.routing.ExperiencePlanStep;
import io.casehub.api.spi.routing.RetrievedExperience;
import jakarta.enterprise.context.ApplicationScoped;
import org.jspecify.annotations.Nullable;

import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@ApplicationScoped
public class LifeCbrExperienceFormatter {

    private static final int MAX_EXPERIENCES = 5;

    public @Nullable String format(List<RetrievedExperience> experiences) {
        if (experiences == null || experiences.isEmpty()) return null;

        List<RetrievedExperience> sorted = experiences.stream()
                .sorted(Comparator.comparingDouble(RetrievedExperience::similarityScore).reversed())
                .limit(MAX_EXPERIENCES)
                .toList();

        StringBuilder sb = new StringBuilder();
        for (var exp : sorted) {
            sb.append("## Similar Case (similarity: ")
              .append(String.format("%.2f", exp.similarityScore())).append(")\n");
            sb.append("Problem: ").append(exp.problem()).append('\n');
            sb.append("Solution: ").append(exp.solution()).append('\n');
            sb.append("Outcome: ").append(exp.outcome()).append('\n');

            if (!exp.features().isEmpty()) {
                String featureStr = exp.features().entrySet().stream()
                        .map(e -> e.getKey() + "=" + e.getValue())
                        .collect(Collectors.joining(", "));
                sb.append("Key features: ").append(featureStr).append('\n');
            }

            if (!exp.featureSimilarities().isEmpty()) {
                String simStr = exp.featureSimilarities().entrySet().stream()
                        .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
                        .map(e -> e.getKey() + " (" + String.format("%.2f", e.getValue()) + ")")
                        .collect(Collectors.joining(", "));
                sb.append("Most similar on: ").append(simStr).append('\n');
            }

            if (!exp.planTrace().isEmpty()) {
                sb.append("Plan trace:\n");
                for (ExperiencePlanStep step : exp.planTrace()) {
                    sb.append("  - ").append(step.capabilityName())
                      .append(": ").append(step.stepOutcome()).append('\n');
                }
            }

            sb.append('\n');
        }
        return sb.toString().stripTrailing();
    }
}
```

- [ ] **Step 5: Implement CbrInputTransformer**

Create `app/src/main/java/io/casehub/life/app/cbr/CbrInputTransformer.java`:

```java
package io.casehub.life.app.cbr;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerExecutionContext;

import java.util.function.UnaryOperator;

public class CbrInputTransformer implements UnaryOperator<JsonNode> {

    private final LifeCbrExperienceFormatter formatter;

    public CbrInputTransformer(LifeCbrExperienceFormatter formatter) {
        this.formatter = formatter;
    }

    @Override
    public JsonNode apply(JsonNode input) {
        WorkerContext ctx = WorkerExecutionContext.current();
        if (ctx == null || ctx.experiences().isEmpty()) {
            return input;
        }
        String formatted = formatter.format(ctx.experiences());
        if (formatted == null) {
            return input;
        }
        ObjectNode enriched = input.deepCopy();
        enriched.put("_cbrContext", formatted);
        return enriched;
    }
}
```

- [ ] **Step 6: Run tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest="LifeCbrExperienceFormatterTest,CbrInputTransformerTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 7: Update LifeTypedCaseHub — inject formatter, wire transformer and CBR suffix into agentWorker()**

Use `ide_edit_member` / `ide_insert_member` to:

1. Add field: `@Inject LifeCbrExperienceFormatter cbrFormatter;`
2. Add field: `private CbrInputTransformer cbrInputTransformer;`
3. Add constant:
```java
static final String CBR_SYSTEM_PROMPT_SUFFIX = """
        If a _cbrContext section is present in the input, it contains summaries of \
        similar past cases. Use these to calibrate your response — adjust cost \
        estimates, timeline predictions, and risk assessments based on historical \
        patterns. If no _cbrContext is present, proceed with your best judgment.""";
```
4. Add `@PostConstruct` method to create the transformer:
```java
@PostConstruct
void initCbrTransformer() {
    this.cbrInputTransformer = new CbrInputTransformer(cbrFormatter);
}
```
5. Update `agentWorker()` body:
```java
protected Worker agentWorker(String capabilityName, String systemPrompt,
                              Class<?> responseSchema) {
    Agent a = Agent.builder()
            .model(openClawFactory.forAgent(agent))
            .systemPrompt(systemPrompt + "\n\n" + CBR_SYSTEM_PROMPT_SUFFIX)
            .inputTransformer(cbrInputTransformer)
            .responseSchema(responseSchema)
            .build();
    return Worker.builder()
            .name(capabilityName + "-agent")
            .capabilityName(capabilityName)
            .function(new AgentWorkerFunction(a))
            .build();
}
```

- [ ] **Step 8: Update LifeCaseService — inject LifeCbrSuggestionService, call suggest() between Phase 1 and Phase 2**

Use `ide_edit_member` to:

1. Add field: `@Inject LifeCbrSuggestionService cbrSuggestionService;`
2. Add field: `@Inject ObjectMapper objectMapper;`
3. Update `startCase()` body — insert CBR enrichment between Phase 1 and Phase 2:

```java
public LifeCaseResponse startCase(CreateLifeCaseRequest request) {
    UUID trackerId = UUID.randomUUID();
    try {
        Map<String, Object> initialContext = prepareAndTrack(trackerId, request);

        CbrSuggestions suggestions = cbrSuggestionService.suggest(
                request.caseType(), initialContext);
        if (!suggestions.isEmpty()) {
            initialContext.put("cbrCalibration",
                    objectMapper.convertValue(suggestions, Map.class));
        }

        CaseHub caseHub = resolve(request.caseType());
        UUID caseId = caseHub.startCase(initialContext).toCompletableFuture().join();

        persistCaseId(trackerId, caseId);
        caseHubRuntime.signal(caseId, "caseId", caseId.toString());

        return new LifeCaseResponse(caseId, request.caseType(), LifeCaseStatus.ACTIVE);
    } catch (Exception e) {
        LOG.errorf(e, "Case start failed for type=%s tracker=%s", request.caseType(), trackerId);
        try {
            markFailed(trackerId);
        } catch (Exception mfe) {
            LOG.errorf(mfe, "markFailed also failed for tracker=%s", trackerId);
        }
        throw new RuntimeException("Case start failed: " + e.getMessage(), e);
    }
}
```

- [ ] **Step 9: Run existing tests to verify no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest="LifeTypedCaseHubTest,TravelPlanCaseHubTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/LifeCbrExperienceFormatter.java app/src/main/java/io/casehub/life/app/cbr/CbrInputTransformer.java app/src/main/java/io/casehub/life/app/engine/LifeTypedCaseHub.java app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java app/src/test/java/io/casehub/life/app/cbr/LifeCbrExperienceFormatterTest.java app/src/test/java/io/casehub/life/app/cbr/CbrInputTransformerTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#56): prompt enrichment + case-start calibration — both CBR paths wired

LifeCbrExperienceFormatter formats RetrievedExperience into prompt text.
CbrInputTransformer reads WorkerExecutionContext.current().experiences()
at worker execution time and merges _cbrContext into the user message.
LifeTypedCaseHub.agentWorker() registers the transformer + CBR suffix
on every Agent. LifeCaseService.startCase() calls suggest() between
Phase 1 and Phase 2, writes cbrCalibration to case context.

Refs #56"
```

---

### Task 5: System Prompt Updates and YAML inputProjection Changes

Updates system prompts with calibration-specific instructions and adds `cbrCalibration` to the inputProjection of workers that benefit from structured calibration.

**Files:**
- Modify: 5 CaseHub subclasses — system prompt updates for 8 workers
- Modify: 6 YAML case definitions — inputProjection updates for 8 capabilities
- Modify: `app/src/test/java/io/casehub/life/app/engine/TestLifeOpenClawChatModelFactory.java` — update system prompt matching patterns

**Interfaces:**
- Consumes: `CBR_SYSTEM_PROMPT_SUFFIX` from Task 4 (appended automatically by `agentWorker()`)
- Produces: Updated system prompts and inputProjections — no new public API

- [ ] **Step 1: Update TravelPlanCaseHub system prompts**

Use `ide_edit_member` on `configureCase()`:
- `budget-assessment`: append calibration instruction about `featureStats.budget` and `historicalSuccessRate`
- `booking`: append calibration instruction about historical success rate for booking decisions

- [ ] **Step 2: Update HomeMaintenanceCaseHub system prompts**

- `schedule-inspection`: append calibration instruction about historical maintenance duration
- `get-quotes`: append calibration instruction about `featureStats.estimatedCost` for quote comparison

- [ ] **Step 3: Update ContractorCoordinationCaseHub system prompts**

- `request-quote`: append calibration instruction about `featureStats.estimatedCost` for historical cost ranges

- [ ] **Step 4: Update AppointmentCycleCaseHub system prompts**

- `book-appointment`: Note — this worker uses manual `Agent.builder()` with `userMessage` template. Add `inputTransformer(cbrInputTransformer)` and `CBR_SYSTEM_PROMPT_SUFFIX` manually. Also add calibration instruction about historical appointment patterns.

This requires making `cbrInputTransformer` accessible. `LifeTypedCaseHub` is the superclass — the field is already available via `protected` or package access. Add a protected getter if needed.

- [ ] **Step 5: Update FinancialReviewCaseHub system prompts**

- `analyse-anomalies`: append calibration instruction about `featureStats.estimatedBudget` for threshold calibration

- [ ] **Step 6: Update CareCoordinationCaseHub system prompts**

- `care-plan`: append calibration instruction about historical care duration and frequency

- [ ] **Step 7: Update YAML inputProjections — add cbrCalibration to 8 capabilities**

For each YAML file, use the Edit tool (YAML, not Java) to update the `inputSchema` line:

| File | Capability | Before | After |
|------|-----------|--------|-------|
| `travel-plan.yaml` | budget-assessment | `"{ flightResults: .flightResults, hotelResults: .hotelResults }"` | `"{ flightResults: .flightResults, hotelResults: .hotelResults, cbrCalibration: .cbrCalibration }"` |
| `travel-plan.yaml` | booking | `"{ selectedDestination: .selectedDestination, flightResults: .flightResults, hotelResults: .hotelResults }"` | `"{ selectedDestination: .selectedDestination, flightResults: .flightResults, hotelResults: .hotelResults, cbrCalibration: .cbrCalibration }"` |
| `home-maintenance.yaml` | schedule-inspection | `"{ request: .request }"` | `"{ request: .request, cbrCalibration: .cbrCalibration }"` |
| `home-maintenance.yaml` | get-quotes | `"{ inspection: .inspection }"` | `"{ inspection: .inspection, cbrCalibration: .cbrCalibration }"` |
| `contractor-coordination.yaml` | request-quote | `"{ contractorRequest: .contractorRequest }"` | `"{ contractorRequest: .contractorRequest, cbrCalibration: .cbrCalibration }"` |
| `appointment-cycle.yaml` | book-appointment | `"{ appointmentType: .appointmentType, provider: .provider }"` | `"{ appointmentType: .appointmentType, provider: .provider, cbrCalibration: .cbrCalibration }"` |
| `financial-review.yaml` | analyse-anomalies | `"{ budgetData: .budgetData }"` | `"{ budgetData: .budgetData, cbrCalibration: .cbrCalibration }"` |
| `care-coordination.yaml` | care-plan | `"{ assessment: .assessment }"` | `"{ assessment: .assessment, cbrCalibration: .cbrCalibration }"` |

- [ ] **Step 8: Update TestLifeOpenClawChatModelFactory**

The test factory matches on system prompt key phrases. Updated system prompts have calibration instructions appended. Verify the factory still matches — the matching uses `contains()` on original key phrases, so appending text should not break matching. If any test fails due to prompt changes, update the matching patterns.

- [ ] **Step 9: Run all CaseHub tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest="TravelPlanCaseHubTest,HomeMaintenanceCaseHubTest,ContractorCoordinationCaseHubTest,AppointmentCycleCaseHubTest,FinancialReviewCaseHubTest,CareCoordinationCaseHubTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/engine/ app/src/main/resources/life/ app/src/test/
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#56): system prompt calibration instructions + YAML inputProjection updates

8 workers gain domain-specific CBR calibration instructions in their
system prompts. 8 YAML capabilities gain cbrCalibration in their
inputProjection for structured parameter access.

Refs #56"
```

---

### Task 6: Full Build Verification and CLAUDE.md Update

Final verification, documentation update, and full build.

**Files:**
- Modify: `CLAUDE.md` — update Layer 8 status

- [ ] **Step 1: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install --batch-mode
```

- [ ] **Step 2: Update CLAUDE.md Layer 8 entry**

Update the Layer 8 entry in the Foundation Layers section to reflect that engine#505, #683, and #707 are all closed and the integration is complete:

```
Layer 8: + casehub-neocortex (CBR) — Case-Based Reasoning for adaptive life automation.
         ...
         Engine dependencies: ~~engine#505~~ CLOSED, ~~engine#683~~ CLOSED, ~~engine#707~~ CLOSED.
         CBR engine integration (#56): prompt enrichment via Agent.inputTransformer
         (LifeCbrExperienceFormatter + CbrInputTransformer) + structured calibration via
         LifeCbrSuggestionService at case start. CbrSuggestions/FeatureStatistics in api/.
         LifeCbrFeatureExtractor consolidates feature extraction across retention + suggestion.
         ✅ COMPLETE (retention + retrieval + integration)  🔲 PENDING (routing effect — engine#505 CLOSED, routing active)
```

Also update the Foundation Gates table and the What This Project Owns section.

- [ ] **Step 3: Commit CLAUDE.md**

```bash
git -C /Users/mdproctor/claude/casehub/life add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/life commit -m "docs(#56): update CLAUDE.md — Layer 8 CBR engine integration complete

Refs #56"
```

- [ ] **Step 4: invoke work-end**
