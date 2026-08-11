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

3. **One format, two runtimes.** A single YAML scenario format parsed by
   the pages backend executor (Java) and executed across both the backend
   (REST calls, event injection) and the frontend TypeScript engine (UI
   actions, data simulation). Connector demo impls consume the `data`
   section only — they load external data files, not the full format.

4. **Concurrent by default.** Steps are a directed graph connected by
   triggers, not a sequential list. Bulk loads run asynchronously while
   other steps continue. Synchronisation is explicit via `AfterTrigger`.

### 2.1 Relationship to pages #140/#142

The pages DataSource spec (#140) defines **client-side data simulation** —
mutation rules, tick-driven evolution, `ScenarioController` managing a
virtual-time priority queue in the browser. Its non-goal "Server-side
simulation engine (all simulation runs client-side)" refers to data
evolution: `simulated()` sources mutating datasets via TypeScript mutation
rules remain a client-side concern.

This spec defines **cross-platform scenario orchestration** — an executor
that coordinates scripted demos across multiple CaseHub services via HTTP.
The executor makes REST calls, injects simulated events, and dispatches
UI automation. This is orchestration, not simulation. The two layers are
complementary:

| Layer | What it does | Where it runs | Spec |
|-------|-------------|---------------|------|
| Data simulation | Mutation rules evolve datasets over virtual time | Client (browser) | #140 |
| Scenario orchestration | REST calls, event injection, UI automation across services | Server (Pages backend) + Client (UI actions) | This spec |

The #140 non-goal should be annotated to clarify: "Server-side simulation
engine" excludes data evolution in the browser. Cross-service orchestration
is a different architectural layer and is covered by this spec.

**Authoring models:** Two authoring models coexist:
- **TypeScript constructors** (`simulated()`, `scenario()`, `replay()`) —
  type-safe, IDE-autocompletion, pages-only scenarios. Defined in #140.
- **YAML scenario files** — portable, cross-platform, parsed by the
  pages backend executor. Connector demo impls consume only the `data`
  section (external data files for Pull-mode queries). Defined here.

The pages YAML parser converts scenario files into `ScenarioConfig` objects
that the existing TypeScript engine executes. Vocabulary mapping:

| YAML | TypeScript type | Notes |
|------|----------------|-------|
| `navigate: "#path"` | `{ type: 'navigate', page: '#path' }` | Direct mapping |
| `click: "[selector]"` | `{ type: 'click', target: '[selector]' }` | Direct mapping |
| `fill: { field: value }` | Multiple `{ type: 'type', target, value }` | Compound — one UIAction per field, targets resolved via `data-field` attributes |
| `await: { event: "..." }` | Synchronisation primitive | Not a UIAction — triggers step completion when condition met |
| `select: { target, value }` | `{ type: 'select', target, value }` | Direct mapping |
| `hover: "[selector]"` | `{ type: 'hover', target: '[selector]' }` | Direct mapping |
| `scroll: { target, to }` | `{ type: 'scroll', target, to }` | Direct mapping |

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
    ui-actions:
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
| `rest` | HTTP call to target app's **normal API** (e.g. `POST /life-tasks`) | Invisible | Fast seeding, background state |
| `ui-form` | Navigate, fill fields, click submit via UI automation | Fully visible | Demo audience watches data entry |
| `simulated` | HTTP call to target app's **injection endpoint** (`POST /scenario/inject/{connector}`), which fires CDI events as if the external system sent them | Invisible to app code, visible in UI via SSE | External events (messages, transactions, sensors) |

The distinction between `rest` and `simulated` is semantic, not mechanical.
Both use HTTP POST. `rest` calls the app's real API — the app processes it
as a normal request. `simulated` calls the injection endpoint — the app
processes the resulting CDI event as if it came from an external system
(WhatsApp, bank feed, etc.). Application code cannot distinguish injected
events from real ones.

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

**`source` resolution:** The `source` field supports two forms:

1. **Literal path** — a quoted string containing `/`:
   `source: "data/bank-transactions-6mo.json"`. Resolved relative to the
   scenario file.
2. **Key lookup** — a bare word matching a key in the top-level `data:`
   map: `source: actors` resolves to the path declared under `data:`.

Resolution order: if the value matches a key in the top-level `data:` map,
use the mapped path. Otherwise, treat it as a literal file path. Keys must
not contain `/` — any value containing `/` is always a literal path.

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
ui-actions:
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

### 3.7 Schema rules

| Field | Required when | Default | Valid values |
|-------|-------------|---------|-------------|
| `name` | Always | — | Unique string within scenario |
| `action` | All steps except pure-await steps | — | Freeform string describing the action |
| `delivery` | All steps with `action` | `rest` | `rest`, `ui-form`, `simulated` |
| `endpoint` | `delivery: rest` | — | HTTP method + path (e.g. `POST /life-tasks`) |
| `target` | `delivery: simulated` | — | Connector name (e.g. `chat`, `bank`, `calendar`) |
| `trigger` | Optional | Fires at T=0 | `TimeTrigger`, `AfterTrigger`, or `DataTrigger` |
| `data` | Optional | — | Inline object or `{ source: "path-or-key", mode: "bulk" }` |
| `ui-actions` | `delivery: ui-form` | — | Array of UI action primitives |
| `await` | Optional | — | `{ event }`, `{ endpoint, match }`, or `{ delay }` |
| `actor` | Optional | `demo-admin` | Actor identity for `X-Scenario-Actor` header |
| `fast-fallback` | Optional | — | Alternative delivery for `fast` speed mode |

`data` cannot have both `source` and inline properties simultaneously —
`source` references an external file; inline properties are the payload.
The `mode` field (`bulk`, `stepped`, `stream`) is only valid with `source`.

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
| `CalendarPlatform` | connectors/calendar-spi (exists — `RefCalendarPlatform`, `GoogleCalendarPlatform`) | Event queries | Event created/updated |
| `BankFeedPlatform` | connectors (new SPI + module required) | Transaction queries | Transaction notifications |
| `EmailPlatform` | connectors (new SPI + module required) | Inbox queries | Inbound emails |
| `DeviceProvider` | iot (new) | Device state, sensor readings | Sensor events, state changes |

**Dependency boundary:** Application repos (life, clinical, iot) add
connector SPI modules as Maven dependencies — e.g. `casehub-connectors-chat-spi`.
This follows the existing pattern: life already depends on connector core
modules transitively via qhorus. The SPI modules are pure interfaces with
no application-specific code, so this does not violate the boundary rule
that application repos don't depend on each other.

### 4.3 Scenario data lifecycle

1. **Startup:** Demo impl loads the scenario's external data files (bulk
   pull datasets). Available immediately for query.
