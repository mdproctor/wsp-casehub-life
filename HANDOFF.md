# Handoff — 2026-06-19
*Updated: engine#536, engine#537 closed — removed from backlog.*

life#25 closed — first real LLM-backed worker (AgentExec + OpenClaw /v1/chat/completions). AgentDescriptor convention established. CDI test pattern fixed. engine#536 and engine#537 landed upstream — @QuarkusTest and AppointmentCycle integration tests now unblocked.

## Last Session

Implemented `WorkerFunction.AgentExec(Agent)` for `book-appointment-agent` in `AppointmentCycleCaseHub`. The key discovery: `AgentExec` and `WorkerProvisioner` are completely different engine execution paths — cannot be substituted. Life#25 uses OpenClaw as an LLM server (`/v1/chat/completions`), not the WorkerProvisioner heartbeat path. Full Layer 7 (real skills: calendar, Home Assistant, banking APIs) requires separate work via `life#37` (WorkerProvisioner) or `life#38` (`/hooks/agent` bridge).

CDI test pattern fixed: `@Alternative @Priority(10)` CDI test bean replaces `@InjectMock` to avoid Quarkus CDI restart → Vert.x codec double-registration.

## Immediate Next Step

engine#536 and engine#537 are closed. Pull latest engine SNAPSHOT, then run the AppointmentCycle integration tests to confirm they're green. If green, start life#38 (`/hooks/agent` direct-call bridge) or life#36 (fix `LifeCaseResourceTest` NoSuchMethodError — quick S/Low).

## What's Left

- life#38 — Layer 7: `/hooks/agent` direct-call integration — real skills · L · High
- life#37 — Layer 7 (full): wire `OpenClawWorkerProvisioner` — heartbeat mode · L · High
- life#36 — fix `LifeCaseResourceTest` NoSuchMethodError — `CaseHubRuntime.signal()` engine SNAPSHOT · S · Low
- life#26 — RBAC-differentiated risk thresholds (blocked on `parent#251` — auth retrofit) · M · Med
- casehubio/engine#527 — add `baseUrl` to `OpenAiChatModelProvider` (deletes `LifeOpenClawChatModelProvider`) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| life#36 | Fix LifeCaseResourceTest NoSuchMethodError | S | Low | Quick unblock — engine SNAPSHOT |
| life#38 | /hooks/agent direct-call bridge for Layer 7 | L | High | Real OpenClaw skills |
| life#37 | WorkerProvisioner heartbeat mode | L | High | Full Layer 7 |
| life#26 | RBAC-differentiated risk thresholds | M | Med | Blocked on parent#251 (auth) |

## References

- Blog: `blog/2026-06-18-mdp01-first-real-worker.md`
- Design spec: `docs/specs/2026-06-17-openclaw-agent-worker-design.md` (rev 5)
- Protocol: `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`
- Garden: GE-20260618-248ce7, GE-20260618-c552c3, GE-20260618-a7a383, GE-20260618-5008f5, GE-20260618-8526c8, GE-20260618-fe7c8e
