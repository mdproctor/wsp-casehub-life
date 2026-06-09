---
layout: post
title: "Sixty-two percent"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [testing, qhorus, channel-slug, uuid]
---

`CommitmentLifecycleScenarioTest` was flagged in the handover as a pre-existing isolation failure — passes in the full suite, fails alone. The obvious read: Surefire retries the failing method without running the setup test first, so the static actor ID is null. Fix the static field dependency, done.

That's not what was happening.

Running the class alone passes all five tests. Running the full suite fails Order(2) with a 500. The surefire retry then fails with a null path parameter — but that's a consequence of the retry, not the cause.

The XML report has a suppressed exception buried after the assertion failure:

```
Invalid channel name segment '768203c4-2c44-4f58-8320-b0ca915847b3'
— must match [a-z][a-z0-9]*(-[a-z0-9]+)*
Full name: 'life/actor/768203c4-2c44-4f58-8320-b0ca915847b3'
```

`ChannelSlugValidator` requires each path segment to start with a letter. UUID hex strings start with `0–9` in 62.5% of cases. When the actor UUID starts with a digit, the contractor channel fails to register, the commit endpoint returns 500, and the test fails.

When the class runs alone: the test gets a fresh UUID. If it starts with `a–f`, everything works. If it starts with a digit, it fails — but because of H2 database isolation, the retry gets a new class instance with null static fields, hiding the real error behind an NPE.

When the full suite runs: the test doesn't get the UUID from Order(1) in the retry — Surefire picks up the failed method and reruns it standalone. But Order(1) never ran in that retry context, so `boilerTaskId` is null. The root cause (slug validation) is hidden behind the secondary NPE.

The fix is one character of prefix: `"life/actor/" + id` → `"life/actor/ext-" + id`. Two files, two lines, 277/277.

The interesting part is that the test had been failing in exactly this way since the Layer 3 contractor commitment was written — it just looked like an isolation problem. The TENANCY_ID fix from life#32 unblocked the seed step and exposed it.
