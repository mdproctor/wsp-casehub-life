# Re-enable Layer 5 Integration Tests

**Issue:** life#23
**Date:** 2026-06-04
**Status:** Design

## Context

Layer 5 added 8 CasePlanModel workflows with in-process FuncDSL workers. Integration tests that actually start cases and verify worker execution were blocked by engine#410 — `SchedulerService.getCaseDefinition()` returned null after successful registration due to a mutable-hashCode map key bug in `DefaultCaseDefinitionRegistry`.

engine#410 is now fixed (commit `66a6e34` — immutable `CaseKey` record + `RegistryEntry` map). The fix is in the local `0.2-SNAPSHOT`.

Only one integration test class exists: `AppointmentCycleIntegrationTest` (3 tests). All other engine test classes (`*CaseHubTest`, `*CaseDefinitionsTest`) are definition-level tests that verify YAML loading and worker counts — they don't start cases and already pass.

## Changes

### 1. Remove `@Disabled` from `AppointmentCycleIntegrationTest`

Delete the class-level `@Disabled("Blocked by engine#410 — CaseDefinition not found after registration")`.

### 2. Remove defensive `assumeTrue` guards

The test has three layers of defence against engine#410:

- **`@BeforeAll checkEngineCompatibility()`** + `engineCompatible` field — checks `PlanExecutionContext` constructor reflectively. This was a compatibility guard, not an engine#410 guard, but it's dead code — `engineCompatible` is never checked after being set.
- **`startCaseOrSkip()` try/catch around `startCase()`** — catches exceptions and calls `assumeTrue(false)` to skip instead of fail.
- **`startCaseOrSkip()` try/catch around `scheduledWorkerNames()`** — catches failures polling for scheduled workers and calls `assumeTrue(false)`.

All three are removed. `startCase()` failures and worker scheduling failures become real test failures. The method `startCaseOrSkip()` is inlined or replaced with a direct `startCase()` call + assertion.

### 3. Simplify test structure

After removing the guards, each test method calls `caseHub.startCase(input)` directly. The `scheduledWorkerNames()` helper is retained — it's a useful query abstraction.

### 4. Update documentation

- **CLAUDE.md**: Remove "integration tests blocked on engine#410" from Layer 5 status. Change `🔲 PENDING` to `✅ COMPLETE`.
- **ARC42STORIES.MD**: Update §9.4 Layer 5 open-issues table — mark engine#410 row as RESOLVED.
- **LAYER-LOG.md**: Update Layer 5 section — note engine#410 resolved, integration tests re-enabled.

## Out of scope

- Adding integration tests for the other 7 case definitions (home-maintenance, travel-plan, etc.) — that's new work, not re-enabling existing tests.
- Modifying the test assertions or workflow logic.
