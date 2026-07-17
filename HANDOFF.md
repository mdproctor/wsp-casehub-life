# Handoff — 2026-07-17

Closed #71 (CareCoordinationCaseHubTest Expression wrapper), #72 (CbrCaseMemoryStore signature + blocking IO). Filed engine#745 (worker output not propagated — HomeMaintenance/TravelPlan integration tests). 35 CDI errors from previous handover resolved by engine SNAPSHOT update — no action needed.

## Last Session

SNAPSHOT compat fix branch — adapted to neocortex and engine API changes. CbrCaseMemoryStore.store() gained 7th Path param, CbrQuery.of() gained 3rd Path param — updated all callers (LifeCaseOutcomeCbrWriter, LifeRoutingOutcomeRecorder, LifeCbrSuggestionService) and test mocks. Fixed LifeRoutingOutcomeRecorder blocking IO (.emitOn → .runSubscriptionOn). Fixed CareCoordinationCaseHubTest SubCaseMapping.Expression pattern match. FinancialReviewIntegrationTest now passes; HomeMaintenance + TravelPlan are engine regressions.

## Immediate Next Step

No open issues in the current batch. Remaining open issues are #60 (OpenClaw skill integration, L/High, blocked on openclaw Epic 4) and engine#738 (PlanAdapter wiring upstream). engine#745 blocks HomeMaintenance/TravelPlan integration tests.

## What's Left

- engine#745 — worker output not propagated to case context (blocks 2 integration tests) · M · Med
- engine#738 — PlanAdapter wiring into CbrRetrievalService · M · Med
- neocortex#157 — FeatureStatistics upstream move · XS · Low
- LifeTaskResourceTest.createLifeTask_withPastDeadline — SLA breach escalation candidateGroups wrong (household-member vs household-admin) — unfiled · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #60 | OpenClaw skill integration | L | High | Blocked on openclaw Epic 4 |

## References

- Blog: `blog/2026-07-16-mdp02-trust-scores-meet-cbr-adaptation.md`
