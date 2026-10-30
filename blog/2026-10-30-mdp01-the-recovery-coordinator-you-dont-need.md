---
layout: post
title: "The recovery coordinator you don't need"
date: 2026-10-30
entry_type: article
subtype: diary
projects: [casehubio/life]
tags: [playbooks, failure-recovery, spi-design, triage, cbr, engine-architecture]
series: issue-141-playbook-failure-recovery
---

# The recovery coordinator you don't need

Issue #140 gave casehub-life playbook coordination — LLM-generated
workflows that handle the mechanical parts of a case (fetch contacts,
geocode addresses, pull payment history) so domain workers can focus
on judgment. The obvious next question: what happens when a playbook
breaks at runtime?

The issue spec (#141) described the gap precisely. A connector times
out. A parameter is wrong. An endpoint returns 404 instead of a
contact list. The current code catches the exception, logs a warning,
and falls through to uncoordinated execution. No feedback loop. No
learning. The same broken playbook gets selected again next time.

I started with what looked like the clean SPI boundary. The engine
has `RecoveryCoordinator` — a single-method interface that gets called
when a worker exhausts its retry budget. Life implements it, intercepts
playbook failures, re-invokes the planning worker to triage, applies
the decision. The design spec captured six decisions around this: per-worker
triage scope, structured four-action response schema, persist improved
playbooks to the catalog, engine-side partial failure detection.

Then we read the engine's actual `DefaultRecoveryCoordinator`.

It's 465 lines of sophisticated recovery machinery. Three levels of
structural recovery: transient classification (the agent stays eligible),
compound adaptation (the plan locally patches around the failure), and
full replan (the entire goal decomposition re-runs). Error classification
via an `ErrorClassifier` SPI. Plan versioning with snapshot and delta
tracking. Compound locking for concurrent safety. And at the very end,
after all structural recovery has been attempted or skipped, it calls
`TriageHandler.evaluate()` — a platform SPI for application-level
triage decisions.

Implementing `RecoveryCoordinator` as an `@Alternative` would have
displaced all of this. Every bit of the engine's structural recovery —
the error classification, the compound adaptation, the replan — gone.
Replaced by life's playbook-specific triage. That's not a recovery
strategy; it's an amputation.

The correct extension point was `TriageHandler`. Life implements
`TriageHandler`, the engine's `DefaultRecoveryCoordinator` does its
structural recovery first, and only when that isn't enough does it call
life's triage handler for the playbook-specific judgment call. The
handler re-invokes the planning worker with the failure context, gets
back a structured `TriageDecision` (retry with modified params, switch
template, generate a fix, or fall back to MCP tools), applies it via
`TriageExecutor`, and fires a CDI event for CBR retention.

The four-action triage model maps cleanly onto the engine's
`TriageDecision` sealed interface — `Retry(modifications)` for the
first three actions, `Abandon(reason)` for the fallback. The engine
manages triage round counting internally via `BindingRecoveryState`,
so the per-worker cap I'd designed is already handled by the engine's
recovery policy. The `TriageRoundTracker` I built still exists as a
supplementary in-memory cache, cleaned up by the case lifecycle
observer, but the engine is the authority.

The playbook improvement loop works through `TriageExecutor`. When the
planner returns `GENERATE_FIX`, the executor validates the new YAML
through the existing `CoordinationTemplateValidator` (parseable, has
steps, compiles, all URLs are local connector paths), then persists it
to `CoordinationTemplateCatalog` as a non-curated template. Curated
templates — the eight shipped on the classpath — are protected: a fix
for `contractor-onboarding` gets saved as `contractor-onboarding-fixed`.
Over time the catalog evolves. Broken playbooks get fixed. The fixed
versions accumulate use counts. CBR surfaces the successful alternatives
for similar contexts.

The CBR integration follows the existing pattern. `LifePlaybookCbrWriter`
observes `PlaybookTriageEvent` and writes a `CbrPlanRecord` with
playbook-specific features — template name, failed step count, error
type (categorised: timeout, auth, not-found, server-error), triage
action taken, round number. A new `playbook-triage` feature schema is
registered alongside the six domain schemas. The adaptation rules can
now surface: "for this contractor, the standard playbook failed last
time — the fixed version worked."

The partial failure problem — 3 of 5 connector steps succeed, 2 fail,
and the result looks valid — needs an engine change. `FlowWorkerFunctionHandler`
currently produces `PlaybookExecutionResult.Success` even when steps
fail. The `PartialSuccess` type already exists in the engine API; it was
designed for this and never produced. Five lines of code. Tracked as
engine#1268.

The design pivot is the interesting part. The spec said
`RecoveryCoordinator`. The implementation found `TriageHandler`. Both
are SPIs the engine exposes for exactly this kind of extension. The
difference is where in the recovery pipeline they sit — and what
you lose by choosing the wrong one. Reading the code caught what
reading the interface wouldn't.
