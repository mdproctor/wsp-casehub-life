---
layout: post
title: "The audit that became the plan"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [architecture, refactoring, domain-descriptors]
---

Before I could write a single line of code for life#27, I needed to find what
I was actually fixing. The issue title said "one-class-per-rule for
LifeActionRiskClassifier" — a narrow refactor of the risk classifier. Pulling
that thread revealed eleven places in six files where domain-specific business
logic was scattered: static maps, switch statements, hardcoded conditionals
keyed on `LifeDomain`, `HouseholdActionType`, `LifeCaseType`.

That's the real problem. Not one class, eleven.

The audit changed the design before we designed anything. We spent the better
part of a session just mapping where knowledge lives versus where it should
live. `LifeOutcomeAttestationWriter` has a static map of domain→capability
strings. `LifeTrustRoutingPolicyProvider` has two correlated static maps (32
entries, 8 more) that have to be updated in sync. `LifeDecisionLedgerObserver`
switches on domain to decide which ledger method to call. `LifeTaskService`
has a `domainFromCategory()` switch that maps category strings to domain enums.
`LifeSlaBreachPolicy` has hardcoded 48-hour/household-admin escalation for
every domain without distinction. Adding a new life domain touches all six.

The right fix: each `LifeDomain` enum value carries a descriptor POJO
(`HealthDomainDescriptor`, etc.) encoding capability tag, routing policy, SLA
escalation, template category, and worker capability names. CDI handler beans
supplement the descriptors with execution behaviour — `DomainLedgerHandler`
implementations for ledger writes, discovered via `@Any Instance<T>`. Service
classes become thin dispatchers. The `LifeCommitmentStrategy` pattern (already
in the codebase, already right) is the reference.

The spec went through five review rounds and caught real problems before code
was written. Two that mattered: I had written `parseAmount()` returning
`OptionalDouble`, then chained `.filter().mapToObj()` on it. Claude came back
with a clean verification: `OptionalDouble` in Java 21 has no `filter()` or
`mapToObj()`. The whole functional chain would have been a compile error. The
fix is `Optional<Double>` — same logic, different type, all methods available.
Second: I had the `LifeSlaBreachPolicy` resolving domain from
`BreachedTask.callerRef()`, stripping the `"life:task/"` prefix to get a
category string. Claude looked up the actual `BreachedTask` record: `taskId`,
`callerRef`, `title`, `candidateGroups`. No `category`, no `scope`. The
callerRef format is `"life:task/{templateRef}"`, not
`"life:task/{domainCategory}"` — so stripping the prefix gives the template
name, and `LifeDomain.fromCategory("life-grocery-order")` returns empty, and
everything silently falls back to HOUSEHOLD. Every domain gets the same 48-hour
escalation deadline regardless. The fix: resolve domain from `LifeTaskContext`
by `taskId`, with a `protected resolveDomain()` hook that unit tests override.

Plan A landed as twelve commits: eight domain descriptor POJOs in `api/`,
`DomainLedgerHandler` interface with three CDI handler implementations,
`LifeTrustRoutingPolicyProvider` rebuilt from forty static map entries to a
`@PostConstruct` capability index derived from the descriptors,
`LifeSlaBreachPolicy` now domain-aware, and all six service/observer classes
reduced to thin dispatchers. The test count for the new classes: each descriptor
is a small POJO, testable with `new HealthDomainDescriptor()` and plain JUnit.
No container, no database.

Plan B is the rest of it — risk rules, commitment escalation additions, case
hub descriptors. The case hub work is where the pattern gets its fullest
expression: worker lambdas that currently live in `AppointmentCycleCaseHub`
move to `AppointmentCycleDescriptor`, with the CDI shell becoming purely a
structural wrapper. All the business logic for appointment cycle coordination
— what a booking decline looks like, what the pre-visit prep worker returns,
what recording a health decision means — in one class, independently readable.

The thing I noticed writing the ADR: the existing `LifeCommitmentStrategy`
implementation was already the correct answer. Three strategies, `applies()` +
`execute()`, discovered via `Instance<T>`, zero dispatcher modification when
adding a fourth mode. We had the right pattern and built the wrong thing next
to it for six more layers. At some point "we should do this consistently" needs
to become a protocol rather than an observation.
