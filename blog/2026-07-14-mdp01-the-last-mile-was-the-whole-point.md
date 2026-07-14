---
layout: post
title: "The Last Mile Was the Whole Point"
date: 2026-07-14
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [cbr, engine-integration, agent-enrichment]
series: issue-56-cbr-engine-integration
---

*Part of a series on [#56 — CBR engine integration](https://github.com/casehubio/life/issues/56). Previous: [CBR: The Foundation Was Already There](2026-07-10-mdp01-cbr-the-foundation-was-already-there.md).*

The CBR epic started with a surprise — the foundation already had most of the machinery. Retention, retrieval, similarity scoring, feature schemas — all present in neocortex and the engine. I wrote about it at the time: the hardest part was realising how little life needed to build.

What I didn't write about was the gap that remained. Data was flowing into the CBR store and routing strategies were consuming it for agent selection, but the agents themselves — the workers that actually make decisions — were blind to it. A health agent scheduling a follow-up had no idea that similar cases resolved in 14-21 days. A contractor agent issuing a quote request didn't know that similar jobs cost £200-350 historically.

Engine#707 closed that gap from the engine side. `WorkerContext` gained an `experiences` field, and `SyncAgentWorkerFunctionHandler` sets it on the thread before calling `agent.execute()`. But `Agent` is final and its system prompt is immutable at construction. The data arrives at the door; nobody opens it.

The design we settled on has two paths. Path 1 is automatic — an `inputTransformer` registered on every Agent that reads `WorkerExecutionContext.current().experiences()` at execution time and merges formatted CBR context into the user message. Every LLM worker sees it without any per-worker configuration. The transformer runs inside `Agent.execute()` after the engine sets the thread-local — this isn't a hack, it's the designed extension point. Path 2 is deterministic — `LifeCbrSuggestionService` queries the CBR store at case start, computes statistical summaries of numeric features from similar past cases, and writes a `cbrCalibration` field to the case context. Workers see p50 costs, typical durations, and historical success rates as structured data.

The two paths serve different purposes and that's intentional. `cbrCalibration.featureStats.estimatedCost.median` tells a contractor agent "similar jobs typically cost £380-450." `_cbrContext` tells it "here's a specific case where a contractor quoted £450 and was negotiated to £380." Statistics calibrate thresholds; examples inform reasoning. A design reviewer pushed on whether they could conflict — they can't, because they answer different questions.

The refactoring was worth doing. `LifeCbrFeatureExtractor` consolidates what had been three independent copies of the same registry-lookup → CbrConfig-check → JQ-evaluation → FeatureValue-conversion pipeline. The retention writers (`LifeCaseOutcomeCbrWriter`, `LifeRoutingOutcomeRecorder`) and the new suggestion service all delegate to it. Three sites down to one.

With engine#505, #683, and #707 all closed, the CBR data path is complete: cases generate experiences at completion, experiences are retrieved during routing and worker execution, and workers incorporate them into their decisions. The store accumulates; the system learns. What's still missing is the adaptation step — #55's REVISE rules that would modify CBR entries based on outcome feedback — but there's no foundation SPI for that yet.
