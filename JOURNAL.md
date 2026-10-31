# Design Journal — issue-142-calendar-conflict-detection

## 2026-10-31 — Design session: calendar conflict detection

### What happened

Designed the calendar conflict detection feature (#142) — the inbound
sync pipeline that detects when CaseHub-managed calendar events are
edited or deleted externally in Google Calendar.

**Design flow:** brainstorming (11 decisions) → light decision review
(12 issues, D5 revised from hash to stored field values, D10/D11 added)
→ spec writing → standard spec review (3 rounds, 16 issues, 15 verified
— key improvements: EventTiming as JSON TEXT, outbound push auto-dismiss,
@Version optimistic locking, FK-safe mapping deletion) → implementation
plan (12 tasks, 4 batches).

### Key decisions

- **Detect + notify** — CaseHub stays authoritative, external edits are
  signals requiring explicit human acceptance
- **Stored field values** (not hash) — avoids provider normalization
  false positives. Existing `summary` field doubles as pushed baseline
- **EventTiming as JSON TEXT** — matches the sealed hierarchy (AllDay/Timed)
  instead of flat timestamp columns
- **Outbound push auto-dismisses stale conflicts** — critical race
  condition caught by spec review
- **Single calendar for now** — data model multi-calendar ready, pipeline
  assumes one

### What's next

Implementation: 12 tasks across 4 batches (Data Model → Sync Pipeline →
Resolution+Audit → UI). Plan at `plans/2026-10-31-calendar-conflict-detection.md`.
