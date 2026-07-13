---
layout: post
title: "Eight Layers of Write, Zero Layers of Read"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [api, analytics, accountability, rest]
---

casehub-life had eight layers of infrastructure — SLA enforcement, commitment tracking, tamper-evident audit, case plan orchestration, trust routing, risk classification, CBR retention — all writing data. WorkItems accumulate. Ledger entries grow. Trust scores compute. Case trackers record outcomes.

None of it was readable through the API.

The ExternalActor endpoint had CRUD and GDPR erasure. The case endpoint had POST. That was it. You could start a contractor coordination case, watch it flow through quote→approval→payment→ledger, and at the end have no way to ask: what's the SLA compliance rate? Which actors have degrading trust? What needs my attention right now?

I decided to fix this with three features, each serving a different concern: entity queries (ExternalActor search, trust history, activity timeline), operational state (pending actions with urgency classification), and historical analysis (case statistics, SLA compliance, trust aggregates). Three concerns, three REST resources — no dashboard abstraction leaking a UI concept into the API.

The pending actions design had a question worth answering: sentinel escalations live inside case context as JSON, not as queryable entities. Do we query the engine's case context for every active case to find unresolved alerts? No. The case plan's `sentinel-escalation` binding already creates a human task WorkItem when `escalationRequired == true`. Anything needing human attention is already a WorkItem. One query surface, one source of truth.

The design review — three rounds, sixteen issues — caught things I'd missed. The trust history endpoint was querying `LedgerEntry.actorId` (the `life-actor:{uuid}` prefixed string) when it should have been querying `LedgerAttestation.subjectId` (a raw UUID). Different field, different entity. The SLA metric was computing average window duration (`expiresAt - createdAt`) instead of average breach latency (`completedAt - expiresAt`) — one measures template configuration, the other measures how late breaches actually are. Both would have compiled and returned plausible numbers.

Implementation was straightforward once the SNAPSHOTs stabilised. The foundation repos had moved — `FeatureValue` replacing raw maps in the CBR store, `TrustRoutingPolicy` gaining a new field, `casehub-engine-work-adapter` renamed to `casehub-work-engine-adapter` with the old artifact sitting stale in `.m2`. The kind of breakage where everything compiles locally but Quarkus refuses to start because Hibernate can't enhance an entity whose bytecode references a class that no longer exists at the old package path.

The analytics endpoint computes p50 and p95 resolution times in Java rather than native SQL `PERCENTILE_CONT`. Simpler, avoids H2/PostgreSQL dialect differences in tests, same result. Trust analytics uses two batch queries against `ActorTrustScore` — global scores and dimension scores — regardless of actor count. Not 3N calls through `TrustGateService`.

The accountability value proposition is now observable. You can ask casehub-life how it's performing and get a quantified answer.
