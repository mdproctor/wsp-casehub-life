# Decisions — Issue #140: Playbook Coordination

## D1: Playbook format

**Choice:** Platform YAML — LLM generates standard `.playbook.yaml` files executed by the existing platform runtime
**Alternatives:**
- Simplified life-specific format — simpler for LLM to generate but doesn't benefit from platform runtime features
- Tiered (life-specific for ad-hoc, platform YAML for curated) — two runtimes, more complexity
**Rationale:** Leverages all existing platform playbook infrastructure (graphql:, concurrent:, forEach:, etc.). Single format, single runtime. Higher bar for LLM generation correctness but the schema is well-defined.
**Trade-offs:** LLM must generate valid platform YAML — generation errors surface at runtime
**Sources:** Platform playbook examples (playbooks/data-delivery/graphql-operation.playbook.yaml), platform#562, #563
**Exploration:** quick
**Status:** captured

## D2: Case lifecycle integration

**Choice:** Case-internal steps — playbooks are a new step type in YAML case definitions, orchestrated by the engine alongside workers
**Alternatives:**
- External orchestration layer — LifePlaybookOrchestrator wraps case lifecycle, decoupled from engine
- Hybrid (pre as orchestration, post as case step) — different integration points for different phases
**Rationale:** Keeps everything visible in the case definition. Leverages existing engine lifecycle, signals, and milestone tracking. No external layer to coordinate.
**Trade-offs:** Requires engine changes for new PlanItem type
**Sources:** Existing YAML case definitions in app/src/main/resources/life/
**Exploration:** quick
**Status:** captured

## D3: LLM planning trigger

**Choice:** Dedicated planning worker — a "coordinator" AgentExec worker runs as the first step in every case, selects/generates playbooks, writes coordination plan to context
**Alternatives:**
- LifeCaseService.startCase() phase — planning in Java during case start, couples to service layer
- Pre-case playbook — separate planning playbook before main case, adds two-phase startup
**Rationale:** Fits existing worker pattern (AgentExec with planning prompt). Keeps planning inside the case definition. Transparent — visible as a step like any other worker.
**Trade-offs:** Adds an LLM call at case start even when playbook selection could be deterministic
**Sources:** LifeTypedCaseHub.agentWorker() pattern, CbrInputTransformer for context injection
**Exploration:** quick
**Status:** captured

## D4: Playbook registry

**Choice:** Classpath YAML + runtime store — curated playbooks as .playbook.yaml on classpath, LLM-generated playbooks in a runtime store (database). Registry SPI merges both sources.
**Alternatives:**
- All runtime (database-backed) — loses code-as-source-of-truth for curated playbooks
- CBR-only (playbooks as cases) — conflates exact match retrieval with similarity search
**Rationale:** Curated playbooks are code artifacts (version controlled, reviewed, tested). Generated playbooks are runtime artifacts (evolve with use). Registry SPI provides a unified query surface.
**Trade-offs:** Two storage backends to manage; promotion path (generated → curated) needs a workflow
**Sources:** Existing case definition classpath loading pattern (life/*.yaml), platform playbook-catalog#617
**Exploration:** quick
**Status:** captured

## D5: Post-processing model

**Choice:** Separate playbook — post-processing is itself a playbook, selected as a pre/post pair by the planner
**Alternatives:**
- Inline case steps — hardcoded final milestones in case definition, no dynamic selection
- Planner-generated inline steps — dynamic but modifying running case definition is complex
**Rationale:** Reusable — same post-processing playbook shared across case types. Follows the same registry/selection pattern as pre-processing. Clean separation of concerns.
**Trade-offs:** Planner must select coherent pre/post pairs; bad pairing could produce gaps
**Sources:** Issue #140 example: store completion report → email summary → update trust score
**Exploration:** quick
**Status:** captured

## D6: Plan output format

**Choice:** Structured plan in case context — planning worker writes a CoordinationPlan (JSON) to case context, downstream steps read from it
**Alternatives:**
- Engine signals per step — event-driven but harder to trace full coordination plan
- Dynamic case augmentation — most flexible but modifying running case is complex
**Rationale:** Simple, transparent, debuggable. Subsequent PlaybookStep entries use guard conditions that evaluate context. Well-established engine pattern.
**Trade-offs:** Context payload grows; plan is a snapshot (no mid-case re-planning without a second planner invocation)
**Sources:** Existing case context pattern (cbrCalibration, adaptedPlan)
**Exploration:** quick
**Status:** captured

## D7: CBR integration

**Choice:** Planning worker queries CBR — receives CBR context via existing CbrInputTransformer, uses it to inform playbook selection
**Alternatives:**
- Extend CBR adaptation rules — add playbook selection as new dimension in LifePlanAdapter, couples two concerns
**Rationale:** No new CBR infrastructure needed. CBR provides data ("past cases used X playbook"), LLM does reasoning. CbrInputTransformer already wired to all workers.
**Trade-offs:** Playbook selection quality depends on LLM reasoning about CBR data, not structured retrieval
**Sources:** LifeCbrSuggestionService, CbrInputTransformer, LifePlanAdapter
**Exploration:** quick
**Status:** captured

## D8: Platform vs. life ownership

**Choice:** Platform primitives + life wiring — PlaybookRegistry SPI and PlaybookStep in platform (casehub-engine). Life provides curated playbooks, planning worker, registry population, domain-specific wiring.
**Alternatives:**
- All in life — faster iteration but violates layering rule and re-implements platform primitives
- Split (registry in platform, execution in life) — execution benefits most from being platform-level
**Rationale:** Follows existing layering rule: if it's not domain-specific, it belongs in the foundation. PlaybookStep and PlaybookRegistry are generic coordination primitives.
**Trade-offs:** Requires platform issues and cross-repo coordination
**Sources:** PLATFORM.md boundary rules, layering rule in CLAUDE.md
**Exploration:** quick
**Status:** captured

## D9: Overall architecture

**Choice:** Approach A + C — PlaybookStep as engine PlanItem type, with planning worker writing CoordinationPlan to context that PlaybookStep guards evaluate
**Alternatives:**
- Playbook-as-Worker (reuse WorkerFunction) — semantically misleading, overloads capability tags
- Pure Approach A (without planning worker context) — case definition must statically determine playbooks
**Rationale:** Cleanest abstraction — playbooks are distinct from workers, engine knows the difference. Planning worker is a regular AgentExec that writes context. PlaybookStep guards consume that context for dynamic activation. Declarative case definition with dynamic selection.
**Trade-offs:** Case definitions list all possible playbooks (most won't run). Guard conditions add visual noise.
**Depends on:** D2 (case-internal steps), D3 (planning worker), D6 (structured plan in context)
**Sources:** Engine PlanItem types, guard condition patterns in existing case definitions
**Exploration:** quick
**Status:** captured
