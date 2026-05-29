# Layer 3: casehub-qhorus Commitment Lifecycle

**Branch:** issue-4-layer3-qhorus-commitment  
**Issue:** casehubio/life#4  
**Date:** 2026-05-29  
**Status:** Approved (rev 2 — post code review)

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
| School pickup delegated | Agent says it'll chase — silent if not | COMMAND to household-member, Watchdog fires if no RESPONSE by deadline |
| Plumber commits Thursday | Agent tracks in memory, may forget | COMMAND on actor channel, Watchdog fires at window close |
| Major purchase approval | Best-effort research, no gate | COMMAND to oversight; WorkItem not created until RESPONSE |

---

## Infrastructure Already in Place (from Layer 2)

- `casehub-qhorus` runtime dep in `app/pom.xml`
- `casehub-qhorus-testing` test dep in `app/pom.xml`
- Qhorus named datasource in both `application.properties` and test properties
- `quarkus.datasource.reactive=false` / `quarkus.datasource.qhorus.reactive=false` (GE-20260508-492336)
- `classpath:db/qhorus/migration,classpath:db/ledger/migration` in Flyway qhorus locations

---

## Qhorus Commitment Auto-Creation

`MessageService.dispatch()` auto-creates a native qhorus `Commitment` when `type=COMMAND && correlationId != null`. The `Commitment.expiresAt` is populated from `MessageDispatch.deadline`. `LifeCommitmentRecord.correlationId` is the supplement link to this native Commitment — the same pattern as `LifeTaskContext.workItemId` linking to the foundation `WorkItem`.

This means: the life layer generates a `correlationId` (`UUID.randomUUID().toString()`) before dispatch, sets it on the `MessageDispatch`, and uses it as the linking key for everything downstream (Commitment lookup, Watchdog evaluation, observer matching).

---

## Watchdog Mechanism

Watchdogs are condition-based, not deadline-based. The relevant condition type for commitment tracking is `APPROVAL_PENDING`. `WatchdogEvaluationService.evaluateApprovalPending()` queries `CommitmentStore` for open COMMAND Commitments whose `expiresAt` has passed and fires `WatchdogAlertEvent`.

Workflow:
1. Strategy dispatches `COMMAND` with `correlationId` and `deadline` → qhorus auto-creates `Commitment{expiresAt=deadline, state=OPEN}`
2. Strategy registers `Watchdog{conditionType=APPROVAL_PENDING, notificationChannel=channelId}` via `WatchdogStore.put()`
3. `WatchdogScheduler` runs every 60s, calls `evaluateApprovalPending()` → detects expired open Commitments on watched channel → fires `WatchdogAlertEvent`
4. `LifeWatchdogAlertObserver` observes the event, looks up `LifeCommitmentRecord` by `correlationId`, creates escalation WorkItem

`WatchdogStore` is blocking (`put(Watchdog)` synchronous). `ReactiveWatchdogService` exists but is not needed here.

---

## SPI Contract and Context — `app/` Module

`LifeCommitmentStrategy` and its context types live in `app/` (not `api/`), because:
- `LifeTaskContext` is a JPA entity defined in `app/` — placing it in `api/` creates a circular Maven dependency
- `WorkItem` carries JPA annotations — it cannot be in the zero-framework `api/` module

These are internal app-layer strategy classes with no external consumers. CDI `Instance<LifeCommitmentStrategy>` collects all registered implementations.

```java
// app/ — package-private internal SPI
interface LifeCommitmentStrategy {
    boolean applies(CommitmentContext context);
    CommitmentOutcome execute(CommitmentContext context);
}
```

**Sealed context hierarchy** — eliminates null-field documentation and NPE risk:

```java
sealed interface CommitmentContext
    permits DelegationContext, ContractorContext, OversightContext {}

record DelegationContext(
    CommitmentRequest request,
    WorkItem          workItem,
    LifeTaskContext   taskContext
) implements CommitmentContext {}

record ContractorContext(
    CommitmentRequest request,
    WorkItem          workItem,
    LifeTaskContext   taskContext,
    ExternalActor     externalActor
) implements CommitmentContext {}

record OversightContext(
    OversightGateRequest request   // no WorkItem — none exists yet
) implements CommitmentContext {}
```

