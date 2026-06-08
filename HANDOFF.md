# Handoff — casehub-life #20 closed, Layer 7 partial
2026-06-07

## Last Session

Implemented `LifeActionRiskClassifier` (life#20) — household agents now pause before consequential actions and route to the `life/oversight` channel. Fixed two engine#402 migration breaks mid-session: `WorkerResult.of()` for FuncDSL `function()` lambdas (7 case hub files), and `ListEvaluator.contains()` removal from `HumanTaskTarget.candidateGroups()` (10 test files, life#29). Branch closed: 9 commits on main, pushed to both fork and upstream.

## Immediate Next Step

Start life#27 — refactor `LifeActionRiskClassifier` to one-class-per-rule pattern (all domain properties of a household action type in one place). Design already clear from the brainstorm; implementation plan needed. Run `/work start 27`.

## What's Left

- `life#25` — migrate CaseHub workers from Worker.builder to FuncWorkflowBuilder · M · Low
- `life#26` — RBAC-differentiated risk thresholds (blocked on auth retrofit) · M · Med
- `life#27` — one-class-per-rule refactor of LifeActionRiskClassifier · M · Low
- engine#437 — clarify `GateRequired.scope` semantics (verify `"casehubio/life/oversight"` is correct)
- Pre-existing: `LEDGER_SUBJECT_SEQUENCE` table missing in H2 — 8 test classes fail on every run
- Backup branch `backup/pre-squash-main-20260607` — can delete after 14 days

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | One-class-per-rule refactor for LifeActionRiskClassifier | M | Low | Design clear; implement immediately |
| #25 | Migrate workers from Worker.builder to FuncWorkflowBuilder | M | Low | Mechanical migration, no design needed |
| — | Layer 7 (full): OpenClaw as WorkerProvisioner | XL | High | Research spec at parent/docs/specs/2026-05-25-openclaw-casehub-integration.md |

## References

- Blog: `blog/2026-06-07-mdp01-fewer-approvers-stricter-gate.md`
- Spec: `docs/specs/2026-06-07-action-risk-classifier-design.md`
- Protocols: `docs/protocols/casehub-life/INDEX.md`
