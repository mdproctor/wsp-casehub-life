# Handoff — 2026-07-17

Closed #71 (CareCoordinationCaseHubTest Expression wrapper), #72 (CbrCaseMemoryStore signature + blocking IO). engine#745 (worker output not propagated) filed and resolved same day — all integration tests green.

## Last Session

SNAPSHOT compat fix branch — adapted to neocortex and engine API changes. CbrCaseMemoryStore.store() gained 7th Path param, CbrQuery.of() gained 3rd Path param — updated all callers and test mocks. Fixed LifeRoutingOutcomeRecorder blocking IO (.emitOn → .runSubscriptionOn). Fixed CareCoordinationCaseHubTest SubCaseMapping.Expression pattern match. All integration tests pass after engine#745 fix.

## Immediate Next Step

No open issues in the current batch. Remaining open issues are #60 (OpenClaw skill integration, L/High, blocked on openclaw Epic 4) and engine#738 (PlanAdapter wiring upstream).

## What's Left

- engine#738 — PlanAdapter wiring into CbrRetrievalService · M · Med
- neocortex#157 — FeatureStatistics upstream move · XS · Low
- LifeTaskResourceTest.createLifeTask_withPastDeadline — SLA breach escalation candidateGroups wrong (household-member vs household-admin) — unfiled · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #60 | OpenClaw skill integration | L | High | Blocked on openclaw Epic 4 |

## References

- Blog: `blog/2026-07-16-mdp02-trust-scores-meet-cbr-adaptation.md`
