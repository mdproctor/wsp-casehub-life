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
   templates, writes a `CoordinationPlan` to case context
2. **Pre-processing flow worker** (FlowWorkerFunction) — mechanical
   connector batches via Serverless Workflow `do:` blocks
3. **Domain workers** (AgentExec, existing) — receive pre-assembled context,
   focus on judgment calls
4. **Post-processing flow worker** (FlowWorkerFunction) — mechanical
   notifications, document storage, calendar updates

The planning worker runs pre-engine (Phase 3 of Extended case start, §3.5).
The remaining three phases are steps in the YAML case definition. No new
engine PlanItem types, no new platform SPIs. The architecture composes
entirely with existing primitives:

- `FlowWorkerFunction` (casehub-engine-support-core, package `io.casehub.engine.flow`) — executes SWF `do:` blocks
- `FlowWorkerFunctionProvider` — resolves `do:` blocks from YAML
- `CasehubCallableTaskBuilder` — routes SWF `call:` steps to dispatchers
- `CallableDispatchRegistry` — maps call names to `CallableDispatcher` impls (requires prefix-match enhancement — see §3.2)

## 3. Architecture

### 3.1 Case definition shape

```yaml
# Example: home-maintenance-cycle.yaml (revised)
# NOTE: The planning worker is NOT a YAML step — it is invoked programmatically
# by LifeCaseService.startCase() as Phase 3 of the Extended case start (§3.5).
# The YAML contains only engine-executed workers.
steps:
  - worker: pre-stage-data
    capability: mechanical-pre-staging
    condition: "coordinationPlan.preTemplate != null"
    # FlowWorkerFunction — do: block loaded from catalog template
    do:
      - call: casehub:connector/contacts/searchContacts
        with:
          query: "${input.contractorName}"
      - call: casehub:connector/location/searchPlaces
        with:
          query: "${input.siteAddress}"
      - call: casehub:connector/calendar/listEvents
        with:
          startDate: "${input.startDate}"
          endDate: "${input.endDate}"

  # ... existing domain workers (AgentExec) ...
  # Receive pre-assembled context from pre-staging output

  - worker: post-process-results
    capability: mechanical-post-processing
    condition: "coordinationPlan.postTemplate != null"
    # FlowWorkerFunction — do: block loaded from catalog template
    do:
      - call: casehub:connector/documents/uploadFile
        with:
          fileName: "${output.reportName}"
          content: "${output.reportContent}"
      - call: casehub:connector/messaging/sendEmail
        with:
          to: "${output.recipientEmail}"
          subject: "${output.emailSubject}"
          body: "${output.emailBody}"
```

### 3.2 ConnectorCallableDispatcher

A single `CallableDispatcher` implementation registered in
`CallableDispatchRegistry` at startup. Routes SWF `call:` steps
to connector REST APIs.

**Call name convention:** `casehub:connector/{domain}/{operation}`

Examples:
- `casehub:connector/contacts/searchContacts`
- `casehub:connector/health/searchProviders`
- `casehub:connector/documents/uploadFile`
- `casehub:connector/bank/listTransactions`

**Resolution:** The dispatcher parses the call name into domain and
operation, constructs the REST path from the `@McpDomain.basePath`
convention (`/api/connectors/{domain}/{operationPath}`), and invokes
it with the `with:` params as query parameters (GET/queries) or
request body (POST/mutations).

```java
@ApplicationScoped
public class ConnectorCallableDispatcher implements CallableDispatcher {

    @Inject ConnectorDispatchRouter router;

    @Override
    public CompletableFuture<Map<String, Object>> dispatch(
            String workflowInstanceId, Map<String, Object> args) {
        String domain = args.get("_domain").toString();
        String operation = args.get("_operation").toString();
        Map<String, Object> params = filterInternalKeys(args);
        return router.invoke(domain, operation, params);
    }
}
```

