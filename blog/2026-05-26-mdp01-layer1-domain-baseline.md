---
layout: post
title: "Layer 1 is real code — designing the household domain baseline"
date: 2026-05-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [casehub, quarkus, jpa, flyway, tdd]
---

The first layer of casehub-life is done. Before we got there, I spent more of the session than expected on one design question.

`ExternalActor` needs to be referenced from `HouseholdTask` — that much was clear. The question was how. A typed `@ManyToOne` FK is the obvious JPA move. But casehub-life is a showcase for how the platform handles household automation, and the platform has its own opinion about actor identity.

So I went to the source. `WorkItem.java` in casehub-work:

```java
@Column(name = "assignee_id")
public String assigneeId;
```

No FK. No typed object. An opaque String, because casehub-work doesn't own the actor registry — any application can use it. The platform's `ActorTypeResolver` classifies actors from string prefixes: `agent:*` → AGENT, everything else → HUMAN. There is no "external" type at the platform level at all.

That resolved it. `HouseholdTask.externalActorId` is a raw `UUID` column — no `@ManyToOne`, no cascade decisions, no lazy-loading complexity before we even understand the lifecycle. The clinical harness does the same: `AdverseEvent.enrollmentId` points to `PatientEnrollment` as a UUID with no JPA relationship. When the Store SPI pattern arrives in Layer 2, there's nothing to unwind.

## The Flyway ambush

The test suite threw `"Found more than one migration with version 1.0.0"` on first boot. That error message points nowhere — no migration in the codebase is named V1.0.0. Claude diagnosed it: `casehub-engine-persistence-hibernate` puts `V1.0.0__Create_Quartz_Tables.sql` at `classpath:db/migration`, and Flyway treats that as the same version as casehub-work's `V1__initial_schema.sql`. The `.0.0` suffix looks different; Flyway's versioning scheme disagrees.

The fix is what casehub-clinical already does: disable Flyway in tests and use H2 `drop-and-create`. Hibernate creates the schema from entity definitions; the classpath JAR collision never fires. Flyway still runs in production against real Postgres. It's a real gap — the migrations aren't tested in `@QuarkusTest` — and it's deliberate for now.

## @Blocking — the invisible miss

Claude caught something the tests couldn't. We used `quarkus-rest` (RESTEasy Reactive), which runs on the Vert.x I/O thread by default. Without `@Blocking` on the resource classes, every Panache JDBC call blocks that thread. The `@QuarkusTest` environment doesn't enforce reactive thread discipline — tests pass cleanly — but production throughput would degrade under any real concurrent load. No compiler warning. No startup error.

The fix is one annotation per class. These are now formalised as harness protocols, because I'll forget them when building the next layer.

## What we built

The domain model: `ExternalActor`, `HouseholdTask`, `LifeGoal`, `LifeEvent`. Four REST resources. Four Flyway migrations at V100–V103. A `ShowcaseScenarioTest` that walks a household week — contractor with a missed deadline, health appointment with a silent SLA breach, financial decision with no approval gate. Each test method asserts real domain state; the gap commentary lives in `LAYER-LOG.md`, not in the test code.

One naming decision worth recording: the life-domain actor type is `LifeActorType`, not `ActorType`. The platform already has `io.casehub.platform.api.identity.ActorType` (HUMAN, AGENT, SYSTEM) on the same classpath. A naming collision at Layer 1 is subtle and annoying by Layer 4.

Layer 2 adds casehub-work. The domain model already has `slaHours` on `HouseholdTask` — stored, unenforced, waiting. That's about to change.
