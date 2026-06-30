---
layout: post
title: "Layer 2, and the domain model we thought we had"
date: 2026-05-28
type: pivot
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [casehub, casehub-work, domain-design, quarkus]
---

The plan going into this session was mechanical. Add a DTO layer, wire casehub-work WorkItems to HouseholdTask, implement LifeSlaBreachPolicy. The brainstorming session found something else.

## The entity that wasn't there

I asked Claude to check whether we actually needed a DTO layer for HouseholdTask. Its response: what's the difference between a HouseholdTask and a Task with a Household as a domain tag? I didn't have a good answer.

HouseholdTask has: title, description, domain, deadline, slaHours, assignedTo, externalActorId, recurrence. WorkItem has: title, description, category, assigneeId, claimDeadline, expiresAt, candidateGroups, status. Strip out the field-by-field mapping and they're the same thing. We were wrapping a foundation primitive with a domain label and calling it a domain entity.

The same question applied to LifeGoal. Claude asked how it compared to a CaseInstance. It maps exactly — title and description in case context, targetDate as caseBudgetDeadline, lifecycle status as case status. LifeEvent: a completed case with an outcome recorded in CaseLedgerEntry.

After about an hour of this, we'd established that casehub-life owns almost nothing. One genuine domain entity: ExternalActor. A typed, tracked external party with contact details and trust dimensions — no foundation equivalent, cross-case lifecycle, independent query requirements. Everything else maps to something the foundation already provides.

## What casehub-life is actually for

Stripping out the entities made the application design clearer. casehub-life isn't a CRUD layer on top of foundation WorkItems. It's a configuration layer — WorkItemTemplate definitions that give foundation WorkItems their household vocabulary, CasePlanModel YAML files that define life-domain workflows, SPI implementations that enforce household-specific SLA policy.

The primary interaction surface isn't a custom UI. It's the person's existing calendar, todo app, voice assistant. A Claude skill with voice mode handles natural language; a set of domain-shaped MCP tools lets Claude create contractor coordination cases, query ExternalActor trust scores, check commitment status. The accountability layer is invisible — it just means that when an agent says it will chase the contractor on Thursday, that commitment is tracked, enforced, and escalated if missed. The person sees a calendar update and a message. They never interact with casehub-life directly.

## What we actually built

Layer 2 ended up cleaner than Layer 1. HouseholdTask, LifeGoal, and LifeEvent are gone. ExternalActor stays. LifeTaskContext is a thin supplement — workItemId, domain, externalActorId, recurrence — carrying only what WorkItem doesn't have a field for.

`POST /life-tasks` resolves a named WorkItemTemplate, validates the optional ExternalActor reference, and creates a WorkItem and LifeTaskContext atomically. No candidateGroups override on the request — groups come from the template exclusively, because LifeSlaBreachPolicy's tier detection reads candidateGroups to determine whether escalation has already happened.

The policy itself is nine lines:

```java
if (ctx.task().candidateGroups().contains("household-admin")) {
    return new BreachDecision.Fail("life-sla-exhausted");
}
return BreachDecision.EscalateTo.to("household-admin")
    .withDeadline(Duration.ofHours(48));
```

First breach: escalate to household-admin, 48-hour window. Second breach — household-admin is now in candidateGroups because EscalateTo put it there — fail terminally. No state stored anywhere. The WorkItem's own candidateGroups encodes the escalation tier.

## The engine SNAPSHOT ambush

Every `@QuarkusTest` failed during augmentation with:

```
Unable to load type: io.casehub.engine.internal.event.SignalReceivedEvent
    at io.quarkus.vertx.deployment.VertxProcessor.tryLoad
```

The engine SNAPSHOT had refactored internal event classes to a new package in casehub-engine-common, but casehub-engine-work-adapter still referenced the old paths in its built-in Jandex index. The failure was masked on any machine that had a cached Quarkus augmentation from before the refactor. Fresh build, hidden problem.

The right fix for a Layer 2 harness: remove all casehub-engine dependencies. They're Layer 5 concerns — CasePlanModel orchestration, humanTask bindings, work-adapter integration. Layer 2 only needs casehub-work. The engine team gets two filed issues; we get a clean test run.

Layer 2 is done. The domain model is smaller and more honest than Layer 1's was.