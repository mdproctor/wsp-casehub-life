---
layout: post
title: "The tests that were never green"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [casehub-engine, integration-tests, debugging]
---

# The tests that were never green

I'd been staring at `@Disabled("Blocked by engine#410")` on `AppointmentCycleIntegrationTest` for weeks. Engine#410 — the `CaseDefinition not found after successful registration` bug — was closed. Time to remove the gate and see what happens.

What happened: three distinct failures, none of which was engine#410.

The first surprise was that the `0.2-SNAPSHOT` engine jar in `~/.m2` didn't actually contain the fix. The `RegistryEntry` inner class was present — looked right — but the jar predated the commit that made the fix effective. Maven SNAPSHOT metadata timestamps reflect when the metadata was fetched, not when the source was compiled. I spent real time debugging the wrong problem before rebuilding from source.

With the fix genuinely in place, `startCase()` worked. Workers fired. The appointment-cycle workflow chained correctly: book → confirm → prep → humanTask → record. But the test expected `CaseStatus.WAITING` after the humanTask binding created a WorkItem. The engine doesn't do that. Cases stay `RUNNING` while humanTask WorkItems are pending. `WAITING` exists in the enum but the engine never transitions to it for humanTasks. Nowhere documented.

The second real failure was invisible. Awaitility polls `WorkItem.find("status", PENDING)` on a background thread. Panache needs a transaction context. The Awaitility thread doesn't have one — `ContextNotActiveException`. But Surefire's `rerunFailingTestsCount=2` swallows the first failure: the retry restarts Quarkus, which loses the case definition registry, and the surefire report shows `CaseDefinition not found` — the wrong error entirely.

The third was cross-test interference. Three test methods each start a case that eventually creates a PENDING WorkItem. `WorkItem.find("status", PENDING).firstResult()` is a global query — it returns the first match from any test's case. The golden path test was completing another test's WorkItem. Its own case never finished.

The fixes were mechanical once I understood the problems: `QuarkusTransaction.requiringNew()` wraps every Panache query inside Awaitility lambdas, `callerRef` prefix filtering isolates WorkItems to the right case, and the `WAITING` assertion goes away.

One more discovery along the way: `casehub-engine-ledger`'s `WorkerDecisionEntry` entity was throwing `Unknown entity type — does not belong to this persistence unit`. Adding `io.casehub.ledger.model` to the qhorus PU packages in `application.properties` fixed it — this is a separate package tree from `io.casehub.ledger.runtime`, and it wasn't covered.

233 tests pass now. Zero skipped. The appointment-cycle golden path runs end-to-end: case starts, five workers fire in sequence, humanTask creates a WorkItem, completion signals the engine, record-health-decision writes the ledger entry, goal met, case completed. Layer 5 is done.
