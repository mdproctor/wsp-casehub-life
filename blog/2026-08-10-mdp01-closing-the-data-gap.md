---
layout: post
title: "Closing the Data Gap"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [life-ui, routing, cbr, channels, inbox, data-wiring]
series: issue-85-life-ui-data-wiring
---

The Life UI scaffolding from the previous session put tabs on every view — Routing, CBR, Channels — each pointing at endpoints that didn't exist yet. This session built those endpoints and gave the inbox the ability to tell different action types apart.

The routing endpoint was the most interesting because it crosses a boundary most query services don't. Routing decisions live in the tamper-evident ledger as `WorkerDecisionEntry` records — one per worker execution per case, each carrying the trust score at routing time, the threshold applied, and a rationale string. The endpoint joins this with `LifeTrustRoutingPolicyProvider`, which resolves the active policy for each capability tag (threshold, blend factor, quality floors). The blocks-ui `<blocks-routing-rationale>` component gets the full picture: who was selected, what score they had, what the policy required.

The CBR endpoint required a design decision. The CBR store is query-based — you retrieve similar cases by constructing a feature vector, not by looking up a case ID. The original retrieval happens during `startCase()`, when the feature vector is available from the initial context. By the time a user opens the CBR tab, that context is gone. Re-querying would mean reconstructing the features from scratch, and the store has no "find by case ID" method.

We persisted the precedent data instead. A `cbrPrecedentsJson` TEXT column on `LifeCaseTracker`, populated at case start from the `ScoredCbrCase` results. The endpoint reads it back and deserializes. One migration, no new entity, no re-computation. The tradeoff is that the data is frozen at case start — but CBR precedents don't change after that, so it's the right snapshot.

The channels endpoint follows the same query pattern as `LifeChannelContextProvider` — resolve delegation, oversight, and per-actor channels, scan messages, map to the `QhorusMessage` shape. Per-actor channels are inherently case-scoped (linked via WorkItem → LifeTaskContext → externalActorId). Delegation and oversight are shared channels, so the UI shows recent activity across all cases.

The inbox was overdue for discrimination. Until now, `<work-item-workbench>` rendered every pending action identically — no visual distinction between an oversight gate waiting for approval and a routine grocery task. The workbench doesn't support custom detail rendering, so we replaced it with a custom split-pane that checks `actionType` and renders the appropriate component: `<blocks-approval-gate>` for oversight decisions, `<blocks-sla-indicator>` for watchdog alerts, delegation info for delegated tasks.

Making watchdog alerts visible required one upstream fix. `LifeWatchdogAlertObserver` creates escalation WorkItems using the `life-escalation` template, which produces a callerRef of `"life:task/life-escalation"`. We check that callerRef before falling through to commitment-based type resolution. No new field, no new entity — the convention was already there, just unused for classification.

A pattern emerged across all five endpoints: `Optional<List<T>>` as the return type, where the Optional encodes case existence (empty → 404) and the List encodes data presence (empty → 200). We formalised it as a protocol.

The second half of the session was diagnostic work — three infrastructure bugs surfaced during the close sequence.

The first was the most expensive. Five test classes (15 tests) were returning HTTP 500 instead of the expected 404, 422, or 409. No server-side exception trace. The correlation pointed at `FixedCurrentPrincipal` — tests that injected it passed, tests that didn't failed. Claude dug through the CDI chain, checked for custom exception mappers in the project (none), and eventually found the real culprit in a dependency JAR: `LedgerExceptionMapper` in `casehub-ledger-rest` implements `ExceptionMapper<RuntimeException>`, catches everything including `WebApplicationException` subclasses, and silently converts unrecognised types to 500. The `FixedCurrentPrincipal` correlation was a red herring — those tests happened to test success paths where no exception was thrown. The fix is a one-class workaround: register `ExceptionMapper<WebApplicationException>`, which is more specific in the JAX-RS type hierarchy and wins resolution. Filed ledger#191 for the upstream fix.

The second was the work-end handoff detection. The `work_router.py` `_handoff_references_branch()` function reads `HANDOFF.md` from `git show main:HANDOFF.md` — but when the workspace is on a feature branch, the handoff was written there, not on main. The function reports `HAS_HANDOFF=no`, the continue flow takes the "first session" path, and the handoff narrative is lost. Filed soredium#206.

The third hit during the rebase onto main after `issue-81` was squash-merged. Git silently dropped all our commits — "skipped previously applied commit" in the hints, zero commits ahead of main after rebase. The squash commit's diff encompassed all the changes our commits also produced, so git concluded they were already applied. Recovery was `git cherry-pick` from the reflog. The lesson: `--reapply-cherry-picks` exists for exactly this case, but it's easy to miss when git doesn't error.

The routing tab frontend wiring needs to be re-applied onto the base branch's redesigned `cases-view.ts` — the rebase conflict resolution kept the base's version (which has filters, domain icons, and SSE event integration) but dropped our tab additions. The backend endpoints are all in place and tested; the re-wiring is purely frontend template work.
