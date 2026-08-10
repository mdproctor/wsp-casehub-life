# Cross-Platform Scenario Engine — Unified Demo and Verification

**Date:** 2026-08-11
**Status:** Draft
**Scope:** Cross-repo (parent, pages, connectors, engine, work, life, clinical, iot, aml)
**Pages reference:** casehub-pages#140 (DataSource), casehub-pages#142 (Scenario Engine)

## 1. Goal

Any CaseHub application can run scripted demos and automated verifications
using a single scenario file. The same file drives backend integration (REST
calls, simulated inbound events, bulk data loads) and frontend automation
(form fills, navigation, observable interactions). The application code
cannot distinguish scenario data from real data.

Pages is the execution engine. Its Quarkus backend makes REST calls to any
CaseHub service. Its frontend engine automates the UI. The scenario controls
(`<scenario-controls>`) provide play/pause/speed/step for live demos (slow,
visible) and automated verification (fast, headless).

## 2. Principles

1. **Actions are abstract.** A step declares what happens ("create task"),
   not how. Delivery mode (REST, UI form, simulated inbound) is a property
   of the step, not the action.

2. **The app never knows.** Every external integration SPI ships with a demo
   `@Alternative`. CDI profile switching selects the implementation. No
   conditional logic in application code.

3. **One format, two runtimes.** A single JSON/YAML scenario format consumed
   by both TypeScript (pages ScenarioController) and Java (connector demo
   impls). Temporal semantics (offset, speed, loop, triggers) are identical.

4. **Concurrent by default.** Steps are a directed graph connected by
   triggers, not a sequential list. Bulk loads run asynchronously while
   other steps continue. Synchronisation is explicit via `AfterTrigger`.

## 3. Scenario Format

### 3.1 Top-level structure

```yaml
scenario: life-household-demo
description: "A week in the life of a family — tasks, contractors, health, finance"
speed: 1                          # default playback speed
loop: false                       # restart when complete

data:                             # external data files (relative paths)
  bank-transactions: "data/bank-transactions-6mo.json"
  whatsapp-history: "data/whatsapp-messages.json"
  calendar-events: "data/family-calendar.json"

steps:
  - name: seed-actors
    action: load-external-actors
    delivery: rest
    endpoint: POST /external-actors
    data:
      source: "data/actors.json"
      mode: bulk

  - name: plumber-message
    trigger: { after: seed-actors, delay: 5000 }
    action: whatsapp-message-arrives
    delivery: simulated
    target: chat
    data:
      from: "Bob's Plumbing"
      text: "Thursday 2pm works for us"

  - name: create-task-visible
    trigger: { after: plumber-message, delay: 3000 }
    action: create-task
    delivery: ui-form
    data:
      title: "Chase Bob for quote"
      domain: "CONTRACTOR_COORDINATION"
      deadline: "+3d"
    steps:
      - navigate: "#home"
      - click: "[data-action='new-task']"
      - fill: { from: data }
      - click: "[data-expand='scheduling']"
      - fill: { recurrence: "weekly" }
      - click: "[data-action='submit']"
      - await: { event: "work-item-created" }
```

### 3.2 Delivery modes

| Mode | What happens | Visibility | Use case |
|------|-------------|------------|----------|
| `rest` | HTTP call to target app API | Invisible | Fast seeding, background state |
| `ui-form` | Navigate, fill fields, click submit | Fully visible | Demo audience watches data entry |
| `simulated` | Inject into connector SPI demo impl | Invisible to app, visible in UI via SSE | External events (messages, transactions, sensors) |

### 3.3 Data shapes

| Shape | `mode` value | Behaviour |
|-------|-------------|-----------|
| **Bulk** | `bulk` | Async ingestion of full dataset. May take time. Other steps continue. |
| **Stepped** | `stepped` | One item at a time, paced by scenario speed. Each item is a step. |
| **Stream** | `stream` | Continuous emission at configured interval. Runs until scenario ends or step is cancelled. |
| **Single** | (default) | One data payload, one action. |

### 3.4 Data sourcing

Data can be **inline** (in the scenario file) or **external** (referenced path):

