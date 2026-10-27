# LLM-Orchestrated Mechanical Coordination — Design Spec

**Date:** 2026-10-27
**Issue:** casehubio/life#140
**Status:** Draft
**Branch:** `issue-140-playbook-coordination`

## 1. Problem

casehub-life workers interact with connectors via MCP tool calls — each call
is an LLM round-trip that burns reasoning budget on mechanical work. A home
renovation case touches contacts, calendar, documents, location, and finance.
Each worker makes its own lookups independently. The LLM reasons about what
to look up, executes the lookup, processes results — for work that could be
batched and executed deterministically.

The connector surface is now 100+ GraphQL operations across 12+ domains
(contacts, calendar, bank, chat, commerce, document, email, health, location,
messaging, project, travel). Each operation is accessible both via MCP tools
(for in-flight LLM decisions) and via `@McpDomain` REST/GraphQL APIs (for
programmatic batching). The dual-path architecture is built; what's missing
is the orchestration layer that batches the programmatic path.

## 2. Solution

A four-phase execution model within existing case definitions:

1. **Planning worker** (AgentExec) — LLM selects/generates coordination
   playbooks, writes a `CoordinationPlan` to case context
2. **Pre-processing flow worker** (FlowWorkerFunction) — mechanical
   connector batches via compiled CaseHub playbooks
3. **Domain workers** (AgentExec, existing) — receive pre-assembled context,
   focus on judgment calls
4. **Post-processing flow worker** (FlowWorkerFunction) — mechanical
   notifications, document storage, calendar updates

The planning worker runs pre-engine (Phase 3 of Extended case start, §3.5).
The remaining three phases are steps in the YAML case definition. No new
engine PlanItem types, no new platform SPIs, no engine changes at all.

**Execution pipeline:** Playbook YAML → `PlaybookCompiler.compile()` →
`Workflow` → `FlowWorkerFunction` → `FlowWorkerFunctionHandler` (SWF
SDK-Java). Playbooks are the authoring format; SWF 1.0 is the execution
engine. This is the universal CaseHub pattern — playbooks everywhere,
SWF under the hood.

Connector operations use `rest-request:` steps pointing at connector
REST endpoints (`/api/connectors/{domain}/{operation}`). No
`CallableDispatcher` needed — playbooks make HTTP calls directly.

## 3. Architecture

### 3.1 Case definition shape

The YAML file contains only domain workers. The planning worker runs
pre-engine (§3.5), and flow workers are added programmatically per-case
by `buildFreshDefinition()` (§3.5) based on the planning output.

```yaml
# home-maintenance.yaml — domain workers only
# Flow workers (pre-stage-data, post-process-results) are NOT in this YAML.
# They are wired per-case by buildFreshDefinition() after the planning
# worker selects templates. See §3.5 Extended case start.
steps:
  - worker: schedule-inspection
    capability: schedule-inspection
    # AgentExec — domain judgment
  - worker: get-quotes
    capability: get-quotes
    # AgentExec — domain judgment
  # ... remaining domain workers ...
```

**After `buildFreshDefinition()` wires flow workers** (conceptual — shows
what the engine receives, not the YAML file content):

```
  pre-stage-data              ← FlowWorkerFunction (from "contractor-onboarding" template)
    call: casehub:connector/contacts/searchContacts
    call: casehub:connector/location/searchPlaces
    call: casehub:connector/calendar/listEvents
  schedule-inspection         ← AgentExec (from YAML, copied from cached base)
  get-quotes                  ← AgentExec (from YAML, copied from cached base)
  ...
  post-process-results        ← FlowWorkerFunction (from "completion-report" template)
    call: casehub:connector/documents/uploadFile
    call: casehub:connector/messaging/sendEmail
```

### 3.2 Connector operations via rest-request: steps

No custom dispatcher needed. Playbook `rest-request:` steps call connector
REST endpoints directly:

```yaml
steps:
  - rest-request:
      method: GET
      url: /api/connectors/contacts/searchContacts?query=${input.contractorName}
    timeout: 10s
    retry: 3
```

`PlaybookCompiler` compiles this to a `CallHTTP` task in the SWF `Workflow`.
`FlowWorkerFunctionHandler` executes it. The connector REST APIs (deployed
in the same Quarkus application via `@McpDomain` annotations) handle the
request. No `CallableDispatcher`, no `ConnectorDispatchRouter`, no engine
changes.

