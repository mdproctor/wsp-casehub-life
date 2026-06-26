# Handoff — 2026-06-26

*Updated: engine#527 closed — removed from backlog.*

life#38 closed — all 32 workers across 7 YamlCaseHubs converted from stubs to real OpenClaw agents via `/hooks/agent` direct-call bridge. life#45 fixed (qhorus ACL enforcement). Engine API migration (engine#543/567) landed.

## Last Session

Designed and implemented the direct-call bridge architecture: `LifeOpenClawChatModelFactory` → `OpenClawAgentProvider` → `DirectCallBridge` → `/hooks/agent`. Phase 1 (bridge infrastructure in casehub-openclaw#49) was built by the openclaw session. Phase 2 (life consumption + 32 worker conversions) was implemented here across 6 tasks. Hit engine SNAPSHOT breaking changes (Worker/Capability moved to worker-api, AgentDescriptor moved to CaseDefinition) — migrated all code. Also fixed qhorus ACL enforcement change (life#45). Final code review clean (no Critical/Important findings).

## Immediate Next Step

Start life#37 (WorkerProvisioner heartbeat mode) — the second half of the two-mode architecture. casehub-openclaw-casehub already has `OpenClawWorkerProvisioner`. Life needs to wire it for capabilities without matching workers (ambient monitoring use cases).

Run `/work` when ready.

## What's Left

- life#37 — Layer 7 (full): wire OpenClawWorkerProvisioner — heartbeat mode · L · High
- life#46 — refactor: extract shared AgentDescriptor factory methods · S · Low
- casehubio/engine#569 — convenience: make AgentBuilder.model(ChatModel) public · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| life#37 | WorkerProvisioner heartbeat mode | L | High | Phase 2 of two-mode architecture |
| life#46 | Extract shared AgentDescriptor methods | S | Low | Tech debt from life#38 review |

## References

- Spec: `specs/2026-06-24-hooks-agent-direct-call-design.md` (rev 5)
- Plans: `plans/2026-06-25-phase1-openclaw-direct-call-bridge.md`, `plans/2026-06-25-phase2-life-worker-conversions.md`
- Garden: GE-20260626-a37306 (CDI transitive activation), GE-20260626-0e976f (test factory behavioral intent), GE-20260624-3324b6 (WorkerFunction cast — revised)
- Issues filed: casehubio/openclaw#49 (bridge), casehubio/engine#569 (AgentBuilder convenience), life#46 (descriptor dedup)