The `ConnectorDispatchRouter` resolves domain → connector REST endpoint
using a static domain-to-basePath mapping table. The connector REST/GraphQL
APIs (in `casehub-connectors`) are deployed in the same Quarkus application
as life — invocation is local HTTP to `localhost:{port}/api/connectors/{domain}/{operation}`.
The `@McpDomain` annotations on connector API classes are in the connectors
repo (not on life's compile classpath), so discovery uses a static map rather
than runtime annotation scanning.

**Registration lifecycle:** `CasehubFlowContext.init()` is called by the
engine framework at bootstrap, creating the `CallableDispatchRegistry`
instance. `ConnectorCallableDispatcher` registers via a CDI `@Observes
StartupEvent` observer that calls
`CasehubFlowContext.callableRegistry().registerPrefix("casehub:connector/", this)`.
This requires the `registerPrefix()` enhancement (see §3.8).

### 3.3 Coordination template catalog

**Purpose:** A library of reusable `do:` block templates that the planning
worker can select from. Templates are domain-specific coordination recipes.

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
    String templateYaml;   // SWF do: block YAML
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
}
```

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

**Guard conditions:** Pre/post-processing flow workers have guards that
check `coordinationPlan.preTemplate` / `coordinationPlan.postTemplate`.
If the planner doesn't select a template, the flow worker is skipped.

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

The `LifeTypedCaseHub` subclasses wire flow workers in `configureCase()`.
The actual API is `configureCase(CaseDefinition definition)` — there is no
`CaseDefinitionBuilder`. Workers are added via `definition.getWorkers().add()`
with `Worker.builder()`:

```java
@Override
protected void configureCase(CaseDefinition definition) {
    // NOTE: planning worker is NOT added here — it runs pre-engine
    // in LifeCaseService.startCase() Phase 3

    // ... existing domain workers (agentWorker calls) ...

    // Pre-processing — FlowWorkerFunction, guarded
    // Template wired at Extended case start (§3.5 below), not here
    definition.getWorkers().add(Worker.builder()
        .name("pre-stage-data")
        .capabilityName("mechanical-pre-staging")
        .function(new FlowWorkerFunction(defaultPreTemplate()))
        .build());

    // Post-processing — FlowWorkerFunction, guarded
    definition.getWorkers().add(Worker.builder()
        .name("post-process-results")
        .capabilityName("mechanical-post-processing")
        .function(new FlowWorkerFunction(defaultPostTemplate()))
        .build());
}
```

**Caching bypass — Extended case start:** `YamlCaseHub.getDefinition()`
uses volatile + double-checked locking to cache the `CaseDefinition` per
`CaseHub` instance. Calling `CaseHub.startCase(inputData)` returns this
cached singleton, making per-case dynamic template selection impossible.

The solution: `LifeCaseService.startCase()` calls `caseHubRuntime.startCase(
definition, inputData)` directly (the public `CaseHubRuntime` API), bypassing
`CaseHub.startCase()`. The `LifeTypedCaseHub` exposes a
`buildFreshDefinition(CoordinationPlan plan)` method that:

1. Calls `getDefinition()` to get the base cached definition
2. Deep-copies it
3. Replaces the flow worker `FlowWorkerFunction` instances with the
   templates selected by the planning worker

This extends the existing three-phase case start (PP-20260529-3ffe28):

1. Case definition scaffold (existing Phase 1)
2. CBR retrieval + adaptation (existing Phase 2)
3. **Planning worker invocation** — LLM selects templates (new Phase 3)
4. **Case definition build** — `buildFreshDefinition(plan)` wires selected
   templates into a fresh definition copy (new Phase 4)
5. **Engine start** — `caseHubRuntime.startCase(freshDefinition, inputData)`

The case definition is fully resolved before the engine starts executing.
Template YAML is loaded and parsed into immutable `Workflow` objects at
definition-build time. No runtime template swapping or TOCTOU race.

### 3.6 Component inventory

| Component | Module | New/Existing |
|-----------|--------|-------------|
| `ConnectorCallableDispatcher` | app/ | New — routes `call: casehub:connector/*` to REST |
| `ConnectorDispatchRouter` | app/ | New — resolves domain/operation to REST path |
| `CoordinationTemplate` | app/ | New — JPA entity for runtime templates |
| `CoordinationTemplateCatalog` | app/ | New — unified query over classpath + runtime |
| `CoordinationTemplateValidator` | app/ | New — allowlist + structural validation for generated templates |
| `CoordinationPlan` | api/ | New — record for planning worker output (`templateYaml` is `String`) |
| `CoordinationPlanResult` | app/ | New — response schema for planning worker |
| Planning worker prompts | app/engine/ | New — per-CaseHub planning system prompts |
| Classpath templates | resources/life/templates/ | New — curated `do:` block YAML files |
| `FlowWorkerFunction` | casehub-engine-support-core | Existing — SWF execution |
| `FlowWorkerFunctionProvider` | casehub-engine-support-core | Existing — `do:` block parsing |
| `CallableDispatchRegistry` | casehub-engine-support-core | Existing — dispatcher routing (needs `registerPrefix()`) |
| `CasehubCallableTaskBuilder` | casehub-engine-support-core | Existing — SWF `call:` routing |
| `CbrInputTransformer` | app/cbr/ | Existing — CBR context injection |

### 3.7 Flyway migrations

One new migration for the runtime template store:

- `V114__coordination_template.sql` — `coordination_template` table
  (id, name, domain, template_yaml, description, curated, created_at,
  use_count, tenancy_id)

Note: V113 is taken by `V113__household_and_members.sql`.

### 3.8 Engine-level dependency: CallableDispatchRegistry prefix matching

`FlowWorkerFunction`, `FlowWorkerFunctionProvider`, `FlowWorkerFunctionHandler`,
`CasehubCallableTaskBuilder`, `CallableDispatcher`, and `CallableDispatchRegistry`
are all in `casehub-engine-support-core` (artifact `casehub-engine-support-core`,
package `io.casehub.engine.flow`). This artifact is already a transitive
dependency via `casehub-engine` in `app/pom.xml`. No new dependency needed.

**Required engine change:** `CallableDispatchRegistry` currently uses
`ConcurrentHashMap.get()` for exact-match lookup only. The spec's
single-dispatcher model requires prefix-based dispatch. Add a
`registerPrefix(String prefix, CallableDispatcher dispatcher)` method
that stores prefix-based dispatchers separately. The `get()` method
tries exact match first, then falls back to longest-prefix match:

```java
public CallableDispatcher get(final String callName) {
    CallableDispatcher d = dispatchers.get(callName);
    if (d != null) return d;
    for (var entry : prefixDispatchers.entrySet()) {
        if (callName.startsWith(entry.getKey())) return entry.getValue();
    }
    throw new UnsupportedOperationException("No dispatcher for: " + callName);
}
```

This is a backwards-compatible enhancement — existing exact-match
registrations continue to work. File engine issue for this change.

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
- ConnectorCallableDispatcher + ConnectorDispatchRouter
- CoordinationTemplate entity + catalog + CoordinationTemplateValidator
- CoordinationPlan record + planning worker prompt
- Curated templates (8 initial)
- Pre/post flow workers in 6 case definitions
- Flyway migration for template store (V114)
- CallableDispatchRegistry prefix-match enhancement (engine-level, casehub-engine-support-core)

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
- `casehub-engine-support-core`: `CallableDispatchRegistry.registerPrefix()` — new method
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
