# Handoff — 2026-06-16
Layer 6 epic (#7) closed — validation found two test bugs; ARC42 §9.4 written; #36 filed for engine regression.

*Updated: engine#463 closed — removed from backlog.*

## Last Session

Closed Layer 6 (trust routing) properly. Pre-closure validation found two bugs: (1) `JpaActorTrustScoreRepository` missing from `selected-alternatives` — `NoOpActorTrustScoreRepository` was silently discarding all trust score writes, making `ExternalActorTrustEnrichmentTest` a permanent false-pass; (2) `ledger_subject_sequence` missing `tenancy_id` column in `import-qhorus.sql` after ledger SNAPSHOT update — HTTP 500 on all ledger-writing integration tests. Both fixed, committed, CLAUDE.md updated. `LifeCaseResourceTest` has a pre-existing `NoSuchMethodError` on `CaseHubRuntime.signal()` (engine SNAPSHOT API change) — filed as #36. ARC42STORIES.MD §9.4 entry written for Layer 6. Issue #7 closed.

## Immediate Next Step

`engine#463` is now closed — function-as-worker abstraction is designed. Start Layer 7: either fix `life#36` first (S/Low — engine API test regression), or read `engine#463` design and begin OpenClaw WorkerProvisioner implementation path.

## What's Left

- `life#25` — apply function-as-worker abstraction to first real OpenClaw workers (blocked on casehub-openclaw WorkerProvisioner impl) · M · Med
- `life#26` — RBAC-differentiated risk thresholds (blocked on `parent#251` — auth retrofit) · M · Med
- `life#36` — fix `LifeCaseResourceTest` NoSuchMethodError — `CaseHubRuntime.signal()` engine SNAPSHOT API change · S · Low
- 8 branches with closed issues but no `chore: branch closed` stamp — awaiting explicit YES per branch before deletion (see epic hygiene note in session)
- Branch `issue-2-layer1-naive-domain` — 3+ weeks past, explicit YES still required
- Branch `issue-16-17-18-cleanup` — 3+ weeks past, explicit YES still required

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | casehub-openclaw WorkerProvisioner impl | L | High | Blocked on casehub-openclaw repo work |
| #25 | Wire OpenClaw workers into casehub-life | M | Med | Blocked on casehub-openclaw |
| — | Layer 7 (full): OpenClaw as WorkerProvisioner | XL | High | Research spec at parent/docs/specs/2026-05-25-openclaw-casehub-integration.md |

## References

- Blog: `blog/2026-06-16-mdp02-closing-layer-6.md`
- ARC42 Layer 6 §9.4: `ARC42STORIES.MD` — "Layer — Trust routing" (completed 2026-06-16)
- Garden: GE-20260616-036128, GE-20260616-17187e, GE-20260616-90a867, GE-20260616-101fc0
- Issue filed: `casehubio/life#36` (LifeCaseResourceTest engine API regression)
