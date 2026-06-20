# Handoff — 2026-06-20
*Updated: PR #39 merged — removed from backlog.*

life#36 closed — @QuarkusTest suite fully green (286 tests, 0 skipped). No code change in life needed; engine SNAPSHOT already had the restored signal() signature. Stale CLAUDE.md status warning removed. PR #39 merged.

## Last Session

Picked up the two unblocked engine issues (engine#536, engine#537 both fixed). Expected to need to update `CaseHubRuntime.signal()` call site in `LifeCaseService` — checking `javap` on the installed SNAPSHOT jar showed the method was already present with the correct signature. `LifeCaseResourceTest` passed first try with no changes. Full suite: 286 tests, 0 failures, 0 skipped across all 52 test classes including all 8 `*IntegrationTest` classes.

Cleaned up CLAUDE.md (removed stale @QuarkusTest status row, engine#536 parenthetical refs) and ARC42STORIES.MD (improvement refs marked resolved). PR #39 created, CI green.

## Immediate Next Step

Start life#38 (`/hooks/agent` direct-call bridge) or life#37 (WorkerProvisioner) for Layer 7 full. PR #39 already merged.

Run `/work` when ready.

## What's Left

- life#38 — Layer 7: `/hooks/agent` direct-call integration — real skills · L · High
- life#37 — Layer 7 (full): wire `OpenClawWorkerProvisioner` — heartbeat mode · L · High
- life#26 — RBAC-differentiated risk thresholds (blocked on `parent#251` — auth retrofit) · M · Med
- casehubio/engine#527 — add `baseUrl` to `OpenAiChatModelProvider` (deletes `LifeOpenClawChatModelProvider`) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| life#38 | /hooks/agent direct-call bridge for Layer 7 | L | High | Real OpenClaw skills |
| life#37 | WorkerProvisioner heartbeat mode | L | High | Full Layer 7 |
| life#26 | RBAC-differentiated risk thresholds | M | Med | Blocked on parent#251 (auth) |

## References

- Blog: `blog/2026-06-19-mdp01-snapshot-already-moved.md`
- PR: casehubio/life#39
- Garden: GE-20260618-248ce7, GE-20260618-c552c3, GE-20260618-a7a383, GE-20260618-5008f5, GE-20260618-8526c8, GE-20260618-fe7c8e
