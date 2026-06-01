---
layout: post
title: "Layer 5: Eight Workflows and an Engine Bug"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [casehub, engine, case-plan-model, yaml, dsl, quarkus]
---

Layer 5 adds casehub-engine — the orchestration layer that turns sequences of workers into multi-step case workflows. Where Layers 2–4 gave casehub-life individual capabilities (SLA enforcement, commitment tracking, tamper-evident audit), Layer 5 wires them into coordinated flows: an appointment cycle that books, preps, waits for the human visit, then records the outcome; a travel plan where family members vote on the destination before anyone books flights.

I decided to build all eight case definitions in one layer rather than the incremental approach AML and clinical took (one and three respectively). The reasoning: casehub-life exists to demonstrate the full breadth of engine capabilities in a single harness — parallel execution, adaptive gates, M-of-N SubCase quorum, cross-case signals, milestones, FuncDSL workers. Splitting them across layers would have produced seven more issues and seven more branch ceremonies for work that shares the same infrastructure.

Each definition is a paired YAML + fluent Java DSL. The YAML defines the case structure — bindings, workers, sentries, goals. The DSL companion builds the same structure programmatically, used for tests and for augmentation when YAML doesn't support a feature. We formalised this as a protocol (PP-20260518): every YAML must have a DSL companion, and both are equal authoring paths.

The most interesting structural pattern was the travel-plan. It runs four workers in parallel (research-destinations, check-budget, check-calendar, check-weather), then hits an adaptive gate: if the estimated cost exceeds a threshold, a family-vote SubCase spawns requiring M-of-N household members to approve before booking proceeds. The family-vote itself is a separate CaseDefinition — a lightweight M-of-N child case that creates one WorkItem per voter and completes when the quorum threshold is met. The M-of-N fields (groupId, totalInGroup, requiredCount) turned out to be DSL-only — YAML doesn't support them — so the travel-plan's `YamlCaseHub` augments the YAML definition with Java code that adds the SubCase bindings.

The scope retrofit was a prerequisite I hadn't anticipated. WorkItem scope was a flat string `"life"` — useless for the `LifeDecisionLedgerObserver`, which needs to know the domain (HEALTH, FINANCE, LEGAL) to write the correct ledger entry subclass. We changed scope to a hierarchical path format: `casehubio/life/household`, `casehubio/life/health`, etc. The observer now resolves domain from the scope Path directly, falling back to LifeTaskContext only when the scope doesn't contain a domain segment. Engine-created WorkItems produce correct ledger entries without needing supplements.

CDI wiring for the engine's in-memory persistence layer took some debugging. Quarkus needs three `@Alternative` beans — `MemorySubCaseGroupRepository`, `MemoryPlanItemStore`, `MemoryReactivePlanItemStore` — registered in `quarkus.arc.selected-alternatives`. Without them, the engine silently falls back to no-op implementations and cases start but never progress.

Worker functions use FuncDSL (PP-20260531) — `FuncWorkflowBuilder.workflow().tasks(FuncDSL.function(...)).build()` — rather than raw lambdas. Each worker is a `Function<Map<String, Object>, Map<String, Object>>` that receives the case input and returns output data that flows into subsequent bindings. The contractor-coordination case's completion worker demonstrates cross-case signalling: when a contractor job completes, the worker looks up active financial-review cases via `LifeCaseTracker` and dispatches a signal to trigger reconciliation.

The layer produced 199 tests — 67 definition tests (YAML loads correctly, binding counts, goal verification) and 129 CaseHub/service tests. Three integration tests exist for the appointment-cycle golden path, decline recovery, and sequential booking verification, but they're disabled. The engine SNAPSHOT has a runtime bug where `SchedulerService.getCaseDefinition()` returns null after a case definition was successfully registered at startup — a forward lookup on `ConcurrentHashMap` fails despite a reverse lookup succeeding on the same map. Filed as engine#410. The hypothesis is Vert.x event bus serialization dropping fields from the `CaseMetaModel` key during the `CaseStartedEvent` handler chain, but that's unconfirmed.

That one bug is all that stands between Layer 5 and closure. The LAYER-LOG entry needs writing, then the branch can close. Layer 6 is trust routing — Bayesian Beta scores on ExternalActors, updated from WorkItem outcomes and commitment attestations, used by the engine's `TrustWeightedSelectionStrategy` to route work to the most reliable agent for each domain.
