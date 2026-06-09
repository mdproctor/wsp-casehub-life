# Handoff — casehub-life #20 closed, Layer 7 partial
2026-06-07

*Updated: #27 closed — removed from backlog.*

## Last Session

Implemented `LifeActionRiskClassifier` (life#20) — household agents now pause before consequential actions and route to the `life/oversight` channel. Fixed two engine#402 migration breaks mid-session: `WorkerResult.of()` for FuncDSL `function()` lambdas (7 case hub files), and `ListEvaluator.contains()` removal from `HumanTaskTarget.candidateGroups()` (10 test files, life#29). Branch closed: 9 commits on main, pushed to both fork and upstream.

## Immediate Next Step

No clear next feature — life#25 is now blocked. Options: investigate `LEDGER_SUBJECT_SEQUENCE` H2 gap (8 failing tests), or start engine#463 brainstorming on the engine side.

## What's Left

- `life#25` — worker migration (blocked on engine#463 — function-as-worker design) · M · Med
- `life#26` — RBAC-differentiated risk thresholds (blocked on auth retrofit) · M · Med
- engine#437 — clarify `GateRequired.scope` semantics (verify `"casehubio/life/oversight"` is correct)
- Pre-existing: `LEDGER_SUBJECT_SEQUENCE` table missing in H2 — 8 test classes fail on every run
- Backup branch `backup/pre-squash-main-20260607` — can delete after 14 days (eligible 2026-06-21)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #25 | Worker migration — scope TBD | M | Med | Blocked on engine#463 design |
| — | Fix H2 `LEDGER_SUBJECT_SEQUENCE` gap | S | Low | 8 test classes affected; likely missing Flyway migration |
| — | Layer 7 (full): OpenClaw as WorkerProvisioner | XL | High | Research spec at parent/docs/specs/2026-05-25-openclaw-casehub-integration.md |

## Cleaned up

- `#27 — one-class-per-rule pattern for LifeActionRiskClassifier` — closed, removed from backlog

## References

- Blog: `blog/2026-06-07-mdp01-fewer-approvers-stricter-gate.md`
- Spec: `docs/specs/2026-06-07-action-risk-classifier-design.md`
- Protocols: `docs/protocols/casehub-life/INDEX.md`
