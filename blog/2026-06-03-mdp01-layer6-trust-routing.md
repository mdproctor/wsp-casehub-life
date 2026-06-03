---
layout: post
title: "Layer 6: Trust Routing — Where the Scores Come From"
date: 2026-06-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [trust, attestation, routing, casehub-ledger]
---

## The spec that got rewritten

I started Layer 6 with a spec that looked right: add trust score fields to ExternalActor, wire a routing provider, expose scores on the REST response. Clean, mechanical, done in an afternoon.

The review tore it apart — correctly. Three structural problems:

1. The spec conflated two unrelated concerns. Agent routing (which AI worker handles a case step) and external actor trustworthiness (how reliable is this contractor) are orthogonal. They share dependency wiring but deliver different things.

2. There was no attestation pipeline. Without attestations, `TrustScoreComputer` produces Beta(1,1) = 0.5 for every actor permanently. The javadoc says it plainly: "Unattested decisions contribute nothing — they do not inflate the score." I'd wired the infrastructure that reads trust scores but built nothing that writes the evidence those scores are computed from.

3. The routing policies were keyed to domain categories (`health-coordination`, `contractor-coordination`) but the YAML case definition workers use fine-grained capability names (`book-appointment`, `request-quote`). No mapping between them meant every capability would silently fall through to `TrustRoutingPolicy.DEFAULT`.

The rewrite addressed all three. The attestation pipeline turned out to be the layer's real contribution.

## Attestations make trust real

`LifeOutcomeAttestationWriter` converts WorkItem outcomes into `LedgerAttestation` records. A completed task with an ExternalActor produces a SOUND verdict. An SLA breach produces FLAGGED. Each attestation carries a capability tag derived from the WorkItem's scope path, so trust scores accumulate per-domain: a contractor's `deadline-reliability` is separate from their `cost-accuracy`.

The dimension score for `deadline-reliability` is computed from SLA data — on-time delivery scores 1.0, late delivery degrades linearly over a seven-day grace period. This is an objective, binary signal. The other two dimensions (`factual-accuracy`, `proactive-alerting`) have no automatic signal — they need human or AI assessment, which arrives with Layer 7's OpenClaw agents.

The actorId convention was the prerequisite. The existing ledger writer hardcoded `"life-system"` on every entry, so all trust evidence accumulated against the system rather than individual actors. The fix: `life-actor:{uuid}` for ExternalActor behaviour, `"life-system"` retained for system actions like SLA breach detection.

## The binary else bug

Claude's code review caught something I'd have missed. The verdict mapping:

```java
var verdict = eventType == COMPLETED ? SOUND : FLAGGED;
```

reads as correct — but `LifeDecisionEventType` has three values: CREATED, SLA_BREACH, COMPLETED. Every task creation with an ExternalActor was silently writing a FLAGGED attestation, degrading trust scores before any work was done. The pattern is idiomatic for booleans. On a three-value enum, the else branch silently absorbs the third value with no compilation warning, and the data corruption shows up in the nightly `TrustScoreJob` output — far from where the bug lives.

## Routing policies and the single-candidate limitation

The `LifeTrustRoutingPolicyProvider` maps 32 fine-grained worker capabilities to 8 domain policies. `book-appointment` resolves to the health-coordination policy (threshold 0.75, 10 minimum observations). `request-quote` resolves to contractor-coordination (threshold 0.65, 8 observations). Thresholds and minimum observations are code-level design decisions expressing domain knowledge. Blend factors and quality floors are runtime-tunable via YAML — the operational knobs an administrator adjusts without code changes.

With FuncDSL workers there's exactly one candidate per capability, so `TrustWeightedAgentStrategy` always picks the only option. The routing decision is trivial. This changes at Layer 7 when OpenClaw provides multiple competing agents — at that point the policies, thresholds, and trust scores already exist. The infrastructure is exercised correctly; the decisions just aren't interesting yet.

## What platform coherence prevented

The original spec proposed storing Beta(α,β) fields directly on ExternalActor. PLATFORM.md's boundary rule stopped that: "Do not add trust scoring to casehub-work or casehub-engine. Trust lives in casehub-ledger." The `ActorTrustScore` table in the ledger already stores per-actor, per-capability, per-dimension Bayesian scores. casehub-life reads them via `TrustGateService` — three queries per actor, returned as a `TrustProfile` on the REST response. No duplication, no local storage, no drift.

The protocol tension was worth noting: `trust-maturity-model.md` says "never hard-code trust thresholds" but the devtown reference implementation puts thresholds in code. I filed parent#148 to clarify. The intent is that thresholds are domain design decisions (code), while blend factors and quality floors are operational tuning (YAML). The protocol's wording doesn't distinguish these.