2. **Runtime:** Scenario executor injects push events via injection
   endpoint at times defined in the scenario file.
3. **Teardown:** H2 in-memory database. No cleanup needed — restart clears
   everything.

### 4.4 Scenario discovery and Pull-mode loading

**How demo impls know which scenario is active:**

Each target service reads a config property:
```properties
casehub.scenario.active=life-household-demo
casehub.scenario.path=scenarios/
```

At startup, the demo impl scans `casehub.scenario.path` for a scenario file
matching `casehub.scenario.active`, loads its `data` section, and pre-loads
the referenced external data files for Pull-mode queries.

**Startup ordering:** The executor (pages) checks target service health via
`GET /q/health` before starting scenario playback. Services must be running
and healthy (Pull-mode data loaded) before the first step fires. The
executor retries with exponential backoff (max 30s) and fails with a
diagnostic if any target service is unreachable.

## 5. Pages as Universal Executor

Pages' Quarkus backend becomes the scenario coordinator:

1. **Load** — parse scenario YAML, build trigger graph
2. **Schedule** — register triggers with the backend `ScenarioExecutor`
3. **Execute per delivery mode:**
   - `rest` → HTTP call to target app API
   - `simulated` → HTTP call to target app's `/scenario/inject/{connector}` endpoint
   - `ui-form` → dispatch UIActions to pages frontend via `ControlChannel`
