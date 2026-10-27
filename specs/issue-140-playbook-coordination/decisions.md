# Decisions — Issue #140: Playbook Coordination

## D1: Execution format (REVISED after decision review)

**Choice:** Serverless Workflow YAML — mechanical coordination defined as `do:` blocks executed by `FlowWorkerFunction`
**Original choice:** Platform playbook YAML (.playbook.yaml)
**Revision trigger:** R1-01, R1-09 — the platform playbook runtime is a content/UI system (pages/aria), not a connector-operation batching engine. `FlowWorkerFunction` + CNCF Serverless Workflow SDK is the platform's actual multi-step execution model for workers.
**Alternatives:**
- Platform playbook YAML — content/UI-oriented runtime, not designed for connector batching
- Simplified life-specific format — doesn't benefit from SWF's error handling, branching, retry
**Rationale:** SWF is a CNCF standard with tooling, validation, and schema support. `FlowWorkerFunctionProvider` already handles `do:` blocks in YAML case definitions. `CasehubCallableTaskBuilder` routes `call:` steps to `CallableDispatcher` implementations. Error handling, conditional branching, and retry come free from the SDK.
**Trade-offs:** SWF YAML is more verbose than a purpose-built format; LLM generation must target a standards-track DSL
**Sources:** FlowWorkerFunction.java, FlowWorkerFunctionProvider.java, FlowWorkerFunctionHandler.java, CasehubCallableTaskBuilder.java, CallableDispatcher.java (all in casehub-engine-flow)
**Exploration:** quick → revised after decision review (R1-01, R1-09)
**Status:** revised

## D2: Case lifecycle integration (REVISED after decision review)

**Choice:** FlowWorkerFunction within standard worker bindings — mechanical coordination uses `do:` blocks in worker definitions, executed by the existing `FlowWorkerFunctionHandler`
**Original choice:** New PlaybookStep PlanItem type
**Revision trigger:** R1-02 — `PlanItemDefinition` is a sealed interface; `WorkerFunctionProvider` is the established extension point. `FlowWorkerFunction` wraps entire SWF workflows inside workers — same pattern.
**Alternatives:**
- New PlaybookStep PlanItem type — requires breaking sealed `PlanItemDefinition` hierarchy
- External orchestration layer — decoupled but adds coordination layer outside the case
**Rationale:** Zero engine modifications. `FlowWorkerFunctionProvider` already handles `do:` blocks. Workers are already multi-step execution containers (FlowWorkerFunction wraps full workflows). All engine lifecycle features (guards, conditions, binding evaluation, completion semantics, ledger writes) apply automatically.
**Trade-offs:** "Worker" label is slightly misleading for purely mechanical steps, but this is the established pattern
**Sources:** FlowWorkerFunctionProvider.java, WorkerFunctionProvider SPI, R1-02 (sealed hierarchy analysis)
**Exploration:** quick → revised after decision review (R1-02)
**Status:** revised

## D3: LLM planning trigger

**Choice:** Dedicated planning worker — a "coordinator" AgentExec worker runs as the first step in every case, selects/generates playbooks, writes coordination plan to context
**Alternatives:**
- LifeCaseService.startCase() phase — planning in Java during case start, couples to service layer
- Pre-case playbook — separate planning playbook before main case, adds two-phase startup
**Rationale:** Fits existing worker pattern (AgentExec with planning prompt). Keeps planning inside the case definition. Transparent — visible as a step like any other worker.
**Trade-offs:** Adds an LLM call at case start even when playbook selection could be deterministic. R1-05 suggests a deterministic fast path via LifeAdaptationRule — worth considering as a follow-up optimisation, but the LLM path is the correct starting point for novel situations.
**Sources:** LifeTypedCaseHub.agentWorker() pattern, CbrInputTransformer for context injection
**Exploration:** quick
**Status:** captured

## D4: Playbook catalog (REVISED after decision review)

