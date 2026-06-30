---
layout: post
title: "When Duplication Points Upward"
date: 2026-06-30
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [refactoring, template-method, foundation-design]
---

Seven CaseHub classes in casehub-life shared the same structural boilerplate: a volatile field, a double-checked lock in `getDefinition()`, an identical `cap()` helper, identical CDI injections, and 32 worker methods that followed the exact same 8-line pattern. The initial instinct was to extract a base class within life and call it done.

That instinct was wrong — or rather, incomplete. The double-checked lock and the `getDefinition()` override existed in life because `YamlCaseHub` in the engine didn't provide an augmentation hook. Every consumer had to reinvent the same caching pattern. The right fix was to push the foundation work into engine first: make `getDefinition()` final in `YamlCaseHub`, move the caching there, and add a `protected void augment(CaseDefinition)` hook for subclasses. Engine#591 delivered that. Once it landed, the life-side extraction became a consumer of foundation infrastructure rather than a parallel reimplementation of it.

The resulting `LifeTypedCaseHub` uses a template method: `augment()` is final and calls `configureCase()` (abstract), then registers agent descriptors unconditionally. Subclasses declare their workers via an `agentWorker(capabilityName, systemPrompt, responseSchema)` helper that builds the entire Agent→Worker→AgentWorkerFunction chain from three parameters. Each CaseHub dropped from 200+ lines to about 50.

The design review caught a meaningful flaw in the initial approach. I'd designed `augment()` as non-final, with subclasses calling `super.augment()` to trigger descriptor registration. The reviewer correctly identified this as a footgun — a subclass that forgets `super.augment()` silently loses its agent descriptors, and the failure is runtime, not compile-time. Making `augment()` final and introducing `configureCase()` as the abstract hook eliminates that entirely. The correct behaviour is the only possible behaviour.

The same review caught that `CareEpisodeCaseHub` should not extend `LifeTypedCaseHub`. It has no `LifeCaseType` — it's spawned exclusively as a sub-case by care-coordination, never resolved by `LifeCaseService`. Including it in the `LifeTypedCaseHub` hierarchy would pollute `Instance<LifeTypedCaseHub>` CDI discovery with a class that can't fulfil the `lifeCaseType()` contract.

A secondary API break was hiding behind the primary one. `Worker.Builder` no longer has `capabilities(List<Capability>)` — it now stores `Set<String>` and offers `capabilityName(String)`. The `cap()` helper and the `Capability` type disappeared from the worker API entirely. Both breaks — `getDefinition()` final and `capabilities()` removed — had to be fixed atomically across all eight augmenting CaseHubs.

The pattern that emerged is worth naming: when application-layer classes all duplicate the same structural pattern around a foundation extension point, the fix often isn't an application-layer base class — it's a better extension point in the foundation. The duplication is a signal that the foundation API is missing a hook.
