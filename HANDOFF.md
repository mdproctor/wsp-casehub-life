# Handoff — casehub-life Layer 3 complete
2026-05-29

## Last Session

Layer 3 (casehub-qhorus commitment lifecycle) designed, implemented, reviewed (3 spec rounds + code review), and closed. Three accountability patterns: family delegation (COMMAND to household-member + Watchdog), contractor follow-up (COMMAND on per-actor channel + Watchdog), oversight gate (COMMAND to life/oversight; WorkItem created only on RESPONSE). 63 tests, 0 failures. Squashed to 1 commit, pushed to casehubio/life main.

## Immediate Next Step

Start Layer 4: casehub-ledger tamper-evident audit. Run `/work` for life#5.

Layer 4 adds: tamper-evident Merkle records for health decisions, financial decisions, legal actions. GDPR Art.17 for personal data held about contractors and dependents.

## Cross-Module

**Blocked by:**
- `casehub-engine` — SNAPSHOT has broken internal package refactor (engine#379, engine#380). Layer 5 deps removed from pom.xml. Not blocking Layer 4.

## Design Decisions — Carry Forward

- `LifeCommitmentStrategy` SPI lives in `app/`, not `api/` — context types reference JPA entities (circular dep if in api/)
- Oversight gate: `workItemId` is null until RESPONSE — no WorkItem for unapproved work
- `WatchdogAlertEvent` has NO `correlationId` — has `notificationChannel` (channel name String). Observer queries `LifeCommitmentRecord` by channel name.
- `MessageDispatch.Builder` requires `.actorType()` — throws `IllegalArgumentException: actorType is required` at `build()` if missing. Use `ActorType.SYSTEM` for life-system.
- `delegateTo` column repurposed as dedup key for OVERSIGHT mode — partial unique index enforces dedup
- casehub-work SNAPSHOT drift: `SelectionContext` constructor changes between SNAPSHOTs → `NoSuchMethodError`. Fix: `mvn install -DskipTests -pl api,core,deployment -am -f /path/to/casehub-work/pom.xml`
- Layer 5 engine deps deferred — pom.xml comment explains why

## Open Issues Filed This Session

| Issue | Repo | What |
|-------|------|------|
| life#16 | casehubio/life | docs/specs/life-automation.md layer table is stale |
| life#17 | casehubio/life | LifeWatchdogAlertObserver escalation integration test gap |
| life#18 | casehubio/life | REST resource consistency (@Produces/@Consumes, 201 for commitment) |
| parent#96 | casehubio/parent | casehub-life.md: Layer 3 complete |

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #5 | Layer 4: + casehub-ledger tamper-evident audit | M | Med | Start here |
| #6 | Layer 5: + casehub-engine CasePlanModel workflows | L | High | Blocked by engine#379, #380 |

## References

- Spec: `docs/specs/2026-05-29-layer3-qhorus-commitment.md`
- LAYER-LOG: `LAYER-LOG.md` (Layer 1–3 marked complete)
- Blog: `blog/2026-05-29-mdp01-layer3-qhorus-commitment.md`
- Garden: GE-20260529-bfa5d5 (WatchdogAlertEvent no correlationId), GE-20260529-e32a4d (MessageDispatch actorType required)
- Protocol: PP-20260529-e30ebd (life-domain channel naming)
