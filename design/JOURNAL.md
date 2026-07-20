# Design Journal — issue-74-household-hub-phase1

## 2026-07-19 — Household Hub design + backend MVP

### Product concept
"Household Hub" — single web app for 3-5 household members. Four concerns:
ambient intake (listen to WhatsApp/email), personal inbox + chat (Claudony),
operational visibility (dashboard), decision journal + reports. Spec designed
and reviewed (3 rounds, 19 issues, all resolved).

### Architecture decisions
- **LifeCaseType carries its LifeDomain** — `domain()` method on enum, not computed at query time. Persisted on LifeCaseTracker via V111 migration.
- **SSE uses snapshot-on-reconnect** — no event replay, no Last-Event-ID tracking. BroadcastProcessor with DROP overflow. Client reconnects get fresh state.
- **LifeEventBroadcaster** — simple pub/sub with CopyOnWriteArrayList, not Mutiny BroadcastProcessor. Simpler, testable without Quarkus, CDI events bridge into it.
- **Visibility policy SPI** — `LifeCaseVisibilityPolicy` mirrors `LifeTaskVisibilityPolicy`. Junior filtered at SPI level; admin/member pass through.
- **Demo mode via Quarkus profile** — `%demo` profile, Flyway seeds at V9000+, static LifeCaseTracker records (no engine dependency).
- **Frontend deferred** — life-ui (Lit SPA via Quinoa) designed but not implemented. Needs blocks-ui package availability investigation first.
