# Handoff — casehub-life cleanup + Layer 4 next
2026-05-29

## Last Session

Completed all S/XS backlog items: fixed stale tutorial layer table in `docs/specs/life-automation.md` (#16), added 7-test `LifeWatchdogAlertObserver` integration test suite + NPE fix in `createEscalationTask()` for null `delegateTo` (#17), added `@Produces/@Consumes` class-level to `ExternalActorResource` and changed commitment creation endpoints from 200 → 201 (#18). 57 tests passing. All three issues closed, branch merged to casehubio/life main.

Key discovery: Flyway disabled in tests (`migrate-at-start=false`) — WorkItemTemplates must be seeded programmatically in `@BeforeEach`. Pattern now in CLAUDE.md.

## Immediate Next Step

Start Layer 4: casehub-ledger tamper-evident audit. Run `/work` for life#5.

Layer 4 adds: tamper-evident Merkle records for health decisions, financial decisions, legal actions. GDPR Art.17 for personal data held about contractors and dependents.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Design Decisions — Carry Forward

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- `parent#96` — casehub-life.md: Layer 3 complete (cross-repo issue, already filed) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #5 | Layer 4: + casehub-ledger tamper-evident audit | M | Med | Start here |
| #6 | Layer 5: + casehub-engine CasePlanModel workflows | L | High | Blocked by engine#379, #380 |

## References

- Spec: `docs/specs/2026-05-29-layer3-qhorus-commitment.md`
- LAYER-LOG: `LAYER-LOG.md` (Layer 1–3 marked complete)
- Blog: `blog/2026-05-29-mdp01-layer3-qhorus-commitment.md`
- Garden: GE-20260529-bfa5d5, GE-20260529-e32a4d, GE-20260519-114395 (CDI proxy observer testing)
- Protocol: PP-20260529-e30ebd (life-domain channel naming)
