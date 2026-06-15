# Handoff — 2026-06-14
OpenClaw Testcontainers pattern proven — test infrastructure ready for Layer 7

*Updated: engine#437 closed — removed from backlog.*

## Last Session

Research session: explored OpenClaw worker architecture and proved the Docker/Testcontainers testing pattern. Mini project at `/tmp/openclaw-isolation-test/` passes 3/3 tests in 8s. Key findings: correct endpoint is `POST /v1/chat/completions` with `model: "openclaw"` (not `/api/sessions/main/messages` which returns 404); `execInContainer` required for HTTP calls (Podman custom-network host port mapping broken); `contextWindow: 200000` needed for mock model. 7 forage entries submitted. ARC42STORIES trust-routing stale blocker cleared.

## Immediate Next Step

Start engine#463 in the engine session — the function-as-worker abstraction design. This unblocks the OpenClaw WorkerProvisioner work (casehub-openclaw) and then life#25.

## What's Left

- `life#25` — apply function-as-worker abstraction to first real OpenClaw workers (blocked on engine#463) · M · Med
- `life#26` — RBAC-differentiated risk thresholds (blocked on parent#251 — auth retrofit epic) · M · Med
- `engine#463` — design: first-class function-as-worker support (raw lambda vs FuncWorkflowBuilder gap) · M · High
- Branch `issue-2-layer1-naive-domain` — past deletion date (marked 2026-06-09, now 5 days over) — awaiting explicit YES before deleting
- Branch `issue-16-17-18-cleanup` — past deletion date (marked 2026-06-12, now 2 days over) — awaiting explicit YES before deleting

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#463 | Design function-as-worker abstraction | M | High | Critical path — unblocks everything below |
| — | casehub-openclaw WorkerProvisioner impl | L | High | Blocked on engine#463; lives in casehub-openclaw repo |
| #25 | Wire OpenClaw workers into casehub-life | M | Med | Blocked on engine#463 + casehub-openclaw |
| — | Layer 7 (full): OpenClaw as WorkerProvisioner | XL | High | Research spec at parent/docs/specs/2026-05-25-openclaw-casehub-integration.md |

## References

- Blog: `blog/2026-06-14-mdp01-eight-seconds.md`
- Mini project: `/tmp/openclaw-isolation-test/` (Testcontainers proof — passes 3/3)
- Garden: 6 new entries + revise GE-20260520-c0e5b4 (SSH tunnel alternative)
