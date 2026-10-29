# Playbook Failure Detection, Triage, and Adaptive Recovery

**Issue:** casehubio/life#141
**Depends on:** casehubio/life#140 (playbook coordination foundation)
**Date:** 2026-10-29

## Problem

LLM-generated playbooks may fail at runtime — connector timeouts, wrong parameters, unexpected response shapes, authorization failures, endpoint changes. The current system has no failure detection, no triage, and no feedback loop. Silent failure is the worst outcome: a playbook fails, the case sits doing nothing, and the same broken playbook gets selected again for similar cases.

Current failure paths:
- **Full failure:** `FlowWorkerFunctionHandler` catches exceptions and wraps them in `PlaybookExecutionException` with `PlaybookExecutionResult.Failure`. The engine's `RetryOrchestrator` retries per the retry policy, then fires `WorkerRetriesExhaustedEvent`. No recovery beyond that.
- **Partial failure:** A playbook with 5 connector steps where 2 fail produces a `PlaybookExecutionResult.Success` — the successful steps mask the failures. The domain worker receives incomplete data and makes worse decisions.
- **Planning worker failure:** `LifeCaseService.invokePlanningWorker()` catches exceptions silently (`catch → LOG.warn → return null`), falling through to uncoordinated execution. No feedback.

## Design

### 1. Failure Detection — Engine Change

`FlowWorkerFunctionHandler` gains partial failure detection. After workflow completion (no exception thrown), the handler inspects the collected `StepResult` list. If any step has `StepStatus.FAILED`, it produces `PlaybookExecutionResult.PartialSuccess` instead of `Success`.

```java
// In FlowWorkerFunctionHandler.execute(), after line 96:
List<StepResult> steps = registry.getSteps(instanceId);
registry.remove(instanceId);

boolean hasFailedSteps = steps.stream()
    .anyMatch(s -> s.status() == StepStatus.FAILED);

// ... existing output map extraction ...

PlaybookExecutionResult execResult = hasFailedSteps
    ? new PlaybookExecutionResult.PartialSuccess(outputMap, steps)
    : new PlaybookExecutionResult.Success(outputMap, steps);
```

`PartialSuccess` propagates through `HandlerResult.executionResult()`. The engine treats `PartialSuccess` as a failure — the `RetryOrchestrator` fires, and when retries exhaust, `RecoveryCoordinator.handleFailure()` receives the `PlaybookExecutionResult` via `RecoveryContext`.

**Engine issue required.** The change is ~5 lines in `FlowWorkerFunctionHandler` plus a test. The `PartialSuccess` type already exists in the engine API — it was designed for this scenario.

**PartialSuccess as failure policy:** `PartialSuccess` needs to be treated as a worker failure by the engine's execution pipeline so that the retry policy applies. This requires the handler to either throw (wrapping partial output in an exception) or the engine's executor to check `HandlerResult.executionResult()` for non-Success types. The cleaner path: the handler throws a `PlaybookExecutionException` for `PartialSuccess` too, carrying the partial output. The `RetryOrchestrator` already extracts `PlaybookExecutionResult` from `ExecutionResultCarrier` on the cause.

### 2. Triage Orchestration — LifePlaybookRecoveryCoordinator

Life implements the engine's `RecoveryCoordinator` SPI.

```
io.casehub.life.app.coordination.recovery
├── LifePlaybookRecoveryCoordinator.java    — RecoveryCoordinator impl
├── TriageRequest.java                      — context record for the planning worker
├── TriageDecision.java                     — structured response schema
├── TriageAction.java                       — enum: RETRY_MODIFIED, SWITCH_PLAYBOOK,
│                                             GENERATE_FIX, FALLBACK_MCP
├── TriageExecutor.java                     — applies TriageDecision to case definition
└── TriageRoundTracker.java                 — per-worker round counting (cap at 2)
```

#### LifePlaybookRecoveryCoordinator

`@ApplicationScoped`, implements `RecoveryCoordinator`. The `handleFailure(RecoveryContext)` method:

