# Decisions — Scenario Engine (cross-platform)

## D1: Pattern vs service

**Choice:** Pattern with thin shared utilities, not a centralized service
**Alternatives:**
- Central scenario service — adds coupling without value; WhatsApp messages and bank transactions have nothing in common structurally
- Pure convention (no shared code) — misses the small reusable pieces (timeline replay, scenario loading)
**Rationale:** CDI profile switching already provides the mechanism. Each connector SPI gets a demo `@Alternative`. The shared piece is scenario file loading and timeline replay — maybe 3 classes.
**Trade-offs:** Each connector must implement its own demo alternative; no enforcement beyond convention.
**Exploration:** deep-analysis
**Status:** captured

## D2: Unified format across TS and Java

**Choice:** Single JSON scenario format consumed by both pages (TS) and platform (Java)
**Alternatives:**
- Separate formats per language — inevitable drift, two specs to maintain
- TS-only (pages spec's current non-goal) — backend has no replay capability
**Rationale:** The temporal semantics (offset, speed, loop, triggers) are identical on both sides. One format, two execution runtimes.
**Trade-offs:** Format changes require coordination across repos.
**Exploration:** quick
**Status:** captured

## D3: Actions are abstract, delivery is per-step

**Choice:** Steps declare an action (what) and a delivery mode (how) independently
**Alternatives:**
- Actions imply their delivery — loses flexibility; "create task" should be REST or UI form depending on context
- Delivery-first model — confusing; the scenario author thinks in actions, not transport
**Rationale:** Same action, different delivery: REST (fast, invisible), ui-form (observable, navigable), simulated (inbound from external system).
**Trade-offs:** Scenario files are slightly more verbose.
**Exploration:** deep-analysis
**Status:** captured

## D4: Reactive chain model for triggers

**Choice:** Steps trigger via time, after-step, or data-condition — same model as pages ScenarioEngine
**Alternatives:**
- Flat absolute timeline only — can't express "after task is created, wait 5s, then WhatsApp reply arrives"
- Imperative script (async/await) — harder to serialise, can't be authored as data
**Rationale:** Pages already designed TimeTrigger, AfterTrigger, DataTrigger. Reuse the same model on the backend.
**Trade-offs:** More complex than flat timeline; requires a trigger evaluation engine.
**Depends on:** D2 (unified format)
**Exploration:** quick
**Status:** captured

## D5: Pages as universal executor

**Choice:** Pages (Quarkus backend + Lit frontend) is the scenario execution engine for the entire platform
**Alternatives:**
- Each app runs its own scenarios — duplicated execution logic, no cross-app coordination
- Standalone scenario service — new infrastructure; pages already has the backend and the UI automation
**Rationale:** Pages has the Java backend (can make REST calls to any casehub service), the UI automation engine (UIActions), and every app already consumes pages components. Natural home for the executor.
**Trade-offs:** Pages takes on a new responsibility; scenario execution becomes a platform concern, not just a UI concern.
**Exploration:** deep-analysis
**Status:** captured

## D6: UI form delivery supports drill-down

**Choice:** A ui-form delivery step is a composed sequence of UIActions (navigate, click, fill, expand, submit, await)
**Alternatives:**
- Single fill-and-submit — can't handle nested forms, expandable sections, multi-step wizards
- Full Playwright-style scripting — too low-level; scenario authors shouldn't write CSS selectors
**Rationale:** Real forms have nested sections, expandable panels, tabbed inputs. The step needs to navigate into them.
**Trade-offs:** Scenario files reference UI structure (selectors) which couples to the component implementation. Mitigation: use data-attributes not CSS classes.
**Depends on:** D3 (delivery modes)
**Exploration:** quick
**Status:** captured

## D7: Connector SPIs get demo alternatives

**Choice:** Every connector SPI (ChatPlatform, CalendarPlatform, future BankFeedPlatform, etc.) ships with a demo @Alternative that serves from scenario data
**Alternatives:**
- App-level mocking — each app reimplements the same mock; no reuse
- No demo impls — apps can only demo with real external systems connected
**Rationale:** The SPI boundary is the natural seam. Demo impl serves pre-loaded data for pull queries and accepts timeline-driven injection for push events.
**Trade-offs:** Every new connector must include a demo impl — adds to the definition of "done" for connectors.
**Depends on:** D1 (pattern-based)
**Exploration:** quick
**Status:** captured

## D8: Clinical DemoDataSeeder migrates to this system

**Choice:** Clinical's existing DemoDataSeeder is the proof-of-concept; migrate it to use the unified scenario format and executor
**Alternatives:**
- Leave clinical as-is — working code diverges from the platform pattern
- Rewrite from scratch — wastes the validated approach
**Rationale:** Clinical already proved the pattern: replay through real service calls, indistinguishable from production. Migration validates the new system against a known-good implementation.
**Trade-offs:** Migration work on a working system; risk of regression.
**Depends on:** D5 (pages as executor)
**Exploration:** quick
**Status:** captured

## D9: Unify with pages #142 (Scenario Engine)

**Choice:** Extend pages #142 to cover full-stack scenarios, not just client-side
**Alternatives:**
- Separate backend scenario system — two engines, synchronisation problem
- Backend-only — loses the UI automation capability
**Rationale:** It's the same system. Actions, triggers, timeline, speed control. Pages #142 designed the client half; this adds backend delivery modes and cross-service coordination.
**Trade-offs:** Changes pages #142's scope; "Non-Goal: Server-side simulation engine" in the original spec needs to be revised.
**Depends on:** D2 (unified format), D5 (pages as executor)
**Exploration:** quick
**Status:** captured
