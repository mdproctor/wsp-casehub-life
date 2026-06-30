# Extract Structural CaseHub Duplication

**Issue:** casehubio/life#47
**Depends on:** casehubio/engine#591 (closed — `YamlCaseHub` augmentation support)
**Date:** 2026-06-30

## Problem

7 `YamlCaseHub` subclasses in casehub-life duplicate identical structural patterns:

1. `volatile CaseDefinition augmentedDefinition` field + double-checked lock in `getDefinition()` — now redundant (engine#591 moved this to the base class and made `getDefinition()` final)
2. `private static Capability cap(String name)` helper — dead API (`Worker.Builder` now uses `capabilityName(String)`, stores `Set<String>` not `List<Capability>`)
3. `@Inject LifeOpenClawChatModelFactory` and `@Inject LifeAgentDescriptorFactory` — identical in all 7
4. `setAgentDescriptors(Map.of(AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)))` — identical in all 7
5. 32 worker methods following the same 8-line Agent→Worker→AgentWorkerFunction pattern with only 3 varying parameters (capability name, system prompt, response schema)

The current code won't compile against the latest engine SNAPSHOT — `getDefinition()` is now `final`.

## Design

### `LifeYamlCaseHub` base class

New abstract class at `io.casehub.life.app.engine.LifeYamlCaseHub`, extending `YamlCaseHub`.

**Responsibilities:**
- Holds the two shared CDI injections (`openClawFactory`, `descriptorFactory`)
- Takes `LifeAgent` in constructor alongside the YAML path
- Overrides `augment(CaseDefinition)` to set agent descriptors via `setAgentDescriptors()`
- Provides `agentWorker(capabilityName, systemPrompt, responseSchema)` convenience method

**`augment()` contract:** non-final. Subclasses override to add workers (and any extra augmentation like SubCase bindings), then call `super.augment(definition)` to register the agent descriptor. Forgetting `super.augment()` causes immediate test failure — no descriptor means workers won't resolve.

**`agentWorker()` method:** 3 parameters — capability name, system prompt, response schema class. Derives worker name as `{capabilityName}-agent` (matches existing convention across all 32 workers). Builds the Agent→Worker→AgentWorkerFunction chain internally. No `userMessage` overload — the single worker that needs one (book-appointment-agent) constructs the Agent manually using the `agent()` getter and `openClawFactory`.

**`agent()` getter:** protected, returns the `LifeAgent` passed at construction. Used by the one CaseHub that needs manual Agent construction.

### Migrated CaseHub shape

Each CaseHub (e.g. FinancialReviewCaseHub) simplifies from ~228 lines to ~50:

```java
@ApplicationScoped
public class FinancialReviewCaseHub extends LifeYamlCaseHub {

    public FinancialReviewCaseHub() {
        super("life/financial-review.yaml", LifeAgent.FINANCE);
    }

    @Override
    protected void augment(CaseDefinition definition) {
        definition.getWorkers().addAll(List.of(
            agentWorker("gather-data", "...", GatherDataResult.class),
            agentWorker("analyse-anomalies", "...", AnalyseAnomaliesResult.class),
            agentWorker("escalate-anomalies", "...", EscalateAnomaliesResult.class),
            agentWorker("oversight-response", "...", OversightResponseResult.class),
            agentWorker("produce-report", "...", ProduceReportResult.class)
        ));
        super.augment(definition);
    }
}
```

### Exception cases

**TravelPlanCaseHub** — adds SubCase bindings alongside workers in `augment()`:

```java
@Override
protected void augment(CaseDefinition definition) {
    definition.getWorkers().addAll(List.of(...));
    definition.getBindings().addAll(List.of(
        familyVoteBinding("family-vote-a"),
        familyVoteBinding("family-vote-b"),
        familyVoteBinding("family-vote-c")
    ));
    super.augment(definition);
}
```

The `familyVoteBinding()` private method stays — it's case-specific logic, not shared infrastructure.

**AppointmentCycleCaseHub** — one worker (book-appointment-agent) has a `userMessage`. This worker is constructed manually using `openClawFactory` and `agent()`:

```java
Agent bookingAgent = Agent.builder()
    .model(openClawFactory.forAgent(agent()))
    .systemPrompt("...")
    .userMessage("Book a {{appointmentType}} appointment with provider {{provider}}.")
    .responseSchema(BookingResult.class)
    .build();
```

All other workers in that CaseHub use `agentWorker()`.

**FamilyVoteCaseHub** — unchanged. Stays on `YamlCaseHub` directly (no augmentation, no agent).

## Migration scope

| File | Change |
|------|--------|
| `LifeYamlCaseHub` (new) | Base class |
| `AppointmentCycleCaseHub` | Extend LifeYamlCaseHub, one manual worker |
| `CareCoordinationCaseHub` | Extend LifeYamlCaseHub |
| `CareEpisodeCaseHub` | Extend LifeYamlCaseHub |
| `ContractorCoordinationCaseHub` | Extend LifeYamlCaseHub |
| `FinancialReviewCaseHub` | Extend LifeYamlCaseHub |
| `HomeMaintenanceCaseHub` | Extend LifeYamlCaseHub |
| `TravelPlanCaseHub` | Extend LifeYamlCaseHub, keeps SubCase bindings |
| `FamilyVoteCaseHub` | Unchanged |
| `openclaw-agent-worker-pattern.md` | Update to reference `LifeYamlCaseHub.agentWorker()` |

## Testing

No behavioral change — pure structural refactoring. Existing 401 integration tests verify correctness.

New unit test `LifeYamlCaseHubTest`:
- `agentWorker()` builds correct Worker name, capability, and AgentWorkerFunction
- `augment()` sets agent descriptors
- Subclass override + `super.augment()` composes correctly

## What's removed per CaseHub

- `volatile CaseDefinition augmentedDefinition` field
- `getDefinition()` override (10 lines)
- `cap()` helper method (3 lines)
- 2 `@Inject` field declarations
- `setAgentDescriptors(...)` call
- Individual worker methods (each ~15 lines) → single `agentWorker()` calls

Net reduction: ~1,100 lines across 7 files.
