# Agents Need Eyes, Not Just Ears

**Date:** 2026-07-07
**Author:** mdp
**Tags:** casehub-life, layer-7, channel-context, cross-agent, heartbeat

The heartbeat sentinels were structurally complete — provisioned, scheduled, executing, signalling back. But they were context-blind. A grocery-agent querying case state had no idea finance-agent had just posted a budget warning on the delegation channel. Each agent operated in its own case silo, deaf to the household's cross-domain conversation.

The fix was `LifeChannelContextProvider` — a CDI bean that queries recent messages from the household's qhorus channels (delegation, oversight, per-actor) and merges them into the heartbeat context before each agent executes. The design went through adversarial review (3 rounds, 12 issues, $13) which narrowed scope significantly: ambient WorkItem creation was already delivered by life#37, and #8 can't honestly close while skill integration is blocked on openclaw Epic 4.

The interesting design decisions: channel scoping follows the existing topology (delegation always, oversight always, actor channels only when the case involves that actor), actor resolution walks the `WorkItem.callerRef` → `LifeTaskContext.externalActorId` chain, and channel context failures are fault-isolated so a broken channel query never kills a heartbeat tick.

Also consumed three upstream SNAPSHOT breaks in one pass: casehub-work `category` → `types`, qhorus CDI ambiguity fix (qhorus#322 — the Maven exclusion workaround is gone), and the flaky `LifeTaskResourceTest` that raced manual `checkExpired()` against Quartz's `ExpiryTimerJob`. The flaky test fix was the most satisfying — replaced a race with an Awaitility-based assertion that tests the real production path.