**Two request types — one per endpoint:**

```java
// for POST /life-tasks/{id}/commit
record CommitmentRequest(
    String  delegateTo,       // DELEGATION: principal id (required if not contractor)
    UUID    externalActorId,  // CONTRACTOR: actor to hold to commitment
    Instant deadline          // required for both modes
) {}

// for POST /life-oversight-gates
record OversightGateRequest(
    String                deadline_iso,  // Instant — when oversight expires
    CreateLifeTaskRequest pendingTask    // task to create on RESPONSE
) {}
```

**Outcome:**

```java
record CommitmentOutcome(
    UUID             recordId,
    String           correlationId,
    CommitmentMode   mode,
    CommitmentStatus status
) {}

enum CommitmentMode   { DELEGATION, CONTRACTOR, OVERSIGHT }
enum CommitmentStatus { PENDING_RESPONSE, FULFILLED, FAILED, EXPIRED }
```

`LifeTaskResponse` extended with `CommitmentMode commitmentMode` and `CommitmentStatus commitmentStatus` (null if no commitment).

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
    public String correlationId;        // links to qhorus Commitment.correlationId

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public CommitmentMode mode;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public CommitmentStatus status;

    @Column(name = "work_item_id")
    public UUID workItemId;             // null for OVERSIGHT until RESPONSE fulfills gate

    @Column(name = "external_actor_id")
    public UUID externalActorId;        // CONTRACTOR only

    @Column(name = "delegate_to")
    public String delegateTo;           // DELEGATION only

    @Column(name = "channel_id", nullable = false)
    public String channelId;

    public Instant deadline;

    @Column(name = "pending_task_json", columnDefinition = "TEXT")
    public String pendingTaskJson;      // OVERSIGHT only — serialized CreateLifeTaskRequest

    @Column(name = "created_at", nullable = false)
    public Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    public Instant updatedAt;
}
```

No foreign key to `work_item` — cross-datasource coupling avoided.

### Flyway: `V103__life_commitment_record.sql`

```sql
CREATE TABLE life_commitment_record (
    id                UUID                     NOT NULL,
    correlation_id    VARCHAR(255)             NOT NULL,
    mode              VARCHAR(32)              NOT NULL,
    status            VARCHAR(32)              NOT NULL,
    work_item_id      UUID,
    external_actor_id UUID,
    delegate_to       VARCHAR(255),
    channel_id        VARCHAR(255)             NOT NULL,
    deadline          TIMESTAMP WITH TIME ZONE,
    pending_task_json TEXT,
    created_at        TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at        TIMESTAMP WITH TIME ZONE NOT NULL,
    CONSTRAINT pk_life_commitment_record PRIMARY KEY (id),
    CONSTRAINT uq_life_commitment_correlation UNIQUE (correlation_id)
);

CREATE INDEX idx_life_commitment_work_item   ON life_commitment_record (work_item_id);
CREATE INDEX idx_life_commitment_correlation ON life_commitment_record (correlation_id);
```

---

## Channel Topology

Life domain channels are application-specific coordination channels, distinct from the qhorus normative 3-channel mesh (`/work`, `/observe`, `/oversight` suffix convention). The normative layout applies to agent orchestration channels managed by `NormativeChannelLayout` SPI (Claudony). Life channels serve household domain coordination and intentionally use domain-scoped names.

| Channel | Scope | `allowedWriters` | `allowedTypes` |
|---------|-------|-----------------|----------------|
| `life/delegation` | Shared — all family delegation | `household-admin`, `household-member` | not restricted |
| `life/oversight` | Shared — all oversight gates | `household-admin` only | `COMMAND,RESPONSE` |
| `life/actor/{externalActorId}` | Per-actor | `household-admin`, `household-member` | not restricted |

### `LifeChannelInitializer`

```java
@ApplicationScoped
class LifeChannelInitializer {

    static final String DELEGATION_CHANNEL = "life/delegation";
    static final String OVERSIGHT_CHANNEL  = "life/oversight";

    void onStart(@Observes StartupEvent ev) {
        ensureChannel(DELEGATION_CHANNEL,
            List.of("household-admin", "household-member"), null);
        ensureChannel(OVERSIGHT_CHANNEL,
            List.of("household-admin"), "COMMAND,RESPONSE");
    }

