---
entry_type: note
subtype: diary
title: "CBR — the foundation was already there"
date: 2026-07-10
project: casehub-life
phase: Layer 8
author: mdp
tags: [cbr, neocortex, engine, architecture, platform-coherence]
---

I went into the CBR epic expecting to build a similarity engine, a retrieval pipeline, and a retention mechanism. Three substantial pieces of infrastructure. The issue descriptions assumed the foundation didn't have any of this.

It did. All of it.

`CbrCaseMemoryStore` in neocortex — with InMemory, Qdrant, and NoOp adapters. `CbrSimilarityScorer` with four similarity spec types, weighted per-field scoring, and composite feature+vector blending. `CbrRetrievalService` in the engine, wired to YAML `spec.cbr` configuration, with JQ feature extraction and CASE_LIFETIME caching. `CaseOutcomeObserver` SPI for per-case retention. `RoutingOutcomeRecorder` for per-routing-decision retention. `AgentRoutingContext.experiences()` carrying retrieved cases to routing strategies.

The parent CBR epic (parent#227) had tracked all of this across three waves. The engine issues (#476, #477, #478) were all closed months ago. I filed a duplicate of #477 before finding the parent epic — embarrassing, but the right lesson: check the platform docs before assuming a gap exists.

So what did we actually build? The domain-specific parts that only life can provide. Six `LifeCbrDescriptionProvider` implementations that know what "Contractor: boiler-repair in kitchen, budget 500" means as a problem description. A `LifeCaseOutcomeCbrWriter` that observes case completion and writes the case as a retrievable experience. A `LifeRoutingOutcomeRecorder` that captures every worker execution with its PlanTrace. Feature schemas with `CategoricalTable` similarity for seasons (summer/winter = 0.3, summer/spring = 0.7) and `GaussianDecay` for costs and durations. JQ feature extractors on all six case YAMLs.

The one gap that still matters: no production routing strategy reads the experiences. `TrustWeightedAgentStrategy` passes `AgentRoutingContext.experiences()` through untouched. Engine#505 tracks this. Until it lands, the CBR store accumulates data and the retrieval pipeline runs, but the output has no effect on routing decisions. The data is flowing through a pipe that nobody reads from yet.

The design review caught real issues. `FeatureVectorCbrCase` would have been invisible to `CbrRetrievalService` (it exclusively retrieves `PlanCbrCase.class`). The YAML timing enum needed kebab-case (`case-lifetime`) not Java constants (`CASE_LIFETIME`) — jsonschema2pojo's `fromValue()` maps from the JSON schema value, not the constant name. `CaseOutcomeEvent` has no `tenantId` field — interim constant until the engine adds it.

The architectural takeaway: when the foundation is mature, the application layer's job is domain semantics — what a contractor case looks like, what features make two health appointments similar, which entity is a GDPR data subject. The infrastructure (store, score, retrieve, cache, degrade gracefully) is already solved.
