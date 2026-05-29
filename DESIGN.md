# casehub-life Design

Living design record. Updated at each layer close.

---

## Layer 1 — Domain Baseline (2026-05-26)

**Spec:** `docs/specs/2026-05-26-layer1-domain-baseline.md` in project repo — full field-level detail.

### What was built

- `api/`: LifeDomain, LifeActorType, LifeCapabilities, LifeTrustDimensions, HouseholdTaskStatus, LifeGoalStatus
- `app/entity/`: ExternalActor, HouseholdTask, LifeGoal, LifeEvent (Panache Active Record)
- `app/service/`: four services, transaction ownership at service layer, NotFoundException thrown from delete()
- `app/resource/`: four @Blocking @ApplicationScoped resources with @Valid bodies
- Flyway V100–V103 for production

### Design decisions

**LifeActorType naming.** `ActorType` would collide with `io.casehub.platform.api.identity.ActorType` (HUMAN/AGENT/SYSTEM) once foundation modules activate. Named `LifeActorType` to eliminate import conflicts in later layers.

**externalActorId as raw UUID, no @ManyToOne.** Raw UUID column on HouseholdTask consistent with clinical's `AdverseEvent.enrollmentId` pattern. Avoids cascade decisions before the Store SPI pattern arrives in Layer 2. Future FK constraint can be a Flyway migration.

**No DB FK constraint.** The 409 guard (refuse ExternalActor delete when tasks reference it) is enforced in `ExternalActorService.delete()` within a single @Transactional boundary. Check-and-delete in one transaction eliminates the TOCTOU race.

**H2 drop-and-create in tests, not Flyway.** `casehub-engine-persistence-hibernate` puts `V1.0.0__Create_Quartz_Tables.sql` at `classpath:db/migration`, colliding with casehub-work's `V1__initial_schema.sql` (Flyway treats V1.0.0 and V1 as the same version). Solution: `database.generation=drop-and-create` for both PUs in test config (clinical's validated pattern). Flyway runs in production only.

**@Blocking on all resources.** quarkus-rest (RESTEasy Reactive) runs on the I/O thread. JDBC Panache blocks. Without `@Blocking`, the event loop thread is blocked silently — no exception in tests, degrades under load. Applied at class level. Protocol: PP-20260526-d0b921.

**@Transactional at service layer only.** Resources have no @Transactional. Services own the transaction boundary. delete() methods throw NotFoundException (404) or ClientErrorException (409) — cleaner exception semantics, eliminates TOCTOU. Protocol: PP-20260526-75d9c9.

### Test configuration

Two-PU H2 config for @QuarkusTest (canonical reference for subsequent layers):

```properties
# Default PU — life domain + casehub-work entities
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:life_${surefire.forkNumber:0};MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.packages=io.casehub.work.runtime.model,io.casehub.work.runtime.filter,io.casehub.life.app.entity
quarkus.hibernate-orm.database.generation=drop-and-create

# Qhorus PU — qhorus + ledger entities
quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:qhorus_${surefire.forkNumber:0};MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm."qhorus".datasource=qhorus
quarkus.hibernate-orm."qhorus".packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime.model,io.casehub.ledger.model
quarkus.hibernate-orm."qhorus".database.generation=drop-and-create

# Both reactive suppressed — JDBC-only
quarkus.datasource.reactive=false
quarkus.datasource.qhorus.reactive=false
```

### Deferred decisions

| Issue | Layer | What |
|-------|-------|------|
| life#10 | Layer 4 | ExternalActor actorId string convention for LedgerEntry integration |
| life#11 | Layer 6 | ExternalActor trust dimension score fields (Beta α,β per dimension) |
| life#12 | Layer 2 | DTO layer — api/ response records, resources stop returning JPA entities |
| life#13 | — | casehub-engine-persistence-memory compile vs test scope — verify scaffold intent |

---

## Layer 3 — casehub-qhorus Commitment Lifecycle (2026-05-29)

**Spec:** `docs/specs/2026-05-29-layer3-qhorus-commitment.md` in project repo.

### What was built

- `LifeCommitmentRecord` — supplement to qhorus native `Commitment`, keyed by `correlationId`; tracks DELEGATION/CONTRACTOR/OVERSIGHT mode and status. `workItemId` null for OVERSIGHT until RESPONSE.
- `LifeCommitmentStrategy` SPI — sealed context hierarchy (`DelegationContext`, `ContractorContext`, `OversightContext`); three `@ApplicationScoped` implementations; CDI `Instance<T>` dispatch with exactness assertion.
- `LifeChannelInitializer` — creates `life/delegation`, `life/oversight` (COMMAND+RESPONSE only), `life/actor/{id}` channels at startup; one APPROVAL_PENDING Watchdog per channel (`thresholdSeconds=0`).
- `LifeOversightResponseObserver` (MessageObserver) — RESPONSE on oversight channel triggers WorkItem creation and FULFILLED status.
- `LifeWatchdogAlertObserver` (@ObservesAsync) — queries expired PENDING_RESPONSE records by `event.notificationChannel()` name.
- REST: `POST /life-tasks/{id}/commit`, `POST /life-oversight-gates`, `GET /life-tasks/{id}`.
- Flyway V103 (`life_commitment_record`), V104 (`life-escalation` WorkItemTemplate seed).

### Design decisions

**LifeCommitmentStrategy SPI in `app/`, not `api/`.** Context types reference JPA entities (`WorkItem`, `LifeTaskContext`, `ExternalActor`). Placing in `api/` creates a circular Maven dependency. No external consumers — CDI `Instance<LifeCommitmentStrategy>` collects all three implementations.

**Oversight gate: no WorkItem until RESPONSE.** A pending financial decision is not a WorkItem. `OversightGateStrategy` persists `LifeCommitmentRecord{workItemId=null, pendingTaskJson=...}` and dispatches COMMAND. `LifeOversightResponseObserver` creates the WorkItem when RESPONSE arrives.

**`delegateTo` reused as dedup key for OVERSIGHT.** OVERSIGHT has no `delegateTo` in domain sense; the column is repurposed as a `title:templateRef` hash for duplicate gate detection. Mitigated by a partial unique index on `(delegate_to) WHERE mode='OVERSIGHT' AND status='PENDING_RESPONSE'`. A dedicated `oversight_key` column would be cleaner — deferred as known trade-off.

**Life channels use `life/` namespace prefix, not normative mesh layout.** The normative 3-channel layout (`/work`, `/observe`, `/oversight` suffix convention) governs Claudony's agent orchestration channels. Life household channels are domain coordination channels with independent naming. See PP-20260529-e30ebd.

**`LifeOversightResponseObserver` uses `@Transactional(REQUIRES_NEW)`.** `MessageService.dispatch()` calls MessageObserver synchronously inside the qhorus dispatch transaction; the observer suspends it and opens its own for atomic WorkItem creation + record update.

**`LifeWatchdogAlertObserver` uses `@ObservesAsync`.** qhorus fires `WatchdogAlertEvent` via `fireAsync()`. Only `@ObservesAsync` observers receive async events — switching to `@Observes` silently breaks delivery.

### Open issues

| Issue | What |
|-------|------|
| life#17 | `LifeWatchdogAlertObserver` escalation integration test — `@ObservesAsync` timing makes Awaitility unreliable |
| life#18 | REST resource consistency — `@Produces/@Consumes` on all resources, 201 for commitment creation |
