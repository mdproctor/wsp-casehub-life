# Calendar Conflict Detection — Design Spec

**Issue:** casehubio/life#142
**Branch:** issue-142-calendar-conflict-detection
**Date:** 2026-10-31
**Depends on:** #114 (calendar sync — outbound push + inbound read)

## Problem

CaseHub creates calendar events for WorkItems with deadlines (outbound push, #114). When a user edits or deletes one of these events directly in Google Calendar, CaseHub has no way to detect the change. The WorkItem deadline and the calendar event diverge silently.

## Solution

Add an inbound sync pipeline that detects external edits to CaseHub-managed calendar events, surfaces them as conflicts, and records their resolution in the tamper-evident ledger.

**Core semantics:** CaseHub is authoritative. External edits are signals, not commands. The WorkItem deadline stays unchanged until a human explicitly accepts the external change through CaseHub (D1).

## Scope

**In scope:**
- Incremental sync via `CalendarPlatform.listEventsSync()` with stored sync tokens
- Change detection by comparing sync results against stored pushed field values
- Conflict entity with lifecycle (DETECTED → ACCEPTED/DISMISSED)
- Resolution actions: accept (update WorkItem) or dismiss (push CaseHub state back)
- Tamper-evident ledger entries for conflict detection and resolution
- Scheduled polling (Quartz) + on-demand manual refresh
- UI: conflict indicators, diff display, resolution buttons in existing calendar panel
- Sync token expiry recovery (full re-sync without false conflicts)

**Out of scope (explicit):**
- Multi-calendar support — data model is ready (LifeCalendarSyncState per platformId/calendarId) but pipeline and UI assume single default calendar (D11)
- Push notifications / webhook triggers — requires CalendarPlatform SPI extension and publicly-accessible endpoint; polling is the primary trigger for this issue
- Domain-specific resolution strategies — all domains use detect+notify uniformly
- Calendar conflict escalation — no automatic escalation for unresolved conflicts; existing SLA breach policy handles deadline enforcement independently

## Architecture

### Data Flow

```
                    ┌─────────────────────────────────────────────┐
                    │              Inbound Sync Pipeline          │
                    │                                             │
  Quartz job ──────►│ LifeCalendarInboundSyncJob                  │
  (or manual        │   │                                         │
   refresh)         │   ▼                                         │
                    │ CalendarPlatform.listEventsSync()            │
                    │   │                                         │
                    │   ▼                                         │
                    │ LifeCalendarConflictDetector                 │
                    │   │  compare sync results vs pushed state   │
                    │   │  on LifeCalendarMapping                 │
                    │   │                                         │
                    │   ▼                                         │
                    │ CDI: CalendarConflictDetected event          │
                    │   │                                         │
                    │   ├──► LifeCalendarConflictLedgerObserver    │
                    │   │    (writes ledger entry)                │
                    │   │                                         │
                    │   └──► LifeEventBroadcaster                 │
                    │        (SSE to UI)                          │
                    └─────────────────────────────────────────────┘

  User resolves ──► POST /life-calendar/conflicts/{id}/resolve
                      │
                      ▼
                    LifeCalendarConflictService
                      │  accept: update WorkItem deadline
                      │  dismiss: push CaseHub state back to Google
                      │
                      ▼
                    CDI: CalendarConflictResolved event
                      │
                      ├──► LedgerObserver (resolution entry)
                      └──► SSE broadcast (clear conflict indicator)
```

### New Components

| Component | Module | Responsibility |
|-----------|--------|----------------|
| `LifeCalendarSyncState` | api/ (record), app/ (JPA entity) | Sync token persistence per calendar |
| `LifeCalendarSyncStateStore` | api/ (SPI), app/ (JPA impl) | CRUD for sync state |
| `LifeCalendarConflict` | api/ (record response), app/ (JPA entity) | Conflict with lifecycle |
| `LifeCalendarConflictStore` | api/ (SPI), app/ (JPA impl) | CRUD for conflicts |
| `LifeCalendarInboundSyncJob` | app/ | Quartz scheduled incremental sync |
| `LifeCalendarConflictDetector` | app/ | Compares sync results against pushed state |
| `LifeCalendarConflictService` | app/ | Accept/dismiss resolution logic |
| `LifeCalendarConflictResource` | app/ | REST endpoints for conflicts and manual sync |
| `LifeCalendarConflictLedgerObserver` | app/ | Writes ledger entries on detect/resolve |
| `CalendarConflictLedgerEntry` | app/ledger/ | JOINED LedgerEntry subclass (qhorus PU) |
| `CalendarConflictDetected` | app/ | CDI event record |
| `CalendarConflictResolved` | app/ | CDI event record |
| `CalendarConflictType` | api/ | Enum: MODIFIED, DELETED |
| `CalendarConflictStatus` | api/ | Enum: DETECTED, ACCEPTED, DISMISSED |

### Extended Components

| Component | Change |
|-----------|--------|
| `JpaLifeCalendarMapping` | Add `pushedSummary`, `pushedStartTime`, `pushedEndTime` columns |
| `LifeCalendarSyncService` | Populate pushed fields on every outbound create/update |
| `life-calendar-panel` | Conflict indicators, diff display, resolution buttons, refresh button |

## Data Model

### LifeCalendarSyncState (V116)

```sql
CREATE TABLE life_calendar_sync_state (
    id          UUID PRIMARY KEY,
    platform_id VARCHAR(50)  NOT NULL,
    calendar_id VARCHAR(255) NOT NULL,
    sync_token  VARCHAR(1024),
    last_sync_at TIMESTAMP,
    UNIQUE (platform_id, calendar_id)
);
```

One row per calendar being watched. `syncToken` is null before the first successful sync (triggers initial full sync).

**SPI — LifeCalendarSyncStateStore (api/):**
- `Optional<LifeCalendarSyncState> findByCalendar(String platformId, String calendarId)`
- `LifeCalendarSyncState save(LifeCalendarSyncState state)`

### LifeCalendarMapping Extension (V116)

```sql
ALTER TABLE life_calendar_mapping
    ADD COLUMN pushed_summary    VARCHAR(500),
    ADD COLUMN pushed_start_time TIMESTAMP,
    ADD COLUMN pushed_end_time   TIMESTAMP;
```

Populated by `LifeCalendarSyncService` on every outbound create/update. Null for mappings created before this migration (pre-migration state). The conflict detector skips mappings with null pushed fields — no baseline means no meaningful comparison. The next outbound push (from any WorkItem update or SLA change) establishes the baseline automatically.

### LifeCalendarConflict (V117)

```sql
CREATE TABLE life_calendar_conflict (
    id                UUID PRIMARY KEY,
    mapping_id        UUID NOT NULL REFERENCES life_calendar_mapping(id),
    conflict_type     VARCHAR(20) NOT NULL,  -- MODIFIED, DELETED
    detected_at       TIMESTAMP   NOT NULL,
    external_summary  VARCHAR(500),          -- null for DELETED
    external_start_time TIMESTAMP,           -- null for DELETED
    external_end_time   TIMESTAMP,           -- null for DELETED
    status            VARCHAR(20) NOT NULL,  -- DETECTED, ACCEPTED, DISMISSED
    resolved_at       TIMESTAMP,
    resolved_by       VARCHAR(255),
    domain            VARCHAR(50) NOT NULL
);

CREATE INDEX idx_calendar_conflict_status ON life_calendar_conflict(status);
CREATE INDEX idx_calendar_conflict_mapping ON life_calendar_conflict(mapping_id);
```

**SPI — LifeCalendarConflictStore (api/):**
- `LifeCalendarConflict save(LifeCalendarConflict conflict)`
- `Optional<LifeCalendarConflict> findById(UUID id)`
- `List<LifeCalendarConflict> findByMapping(UUID mappingId)`
- `List<LifeCalendarConflict> findUnresolved()`
- `long countUnresolved()`
- `Optional<LifeCalendarConflict> findUnresolvedByMapping(UUID mappingId)`

### CalendarConflictLedgerEntry (V2102, qhorus PU)

```sql
CREATE TABLE calendar_conflict_ledger_entry (
    id             UUID PRIMARY KEY REFERENCES ledger_entry(id),
    conflict_id    UUID         NOT NULL,
    conflict_type  VARCHAR(20)  NOT NULL,
    domain         VARCHAR(50)  NOT NULL,
    resolution     VARCHAR(20)  NOT NULL,  -- DETECTED, ACCEPTED, DISMISSED
    entity_type    VARCHAR(30)  NOT NULL,
    entity_id      VARCHAR(255) NOT NULL
);
```

JOINED inheritance from `LedgerEntry`. `@DiscriminatorValue("CALENDAR_CONFLICT")`. Package: `io.casehub.life.app.ledger` (qhorus PU). `domainContentBytes()` joins `conflictId|conflictType|domain|resolution|entityType|entityId` with `|` separator, returns as UTF-8 bytes.

## Sync Pipeline

### LifeCalendarInboundSyncJob

Quartz job, `@ApplicationScoped`, `@DisallowConcurrentExecution`.

**Configuration:**
- `casehub.life.calendar.sync.interval` — polling interval, default `PT5M`
- `casehub.life.calendar.sync.page-size` — events per sync page, default `100`

**Normal flow (sync token present):**
1. Load `LifeCalendarSyncState` for (defaultPlatformId, defaultCalendarId)
2. Call `CalendarPlatform.listEventsSync(calendarId, SyncRequest.withToken(syncToken))`
3. If `hasMore`: paginate until all pages consumed
4. Pass all `SyncResult.items()` and `SyncResult.deletedIds()` to `LifeCalendarConflictDetector`
5. Update `syncToken` and `lastSyncAt` on sync state
6. Single transaction wraps steps 4-5

**Initial sync (no sync token):**
1. Call `CalendarPlatform.listEventsSync(calendarId, SyncRequest.initial(pageSize))`
2. Paginate until complete
3. For each event: look up mapping by `calendarEventId`. If mapped and pushed fields are populated, run conflict detection. If mapped but pushed fields are null (pre-migration mapping), skip conflict detection for this event.
4. Store the sync token

**Sync token expiry recovery:**
1. Catch `SyncTokenExpiredException` from `listEventsSync()`
2. Clear `syncToken` to null on `LifeCalendarSyncState`
3. Execute the initial sync flow above
4. The idempotency guard in the conflict detector (check for existing DETECTED conflict on the same mapping) prevents duplicate conflicts from re-sync

### LifeCalendarConflictDetector

`@ApplicationScoped`. Pure comparison logic, no scheduling concerns.

**For each changed event in sync results:**
1. Look up mapping by `calendarEventId` via `LifeCalendarMappingStore.findByCalendarEventId()`
2. No mapping → skip (not CaseHub-managed)
3. Compare returned `(summary, startTime, endTime)` against mapping's `(pushedSummary, pushedStartTime, pushedEndTime)`
4. If pushed fields are null (pre-migration) → skip (no baseline to compare)
5. If all match → skip (CaseHub's own update echoed back)
6. If different → check `LifeCalendarConflictStore.findUnresolvedByMapping(mappingId)` for existing DETECTED conflict (idempotency)
7. If no existing conflict → create `LifeCalendarConflict` with type=MODIFIED, status=DETECTED, populate external fields from sync result
8. Fire `CalendarConflictDetected` CDI event

**For each deleted event ID:**
1. Look up mapping by `calendarEventId`
2. No mapping → skip
3. Check for existing unresolved conflict (idempotency)
4. Create `LifeCalendarConflict` with type=DELETED, status=DETECTED, external fields null
5. Fire `CalendarConflictDetected` CDI event

## Conflict Resolution

### LifeCalendarConflictService

`@ApplicationScoped`. Both operations `@Transactional`.

**Accept:**
1. Load conflict by ID, verify status is DETECTED (409 if already resolved)
2. Load mapping by `conflict.mappingId`
3. If MODIFIED:
   - Load WorkItem by `mapping.entityId` (entityType is WORK_ITEM)
   - Update WorkItem `expiresAt` to match external timing
   - Update mapping's pushed fields to match external values
4. If DELETED:
   - Delete the mapping (WorkItem keeps its deadline)
5. Set conflict: status=ACCEPTED, resolvedAt=now, resolvedBy=currentPrincipal.actorId()
6. Fire `CalendarConflictResolved` CDI event

**Dismiss:**
1. Load conflict by ID, verify status is DETECTED
2. Load mapping by `conflict.mappingId`
3. Load WorkItem by `mapping.entityId`
4. If MODIFIED:
   - Call `CalendarPlatform.updateEvent()` with CaseHub's current WorkItem state
   - Update mapping's pushed fields
5. If DELETED:
   - Staleness guard: call `CalendarPlatform.listEvents()` and check for existing event with matching summary and timing to prevent duplicates
   - Call `CalendarPlatform.createEvent()` to recreate
   - Update mapping with new `calendarEventId` and pushed fields
6. Set conflict: status=DISMISSED, resolvedAt=now, resolvedBy=currentPrincipal.actorId()
7. Fire `CalendarConflictResolved` CDI event

**Error handling for dismiss:**
- `CalendarPlatform` API failure (auth expired, network): throw, transaction rolls back, conflict stays DETECTED. User can retry.
- Duplicate event on dismiss-after-delete: staleness guard prevents. If guard itself fails (timing race), the duplicate is benign — user can delete manually.

### LifeCalendarConflictResource

`@Blocking @ApplicationScoped @Path("/life-calendar") @Produces(APPLICATION_JSON) @Consumes(APPLICATION_JSON)`

| Endpoint | Method | Auth | Response |
|----------|--------|------|----------|
| `/life-calendar/conflicts` | GET | household-member | `List<CalendarConflictResponse>` |
| `/life-calendar/conflicts/{id}` | GET | household-member | `CalendarConflictResponse` with diff |
| `/life-calendar/conflicts/{id}/resolve` | POST | household-member | `CalendarConflictResponse` (resolved) |
| `/life-calendar/sync` | POST | household-member | `SyncResultResponse` (conflictsDetected count) |
| `/life-calendar/conflicts/count` | GET | household-member | `long` (unresolved count for badge) |

**CalendarConflictResponse** (api/ record):
```
{id, mappingId, conflictType, detectedAt, status, resolvedAt, resolvedBy, domain,
 casehubSummary, casehubStartTime, casehubEndTime,
 externalSummary, externalStartTime, externalEndTime,
 entityType, entityId}
```

CaseHub-side fields derived from WorkItem at query time (not stored on conflict — always shows current CaseHub state).

**ResolveConflictRequest** (api/ record): `{action: "ACCEPT" | "DISMISS"}`

## Audit Trail

### LifeCalendarConflictLedgerObserver

`@ApplicationScoped`

- `@Observes CalendarConflictDetected` → `LifeLedgerWriter` writes `CalendarConflictLedgerEntry` with resolution="DETECTED", actorId="life-system", capability tag from domain
- `@Observes CalendarConflictResolved` → writes entry with resolution="ACCEPTED"/"DISMISSED", actorId from `resolvedBy`
- `@Transactional(REQUIRES_NEW)` — same pattern as `LifeDecisionLedgerObserver`

Capability tag mapping: domain → existing capability tags (e.g. HEALTH → "health-coordination", LEGAL → "legal-deadline"). Reuses `LifeCapabilities` constants.

## Outbound Push Changes

### LifeCalendarSyncService Extension

On every outbound `syncWorkItem()` call (create or update), after the CalendarPlatform call succeeds:
- Set `mapping.pushedSummary` = the summary sent to the provider
- Set `mapping.pushedStartTime` = the start time sent
- Set `mapping.pushedEndTime` = the end time sent
- Save the mapping

This establishes the baseline for conflict detection. After a push, any sync result that returns different values for these fields indicates an external edit.

## UI Changes

### life-calendar-panel

**Conflict indicators:**
- Events with unresolved conflicts: amber warning triangle icon alongside the existing CaseHub-managed blue dot
- DELETED conflicts: shown as a separate "Deleted externally" card with a dashed border

**Conflict detail (expanded):**
- Field-level diff: "Summary: [CaseHub value] → [External value]", "Start: [date] → [date]"
- Two action buttons: "Accept Change" (primary), "Dismiss" (secondary)
- For DELETED: "This event was deleted externally" with "Accept (remove link)" / "Dismiss (recreate)"

**Conflict count badge:**
- Calendar dock icon shows unresolved conflict count badge (same pattern as notification badge)
- Updated via SSE on `CalendarConflictDetected` / `CalendarConflictResolved` events

**Manual refresh:**
- Refresh icon button in the panel header
- Calls `POST /life-calendar/sync`
- Shows spinner during sync, then refreshes the event list

### SSE Events

New event types via `LifeEventBroadcaster`:
- `calendar.conflict.detected` — payload: `{conflictId, conflictType, domain, entityId}`
- `calendar.conflict.resolved` — payload: `{conflictId, resolution}`

`LifeEventController` in life-ui subscribes to these for real-time badge updates and panel refresh.

## Configuration

Added to `casehub/life/calendar-config.yaml`:

```yaml
casehub:
  life:
    calendar:
      sync:
        interval: PT5M
        page-size: 100
```

Registered in `application.properties` per PP-20260607-4c59f4.

## Testing Strategy

**Unit tests:**
- `LifeCalendarConflictDetectorTest` — compare logic: matching fields (no conflict), differing fields (MODIFIED), deleted IDs (DELETED), idempotency (existing unresolved conflict), null pushed fields (pre-migration skip)
- `CalendarConflictLedgerEntryDomainContentBytesTest` — `domainContentBytes()` field joining
- `CalendarConflictLedgerEntry` build-time validator

**Integration tests (`@QuarkusTest`):**
- `LifeCalendarInboundSyncJobTest` — full pipeline: seed mapping with pushed fields → mock CalendarPlatform to return changed event → verify conflict created, ledger entry written, SSE event fired
- `LifeCalendarConflictServiceTest` — accept: verify WorkItem deadline updated, mapping pushed fields updated, conflict resolved. Dismiss: verify CalendarPlatform.updateEvent called, pushed fields updated.
- `LifeCalendarConflictResourceTest` — REST endpoints: list, get, resolve, sync, count
- `LifeCalendarSyncTokenExpiryTest` — mock SyncTokenExpiredException → verify recovery: token cleared, full re-sync, no false conflicts
- `LifeCalendarConflictLedgerObserverTest` — verify ledger entries on detect and resolve events

**Robustness tests:**
- Concurrent sync: verify `@DisallowConcurrentExecution` prevents duplicate processing
- Dismiss-after-delete staleness: mock existing event in calendar → verify no duplicate created
- Pre-migration mappings (null pushed fields): verify no false conflicts
- CalendarPlatform API failure during dismiss: verify transaction rollback, conflict stays DETECTED

## Migration Sequence

1. V116 — `life_calendar_sync_state` table + `life_calendar_mapping` ALTER (pushed fields)
2. V117 — `life_calendar_conflict` table
3. V2102 — `calendar_conflict_ledger_entry` join table (qhorus PU, `db/life/ledger/migration/`)

## References

- casehubio/life#114 — calendar sync outbound push (prerequisite)
- `CalendarPlatform.listEventsSync()` — casehub-connectors SPI (connectors#88)
- `LifeCalendarSyncService` — `app/src/main/java/io/casehub/life/app/service/LifeCalendarSyncService.java`
- `JpaLifeCalendarMapping` — `app/src/main/java/io/casehub/life/app/entity/JpaLifeCalendarMapping.java`
- `LifeCalendarOutboundObserver` — `app/src/main/java/io/casehub/life/app/observer/LifeCalendarOutboundObserver.java`
- `LifeCalendarResource` — `app/src/main/java/io/casehub/life/app/resource/LifeCalendarResource.java`
- `LifeDecisionLedgerObserver` — CDI observer pattern reference
- `LifeHeartbeatJob` — Quartz job pattern reference
- `dual-trail-audit-pattern.md` — operational vs compliance trail
- Decision review: R1-03 (hash false positives → stored field values), R1-04 (sync token expiry), R1-05 (conflict entity), R1-09 (dismiss duplicate prevention), R1-10 (single-calendar scope), R1-11 (concurrency)
