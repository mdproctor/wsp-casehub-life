# Fix and Re-enable Layer 5 Integration Tests

**Issue:** life#23
**Date:** 2026-06-04
**Status:** Design

## Context

Layer 5 added 8 CasePlanModel workflows with in-process workers. Integration tests that start cases and verify worker execution were blocked by engine#410 — `SchedulerService.getCaseDefinition()` returned null after successful registration due to a mutable-hashCode map key bug in `DefaultCaseDefinitionRegistry`.

engine#410 is now fixed (commit `66a6e34` — immutable `CaseKey` record + `RegistryEntry` map). engine#408 (constructor mismatches) is also fixed.

Only one integration test class exists: `AppointmentCycleIntegrationTest` (3 tests). These tests were written speculatively during Layer 5 implementation and have **never been green** — they were `@Disabled` immediately. They may fail for reasons beyond engine#410 (incorrect binding conditions, wrong Awaitility timeouts, missing CDI beans, transactional issues).

All other engine test classes (`*CaseHubTest`, `*CaseDefinitionsTest`) are definition-level tests that verify YAML loading and worker counts — they don't start cases and already pass. Six `*CaseHubTest` classes carry stale Javadoc referencing engine#408 as preventing worker execution — engine#408 is also closed.

## Prerequisites

### 0. Verify engine SNAPSHOT contains #410 fix

Confirm `DefaultCaseDefinitionRegistry$RegistryEntry` exists in the local `casehub-engine-0.2-SNAPSHOT.jar`. If not, rebuild engine:

```
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f ../engine/pom.xml -DskipTests --batch-mode
```

Verified 2026-06-04: `RegistryEntry` class present in local SNAPSHOT (updated 02:11, postdates fix commit).

## Changes

### 1. Remove `@Disabled` from `AppointmentCycleIntegrationTest`

Delete the class-level `@Disabled("Blocked by engine#410 — CaseDefinition not found after registration")`.

### 2. Remove all defensive guards

The test has three layers of defence against engine#410 — all removed:

- **`@BeforeAll checkEngineCompatibility()`** + `engineCompatible` field — dead code: `engineCompatible` is set but never read. Delete entirely.
- **`startCaseOrSkip()` method** — wraps `startCase()` in try/catch calling `assumeTrue(false)` on failure. Delete; inline `caseHub.startCase(input).toCompletableFuture().join()` directly in each test.
- **`startCaseOrSkip()` worker-polling guard** — catches worker scheduling failures and calls `assumeTrue(false)`. Delete; let Awaitility timeout failures propagate as real test failures.

Remove `assumeTrue` import once no longer used.

### 3. Fix `scheduledWorkerNames()` error swallowing

The helper's try/catch returns `Set.of()` on exception — written to mask engine#410 failures during Awaitility polling. Post-fix, this hides real bugs: if `eventLog()` throws, the Awaitility poll silently returns empty and times out with a generic "condition not met" message instead of the actual exception.

Remove the try/catch. Let exceptions propagate through Awaitility — it surfaces the root cause in its timeout message.

### 4. Fix double-query pattern in `goldenPath` test

Lines 154–156 query for a PENDING WorkItem inside the Awaitility block, then query again immediately after. Between the two calls, state could theoretically change. Refactor to capture the WorkItem inside the await block or use `AtomicReference` to carry it out.

### 5. Clean stale engine#408 Javadoc from 6 CaseHubTest classes

Six `*CaseHubTest` classes have Javadoc saying "Does not start a case — engine#408 prevents worker execution." engine#408 is closed. Remove the stale engine#408 references. These are definition-level tests by design (not by limitation) — update the Javadoc to reflect that.

Files: `CareCoordinationCaseHubTest`, `FinancialReviewCaseHubTest`, `CareEpisodeCaseHubTest`, `ContractorCoordinationCaseHubTest`, `TravelPlanCaseHubTest`, `HomeMaintenanceCaseHubTest`.

### 6. Verify tests pass

```
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=AppointmentCycleIntegrationTest --batch-mode
```

If tests fail for reasons beyond engine#410 (binding conditions, timeouts, CDI wiring), diagnose and fix — the tests have never been green.

### 7. Update documentation

- **CLAUDE.md line 231** (testing guidance): Update "Integration tests... are blocked on engine#410" to note integration tests now pass.
- **CLAUDE.md lines 388–389** (Layer 5 status): Change `🔲 PENDING — integration tests blocked on engine#410` to `✅ COMPLETE`.
- **LAYER-LOG.md line 388**: Update "Not closed here: integration tests `@Disabled`" to note resolved.
- **LAYER-LOG.md lines 464, 466–469**: Annotate the gotcha entries as RESOLVED with commit reference `66a6e34`. Do not delete — gotchas are historical records.
- **ARC42STORIES.MD line 948**: Update Layer 5 closure note.
- **ARC42STORIES.MD lines 1023, 1025–1028**: Annotate gotcha as RESOLVED.
- **ARC42STORIES.MD line 1123**: Change open-issues table row from `OPEN` to `RESOLVED`.

## Protocol note

All 8 CaseHub classes use `Worker.builder().function(lambda)` — raw lambdas, not FuncWorkflowBuilder (PP-20260531-worker-func-exec). This is a pre-existing violation across all case definitions, not introduced here. Filed as a follow-up issue.

## Follow-up issues to file

1. **Integration tests for remaining 7 case definitions** — 1 of 8 case definitions has an integration test. The gap is acceptable for this issue but should be tracked.
2. **Migrate all CaseHub workers from `Worker.builder().function(lambda)` to FuncWorkflowBuilder** — PP-20260531-worker-func-exec compliance across all 8 case definitions.

## Out of scope

- Adding integration tests for the other 7 case definitions — tracked as follow-up issue.
- Migrating workers to FuncWorkflowBuilder — tracked as follow-up issue.
- Integration test base class — premature until the second integration test class is written (follow-up #1).
