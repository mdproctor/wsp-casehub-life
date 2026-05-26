# Design Journal — issue-2-layer1-naive-domain

## 2026-05-26 — Layer 1 implementation complete

### What was built

Full Layer 1 domain baseline:

- `api/`: LifeDomain, LifeActorType, LifeCapabilities, LifeTrustDimensions, HouseholdTaskStatus, LifeGoalStatus
- `app/entity/`: ExternalActor, HouseholdTask, LifeGoal, LifeEvent (Panache Active Record)
- `app/service/`: four services, transaction ownership at service layer, NotFoundException thrown from delete()
- `app/resource/`: four @Blocking @ApplicationScoped resources with @Valid bodies
- Flyway V100–V103 for production
- 51 tests: api/ pure-Java, app/ @QuarkusTest, ShowcaseScenarioTest narrative

### Key design decisions made this session

**LifeActorType naming.** `ActorType` would collide with `io.casehub.platform.api.identity.ActorType` (HUMAN/AGENT/SYSTEM) once foundation modules activate. Named `LifeActorType` to eliminate import conflicts in later layers.

**externalActorId as raw UUID, no @ManyToOne.** Raw UUID column on HouseholdTask consistent with clinical's `AdverseEvent.enrollmentId` pattern. Avoids cascade decisions before the Store SPI pattern arrives in Layer 2, and avoids ORM entanglement. Future FK constraint can be a Flyway migration.

**No DB FK constraint.** The 409 guard (refuse ExternalActor delete when tasks reference it) is enforced in `ExternalActorService.delete()` within a single @Transactional boundary. No race: check-and-delete in one transaction.

**H2 drop-and-create in tests, not Flyway.** `casehub-engine-persistence-hibernate` puts `V1.0.0__Create_Quartz_Tables.sql` at `classpath:db/migration` — same classpath path as casehub-work's `V1__initial_schema.sql`. Flyway treats both as version 1 and fails. Solution: `database.generation=drop-and-create` for both PUs in test config, following clinical's validated pattern. Flyway migrations are not tested in @QuarkusTest; they run against real Postgres in production.

**@Blocking on all resources.** quarkus-rest (RESTEasy Reactive) runs on the I/O thread. JDBC Panache blocks. Without `@Blocking`, event loop thread is blocked silently (no exception in tests, but will degrade under load). Applied at class level to all four resources.

**@Transactional at service layer only.** Resources have no @Transactional. Services own the transaction boundary. delete() methods throw NotFoundException (404) or ClientErrorException (409) rather than returning boolean — cleaner exception semantics, eliminates TOCTOU.

### Deferred decisions (tracked as issues)

- life#10: ExternalActor actorId string convention for LedgerEntry integration (Layer 4)
- life#11: ExternalActor trust dimension score fields, Beta(α,β) per dimension (Layer 6)
- life#12: DTO layer — api/ response records, resources stop returning JPA entities (Layer 2)
- life#13: casehub-engine-persistence-memory scoped compile vs test — verify scaffold intent

### Test config pattern discovered

Two-PU H2 config for Layer 1 @QuarkusTest (canonical reference for subsequent layers):

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

### Spec

`docs/specs/2026-05-26-layer1-domain-baseline.md` — full field-level spec with all entity shapes, Flyway V numbers, REST endpoints, test classes.
