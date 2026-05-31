# Design Journal — issue-6-layer5-engine-workflows

## 2026-05-31 — Layer 5 implementation session

### Key design decisions

1. **All 8 case definitions in one layer** (7 + care-episode child). AML did 1, clinical did 3 — life does 8 to demonstrate the full breadth of engine capabilities in one harness. Split into Phase 1 (infrastructure + 3 core) and Phase 2 (4 advanced) for reviewability.

2. **YAML + fluent DSL pairing rule** — revised protocol PP-20260518 to establish both as equal authoring paths, not "runtime vs test." Every YAML gets a DSL companion. Worker functions use FuncWorkflowBuilder per new protocol PP-20260531.

3. **Scope retrofit** — changed WorkItem scope from `"life"` to `casehubio/life/{domain}` (hierarchical). This enabled LifeDecisionLedgerObserver to resolve domain from scope Path instead of requiring LifeTaskContext — engine-created WorkItems now produce ledger entries without supplements.

4. **SubCase M-of-N is DSL-only** — YAML schema doesn't support groupId/totalInGroup/requiredCount. Travel-plan's family-vote bindings added via Java augmentation. Filed for engine YAML schema expansion.

5. **Cross-case signals live in workers, not observers** — LifeCaseTrackerObserver is pure infrastructure (status update). Domain cross-case logic (contractor-to-financial-review signal) goes in the completing worker itself. Matches clinical pattern.

### Blockers

- **engine#408** — engine SNAPSHOT has stale API signatures (CaseMetaModelRepository.findByKey 3-to-4 params, PlanItemSaveRequest 7-to-8 fields). Can't rebuild from source (compilation errors in engine itself). @QuarkusTest integration tests blocked. 67 pure unit/DSL tests pass.

### Issues filed this session

- engine#408 — engine compilation errors blocking SNAPSHOT rebuild
- aml#45, aml#46 — fluent DSL companion + FuncDSL migration
- clinical#50 — fluent DSL companions for 3 case definitions
- devtown#60 — fluent DSL companions
- parent#119 — fix case-definition-layers protocol path in PLATFORM.md
