# Handoff — 2026-06-16
Layer 6 epic (#7) closed — validation found two test bugs; ARC42 §9.4 written; #36 filed for engine regression.

## Last Session

Closed Layer 6 (trust routing) properly. Pre-closure validation found two bugs: (1) `JpaActorTrustScoreRepository` missing from `selected-alternatives` — `NoOpActorTrustScoreRepository` was silently discarding all trust score writes, making `ExternalActorTrustEnrichmentTest` a permanent false-pass; (2) `ledger_subject_sequence` missing `tenancy_id` column in `import-qhorus.sql` after ledger SNAPSHOT update — HTTP 500 on all ledger-writing integration tests. Both fixed, committed, CLAUDE.md updated. `LifeCaseResourceTest` has a pre-existing `NoSuchMethodError` on `CaseHubRuntime.signal()` (engine SNAPSHOT API change) — filed as #36. ARC42STORIES.MD §9.4 entry written for Layer 6. Issue #7 closed.

## Immediate Next Step

Start `engine#463` in the engine session — function-as-worker abstraction design. This is the critical path before anything else moves.

## What's Left

- `life#25` — apply function-as-worker abstraction to first real OpenClaw workers (blocked on `engine#463`) · M · Med
- `life#26` — RBAC-differentiated risk thresholds (blocked on `parent#251` — auth retrofit) · M · Med
- `life#36` — fix `LifeCaseResourceTest` NoSuchMethodError — `CaseHubRuntime.signal()` engine SNAPSHOT API change · S · Low
- `engine#463` — design: first-class function-as-worker support · M · High
- 8 branches with closed issues but no `chore: branch closed` stamp — awaiting explicit YES per branch before deletion (see epic hygiene note in session)
- Branch `issue-2-layer1-naive-domain` — 3+ weeks past, explicit YES still required
- Branch `issue-16-17-18-cleanup` — 3+ weeks past, explicit YES still required

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

- Blog: `blog/2026-06-16-mdp02-closing-layer-6.md`
- ARC42 Layer 6 §9.4: `ARC42STORIES.MD` — "Layer — Trust routing" (completed 2026-06-16)
- Garden: GE-20260616-036128, GE-20260616-17187e, GE-20260616-90a867, GE-20260616-101fc0
- Issue filed: `casehubio/life#36` (LifeCaseResourceTest engine API regression)
