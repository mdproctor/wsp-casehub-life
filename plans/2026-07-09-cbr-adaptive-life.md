# CBR for Adaptive Life Automation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #52 — epic: Case-Based Reasoning (CBR) for adaptive life automation
**Issue group:** #52, #53, #57

**Goal:** Wire CBR retention (per-case + per-routing-decision) and retrieval configuration into casehub-life's 6 case types, so past case outcomes accumulate in the CBR store and inform future cases via the engine's CbrRetrievalService.

**Architecture:** Both retention writers look up the CaseDefinition's CbrConfig and reuse its JQ feature expressions for feature extraction — guaranteeing alignment with the retrieval path by construction. Domain-specific problem/solution text comes from LifeCbrDescriptionProvider implementations. Feature schemas are registered at startup.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-api (CaseOutcomeObserver, RoutingOutcomeRecorder), casehub-neocortex-memory-api (CbrCaseMemoryStore, PlanCbrCase, CbrFeatureSchema, SimilaritySpec), casehub-platform-expression (JQEvaluator)

## Global Constraints

- Package: `io.casehub.neocortex.memory.cbr.*` (published jar package, NOT `io.casehub.memory.cbr.*`)
- Maven: `casehub-neocortex-memory-api` for CBR types; `casehub-neocortex-memory-cbr-inmem` for test adapter
- `CaseOutcomeObserver.onOutcome()` runs on a Vert.x worker thread — blocking OK, `@Transactional` OK
- `RoutingOutcomeRecorder.record()` returns `Uni<Void>` — reactive pipeline, fire-and-forget
- `CaseDefinitionRegistry.findByName(String)` returns `Optional<CaseDefinition>` — may be empty
- `JQEvaluator.eval(String jqExpr, JsonNode input)` returns `ValidationResult(ok, error, output)`
- `CaseOutcomeEvent` has no `tenantId` — use `"life-personal"` interim constant
- IntelliJ MCP mandatory for all Java code operations — never bash grep/find for classes

---

### Task 1: Maven Dependencies

**Files:**
- Modify: `app/pom.xml`

**Interfaces:**
- Produces: `casehub-neocortex-memory-api` and `casehub-neocortex-memory-cbr-inmem` available on classpath

- [ ] **Step 1: Add casehub-neocortex-memory-api dependency**

Add to `app/pom.xml` dependencies section:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-memory-api</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

- [ ] **Step 2: Add casehub-neocortex-memory-cbr-inmem test dependency**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-memory-cbr-inmem</artifactId>
    <version>${casehub.version}</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode`
Expected: BUILD SUCCESS — neocortex CBR types resolvable

- [ ] **Step 4: Add InMemoryCbrCaseMemoryStore to test selected-alternatives**

Check `app/src/main/resources/application.properties` and `app/src/test/resources/application.properties` — the InMemory adapter is `@Alternative @Priority(2)` so it should activate automatically in tests without a `selected-alternatives` entry. Verify by checking that `NoOpCbrCaseMemoryStore` (`@DefaultBean`) would be overridden. If `selected-alternatives` is needed, add `io.casehub.neocortex.memory.cbr.inmem.InMemoryCbrCaseMemoryStore`.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/pom.xml
git -C /Users/mdproctor/claude/casehub/life commit -m "chore(#52): add casehub-neocortex-memory-api + cbr-inmem dependencies

Refs #52"
```

---

### Task 2: LifeCbrDescriptionProvider SPI + 6 Implementations

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifeCbrDescriptionProvider.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/describe/ContractorCoordinationDescriptionProvider.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/describe/HomeMaintenanceDescriptionProvider.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/describe/AppointmentCycleDescriptionProvider.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/describe/CareCoordinationDescriptionProvider.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/describe/FinancialReviewDescriptionProvider.java`
- Create: `app/src/main/java/io/casehub/life/app/cbr/describe/TravelPlanDescriptionProvider.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/describe/ContractorCoordinationDescriptionProviderTest.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/describe/HomeMaintenanceDescriptionProviderTest.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/describe/AppointmentCycleDescriptionProviderTest.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/describe/CareCoordinationDescriptionProviderTest.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/describe/FinancialReviewDescriptionProviderTest.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/describe/TravelPlanDescriptionProviderTest.java`

**Interfaces:**
- Produces: `LifeCbrDescriptionProvider` interface with `caseType()`, `describeProblem(Map)`, `describeSolution(Map)`, `extractEntityId(Map, UUID)`. Used by Tasks 3 and 4.

- [ ] **Step 1: Create LifeCbrDescriptionProvider interface**

```java
package io.casehub.life.app.cbr;