1. **Guard:** If `context.executionResult()` is null → return false (not a playbook failure, let the engine handle it).
2. **Guard:** If `TriageRoundTracker.roundCount(caseId, workerName) >= 2` → return false (triage budget exhausted, fall through to `WorkerRetriesExhaustedEvent`).
3. **Build triage request:** Assemble `TriageRequest` from `RecoveryContext` — worker name, capability, step results, error messages, partial output, template name (resolved from the worker's function), the current coordination plan, and the full template catalog listing.
4. **Invoke planning worker:** Use the same `LifeAgentProviderFactory` and agent infrastructure to invoke the planning worker with a triage-specific system prompt and the `TriageDecision` response schema.
5. **Apply decision:** Delegate to `TriageExecutor.apply(TriageDecision, caseId)`.
6. **Record CBR:** Fire `PlaybookTriageEvent` CDI event for `LifePlaybookCbrWriter` to capture.
7. **Increment round:** `TriageRoundTracker.increment(caseId, workerName)`.
8. **Return true:** The engine re-schedules the worker with fresh retry budget (new triage round).

#### TriageRequest

Record carrying all context the planning worker needs to make a triage decision:

```java
public record TriageRequest(
    String workerName,
    String capabilityName,
    String templateName,
    PlaybookExecutionResult executionResult,
    List<StepResult> steps,
    int triageRound,
    List<String> catalogTemplates,
    Map<String, Object> caseContext) {}
```

#### TriageDecision (response schema)

```java
public record TriageDecision(
    TriageAction action,
    String reasoning,
    // For RETRY_MODIFIED:
    Map<String, Object> modifiedParams,
    // For SWITCH_PLAYBOOK:
    String alternateTemplateName,
    // For GENERATE_FIX:
    String fixedPlaybookYaml,
    String fixedPlaybookName,
    // For FALLBACK_MCP:
    String fallbackReason) {}
```

#### TriageAction enum

```java
public enum TriageAction {
    RETRY_MODIFIED,
    SWITCH_PLAYBOOK,
    GENERATE_FIX,
    FALLBACK_MCP
}
```

#### TriageExecutor

Applies the triage decision:

- **RETRY_MODIFIED:** Injects `modifiedParams` into the worker's input data via case context update. The same playbook template executes with adjusted parameters.
- **SWITCH_PLAYBOOK:** Looks up `alternateTemplateName` in `CoordinationTemplateCatalog`. Resolves to a `PlaybookWorkerFunction`. Rebuilds the case definition via `LifeTypedCaseHub.buildFreshDefinition()` with the swapped template.
- **GENERATE_FIX:** Validates the fixed YAML via `CoordinationTemplateValidator`. If valid, persists to `CoordinationTemplateCatalog` (replaces existing non-curated template or creates new). Compiles and swaps into the definition.
- **FALLBACK_MCP:** Sets a `skipPlaybook` flag in case context for the affected worker. The domain worker's standard MCP tool path executes instead of the playbook path. The coordinator returns true but marks that this worker should not be re-planned.

#### TriageRoundTracker

In-memory tracking of triage rounds per (caseId, workerName). `ConcurrentHashMap<String, AtomicInteger>` keyed by `caseId:workerName`. Cleaned up when `LifeProvisionerCleanupObserver` fires on case terminal state (piggyback on existing cleanup).

The tracker integrates with the engine's `triageRoundLookup` function — `RetryOrchestrator` already accepts a `BiFunction<UUID, String, Integer>` that scopes failure counting per triage round. Life provides this function from `TriageRoundTracker.roundCount()`.

### 3. Planning Worker Triage Prompt

The planning worker is re-invoked with a triage-specific system prompt that provides:

1. The failure context — which steps failed, what errors occurred, what partial output was produced
2. The original coordination plan that produced the failing playbook
3. The template catalog listing (what alternatives exist)
4. The triage round number (so the planner knows how many chances remain)
5. Explicit instruction to return a `TriageDecision` with one of the four actions

The prompt guides the planner:
- Round 1: prefer `RETRY_MODIFIED` or `SWITCH_PLAYBOOK` — the cheapest recovery
- Round 2: if the same failure recurs, `GENERATE_FIX` is appropriate — the template itself is broken
- If the planner cannot determine a fix: `FALLBACK_MCP` — let the domain worker handle it directly

### 4. CBR Retention — LifePlaybookCbrWriter

New `@ApplicationScoped` CDI observer that listens for `PlaybookTriageEvent` (fired by the coordinator after each triage decision).

Writes a `PlanCbrCase` to `CbrCaseMemoryStore` with enriched features:

| Feature | Type | Source |
|---------|------|--------|
| `templateName` | CategoricalVal | Triage request |
| `failedStepCount` | NumberVal | Step results |
| `failedStepNames` | CategoricalVal | Step results (comma-joined) |
| `triageAction` | CategoricalVal | Triage decision |
| `triageRound` | NumberVal | Round tracker |
| `connectorErrorType` | CategoricalVal | First failed step error (categorized) |
| Domain features | Various | Existing case context extractors |

The existing `LifeCbrFeatureSchemaRegistrar` gains a `playbook-failure` feature set registered alongside existing domain schemas. `SimilaritySpec`: `CategoricalTable` for template name and error type, `GaussianDecay` for failure count.

**Integration with adaptation rules:** The existing per-domain adaptation rules (e.g., `ContractorAdaptationRule`) gain a playbook check: if CBR retrieval surfaces prior failures for the same template + similar context, the adaptation rule can recommend a different template or add pre-check steps.

### 5. Playbook Improvement Loop

When the planner returns `GENERATE_FIX`:

1. `TriageExecutor` validates the fixed YAML via `CoordinationTemplateValidator` (existing validator — checks parseability, has steps, compiles, all URLs are local `/api/connectors/*`).
2. If validation passes: persist to `CoordinationTemplateCatalog` with `curated=false`. If a template with the same name exists and is not curated, it is replaced. Curated templates are never replaced — the fix gets a new name (e.g., `contractor-onboarding-fixed-v1`).
3. The `useCount` on the new template starts at 0 — it earns trust through successful executions.
4. CBR retention records the fix: `triageAction=GENERATE_FIX`, so future CBR retrievals can surface "this template was created to fix a failure in template X."

Over time, the catalog evolves: broken playbooks get fixed, the fixed versions accumulate use counts, and CBR surfaces the successful alternatives for similar contexts.

### 6. Planning Worker Failure Hardening

The current `LifeCaseService.invokePlanningWorker()` silently catches exceptions. This must be hardened:

- **On planning worker exception:** Log at WARN (existing), but also write a CBR record with `triageAction=FALLBACK_MCP` and `connectorErrorType=PLANNING_FAILURE`. The case proceeds uncoordinated (existing fallback), but the failure is recorded.
- **On triage planning worker exception:** The coordinator catches the exception, logs, and returns false (triage failed — let the engine's default exhaustion path handle it). A CBR record is written with `triageAction=TRIAGE_FAILURE`.

### 7. Engine Integration Points

#### Required engine changes (new issue on casehubio/engine):

1. **PartialSuccess detection** in `FlowWorkerFunctionHandler` — produce `PartialSuccess` when any step failed, throw `PlaybookExecutionException` to trigger retry.
2. **TriageRoundLookup wiring** — ensure `RetryOrchestrator` accepts the `triageRoundLookup` BiFunction from CDI (already parameterized — verify the wiring path from life's CDI to the engine's constructor).

#### Existing engine infrastructure consumed (no changes needed):

- `RecoveryCoordinator` SPI — life implements it
- `RecoveryContext` — carries `PlaybookExecutionResult` and execution metadata
- `RetryOrchestrator` — handles retry budget per triage round
- `PlaybookExecutionResult` sealed interface — `Success`, `PartialSuccess`, `Failure`
- `PlaybookExecutionException` implements `ExecutionResultCarrier`
- `FlowExecutionRegistry` — step-level result tracking
- `WorkerRetriesExhaustedEvent` — fallback when triage fails

### 8. Module Placement

All new code lives in `app/`:

```
app/src/main/java/io/casehub/life/app/coordination/recovery/
├── LifePlaybookRecoveryCoordinator.java
├── TriageRequest.java
├── TriageDecision.java
├── TriageAction.java
├── TriageExecutor.java
├── TriageRoundTracker.java
├── PlaybookTriageEvent.java
└── LifePlaybookCbrWriter.java
```

No `api/` changes — all types are internal to `app/`. The `TriageDecision` response schema is a Java record in `app/` (like the existing 32 response schemas in `app/engine/agent/`).

### 9. Testing Strategy

| Test | Type | What it verifies |
|------|------|-----------------|
| `TriageRoundTrackerTest` | Unit | Round counting, cap enforcement, cleanup |
| `TriageExecutorTest` | Unit | Each action type applies correctly; validation gate for GENERATE_FIX |
| `LifePlaybookRecoveryCoordinatorTest` | Unit (mocked) | Guard checks, planner invocation, decision dispatch, CBR event firing |
| `LifePlaybookCbrWriterTest` | Unit | Feature extraction from PlaybookTriageEvent, PlanCbrCase assembly |
| `PlaybookRecoveryIntegrationTest` | `@QuarkusTest` | Full path: playbook fails → retries exhaust → coordinator called → planner triages → decision applied → worker re-executes |
| `PartialSuccessDetectionTest` | Unit (engine) | FlowWorkerFunctionHandler produces PartialSuccess for mixed step results |

### 10. Non-Goals

- Automatic playbook repair without LLM involvement — too risky. Mechanical fixes to mechanical coordination could compound errors.
- Infinite retry loops — capped at 2 triage rounds per worker.
- Curated template replacement — curated templates are fixed in source code, not at runtime.

## References

- `engine/common-core/.../RecoveryCoordinator.java` — SPI for handling exhausted retries
- `engine/common-core/.../RetryOrchestrator.java` — retry policy evaluation, triage round scoping
- `engine/common-core/.../RecoveryContext.java` — carries PlaybookExecutionResult to recovery handler
- `engine/api/.../PlaybookExecutionResult.java` — sealed interface with Success/PartialSuccess/Failure
- `engine/api/.../StepResult.java` — per-step execution outcome
- `engine/flow/FlowWorkerFunctionHandler.java` — playbook execution, step result collection
- `engine/flow/PlaybookExecutionException.java` — implements ExecutionResultCarrier
- `life/app/coordination/CoordinationTemplateCatalog.java` — classpath + DB template storage
- `life/app/coordination/CoordinationTemplateValidator.java` — YAML validation
- `life/app/coordination/CoordinationPlanResult.java` — planning worker output
- `life/app/engine/LifeCaseService.java` — planning worker invocation (current silent-catch path)
- `life/app/engine/LifeTypedCaseHub.java` — buildFreshDefinition() for template swapping
- `life/app/cbr/LifeCaseOutcomeCbrWriter.java` — existing CBR case-level retention
- `life/app/cbr/LifeCbrFeatureSchemaRegistrar.java` — domain feature schema registration
- Issue casehubio/life#140 — playbook coordination foundation
