---
layout: post
title: "Fewer Approvers, Stricter Gate"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [action-risk-classifier, trust-routing, casehub-engine]
---

# casehub-life — Fewer Approvers, Stricter Gate

**Date:** 2026-06-07  
**Type:** phase-update

---

## What changed

Household agents can now stop themselves before acting. The `LifeActionRiskClassifier` classifies every proposed action — a grocery purchase above £100, a non-refundable booking, a contractor instruction, a specialist referral — and returns either `Autonomous` (proceed) or `GateRequired` (pause, route to the oversight channel, wait for approval). Workers write zero approval logic.

This is the first piece of Layer 7. It was unblocked by engine#402 shipping `ActionRiskClassifier` as a platform SPI.

## The taxonomy

The classifier's job is thin: parse the action type, look up domain properties, check thresholds if needed, build the gate. The domain properties — always-gated vs. amount-threshold vs. never, reversible or not, who can approve — live in one place: a `HouseholdActionType` enum in `api/`.

Eleven constants. Three gate policies. A sample:

```java
BOOKING_NONREFUNDABLE (ALWAYS,           irreversible, [household-admin])
HEALTH_MEDICATION_FLAG(ALWAYS,           irreversible, [household-admin, household-member])
ELDER_CARE_DECISION   (ALWAYS,           reversible,   [household-admin, household-member])
SPEND_PURCHASE        (AMOUNT_THRESHOLD, reversible,   [household-admin])
HEALTH_APPOINTMENT_GP (NEVER, ...)
```

The alternative would have been a classifier full of switch statements — one for threshold key selection, one for reason strings, one for candidate groups. The enum approach collapses all of that: the classifier dispatches on gate policy and reads everything else from the constant. Adding a new action type means one new enum entry, nothing else.

## The candidateGroups surprise

`GateRequired` carries a `candidateGroups` list — which groups can approve the gate. When the engine chains multiple classifiers together, it picks the most restrictive result. "Most restrictive" is the result with the *fewer* `candidateGroups`.

This is backwards from the obvious reading. More groups sounds stricter — more people involved, more consensus required. But the field is an eligibility list, not a quorum: more groups means more people can approve, which is less restrictive.

So health medication flags and elder care decisions get two groups — `[household-admin, household-member]` — not to make the gate stricter, but to make approval faster. Any adult in the household can respond. Safety gates should not queue behind a single principal's inbox.

Spending and contractor gates get one — `[household-admin]` — because those decisions should go to whoever manages the household budget.

## What reversible actually means

First cut had `SPEND_SUBSCRIPTION_CANCEL` marked `reversible=true`. The reasoning: you can always re-subscribe. But `reversible` in `GateRequired` signals whether the action itself can be undone, not whether a substitute action exists. Re-subscribing after a cancellation is a new contract at potentially different terms. The cancellation is irreversible.

Changed to `false`. A small thing, but the distinction matters when the gate message tells the approver "this cannot be undone."

## The API breaks mid-implementation

Engine#402 landed with three simultaneous breaking changes that only surfaced at compile time:

- `function()` now returns `WorkerResult`, not `Map<String, Object>` — every `.function(lambda)` in the seven case hubs needed `WorkerResult.of(map)` wrapping
- `TrustRoutingPolicy` record gained a sixth field — `bootstrapEscalationRequired boolean` — and the 5-arg constructor no longer exists
- `TrustGateService.dimensionScores()` renamed to `allDimensionScores()`

Nine files, all mechanical. The error message for the first one — "bad return type in lambda expression... Map cannot conform to WorkerResult" — is clear enough once you know what changed. Before that it reads like a generic type inference failure.

## What's still open

`GateRequired.scope` accepts a string that the engine maps to a Qhorus channel. I set it to `"casehubio/life/oversight"` — the platform Path format for the life oversight channel — but the exact mapping in the engine runtime isn't documented. Filed engine#437 to get confirmation.

The classifier is wired. The gates will start mattering once real external agents are executing consequential actions. That's Layer 7 — casehub-openclaw as WorkerProvisioner — and what this was designed for.
