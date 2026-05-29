# Handoff — casehub-life Layer 2 complete
2026-05-29

## Last Session

Layer 2 designed, implemented, reviewed, and closed. The brainstorming session
surfaced a fundamental domain model correction: `HouseholdTask`, `LifeGoal`, and
`LifeEvent` were removed as they duplicated foundation primitives. `LifeTaskContext`
(domain supplement) replaced them. `POST /life-tasks` creates WorkItem + LifeTaskContext
atomically. `LifeSlaBreachPolicy` enforces stateless two-tier escalation. 38 tests, 0 failures.
Branch squashed (20→7 commits), pushed to casehubio/life main.

## Immediate Next Step

Start Layer 3: casehub-qhorus commitment lifecycle. Run `work-start` for life#4.

Layer 3 adds: family delegation (COMMAND to household-member), contractor follow-up
(COMMAND + Watchdog), oversight gates for major financial decisions
(COMMAND to household-admin; no action until RESPONSE received).

## Cross-Module

**Blocked by:**
- `casehub-engine` — SNAPSHOT has broken internal package refactor (engine#379, engine#380).
  Layer 5 deps removed from pom.xml until fixed. Not blocking Layer 3.

## Design Decisions — Carry Forward

- `HouseholdTask`, `LifeGoal`, `LifeEvent` are GONE — they were foundation primitive wrappers
- Domain model: `ExternalActor` (JPA entity) + `LifeTaskContext` (supplement) only
- `WorkItemPriority` enum: no NORMAL — use MEDIUM
- `Response.Status.UNPROCESSABLE_ENTITY` doesn't exist in current JAX-RS version — use raw 422
- `@BeforeEach @Transactional` for WorkItemTemplate seeding in tests (PP-20260528-913df2)
- Layer 5 engine deps deferred — pom.xml comment explains why

## Open Issues Filed This Session

| Issue | Repo | What |
|-------|------|------|
| engine#379 | casehubio/engine | SignalReceivedEvent missing from engine jar |
| engine#380 | casehubio/engine | work-adapter Jandex references old engine internal packages |
| parent#79 | casehubio/parent | AGENTIC-HARNESS-GUIDE: domain entity discipline |
| parent#86 | casehubio/parent | casehub-life.md: Layer 2 complete, domain model correction |
| life#15 | casehubio/life | Refactor: extract shared WorkItemTemplate test fixture |

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #4 | Layer 3: + casehub-qhorus commitment lifecycle | M | Med | Start here |
| #5 | Layer 4: + casehub-ledger tamper-evident audit | M | Med | Blocked by #4 |

## References

- Spec: `docs/specs/2026-05-27-layer2-casehub-work-sla.md`
- LAYER-LOG: `LAYER-LOG.md` (Layer 1 + Layer 2 marked complete)
- ADR: `docs/adr/0001-life-domain-model-foundation-primitives-over-wrappers.md`
- Blog: `blog/2026-05-28-mdp01-layer2-domain-model-pivot.md`
- Protocols: PP-20260527-da1f66 (domain supplement), PP-20260528-913df2 (template test seeding)
- Garden: GE-20260528-d4b81d (engine SNAPSHOT), GE-20260528-35a81c (WorkItemPriority NORMAL), GE-20260528-936918 (UNPROCESSABLE_ENTITY), GE-20260528-3d3847 (@Transactional static), GE-20260528-c968e2 (ExpiryLifecycleService injection)