### 3.3 Coordination template catalog

**Purpose:** A library of reusable `.playbook.yaml` templates that the
planning worker can select from. Templates are domain-specific coordination
recipes authored in the CaseHub playbook language.

**Storage — two tiers:**

1. **Classpath templates** — curated, version-controlled YAML fragments
   at `app/src/main/resources/life/templates/`. Ship with the application.
   ```
   life/templates/
     contractor-onboarding.yaml     # search contacts + check trust + geocode
     appointment-prep.yaml          # search providers + check calendar + pull records
     completion-report.yaml         # upload document + send email + send chat
     financial-review-prep.yaml     # list transactions + list accounts + pull docs
   ```

2. **Runtime templates** — LLM-generated, stored in a `coordination_template`
   database table. Created when the planning worker encounters a situation
   with no matching classpath template.

**Schema:**
```java
@Entity
public class CoordinationTemplate {
    @Id UUID id;
    String name;           // "contractor-onboarding"
    String domain;         // "contractor-coordination"
    String templateYaml;   // Playbook YAML
    String description;    // LLM-generated description
    boolean curated;       // false for runtime, true for promoted
    Instant createdAt;
    int useCount;          // incremented each time selected
}
```

**Catalog service:**
```java
@ApplicationScoped
public class CoordinationTemplateCatalog {
    List<CoordinationTemplate> findByDomain(String domain);
    List<CoordinationTemplate> findByCapabilities(Set<String> capabilities);
    Optional<CoordinationTemplate> findByName(String name);
    void save(CoordinationTemplate template);  // runtime store

    /** Resolve a template name to a compiled SWF Workflow. */
    Workflow resolveToWorkflow(String templateName) {
        CoordinationTemplate t = findByName(templateName)
            .orElseThrow(() -> new IllegalArgumentException(
                "Unknown template: " + templateName));
        return compilePlaybook(t.getTemplateYaml());
    }

    /** Compile playbook YAML into a Workflow via PlaybookCompiler. */
    static Workflow compilePlaybook(String playbookYaml) {
        PlaybookDocument doc = PlaybookDocument.parse(playbookYaml);
        CompilationOutcome outcome = new PlaybookCompiler().compile(doc);
        if (outcome instanceof CompilationOutcome.Failure f) {
            throw new IllegalArgumentException(
                "Playbook compilation failed: " + f.diagnostics());
        }
        return ((CompilationOutcome.Success) outcome).result().workflow();
    }
}
```

The `resolveToWorkflow()` convenience method encapsulates the full
resolution chain: `findByName()` → `templateYaml()` → parse with
document wrapper → `WorkflowReader.readWorkflowFromString()`.
The same parse path that `FlowWorkerFunctionProvider.create()` uses
internally, extracted for reuse.

Classpath templates are loaded at startup and merged with runtime templates.
The catalog exposes a unified query surface for the planning worker.

### 3.4 Planning worker

An AgentExec worker that runs as the first step in each case. It receives:

- **Request context** — the incoming case request (what needs to happen)
- **CBR data** — via `CbrInputTransformer` (past cases, what worked)
- **Template catalog listing** — available templates with descriptions

It outputs a `CoordinationPlan`:
```json
{
  "preTemplate": "contractor-onboarding",
  "preParams": {
    "contractorName": "Acme Plumbing",
    "siteAddress": "42 Oak Lane, Bristol"
  },
  "postTemplate": "completion-report",
  "postParams": {
    "recipientEmail": "mark@example.com",
    "reportType": "maintenance-completion"
  },
  "generatedTemplates": [
    {
      "name": "site-assessment",
      "templateYaml": "..."
    }
  ],
  "workerActivation": {
    "skipWorkers": [],
    "workerParams": {}
  }
}
```

**Worker activation:** The `workerActivation` section controls domain
workers:
- `skipWorkers` — names of domain workers to remove from the definition
  (e.g., skip `get-quotes` when the contractor is pre-selected)
- `workerParams` — key-value pairs injected into case context for
  specific workers (e.g., `{"schedule-inspection": {"priority": "urgent"}}`)

`buildFreshDefinition()` applies `skipWorkers` by filtering the workers
list. `workerParams` are merged into `initialContext` before engine start.