```yaml
# Inline — small payloads
- action: create-task
  data:
    title: "GP follow-up"
    domain: "HEALTH"

# External — large datasets
- action: load-transactions
  data:
    source: "data/bank-transactions-6mo.json"
    mode: bulk
```

External files are resolved relative to the scenario file. Scenario files
and their data files ship together as a directory.

### 3.5 Triggers

Same model as pages ScenarioEngine (#142):

| Trigger | Fires when | Example |
|---------|-----------|---------|
| `TimeTrigger` | Scenario clock reaches offset | `{ at: 10000 }` — 10s into scenario |
| `AfterTrigger` | Named step completes | `{ after: "seed-actors", delay: 5000 }` |
| `DataTrigger` | Dataset matches predicate | `{ when: { dataset: "tasks", match: { status: "COMPLETED" } } }` |

Steps without a trigger fire at T=0. Multiple steps can share the same
trigger — they execute concurrently.

### 3.6 UI form delivery — drill-down

A `ui-form` delivery step is a composed sequence of UIActions:

```yaml
steps:
  - navigate: "#cases"                      # go to the right view
  - click: "[data-case-id='abc123']"        # select a case
  - click: "[data-tab='commitments']"       # drill into a tab
  - click: "[data-action='new-commitment']" # open nested form
  - fill: { mode: "DELEGATION", delegateTo: "Sarah" }
  - click: "[data-expand='deadline']"       # expand nested section
  - fill: { deadline: "+7d" }
  - click: "[data-action='submit']"
  - await: { event: "commitment-created" }
```

Targets use `data-*` attributes, not CSS classes — decoupled from styling.
The `await` at the end confirms the backend processed the submission.

## 4. Connector SPI Demo Pattern

Every connector SPI gets a demo `@Alternative @Priority(300)` activated
by `@IfBuildProfile("demo")`.

### 4.1 Two modes, same interface

**Pull** — app queries for data. Demo impl serves from pre-loaded dataset.
```java
@IfBuildProfile("demo")
@Alternative @Priority(300)
public class DemoChatPlatform implements ChatPlatform {
    // Loaded from scenario data file at startup
    // GET /messages returns from this dataset
}
```

**Push** — events arrive asynchronously. Demo impl accepts injections from
the scenario executor via a scenario injection endpoint.
```java
@Path("/scenario/inject")
public class ScenarioInjectionResource {
    @POST @Path("/chat")
    public void injectChatMessage(ReceivedMessage message) {
        // Fires the same CDI event as real WhatsApp adapter
        chatInboundEvent.fire(message);
    }
}
```

The scenario executor (pages backend) calls the injection endpoint. The app
processes the injected event identically to a real inbound message.

### 4.2 SPIs requiring demo alternatives

| SPI | Module | Pull (queries) | Push (events) |
|-----|--------|---------------|---------------|
| `ChatPlatform` | connectors/chat-spi | Message history, channels | Inbound messages |
| `CalendarPlatform` | connectors/calendar-spi | Event queries | Event created/updated |
| `BankFeedPlatform` | connectors (new) | Transaction queries | Transaction notifications |
| `EmailPlatform` | connectors (new) | Inbox queries | Inbound emails |
| `DeviceProvider` | iot | Device state, sensor readings | Sensor events, state changes |

### 4.3 Scenario data lifecycle

1. **Startup:** Demo impl loads the scenario's external data files (bulk
   pull datasets). Available immediately for query.
2. **Runtime:** Scenario executor injects push events via injection
   endpoint at times defined in the scenario file.
3. **Teardown:** H2 in-memory database. No cleanup needed — restart clears
   everything.

## 5. Pages as Universal Executor

Pages' Quarkus backend becomes the scenario coordinator:

1. **Load** — parse scenario file (JSON/YAML), build trigger graph
2. **Schedule** — register triggers with `ScenarioController`
3. **Execute per delivery mode:**
   - `rest` → HTTP call to target app API
   - `simulated` → HTTP call to target app's `/scenario/inject/{connector}` endpoint
   - `ui-form` → dispatch UIActions to pages frontend ScenarioEngine