    String ensureActorChannel(UUID externalActorId) {
        String id = "life/actor/" + externalActorId;
        ensureChannel(id, List.of("household-admin", "household-member"), null);
        return id;
    }

    private void ensureChannel(String name, List<String> writers, String allowedTypes) {
        // ChannelService.create() does NOT register in ChannelGateway (GE-20260526-5247f2).
        // Always call initChannel() after create or find.
        channelService.findByName(name).ifPresentOrElse(
            ch -> channelGateway.initChannel(ch.id, new ChannelRef(ch.id, ch.name)),
            () -> {
                String writersStr = String.join(",", writers);
                var ch = channelService.create(
                    name, name, ChannelSemantic.APPEND,
                    writersStr, null, allowedTypes);
                channelGateway.initChannel(ch.id, new ChannelRef(ch.id, ch.name));
            }
        );
    }
}
```

`ensureChannel` is idempotent. Oversight channel enforces `allowedTypes = "COMMAND,RESPONSE"` for machine-checkable normative enforcement.

---

## Strategy Implementations

`LifeCommitmentService` dispatches to strategies:

```java
@ApplicationScoped
@Transactional
class LifeCommitmentService {

    @Inject @All List<LifeCommitmentStrategy> strategies;

    public CommitmentOutcome applyCommitment(CommitmentContext context) {
        List<LifeCommitmentStrategy> matched = strategies.stream()
            .filter(s -> s.applies(context))
            .toList();
        if (matched.isEmpty())
            throw new IllegalArgumentException("No strategy applies to context: " + context);
        if (matched.size() > 1)
            throw new IllegalStateException("Ambiguous strategies for context: " + matched);
        return matched.get(0).execute(context);
    }
}
```

### `DelegationCommitmentStrategy`

Applies when: `context instanceof DelegationContext dc && dc.request().delegateTo() != null && dc.request().deadline() != null`

```java
@ApplicationScoped
class DelegationCommitmentStrategy implements LifeCommitmentStrategy {

    @Override
    public CommitmentOutcome execute(CommitmentContext ctx) {
        var dc = (DelegationContext) ctx;
        String correlationId = UUID.randomUUID().toString();

        messageService.dispatch(MessageDispatch.builder(
                channelInitializer.ensureChannel(DELEGATION_CHANNEL, ...),
                currentPrincipal.actorId(), COMMAND,
                "Task delegation: " + dc.workItem().title())
            .correlationId(correlationId)
            .target(dc.request().delegateTo())
            .deadline(dc.request().deadline())
            .build());

        // Auto-created qhorus Commitment{expiresAt=deadline} enables APPROVAL_PENDING
        watchdogStore.put(new Watchdog(APPROVAL_PENDING, DELEGATION_CHANNEL,
            dc.request().deadline()));

        var record = new LifeCommitmentRecord();
        record.id = UUID.randomUUID();
        record.correlationId = correlationId;
        record.mode = DELEGATION;
        record.status = PENDING_RESPONSE;
        record.workItemId = dc.workItem().id();
        record.delegateTo = dc.request().delegateTo();
        record.channelId = DELEGATION_CHANNEL;
        record.deadline = dc.request().deadline();
        record.createdAt = record.updatedAt = Instant.now();
        record.persist();

        return new CommitmentOutcome(record.id, correlationId, DELEGATION, PENDING_RESPONSE);
    }
}
```

### `ContractorCommitmentStrategy`

Applies when: `context instanceof ContractorContext cc && cc.request().deadline() != null`

- `ensureActorChannel(externalActor.id())` — creates per-actor channel on demand
- Dispatches COMMAND with `correlationId` + `deadline` — qhorus auto-creates `Commitment{expiresAt=deadline}`
- Registers `APPROVAL_PENDING` Watchdog on actor channel
- Persists `LifeCommitmentRecord{mode=CONTRACTOR, externalActorId}`

### `OversightGateStrategy`

Applies when: `context instanceof OversightContext`

- No WorkItem — none exists yet; `LifeCommitmentRecord.workItemId = null` until RESPONSE
- Dispatches COMMAND to `life/oversight` with `correlationId` + `deadline`
- qhorus auto-creates `Commitment{expiresAt=deadline}` on oversight channel
- Registers `APPROVAL_PENDING` Watchdog on oversight channel
- Serializes `pendingTask` to JSON via ObjectMapper
- Persists `LifeCommitmentRecord{mode=OVERSIGHT, workItemId=null, pendingTaskJson}`

**Duplicate gate guard:** before dispatch, query `LifeCommitmentRecord` for any `mode=OVERSIGHT, status=PENDING_RESPONSE` with matching `pendingTask` identity. If found, return 409 Conflict. (Note: exact deduplication key for `pendingTask` is hash of title+domain — documented as best-effort; full uniqueness enforcement is a later-layer concern.)

**Protocol invariants (PP-20260522-3dca14):** COMMAND requires no reply fields — builder accepts without `inReplyTo`. ✓

---

## Oversight Bridge — `LifeOversightResponseObserver`

```java
@ApplicationScoped
class LifeOversightResponseObserver implements MessageObserver {