**Guard conditions:** Pre/post-processing flow workers are only added by
`buildFreshDefinition()` when the planning worker selects a template.
If the planner doesn't select a template, no flow worker is added.

**Template generation:** When no existing template matches, the planner
generates a new `do:` block YAML and includes it in `generatedTemplates`.
The `LifeCaseService.startCase()` Extended case start retrieves generated
templates from the `CoordinationPlan` and wires them as `FlowWorkerFunction`
workers in the fresh `CaseDefinition`.

**Generated template validation:** Before a generated template is wired
into the case definition, it passes through `CoordinationTemplateValidator`:

1. **Operation allowlist** — every `call:` step in the generated YAML is
   checked against the set of operations registered in `ConnectorDispatchRouter`.
   Unknown operations are rejected.
2. **Structural validation** — the YAML is parsed as a SWF `do:` block via
   `FlowWorkerFunctionProvider.create()`. Malformed YAML fails fast.
3. **Parameter sanitisation** — `with:` blocks may only reference
   `${input.*}` and `${output.*}` expressions. Arbitrary expressions
   or external references are rejected.

Generated templates that pass validation are persisted to the runtime
template store for future reuse. Templates that fail validation cause
the planning worker to fall back to MCP tool calls for that operation set.

### 3.5 Template injection in configureCase()

The `LifeTypedCaseHub` subclasses wire domain workers in `configureCase()`.
The actual API is `configureCase(CaseDefinition definition)` — there is no
`CaseDefinitionBuilder`. Workers are added via `definition.getWorkers().add()`.

Flow workers are NOT added in `configureCase()` — they are added per-case
by `buildFreshDefinition()` (see below). The base cached definition contains
only domain workers:

```java
@Override
protected void configureCase(CaseDefinition definition) {
    // Domain workers only — no flow workers here.
    // Flow workers are added per-case by buildFreshDefinition().
    definition.getWorkers().add(agentWorker("schedule-inspection", ...));
    definition.getWorkers().add(agentWorker("get-quotes", ...));
    // ... etc
}
```

**Caching bypass — Extended case start:** `YamlCaseHub.getDefinition()`
uses volatile + double-checked locking to cache the `CaseDefinition` per
`CaseHub` instance. Calling `CaseHub.startCase(inputData)` returns this
cached singleton, making per-case dynamic template selection impossible.

The solution: `LifeCaseService.startCase()` calls `caseHubRuntime.startCase(
definition, inputData)` directly (the public `CaseHubRuntime` API), bypassing
`CaseHub.startCase()`.

**Per-case definition construction:** `CaseDefinition` has no `copy()`,
`clone()`, or `toBuilder()` method. However, its constructor is public:
`new CaseDefinition(namespace, name, version)` creates a fresh instance
with empty mutable lists (`getWorkers()`, `getCapabilities()`, `getBindings()`,
`getMilestones()`, `getGoals()` all return `ArrayList`). Setter methods
exist for non-list fields (`setDsl()`, `setTitle()`, `setSummary()`,
`setAgentDescriptors()`, `setSemanticData()`, `setCompletion()`, etc.).

`LifeTypedCaseHub.buildFreshDefinition(CoordinationPlan plan)`:

