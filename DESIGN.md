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

---

## Layer 5 — casehub-engine CasePlanModel Workflows (2026-06-01)

**Spec:** `docs/specs/2026-05-31-layer5-casehub-engine-design.md` in project repo.

### What was built

- 8 `YamlCaseHub` subclasses + 8 fluent DSL companions in `io.casehub.life.app.engine`: appointment-cycle, home-maintenance, family-vote, travel-plan, contractor-coordination, care-coordination, care-episode, financial-review.
- 8 YAML case definitions at `app/src/main/resources/life/`.
- `LifeCaseService` — three-phase case start (PP-20260529-3ffe28): direct injection of each YamlCaseHub, switch on LifeCaseType.
- `LifeCaseTracker` — JPA entity tracking active engine cases by type for cross-case signal lookup.
- `LifeCaseTrackerObserver` — `@ObservesAsync CaseLifecycleEvent` updates tracker status.
- `LifeCaseResource` — `POST /life-cases` endpoint.
- `LifeCaseType` enum (api/): TRAVEL_PLAN, HOME_MAINTENANCE, CARE_COORDINATION, APPOINTMENT_CYCLE, CONTRACTOR_COORDINATION, FINANCIAL_REVIEW.
- `LifeCaseStatus` enum (api/): ACTIVE, COMPLETED, FAILED.
- Scope retrofit: WorkItem scope changed from `"life"` to `"casehubio/life/{domain}"` (hierarchical Path format).
- `LifeDecisionLedgerObserver` refactored: domain resolution now uses WorkItem scope Path (primary), LifeTaskContext (fallback).
- Flyway V107 (`life_case_tracker`).

### Design decisions

**All 8 case definitions in one layer.** AML did 1, clinical did 3. casehub-life does 8 to demonstrate the full breadth of engine capabilities in a single harness — parallel execution, adaptive gates, M-of-N SubCase quorum, cross-case signals, milestones, FuncDSL workers. Splitting across layers would have produced seven more issues and branch ceremonies for work sharing the same infrastructure.

**YAML + DSL pairing rule (PP-20260518).** Every YAML definition has a fluent DSL companion. Both are equal authoring paths. The DSL is not "the test version" — it produces the same CaseDefinition and is used for augmentation when YAML doesn't support a feature.

**SubCase M-of-N is DSL-only.** YAML schema doesn't support groupId/totalInGroup/requiredCount. Travel-plan's family-vote SubCase bindings are added via Java augmentation in `TravelPlanCaseHub.getDefinition()`.

**Scope retrofit to hierarchical format.** `casehubio/life/{domain}` enables `LifeDecisionLedgerObserver` to resolve domain from scope Path instead of requiring a LifeTaskContext supplement. Engine-created WorkItems produce correct ledger entries without supplements.

**Cross-case signals in workers, not observers.** `LifeCaseTrackerObserver` is pure infrastructure (status update). Domain cross-case logic (contractor-to-financial-review signal) goes in the completing worker itself. Matches clinical pattern.

**FuncDSL for workers (PP-20260531).** `FuncWorkflowBuilder.workflow().tasks(FuncDSL.function(...)).build()` — not raw lambdas. Each worker is a `Function<Map<String, Object>, Map<String, Object>>`.

### Blockers

| Issue | What |
|-------|------|
| engine#410 | `SchedulerService.getCaseDefinition()` returns null after successful registration — forward ConcurrentHashMap lookup fails. Integration tests `@Disabled` until fixed. |
