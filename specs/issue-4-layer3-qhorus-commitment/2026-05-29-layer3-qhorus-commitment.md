# Layer 3: casehub-qhorus Commitment Lifecycle

**Branch:** issue-4-layer3-qhorus-commitment  
**Issue:** casehubio/life#4  
**Date:** 2026-05-29  
**Status:** Approved

---

## Purpose

Layer 3 adds casehub-qhorus to casehub-life, introducing formal commitment tracking for three household accountability patterns:

| Pattern | What it solves |
|---------|---------------|
| Family delegation | Task assigned to household member → COMMAND with Watchdog; silent failure replaced by tracked obligation |
| Contractor follow-up | External actor commits to time/date → COMMAND + Watchdog; no-show triggers escalation WorkItem |
| Oversight gate | Major decision requires approval → COMMAND to oversight channel; WorkItem created only after household-admin RESPONSE |

**Tutorial contrast (OpenClaw alone vs casehub-life Layer 3):**

| Scenario | OpenClaw alone | + Layer 3 |
|----------|---------------|-----------|
| School pickup delegated | Agent says it'll chase — silent if not | COMMAND to household-member, Watchdog at 3pm |
| Plumber commits Thursday | Agent tracks in memory, may forget | COMMAND on actor channel, Watchdog at window close |
| Major purchase approval | Best-effort research, no gate | COMMAND to oversight; WorkItem not created until RESPONSE |

---

## Infrastructure Already in Place (from Layer 2)

- `casehub-qhorus` runtime dep in `app/pom.xml`
- `casehub-qhorus-testing` test dep in `app/pom.xml`
- Qhorus named datasource configured in both `application.properties` and `src/test/resources/application.properties`
- `quarkus.datasource.reactive=false` / `quarkus.datasource.qhorus.reactive=false` in both property files (GE-20260508-492336)
- `classpath:db/qhorus/migration,classpath:db/ledger/migration` in Flyway qhorus locations

---

## SPI Contract — `api/` Module

All types are pure Java. No framework imports.

```java
package io.casehub.life.api.spi;

public interface LifeCommitmentStrategy {
    boolean applies(CommitmentContext context);
    CommitmentOutcome execute(CommitmentContext context);
}
```

```java
package io.casehub.life.api.commitment;

public record CommitmentContext(
    CommitmentRequest  request,
    WorkItem           workItem,      // null for OVERSIGHT — no task exists yet
    LifeTaskContext    taskContext,   // null for OVERSIGHT
    ExternalActor      externalActor // null for DELEGATION / OVERSIGHT
) {}

public record CommitmentRequest(
    String                 delegateTo,      // DELEGATION: principal id
    UUID                   externalActorId, // CONTRACTOR: actor to hold to commitment
    Instant                deadline,
    boolean                oversightRequired,
    CreateLifeTaskRequest  pendingTask      // OVERSIGHT: task to create on RESPONSE
) {}

public record CommitmentOutcome(
    UUID             recordId,
    String           correlationId,
    CommitmentMode   mode,
    CommitmentStatus status
) {}

public enum CommitmentMode   { DELEGATION, CONTRACTOR, OVERSIGHT }
public enum CommitmentStatus { PENDING_RESPONSE, FULFILLED, FAILED, EXPIRED }
```

`LifeTaskResponse` extended with:
```java
CommitmentMode   commitmentMode    // null if no commitment on this task
CommitmentStatus commitmentStatus  // null if no commitment on this task
```

---

## Data Model — `app/` Module

### JPA Entity: `LifeCommitmentRecord`

```java
@Entity
@Table(name = "life_commitment_record")
class LifeCommitmentRecord extends PanacheEntityBase {
    @Id
    public UUID id;

    @Column(name = "correlation_id", nullable = false, unique = true)
    public String correlationId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public CommitmentMode mode;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public CommitmentStatus status;

    @Column(name = "work_item_id")
    public UUID workItemId;         // null for OVERSIGHT until RESPONSE

    @Column(name = "external_actor_id")
    public UUID externalActorId;    // CONTRACTOR only

    @Column(name = "delegate_to")
    public String delegateTo;       // DELEGATION only

    @Column(name = "channel_id", nullable = false)
    public String channelId;

    public Instant deadline;

    @Column(name = "pending_task_json", columnDefinition = "TEXT")
    public String pendingTaskJson;  // OVERSIGHT only — serialized CreateLifeTaskRequest
}
```