```java
public CaseDefinition buildFreshDefinition(CoordinationPlan plan) {
    CaseDefinition cached = getDefinition();  // base with domain workers
    CaseDefinition fresh = new CaseDefinition(
        cached.getNamespace(), cached.getName(), cached.getVersion());

    // Copy domain workers from cached base
    fresh.getWorkers().addAll(cached.getWorkers());
    // Copy other mutable lists
    fresh.getCapabilities().addAll(cached.getCapabilities());
    fresh.getBindings().addAll(cached.getBindings());
    fresh.getMilestones().addAll(cached.getMilestones());
    fresh.getGoals().addAll(cached.getGoals());
    // Copy setter-based fields
    fresh.setAgentDescriptors(cached.getAgentDescriptors());
    fresh.setCompletion(cached.getCompletion());
    // ... remaining setters

    // Apply workerActivation — skip/parameterise domain workers
    if (plan.workerActivation() != null) {
        if (plan.workerActivation().skipWorkers() != null) {
            fresh.getWorkers().removeIf(w ->
                plan.workerActivation().skipWorkers().contains(w.name()));
        }
        // workerParams are injected into case context for workers to read
    }

    // Add flow workers for this specific case
    if (plan.preTemplate() != null) {
        Workflow preWf = catalog.resolveToWorkflow(plan.preTemplate());
        fresh.getWorkers().add(0, Worker.builder()
            .name("pre-stage-data")
            .capabilityName("mechanical-pre-staging")
            .function(new FlowWorkerFunction(preWf))
            .build());
    }
    if (plan.postTemplate() != null) {
        Workflow postWf = catalog.resolveToWorkflow(plan.postTemplate());
        fresh.getWorkers().add(Worker.builder()
            .name("post-process-results")
            .capabilityName("mechanical-post-processing")
            .function(new FlowWorkerFunction(postWf))
            .build());
    }
    // Wire generated templates (validated by CoordinationTemplateValidator)
    for (var gen : plan.generatedTemplates()) {
        Workflow genWf = CoordinationTemplateCatalog
            .parseTemplateYaml(gen.templateYaml());
        fresh.getWorkers().add(Worker.builder()
            .name(gen.name())
            .capabilityName(gen.name())
            .function(new FlowWorkerFunction(genWf))
            .build());
    }
    return fresh;
}
```

This is a shallow copy — sufficient because `Worker` is an immutable Java
record and the only per-case variation is WHICH workers are in the list.
Shared references to bindings, milestones, goals are safe since they are
not modified per case.

**Follow-up:** A proper `CaseDefinition.copy()` method in `casehub-engine-api`
would eliminate the fragile field-by-field copy. Filed as an engine enhancement.

**Planning worker invocation (Phase 3):** The planning worker runs
pre-engine via direct `WorkerAgentInvoker.execute()` call — the same
mechanism that `AgentWorkerFunction` uses inside the engine, invoked
outside it:

```java
// In LifeCaseService.startCase(), after CBR retrieval (Phase 2):
WorkerAgentInvoker plannerInvoker = hub.agentProviderFactory
    .invokerFor(hub.agent(), planningSystemPrompt,
                null, CoordinationPlanResult.class,
                hub.cbrInputTransformer);

// Synchronous LLM call — blocks until completion (120s default timeout)
WorkerResult<Map<String, Object>> result = plannerInvoker.execute(initialContext);

// Deserialize to typed result
CoordinationPlanResult planResult = objectMapper.convertValue(
    result.output(), CoordinationPlanResult.class);

// Phase 4: build fresh definition with selected templates
CaseDefinition freshDef = hub.buildFreshDefinition(planResult.plan());

// Phase 5: start engine
UUID caseId = caseHubRuntime.startCase(freshDef, initialContext);
```

Error handling: `WorkerAgentInvoker` throws `AgentException` on LLM
failure, timeout, or invalid JSON response. `LifeCaseService.startCase()`
already catches `Exception` and calls `markFailed(trackerId)` — the
planning worker failure follows the same error recovery path as any
other case start failure.

This extends the existing three-phase case start (PP-20260529-3ffe28):

1. Case definition scaffold + tracking (existing Phase 1)
2. CBR retrieval + adaptation (existing Phase 2)
3. **Planning worker invocation** — synchronous `WorkerAgentInvoker.execute()`
   selects templates (new Phase 3)
4. **Case definition build** — `buildFreshDefinition(plan)` constructs
   a fresh definition with flow workers wired (new Phase 4)
5. **Engine start** — `caseHubRuntime.startCase(freshDefinition, inputData)`

The case definition is fully resolved before the engine starts executing.
Template YAML is loaded and parsed into immutable `Workflow` objects at
definition-build time. No runtime template swapping or TOCTOU race.

### 3.6 Component inventory

