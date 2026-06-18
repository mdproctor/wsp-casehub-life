# Handoff — 2026-06-18
life#25 closed — first real LLM-backed worker (AgentExec + OpenClaw /v1/chat/completions). AgentDescriptor convention established. CDI test pattern fixed (engine#536 filed). Two new engine issues filed (engine#536, engine#537).

## Last Session

Implemented `WorkerFunction.AgentExec(Agent)` for `book-appointment-agent` in `AppointmentCycleCaseHub`. The key discovery: `AgentExec` and `WorkerProvisioner` are completely different engine execution paths — cannot be substituted. Life#25 uses OpenClaw as an LLM server (`/v1/chat/completions`), not the WorkerProvisioner heartbeat path. Full Layer 7 (real skills: calendar, Home Assistant, banking APIs) requires separate work via `life#37` (WorkerProvisioner) or `life#38` (`/hooks/agent` bridge).

Major CDI testing fix: `@InjectMock` on `@ApplicationScoped` beans causes Quarkus CDI restart → `BlackboardEventCodecRegistrar` double-registers Vert.x codecs → all subsequent `@QuarkusTest` classes fail. Fix: `@Alternative @Priority(10)` CDI test bean registered in `selected-alternatives`. This also unmasked `CaseContextChangedEventHandler` running JPA on IO thread (engine#537).

Build verification: unit tests pass (3/3). Integration tests fail due to pre-existing `CaseContextChangedEventHandler` IO thread violation (engine#537). `ShowcaseScenarioTest` fails with pre-existing optimistic lock race (unrelated).

## Immediate Next Step

Fix `CaseContextChangedEventHandler` IO thread violation in engine (engine#537) — once that lands the AppointmentCycle integration tests should go green. OR start life#37 (WorkerProvisioner wiring) / life#38 (/hooks/agent bridge) for real Layer 7. Either is unblocked.

## What's Left

- life#37 — Layer 7 (full): wire `OpenClawWorkerProvisioner` — heartbeat mode · L · High
- life#38 — Layer 7: `/hooks/agent` direct-call integration — real skills · L · High
- life#26 — RBAC-differentiated risk thresholds (blocked on `parent#251` — auth retrofit) · M · Med
- life#36 — fix `LifeCaseResourceTest` NoSuchMethodError — `CaseHubRuntime.signal()` engine SNAPSHOT · S · Low
- engine#536 — `BlackboardEventCodecRegistrar` idempotency (blocks all @QuarkusTest) · S · Low
- engine#537 — `CaseContextChangedEventHandler` @Blocking annotation (blocks AppointmentCycle integration tests) · S · Low
- casehubio/engine#527 — add `baseUrl` to `OpenAiChatModelProvider` (deletes `LifeOpenClawChatModelProvider`) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#536 | BlackboardEventCodecRegistrar idempotency | S | Low | Unblocks all @QuarkusTest |
| engine#537 | CaseContextChangedEventHandler @Blocking | S | Low | Unblocks AppointmentCycle tests |
| #38 | /hooks/agent direct-call bridge for Layer 7 | L | High | Real OpenClaw skills |
| #37 | WorkerProvisioner heartbeat mode | L | High | Full Layer 7 |

## References

- Blog: `blog/2026-06-18-mdp01-first-real-worker.md`
- Design spec: `docs/specs/2026-06-17-openclaw-agent-worker-design.md` (rev 5)
- Protocol: `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`
- Garden: GE-20260618-248ce7, GE-20260618-c552c3, GE-20260618-a7a383, GE-20260618-5008f5, GE-20260618-8526c8, GE-20260618-fe7c8e