    @Inject LifeTaskService lifeTaskService;
    @Inject LifeCommitmentRecordRepository records;
    @Inject ObjectMapper json;

    @Override
    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void onMessage(Message message) {
        // MessageObserver is application-wide — channel + type guards required
        // (GE-20260517-f28d15: InboundNormaliser caveat applies equally to MessageObserver)
        if (message.type() != RESPONSE) return;
        if (!OVERSIGHT_CHANNEL.equals(message.channelId())) return;
        if (message.correlationId() == null) return;

        records.findByCorrelationId(message.correlationId())
            .filter(r -> r.mode == OVERSIGHT && r.status == PENDING_RESPONSE)
            .ifPresent(record -> {
                try {
                    CreateLifeTaskRequest pending = json.readValue(
                        record.pendingTaskJson, CreateLifeTaskRequest.class);
                    LifeTaskResponse created = lifeTaskService.createTask(pending);
                    record.workItemId = created.workItemId();  // populated after creation
                    record.status = FULFILLED;
                    record.updatedAt = Instant.now();
                    record.persist();
                } catch (JsonProcessingException e) {
                    // Log and skip — oversight request JSON was corrupted at gate creation
                    log.errorf(e, "Failed to deserialize pendingTaskJson for correlationId %s",
                        message.correlationId());
                }
            });
    }
}
```

RESPONSE carries `correlationId` matching the original COMMAND — qhorus builder validation enforces this at `build()` time.

---

## Watchdog Alert Handling — `LifeWatchdogAlertObserver`

```java
@ApplicationScoped
class LifeWatchdogAlertObserver {

    @Inject LifeTaskService  lifeTaskService;
    @Inject LifeCommitmentRecordRepository records;
    @Inject ObjectMapper json;

    @ObservesAsync
    @Transactional(Transactional.TxType.REQUIRES_NEW)
    void onAlert(WatchdogAlertEvent event) {
        records.findByCorrelationId(event.correlationId())
            .filter(r -> r.status == PENDING_RESPONSE)
            .ifPresent(record -> {
                createEscalationTask(record);
                record.status = EXPIRED;
                record.updatedAt = Instant.now();
                record.persist();
            });
    }

    private void createEscalationTask(LifeCommitmentRecord record) {
        String title = switch (record.mode) {
            case DELEGATION -> record.delegateTo + " has not confirmed — action required";
            case CONTRACTOR -> "Contractor has not confirmed by deadline";
            case OVERSIGHT  -> "Oversight gate expired — request not approved";
        };

        LifeDomain domain = switch (record.mode) {
            case DELEGATION -> LifeDomain.HOUSEHOLD;
            case CONTRACTOR -> LifeDomain.CONTRACTOR_COORDINATION;
            case OVERSIGHT  -> oversightDomain(record);
        };

        lifeTaskService.createTask(new CreateLifeTaskRequest(
            title, domain, null, WorkItemPriority.HIGH, null));
        // Escalation WorkItem uses the ESCALATION WorkItemTemplate
        // (seeded at V102 alongside the existing templates)
    }

