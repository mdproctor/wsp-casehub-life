# Handoff — 2026-07-16

Closed #55 (CBR domain adaptation rules). 6 per-domain LifeAdaptationRule impls + LifePlanAdapter displacing NoOpPlanAdapter. Design reviewed (4 rounds, 18 issues, $15.42). Code reviewed (4 rounds, 17 issues, $14.41). Blog: adaptation-is-where-domain-knowledge-lives. Filed engine#738 (PlanAdapter wiring), life#66 (FeatureStatistics upstream), life#67 (trust-aware adaptation), life#68-70 (code review deferred items).

## Last Session

CBR adaptation rules (#55) — brainstormed, designed, reviewed, implemented, code-reviewed. Composite LifePlanAdapter (@Alternative) dispatches to 6 per-domain rules (contractor, home-maintenance, health, appointment-cycle, financial, travel). Each rule is a pure function on feature deltas (season, budget, severity, escalation patterns). SeverityScaling shared helper. LifeCbrRetrievalResult + retrieveForAdaptation() splits adaptation (≥1 case) from statistics (≥2). CbrInputTransformer enhanced to format adapted plan alongside raw experiences. AdaptationTrace fired as CDI event.

## Immediate Next Step

Pick up next work. Remaining issues (#60, #66, #67, #68, #69, #70) are mostly small or blocked. engine#738 tracks PlanAdapter wiring upstream.

## What's Left

- Pre-existing test failures: CareCoordinationCaseHubTest (Expression wrapper), integration tests (SNAPSHOT tenant compat) — still unfiled
- engine#738 — PlanAdapter wiring into CbrRetrievalService · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #66 | Move FeatureStatistics upstream to neocortex-memory-api | XS | Low | |
| #67 | Trust-score-aware adaptation | S | Med | Augment currentFeatures with trust scores |
| #68 | Evaluate providerId vs providerType/careType | XS | Low | Deferred from code review |
| #69 | Evaluate AdaptationTrace firing for empty adaptations | XS | Low | Deferred from code review |
| #70 | End-to-end LifeCaseService integration test | S | Med | |
| #60 | OpenClaw skill integration | L | High | Blocked on openclaw Epic 4 |

## References

- Spec: `docs/specs/2026-07-14-cbr-adaptation-rules-design.md`
- Blog: `blog/2026-07-16-mdp01-adaptation-is-where-domain-knowledge-lives.md`
- Review (spec): `~/adr/casehub-life/cbr-adaptation-rules-20260714-195035/`
- Review (code): `~/adr/casehub-life/cbr-adaptation-rules-code-20260716-125410/`
- Plan: `plans/attic/2026-07-14-cbr-adaptation-rules.md`
