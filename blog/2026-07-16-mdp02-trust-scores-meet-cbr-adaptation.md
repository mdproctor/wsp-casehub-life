---
layout: post
title: "Trust Scores Meet CBR Adaptation"
date: 2026-07-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [cbr, trust, adaptation]
---

# CaseHub Life — Trust Scores Meet CBR Adaptation

**Date:** 2026-07-16
**Type:** phase-update

---

## What I was trying to achieve: close the CBR follow-up backlog

After landing the adaptation rules last session, five follow-up issues remained — three evaluation questions from the code review, plus trust-score-aware adaptation and an integration test for the CBR path through `LifeCaseService`. All small or medium. I wanted to clear the batch in one pass.

## The three that didn't need code

Three of the five issues were evaluation questions — the kind that arise when a code review flags something as "worth thinking about" rather than "definitely wrong."

\#68 asked whether `HealthAdaptationRule` should check `providerId` instead of `careType`. The answer is no — CBR feature comparison operates on categorical attributes (GP vs Specialist), not entity identifiers. Entity-level tracking belongs in the trust layer, which is exactly what \#67 delivers.

\#69 asked whether `AdaptationTrace` should fire when adaptation produces zero steps. The trace still carries the sourceCaseId, similarity score, and current features — all useful for audit even without adapted steps. "We looked and found nothing to change" is a meaningful signal, not noise.

\#66 asked for `FeatureStatistics` to move upstream to `casehub-neocortex-memory-api`. The record is pure math — nearest-rank percentiles on a `double[]`, zero domain dependencies. It belongs there. But neocortex is a peer repo, so I filed [neocortex#157](https://github.com/casehubio/neocortex/issues/157) and closed \#66 as tracked upstream. The life-side is a trivial import change once it ships.

## Trust scores as CBR features

The design for \#67 turned out cleaner than I expected. The adaptation rules are pure functions — they take a feature map and produce adapted steps. No injected dependencies, no service calls. So instead of complicating the rules with trust lookups, we enriched the feature map before calling them.

`LifeTrustFeatureEnricher` sits between retrieval and adaptation in `LifeCaseService.startCase()`. If the initial context contains an `externalActorId`, the enricher queries `TrustGateService` for the global score and four dimension scores (`deadline-reliability`, `cost-accuracy`, `factual-accuracy`, `proactive-alerting`), adding each as a `FeatureValue.NumberVal` entry. Only present scores get added — `OptionalDouble.empty()` means "no data," and the feature stays absent.

The adaptation rules then check these features like any other. A global trust score below 0.3 appends a reason — "consider alternative contractor" for `ContractorAdaptationRule`, "review provider selection" for health and appointment rules. Low dimension scores (below 0.5) boost monitoring capabilities: `watchdog-escalation` for contractors, `maintenance-sentinel` for home maintenance, `health-check` for health providers with low factual accuracy.

Four rules gained trust-aware logic (Contractor, Health, AppointmentCycle, HomeMaintenance). Financial and Travel don't involve external actors, so they stay unchanged. The SPI didn't change — the enrichment is invisible to it.

## Where this leaves us

Layer 8 is now complete through trust-aware adaptation. The only remaining CBR item is \#60 (OpenClaw skill integration), which is blocked on openclaw Epic 4. `FeatureStatistics` upstream move is tracked at neocortex#157.

The enrichment pattern — injecting domain-specific context into the feature map before a generic adapt() call — is worth watching. It keeps the adaptation rules honest (pure functions of features) while letting the orchestration layer bring in whatever context is available. If other signal sources emerge (seasonal pricing data, weather, appointment history), they slot in the same way: enrich the feature map, let the rules react.
