# HANDOFF — casehub-life

**Last updated:** 2026-11-01
**Branch:** main
**State:** Between issues — #142 closed, Phase 1 epic updated

---

## What Happened This Session

Implemented and closed #142 (calendar conflict detection) — the full
inbound sync pipeline from design through to merged code. Then analysed
#110 (conversational intake pane) from first principles, concluded it was
superseded by the avatar integration, and restructured the Phase 1 epic.

### #142 — Calendar conflict detection (closed)

Executed the 12-task implementation plan across 4 batches:

- **Data model** — LifeCalendarSyncState (sync token per calendar),
  LifeCalendarConflict (DETECTED/ACCEPTED/DISMISSED lifecycle),
  pushedTiming + @Version on mapping. V116, V117, V2102 migrations.
- **Sync pipeline** — Quartz-scheduled incremental sync via
  CalendarPlatform.listEventsSync(). ConflictDetector with echo
  filtering, idempotency guards, stored field value comparison.
- **Resolution** — Accept (update WorkItem deadline) or dismiss
  (push CaseHub state back). Auto-dismiss on outbound push.
  CalendarConflictLedgerEntry for tamper-evident audit.
- **REST + UI** — 6 endpoints on LifeCalendarResource. Calendar panel
  with conflict cards, diff view, accept/dismiss, count badge, SSE refresh.

Squashed to single commit, merged to main, pushed.
Blog entry: "External Edits Are Signals, Not Commands."

### Epic #139 restructuring

First-principles analysis of #110 (conversational intake pane). Conclusion:
with the playbook catalogue done (#140, #141) and the neocortex cognitive
agent planned, #110 would build a temporary LLM classifier that gets
replaced. The dispatch layer IS the playbook catalogue lookup.

- Closed #110 and #109 as superseded
- Created #143 (P1.4 — avatar integration)
- Created #144 (P1.5 — intent resolution via cognitive agent)
- Created #145 (P1.6 — avatar-driven UI)
- Updated epic #139 body — P1.1/P1.2/P1.3 marked complete, new issues linked

Also filed connectors#195 — investigation into generic conflict
reconciliation pattern in connectors-core (not ready to implement,
needs analysis of banking/IoT/commerce use cases first).

## Key Decisions

- #110 superseded — the cognitive agent (neocortex) handles intent
  classification, not a standalone LLM classifier
- NLP (#137) is cost optimisation for later, not the initial path
- Three execution tiers (D3) remain the dispatch model: direct MCP,
  playbook, engine case
- Generic reconciliation pattern (connectors#195) deferred until a
  second connector hits the same problem

## Next Action

Phase 1 remaining work is P1.4 → P1.5 → P1.6 (#143 → #144 → #145).
All depend on neocortex cognitive architecture being wirable into life.
Check blocks#311 epic status before starting.