4. **Coordinate** — `ScenarioController` manages speed, pause, step, elapsed
   time across both backend and frontend execution
5. **Observe** — `<scenario-controls>` and `<dataset-explorer>` from pages
   #142 Phase 7 provide the demo operator UI

### 5.1 Cross-service coordination

Pages backend acts as orchestrator. It doesn't import app-specific code —
it makes HTTP calls. Any CaseHub app with REST endpoints can be a scenario
target without pages knowing its internals.

```
┌─────────────────────────────────────────────────┐
│  Pages (Scenario Executor)                       │
│                                                  │
│  ScenarioController ─── speed, pause, timeline   │
│       │                                          │
│       ├── REST delivery ──→ POST /life-tasks      │
│       ├── REST delivery ──→ POST /trials          │
│       ├── Simulated    ──→ POST /scenario/inject/ │
│       └── UI-form      ──→ UIAction dispatch      │
│                              │                    │
│                         ┌────┴────┐               │
│                         │ Browser │               │
│                         │ (Lit)   │               │
│                         └─────────┘               │
└─────────────────────────────────────────────────┘
         │              │              │
    ┌────┴───┐    ┌────┴───┐    ┌────┴───┐
    │  life  │    │clinical│    │  iot   │
    │ :8080  │    │ :8081  │    │ :8082  │
    └────────┘    └────────┘    └────────┘
```

### 5.2 Speed control

| Mode | Speed | UI form visibility | Use case |
|------|-------|-------------------|----------|
| Demo | 0.5x–1x | Visible — slow form fills, annotations | Sales demo, conference |
| Normal | 1x | Visible — real-time pace | Development testing |
| Fast | 10x–100x | Skipped or instant | Automated verification |
| Step | manual | One step at a time | Debugging |

In `fast` mode, `ui-form` delivery can optionally degrade to `rest`
delivery for speed — same action, invisible execution. Configurable
per scenario.

### 5.3 Await and verification

Every step can include an `await` that confirms the backend processed the
action:

```yaml
- await: { event: "work-item-created" }           # SSE event
- await: { endpoint: "GET /life-tasks", match: { title: "Chase Bob" } }  # poll
- await: { delay: 2000 }                           # simple wait
```

For automated verification, `await` becomes an assertion — if the expected
state doesn't materialise within a timeout, the scenario fails with a
diagnostic.

## 6. Migration Path

### 6.1 Clinical DemoDataSeeder (first migration)

Clinical's existing `DemoDataSeeder` replays full trial scenarios through
real service calls. Convert to a scenario file:

```yaml
scenario: clinical-trial-demo
steps:
  - name: seed-trial
    action: create-trial
    delivery: rest
    endpoint: POST /trials
    data: { source: "data/trial-seed.json", mode: bulk }

  - name: screen-patient
    trigger: { after: seed-trial }
    action: screen-patient
    delivery: rest
    endpoint: POST /trials/{trialId}/screening
    data: { patientId: "patient-001", eligibility: "ELIGIBLE" }

  - name: report-ae
    trigger: { after: screen-patient, delay: 10000 }
    action: report-adverse-event
    delivery: ui-form
    data: { severity: "SERIOUS", description: "Headache grade 3" }
    steps:
      - navigate: "#trial/patients/patient-001"
      - click: "[data-action='report-ae']"
      - fill: { from: data }
      - click: "[data-action='submit']"
      - await: { event: "ae-reported" }
```

Same outcome as current `DemoDataSeeder` but portable, executable by pages,
and optionally visible in the UI.

### 6.2 Life household demo

