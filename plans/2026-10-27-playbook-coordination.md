# Playbook Coordination Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #140 — feat: LLM-orchestrated mechanical coordination
**Issue group:** #140

**Goal:** Shift mechanical connector work from LLM tool calls to deterministic playbook execution. Planning worker selects/generates playbooks, `PlaybookWorkerFunction` wraps compiled playbooks, `FlowWorkerFunctionHandler` executes them.

**Architecture:** Planning worker (AgentExec) runs pre-engine, selects playbook templates from a catalog, writes a CoordinationPlan to context. `LifeCaseService.startCase()` builds a fresh CaseDefinition with `PlaybookWorkerFunction` workers wired from compiled playbooks. Connector operations use `rest-request:` steps in playbook YAML — HTTP calls to connector REST endpoints, no custom dispatcher.

**Tech Stack:** Java 21, Quarkus 3.32.2, CaseHub Playbook Language, PlaybookCompiler (casehub-platform-playbook-compiler), PlaybookWorkerFunction + FlowWorkerFunctionHandler (casehub-engine-support-core, engine#1263 — landed)

## Global Constraints

- No OpenClaw dependencies (PP-20261026-8a9dce) — use LifeAgentProviderFactory
- api/ module: zero framework imports, no JPA
- Flyway: life domain migrations at `db/life/migration/`, V114 next available
- `@Transactional` on service methods only, never resource methods
- All commits reference issue: `Refs #140`
- Test config: Flyway disabled (`migrate-at-start=false`), H2 `drop-and-create`
- `tenancyId` in tests: `"278776f9-e1b0-46fb-9032-8bddebdcf9ce"`
- Playbooks are the universal CaseHub format — SWF 1.0 is the execution engine underneath

---

## Batch 1: Foundation — types, entity, catalog, templates

### Task 1: CoordinationPlan + CoordinationTemplate + catalog + curated playbooks

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/coordination/CoordinationPlan.java`
- Create: `api/src/main/java/io/casehub/life/api/coordination/GeneratedTemplate.java`
- Create: `api/src/main/java/io/casehub/life/api/coordination/WorkerActivation.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/CoordinationPlanResult.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/CoordinationTemplate.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/CoordinationTemplateCatalog.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/CoordinationTemplateValidator.java`
- Create: `app/src/main/resources/db/life/migration/V114__coordination_template.sql`
- Create: `app/src/main/resources/life/playbooks/contractor-onboarding.playbook.yaml`
- Create: `app/src/main/resources/life/playbooks/appointment-prep.playbook.yaml`
- Create: `app/src/main/resources/life/playbooks/site-assessment.playbook.yaml`
- Create: `app/src/main/resources/life/playbooks/financial-review-prep.playbook.yaml`
- Create: `app/src/main/resources/life/playbooks/travel-research.playbook.yaml`
- Create: `app/src/main/resources/life/playbooks/completion-report.playbook.yaml`
- Create: `app/src/main/resources/life/playbooks/appointment-followup.playbook.yaml`
- Create: `app/src/main/resources/life/playbooks/payment-and-close.playbook.yaml`
- Create: `app/src/test/java/io/casehub/life/app/coordination/CoordinationPlanResultTest.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/CoordinationTemplateCatalogTest.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/CoordinationTemplateValidatorTest.java`

**Interfaces:**
- Produces: `CoordinationPlan(String preTemplate, Map<String,Object> preParams, String postTemplate, Map<String,Object> postParams, List<GeneratedTemplate> generatedTemplates, WorkerActivation workerActivation)`
- Produces: `CoordinationTemplateCatalog.findByName(String)` → `Optional<CoordinationTemplate>`
- Produces: `CoordinationTemplateCatalog.findByDomain(String)` → `List<CoordinationTemplate>`
- Produces: `CoordinationTemplateCatalog.all()` → `List<CoordinationTemplate>`
- Produces: `CoordinationTemplateCatalog.resolveToPlaybookWorker(String)` → `PlaybookWorkerFunction`
- Produces: `CoordinationTemplateCatalog.compilePlaybook(String)` → `CompilationResult` (static)
- Produces: `CoordinationTemplateValidator.validate(String)` → `ValidationResult`

- [ ] **Step 1: Create api/ records**

`CoordinationPlan.java`:
```java
package io.casehub.life.api.coordination;

import java.util.List;
import java.util.Map;

public record CoordinationPlan(
    String preTemplate,
    Map<String, Object> preParams,
    String postTemplate,
    Map<String, Object> postParams,
    List<GeneratedTemplate> generatedTemplates,
    WorkerActivation workerActivation) {}
```

`GeneratedTemplate.java`:
```java
package io.casehub.life.api.coordination;

public record GeneratedTemplate(String name, String templateYaml) {}
```

`WorkerActivation.java`:
```java
package io.casehub.life.api.coordination;

import java.util.List;
import java.util.Map;

public record WorkerActivation(
    List<String> skipWorkers,
    Map<String, Map<String, Object>> workerParams) {}
```

`CoordinationPlanResult.java` (app/):
```java
package io.casehub.life.app.coordination;

import io.casehub.life.api.coordination.CoordinationPlan;

public record CoordinationPlanResult(CoordinationPlan plan) {}
```

- [ ] **Step 2: Write CoordinationPlanResult serialisation test**

```java
package io.casehub.life.app.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.life.api.coordination.*;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class CoordinationPlanResultTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Test
    void round_trips_through_json() throws Exception {
        var plan = new CoordinationPlan(
            "contractor-onboarding", Map.of("contractorName", "Acme"),
            "completion-report", Map.of("recipientEmail", "mark@example.com"),
            List.of(new GeneratedTemplate("custom", "steps:\n  - log: test")),
            new WorkerActivation(List.of("get-quotes"), Map.of()));
        String json = MAPPER.writeValueAsString(new CoordinationPlanResult(plan));
        var result = MAPPER.readValue(json, CoordinationPlanResult.class);
        assertThat(result.plan().preTemplate()).isEqualTo("contractor-onboarding");
        assertThat(result.plan().workerActivation().skipWorkers()).containsExactly("get-quotes");
    }

    @Test
    void handles_null_templates() throws Exception {
        var plan = new CoordinationPlan(null, null, null, null, List.of(), null);
        String json = MAPPER.writeValueAsString(new CoordinationPlanResult(plan));
        var result = MAPPER.readValue(json, CoordinationPlanResult.class);
        assertThat(result.plan().preTemplate()).isNull();
    }
}
```

- [ ] **Step 3: Run serialisation test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CoordinationPlanResultTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

- [ ] **Step 4: Create CoordinationTemplate JPA entity + V114 migration**

Entity at `app/src/main/java/io/casehub/life/app/coordination/CoordinationTemplate.java` — standard JPA entity with fields: id (UUID), name (unique), domain, templateYaml (TEXT), description (TEXT), curated (boolean), createdAt, useCount, tenancyId.

Migration `V114__coordination_template.sql`:
```sql
CREATE TABLE coordination_template (
    id            UUID         NOT NULL PRIMARY KEY,
    name          VARCHAR(255) NOT NULL UNIQUE,
    domain        VARCHAR(255) NOT NULL,
    template_yaml TEXT         NOT NULL,
    description   TEXT,
    curated       BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at    TIMESTAMP    NOT NULL,
    use_count     INT          NOT NULL DEFAULT 0,
    tenancy_id    VARCHAR(255) NOT NULL
);
CREATE INDEX idx_coordination_template_domain ON coordination_template(domain);
```

- [ ] **Step 5: Create 8 curated playbook templates**

Each at `app/src/main/resources/life/playbooks/`. Example `contractor-onboarding.playbook.yaml`:

```yaml
playbook: "1.0"
schema: client
---
meta:
  title: Contractor Onboarding
  description: Search contacts, geocode site, pull recent transactions
  tags: [contractor-coordination, pre-processing]
  domain: contractor-coordination
