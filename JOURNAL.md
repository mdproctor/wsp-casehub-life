# Design Journal — issue-143-avatar-integration

## 2026-11-02 — Brainstorming and plan complete

Full brainstorming cycle for P1.4 (avatar integration). Key outcomes:

**Decisions (9):** Full CognitionCore, single AgentProvider, expandable
panel, server-side sessions, tool-enabled agent with MCP, pages transport,
full state snapshot, dynamic identity, direct composition architecture.

**Reviews surfaced critical issues:**
- CognitionCore artifacts not published to Maven — phased delivery
  (Phase A without CognitionCore, Phase B when available)
- WebSocket transport wrong for this use case — POST+SSE matches
  existing LifeEventSseResource pattern
- AgentSession.clear() default is no-op — dirty flag + context
  update message approach instead
- RoutingAgentProvider already supports openSession() — no new
  provider needed, just add casehub-platform-agent-claude dependency
- @RolesAllowed must include all three household roles
- Avatar actions need `delegatedVia: "avatar"` audit marker

**Plan:** 4 batches, 8 tasks covering API types, backend wiring,
REST endpoint, and frontend conversation panel.

**Spec review:** 3 rounds, 25 issues raised, 20 verified, 2 accepted.
