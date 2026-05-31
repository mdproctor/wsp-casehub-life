# Handoff — casehub-life Layer 5 implemented, on branch issue-6-layer5-engine-workflows
2026-05-31

## Last Session

Layer 5 (casehub-engine) implemented: 8 case definitions (YAML + fluent DSL + FuncDSL
workers), LifeCaseTracker entity, LifeCaseService (three-phase), LifeCaseResource,
scope retrofit, observer adaptation. 67 unit/DSL tests pass. @QuarkusTest integration
tests blocked on engine#408 (stale SNAPSHOT API signatures). Two new protocols created
(case-definition-layers revision, worker-function-execution-model). 4 garden entries
submitted. Issues filed: engine#408, aml#45, aml#46, clinical#50, devtown#60, parent#119.

## Immediate Next Step

Resolve engine#408 — engine source needs compilation fixes in `QuartzWorkerExecutionJobListener`
(CaseLifecycleEvent 7→8 params) and `PlanItemStoreContractTest` (PlanItemSaveRequest 7→8 fields).
Then `mvn install -Dmaven.test.skip=true -f ../engine/pom.xml` to rebuild SNAPSHOT. After that,
re-enable @QuarkusTest integration tests and run the full code review.

## Cross-Module

**Blocked by:**
- `casehub-engine` — engine#408 compilation errors prevent SNAPSHOT rebuild · S · Low

## What's Left

- Code review (superpowers:requesting-code-review) — deferred until engine#408 resolved · M · Med
- LAYER-LOG.md Layer 5 entry — write after code review + merge · S · Low
- `parent#114` — sync `docs/repos/casehub-life.md` for Layers 3–5 · XS · Low
- `parent#96` — superseded by parent#114 · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #11 | Trust dimension score fields on ExternalActor | S | Med | Blocked by Layer 6 |
| #7 | Layer 6: Trust routing | L | High | Blocked by Layer 5 merge |
| #20 | Explore ActionRiskClassifier oversight gate | M | High | Research — no blocker |

## References

- Spec: `docs/specs/2026-05-31-layer5-casehub-engine-design.md`
- Plan: `plans/2026-05-31-layer5-casehub-engine.md`
- Garden: GE-20260531-1e51d4, GE-20260531-4e21c1, GE-20260531-d896bf, GE-20260531-e5a1aa
- Protocols: PP-20260518 (revised), PP-20260531-worker-func-exec (new)
