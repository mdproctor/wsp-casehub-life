# Calendar Conflict Detection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #142 — feat: Calendar conflict detection — detect and respect external edits to CaseHub-created events
**Issue group:** #142

**Goal:** Add an inbound sync pipeline that detects when CaseHub-managed calendar events are modified or deleted externally, surfaces conflicts to users, and records resolutions in the tamper-evident ledger.

**Architecture:** Quartz-scheduled incremental sync via `CalendarPlatform.listEventsSync()` compares returned events against stored pushed state on `LifeCalendarMapping`. Differences create `LifeCalendarConflict` entities. Users resolve via REST (accept = update WorkItem, dismiss = push CaseHub state back). `CalendarConflictLedgerEntry` provides tamper-evident audit.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Flyway, Quartz, Lit 3.x (frontend)

## Global Constraints

- Java 21 source, Java 26 JVM
- Quarkus 3.32.2
- Flyway: life domain migrations at `db/life/migration/` (V116+), ledger join tables at `db/life/ledger/migration/` (V2102)
- JPA entities in `io.casehub.life.app.entity` (default PU), ledger subclasses in `io.casehub.life.app.ledger` (qhorus PU)
- SPIs in `api/` (zero framework imports, no JPA), implementations in `app/`
- REST resources: `@Blocking @ApplicationScoped`, class-level `@Produces`/`@Consumes`, `@Transactional` on service methods only
- `EventTiming` is `io.casehub.connectors.calendar.spi.EventTiming` — sealed interface with `AllDay(LocalDate, LocalDate)` and `Timed(Instant, Instant, ZoneId)` variants
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api` then `mvn install -pl app`
- Test single class: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClassName --batch-mode`
- Tenancy ID in tests: `278776f9-e1b0-46fb-9032-8bddebdcf9ce`

---

## Batch 1: Data Model — sync state, mapping extension, conflict entity

After this batch: Flyway migrations run, JPA entities persist, SPI interfaces compile, all store operations work with in-memory and JPA implementations. No sync pipeline yet.

### Task 1: LifeCalendarSyncState — entity, SPI, stores, migration

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/LifeCalendarSyncState.java`
- Create: `api/src/main/java/io/casehub/life/api/LifeCalendarSyncStateStore.java`
- Create: `app/src/main/java/io/casehub/life/app/entity/JpaLifeCalendarSyncState.java`
- Create: `app/src/main/java/io/casehub/life/app/repository/JpaLifeCalendarSyncStateStore.java`
- Create: `app/src/test/java/io/casehub/life/app/repository/InMemoryLifeCalendarSyncStateStore.java`
- Create: `app/src/main/resources/db/life/migration/V116__calendar_sync_state_and_mapping_extension.sql`
- Test: `app/src/test/java/io/casehub/life/app/repository/JpaLifeCalendarSyncStateStoreTest.java`

**Interfaces:**
- Produces: `LifeCalendarSyncState(UUID id, String platformId, String calendarId, String syncToken, Instant lastSyncAt)` — api record
- Produces: `LifeCalendarSyncStateStore.findByCalendar(String platformId, String calendarId) → Optional<LifeCalendarSyncState>`
- Produces: `LifeCalendarSyncStateStore.save(LifeCalendarSyncState state) → LifeCalendarSyncState`

- [ ] **Step 1: Write V116 Flyway migration**

```sql
-- V116__calendar_sync_state_and_mapping_extension.sql

CREATE TABLE life_calendar_sync_state (
    id           UUID PRIMARY KEY,
    platform_id  VARCHAR(50)   NOT NULL,
    calendar_id  VARCHAR(255)  NOT NULL,
    sync_token   VARCHAR(1024),
    last_sync_at TIMESTAMP,
    UNIQUE (platform_id, calendar_id)
);

ALTER TABLE life_calendar_mapping
    ADD COLUMN pushed_timing TEXT;

ALTER TABLE life_calendar_mapping
    ADD COLUMN version BIGINT NOT NULL DEFAULT 0;

CREATE INDEX idx_calendar_mapping_event_id ON life_calendar_mapping(calendar_event_id);
```

- [ ] **Step 2: Write the api record**

```java
// api/src/main/java/io/casehub/life/api/LifeCalendarSyncState.java
package io.casehub.life.api;

import java.time.Instant;
import java.util.UUID;

public record LifeCalendarSyncState(
        UUID id,
        String platformId,
        String calendarId,
        String syncToken,
        Instant lastSyncAt) {
}
```

- [ ] **Step 3: Write the SPI interface**

```java
// api/src/main/java/io/casehub/life/api/LifeCalendarSyncStateStore.java
package io.casehub.life.api;

import java.util.Optional;

public interface LifeCalendarSyncStateStore {

    Optional<LifeCalendarSyncState> findByCalendar(String platformId, String calendarId);

