# Decisions — issue-141-playbook-failure-recovery

## D1: Triage scope — per-worker budget

**Choice:** Per-worker triage. Each playbook worker independently gets up to 2 triage rounds before falling back to MCP tools.
**Alternatives:**
- Per-case — 2 total rounds across all workers. One bad worker blocks recovery for all others.
- Hybrid — per-worker limit of 2 + global per-case limit of 4. Prevents both individual and aggregate runaway.
**Rationale:** Workers are independent — a pre-processing failure shouldn't consume the post-processing triage budget. The engine's `triageRound` mechanism in `RetryOrchestrator` already scopes per-worker, making this the natural alignment.
**Trade-offs:** No global safeguard against a case where every worker hits its triage cap (worst case: 2 triage rounds × N playbook workers). Acceptable because each triage is a fast LLM call, not a cascading retry.
**Sources:** `engine/common-core/.../RetryOrchestrator.java` — triageRound scoping per binding/worker
**Exploration:** quick
**Status:** captured

## D2: Improved playbook persistence

**Choice:** Persist improved playbooks to `CoordinationTemplateCatalog` (DB, `curated=false`), replacing the broken version. Future cases benefit immediately.
**Alternatives:**
- Current case only — simpler but the same playbook fails again next time.
- Persist as variant — store alongside original with revision marker. Preserves history but adds complexity and the planner must choose between versions.
**Rationale:** Matches the issue's explicit goal: "the catalog evolves: broken playbooks get fixed, edge cases get handled." The template catalog already distinguishes curated (classpath) from generated (DB) — replacement is natural for generated templates.
**Trade-offs:** Curated templates are not replaced (they're loaded from classpath and would be re-loaded on restart). This means curated template fixes require a code change, which is correct — curated templates should be fixed at the source.
**Sources:** `life/app/coordination/CoordinationTemplateCatalog.java` — classpath vs DB tiers
**Exploration:** quick
**Status:** captured

## D3: Partial failure detection location

**Choice:** Engine-side detection. `FlowWorkerFunctionHandler` produces `PlaybookExecutionResult.PartialSuccess` when any step in the step results has `StepStatus.FAILED` but the workflow didn't throw.
**Alternatives:**
- Life wrapper — post-validate output against expected fields. Duplicates step execution knowledge.
- Both — engine for step-level, life for domain-level. Defense in depth but more complex.
**Rationale:** The engine already has all the data: `FlowExecutionRegistry.getSteps()` returns per-step results with `StepStatus`. The check is ~5 lines. Partial failure is an execution concern, not a domain concern.
**Trade-offs:** Requires a small engine change (engine issue needed). Life can't detect partial failures until the engine change lands. Acceptable because the `PartialSuccess` type already exists — it was designed for this.
**Sources:** `engine/api/.../PlaybookExecutionResult.java` — `PartialSuccess` variant exists, never produced; `engine/flow/FlowWorkerFunctionHandler.java:96` — step results available
**Exploration:** quick
**Status:** captured

## D4: Triage orchestration architecture

**Choice:** Implement the engine's `RecoveryCoordinator` SPI in life. When any playbook worker exhausts retries, the engine calls `LifePlaybookRecoveryCoordinator.handleFailure(RecoveryContext)`. The coordinator re-invokes the planning worker with failure context.
**Alternatives:**
- CDI event observer — observe `WorkerRetriesExhaustedEvent`. But the event fires after `RecoveryCoordinator` returns false — it's the fallback path. Using it means fighting the engine's control flow.
- Pre-emptive wrapper — wrap playbook workers in `TriageWorkerFunction`. Self-contained but hides retry logic from engine event log and `triageRound` mechanism.
**Rationale:** `RecoveryCoordinator` was designed for this exact scenario. The SPI gives life access to `RecoveryContext` with `PlaybookExecutionResult`, the attempt count, and the worker identity. The `triageRound` mechanism gives re-triaged workers fresh retry budgets. All existing infrastructure composes naturally.
**Trade-offs:** Life takes a dependency on `RecoveryCoordinator` (engine SPI). This is a stable API — it's a simple boolean-returning interface. The SPI is in `engine-common-core`, which life already depends on.
**Sources:** `engine/common-core/.../RecoveryCoordinator.java`, `engine/common-core/.../RetryOrchestrator.java:128`
**Exploration:** deep-analysis
**Status:** captured

## D5: Triage decision model

**Choice:** 4-action structured `TriageDecision` response schema: `RETRY_MODIFIED`, `SWITCH_PLAYBOOK`, `GENERATE_FIX`, `FALLBACK_MCP`.
**Alternatives:**
- 3-action set — no `SWITCH_PLAYBOOK`. Simpler but can't leverage existing templates.
- 5-action set — adds `SKIP_WORKER`. More nuanced but the coordinator already handles worker skip via `FALLBACK_MCP` + worker activation params.
**Rationale:** Each action maps to a concrete coordinator operation. `RETRY_MODIFIED` = same template, adjusted params. `SWITCH_PLAYBOOK` = different template from catalog. `GENERATE_FIX` = new YAML, validated, persisted. `FALLBACK_MCP` = set `skipPlaybook` flag, domain worker uses MCP tools. The four actions cover every scenario from the issue.
**Trade-offs:** If a genuinely new action is needed later, the enum must be extended. Acceptable — the sealed schema is testable and auditable.
**Sources:** Issue #141 requirements; `api/.../CoordinationPlan.java` — planning worker's initial output schema
**Exploration:** quick
**Status:** captured

## D6: CBR integration model

**Choice:** Enrich existing `PlanCbrCase` with playbook-specific features (templateName, failedSteps, triageAction, connectorErrors). Per-failure records written immediately when triage completes.
**Alternatives:**
- Distinct `PlaybookCbrCase` type — clean separation but requires new feature schemas, similarity specs, and a separate retrieval path. Duplicates infrastructure.
**Rationale:** The existing CBR infrastructure (feature schemas, similarity specs, retrieval, adaptation rules) already handles domain-level features. Adding playbook features enriches the existing signal. The adaptation rules can naturally incorporate "last time with this contractor, the standard playbook failed."
**Trade-offs:** Feature schemas become larger per domain. Acceptable — the JQ extractors keep the schema modular.
**Sources:** `life/app/cbr/LifeCbrFeatureSchemaRegistrar.java`, `life/app/cbr/LifeCaseOutcomeCbrWriter.java`
**Exploration:** quick
**Status:** captured