    private LifeDomain oversightDomain(LifeCommitmentRecord record) {
        try {
            CreateLifeTaskRequest pending = json.readValue(
                record.pendingTaskJson, CreateLifeTaskRequest.class);
            return pending.domain();
        } catch (JsonProcessingException e) {
            log.warn("Could not deserialize pendingTaskJson for oversight domain — defaulting to FINANCE");
            return LifeDomain.FINANCE;
        }
    }
}
```

**Per mode:**

| Mode | Escalation WorkItem title | Domain |
|------|--------------------------|--------|
| DELEGATION | "{delegateTo} has not confirmed — action required" | HOUSEHOLD |
| CONTRACTOR | "Contractor has not confirmed by deadline" | CONTRACTOR_COORDINATION |
| OVERSIGHT | "Oversight gate expired — request not approved" | From `pendingTask.domain()` |

For OVERSIGHT expired gate: `pendingTaskJson` is not acted upon — the data remains in the column. Not "discarded." Layer 4 GDPR note: `pendingTaskJson` may contain personal-preference data (e.g. purchase details) for an expired gate; GDPR Art.17 erasure scope should include this column.

`@Transactional(REQUIRES_NEW)` on the observer method owns its own transaction boundary — the observer runs asynchronously outside the qhorus Watchdog transaction (GE-20260527-cad5ba).

---

## REST API

All resources: `@Blocking @ApplicationScoped` (PP-20260526-d0b921).  
`@Transactional` on service methods only, never resource methods (PP-20260526-75d9c9).

### `POST /life-tasks/{id}/commit`

Applies commitment to an existing task (DELEGATION or CONTRACTOR).

Pre-conditions checked in service before building context:
- Task exists → 404 if not
- `externalActorId` exists if provided → 422 if not found (consistent with `LifeTaskService.createTask()`)
- No existing `PENDING_RESPONSE` commitment on this `workItemId` → 409 Conflict if duplicate

**Request:** `CommitmentRequest` — must have `delegateTo` XOR (`externalActorId` + `deadline`)  
**Response:** `CommitmentOutcome`  
**Errors:** 404 task not found; 409 duplicate commitment; 422 invalid request or actor not found

### `POST /life-oversight-gates`

Creates a pre-approval gate before a task exists.

Pre-conditions: no existing `PENDING_RESPONSE` OVERSIGHT gate for matching `pendingTask` title+domain → 409 Conflict (best-effort dedup).

**Request:** `OversightGateRequest{pendingTask, deadline}`  
**Response:** `CommitmentOutcome{workItemId=null, mode=OVERSIGHT, status=PENDING_RESPONSE}`

### `GET /life-tasks/{id}` (extended)

`LifeTaskResponse` includes `commitmentMode` and `commitmentStatus`. Joined from `LifeCommitmentRecord` on `workItemId`. For OVERSIGHT: `workItemId` is populated only after RESPONSE fulfills the gate; before that, the commitment is not visible via this endpoint (by design — no task exists yet).

---

## Known Limitations

**Dual-datasource atomicity:** `MessageService.dispatch()` writes to the qhorus datasource; `LifeCommitmentRecord.persist()` writes to the life datasource. These are separate transactions. If `persist()` fails after `dispatch()` succeeds, a COMMAND is live in qhorus with no local tracking record. The Watchdog's `WatchdogAlertEvent` will fire, `LifeWatchdogAlertObserver.findByCorrelationId()` will return empty, and the escalation will be silently dropped. Proper fix is an outbox pattern (persist intended dispatch to life datasource first, publish from there) — deferred beyond Layer 3.

**Double oversight gate:** two `POST /life-oversight-gates` calls with the same `pendingTask` title+domain are deduped by a best-effort hash check. Two gates with different titles for logically identical requests can both RESPONSE and create two WorkItems. Full idempotency requires a richer deduplication key — deferred.

---

## Testing Strategy

### `LifeCommitmentStrategyTest` (unit, no Quarkus)

Verifies `applies()` routing via type-switching:
- `DelegationContext` → only DelegationStrategy applies
- `ContractorContext` → only ContractorStrategy applies
- `OversightContext` → only OversightStrategy applies
- Exactly one strategy applies per context type (exclusivity invariant)

### `LifeCommitmentResourceTest` (`@QuarkusTest`)

REST integration with qhorus-testing in-memory stores:
- `POST /life-tasks/{id}/commit` with `delegateTo + deadline` → 200, `mode=DELEGATION`
- `POST /life-tasks/{id}/commit` with `externalActorId + deadline` → 200, `mode=CONTRACTOR`
- `POST /life-tasks/{id}/commit` with neither → 422
- `POST /life-tasks/{id}/commit` twice → 409 Conflict
- `POST /life-tasks/{id}/commit` with unknown `externalActorId` → 422
- `POST /life-oversight-gates` → 200, `mode=OVERSIGHT`, `workItemId=null`
- `GET /life-tasks/{id}` after commit → includes `commitmentMode` + `commitmentStatus`

### `LifeOversightResponseObserverTest` (`@QuarkusTest`)

Verifies the bridge:
1. Insert `LifeCommitmentRecord{mode=OVERSIGHT, PENDING_RESPONSE, pendingTaskJson=...}`
2. Dispatch a RESPONSE with matching `correlationId` via `MessageService`
3. Assert `WorkItem` created (via casehub-work store)
4. Assert `LifeCommitmentRecord.status == FULFILLED`
5. Assert `LifeCommitmentRecord.workItemId` populated

### `CommitmentLifecycleScenarioTest` (`@QuarkusTest`)

End-to-end showcase (mirrors `ShowcaseScenarioTest` from Layer 2):

**Contractor scenario:**
- Create task → `POST /life-tasks/{id}/commit{externalActorId, deadline}` → assert COMMAND in `life/actor/{id}` → assert Watchdog registered in WatchdogStore

**Watchdog-to-escalation scenario (core accountability feature):**
- Create task → `POST /life-tasks/{id}/commit{delegateTo, deadline=now+1s}` → assert `LifeCommitmentRecord{PENDING_RESPONSE}`
- `watchdogEvaluationService.evaluateAll()` (injected directly — GE-20260414-fbf82f, no scheduler)
- Assert escalation WorkItem created
- Assert `LifeCommitmentRecord.status == EXPIRED`

**Oversight gate scenario:**
- `POST /life-oversight-gates{pendingTask, deadline}` → assert COMMAND on `life/oversight` → assert `workItemId=null`
- Dispatch RESPONSE with matching `correlationId` → assert WorkItem created → assert `LifeCommitmentRecord.workItemId` populated

`@BeforeEach @Transactional` for `WorkItemTemplate` seeding (PP-20260528-913df2).

---

## Platform Coherence Notes

| Concern | Decision |
|---------|----------|
| `MessageService.dispatch()` | Single enforcement gate for all strategies (PP-20260523-a08b97) |
| COMMAND auto-creates Commitment | Via `dispatch()` with `correlationId` — `Commitment.expiresAt` = dispatch deadline |
| `ChannelService.create()` + `initChannel()` | Both called in `ensureChannel()` — GE-20260526-5247f2 |
| `MessageObserver` guard | channelId + type + correlationId checked — GE-20260517-f28d15 |
| `life_commitment_record` datasource | Default (life domain) — not qhorus datasource |
| No FK to `work_item` | Supplement pattern — no cross-datasource FK |
| `@Transactional` placement | Service methods only; observer uses `REQUIRES_NEW` |
| Reactive suppression | Already in both property files — GE-20260508-492336 |
| Strategy CDI injection | `@All List<LifeCommitmentStrategy>` — collect-all with exactness assertion |
| SPI location | `app/` only — `api/` cannot reference JPA types from app/ (circular dep) |
| Channel allowedTypes | Oversight: `"COMMAND,RESPONSE"` — normative enforcement via `ChannelService.create()` |
| Watchdog registration | `WatchdogStore.put()` (blocking) with `conditionType=APPROVAL_PENDING` |

---

## Deferred (Out of Scope for Layer 3)

| Item | Issue / Notes |
|------|---------------|
| WhatsApp/SMS chase on Watchdog fire | Layer 7 (OpenClaw messaging skill) |
| Outbox pattern for dual-datasource atomicity | Layer 4+; documented as known limitation |
| Full oversight gate deduplication | Best-effort title+domain hash for now |
| GDPR: `pending_task_json` erasure scope | Layer 4 (GDPR Art.17) |
| `life-automation.md` layer table correction | life#16 |
| `casehub-life.md` in parent — Layer 3 complete | parent#96 (after merge) |
| Oversight RESPONSE role validation (household-admin only) | Auth retrofit — not yet wired |