    LifeCalendarSyncState save(LifeCalendarSyncState state);
}
```

- [ ] **Step 4: Write the JPA entity**

```java
// app/src/main/java/io/casehub/life/app/entity/JpaLifeCalendarSyncState.java
package io.casehub.life.app.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "life_calendar_sync_state")
public class JpaLifeCalendarSyncState {

    @Id
    public UUID id;

    @Column(name = "platform_id", nullable = false, length = 50)
    public String platformId;

    @Column(name = "calendar_id", nullable = false)
    public String calendarId;

    @Column(name = "sync_token", length = 1024)
    public String syncToken;

    @Column(name = "last_sync_at")
    public Instant lastSyncAt;
}
```

- [ ] **Step 5: Write the failing JPA store test**

```java
// app/src/test/java/io/casehub/life/app/repository/JpaLifeCalendarSyncStateStoreTest.java
@QuarkusTest
class JpaLifeCalendarSyncStateStoreTest {

    @Inject JpaLifeCalendarSyncStateStore store;

    @Test @Transactional
    void save_and_findByCalendar() {
        var state = new LifeCalendarSyncState(
                UUID.randomUUID(), "google", "primary", "token-1", Instant.now());
        store.save(state);

        var found = store.findByCalendar("google", "primary");
        assertTrue(found.isPresent());
        assertEquals("token-1", found.get().syncToken());
    }

    @Test @Transactional
    void save_updates_existing() {
        var id = UUID.randomUUID();
        store.save(new LifeCalendarSyncState(id, "google", "primary", "token-1", Instant.now()));
        store.save(new LifeCalendarSyncState(id, "google", "primary", "token-2", Instant.now()));

        var found = store.findByCalendar("google", "primary");
        assertEquals("token-2", found.get().syncToken());
    }

