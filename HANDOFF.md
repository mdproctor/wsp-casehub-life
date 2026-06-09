# Handoff — 2026-06-09
Test suite fully green: life#32 + life#33 closed

## Last Session

Two fixes landed. life#32: `LifeTestFixtures` missing `tenancyId` (V35 NOT NULL, no H2 default) + `ledger_subject_sequence` missing from H2 (now seeded via `import-qhorus.sql`). life#33: contractor actor channel name `life/actor/{uuid}` fails `ChannelSlugValidator` when UUID starts with a digit (62.5% of UUIDs) — prefixed with `ext-`. Both CLAUDE.md and protocols updated. 277/277, no flakes.

## Immediate Next Step

No immediate bugs. Discretionary: start engine#463 brainstorming (function-as-worker design, unblocks life#25), or tackle life#26 (RBAC risk thresholds, blocked on auth retrofit).

## What's Left

- `life#25` — apply function-as-worker abstraction to first real OpenClaw workers (blocked on engine#463) · M · Med
- `life#26` — RBAC-differentiated risk thresholds (blocked on auth retrofit) · M · Med
- `engine#463` — design: first-class function-as-worker support (raw lambda vs FuncWorkflowBuilder gap) · M · High
- `engine#437` — clarify `GateRequired.scope` semantics (verify `"casehubio/life/oversight"` is correct)
- Branch `issue-2-layer1-naive-domain` — eligible for deletion (14 days, marked 2026-06-09)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#463 | Design function-as-worker abstraction | M | High | Needs brainstorming — unblocks life#25 |
| #25 | Apply abstraction to first real OpenClaw workers | M | Med | Blocked on engine#463 |
| — | Layer 7 (full): OpenClaw as WorkerProvisioner | XL | High | Research spec at parent/docs/specs/2026-05-25-openclaw-casehub-integration.md |

## References

- Blog: `blog/2026-06-09-mdp03-sixty-two-percent.md`
- Garden: `GE-20260607-a4d78a` revised — UUID leading digit variant added
- Protocol: `docs/protocols/casehub-life/actor-channel-name-prefix.md` (PP-20260609-982617)
