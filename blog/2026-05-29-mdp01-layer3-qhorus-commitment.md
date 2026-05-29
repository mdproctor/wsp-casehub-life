---
layout: post
title: "Layer 3: Formal Commitment Tracking with casehub-qhorus"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [casehub, qhorus, commitment-lifecycle, oversight-gate, quarkus]
---

The question Layer 3 answers is simple: what happens when a contractor says "I'll be there Thursday" and then isn't? In Layer 2, a WorkItem expires and LifeSlaBreachPolicy escalates. That's the right answer for tasks the system creates. But a contractor commitment is different — it's an obligation from outside the system, tracked against a specific promise, with a Watchdog that should fire the moment the window closes and nobody's confirmed completion.

casehub-qhorus is the layer that handles this. COMMAND/RESPONSE pairs with deadlines, Watchdogs watching for expired open Commitments, oversight channels that block major decisions until a human responds. I added three accountability patterns: family delegation (COMMAND to household-member, Watchdog by 3pm), contractor follow-up (COMMAND on a per-actor channel, Watchdog at the committed window close), and oversight gates for financial decisions (COMMAND to life/oversight; no WorkItem created until household-admin says yes).

That third pattern took the most thinking. The natural implementation is to create a WorkItem for the gated decision and mark it blocked until approval. But that's wrong — a WorkItem shouldn't exist for work that hasn't been approved. The right model: dispatch COMMAND, persist a `LifeCommitmentRecord` with `workItemId = null`, and let `LifeOversightResponseObserver` create the WorkItem when the RESPONSE arrives. The Watchdog on the oversight channel fires if the gate expires without approval and creates an escalation task instead.

The strategy dispatch went through an SPI approach — I'd asked Claude to design toward that model during brainstorming. The result is a sealed context hierarchy (`DelegationContext`, `ContractorContext`, `OversightContext`) and a `LifeCommitmentStrategy` interface with three `@ApplicationScoped` implementations, collected via CDI `Instance<LifeCommitmentStrategy>` with an exactness assertion (zero matches or more than one both throw). We put the SPI in `app/` rather than `api/` after the first code review caught that `CommitmentContext` holds JPA entities — placing it in `api/` would create a circular Maven dependency.

Implementing against the qhorus API produced two surprises worth recording. `MessageDispatch.Builder` requires `.actorType()` — omitting it throws `IllegalArgumentException: actorType is required` at `build()` with no compile-time warning. Nothing in the docs flags it as required. And `WatchdogAlertEvent` has no `correlationId` field. I'd designed `LifeWatchdogAlertObserver` to look up the commitment record by correlationId from the event — that method doesn't exist. The event carries `notificationChannel` (the channel NAME, not UUID), aggregate context (pending count, oldest expiry), and that's it. The observer query had to switch to `LifeCommitmentRecord.findExpiredPendingByChannel(event.notificationChannel(), now)` — one query for all expired records on that channel.

There was also a casehub-work SNAPSHOT drift mid-session that produced `NoSuchMethodError: SelectionContext.<init>(String, String, String, String, String, String, String, String)` when the scenario test ran. `SelectionContext` had been updated in casehub-work-api to take a `Set<Capability>` at position 3; the runtime jar was compiled against the all-String version. Reinstalling casehub-work from source (`mvn install -DskipTests -pl api,core,deployment -am`) fixed it.

The spec went through three rounds of code review, each catching something real: the first found a circular dependency in where the SPI lived; the second found `WatchdogAlertEvent` had no `correlationId` and that `OversightGateRequest.deadline` was typed as `String` (copy-paste error); the third found `actorType` was missing from the builder calls, among several other gaps. We filed two GitHub issues for findings that weren't worth fixing this session — life#17 for the Watchdog escalation integration test (the `@ObservesAsync` handler doesn't run synchronously so Awaitility timing was unreliable), life#18 for REST resource consistency.

63 tests pass. Layer 4 is casehub-ledger — tamper-evident records for health decisions, financial decisions, and legal actions.
