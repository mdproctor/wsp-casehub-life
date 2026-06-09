# Handoff — 2026-06-09
life#32 closed, test suite restored, engine#463 filed

## Last Session

Restored 277/277 test suite (life#32): `LifeTestFixtures` was missing `tenancyId` on `WorkItemTemplate` — V35 adds `NOT NULL` with no H2 default; Hibernate `drop-and-create` never applies the Flyway default. Also added `import-qhorus.sql` via `sql-load-script` to seed `ledger_subject_sequence` (non-JPA table, would fail when ledger code paths activate). Earlier in the session: questioned life#25 (FuncWorkflowBuilder migration for stub workers), decided stubs are exempt pending engine#463, rescoped the issue. PP-20260531-worker-func-exec updated in CLAUDE.md.

## Immediate Next Step

`CommitmentLifecycleScenarioTest.contractorCommitment_dispatchesCommandAndCreatesRecord` is a pre-existing isolation failure (passes in full suite, fails alone). File as a new issue and fix, or investigate engine#463 brainstorming on the engine side.

## What's Left

- `life#25` — apply function-as-worker abstraction to first real OpenClaw workers (blocked on engine#463) · M · Med
- `life#26` — RBAC-differentiated risk thresholds (blocked on auth retrofit) · M · Med
- `engine#463` — design: first-class function-as-worker support (raw lambda vs FuncWorkflowBuilder gap) · M · High
- `engine#437` — clarify `GateRequired.scope` semantics (verify `"casehubio/life/oversight"` is correct)
- `CommitmentLifecycleScenarioTest` — isolation failure, passes in full suite, fails alone — unrelated to #32
- Branch `issue-2-layer1-naive-domain` — eligible for deletion (14 days, marked 2026-06-09)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | File and fix CommitmentLifecycleScenarioTest isolation bug | S | Low | Pre-existing; not blocking but messy |
| engine#463 | Design function-as-worker abstraction | M | High | Needs brainstorming — unblocks life#25 |
| #25 | Apply abstraction to first real OpenClaw workers | M | Med | Blocked on engine#463 |
| — | Layer 7 (full): OpenClaw as WorkerProvisioner | XL | High | Research spec at parent/docs/specs/2026-05-25-openclaw-casehub-integration.md |

## References

- Blog: `blog/2026-06-09-mdp02-wrong-cause-right-fix.md`
- Protocol: `docs/protocols/casehub-life/non-jpa-tables-sql-load-script.md`
- Garden: `GE-20260609-ef7dbe` (Flyway NOT NULL + DEFAULT not applied in H2 drop-and-create)