**Choice:** Classpath `do:` block templates + runtime store — curated coordination templates as YAML fragments on classpath, LLM-generated templates in a runtime store (database). Life-owned catalog, not a platform SPI.
**Original choice:** Platform PlaybookRegistry SPI merging classpath + runtime
**Revision trigger:** R1-03, R1-04 — `PlaybookSchemaRegistry` already exists in the platform for a different purpose; a new platform SPI would duplicate or confuse. Domain-specific coordination recipes belong in life, not the foundation.
**Alternatives:**
- Platform PlaybookRegistry SPI — duplicates PlaybookSchemaRegistry, violates boundary rules for domain coordination
- All runtime (database-backed) — loses code-as-source-of-truth for curated templates
**Rationale:** The catalog stores domain-specific coordination recipes (contractor-onboarding, appointment-prep, etc.). These are life domain knowledge, not platform primitives. The platform already has the execution infrastructure (FlowWorkerFunction); life owns what to execute.
**Trade-offs:** Life-specific catalog isn't reusable by other CaseHub apps. If the pattern proves generic, extract to platform later.
**Sources:** PlaybookSchemaRegistry in casehub-platform-yaml-core, R1-03 (duplication analysis), R1-04 (boundary rules)
**Exploration:** quick → revised after decision review (R1-03, R1-04)
**Status:** revised

## D5: Post-processing model

**Choice:** Separate reusable template — post-processing is a `do:` block template, selected as a pre/post pair by the planner
**Alternatives:**
- Inline case steps — hardcoded final milestones in case definition, no dynamic selection
- CDI observers only — simpler but loses the coordination/sequencing that SWF provides (R1-07 noted this is partially valid for fire-and-forget side effects, but ordered multi-step post-processing like "store report THEN email summary THEN update trust" needs sequencing)
**Rationale:** Reusable — same post-processing template shared across case types. Follows the same catalog/selection pattern as pre-processing. SWF handles step ordering and error handling within the template.
**Trade-offs:** Planner must select coherent pre/post pairs; fire-and-forget side effects (ledger writes, CBR retention) should remain as CDI observers, not be pulled into templates
**Sources:** Issue #140 example: store completion report → email summary → update trust score
**Exploration:** quick
**Status:** captured

## D6: Plan output format

**Choice:** Structured plan in case context — planning worker writes a CoordinationPlan (JSON) to case context, downstream steps read from it
**Alternatives:**
- Engine signals per step — event-driven but harder to trace full coordination plan
- Dynamic case augmentation — most flexible but modifying running case is complex
**Rationale:** Simple, transparent, debuggable. Subsequent flow workers use guard conditions that check context for their template reference. Well-established engine pattern.
**Trade-offs:** Context payload grows; plan is a static snapshot. R1-08 notes no mid-case re-planning — acceptable for v1 since SWF provides error handling and retry within each template. Full re-planning is a follow-up if needed.
**Sources:** Existing case context pattern (cbrCalibration, adaptedPlan)
**Exploration:** quick
**Status:** captured

## D7: CBR integration

**Choice:** Planning worker queries CBR — receives CBR context via existing CbrInputTransformer, uses it to inform template selection
**Alternatives:**
- Extend CBR adaptation rules — add template selection as new dimension in LifePlanAdapter, couples two concerns
**Rationale:** No new CBR infrastructure needed. CBR provides data ("past cases used X template"), LLM does reasoning. CbrInputTransformer already wired to all workers.
**Trade-offs:** Template selection quality depends on LLM reasoning about CBR data, not structured retrieval. R1-06 suggests LifeAdaptationRule could handle deterministic selection — valid as a follow-up optimisation.
**Sources:** LifeCbrSuggestionService, CbrInputTransformer, LifePlanAdapter
**Exploration:** quick
**Status:** captured

## D8: Platform vs. life ownership (REVISED after decision review)

