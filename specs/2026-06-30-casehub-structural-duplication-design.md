# Extract Structural CaseHub Duplication

**Issue:** casehubio/life#47
**Depends on:** casehubio/engine#591 (closed — `YamlCaseHub` augmentation support)
**Evolves:** §Case Hub Descriptors of `docs/specs/2026-06-08-business-logic-centralization.md` (life#27) — engine#591 changed the extension point from `getDefinition()` override to `augment()` hook; this spec adopts `LifeTypedCaseHub` name and `lifeCaseType()` identity from that design
**Date:** 2026-06-30

## Problem

7 `YamlCaseHub` subclasses in casehub-life duplicate identical structural patterns:

1. `volatile CaseDefinition augmentedDefinition` field + double-checked lock in `getDefinition()` — **compilation break**: engine#591 moved this to the base class and made `getDefinition()` final
2. `private static Capability cap(String name)` helper + `Worker.Builder.capabilities(List.of(cap(...)))` calls — **compilation break**: `Worker.Builder` no longer has `capabilities(List<Capability>)`; the builder now uses `capabilityName(String)` / `capabilityNames(String...)` / `capabilityNames(Collection<String>)` and stores `Set<String>` not `List<Capability>`
3. `@Inject LifeOpenClawChatModelFactory` and `@Inject LifeAgentDescriptorFactory` — identical in all 7
4. `setAgentDescriptors(Map.of(AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)))` — identical in all 7
5. 32 worker methods following the same 8-line Agent→Worker→AgentWorkerFunction pattern with only 3 varying parameters (capability name, system prompt, response schema)

Both compilation breaks must be fixed by this migration: every subclass that overrides `getDefinition()` (all 7) and every subclass that calls `cap()` (all 7) will fail to compile against the engine 0.2-SNAPSHOT.

## Design

### `LifeTypedCaseHub` base class

New abstract class at `io.casehub.life.app.engine.LifeTypedCaseHub`, extending `YamlCaseHub`.

This adopts the name and case-type identity concept from the business-logic-centralization spec (life#27, §Case Hub Descriptors), which designed a `LifeTypedCaseHub` abstract class with the same structural role. Engine#591's `augment(CaseDefinition)` hook supersedes that spec's `getDefinition()` override approach — the engine now owns the double-checked lock and caching internally, calling `augment()` as the subclass extension point.

**Responsibilities:**
- Holds the two shared CDI injections (`openClawFactory`, `descriptorFactory`)
- Takes `LifeAgent` in constructor alongside the YAML path
- Declares `lifeCaseType()` abstract method — carries case type identity for `LifeCaseService.resolve()` (prepares for switch elimination under life#27)
- Overrides `augment(CaseDefinition)` as `final` — template method that calls `configureCase()` then registers agent descriptors
- Provides `agentWorker(capabilityName, systemPrompt, responseSchema)` convenience method

**Template method contract:** `augment()` is `final` in `LifeTypedCaseHub`. Subclasses override `configureCase(CaseDefinition)` to add workers (and any extra augmentation like SubCase bindings). They never touch `augment()` directly and cannot forget descriptor registration — it happens unconditionally after `configureCase()` returns:

```java
@Override
protected final void augment(CaseDefinition definition) {
    configureCase(definition);
    definition.setAgentDescriptors(Map.of(
        agent.agentId(), descriptorFactory.descriptorFor(agent)));
}

protected abstract void configureCase(CaseDefinition definition);
```

Making `augment()` final in `LifeTypedCaseHub` also prevents re-introduction of the double-checked lock pattern that engine#591 was designed to eliminate.

**`agentWorker()` implementation:**

```java
protected Worker agentWorker(String capabilityName, String systemPrompt, Class<?> responseSchema) {
    Agent agent = Agent.builder()
        .model(openClawFactory.forAgent(this.agent))
        .systemPrompt(systemPrompt)
        .responseSchema(responseSchema)
        .build();

    return Worker.builder()
        .name(capabilityName + "-agent")
        .capabilityName(capabilityName)
        .function(new AgentWorkerFunction(agent))
        .build();
}
```

Uses `capabilityName(String)` (the new Worker.Builder API, replacing `capabilities(List<Capability>)`). Derives worker name as `{capabilityName}-agent` (matches existing convention across all 32 workers). The `Capability` type and `cap()` helper are fully removed. No `userMessage` overload — the single worker that needs one (book-appointment-agent) constructs the Agent manually using the `agent()` getter and `openClawFactory`.

**`agent()` getter:** protected, returns the `LifeAgent` passed at construction. Used by the one CaseHub that needs manual Agent construction.

### Migrated CaseHub shape

Each CaseHub (e.g. FinancialReviewCaseHub) simplifies from ~228 lines to ~50:

```java
@ApplicationScoped
public class FinancialReviewCaseHub extends LifeTypedCaseHub {

    public FinancialReviewCaseHub() {
        super("life/financial-review.yaml", LifeAgent.FINANCE);
    }

    @Override
    public LifeCaseType lifeCaseType() { return LifeCaseType.FINANCIAL_REVIEW; }

    @Override
    protected void configureCase(CaseDefinition definition) {
        definition.getWorkers().addAll(List.of(
            agentWorker("gather-data", "...", GatherDataResult.class),
            agentWorker("analyse-anomalies", "...", AnalyseAnomaliesResult.class),
            agentWorker("escalate-anomalies", "...", EscalateAnomaliesResult.class),
            agentWorker("oversight-response", "...", OversightResponseResult.class),
            agentWorker("produce-report", "...", ProduceReportResult.class)
        ));
    }
}
```

### Exception cases

**TravelPlanCaseHub** — adds SubCase bindings alongside workers in `configureCase()`:

```java
@Override
protected void configureCase(CaseDefinition definition) {
    definition.getWorkers().addAll(List.of(...));
    definition.getBindings().addAll(List.of(
        familyVoteBinding("family-vote-a"),
        familyVoteBinding("family-vote-b"),
        familyVoteBinding("family-vote-c")
    ));
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

Worker bookingWorker = Worker.builder()
    .name("book-appointment-agent")
    .capabilityName("book-appointment")
    .function(new AgentWorkerFunction(bookingAgent))
    .build();
```

All other workers in that CaseHub use `agentWorker()`.

**CareEpisodeCaseHub** — stays on `YamlCaseHub` directly. Has no `LifeCaseType` (spawned as sub-case by care-coordination, not resolved by `LifeCaseService`). Overrides `augment(CaseDefinition)` on `YamlCaseHub` directly to add its 2 workers and register descriptors. Fixes both compilation breaks (removes `getDefinition()` override, replaces `cap()` with `capabilityName(String)`). Retains its own `@Inject` fields — the duplication is 2 fields for 1 class, not worth a shared base class.

```java
@ApplicationScoped
public class CareEpisodeCaseHub extends YamlCaseHub {

    private static final LifeAgent AGENT = LifeAgent.HEALTH;

    @Inject
    LifeOpenClawChatModelFactory openClawFactory;

    @Inject
    LifeAgentDescriptorFactory descriptorFactory;

    public CareEpisodeCaseHub() {
        super("life/care-episode.yaml");
    }

    @Override
    protected void augment(CaseDefinition definition) {
        Agent assessAgent = Agent.builder()
            .model(openClawFactory.forAgent(AGENT))
            .systemPrompt("...")
            .responseSchema(AssessPatientResult.class)
            .build();
        Agent careAgent = Agent.builder()
            .model(openClawFactory.forAgent(AGENT))
            .systemPrompt("...")
            .responseSchema(ProvideCareResult.class)
            .build();

        definition.getWorkers().addAll(List.of(
            Worker.builder()
                .name("assess-patient-agent")
                .capabilityName("assess-patient")
                .function(new AgentWorkerFunction(assessAgent))
                .build(),
            Worker.builder()
                .name("provide-care-agent")
                .capabilityName("provide-care")
                .function(new AgentWorkerFunction(careAgent))
                .build()
        ));
        definition.setAgentDescriptors(Map.of(
            AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)));
    }
}
```

No `agentWorker()` convenience method — `CareEpisodeCaseHub` does not inherit from `LifeTypedCaseHub`. Worker construction uses `capabilityName(String)` (new API) and `AgentWorkerFunction` directly. Descriptor registration matches `LifeTypedCaseHub.augment()` exactly.

The prior business-logic-centralization spec (life#27) also excluded CareEpisodeCaseHub from `LifeTypedCaseHub` for the same reason: sub-case hubs have no `LifeCaseType` and must not appear in `Instance<LifeTypedCaseHub>` used for case resolution.

**FamilyVoteCaseHub** — unchanged. Stays on `YamlCaseHub` directly (no augmentation, no agent).

## Migration scope

| File | Change |
|------|--------|
| `LifeTypedCaseHub` (new) | Base class with template method, `agentWorker()`, `lifeCaseType()` |
| `AppointmentCycleCaseHub` | Extend LifeTypedCaseHub, one manual worker |
| `CareCoordinationCaseHub` | Extend LifeTypedCaseHub |
| `ContractorCoordinationCaseHub` | Extend LifeTypedCaseHub |
| `FinancialReviewCaseHub` | Extend LifeTypedCaseHub |
| `HomeMaintenanceCaseHub` | Extend LifeTypedCaseHub |
| `TravelPlanCaseHub` | Extend LifeTypedCaseHub, keeps SubCase bindings |
| `CareEpisodeCaseHub` | Stays on YamlCaseHub; fix compilation breaks (`getDefinition()` override removed, `cap()` replaced with `capabilityName(String)`), own `augment()` override |
| `FamilyVoteCaseHub` | Unchanged |
| `LifeCaseService` | **NOT changed.** Switch on `LifeCaseType` and 6 individual `@Inject` CaseHub fields remain. Switch elimination to `Instance<LifeTypedCaseHub>` deferred to life#27 — this spec adds `lifeCaseType()` to prepare for it but does not refactor the service |
| `openclaw-agent-worker-pattern.md` | Update: individual per-worker construction examples become `LifeTypedCaseHub.agentWorker()` reference for the standard case; `cap()` helper removed; `capabilityName(String)` replaces `capabilities(List<Capability>)`; manual Agent construction documented for `userMessage` exception case; template method contract (`configureCase()`, not `augment()`) documented |

## Testing

No behavioral change — pure structural refactoring. Existing 401 integration tests verify correctness.

New unit test `LifeTypedCaseHubTest`:
- `agentWorker()` builds Worker with name `{capabilityName}-agent` and `capabilityNames` containing the single capability string (verifies `capabilityName(String)` API, not `List<Capability>`)
- Worker name follows `{capabilityName}-agent` convention for all standard workers
- `augment()` calls `configureCase()` then sets agent descriptors — workers added in `configureCase()` are present in the final definition alongside descriptors (ordering: configure first, descriptors second)
- `LifeAgent` passed at construction is the one used for both `openClawFactory.forAgent()` and `descriptorFactory.descriptorFor()` — verified via mock expectations
- Subclass `configureCase()` override adds workers correctly
- `lifeCaseType()` returns the expected enum value per subclass

## What's removed per CaseHub

- `volatile CaseDefinition augmentedDefinition` field
- `getDefinition()` override (10 lines)
- `cap()` helper method (3 lines)
- 2 `@Inject` field declarations (except CareEpisodeCaseHub which retains its own)
- `setAgentDescriptors(...)` call
- Individual worker methods (each ~15 lines) → single `agentWorker()` calls

Net reduction: ~1,100 lines across 7 files.
