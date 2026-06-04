# Handoff — casehub-life L5 integration tests re-enabled, on main
2026-06-04

## Last Session

Re-enabled Layer 5 integration tests (life#23). The never-green
AppointmentCycleIntegrationTest required three fixes beyond engine#410:
remove WAITING assertion (engine stays RUNNING during humanTasks),
QuarkusTransaction.requiringNew() for Panache in Awaitility,
callerRef WorkItem filtering for test isolation. 233/233 pass.

## Immediate Next Step

`/work start 20` — Explore ActionRiskClassifier oversight gate
(research/design). Read `docs/specs/life-actor-model.md` for the
actor model and `docs/specs/2026-06-03-layer6-trust-routing.md`
for the trust routing context that #20 builds on.

## What's Left

- `parent#148` — clarify trust-maturity-model protocol (thresholds in code vs YAML) · XS · Low
- `parent#153` — add engine-ledger + platform-config to Cross-Repo Dependency Map · XS · Low
- `life#22` — minor code review findings (test filters, FQ types, YAML duplication) · XS · Low
- `life#24` — integration tests for remaining 7 case definitions · M · Med
- `life#25` — migrate CaseHub workers from Worker.builder to FuncWorkflowBuilder · M · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #20 | Explore ActionRiskClassifier oversight gate | M | High | Research/design — builds on Layer 6 trust routing |
| — | Layer 6 ARC42STORIES.MD §9.4 entry | S | Low | Write before closing Layer 6 fully |
| — | Layer 7: OpenClaw as WorkerProvisioner | L | High | Next layer — makes trust routing decisions consequential |

## References

- Blog: `blog/2026-06-04-mdp01-tests-never-green.md`
- Spec: `docs/specs/2026-06-04-l5-integration-tests.md`
- Garden: GE-20260604-97031b (WorkItem test isolation), GE-20260604-38e09e (WAITING undocumented)
