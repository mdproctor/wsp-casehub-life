# Handoff — casehub-life Layer 1 complete
2026-05-26

## Last Session

Layer 1 domain baseline designed and shipped. Design debate resolved
(ExternalActor FK as raw UUID, assignedTo as opaque String — both from
casehub-work source review + clinical pattern). Implemented TDD: 51
tests, 0 failures. Code review caught @Blocking, double @Transactional,
and TOCTOU on delete — all fixed. Flyway V1.0.0 conflict discovered
(casehub-engine-persistence-hibernate jar). Branch closed, issue #2
closed, pushed to casehubio/life main.

## Immediate Next Step

Start Layer 2: casehub-work SLA enforcement. Run `work-start` for life#3.

Layer 2 adds: WorkItem creation alongside HouseholdTask.persist(),
claimDeadline computed from slaHours, SlaBreachPolicy bean. Also the
right time to introduce the DTO layer (life#12 — resources stop returning
JPA entities).

## Design Decisions — Carry Forward

- `LifeActorType` (not `ActorType`) — avoids platform ActorType collision
- `HouseholdTask.externalActorId: UUID` — raw column, no @ManyToOne
- `@Blocking @ApplicationScoped` required on all REST resources (quarkus-rest + JDBC)
- `@Transactional` at service layer only — resources delegate
- H2 drop-and-create in tests — engine-persistence-hibernate V1.0.0 collides
  with casehub-work V1 at classpath:db/migration
- Test config template in `DESIGN.md` (workspace main)

## Deferred Issues

| Issue | Layer | What |
|-------|-------|------|
| life#10 | 4 | ExternalActor actorId string convention for ledger |
| life#11 | 6 | Trust dimension score fields on ExternalActor |
| life#12 | 2 | DTO layer — api/ response records |
| life#13 | — | engine-persistence-memory compile vs test scope |
| parent#76 | — | casehub-life.md Layer 1 status → complete |

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #3 | Layer 2: + casehub-work SLA enforcement | M | Med | Start here |
| #4 | Layer 3: + casehub-qhorus commitment lifecycle | M | Med | Blocked by #3 |
| #5 | Layer 4: + casehub-ledger tamper-evident audit | M | Med | Blocked by #4 |

## References

- Spec: `docs/specs/2026-05-26-layer1-domain-baseline.md`
- LAYER-LOG: `LAYER-LOG.md` (Layer 1 marked complete)
- Design: `DESIGN.md` (workspace main — Layer 1 decisions + test config)
- Blog: `blog/2026-05-26-mdp01-layer1-domain-baseline.md`
- Protocols: PP-20260526-d0b921 (@Blocking), PP-20260526-75d9c9 (@Transactional)
- Garden: GE-20260526-399a43 (@Blocking silent miss), REVISE GE-20260511-a28064