4. **Coordinate** — `ScenarioExecutor` (backend) is authoritative for
   scenario state; frontend `ScenarioController` (TypeScript, #140)
   synchronises via `ControlChannel`
5. **Observe** — `<scenario-controls>` and `<dataset-explorer>` from pages
   #142 Phase 7 provide the demo operator UI

### 5.0.1 Backend/frontend controller architecture

Two controllers exist, each owning its domain:

| Controller | Runtime | Owns | Defined in |
|-----------|---------|------|-----------|
| `ScenarioExecutor` | Java (Pages backend) | Trigger graph, step scheduling, REST/injection delivery, scenario state (play/pause/speed/elapsed) | This spec |
| `ScenarioController` | TypeScript (browser) | Virtual-time priority queue, data simulation ticks, UIAction dispatch | #140 |

**Synchronisation via `ControlChannel`** (from #142 §4):

The backend `ScenarioExecutor` acts as `ScenarioHost`. The frontend
`ScenarioController` acts as `ScenarioRemote`. They communicate via the
existing `ControlChannel` abstraction (WebSocket in cross-machine mode,
direct call in same-process mode):

1. **State → frontend:** Backend sends `{ type: 'state', state }` on every
   state change (play/pause/speed). Frontend's `ScenarioController` applies
   the state — `setSpeed()`, `play()`/`pause()`.
2. **UI-form dispatch:** Backend sends `{ type: 'ui-action', stepName, actions }`.
   Frontend executes the `UIAction` sequence and sends
   `{ type: 'step-complete', stepName }` on completion (or
   `{ type: 'step-failed', stepName, error }` on failure).
3. **Commands → backend:** `<scenario-controls>` sends play/pause/step/speed
   commands to the backend executor (not directly to the frontend controller).
   The backend applies the command, then broadcasts updated state.

The backend is always authoritative. The frontend never changes speed or
pause state independently — it reflects what the backend tells it.

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
delivery for speed — same action, invisible execution.

**Degradation configuration:** Per-step via `fast-fallback`:
```yaml
- name: create-task-fast
  action: create-task
  delivery: ui-form
  fast-fallback:
    delivery: rest
    endpoint: POST /life-tasks
  data: { title: "Chase Bob" }
  ui-actions:
    - navigate: "#home"
    - click: "[data-action='new-task']"
    - fill: { from: data }
    - click: "[data-action='submit']"
```

When the executor runs in `fast` mode, steps with `fast-fallback` use the
fallback delivery instead of `ui-form`. Steps without `fast-fallback` are
skipped in fast mode (no REST equivalent exists). The `fast-fallback` block
requires its own `delivery` and `endpoint` — the executor does not infer
endpoints from action names.

### 5.3 Await and verification

Every step can include an `await` that confirms the backend processed the
action:

```yaml
- await: { event: "work-item-created" }           # SSE event
- await: { endpoint: "GET /life-tasks", match: { title: "Chase Bob" } }  # poll
- await: { delay: 2000 }                           # simple wait
```

### 5.4 Verification mode

Verification mode is activated by a runtime flag:
```properties
casehub.scenario.mode=verify    # default: demo
```

In `verify` mode:
- Every `await` becomes an assertion with a configurable timeout
- `{ delay: N }` awaits are skipped (pure timing has no assertion value)
- All delivery modes execute normally — `ui-form` steps are NOT skipped.
  Speed defaults to `normal` (1x). The operator can set speed to `fast`
  explicitly (which applies `fast-fallback` rules from §5.2), but
  verification does not force it. This ensures `ui-form` steps like
  oversight gate approvals are verified through the real UI flow.
- Failure produces a JUnit XML report at `target/scenario-results.xml`
  for CI integration, plus a human-readable summary on stderr

**Timeout rules:**
| Await type | Default timeout | Override |
|-----------|----------------|---------|
| `{ event: "..." }` | 10s | `timeout` field: `{ event: "...", timeout: 30000 }` |
| `{ endpoint: "GET ...", match: {...} }` | 10s (polled every 500ms) | `timeout` field |
| `{ delay: N }` | Skipped in verify mode | — |

**Failure output:**
```
FAIL: step "create-task-visible" — await { event: "work-item-created" }
      timed out after 10000ms. No matching SSE event received.
      Last SSE events: [commitment-created, oversight-gate-resolved]
```

Non-zero exit code on any assertion failure.

### 5.5 Error model

Step execution errors are handled per delivery mode:

| Failure | Behaviour |
|---------|-----------|
| `rest` delivery gets 4xx/5xx | Step fails. Error logged with status code and response body. Scenario continues — dependent steps (via `AfterTrigger`) are skipped with a diagnostic. |
| `simulated` injection endpoint unreachable | Step fails. Scenario pauses with diagnostic: "Target service {host}:{port} unreachable for connector {name}". Operator can fix and resume. |
| `await` with `event` never fires | Timeout (10s default). Step fails. Dependent steps skipped. |
| `ui-form` selector not found | Step fails after 5s selector wait. Error: "Selector `[data-action='submit']` not found in DOM". |
| Bulk data load partial failure | Bulk steps report progress. Partial failure logs failed items and continues. Step completes with warning. |

**Failure propagation:** A failed step does not abort the scenario by
default. Steps with no dependency on the failed step continue executing.
Steps chained via `AfterTrigger` to the failed step are skipped with a
diagnostic. In `verify` mode, any step failure is recorded as an
assertion failure in the JUnit report.

## 6. Migration Path

### 6.1 Clinical DemoDataSeeder (first migration)

Clinical would benefit from a scenario seeder that replays full trial
scenarios through real service calls. The scenario file format enables this:

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
    ui-actions:
      - navigate: "#trial/patients/patient-001"
      - click: "[data-action='report-ae']"
      - fill: { from: data }
      - click: "[data-action='submit']"
      - await: { event: "ae-reported" }
```

Portable, executable by pages, and optionally visible in the UI.

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
    ui-actions:
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

`DeviceProvider` demo impl serves device state from scenario data. Sensor
readings stream continuously via the `stream` data mode.

```yaml
scenario: iot-smart-home-demo
description: "Smart home sensors, thermostat control, security events"
data:
  devices: "data/home-devices.json"
  sensor-history: "data/sensor-24h.json"

steps:
  - name: seed-devices
    action: load-devices
    delivery: simulated
    target: iot
    data: { source: devices, mode: bulk }

  - name: temperature-stream
    trigger: { after: seed-devices }
    action: sensor-reading
    delivery: simulated
    target: iot
    data:
      source: "data/temperature-readings.json"
      mode: stream
      interval: 10000

  - name: motion-detected
    trigger: { after: seed-devices, delay: 15000 }
    action: security-event
    delivery: simulated
    target: iot
    data:
      deviceId: "motion-sensor-01"
      event: "MOTION_DETECTED"
      zone: "front-door"

  - name: thermostat-adjust
    trigger: { after: motion-detected, delay: 5000 }
    action: set-thermostat
    delivery: rest
    endpoint: POST /devices/thermostat-01/commands
    data: { command: "SET_TEMPERATURE", value: 22 }
    await: { event: "device-command-acknowledged" }
```

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
| **clinical** | Scenario files for trial demo |
| **iot** | DeviceProvider demo impl. Scenario files for smart home demo |
| **aml** | BankFeedPlatform consumer. Scenario files for investigation demo |
| **blocks-ui** | No scenario-specific changes. `<scenario-controls>` and `<dataset-explorer>` remain in pages — they are part of the executor, not reusable block components |

## 8. Implementation Phases

| Phase | What | Depends on |
|-------|------|-----------|
| 1 | Scenario format spec (parent) + pages executor backend | — |
| 2 | Connector demo SPI pattern + ChatPlatform demo impl | Phase 1 |
| 3 | Life household scenario + conversational intake | Phase 2 |
| 4 | Clinical migration | Phase 1, Phase 2 |
| 5 | IoT demo + BankFeed SPI + Email SPI | Phase 2 |
| 6 | AML scenario | Phase 5 |

## 9. Settled Decisions

SETTLED: YAML as the cross-platform scenario format (from Open Question 1).
YAML is more readable for scenario authors and supports comments. The pages
YAML parser converts files to `ScenarioConfig` objects (see §2.1). TypeScript
constructors remain the developer-friendly authoring model for pages-only
scenarios; the two are complementary.

SETTLED: Demo profile authentication with per-step actor identity (from Open Question 2).
All target services run in `demo` build profile (`@IfBuildProfile("demo")`).
Demo profile disables OIDC. The executor sends a `X-Scenario-Actor` header
on each REST call. Each target service's `DemoCurrentPrincipal` (e.g.
`io.casehub.life.app.demo.DemoCurrentPrincipal`, activated by
`@IfBuildProfile("demo")`) reads the actor identity from this header,
falling back to `demo-admin` if absent. Scenario steps can specify an actor:
```yaml
- name: approve-invoice
  actor: household-admin
  action: approve-oversight-gate
  delivery: ui-form
```
The executor passes `actor` as the `X-Scenario-Actor` header value.

SETTLED: Scenario discovery via config property (from Open Question 4).
See §4.4. The executor reads `casehub.scenario.path` and presents available
scenarios via a UI picker in `<scenario-controls>`. Health checks run before
playback starts.

## 10. Open Questions

1. **Recording:** Pages spec includes `recording(innerSource)` that
   captures timestamped events from a live source. Should the backend
   support recording live REST responses as scenario data files? This
   would let developers record a real session and replay it as a demo.