| Component | Module | New/Existing |
|-----------|--------|-------------|
| ~~`ConnectorCallableDispatcher`~~ | ~~app/~~ | ~~Removed — rest-request: steps call connector APIs directly~~ |
| ~~`ConnectorDispatchRouter`~~ | ~~app/~~ | ~~Removed — no routing needed~~ |
| `CoordinationTemplate` | app/ | New — JPA entity for runtime templates |
| `CoordinationTemplateCatalog` | app/ | New — unified query over classpath + runtime |
| `CoordinationTemplateValidator` | app/ | New — allowlist + structural validation for generated templates |
| `CoordinationPlan` | api/ | New — record for planning worker output (`templateYaml` is `String`) |
| `CoordinationPlanResult` | app/ | New — response schema for planning worker |
| Planning worker prompts | app/engine/ | New — per-CaseHub planning system prompts |
| Classpath templates | resources/life/templates/ | New — curated `do:` block YAML files |
| `FlowWorkerFunction` | casehub-engine-support-core | Existing — SWF execution |
| `FlowWorkerFunctionProvider` | casehub-engine-support-core | Existing — `do:` block parsing |
| `PlaybookCompiler` | casehub-platform-playbook-compiler | Existing — compiles playbook YAML → SWF Workflow |
| `CasehubCallableTaskBuilder` | casehub-engine-support-core | Existing — SWF `call:` routing |
| `CbrInputTransformer` | app/cbr/ | Existing — CBR context injection |

### 3.7 Flyway migrations

One new migration for the runtime template store:

- `V114__coordination_template.sql` — `coordination_template` table
  (id, name, domain, template_yaml, description, curated, created_at,
  use_count, tenancy_id)

Note: V113 is taken by `V113__household_and_members.sql`.

### 3.8 Dependencies

`PlaybookCompiler` is in `casehub-platform-playbook-compiler`. Add to
`app/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-playbook-compiler</artifactId>
</dependency>
```

`FlowWorkerFunction` and `FlowWorkerFunctionHandler` are in
`casehub-engine-support-core`, already a transitive dependency via
`casehub-engine`.

**No engine changes needed.** Playbook `rest-request:` steps compile to
SWF `CallHTTP` tasks. The SWF SDK-Java executes them directly as HTTP
calls. No `CallableDispatcher`, no prefix matching, no call name injection.

## 4. Case definition changes

Each of the 6 `LifeTypedCaseHub` subclasses gains:

1. A **planning worker** invoked programmatically by `LifeCaseService.startCase()`
   as Phase 3 of the Extended case start (not a YAML step)
2. A **pre-processing flow worker** (guarded) as a YAML step
3. A **post-processing flow worker** (guarded) as the last YAML step

The existing domain workers remain unchanged — they continue to use
AgentExec with MCP tool references for in-flight decisions. The
difference is that they now receive pre-assembled context from the
pre-processing flow worker when one runs.

### 4.1 Worker-to-template flow

```
Planning worker output:
  coordinationPlan.preTemplate = "contractor-onboarding"
  coordinationPlan.preParams = { contractorName: "...", siteAddress: "..." }

Pre-processing flow worker:
  Resolves "contractor-onboarding" from catalog
  Executes do: block with preParams as input
  Output: { contacts: [...], location: {...}, calendar: [...] }

Domain workers:
  Receive pre-staging output merged into their input
  Focus on judgment: "Is this quote reasonable?" not "Look up the contractor"

Post-processing flow worker:
  Resolves postTemplate from catalog
  Executes do: block with domain worker outputs as input
  Performs: document upload, email notification, calendar update
```

### 4.2 Curated templates (initial set)

| Template | Domain | Operations batched |
|----------|--------|--------------------|
| `contractor-onboarding` | contractor-coordination | searchContacts, searchPlaces, listTransactions |
| `appointment-prep` | health | searchProviders, searchAvailableSlots, listEvents |
| `site-assessment` | home-maintenance | searchPlaces, getNearby, searchContacts |
| `financial-review-prep` | finance | listAccounts, listTransactions, searchDocuments |
| `travel-research` | travel | searchTransport, searchAccommodation, searchPlaces |
| `completion-report` | cross-domain (post) | uploadFile, sendEmail, sendChat |
| `appointment-followup` | health (post) | createEvent, sendEmail |
| `payment-and-close` | contractor (post) | initiatePayment, uploadFile, sendChat |

## 5. Testing strategy

### 5.1 Unit tests

- `CoordinationTemplateCatalogTest` — classpath + runtime merge, query by domain/capabilities
- `ConnectorCallableDispatcherTest` — call name parsing, domain/operation resolution
- `CoordinationPlanResultTest` — serialisation/deserialisation
- `CoordinationTemplateValidatorTest` — operation allowlist enforcement, parameter sanitisation, malformed YAML rejection
- Planning worker prompt tests — verify template catalog listing is injected

### 5.2 Integration tests (@QuarkusTest)