    @Test
    void findByCalendar_returns_empty_when_not_found() {
        var found = store.findByCalendar("google", "nonexistent");
        assertTrue(found.isEmpty());
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=JpaLifeCalendarSyncStateStoreTest --batch-mode -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `JpaLifeCalendarSyncStateStore` does not exist

- [ ] **Step 7: Write the JPA store implementation**

```java
// app/src/main/java/io/casehub/life/app/repository/JpaLifeCalendarSyncStateStore.java
package io.casehub.life.app.repository;

import io.casehub.life.api.LifeCalendarSyncState;
import io.casehub.life.api.LifeCalendarSyncStateStore;
import io.casehub.life.app.entity.JpaLifeCalendarSyncState;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.NoResultException;
import jakarta.transaction.Transactional;

import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
@io.quarkus.arc.DefaultBean
public class JpaLifeCalendarSyncStateStore implements LifeCalendarSyncStateStore {

    @Inject
    EntityManager em;

    @Override
    @Transactional
    public LifeCalendarSyncState save(LifeCalendarSyncState state) {
        var entity = em.find(JpaLifeCalendarSyncState.class, state.id());
        if (entity == null) {
            entity = new JpaLifeCalendarSyncState();
            entity.id = state.id();
        }
        entity.platformId = state.platformId();
        entity.calendarId = state.calendarId();
        entity.syncToken = state.syncToken();
        entity.lastSyncAt = state.lastSyncAt();
        em.persist(entity);
        return toRecord(entity);
    }

    @Override
    public Optional<LifeCalendarSyncState> findByCalendar(String platformId, String calendarId) {
        try {
            var entity = em.createQuery(
                    "SELECT s FROM JpaLifeCalendarSyncState s WHERE s.platformId = :pid AND s.calendarId = :cid",
                    JpaLifeCalendarSyncState.class)
                    .setParameter("pid", platformId)
                    .setParameter("cid", calendarId)
                    .getSingleResult();
            return Optional.of(toRecord(entity));
        } catch (NoResultException e) {
            return Optional.empty();
        }
    }

    private static LifeCalendarSyncState toRecord(JpaLifeCalendarSyncState e) {
        return new LifeCalendarSyncState(e.id, e.platformId, e.calendarId, e.syncToken, e.lastSyncAt);
    }
}
```

- [ ] **Step 8: Write InMemoryLifeCalendarSyncStateStore for unit tests**

```java
// app/src/test/java/io/casehub/life/app/repository/InMemoryLifeCalendarSyncStateStore.java
package io.casehub.life.app.repository;

import io.casehub.life.api.LifeCalendarSyncState;
import io.casehub.life.api.LifeCalendarSyncStateStore;

import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryLifeCalendarSyncStateStore implements LifeCalendarSyncStateStore {

    private final Map<String, LifeCalendarSyncState> store = new ConcurrentHashMap<>();

    @Override
    public LifeCalendarSyncState save(LifeCalendarSyncState state) {
        store.put(state.platformId() + ":" + state.calendarId(), state);
        return state;
    }

    @Override
    public Optional<LifeCalendarSyncState> findByCalendar(String platformId, String calendarId) {
        return Optional.ofNullable(store.get(platformId + ":" + calendarId));
    }
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=JpaLifeCalendarSyncStateStoreTest --batch-mode -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/life/api/LifeCalendarSyncState.java api/src/main/java/io/casehub/life/api/LifeCalendarSyncStateStore.java app/src/main/java/io/casehub/life/app/entity/JpaLifeCalendarSyncState.java app/src/main/java/io/casehub/life/app/repository/JpaLifeCalendarSyncStateStore.java app/src/test/java/io/casehub/life/app/repository/InMemoryLifeCalendarSyncStateStore.java app/src/test/java/io/casehub/life/app/repository/JpaLifeCalendarSyncStateStoreTest.java app/src/main/resources/db/life/migration/V116__calendar_sync_state_and_mapping_extension.sql
git commit -m "feat(#142): add LifeCalendarSyncState entity, SPI, stores, V116 migration Refs #142"
```

### Task 2: Extend LifeCalendarMapping with pushedTiming and version

**Files:**
- Modify: `api/src/main/java/io/casehub/life/api/LifeCalendarMapping.java`
- Modify: `app/src/main/java/io/casehub/life/app/entity/JpaLifeCalendarMapping.java`
- Modify: `app/src/main/java/io/casehub/life/app/repository/JpaLifeCalendarMappingStore.java`
- Modify: `app/src/test/java/io/casehub/life/app/service/InMemoryLifeCalendarMappingStore.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/LifeCalendarSyncService.java`
- Test: `app/src/test/java/io/casehub/life/api/LifeCalendarMappingTest.java`

**Interfaces:**
- Consumes: V116 migration (Task 1 — adds `pushed_timing` and `version` columns)
- Produces: `LifeCalendarMapping` record now includes `EventTiming pushedTiming` field
- Produces: `JpaLifeCalendarMapping` with `@Version public long version` and `public String pushedTiming` (JSON TEXT)

- [ ] **Step 1: Add `pushedTiming` field to `LifeCalendarMapping` api record**

Change the record to add `EventTiming pushedTiming` between `summary` and `createdAt`:

```java
public record LifeCalendarMapping(
        UUID id,
        CalendarEntityType entityType,
        String entityId,
        String calendarPlatformId,
        String calendarId,
        String calendarEventId,
        LifeDomain domain,
        String summary,
        EventTiming pushedTiming,
        Instant createdAt,
        Instant updatedAt) {
}
```

- [ ] **Step 2: Add fields to `JpaLifeCalendarMapping` JPA entity**

Add two fields:

```java
@Column(name = "pushed_timing", columnDefinition = "TEXT")
public String pushedTiming; // JSON-serialized EventTiming

@Version
@Column(nullable = false)
public long version;
```

- [ ] **Step 3: Update `JpaLifeCalendarMappingStore.toRecord()` and `save()`**

Update `toRecord()` to deserialize `pushedTiming` from JSON TEXT to `EventTiming`. Update `save()` to serialize `EventTiming` to JSON. Use Jackson `ObjectMapper` for serialization.

- [ ] **Step 4: Update `InMemoryLifeCalendarMappingStore`**

The in-memory store stores the record directly — no serialization needed. Update constructor calls to include the new `pushedTiming` parameter.

- [ ] **Step 5: Update `LifeCalendarSyncService.syncWorkItem()`**

After the CalendarPlatform create/update call, set `pushedTiming` on the mapping to the `EventTiming` built by `buildTiming(expiresAt)`:

```java
var timing = buildTiming(expiresAt);
// ... CalendarPlatform call ...
mappingStore.save(new LifeCalendarMapping(
        mapping.id(), mapping.entityType(), mapping.entityId(),
        mapping.calendarPlatformId(), mapping.calendarId(), mapping.calendarEventId(),
        domain, summary, timing, mapping.createdAt(), Instant.now()));
```

- [ ] **Step 6: Fix all compilation errors from the new record field**

All callers of the `LifeCalendarMapping` constructor need the new `pushedTiming` parameter. Fix `LifeCalendarSyncService`, test fixtures, `LifeCalendarOutboundObserver` references, `LifeCalendarResourceTest`.

- [ ] **Step 7: Run full test suite to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#142): extend LifeCalendarMapping with pushedTiming and @Version Refs #142"
```

### Task 3: LifeCalendarConflict — entity, SPI, enums, stores, migration

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/CalendarConflictType.java`
- Create: `api/src/main/java/io/casehub/life/api/CalendarConflictStatus.java`
- Create: `api/src/main/java/io/casehub/life/api/LifeCalendarConflict.java`
- Create: `api/src/main/java/io/casehub/life/api/LifeCalendarConflictStore.java`
- Create: `app/src/main/java/io/casehub/life/app/entity/JpaLifeCalendarConflict.java`
- Create: `app/src/main/java/io/casehub/life/app/repository/JpaLifeCalendarConflictStore.java`
- Create: `app/src/test/java/io/casehub/life/app/repository/InMemoryLifeCalendarConflictStore.java`
- Create: `app/src/main/resources/db/life/migration/V117__calendar_conflict.sql`
- Test: `app/src/test/java/io/casehub/life/app/repository/JpaLifeCalendarConflictStoreTest.java`

**Interfaces:**
- Consumes: `CalendarEntityType`, `LifeDomain` (api/)
- Produces: `CalendarConflictType` enum: `MODIFIED`, `DELETED`
- Produces: `CalendarConflictStatus` enum: `DETECTED`, `ACCEPTED`, `DISMISSED`
- Produces: `LifeCalendarConflict(UUID id, UUID mappingId, CalendarConflictType conflictType, Instant detectedAt, String externalSummary, EventTiming externalTiming, CalendarConflictStatus status, Instant resolvedAt, String resolvedBy, LifeDomain domain)` — api record
- Produces: `LifeCalendarConflictStore` SPI: `save`, `findById`, `findByMapping`, `findUnresolved`, `countUnresolved`, `findUnresolvedByMapping`, `deleteByMapping`

- [ ] **Step 1: Write V117 Flyway migration**

```sql
-- V117__calendar_conflict.sql

CREATE TABLE life_calendar_conflict (
    id                UUID        PRIMARY KEY,
    mapping_id        UUID        NOT NULL REFERENCES life_calendar_mapping(id),
    conflict_type     VARCHAR(20) NOT NULL,
    detected_at       TIMESTAMP   NOT NULL,
    external_summary  VARCHAR(500),
    external_timing   TEXT,
    status            VARCHAR(20) NOT NULL,
    resolved_at       TIMESTAMP,
    resolved_by       VARCHAR(255),
    domain            VARCHAR(50) NOT NULL
);

CREATE INDEX idx_calendar_conflict_status ON life_calendar_conflict(status);
CREATE INDEX idx_calendar_conflict_mapping ON life_calendar_conflict(mapping_id);
```

- [ ] **Step 2: Write enums**

```java
// api/src/main/java/io/casehub/life/api/CalendarConflictType.java
package io.casehub.life.api;
public enum CalendarConflictType { MODIFIED, DELETED }

// api/src/main/java/io/casehub/life/api/CalendarConflictStatus.java
package io.casehub.life.api;
public enum CalendarConflictStatus { DETECTED, ACCEPTED, DISMISSED }
```

- [ ] **Step 3: Write the api record**

```java
// api/src/main/java/io/casehub/life/api/LifeCalendarConflict.java
package io.casehub.life.api;

import io.casehub.connectors.calendar.spi.EventTiming;
import java.time.Instant;
import java.util.UUID;

public record LifeCalendarConflict(
        UUID id,
        UUID mappingId,
        CalendarConflictType conflictType,
        Instant detectedAt,
        String externalSummary,
        EventTiming externalTiming,
        CalendarConflictStatus status,
        Instant resolvedAt,
        String resolvedBy,
        LifeDomain domain) {
}
```

- [ ] **Step 4: Write the SPI interface**

```java
// api/src/main/java/io/casehub/life/api/LifeCalendarConflictStore.java
package io.casehub.life.api;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface LifeCalendarConflictStore {
    LifeCalendarConflict save(LifeCalendarConflict conflict);
    Optional<LifeCalendarConflict> findById(UUID id);
    List<LifeCalendarConflict> findByMapping(UUID mappingId);
    List<LifeCalendarConflict> findUnresolved();
    long countUnresolved();
    Optional<LifeCalendarConflict> findUnresolvedByMapping(UUID mappingId);
    void deleteByMapping(UUID mappingId);
}
```

- [ ] **Step 5: Write JPA entity, JPA store, in-memory store**

Follow the same pattern as `JpaLifeCalendarMapping`/`JpaLifeCalendarMappingStore`. `externalTiming` stored as JSON TEXT (same Jackson serialization as `JpaLifeCalendarMapping.pushedTiming`).

- [ ] **Step 6: Write the failing store test**

Test `save`, `findById`, `findUnresolved`, `findUnresolvedByMapping`, `countUnresolved`, `deleteByMapping`. Seed a mapping first (FK constraint).

- [ ] **Step 7: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=JpaLifeCalendarConflictStoreTest --batch-mode -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — store class not yet functional

- [ ] **Step 8: Implement until tests pass**

- [ ] **Step 9: Run tests to verify they pass**

Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(#142): add LifeCalendarConflict entity, SPI, stores, V117 migration Refs #142"
```

---

## Batch 2: Sync Pipeline — conflict detector, sync job, CDI events

After this batch: incremental sync runs on a Quartz schedule, detects external edits and deletions, creates conflict entities, and fires CDI events. No resolution or audit trail yet.

### Task 4: CDI event records and LifeEventType extension

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/event/CalendarConflictDetected.java`
- Create: `app/src/main/java/io/casehub/life/app/event/CalendarConflictResolved.java`
- Modify: `app/src/main/java/io/casehub/life/app/event/LifeEventType.java`

**Interfaces:**
- Produces: `CalendarConflictDetected(UUID conflictId, CalendarConflictType conflictType, LifeDomain domain, String entityId, UUID mappingId)` — CDI event record
- Produces: `CalendarConflictResolved(UUID conflictId, CalendarConflictStatus resolution, String resolvedBy, LifeDomain domain, String entityId)` — CDI event record
- Produces: `LifeEventType.CALENDAR_CONFLICT_DETECTED`, `LifeEventType.CALENDAR_CONFLICT_RESOLVED` — new enum values

- [ ] **Step 1: Create CDI event records**

```java
// app/src/main/java/io/casehub/life/app/event/CalendarConflictDetected.java
package io.casehub.life.app.event;

import io.casehub.life.api.CalendarConflictType;
import io.casehub.life.api.LifeDomain;
import java.util.UUID;

public record CalendarConflictDetected(
        UUID conflictId,
        CalendarConflictType conflictType,
        LifeDomain domain,
        String entityId,
        UUID mappingId) {
}

// app/src/main/java/io/casehub/life/app/event/CalendarConflictResolved.java
package io.casehub.life.app.event;

import io.casehub.life.api.CalendarConflictStatus;
import io.casehub.life.api.LifeDomain;
import java.util.UUID;

public record CalendarConflictResolved(
        UUID conflictId,
        CalendarConflictStatus resolution,
        String resolvedBy,
        LifeDomain domain,
        String entityId) {
}
```

- [ ] **Step 2: Add enum values to LifeEventType**

Add `CALENDAR_CONFLICT_DETECTED` and `CALENDAR_CONFLICT_RESOLVED` to the enum.

- [ ] **Step 3: Compile to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat(#142): add CDI event records and LifeEventType calendar values Refs #142"
```

### Task 5: LifeCalendarConflictDetector — comparison logic

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/LifeCalendarConflictDetector.java`
- Test: `app/src/test/java/io/casehub/life/app/service/LifeCalendarConflictDetectorTest.java`

**Interfaces:**
- Consumes: `LifeCalendarMappingStore.findByCalendarEventId()`, `LifeCalendarConflictStore.findUnresolvedByMapping()`, `LifeCalendarConflictStore.save()`
- Consumes: `CalendarEvent` (from connectors SPI — `summary()`, `timing()`, `id()`)
- Consumes: `SyncResult.items()`, `SyncResult.deletedIds()`
- Produces: `LifeCalendarConflictDetector.detectConflicts(List<CalendarEvent> changedEvents, List<String> deletedIds)` — fires `CalendarConflictDetected` CDI events, returns number of conflicts detected

- [ ] **Step 1: Write failing tests**

Test cases:
1. Changed event, no mapping → skip (0 conflicts)
2. Changed event, mapping with matching summary+timing → skip (echo)
3. Changed event, mapping with different timing → create MODIFIED conflict
4. Changed event, `CASE_MILESTONE` entity type → skip
5. Changed event, null `pushedTiming` (pre-migration) → skip
6. Changed event, existing unresolved conflict with same external values → skip (idempotency)
7. Changed event, existing unresolved conflict with different external values → update conflict
8. Deleted event ID, mapping exists → create DELETED conflict
9. Deleted event ID, no mapping → skip

Each test uses `InMemoryLifeCalendarMappingStore` and `InMemoryLifeCalendarConflictStore` for fast isolated testing. Mock CDI `Event<CalendarConflictDetected>` with Mockito.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeCalendarConflictDetectorTest --batch-mode -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL

- [ ] **Step 3: Implement LifeCalendarConflictDetector**

```java
@ApplicationScoped
public class LifeCalendarConflictDetector {

    @Inject LifeCalendarMappingStore mappingStore;
    @Inject LifeCalendarConflictStore conflictStore;
    @Inject Event<CalendarConflictDetected> conflictDetectedEvent;

    public int detectConflicts(List<CalendarEvent> changedEvents, List<String> deletedIds) {
        int count = 0;
        for (var event : changedEvents) {
            // 1. lookup mapping by calendarEventId
            // 2. skip if no mapping, CASE_MILESTONE, or null pushedTiming
            // 3. compare summary + timing
            // 4. if different → idempotency check → create/update conflict → fire event
            // ... (full implementation per spec §LifeCalendarConflictDetector)
        }
        for (var deletedId : deletedIds) {
            // 1. lookup mapping
            // 2. skip if no mapping or CASE_MILESTONE
            // 3. idempotency check
            // 4. create DELETED conflict → fire event
        }
        return count;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#142): add LifeCalendarConflictDetector with comparison logic Refs #142"
```

### Task 6: LifeCalendarInboundSyncJob — Quartz scheduled sync

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/LifeCalendarInboundSyncJob.java`
- Test: `app/src/test/java/io/casehub/life/app/service/LifeCalendarInboundSyncJobTest.java`

**Interfaces:**
- Consumes: `CalendarPlatformService` (from connectors), `LifeCalendarSyncStateStore`, `LifeCalendarConflictDetector`
- Consumes: Config: `casehub.life.calendar.sync.interval` (default PT5M), `casehub.life.calendar.sync.page-size` (default 100)
- Produces: `LifeCalendarInboundSyncJob` — Quartz `@Scheduled` job
- Produces: `LifeCalendarInboundSyncJob.executeSync()` — public method for on-demand sync (called by REST endpoint)

- [ ] **Step 1: Write failing integration test**

`@QuarkusTest` — mock `CalendarPlatformService` to return a `SyncResult` with one changed event. Seed a mapping with `pushedTiming`. Verify conflict is created after sync runs.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement the sync job**

Key behaviors:
- Load/create sync state for (defaultPlatformId, defaultCalendarId)
- Call `listEventsSync()` with stored token (or `SyncRequest.initial()` for first run)
- Paginate if `hasMore`
- Pass results to `LifeCalendarConflictDetector.detectConflicts()`
- Update sync token and `lastSyncAt`
- Catch `SyncTokenExpiredException` → clear token → re-run as initial sync
- `@DisallowConcurrentExecution` via Quartz

- [ ] **Step 4: Add calendar sync config to `calendar-config.yaml`**

```yaml
casehub:
  life:
    calendar:
      sync:
        interval: PT5M
        page-size: 100
```

Register in `application.properties` per PP-20260607-4c59f4.

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Write sync token expiry recovery test**

Mock `SyncTokenExpiredException` from `listEventsSync()`. Verify: token cleared, initial sync executed, no false conflicts for pre-migration mappings.

- [ ] **Step 7: Run tests to verify they pass**

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#142): add LifeCalendarInboundSyncJob with Quartz scheduling and token expiry recovery Refs #142"
```

---

## Batch 3: Resolution + Audit — conflict service, REST endpoints, ledger, outbound push changes

After this batch: users can accept/dismiss conflicts via REST. Tamper-evident ledger records all detections and resolutions. Outbound push auto-dismisses stale conflicts and handles mapping deletion with FK safety.

### Task 7: CalendarConflictLedgerEntry — JOINED ledger subclass

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/ledger/CalendarConflictLedgerEntry.java`
- Create: `app/src/main/resources/db/life/ledger/migration/V2102__calendar_conflict_ledger_entry.sql`
- Test: `app/src/test/java/io/casehub/life/app/ledger/CalendarConflictLedgerEntryDomainContentBytesTest.java`

**Interfaces:**
- Consumes: `JpaLedgerEntry` (JOINED inheritance from casehub-ledger)
- Produces: `CalendarConflictLedgerEntry` with fields: `conflictId`, `conflictType`, `domain`, `resolution`, `entityType`, `entityId`

- [ ] **Step 1: Write V2102 migration**

```sql
-- V2102__calendar_conflict_ledger_entry.sql
CREATE TABLE calendar_conflict_ledger_entry (
    id             UUID         PRIMARY KEY REFERENCES ledger_entry(id),
    conflict_id    UUID         NOT NULL,
    conflict_type  VARCHAR(20)  NOT NULL,
    domain         VARCHAR(50)  NOT NULL,
    resolution     VARCHAR(20)  NOT NULL,
    entity_type    VARCHAR(30)  NOT NULL,
    entity_id      VARCHAR(255) NOT NULL
);
```

- [ ] **Step 2: Write the ledger entry class**

Follow `HealthDecisionLedgerEntry` pattern: `@Entity`, `@DiscriminatorValue("CALENDAR_CONFLICT")`, JOINED inheritance, `domainContentBytes()` joins fields with `|`.

- [ ] **Step 3: Write `domainContentBytes()` test**

Verify that `domainContentBytes()` returns `conflictId|conflictType|domain|resolution|entityType|entityId` as UTF-8 bytes.

- [ ] **Step 4: Run test, verify pass**

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#142): add CalendarConflictLedgerEntry JOINED subclass and V2102 migration Refs #142"
```

### Task 8: LifeCalendarConflictLedgerObserver — audit trail

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/observer/LifeCalendarConflictLedgerObserver.java`
- Test: `app/src/test/java/io/casehub/life/app/observer/LifeCalendarConflictLedgerObserverTest.java`

**Interfaces:**
- Consumes: `CalendarConflictDetected`, `CalendarConflictResolved` (CDI events)
- Consumes: `LifeLedgerWriter` for creating ledger entries
- Consumes: `LifeCapabilities` for domain→capability mapping

- [ ] **Step 1: Write failing unit test**

Mock `LifeLedgerWriter`. Fire `CalendarConflictDetected` → verify `CalendarConflictLedgerEntry` written with resolution="DETECTED", actorId="life-system". Fire `CalendarConflictResolved` → verify entry written with resolution matching the event.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement the observer**

`@ApplicationScoped`, `@Observes CalendarConflictDetected`, `@Observes CalendarConflictResolved`, `@Transactional(REQUIRES_NEW)`.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#142): add LifeCalendarConflictLedgerObserver for tamper-evident audit Refs #142"
```

### Task 9: LifeCalendarConflictService — accept and dismiss resolution

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/LifeCalendarConflictService.java`
- Test: `app/src/test/java/io/casehub/life/app/service/LifeCalendarConflictServiceTest.java`

**Interfaces:**
- Consumes: `LifeCalendarConflictStore`, `LifeCalendarMappingStore`, `CalendarPlatformService`, `CurrentPrincipal`
- Consumes: `Event<CalendarConflictResolved>`
- Produces: `LifeCalendarConflictService.accept(UUID conflictId)` → updates WorkItem, mapping; fires CDI event
- Produces: `LifeCalendarConflictService.dismiss(UUID conflictId)` → pushes to Google, updates mapping; fires CDI event

- [ ] **Step 1: Write failing tests**

Test cases:
1. Accept MODIFIED — verify WorkItem `expiresAt` updated (AllDay→Instant and Timed→Instant), mapping `pushedTiming` updated, conflict status=ACCEPTED
2. Accept DELETED — verify mapping deleted (conflicts deleted first for FK), WorkItem keeps deadline
3. Dismiss MODIFIED — verify `CalendarPlatform.updateEvent()` called, mapping `pushedTiming` updated
4. Dismiss DELETED — verify staleness guard (listEvents check), `CalendarPlatform.createEvent()` called, mapping `calendarEventId` updated
5. Already-resolved conflict → 409
6. CDI event fired on both accept and dismiss

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement LifeCalendarConflictService**

Two `@Transactional` methods following the spec's §Conflict Resolution exactly.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#142): add LifeCalendarConflictService with accept/dismiss resolution Refs #142"
```

### Task 10: Outbound push changes — auto-dismiss stale conflicts, FK-safe mapping deletion

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/service/LifeCalendarSyncService.java`
- Modify: `app/src/test/java/io/casehub/life/app/service/LifeCalendarSyncServiceTest.java`

**Interfaces:**
- Consumes: `LifeCalendarConflictStore.findUnresolvedByMapping()`, `LifeCalendarConflictStore.deleteByMapping()`
- Consumes: `Event<CalendarConflictResolved>`

- [ ] **Step 1: Write failing test — outbound push auto-dismisses existing DETECTED conflict**

Seed a mapping with a DETECTED conflict. Call `syncWorkItem()`. Verify conflict status=DISMISSED, CDI event fired.

- [ ] **Step 2: Write failing test — removeWorkItem resolves conflicts before deleting mapping**

Seed a mapping with DETECTED + historical ACCEPTED conflicts. Call `removeWorkItem()`. Verify: DETECTED auto-dismissed with CDI event, all conflict rows deleted, mapping deleted (no FK violation).

- [ ] **Step 3: Run tests to verify they fail**

- [ ] **Step 4: Implement**

Add `LifeCalendarConflictStore` and `Event<CalendarConflictResolved>` dependencies to `LifeCalendarSyncService`. Add auto-dismiss logic to `syncWorkItem()` (after push). Add conflict cleanup to `removeWorkItem()` (before mapping delete).

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#142): auto-dismiss stale conflicts on outbound push, FK-safe mapping deletion Refs #142"
```

### Task 11: LifeCalendarConflictResource — REST endpoints

**Files:**
- Create: `app/src/main/java/io/casehub/life/api/CalendarConflictResponse.java`
- Create: `app/src/main/java/io/casehub/life/api/ResolveConflictRequest.java`
- Create: `app/src/main/java/io/casehub/life/api/SyncResultResponse.java`
- Create: `app/src/main/java/io/casehub/life/app/resource/LifeCalendarConflictResource.java`
- Modify: `app/src/main/java/io/casehub/life/app/resource/LifeCalendarResource.java` (add sync endpoint)
- Test: `app/src/test/java/io/casehub/life/app/resource/LifeCalendarConflictResourceTest.java`

**Interfaces:**
- Consumes: `LifeCalendarConflictService`, `LifeCalendarConflictStore`, `LifeCalendarInboundSyncJob.executeSync()`
- Produces: REST endpoints: `GET /life-calendar/conflicts`, `GET /life-calendar/conflicts/{id}`, `POST /life-calendar/conflicts/{id}/resolve`, `POST /life-calendar/sync`, `GET /life-calendar/conflicts/count`

- [ ] **Step 1: Create api records for REST**

```java
// CalendarConflictResponse — includes CaseHub-side fields derived at query time
// ResolveConflictRequest — {action: "ACCEPT" | "DISMISS"}
// SyncResultResponse — {conflictsDetected: int}
```

- [ ] **Step 2: Write failing REST test**

`@QuarkusTest` with RestAssured. Test list, get, resolve (accept and dismiss), sync trigger, count.

- [ ] **Step 3: Run test to verify it fails**

- [ ] **Step 4: Implement the resource**

`@Blocking @ApplicationScoped @Path("/life-calendar") @Produces(APPLICATION_JSON) @Consumes(APPLICATION_JSON) @RolesAllowed("household-member")`

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#142): add LifeCalendarConflictResource REST endpoints Refs #142"
```

---

## Batch 4: UI — conflict indicators, resolution actions, SSE integration

After this batch: calendar panel shows conflict indicators, users can accept/dismiss inline, badge shows unresolved count, manual refresh triggers sync.

### Task 12: life-calendar-panel — conflict UI

**Files:**
- Modify: `life-ui/src/panels/calendar-panel.ts`
- Modify: `life-ui/src/events/life-event-controller.ts`

**Interfaces:**
- Consumes: `GET /life-calendar/conflicts`, `GET /life-calendar/conflicts/count`, `POST /life-calendar/conflicts/{id}/resolve`, `POST /life-calendar/sync`
- Consumes: SSE event types `CALENDAR_CONFLICT_DETECTED`, `CALENDAR_CONFLICT_RESOLVED`

- [ ] **Step 1: Add SSE event type constants to `life-event-controller.ts`**

Add `CALENDAR_CONFLICT_DETECTED` and `CALENDAR_CONFLICT_RESOLVED` to the event type constants. Add callback registration for these events.

- [ ] **Step 2: Add conflict fetching to `calendar-panel.ts`**

On panel load: fetch `/life-calendar/conflicts` alongside events. Merge conflicts into the event list by mapping ID.

- [ ] **Step 3: Add conflict indicators**

Events with unresolved conflicts get an amber warning triangle icon. DELETED conflicts shown as dashed-border cards.

- [ ] **Step 4: Add conflict detail expansion**

Click a conflicted event → expand to show diff card with field-level comparison ("Summary: [CaseHub] → [External]", "Start: [date] → [date]"). Two buttons: "Accept Change" (primary), "Dismiss" (secondary).

- [ ] **Step 5: Wire accept/dismiss to REST**

Click Accept → `POST /life-calendar/conflicts/{id}/resolve` with `{action: "ACCEPT"}`. Click Dismiss → same with `{action: "DISMISS"}`. Refresh panel after resolution.

- [ ] **Step 6: Add conflict count badge**

Fetch `/life-calendar/conflicts/count` and display on the calendar dock icon. Update via SSE events.

- [ ] **Step 7: Add manual refresh button**

Refresh icon in panel header. Click → `POST /life-calendar/sync`. Show spinner, refresh panel on completion.

- [ ] **Step 8: Build and verify**

Run: `npm run build --prefix life-ui`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#142): calendar panel conflict indicators, resolution UI, SSE integration Refs #142"
```

---

## References

- [2026-10-31-calendar-conflict-detection-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/life/api/LifeCalendarMapping.java` — existing mapping record
- `api/src/main/java/io/casehub/life/api/LifeCalendarMappingStore.java` — existing mapping SPI
- `app/src/main/java/io/casehub/life/app/entity/JpaLifeCalendarMapping.java` — existing JPA entity
- `app/src/main/java/io/casehub/life/app/service/LifeCalendarSyncService.java` — existing outbound sync
- `app/src/main/java/io/casehub/life/app/observer/LifeCalendarOutboundObserver.java` — existing outbound observer
- `app/src/main/java/io/casehub/life/app/ledger/HealthDecisionLedgerEntry.java` — ledger entry pattern reference
- `app/src/main/java/io/casehub/life/app/observer/LifeDecisionLedgerObserver.java` — CDI observer pattern
- `app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java` — Quartz job pattern
- `app/src/main/java/io/casehub/life/app/event/LifeEventType.java` — SSE event types
- `app/src/main/java/io/casehub/life/app/event/LifeEventBroadcaster.java` — SSE broadcast
- `life-ui/src/panels/calendar-panel.ts` — existing calendar panel
- `io.casehub.connectors.calendar.spi.EventTiming` — sealed timing interface
- casehubio/life#114 — prerequisite calendar sync
- casehubio/life#142 — focal issue