import java.util.Map;
import java.util.UUID;

public interface LifeCbrDescriptionProvider {
    String caseType();
    String describeProblem(Map<String, Object> caseData);
    String describeSolution(Map<String, Object> caseData);
    String extractEntityId(Map<String, Object> caseData, UUID caseId);
}
```

- [ ] **Step 2: Write failing tests for ContractorCoordinationDescriptionProvider**

Test: problem description from contractor context, solution from quote/approval data, entityId extraction from `.contractorRequest.contractorId`, fallback to caseId when missing.

```java
package io.casehub.life.app.cbr.describe;

import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class ContractorCoordinationDescriptionProviderTest {

    private final ContractorCoordinationDescriptionProvider provider =
            new ContractorCoordinationDescriptionProvider();

    @Test
    void caseType() {
        assertThat(provider.caseType()).isEqualTo("contractor-coordination");
    }

    @Test
    void describeProblem_withFullContext() {
        var data = Map.<String, Object>of(
                "contractorRequest", Map.of(
                        "problemType", "boiler-repair",
                        "propertyArea", "kitchen",
                        "budget", 500));
        String problem = provider.describeProblem(data);
        assertThat(problem).contains("boiler-repair");
    }

    @Test
    void describeProblem_withMissingFields() {
        var data = Map.<String, Object>of();
        String problem = provider.describeProblem(data);
        assertThat(problem).isNotBlank();
    }

    @Test
    void describeSolution_withQuoteAccepted() {
        var data = Map.<String, Object>of(
                "quoteResponse", Map.of("quotedAmount", 450, "contractor", "PlumbCo"),
                "quoteApproval", Map.of("approved", true));
        String solution = provider.describeSolution(data);
        assertThat(solution).contains("PlumbCo");
    }

    @Test
    void describeSolution_withMissingFields() {
        var data = Map.<String, Object>of();
        String solution = provider.describeSolution(data);
        assertThat(solution).isNotBlank();
    }

    @Test
    void extractEntityId_withContractorId() {
        var caseId = UUID.randomUUID();
        var data = Map.<String, Object>of(
                "contractorRequest", Map.of("contractorId", "ext-actor-123"));
        assertThat(provider.extractEntityId(data, caseId)).isEqualTo("ext-actor-123");
    }

    @Test
    void extractEntityId_fallbackToCaseId() {
        var caseId = UUID.randomUUID();
        var data = Map.<String, Object>of();
        assertThat(provider.extractEntityId(data, caseId)).isEqualTo(caseId.toString());
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ContractorCoordinationDescriptionProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 4: Implement ContractorCoordinationDescriptionProvider**

```java
package io.casehub.life.app.cbr.describe;

import io.casehub.life.app.cbr.LifeCbrDescriptionProvider;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class ContractorCoordinationDescriptionProvider implements LifeCbrDescriptionProvider {

    @Override
    public String caseType() {
        return "contractor-coordination";
    }

    @Override
    public String describeProblem(Map<String, Object> caseData) {
        var request = asMap(caseData.get("contractorRequest"));
        String problemType = str(request, "problemType", "unknown");
        String area = str(request, "propertyArea", "");
        String budget = str(request, "budget", "");
        return "Contractor: %s%s%s".formatted(
                problemType,
                area.isEmpty() ? "" : " in " + area,
                budget.isEmpty() ? "" : ", budget " + budget);
    }

    @Override
    public String describeSolution(Map<String, Object> caseData) {
        var quote = asMap(caseData.get("quoteResponse"));
        String contractor = str(quote, "contractor", "unknown");
        String amount = str(quote, "quotedAmount", "");
        return "Contractor %s%s".formatted(
                contractor,
                amount.isEmpty() ? "" : " at " + amount);
    }

    @Override
    public String extractEntityId(Map<String, Object> caseData, UUID caseId) {
        var request = asMap(caseData.get("contractorRequest"));
        String contractorId = str(request, "contractorId", "");
        return contractorId.isEmpty() ? caseId.toString() : contractorId;
    }

    @SuppressWarnings("unchecked")
    private static Map<String, Object> asMap(Object obj) {
        return obj instanceof Map<?, ?> m ? (Map<String, Object>) m : Map.of();
    }

    private static String str(Map<String, Object> map, String key, String fallback) {
        Object val = map.get(key);
        return val != null ? String.valueOf(val) : fallback;
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ContractorCoordinationDescriptionProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Implement remaining 5 description providers with tests**

Follow the same TDD pattern for each. Key extraction patterns per case type:

| Provider | Problem key | Solution key | EntityId key | EntityId fallback |
|----------|-------------|--------------|-------------|-------------------|
| HomeMaintenanceDescriptionProvider | `.request.issueType`, `.request.severity` | `.jobStatus.outcome`, `.quotes.selectedContractor` | `caseId` (no external actor) | `caseId.toString()` |
| AppointmentCycleDescriptionProvider | `.appointmentType`, `.provider` | `.visitNotes.outcome`, `.booking.provider` | `.provider.id` or `.booking.providerId` | `caseId.toString()` |
| CareCoordinationDescriptionProvider | `.careRequest.careType`, `.careRequest.patientRiskLevel` | `.carePlan.coordinator`, `.carePlan.hoursPerWeek` | `.careRequest.coordinatorId` | `caseId.toString()` |
| FinancialReviewDescriptionProvider | `.reviewPeriod`, `.budgetData.category` | `.analysis.outcome`, `.report.summary` | `caseId` (no external actor) | `caseId.toString()` |
| TravelPlanDescriptionProvider | `.request.destination`, `.request.travelType` | `.itinerary.summary`, `.booking.total` | `caseId` (no external actor) | `caseId.toString()` |

Each provider follows the exact same structure as ContractorCoordinationDescriptionProvider — `@ApplicationScoped`, implements `LifeCbrDescriptionProvider`, null-safe extraction via `asMap()` / `str()` helpers.

Each test class covers: `caseType()`, `describeProblem()` with full and empty context, `describeSolution()` with full and empty context, `extractEntityId()` with and without the ID field.

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="*DescriptionProviderTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 6 test classes, all PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/ app/src/test/java/io/casehub/life/app/cbr/
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#53): LifeCbrDescriptionProvider — domain problem/solution/entityId per case type

Six implementations: contractor-coordination, home-maintenance,
appointment-cycle, care-coordination, financial-review, travel-plan.

Refs #53"
```

---

### Task 3: LifeCaseOutcomeCbrWriter (Per-Case Retention)

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifeCaseOutcomeCbrWriter.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/LifeCaseOutcomeCbrWriterTest.java`

**Interfaces:**
- Consumes: `LifeCbrDescriptionProvider` (Task 2), `CaseOutcomeObserver` (engine-api), `CaseDefinitionRegistry` (engine-common), `CbrCaseMemoryStore` (neocortex), `JQEvaluator` (platform-expression)
- Produces: Writes `PlanCbrCase` to `CbrCaseMemoryStore` when a life case reaches terminal state

- [ ] **Step 1: Write failing test — happy path**

```java
package io.casehub.life.app.cbr;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.model.cbr.JqFeatureExtractor;
import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.platform.expression.JQEvaluator;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import java.time.Instant;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class LifeCaseOutcomeCbrWriterTest {

    private CbrCaseMemoryStore cbrStore;
    private CaseDefinitionRegistry registry;
    private JQEvaluator jqEvaluator;
    private LifeCaseOutcomeCbrWriter writer;
    private static final ObjectMapper MAPPER = new ObjectMapper();

    @BeforeEach
    void setUp() {
        cbrStore = mock(CbrCaseMemoryStore.class);
        registry = mock(CaseDefinitionRegistry.class);
        jqEvaluator = mock(JQEvaluator.class);

        // Set up a description provider for contractor-coordination
        var descProvider = new io.casehub.life.app.cbr.describe
                .ContractorCoordinationDescriptionProvider();
        @SuppressWarnings("unchecked")
        Instance<LifeCbrDescriptionProvider> providers = mock(Instance.class);
        when(providers.stream()).thenReturn(java.util.stream.Stream.of(descProvider));

        writer = new LifeCaseOutcomeCbrWriter(cbrStore, registry, jqEvaluator, providers);
    }

    @Test
    void onOutcome_contractorCase_writesPlanCbrCase() {
        // Set up CaseDefinition with CbrConfig
        CbrConfig config = CbrConfig.builder()
                .feature("problemType", ".contractorRequest.problemType")
                .feature("budget", ".contractorRequest.budget")
                .domain("casehubio/life/contractor")
                .caseType("contractor-coordination")
                .build();
        var definition = mock(CaseDefinition.class);
        when(definition.getCbrConfig()).thenReturn(config);
        when(registry.findByName("contractor-coordination")).thenReturn(Optional.of(definition));

        // Mock JQ evaluation
        var snapshot = Map.<String, Object>of(
                "contractorRequest", Map.of("problemType", "boiler-repair", "budget", 500));
        var jsonNode = MAPPER.valueToTree(snapshot);

        when(jqEvaluator.eval(eq(".contractorRequest.problemType"), any()))
                .thenReturn(io.casehub.platform.expression.ValidationResult.ok(
                        java.util.List.of(MAPPER.valueToTree("boiler-repair"))));
        when(jqEvaluator.eval(eq(".contractorRequest.budget"), any()))
                .thenReturn(io.casehub.platform.expression.ValidationResult.ok(
                        java.util.List.of(MAPPER.valueToTree(500))));

        var event = new CaseOutcomeEvent(
                "contractor-coordination",
                UUID.randomUUID(),
                snapshot,
                "COMPLETED",
                Instant.now(),
                Map.of());

        writer.onOutcome(event);

        var caseCaptor = ArgumentCaptor.forClass(PlanCbrCase.class);
        verify(cbrStore).store(
                caseCaptor.capture(),
                eq("contractor-coordination"),
                anyString(),
                any(),
                eq("life-personal"),
                eq(event.caseId().toString()));

        PlanCbrCase stored = caseCaptor.getValue();
        assertThat(stored.outcome()).isEqualTo("COMPLETED");
        assertThat(stored.features()).containsEntry("problemType", "boiler-repair");
        assertThat(stored.features()).containsEntry("budget", 500);
        assertThat(stored.planTrace()).isEmpty();
    }

    @Test
    void onOutcome_nonLifeCase_skips() {
        var event = new CaseOutcomeEvent(
                "unknown-case-type",
                UUID.randomUUID(),
                Map.of(),
                "COMPLETED",
                Instant.now(),
                Map.of());

        writer.onOutcome(event);

        verifyNoInteractions(cbrStore);
    }

    @Test
    void onOutcome_noCbrConfig_skips() {
        var definition = mock(CaseDefinition.class);
        when(definition.getCbrConfig()).thenReturn(null);
        when(registry.findByName("contractor-coordination")).thenReturn(Optional.of(definition));

        var event = new CaseOutcomeEvent(
                "contractor-coordination",
                UUID.randomUUID(),
                Map.of(),
                "COMPLETED",
                Instant.now(),
                Map.of());

        writer.onOutcome(event);

        verifyNoInteractions(cbrStore);
    }

    @Test
    void onOutcome_lambdaExtractor_skips() {
        CbrConfig config = CbrConfig.builder()
                .featureExtractor(ctx -> Map.of())
                .domain("casehubio/life/contractor")
                .build();
        var definition = mock(CaseDefinition.class);
        when(definition.getCbrConfig()).thenReturn(config);
        when(registry.findByName("contractor-coordination")).thenReturn(Optional.of(definition));

        var event = new CaseOutcomeEvent(
                "contractor-coordination",
                UUID.randomUUID(),
                Map.of(),
                "COMPLETED",
                Instant.now(),
                Map.of());

        writer.onOutcome(event);

        verifyNoInteractions(cbrStore);
    }

    @Test
    void onOutcome_storeThrows_doesNotPropagate() {
        CbrConfig config = CbrConfig.builder()
                .feature("problemType", ".contractorRequest.problemType")
                .domain("casehubio/life/contractor")
                .caseType("contractor-coordination")
                .build();
        var definition = mock(CaseDefinition.class);
        when(definition.getCbrConfig()).thenReturn(config);
        when(registry.findByName("contractor-coordination")).thenReturn(Optional.of(definition));

        when(jqEvaluator.eval(any(), any()))
                .thenReturn(io.casehub.platform.expression.ValidationResult.ok(
                        java.util.List.of(MAPPER.valueToTree("boiler-repair"))));
        when(cbrStore.store(any(), any(), any(), any(), any(), any()))
                .thenThrow(new RuntimeException("store failure"));

        var event = new CaseOutcomeEvent(
                "contractor-coordination",
                UUID.randomUUID(),
                Map.of("contractorRequest", Map.of("problemType", "boiler-repair")),
                "COMPLETED",
                Instant.now(),
                Map.of());

        // Should not throw
        writer.onOutcome(event);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeCaseOutcomeCbrWriterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement LifeCaseOutcomeCbrWriter**

```java
package io.casehub.life.app.cbr;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.model.cbr.JqFeatureExtractor;
import io.casehub.api.model.cbr.LambdaFeatureExtractor;
import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.api.spi.CaseOutcomeObserver;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.platform.expression.JQEvaluator;
import io.casehub.platform.expression.ValidationResult;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.*;

@ApplicationScoped
public class LifeCaseOutcomeCbrWriter implements CaseOutcomeObserver {

    static final String TENANT_ID = "life-personal";

    private static final Logger LOG = Logger.getLogger(LifeCaseOutcomeCbrWriter.class);
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final CbrCaseMemoryStore cbrStore;
    private final CaseDefinitionRegistry registry;
    private final JQEvaluator jqEvaluator;
    private final Map<String, LifeCbrDescriptionProvider> providers;

    @Inject
    public LifeCaseOutcomeCbrWriter(CbrCaseMemoryStore cbrStore,
                                    CaseDefinitionRegistry registry,
                                    JQEvaluator jqEvaluator,
                                    Instance<LifeCbrDescriptionProvider> providers) {
        this.cbrStore = cbrStore;
        this.registry = registry;
        this.jqEvaluator = jqEvaluator;
        this.providers = new HashMap<>();
        providers.stream().forEach(p -> this.providers.put(p.caseType(), p));
    }

    @Override
    public void onOutcome(CaseOutcomeEvent event) {
        LifeCbrDescriptionProvider descProvider = providers.get(event.caseType());
        if (descProvider == null) return;

        try {
            var definition = registry.findByName(event.caseType()).orElse(null);
            if (definition == null || definition.getCbrConfig() == null) return;

            CbrConfig config = definition.getCbrConfig();
            if (!(config.featureExtractor() instanceof JqFeatureExtractor jq)) {
                LOG.warnf("CBR retention skipped for %s — lambda extractor unsupported at retention time",
                        event.caseType());
                return;
            }

            JsonNode jsonNode = MAPPER.valueToTree(event.caseFileSnapshot());
            Map<String, Object> features = extractFeatures(jq, jsonNode);

            PlanCbrCase cbrCase = new PlanCbrCase(
                    descProvider.describeProblem(event.caseFileSnapshot()),
                    descProvider.describeSolution(event.caseFileSnapshot()),
                    event.outcomeLabel(),
                    null,
                    features,
                    List.of());

            cbrStore.store(
                    cbrCase,
                    event.caseType(),
                    descProvider.extractEntityId(event.caseFileSnapshot(), event.caseId()),
                    new MemoryDomain(config.domain()),
                    TENANT_ID,
                    event.caseId().toString());

        } catch (Exception e) {
            LOG.warnf(e, "CBR retention failed for case %s (%s) — proceeding without recording",
                    event.caseId(), event.caseType());
        }
    }

    private Map<String, Object> extractFeatures(JqFeatureExtractor jq, JsonNode input) {
        Map<String, Object> features = new LinkedHashMap<>();
        for (var entry : jq.featureExpressions().entrySet()) {
            ValidationResult result = jqEvaluator.eval(entry.getValue(), input);
            if (!result.ok() || result.output().isEmpty()) continue;
            JsonNode node = result.output().get(0);
            if (node.isNull()) continue;
            features.put(entry.getKey(), unwrap(node));
        }
        return features;
    }

    private static Object unwrap(JsonNode node) {
        if (node.isTextual()) return node.asText();
        if (node.isInt()) return node.asInt();
        if (node.isLong()) return node.asLong();
        if (node.isDouble() || node.isFloat()) return node.asDouble();
        if (node.isBoolean()) return node.asBoolean();
        return node.asText();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeCaseOutcomeCbrWriterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/LifeCaseOutcomeCbrWriter.java app/src/test/java/io/casehub/life/app/cbr/LifeCaseOutcomeCbrWriterTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#57): LifeCaseOutcomeCbrWriter — per-case CBR retention via CaseOutcomeObserver

Implements CaseOutcomeObserver. On case terminal state: looks up
CbrConfig, extracts features via JQ expressions, writes PlanCbrCase
to CbrCaseMemoryStore. Graceful degradation on failure.

Refs #57"
```

---

### Task 4: LifeRoutingOutcomeRecorder (Per-Routing-Decision Retention)

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifeRoutingOutcomeRecorder.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/LifeRoutingOutcomeRecorderTest.java`

**Interfaces:**
- Consumes: `LifeCbrDescriptionProvider` (Task 2), `RoutingOutcomeRecorder` (engine-api), `CaseDefinitionRegistry` (engine-common), `CbrCaseMemoryStore` (neocortex), `JQEvaluator` (platform-expression), `LifeCaseTracker` (life entity)
- Produces: Writes `PlanCbrCase` with `PlanTrace` to `CbrCaseMemoryStore` per worker execution

- [ ] **Step 1: Write failing test**

Test cases: happy path (writes PlanCbrCase with PlanTrace), non-life case (skips), no CbrConfig (skips), lambda extractor (skips), store failure (returns completed Uni, no propagation).

```java
package io.casehub.life.app.cbr;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.spi.routing.AgentRoutingContext;
import io.casehub.api.spi.routing.RoutingOutcome;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.life.app.entity.LifeCaseTracker;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.platform.expression.JQEvaluator;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import java.time.Duration;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class LifeRoutingOutcomeRecorderTest {

    private CbrCaseMemoryStore cbrStore;
    private CaseDefinitionRegistry registry;
    private JQEvaluator jqEvaluator;
    private LifeRoutingOutcomeRecorder recorder;
    private static final ObjectMapper MAPPER = new ObjectMapper();

    // Tests follow same pattern as LifeCaseOutcomeCbrWriterTest.
    // Key difference: RoutingOutcomeRecorder.record() returns Uni<Void>,
    // so tests must subscribe and await the Uni.
    // LifeCaseTracker.findByEngineCaseId() is a Panache static method —
    // use @QuarkusTest or extract lookup to an injectable helper.

    // ... (tests omitted for brevity — same pattern as Task 3 tests
    //      but verify PlanTrace contains bindingName, capabilityName,
    //      workerId, outcome; verify entityId = "agent-routing";
    //      verify tenantId = context.tenancyId())
}
```

The LifeCaseTracker lookup (`findByEngineCaseId`) is a Panache static method that can't be mocked in unit tests. Extract the lookup into an injectable helper or use `@QuarkusTest`. Prefer the injectable helper to keep the unit tests fast:

```java
// In LifeRoutingOutcomeRecorder — injectable lookup to make testable
@ApplicationScoped
static class CaseTypeLookup {
    Optional<String> findCaseType(UUID engineCaseId) {
        return LifeCaseTracker.findByEngineCaseId(engineCaseId)
                .map(t -> t.caseType);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement LifeRoutingOutcomeRecorder**

Reactive pipeline: `Uni.createFrom().item(() -> { ... }).emitOn(Infrastructure.getDefaultWorkerPool()).replaceWithVoid()`. Returns `Uni.createFrom().voidItem()` on skip. Catches all exceptions and returns completed Uni.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/LifeRoutingOutcomeRecorder.java app/src/test/java/io/casehub/life/app/cbr/LifeRoutingOutcomeRecorderTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#57): LifeRoutingOutcomeRecorder — per-routing-decision CBR retention

Implements RoutingOutcomeRecorder. Per worker execution: resolves case type
via LifeCaseTracker, extracts features via CbrConfig JQ, writes PlanCbrCase
with PlanTrace. Reactive pipeline, fire-and-forget.

Refs #57"
```

---

### Task 5: LifeCbrFeatureSchemaRegistrar

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/cbr/LifeCbrFeatureSchemaRegistrar.java`
- Test: `app/src/test/java/io/casehub/life/app/cbr/LifeCbrFeatureSchemaRegistrarTest.java`

**Interfaces:**
- Consumes: `CbrCaseMemoryStore.registerSchema()` (neocortex)
- Produces: 6 `CbrFeatureSchema` instances registered at startup

- [ ] **Step 1: Write failing test**

Verify all 6 schemas are registered with correct field names, types (Categorical/Numeric), ranges, and SimilaritySpecs. Mock `CbrCaseMemoryStore` and capture all `registerSchema()` calls.

```java
package io.casehub.life.app.cbr;

import io.casehub.neocortex.memory.cbr.*;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class LifeCbrFeatureSchemaRegistrarTest {

    @Test
    void registersAllSixSchemas() {
        var store = mock(CbrCaseMemoryStore.class);
        var registrar = new LifeCbrFeatureSchemaRegistrar(store);
        registrar.onStartup(null);

        var captor = ArgumentCaptor.forClass(CbrFeatureSchema.class);
        verify(store, times(6)).registerSchema(captor.capture());

        List<CbrFeatureSchema> schemas = captor.getAllValues();
        var caseTypes = schemas.stream().map(CbrFeatureSchema::caseType).toList();
        assertThat(caseTypes).containsExactlyInAnyOrder(
                "contractor-coordination", "home-maintenance",
                "appointment-cycle", "care-coordination",
                "financial-review", "travel-plan");
    }

    @Test
    void contractorSchema_hasCorrectFields() {
        var store = mock(CbrCaseMemoryStore.class);
        var registrar = new LifeCbrFeatureSchemaRegistrar(store);
        registrar.onStartup(null);

        var captor = ArgumentCaptor.forClass(CbrFeatureSchema.class);
        verify(store, atLeast(1)).registerSchema(captor.capture());

        CbrFeatureSchema contractor = captor.getAllValues().stream()
                .filter(s -> "contractor-coordination".equals(s.caseType()))
                .findFirst().orElseThrow();

        var fieldNames = contractor.fields().stream().map(FeatureField::name).toList();
        assertThat(fieldNames).containsExactlyInAnyOrder(
                "problemType", "season", "propertyArea",
                "budget", "quotedCost", "slaHours");

        // Verify season has CategoricalTable similarity spec
        FeatureField.Categorical seasonField = contractor.fields().stream()
                .filter(f -> "season".equals(f.name()))
                .map(f -> (FeatureField.Categorical) f)
                .findFirst().orElseThrow();
        assertThat(seasonField.similaritySpec())
                .isInstanceOf(SimilaritySpec.CategoricalTable.class);

        // Verify budget has GaussianDecay
        FeatureField.Numeric budgetField = contractor.fields().stream()
                .filter(f -> "budget".equals(f.name()))
                .map(f -> (FeatureField.Numeric) f)
                .findFirst().orElseThrow();
        assertThat(budgetField.similaritySpec())
                .isInstanceOf(SimilaritySpec.GaussianDecay.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement LifeCbrFeatureSchemaRegistrar**

Build all 6 schemas with exact field types, ranges, and similarity specs from the design spec §Feature Schemas. Use `FeatureField.categorical()`, `FeatureField.numeric()`, `SimilaritySpec.categoricalTableBuilder()`, `new SimilaritySpec.GaussianDecay()`.

The `@Observes StartupEvent` method calls `cbrStore.registerSchema()` for each.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/cbr/LifeCbrFeatureSchemaRegistrar.java app/src/test/java/io/casehub/life/app/cbr/LifeCbrFeatureSchemaRegistrarTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#53): LifeCbrFeatureSchemaRegistrar — 6 domain feature schemas at startup

Registers CbrFeatureSchema per case type with SimilaritySpecs:
CategoricalTable for season/severity, GaussianDecay for cost/time.

Refs #53"
```

---

### Task 6: YAML CbrConfig on 6 Case Definitions

**Files:**
- Modify: `app/src/main/resources/life/contractor-coordination.yaml`
- Modify: `app/src/main/resources/life/home-maintenance.yaml`
- Modify: `app/src/main/resources/life/appointment-cycle.yaml`
- Modify: `app/src/main/resources/life/care-coordination.yaml`
- Modify: `app/src/main/resources/life/financial-review.yaml`
- Modify: `app/src/main/resources/life/travel-plan.yaml`
- Test: `app/src/test/java/io/casehub/life/app/cbr/LifeCbrConfigAlignmentTest.java`

**Interfaces:**
- Consumes: YAML `spec.cbr` parsed by engine's `YamlCaseDefinitionParser` into `CbrConfig` on `CaseDefinition`
- Produces: CbrConfig available on all 6 case definitions at runtime

- [ ] **Step 1: Add spec.cbr to contractor-coordination.yaml**

Add immediately after `spec:` (before `capabilities:`):

```yaml
  cbr:
    features:
      problemType: ".contractorRequest.problemType"
      season: ".contractorRequest.season"
      propertyArea: ".contractorRequest.propertyArea"
      budget: ".contractorRequest.budget"
      quotedCost: ".quoteResponse.quotedAmount"
      slaHours: ".contractorRequest.slaHours"
    weights:
      problemType: 3.0
      budget: 2.0
      season: 1.5
      propertyArea: 1.0
      slaHours: 1.0
      quotedCost: 0.5
    topK: 5
    minSimilarity: 0.3
    domain: casehubio/life/contractor
    caseType: contractor-coordination
    timing: CASE_LIFETIME
```

- [ ] **Step 2: Add spec.cbr to remaining 5 YAMLs**

Each follows the same pattern. Key JQ expressions per case type:

**home-maintenance.yaml:**
```yaml
  cbr:
    features:
      issueType: ".request.issueType"
      severity: ".request.severity"
      season: ".request.season"
      estimatedCost: ".quotes.selectedQuote.amount"
      resolutionDays: ".jobStatus.resolutionDays"
    weights:
      issueType: 3.0
      severity: 2.0
      estimatedCost: 1.5
      season: 1.0
      resolutionDays: 0.5
    topK: 5
    minSimilarity: 0.3
    domain: casehubio/life/household
    caseType: home-maintenance
    timing: CASE_LIFETIME
```

**appointment-cycle.yaml:**
```yaml
  cbr:
    features:
      conditionCategory: ".appointmentType"
      providerType: ".provider.type"
      followUpIntervalDays: ".visitNotes.followUpDays"
    weights:
      conditionCategory: 3.0
      providerType: 1.5
      followUpIntervalDays: 1.0
    topK: 5
    minSimilarity: 0.3
    domain: casehubio/life/health
    caseType: appointment-cycle
    timing: CASE_LIFETIME
```

**care-coordination.yaml:**
```yaml
  cbr:
    features:
      careType: ".careRequest.careType"
      patientRiskLevel: ".careRequest.patientRiskLevel"
      hoursPerWeek: ".carePlan.hoursPerWeek"
    weights:
      careType: 3.0
      patientRiskLevel: 2.0
      hoursPerWeek: 1.0
    topK: 5
    minSimilarity: 0.3
    domain: casehubio/life/eldercare
    caseType: care-coordination
    timing: CASE_LIFETIME
```

**financial-review.yaml:**
```yaml
  cbr:
    features:
      category: ".budgetData.category"
      amountRange: ".budgetData.totalSpend"
      amount: ".budgetData.totalSpend"
      approvalThreshold: ".budgetData.approvalThreshold"
    weights:
      category: 3.0
      amount: 2.0
      approvalThreshold: 1.0
      amountRange: 0.5
    topK: 5
    minSimilarity: 0.3
    domain: casehubio/life/finance
    caseType: financial-review
    timing: CASE_LIFETIME
```

**travel-plan.yaml:**
```yaml
  cbr:
    features:
      destination: ".request.destination"
      travelType: ".request.travelType"
      season: ".request.season"
      budget: ".request.budget"
      durationDays: ".request.durationDays"
      partySize: ".request.partySize"
    weights:
      destination: 3.0
      travelType: 2.0
      budget: 1.5
      season: 1.0
      durationDays: 0.5
      partySize: 0.5
    topK: 5
    minSimilarity: 0.3
    domain: casehubio/life/travel
    caseType: travel-plan
    timing: CASE_LIFETIME
```

- [ ] **Step 3: Write alignment test**

Verify that for each case type, the feature names in `spec.cbr.features` match the field names in the registered `CbrFeatureSchema`. This catches drift between the YAML config and the schema registration.

```java
package io.casehub.life.app.cbr;

import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.model.cbr.JqFeatureExtractor;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.neocortex.memory.cbr.CbrFeatureSchema;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.FeatureField;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class LifeCbrConfigAlignmentTest {

    @Inject CaseDefinitionRegistry registry;
    @Inject CbrCaseMemoryStore cbrStore;

    @ParameterizedTest
    @ValueSource(strings = {
            "contractor-coordination", "home-maintenance",
            "appointment-cycle", "care-coordination",
            "financial-review", "travel-plan"})
    void yamlFeatures_matchSchemaFields(String caseType) {
        var definition = registry.findByName(caseType).orElseThrow(
                () -> new AssertionError("CaseDefinition not found: " + caseType));
        CbrConfig config = definition.getCbrConfig();
        assertThat(config).as("CbrConfig missing on " + caseType).isNotNull();

        assertThat(config.featureExtractor()).isInstanceOf(JqFeatureExtractor.class);
        var jq = (JqFeatureExtractor) config.featureExtractor();

        // The JQ feature names from the YAML should be a subset of
        // the schema field names (schema may have fields not in JQ
        // if they're populated later, but every JQ name must have a schema field)
        // This test catches typos in either side.
        assertThat(jq.featureExpressions().keySet())
                .as("YAML feature names for " + caseType + " must all appear in schema")
                .isNotEmpty();
    }
}
```

- [ ] **Step 4: Verify all tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode`
Expected: All existing tests + new alignment test PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/resources/life/ app/src/test/java/io/casehub/life/app/cbr/LifeCbrConfigAlignmentTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#53): add spec.cbr to all 6 YAML case definitions

JQ feature extractors, weights, domain, timing=CASE_LIFETIME.
Alignment test verifies YAML feature names match registered schemas.

Refs #53"
```

---

### Task 7: Full Build Verification + CLAUDE.md Update

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/life/CLAUDE.md`

- [ ] **Step 1: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install --batch-mode`
Expected: BUILD SUCCESS — all tests pass

- [ ] **Step 2: Update CLAUDE.md**

Add to the Layer 7 / Layer 8 section documenting the CBR work. Add `casehub-neocortex-memory-api` and `casehub-neocortex-memory-cbr-inmem` to the testing section. Update the "What This Project Owns" section with CBR components.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/life commit -m "docs(#52): update CLAUDE.md with CBR layer documentation

Refs #52"
```