**Choice:** All in life, composing with existing platform primitives — no new platform SPIs or PlanItem types. Life wires FlowWorkerFunction (platform), CallableDispatcher (platform), and owns the domain catalog, planning worker, and connector dispatcher registration.
**Original choice:** Platform primitives (PlaybookRegistry SPI, PlaybookStep) + life wiring
**Revision trigger:** R1-04 — domain-specific coordination recipes belong in life; platform already provides the execution primitives (FlowWorkerFunction, CallableDispatcher, WorkerFunctionProvider)
**Alternatives:**
- Platform PlaybookRegistry SPI + PlaybookStep PlanItem — requires platform changes for domain-specific logic
**Rationale:** The platform provides generic execution (SWF runtime, callable dispatch). Life provides domain knowledge (what to execute, when, for which situations). This is the correct boundary — foundation owns mechanics, application owns domain logic.
**Trade-offs:** If another CaseHub app needs the same pattern, they'll build their own catalog. Extract to platform when the pattern proves generic.
**Sources:** PLATFORM.md boundary rules, R1-04 (boundary analysis)
**Exploration:** quick → revised after decision review (R1-04)
**Status:** revised

## D9: Overall architecture (REVISED after decision review)

**Choice:** FlowWorkerFunction-based coordination with planning worker — planning worker (AgentExec) writes CoordinationPlan to context; pre/post-processing workers (FlowWorkerFunction with `do:` blocks) execute mechanical connector batches via `call: casehub:connector/*`; guard conditions on flow workers check CoordinationPlan for activation.
**Original choice:** PlaybookStep as engine PlanItem type
**Revision trigger:** R1-02, R1-09 — FlowWorkerFunction is the established pattern for multi-step execution within worker bindings. Zero engine changes needed.
**Alternatives:**
- New PlaybookStep PlanItem type — requires breaking sealed PlanItemDefinition, adds engine complexity
- Pure AgentExec with MCP tools — current model, burns LLM tokens on mechanical work
**Rationale:** Composes entirely with existing platform primitives. Planning worker is a regular AgentExec. Mechanical pre/post-processing workers use FlowWorkerFunction (SWF do: blocks). ConnectorCallableDispatcher bridges call: steps to connector GraphQL API. Domain catalog provides reusable templates. No engine modifications, no new platform SPIs.
**Trade-offs:** Case definitions list all possible flow workers (most won't run for a given case). Guard conditions add visual noise but are a well-established engine pattern.
**Depends on:** D2 (FlowWorkerFunction), D3 (planning worker), D6 (structured plan in context)
**Sources:** FlowWorkerFunction, FlowWorkerFunctionProvider, CasehubCallableTaskBuilder, CallableDispatchRegistry (all casehub-engine-flow)
**Exploration:** quick → revised after decision review (R1-02, R1-09)
**Status:** revised

## D10: ConnectorCallableDispatcher — new bridge component

**Choice:** A `ConnectorCallableDispatcher` registered in `CallableDispatchRegistry` routes SWF `call: casehub:connector/{domain}/{operation}` steps to the connector platform's GraphQL API
**Alternatives:**
- Direct HTTP calls from SWF (no dispatcher) — SWF supports HTTP calls natively, but bypasses the connector SPI layer and loses platform auth/tenancy/audit
- Individual dispatchers per connector domain — 11 dispatchers instead of 1 generic dispatcher with domain routing
**Rationale:** Single dispatcher, generic routing. The call name encodes domain and operation (`casehub:connector/contacts/searchContacts`). The dispatcher resolves the domain, calls the GraphQL endpoint with the operation and params. Reusable for any connector domain without per-domain code.
**Trade-offs:** Single dispatcher must handle all 92 operations — error messages need to be clear about which domain/operation failed
**Sources:** CallableDispatcher.java, CallableDispatchRegistry.java, 92 connector operations across 11 domains (blog entry)
**Exploration:** quick
**Status:** captured
