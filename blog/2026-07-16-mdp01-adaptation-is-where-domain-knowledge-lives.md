---
layout: post
title: "Adaptation Is Where Domain Knowledge Lives"
date: 2026-07-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [cbr, adaptation, neocortex, domain-rules]
series: issue-55-cbr-adaptation-rules
---

*Part of the CBR series on casehub-life. Previous: [The Last Mile Was the Whole Point](2026-07-14-mdp01-the-last-mile-was-the-whole-point.md).*

The previous entry closed out CBR retrieval and integration — past cases flow to agents, calibration statistics land in the case context. This one picks up the missing piece: what happens between retrieval and consumption.

A past boiler repair in summer with a 7-day SLA is not directly reusable for a winter boiler failure. The urgency is different, the SLA needs tightening, the cost expectation has shifted. Raw retrieval gives you the right neighbourhood; adaptation adjusts the answer for the current address.

The foundation already had `PlanAdapter` sitting in `casehub-neocortex-memory-api` — a single-method SPI that takes a retrieved case and current features, returns an adapted plan with per-step actions (RETAIN, BOOST, SUPPRESS). `NoOpPlanAdapter` was the `@DefaultBean`, doing nothing. The SPI was waiting for someone to implement it.

### Six rules, six domains

I wanted each domain's adaptation logic isolated and testable — no monolithic switch statement, no YAML DSL pretending to be a programming language. The pattern was already established: `LifeCbrDescriptionProvider` has six implementations in `describe/`, one per case type. The adaptation rules mirror that exactly.

`LifeAdaptationRule` is the internal SPI. Each rule is a pure function — no injected dependencies, no database calls. Takes a scored past case and current features in, returns adapted steps out. `ContractorAdaptationRule` checks season and budget deltas. `HealthAdaptationRule` scales follow-up priority by patient risk level. `FinancialAdaptationRule` flags threshold miscalibration when past cases for similar amounts were consistently escalated.

The constraint I imposed — pure functions only — has a real consequence. Trust-score-aware adaptation (the issue asked for "preserve contractor if trust score is still above threshold") would require injecting `TrustGateService`. I deferred it to a separate issue rather than compromise the constraint. The SPI accommodates it: augment `currentFeatures` with trust scores before calling `adapt()`. No interface change needed. The rule stays pure; the caller enriches the input.

### The threshold split

The design review caught something I'd missed. `LifeCbrSuggestionService.suggest()` has a `≥2` threshold — you need at least two past cases for meaningful statistics. But adaptation is most valuable with a single highly-similar past case. One case with a detailed plan trace is exactly the scenario where step-by-step adaptation earns its keep.

We split the thresholds: `retrieveForAdaptation()` returns cases at `≥1` (for adaptation) while computing statistics only at `≥2`. The case start flow writes `adaptedPlan` whenever one case exists, `cbrCalibration` only when two or more do. A single past case that matches well now drives adaptation without being discarded by the statistics threshold.

### What the engine doesn't do yet

The engine's `CbrRetrievalService` retrieves cases and maps them to `RetrievedExperience` — it does not call `PlanAdapter`. Life calls it directly from `LifeCaseService.startCase()`, which is temporary wiring. engine#738 tracks the proper integration: retrieve → adapt → map, so all apps get adaptation for free. Until then, the adapted plan reaches workers through case context — serialised to `adaptedPlan` in the initial context, picked up by `CbrInputTransformer`, formatted alongside raw experiences in `_cbrContext`. Advisory, not directive — agents see the recommendation and make the final call.

The broader pattern here is worth naming: life's CBR layer is doing generic orchestration that belongs in the engine. Feature extraction, suggestion statistics, experience formatting, input transformation — all domain-agnostic. What belongs in life is thin: six description providers, six adaptation rules, domain config. The rest is plumbing that every CBR-enabled app would need to reimplement. That extraction is a separate concern, but the direction is clear — harden upstream, make apps thinner.