---
steps:
  - rest-request:
      method: GET
      url: /api/connectors/contacts/searchContacts?query=${input.contractorName}
    timeout: 10s
    retry: 3
  - rest-request:
      method: GET
      url: /api/connectors/location/searchPlaces?query=${input.siteAddress}
    timeout: 10s
  - rest-request:
      method: GET
      url: /api/connectors/bank/listTransactions?accountId=${input.accountId}&limit=10
    timeout: 15s
    retry: 2
```

All 8 templates follow this pattern with domain-appropriate operations:
- `appointment-prep` — searchProviders, searchAvailableSlots, listEvents
- `site-assessment` — searchPlaces, searchNearby, searchContacts
- `financial-review-prep` — listAccounts, listTransactions, searchDocuments
- `travel-research` — searchTransport, searchAccommodation, searchPlaces
- `completion-report` — uploadFile (POST), sendEmail (POST), sendChat (POST)
- `appointment-followup` — createEvent (POST), sendEmail (POST)
- `payment-and-close` — initiatePayment (POST), uploadFile (POST), sendChat (POST)

- [ ] **Step 6: Write failing test for CoordinationTemplateCatalog**

```java
package io.casehub.life.app.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CoordinationTemplateCatalogTest {

    private CoordinationTemplateCatalog catalog;

    @BeforeEach
    void setUp() {
        catalog = new CoordinationTemplateCatalog();
        catalog.loadClasspathPlaybooks("life/playbooks/");
    }

    @Test
    void finds_classpath_template_by_name() {
        var result = catalog.findByName("contractor-onboarding");
        assertThat(result).isPresent();
        assertThat(result.get().getDomain()).isEqualTo("contractor-coordination");
        assertThat(result.get().isCurated()).isTrue();
    }

    @Test
    void finds_templates_by_domain() {
        var results = catalog.findByDomain("contractor-coordination");
        assertThat(results).isNotEmpty();
    }

    @Test
    void resolves_template_to_playbook_worker_function() {
        var pwf = catalog.resolveToPlaybookWorker("contractor-onboarding");
        assertThat(pwf).isNotNull();
        assertThat(pwf.workflow().getDo()).isNotEmpty();
        assertThat(pwf.compilationResult()).isNotNull();
    }

    @Test
    void compile_playbook_produces_valid_compilation_result() {
        String yaml = """
            steps:
              - rest-request:
                  method: GET
                  url: /api/connectors/contacts/searchContacts?query=test
            """;
        var result = CoordinationTemplateCatalog.compilePlaybook(yaml);
        assertThat(result).isNotNull();
        assertThat(result.workflow().getDo()).hasSize(1);
    }

    @Test
    void throws_for_unknown_template() {
        assertThatThrownBy(() -> catalog.resolveToWorkflow("nonexistent"))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 7: Implement CoordinationTemplateCatalog**

Key methods — `compilePlaybook()` returns the full `CompilationResult` (preserving source maps and coordination specs for error reporting), and `resolveToPlaybookWorker()` wraps it in `PlaybookWorkerFunction`:

```java
public static CompilationResult compilePlaybook(String playbookYaml) {
    var doc = PlaybookDocument.parse(playbookYaml);
    var outcome = new PlaybookCompiler().compile(doc);
    if (outcome instanceof CompilationOutcome.Failure f) {
        throw new IllegalArgumentException(
            "Playbook compilation failed: " + f.diagnostics());
    }
    return ((CompilationOutcome.Success) outcome).result();
}

public PlaybookWorkerFunction resolveToPlaybookWorker(String templateName) {
    CoordinationTemplate t = findByName(templateName)
        .orElseThrow(() -> new IllegalArgumentException(
            "Unknown template: " + templateName));
    CompilationResult result = compilePlaybook(t.getTemplateYaml());
    return new PlaybookWorkerFunction(result.workflow(), result);
}
```

`loadClasspathPlaybooks()` reads `.playbook.yaml` files from classpath, extracts `meta.domain` and `meta.description`, creates `CoordinationTemplate` entries in the in-memory map.

- [ ] **Step 8: Write failing test for CoordinationTemplateValidator**

```java
package io.casehub.life.app.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class CoordinationTemplateValidatorTest {

    private final CoordinationTemplateValidator validator = new CoordinationTemplateValidator();

    @Test
    void accepts_valid_playbook_with_rest_requests() {
        String yaml = """
            steps:
              - rest-request:
                  method: GET
                  url: /api/connectors/contacts/searchContacts?query=test
            """;
        var result = validator.validate(yaml);
        assertThat(result.valid()).isTrue();
    }

    @Test
    void rejects_external_urls() {
        String yaml = """
            steps:
              - rest-request:
                  method: GET
                  url: https://evil.com/steal-data
            """;
        var result = validator.validate(yaml);
        assertThat(result.valid()).isFalse();
        assertThat(result.errors()).anyMatch(e -> e.contains("external URL"));
    }

    @Test
    void rejects_malformed_yaml() {
        var result = validator.validate("{{invalid yaml}}");
        assertThat(result.valid()).isFalse();
    }
}
```

- [ ] **Step 9: Implement CoordinationTemplateValidator**

Validates: structural compilation via `PlaybookCompiler`, URL allowlist (only `/api/connectors/*` paths), no external HTTP calls.

- [ ] **Step 10: Run all coordination tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="io.casehub.life.app.coordination.*" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: ALL PASS

- [ ] **Step 11: Add playbook-compiler dependency to app/pom.xml**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-playbook-compiler</artifactId>
</dependency>
```

- [ ] **Step 12: Commit**

```bash
git add api/src/main/java/io/casehub/life/api/coordination/
git add app/src/main/java/io/casehub/life/app/coordination/
git add app/src/main/resources/db/life/migration/V114__coordination_template.sql
git add app/src/main/resources/life/playbooks/
git add app/src/test/java/io/casehub/life/app/coordination/
git add app/pom.xml
git commit -m "feat(#140): add coordination types, catalog, playbook templates, and validator

CoordinationPlan record (api/) for planning worker output.
CoordinationTemplate JPA entity with V114 migration.
CoordinationTemplateCatalog compiles .playbook.yaml via PlaybookCompiler.
CoordinationTemplateValidator enforces URL allowlist.
8 curated playbook templates for connector coordination.

Refs #140"
```

---

## Batch 2: Case Integration

Planning worker, buildFreshDefinition(), and LifeCaseService extended to 5-phase start.

### Task 2: buildFreshDefinition + planning worker + LifeCaseService extension

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeTypedCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/LifeCaseService.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/BuildFreshDefinitionTest.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/LifeCoordinationIntegrationTest.java`

**Interfaces:**
- Consumes: `CoordinationPlan`, `CoordinationTemplateCatalog`, `CoordinationTemplateValidator` — from Task 1
- Consumes: `LifeAgentProviderFactory.invokerFor()` — existing
- Consumes: `CaseHubRuntime.startCase(CaseDefinition, Map)` — existing
- Consumes: `PlaybookWorkerFunction(Workflow, CompilationResult)` — from engine#1263
- Produces: `LifeTypedCaseHub.buildFreshDefinition(CoordinationPlan)` → `CaseDefinition`
- Produces: Extended 5-phase `LifeCaseService.startCase()`

- [ ] **Step 1: Write failing test for buildFreshDefinition**

Test that given a CoordinationPlan with pre/post template names, the fresh definition includes PlaybookWorkerFunction workers at the right positions.

- [ ] **Step 2: Implement buildFreshDefinition on LifeTypedCaseHub**

```java
public CaseDefinition buildFreshDefinition(CoordinationPlan plan) {
    CaseDefinition cached = getDefinition();
    CaseDefinition fresh = new CaseDefinition(
        cached.getNamespace(), cached.getName(), cached.getVersion());
    // Copy domain workers, capabilities, bindings, milestones, goals
    fresh.getWorkers().addAll(cached.getWorkers());
    // ... copy all other fields from cached ...

    // Apply skipWorkers
    if (plan.workerActivation() != null && plan.workerActivation().skipWorkers() != null) {
        fresh.getWorkers().removeIf(w ->
            plan.workerActivation().skipWorkers().contains(w.name()));
    }

    // Add pre-processing playbook worker
    if (plan.preTemplate() != null) {
        PlaybookWorkerFunction preWf = coordinationCatalog.resolveToPlaybookWorker(plan.preTemplate());
        fresh.getWorkers().add(0, Worker.builder()
            .name("pre-stage-data")
            .capabilityName("mechanical-pre-staging")
            .function(preWf)
            .build());
    }

    // Add post-processing playbook worker
    if (plan.postTemplate() != null) {
        PlaybookWorkerFunction postWf = coordinationCatalog.resolveToPlaybookWorker(plan.postTemplate());
        fresh.getWorkers().add(Worker.builder()
            .name("post-process-results")
            .capabilityName("mechanical-post-processing")
            .function(postWf)
            .build());
    }

    // Wire validated generated playbooks
    for (var gen : plan.generatedTemplates()) {
        var validation = coordinationValidator.validate(gen.templateYaml());
        if (validation.valid()) {
            CompilationResult result = CoordinationTemplateCatalog.compilePlaybook(gen.templateYaml());
            fresh.getWorkers().add(Worker.builder()
                .name(gen.name()).capabilityName(gen.name())
                .function(new PlaybookWorkerFunction(result.workflow(), result))
                .build());
        }
    }
    return fresh;
}
```

- [ ] **Step 3: Run buildFreshDefinition test**

Expected: PASS

- [ ] **Step 4: Extend LifeCaseService.startCase() to 5-phase**

Insert between CBR retrieval (Phase 2) and engine start:

Phase 3: `invokePlanningWorker(hub, initialContext)` — synchronous LLM call via `WorkerAgentInvoker.execute()`, returns `CoordinationPlan` (null on failure — non-fatal fallback).

Phase 4: `hub.buildFreshDefinition(plan)` — builds fresh CaseDefinition with playbook workers wired. Falls back to `hub.getDefinition()` if plan is null.

Phase 5: `caseHubRuntime.startCase(freshDefinition, initialContext)` — engine start with the fresh definition.

- [ ] **Step 5: Write integration test**

```java
@QuarkusTest
class LifeCoordinationIntegrationTest {
    @Inject LifeCaseService caseService;

    @Test
    void case_starts_with_coordination_planning() {
        // TestLifeAgentProviderFactory returns canned CoordinationPlanResult
        // for the planning worker prompt.
        // Verify: case starts, no errors.
    }

    @Test
    void case_starts_without_coordination_when_planner_returns_null() {
        // Verify: graceful fallback, original case definition used.
    }
}
```

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode -am`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/engine/LifeTypedCaseHub.java
git add app/src/main/java/io/casehub/life/app/service/LifeCaseService.java
git add app/src/test/java/io/casehub/life/app/coordination/
git commit -m "feat(#140): wire playbook coordination into case start

LifeTypedCaseHub.buildFreshDefinition() creates per-case CaseDefinition
with PlaybookWorkerFunction workers compiled from playbook templates.
LifeCaseService.startCase() extended to 5-phase: scaffold → CBR →
planning worker → definition build → engine start.

Refs #140"
```

---

## References

- [2026-10-27-playbook-coordination-design.md] — design spec
- [PlaybookCompiler.java] — casehub-platform-playbook-compiler
- [PlaybookWorkerFunction.java] — casehub-engine-support-core (engine#1263, landed)
- [PlaybookWorkerFunctionProvider.java] — detects `steps:`, compiles via PlaybookCompiler
- [FlowWorkerFunctionHandler.java] — SWF execution (handles both FlowWorkerFunction and PlaybookWorkerFunction via SWFWorkerFunction sealed interface)
- [LifeCaseService.java] — 4-phase case start (extends to 5)
- [LifeTypedCaseHub.java] — abstract base for case hubs
- [WorkerAgentInvoker.java] — synchronous agent execution
- [PP-20261026-8a9dce] — no-openclaw-dependencies protocol
- [PP-20260529-3ffe28] — three-phase case start protocol
- [casehubio/life#140] — focal issue
- [casehubio/engine#1263] — PlaybookWorkerFunction (landed, closed)