```yaml
scenario: life-household-week
description: "A realistic week for a family of four"
data:
  actors: "data/demo-actors.json"
  transactions: "data/bank-6mo.json"
  calendar: "data/family-calendar.json"
  messages: "data/whatsapp-week.json"

steps:
  # Bulk seed — async, other steps can proceed
  - name: seed-actors
    action: load-actors
    delivery: rest
    endpoint: POST /external-actors
    data: { source: actors, mode: bulk }

  - name: seed-transactions
    action: load-transactions
    delivery: simulated
    target: bank
    data: { source: transactions, mode: bulk }

  - name: seed-calendar
    action: load-calendar
    delivery: simulated
    target: calendar
    data: { source: calendar, mode: bulk }

  # Stepped demo — visible in UI
  - name: plumber-confirms
    trigger: { after: seed-actors, delay: 5000 }
    action: whatsapp-message
    delivery: simulated
    target: chat
    data:
      from: "Bob's Plumbing"
      text: "Thursday 2pm confirmed for the boiler service"

  - name: system-creates-commitment
    trigger: { after: plumber-confirms }
    await: { event: "commitment-created" }

  - name: approve-invoice
    trigger: { after: system-creates-commitment, delay: 8000 }
    action: approve-oversight-gate
    delivery: ui-form
    data: { decision: "APPROVED", amount: 450 }
    steps:
      - navigate: "#inbox"
      - click: "[data-urgency='OVERDUE']:first-child"
      - click: "[data-action='approve']"
      - fill: { from: data }
      - click: "[data-action='confirm']"
      - await: { event: "oversight-gate-resolved" }

  # Stream — sensor readings every 10s
  - name: home-sensors
    trigger: { at: 0 }
    action: sensor-readings
    delivery: simulated
    target: iot
    data:
      source: "data/sensor-readings.json"
      mode: stream
      interval: 10000
```

### 6.3 IoT smart home demo

Scenario file for casehub-iot. `DeviceProvider` demo impl serves device
state from scenario data. Sensor readings stream continuously.

## 7. Repos Affected

| Repo | What changes |
|------|-------------|
| **parent** | Platform protocol document — scenario format spec, demo SPI convention |
| **pages** | Extend #142 scope — backend executor, REST/simulated delivery, cross-service coordination. Revise "Non-Goal: server-side simulation" |
| **connectors** | Demo `@Alternative` for ChatPlatform, CalendarPlatform. New BankFeedPlatform + EmailPlatform SPIs with demo impls. Scenario injection endpoint pattern |
| **engine** | Scenario injection for case lifecycle events (optional — cases can be started via REST) |
| **work** | Scenario injection for WorkItem lifecycle events (optional — tasks can be created via REST) |
| **platform** | Profile convention doc. Possibly shared scenario loading utility |
| **life** | Scenario files for household demo. Consumer of all connector demo impls |
| **clinical** | Migrate DemoDataSeeder to scenario file format |
| **iot** | DeviceProvider demo impl. Scenario files for smart home demo |
| **aml** | BankFeedPlatform consumer. Scenario files for investigation demo |
| **blocks-ui** | Scenario-aware components — `<scenario-controls>`, `<dataset-explorer>` from pages may move here if reused across apps |

## 8. Implementation Phases

| Phase | What | Depends on |
|-------|------|-----------|
| 1 | Scenario format spec (parent) + pages executor backend | — |
| 2 | Connector demo SPI pattern + ChatPlatform demo impl | Phase 1 |
| 3 | Life household scenario + conversational intake | Phase 2 |
| 4 | Clinical migration | Phase 1, Phase 2 |
| 5 | IoT demo + BankFeed SPI + Email SPI | Phase 2 |
| 6 | AML scenario | Phase 5 |

## 9. Open Questions

1. **Scenario file format:** JSON or YAML? YAML is more readable for
   scenario authors; JSON is more portable across runtimes. The pages
   DataSource spec uses TypeScript constructors, not data files — format
   alignment needed.

2. **Authentication in demo mode:** Pages executor makes REST calls to
   target apps. Demo profile disables OIDC, but the executor still needs
   identity context (`CurrentPrincipal`). Shared demo token? Per-step
   actor identity?

3. **Scenario discovery:** How does the executor find available scenarios?
   Classpath scan? Config property pointing to a directory? UI picker?

4. **Multi-app coordination:** The executor calls multiple apps. Startup
   ordering? Health checks before scenario begins?

5. **Recording:** Pages spec includes `recording(innerSource)` that
   captures timestamped events from a live source. Should the backend
   support recording live REST responses as scenario data files? This
   would let developers record a real session and replay it as a demo.