No foreign key to `work_item` — cross-datasource coupling avoided. `LifeCommitmentRecord` is a supplement, not an owner.

### Flyway: `V103__life_commitment_record.sql`

```sql
CREATE TABLE life_commitment_record (
    id                UUID         NOT NULL,
    correlation_id    VARCHAR(255) NOT NULL,
    mode              VARCHAR(32)  NOT NULL,
    status            VARCHAR(32)  NOT NULL,
    work_item_id      UUID,
    external_actor_id UUID,
    delegate_to       VARCHAR(255),
    channel_id        VARCHAR(255) NOT NULL,
    deadline          TIMESTAMP WITH TIME ZONE,
    pending_task_json TEXT,
    CONSTRAINT pk_life_commitment_record PRIMARY KEY (id),
    CONSTRAINT uq_life_commitment_correlation UNIQUE (correlation_id)
);

CREATE INDEX idx_life_commitment_work_item   ON life_commitment_record (work_item_id);
CREATE INDEX idx_life_commitment_correlation ON life_commitment_record (correlation_id);
```

`idx_life_commitment_correlation` is the hot index — hit by `LifeOversightResponseObserver` on every RESPONSE message.

---

## Channel Topology

| Channel | Scope | `allowed_writers` |
|---------|-------|-------------------|
| `life/delegation` | Shared — all family delegation tasks | `household-admin`, `household-member` |
| `life/oversight` | Shared — all oversight gates | `household-admin` only |
| `life/actor/{externalActorId}` | Per-actor — one per ExternalActor | `household-admin`, `household-member` |

### `LifeChannelInitializer`

`@ApplicationScoped`, observes `StartupEvent`. Initializes both shared channels at startup. Per-actor channels created on-demand in `ContractorCommitmentStrategy`.

```java
@ApplicationScoped
class LifeChannelInitializer {

    static final String DELEGATION_CHANNEL = "life/delegation";
    static final String OVERSIGHT_CHANNEL  = "life/oversight";

    void onStart(@Observes StartupEvent ev) {
        ensureChannel(DELEGATION_CHANNEL, List.of("household-admin", "household-member"));
        ensureChannel(OVERSIGHT_CHANNEL,  List.of("household-admin"));
    }

    String ensureActorChannel(UUID externalActorId) {
        String id = "life/actor/" + externalActorId;
        ensureChannel(id, List.of("household-admin", "household-member"));
        return id;
    }

    private void ensureChannel(String channelId, List<String> allowedWriters) {
        // ChannelService.create() does NOT register in ChannelGateway (GE-20260526-5247f2).
        // Always call initChannel() after create().
        channelService.findById(channelId).ifPresentOrElse(
            ch -> channelGateway.initChannel(ch.id, new ChannelRef(ch.id, ch.name)),
            () -> {
                var ch = channelService.create(channelId, allowedWriters);
                channelGateway.initChannel(ch.id, new ChannelRef(ch.id, ch.name));
            }
        );
    }
}
```

`ensureChannel` is idempotent — restart-safe.

---

## Strategy Implementations

All three are `@ApplicationScoped` CDI beans. `LifeCommitmentService` collects them via `@Inject Instance<LifeCommitmentStrategy>`, finds first `applies()`, executes. Throws `IllegalStateException` if zero match; throws `IllegalStateException` if more than one match (exclusivity invariant).

### `DelegationCommitmentStrategy`

Applies when: `request.delegateTo() != null`

- Dispatches `COMMAND` on `life/delegation` with `target = delegateTo`, `correlationId`, `deadline`
- Registers Watchdog at `deadline`
- Persists `LifeCommitmentRecord{mode=DELEGATION, status=PENDING_RESPONSE, workItemId, delegateTo}`

### `ContractorCommitmentStrategy`

Applies when: `externalActor != null && request.deadline() != null`

- Calls `channelInitializer.ensureActorChannel(externalActor.id())` — creates channel on demand
- Dispatches `COMMAND` on `life/actor/{id}` with `correlationId`, `deadline`
- Registers Watchdog at `deadline`
- Persists `LifeCommitmentRecord{mode=CONTRACTOR, status=PENDING_RESPONSE, workItemId, externalActorId}`

