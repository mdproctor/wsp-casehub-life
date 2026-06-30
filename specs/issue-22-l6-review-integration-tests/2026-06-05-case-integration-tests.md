# Integration Tests for 7 Case Definitions — Design Spec

**Issue:** life#24
**Scope:** 7 integration test classes + 1 shared helper utility

## Shared Infrastructure

`CaseIntegrationTestSupport` — static utility (not base class):
- `scheduledWorkerNames(runtime, caseId)` — event log query for WORKER_SCHEDULED events
- `startCase(caseHub, input)` — start + await first worker
- `awaitWorker(runtime, caseId, workerName)` — Awaitility until worker name in scheduled set
- `completeHumanTask(workItemService, caseId)` — find WorkItem by `case:{caseId}` callerRef prefix, complete via `requiringNew()`
- `awaitCaseCompleted(cache, caseId)` — Awaitility until COMPLETED

Refactor `AppointmentCycleIntegrationTest` to use shared helpers for consistency.

## Test Classes

### FamilyVoteIntegrationTest
- `goldenPath_humanTaskToCompletion` — start → humanTask → complete → COMPLETED

### CareEpisodeIntegrationTest
- `goldenPath_workersAndHumanTaskToCompletion` — start → assess-patient → provide-care → record-notes humanTask → complete → COMPLETED

### TravelPlanIntegrationTest
- `approvalPath_completesAfterHumanTask` — start → research → parallel searches → budget → approval-gate humanTask → complete → booking → confirmation → COMPLETED
- `parallelSearchesBothFire` — verify flight-search and hotel-search both scheduled

### HomeMaintenanceIntegrationTest
- `workersFireAndHumanTaskCreated` — start → schedule-inspection → get-quotes → approve-contractor humanTask created
- `afterApproval_issueCommitmentFires` — complete humanTask → issue-commitment fires

### ContractorCoordinationIntegrationTest
- `requestQuoteFires` — start → request-quote-agent fires

### FinancialReviewIntegrationTest
- `workersFireThroughEscalation` — start → gather-data → analyse-anomalies → escalate-anomalies fires

### CareCoordinationIntegrationTest
- `workersFireAndAssignCarerCreated` — start → needs-assessment → care-plan → assign-carer humanTask created

## Patterns

All tests follow AppointmentCycleIntegrationTest:
- `@QuarkusTest`, `@BeforeEach @Transactional` seeds `LifeTestFixtures.seedStandardTemplates()`
- `QuarkusTransaction.requiringNew()` for Panache queries in Awaitility
- Filter WorkItems by `callerRef` prefix for test isolation
- No `CaseStatus.WAITING` assertion — engine stays RUNNING during humanTasks

Bridge-dependent cases (home-maintenance, contractor-coordination, financial-review) test up to the bridge point. Full bridge testing is infrastructure scope, not case definition scope.