- `LifeCoordinationIntegrationTest` — end-to-end: start case → planner selects template → flow worker executes → domain worker receives pre-assembled context
- `ConnectorCallableDispatcherQuarkusTest` — dispatcher registered, routes to connector APIs, returns results
- `CoordinationTemplateRepositoryTest` — CRUD for runtime templates

### 5.3 Template validation tests

- Each curated template YAML is validated against the SWF schema at build time
- Call names in templates (`casehub:connector/*`) are validated against known connector domains

## 6. Open questions

1. **Template versioning:** When a curated template changes, running cases
   use the version loaded at case start. No migration needed for in-flight
   cases. But runtime templates that evolve — should they be versioned?
   Defer: use `useCount` and `createdAt` for now; add versioning when the
   template library grows past initial set.

2. **Deterministic fast path (R1-05):** The planning worker always runs,
   even when template selection is obvious from case type. A follow-up
   could add a `LifeAdaptationRule` dimension that selects templates
   deterministically when features match known patterns, skipping the
   LLM call. Not in scope for v1.

3. **Re-planning (R1-08):** The `CoordinationPlan` is a static snapshot.
   If a pre-processing step fails, the SWF error handling within the
   template handles retry/fallback. Full re-planning (re-invoke the
   planner) is a follow-up if needed.

4. **Fire-and-forget side effects (R1-07):** Ledger writes, CBR retention,
   and other fire-and-forget side effects should remain as CDI observers
   (existing pattern). Post-processing templates handle sequenced,
   multi-step side effects (document upload → email → calendar update).
   The boundary: if ordering matters, use a template; if not, use a CDI
   observer.

## 7. Scope boundaries

**In scope:**
- CoordinationTemplate entity + catalog + CoordinationTemplateValidator
- CoordinationPlan record + planning worker prompt
- Curated playbook templates (8 initial `.playbook.yaml` files)
- Pre/post flow workers in 6 case definitions
- Flyway migration for template store (V114)
- casehub-platform-playbook-compiler dependency

**Out of scope:**
- **Case mapping** — LLM pre-maps which CasePlanModels to start with what
  context. Listed in issue #140 but orthogonal to template coordination.
  The planning worker selects templates within an already-chosen case type;
  case mapping selects which case type(s) to start. Follow-up issue filed:
  life#TBD.
- Deterministic template selection via LifeAdaptationRule (follow-up)
- Template versioning beyond createdAt/useCount (follow-up)
- Mid-case re-planning (follow-up)
- UI for template management (follow-up)
- LLM template generation prompt engineering (deferred to implementation)

**Cross-repo dependencies:**
- `casehub-platform-playbook-compiler` — PlaybookCompiler (existing, no changes)
- `casehub-engine-support-core` — FlowWorkerFunction, FlowWorkerFunctionHandler (existing, no changes)
- *(follow-up)* `CaseDefinition.copy()` in `casehub-engine-api` — proper copy method to replace field-by-field construction

**Resolved dependencies:**
- `casehub-connectors#174` (ConnectorDocumentApi expansion) — **CLOSED**, dependency satisfied

## References

- `FlowWorkerFunction.java` — casehub-engine-support-core (`io.casehub.engine.flow`), SWF worker execution
- `FlowWorkerFunctionProvider.java` — `do:` block parsing from YAML
- `FlowWorkerFunctionHandler.java` — SWF execution with WorkflowApplication
- `CasehubCallableTaskBuilder.java` — SWF `call:` step routing
- `CallableDispatcher.java` — dispatcher SPI (`@FunctionalInterface`)
- `CallableDispatchRegistry.java` — dispatcher registration
- `CasehubFlowContext.java` — static context holding `CallableDispatchRegistry` instance
- Blog: 2026-10-26-mdp01-connector-wiring-sprint.md — dual-path architecture
- Platform#566 — SWF as durable execution backend
- Platform#661 — playbook runtime (Serverless Workflow)
- Connectors#166 — HealthPlatform SPI
- Connectors#170 — TravelPlatform production implementation
- Connectors#169 — CommercePlatform production implementation
- Connectors#171 — Life-domain simulation corpuses
- Decision review: R1-01 through R1-11 (issue-140-decision-20261027-052504)
- PP-20260529-3ffe28 — three-phase case start
- PP-20261026-8a9dce — no-openclaw-dependencies