### `OversightGateStrategy`

Applies when: `request.oversightRequired() == true`

- `workItem` and `taskContext` are null — no WorkItem exists yet
- Dispatches `COMMAND` on `life/oversight` with `correlationId`, `deadline`
- Registers Watchdog at `deadline`
- Serializes `request.pendingTask()` to JSON
- Persists `LifeCommitmentRecord{mode=OVERSIGHT, status=PENDING_RESPONSE, workItemId=null, pendingTaskJson}`

**Protocol invariants (PP-20260522-3dca14):** `COMMAND` has no required reply fields — builder accepts it without `inReplyTo`.

---

## Oversight Bridge — `LifeOversightResponseObserver`

Implements qhorus `MessageObserver` SPI. Fires on every message; guards narrow to RESPONSE on oversight channel.

```java
@ApplicationScoped
class LifeOversightResponseObserver implements MessageObserver {

    @Inject LifeTaskService lifeTaskService;
    @Inject LifeCommitmentRecordRepository records;
    @Inject ObjectMapper json;

    @Override
    public void onMessage(Message message) {
        // MessageObserver is application-wide — guard required (GE-20260517-f28d15)
        if (message.type() != RESPONSE) return;
        if (!OVERSIGHT_CHANNEL.equals(message.channelId())) return;
        if (message.correlationId() == null) return;

        records.findByCorrelationId(message.correlationId())
            .filter(r -> r.mode == OVERSIGHT && r.status == PENDING_RESPONSE)
            .ifPresent(record -> {
                CreateLifeTaskRequest pending = json.readValue(
                    record.pendingTaskJson, CreateLifeTaskRequest.class);
                lifeTaskService.createTask(pending); // creates WorkItem + LifeTaskContext

                record.status = FULFILLED;
                record.persist();
            });
    }
}
```

The RESPONSE from household-admin carries `correlationId` matching the original COMMAND. qhorus builder validation enforces `correlationId` on RESPONSE at `build()` time.

---

## Watchdog Alert Handling — `LifeWatchdogAlertObserver`

When a registered Watchdog deadline passes without a RESPONSE, qhorus fires a `WatchdogAlertEvent` via CDI `fireAsync()`. `LifeWatchdogAlertObserver` observes this event, looks up the relevant `LifeCommitmentRecord` by `correlationId`, and creates an escalation WorkItem.

```java
@ApplicationScoped
class LifeWatchdogAlertObserver {

    @Inject LifeTaskService  lifeTaskService;
    @Inject LifeCommitmentRecordRepository records;

    void onAlert(@ObservesAsync WatchdogAlertEvent event) {
        records.findByCorrelationId(event.correlationId())
            .filter(r -> r.status == PENDING_RESPONSE)
            .ifPresent(record -> {
                // Create escalation WorkItem for household-admin
                lifeTaskService.createEscalationTask(record);

                record.status = EXPIRED;
                record.persist();
            });
    }
}
```

**Per mode:**

| Mode | Escalation WorkItem title | Domain |
|------|--------------------------|--------|
| DELEGATION | "{delegateTo} has not confirmed — action required" | HOUSEHOLD |
| CONTRACTOR | "Contractor {actor.name} has not confirmed by deadline" | CONTRACTOR_COORDINATION |
| OVERSIGHT | "Oversight gate expired — request not approved" | Finance domain of pending task |

For OVERSIGHT with expired gate: the `LifeCommitmentRecord.pendingTaskJson` is discarded — the request lapses. No WorkItem is ever created for the gated task.

**Testing (from GE-20260414-fbf82f):** inject `WatchdogEvaluationService` directly in tests, call `evaluateAll()` to trigger the alert synchronously — no scheduler timing needed.

**Ordering note (GE-20260527-cad5ba):** qhorus fires `WatchdogAlertEvent` before internal channel dispatch. Our observer (`@ObservesAsync`) runs asynchronously — outside the qhorus transaction. Use `@Transactional(REQUIRES_NEW)` on the observer method to own its own transaction boundary.

---

## REST API

All resources: `@Blocking @ApplicationScoped` (PP-20260526-d0b921).  
`@Transactional` on service methods only, never resource methods (PP-20260526-75d9c9).

### `POST /life-tasks/{id}/commit`

Applies commitment to an existing task (DELEGATION or CONTRACTOR).

**Request:** `CommitmentRequest` — must have `delegateTo` XOR (`externalActorId` + `deadline`)  
**Response:** `CommitmentOutcome`  
**Error:** 422 if no strategy applies; 404 if task not found

### `POST /life-oversight-gates`

Creates an oversight gate before a task exists. OVERSIGHT only.

**Request:** `CommitmentRequest` with `oversightRequired=true`, `pendingTask`, `deadline`  
**Response:** `CommitmentOutcome{workItemId=null, mode=OVERSIGHT, status=PENDING_RESPONSE}`

### `GET /life-tasks/{id}` (extended)

`LifeTaskResponse` now includes `commitmentMode` and `commitmentStatus`. Populated by joining `LifeCommitmentRecord` on `workItemId`. Null if no commitment.

---

## Testing Strategy

### `LifeCommitmentStrategyTest` (unit, no Quarkus)

Verifies `applies()` routing:
- Each strategy applies to its own context and not to the others (exclusivity)
- No strategy applies to an empty `CommitmentRequest`

### `LifeCommitmentResourceTest` (`@QuarkusTest`)

REST integration with qhorus-testing in-memory stores:
- `POST /life-tasks/{id}/commit` with `delegateTo` → 200, `mode=DELEGATION`
- `POST /life-tasks/{id}/commit` with `externalActorId + deadline` → 200, `mode=CONTRACTOR`
- `POST /life-tasks/{id}/commit` with neither → 422
- `POST /life-oversight-gates` → 200, `mode=OVERSIGHT`, `workItemId=null`
- `GET /life-tasks/{id}` after commit → includes `commitmentMode` + `commitmentStatus`

### `LifeOversightResponseObserverTest` (`@QuarkusTest`)

Verifies the bridge:
1. Insert `LifeCommitmentRecord{mode=OVERSIGHT, PENDING_RESPONSE, pendingTaskJson=...}`
2. Dispatch RESPONSE with matching `correlationId` via `MessageService`
3. Assert `WorkItem` created (casehub-work store)
4. Assert `LifeCommitmentRecord.status == FULFILLED`

### `CommitmentLifecycleScenarioTest` (`@QuarkusTest`)

End-to-end showcase (mirrors `ShowcaseScenarioTest` from Layer 2):
- Contractor scenario: create task → `POST /life-tasks/{id}/commit` → assert COMMAND in `life/actor/{id}` → assert Watchdog registered
- Delegation scenario: create task → commit → assert COMMAND on `life/delegation` with correct target
- Oversight gate scenario: `POST /life-oversight-gates` → assert COMMAND on `life/oversight` → dispatch RESPONSE → assert WorkItem created

`@BeforeEach @Transactional` for `WorkItemTemplate` seeding (PP-20260528-913df2).

---

## Platform Coherence Notes

| Concern | Decision |
|---------|----------|
| `MessageService.dispatch()` | Single enforcement gate for all three strategies (PP-20260523-a08b97) |
| `ChannelService.create()` + `initChannel()` | Both called in `ensureChannel()` — GE-20260526-5247f2 |
| `MessageObserver` guard | channelId + type checked first — GE-20260517-f28d15 |
| `life_commitment_record` datasource | Default (life domain) — not qhorus datasource |
| No FK to `work_item` | Supplement pattern, not owner pattern |
| `@Transactional` placement | Service methods only |
| Reactive suppression | Already in both property files — GE-20260508-492336 |
| Strategy CDI injection | `Instance<LifeCommitmentStrategy>` — all three active simultaneously, no `@Alternative` needed |

---

## Deferred (Out of Scope for Layer 3)

| Item | Issue |
|------|-------|
| WhatsApp/SMS chase when Watchdog fires | Layer 7 (OpenClaw messaging skill) |
| Watchdog → escalation WorkItem wiring | casehub-qhorus handles Watchdog; escalation WorkItem is qhorus-native |
| `life-automation.md` layer table correction | life#16 |
| `casehub-life.md` in parent — Layer 3 complete | parent#96 (after merge) |
| RESPONSE validation in `LifeOversightResponseObserver` — verify household-admin role | Auth retrofit (not yet wired) |
