# Avatar Integration — Neocortex Cognitive Agent into life-ui

> Design spec for life#143 (P1.4 in epic #139)
> Revised after decision review (15 findings) and spec review (16 findings)

## Overview

Wire the neocortex CognitionCore as the primary conversational surface in
casehub-life. The avatar is the central intelligence (D2 from the roadmap) —
it understands intent, knows the user's life state, uses MCP tools to act,
and develops personality through cognitive subsystems.

### Phased delivery

CognitionCore (`casehub-neocortex-cognition`) is NOT currently published
to Maven as a consumable artifact for casehub-life. The neocortex cognitive
architecture roadmap Phase 1 extraction is "Batches 1–6f done, Batch 7 ready
for execution" — the API surface is settling but not yet available as a
dependency.

This spec therefore has two implementation phases:

**Phase A — Conversational surface without CognitionCore:**
Build the full conversation infrastructure: REST endpoint, SSE streaming,
domain context injection, MCP tool access, eidos-based identity, conversation
panel. The avatar is functional and tool-enabled but has static personality.

**Phase B — CognitionCore integration (when artifacts publish):**
Add CognitionCore wiring: mood, drives, BDI extraction, narrative, user model,
goal proposals. The avatar gains dynamic personality. Phase B is additive —
Phase A's architecture is designed so CognitionCore plugs in without redesign.

The separation is clean: LifeCognitiveAgentService uses CDI optional
injection (`Instance<CognitionCore>`). Phase A: CognitionCore is absent —
the service operates without it. Phase B: CognitionCore is on the
classpath and injected — cognitive processing activates automatically.

## Architecture

### Server-side components

**1. LifeCognitiveAgentService** (`app/service/`)
- `@ApplicationScoped`, manages conversation sessions via a single
  application-level `SessionPipeline` (`@ApplicationScoped`)
- Optional CognitionCore (CDI `Instance<CognitionCore>` — Phase A
  works without it)
- Sessions keyed by `tenantId:principalId`, one per user
- SessionPipeline provides idle eviction (PT30M), AIMD admission
  (ceiling 5), and SIZE_AWARE clearing policy — no custom session
  management code needed
- Per-session in-flight guard: `LifeCognitiveAgentService` maintains
  an `AtomicBoolean` per session. If a second POST arrives while a
  query is streaming, return `409 Conflict`. AIMD admission is global
  (prevents overwhelming the LLM backend), but does not provide
  per-session mutual exclusion.

Key methods:
- `sendMessage(principalId, userMessage)` → `Multi<AvatarEvent>` (streaming)
- `closeSession(principalId)`

**AgentProvider implementation:** The existing LifeAgentProviderFactory
(OpenClaw wrapper) does NOT support openSession(). This issue requires
wiring a real `AgentBackend` implementation (casehub-platform-agent-openai
for Claude/Anthropic API) via `SessionPoolRegistry` and
`BackendInstanceRegistry`. Wiring:
- `BackendInstanceRegistry`: register Anthropic backend instance
  (key "life-avatar", model from config)
- `SessionPoolRegistry`: register pool for "life-avatar" backend
- `SessionLifecycleManager`: wraps SessionPoolRegistry
- `SessionPipeline`: wraps SessionLifecycleManager. Constructor args:
  backendKey "life-avatar", ceiling 5, clearAtTokens 50000, idle
  timeout PT30M. Single `@ApplicationScoped` instance.

Engine workers continue to use the existing OpenClaw invoke() path via
`LifeAgentProviderFactory`. The avatar's `SessionPipeline` is entirely
separate — it does not modify `LifeAgentProviderFactory`. The factory's
`openSession()` UnsupportedOperationException remains correct for engine
workers; the avatar bypasses it entirely via direct backend wiring.

On each message:
1. Assemble system prompt: eidos descriptor + (CognitionCore.promptSections()
   if Phase B) + domain context summary
2. Send to AgentSession.query() — session has MCP tools configured
3. Stream response as AvatarEvent (text deltas, tool calls, structured cards)
4. If Phase B: call CognitionCore.recordInteraction() (mood appraisal, BDI
   extraction, user model update, strategy learning)

**2. LifeDomainContextProvider** (`app/service/`)
- `@ApplicationScoped`, assembles life state summary for system prompt
- Gathers from: DashboardService (briefing — overdue + due-today),
  PendingActionsService (top-N by urgency), ExternalActorService
  (actor count + any with low trust), LifeCaseService (active case count
  by type), unresolved calendar conflict count
- Renders as compact structured text (counts + flagged items, NOT full
  entity dumps) to stay within prompt token budget
- Token budget guard: summary capped at ~2000 tokens. If a user has
  many active items, summarise to counts + top-3 urgent items. The avatar
  uses MCP tools to query detail when needed.
- Invalidation: significant domain events (SLA_BREACH, CASE_COMPLETED,
  CALENDAR_CONFLICT_DETECTED) set a dirty flag. Next user message triggers
  rebuild before sending to LLM.

**3. LifeAvatarDescriptorProvider** (`app/engine/agent/`)
- Provides the base AgentDescriptor (eidos) for the life avatar
- Coexists with existing LifeAgentDescriptorFactory — the avatar is NOT
  a LifeAgent enum value (it's conversational, not a case-execution worker)
- Identity: name, personality traits, constraints, goals
- Uses `AgentDescriptor.disposition()` (not DispositionProfile — that
  type doesn't exist in eidos)
- Config-driven: `casehub.life.avatar.*` properties

**4. LifeAvatarResource** (`app/resource/`)
- REST endpoint for the conversation surface
- `POST /avatar/message` — send message, returns SSE stream of response events
- `POST /avatar/confirm` — approve/reject pending confirmation, returns
  SSE stream of result events. Body: `{"actionId": "...", "approved": true}`
- `DELETE /avatar/session` — close current session
- `@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER,
  HouseholdGroups.JUNIOR})` — all household roles can converse

**Transport: POST + SSE (not WebSocket)**
The user sends messages via POST. Response tokens stream back via SSE on
the POST response itself (Quarkus `@Produces(SERVER_SENT_EVENTS)` with
`Multi<SseEvent<AvatarEvent>>`). This matches the existing
LifeEventSseResource pattern, requires no new dependencies, and naturally
supports one-directional streaming (which is the primary flow: user sends
text, avatar streams response).

WebSocket is NOT used. No `quarkus-websockets-next` dependency.
EventConnection from pages-data is NOT used (it's a topic pub/sub
system, not suitable for conversational request/response).

**5. AvatarEvent** (`api/`)
- Sealed interface mapping from platform AgentEvent:
  - `TextDelta(text)` ← AgentEvent.TextDelta
  - `ThinkingDelta(text)` ← AgentEvent.ThinkingDelta (surfaced if
    extended thinking is enabled — UX decision: show in collapsed
    "thinking..." indicator, not full text)
  - `ToolCallStarted(toolName)` ← first AgentEvent.ToolCallDelta for
    a given index (signals tool invocation began)
  - `ToolCallCompleted(toolName, arguments)` ← AgentEvent.ToolCallComplete
  - `ToolResult(toolName, result, isError)` ← AgentEvent.ToolResult
    (surfaced — errors must be visible to user)
  - `ActionCard(type, data)` — life-specific: structured response for
    task created, case started, conflict resolved, commitment made.
    Derived from ToolResult content.
  - `ConfirmationRequired(actionId, description, consequences)` — emitted
    by the avatar before consequential actions
  - `SessionComplete(usage)` ← AgentEvent.InvocationComplete

### Frontend components

**6. life-conversation-panel** (`life-ui/src/panels/`)
- Lit 3.x component replacing `life-comms-mock`
- Two modes:
  - **Collapsed** (280px, default): recent messages summary, input field,
    notification count
  - **Expanded** (~480px): full message history, typing indicator,
    structured response cards, action confirmations. Positioned as
    absolute overlay on centre content (does not resize dock grid).
- Transport: `fetch()` POST to `/avatar/message`, response body is SSE
  stream. Parsed via `fetch()` + `response.body.getReader()` +
  `TextDecoderStream` — manual `text/event-stream` line parsing. This
  is NOT `SSEManager` (which wraps browser `EventSource`, GET-only).
  A new `FetchSseReader` utility in `life-ui/src/utils/` handles the
  POST response stream: splits on `\n\n`, parses `event:` / `data:`
  fields, dispatches typed `AvatarEvent` objects to the panel.
- Message rendering: user bubbles (right), avatar bubbles (left), action
  cards (full-width), confirmation dialogs (modal)
- Markdown rendering via `marked` + `dompurify` (already in deps)
- Streaming: tokens arrive via SSE, append to current avatar bubble
  with typing animation
- Thinking indicator: when ThinkingDelta arrives, show animated
  "thinking..." pill (content not displayed)

**7. Integration with home-view**
- Replace `life-comms-mock` registration in right dock with
  `life-conversation-panel`
- Expand/collapse toggle: dock bar icon switches panel mode
- Notification badge: unread avatar messages via LifeEventController

### CognitionCore wiring (Phase B)

**8. CognitionCore configuration**
- Conditional: only wired when `casehub-neocortex-cognition` is on the
  classpath. LifeCognitiveAgentService accepts @Inject Instance<CognitionCore>
  (CDI optional injection).
- All subsystems enabled (CognitionConfig.all())
- The main conversation uses the application-level SessionPipeline
  (from §1). CognitionCore's own internal LLM calls (mood appraisal,
  BDI extraction) use `AgentProvider.invoke()` directly — these are
  single-shot calls, separate from the multi-turn conversation session.
- MoodOrchestrator: PAD model with life-calibrated defaults
- DriveOrchestrator: life-specific drives (household efficiency, family
  wellbeing, financial prudence, health vigilance)
- UserModelOrchestrator: builds model of the household member over time
- MentalModelOrchestrator: BDI extraction from conversations
- NarrativeOrchestrator: tracks ongoing life narratives
- GoalProposalOrchestrator: proposes goals from patterns (proactive)
- MemoryHygieneOrchestrator: manages memory store pruning

**9. Cognitive tick model (Phase B)**
- CognitionCore.tick() called after each conversation turn (not on a timer)
- The tick processes: mood decay, drive updates, attention shifts, narrative
  evolution, goal priority adjustments
- CognitionCore.recordInteraction() called with user message + agent response
  after each turn — drives mood appraisal and BDI extraction

### MCP tool access

**10. Tool registration and safety classification**
- AgentSessionInit.mcpServers configured at session-open time
- Points at the platform's MCP server endpoint (HTTP MCP transport)
- All 6 @McpDomain services available: life/tasks, life/actors, life/cases,
  life/dashboard, life/analytics, life/events
- Playbook execution: via life/cases domain (start case with template)
- Tool descriptions enriched with domain context for the LLM

**Tool safety classification:**
The avatar introduces prompt injection as a new attack surface — an
LLM decides which tools to call based on user text, unlike the current
UI where users click specific endpoints.

MCP tools are classified by risk level:

| Classification | Tools | Policy |
|---|---|---|
| Read-only | life/dashboard, life/analytics, GET on life/tasks, life/actors, life/cases, life/events | Auto-allowed |
| Low-risk mutation | POST life/tasks (create task) | Auto-allowed |
| High-risk mutation | POST life/cases (start case), POST life/tasks/{id}/commit (commitment), POST life/dashboard/oversight-gates | Confirmation required |
| Restricted | POST life/actors/{id}/personal-data/erase (GDPR), POST life/actors/{id}/delete | Blocked for junior; confirmation required for admin/member |

**Phase A enforcement: system prompt (conversational confirmation).**
The platform `AgentSession` API has no tool execution callback — the
backend internally manages the model→tool→model loop (`query()` returns
`Multi<AgentEvent>` as observation-only events, and `AgentSessionInit`
has no callback field). There is no hook to intercept between "model
wants a tool" and "tool is executed."

Phase A therefore uses system prompt enforcement: the tool safety
classification table above is encoded as system prompt rules. The LLM
manages the confirmation flow conversationally — it asks "Should I
start the boiler repair case?" before calling the tool, and the user
responds naturally. This is actually more coherent than out-of-band
confirmation because:
1. The LLM maintains full conversation history including tool
   call and result
2. The confirmation is a natural conversational exchange
3. No history gap between tool call and tool result (cf. §11)

**Phase B+ aspiration: server-side tool gating.**
True server-side enforcement requires a platform API extension: a tool
execution callback in `AgentSessionInit` (e.g.,
`Function<ToolCallComplete, ToolGatingDecision>`) and backend support
for pausing before MCP execution. This is tracked as a platform
dependency. When available, the service can intercept tool calls before
execution, enforcing the classification table in code rather than in
the system prompt.

Until server-side gating exists, the system prompt approach is the
practical reality. The classification table ensures the system prompt
rules are comprehensive and consistent, even though enforcement is
advisory rather than programmatic.

**Security note:** @McpDomain-generated REST endpoints currently lack
@RolesAllowed (LifeRestSecurityTest is disabled). MCP tool access in
this issue is gated by the avatar session's own authentication —
only authenticated household members can converse with the avatar.
Full MCP auth wiring is tracked separately.

**11. Confirmation flow**

The avatar's confirmation flow is SEPARATE from the engine's
ActionRiskClassifier (which gates worker actions during case execution).

**Phase A: conversational confirmation (system prompt).**
The LLM manages the confirmation flow as part of the conversation. For
high-risk or restricted tools (per §10 classification), the system
prompt instructs the avatar to ask "Should I proceed?" before calling
the tool. The user responds naturally ("yes", "go ahead", "no").
The LLM then calls or skips the tool — with the full exchange in
conversation history.

This is the cleanest UX: the confirmation is a natural conversational
exchange, not a modal dialog. The tool call and result stay in the
LLM's history, so the avatar knows what it did.

**Phase B+ protocol: out-of-band confirmation (when server-side tool
gating exists).**
When the platform API supports tool execution callbacks (see §10):

1. Service intercepts high-risk tool call → emits
   `ConfirmationRequired(actionId, description, consequences)` in the
   SSE stream, followed by `SessionComplete`
2. Server stores pending confirmation in `LifeCognitiveAgentService`:
   `ConcurrentHashMap<String, PendingConfirmation>` keyed by actionId
   (UUID). Each entry holds: toolName, toolArguments, principalId,
   createdAt.
3. User approves/rejects → `POST /avatar/confirm` with body
   `{"actionId": "...", "approved": true/false}`. Returns SSE stream.
4. If approved: service executes the stored MCP tool call, streams
   result events (ToolResult, ActionCard, SessionComplete). The tool
   result must also be injected into the LLM's session history (via
   synthetic tool_result prepended to the next `query()` call) so the
   LLM knows the action was taken.
5. If rejected: service removes pending confirmation, streams
   acknowledgment text + SessionComplete.

**Conversation history coherence (Phase B+):** when the out-of-band
protocol executes a tool call outside the LLM's session, the result is
NOT in the LLM's conversation history. Without injection, the LLM may
not know the action was taken and could attempt to call the tool again.
Mitigation: prepend the confirmed tool result as a synthetic context
block in the next `query()` call (same pattern as `<domain_context>`).

**Endpoint:** `POST /avatar/confirm` — same `@RolesAllowed` as
`/avatar/message`. Returns `Multi<SseEvent<AvatarEvent>>`. Defined now
but used primarily in Phase B+. In Phase A, confirmations happen
conversationally and this endpoint is not needed.

**Timeout:** pending confirmations expire after 5 minutes
(configurable via `casehub.life.avatar.confirmation.timeout`). Expired
entries are removed lazily on next access or by SessionPipeline's
eviction sweep.

**Interleaving:** if the user sends a regular `/avatar/message` while
a confirmation is pending, the message is processed normally. The
pending confirmation remains valid until timeout. The avatar may
reference it in its response.

**Junior users:** cannot confirm consequential actions. In Phase A, the
system prompt instructs the avatar to refuse execution for junior users.
In Phase B+, the `/avatar/confirm` endpoint rejects with 403 for
`household-junior` on high-risk and restricted operations.

### Cognitive state persistence (Phase B)

CognitionCore accumulates state in-memory during a session, but key
subsystems persist across sessions via neocortex stores:
- **MoodState** — persisted via `moodPersister` callback → CaseMemoryStore
- **UserModel** — persisted via UserModelOrchestrator → CaseMemoryStore
- **MentalModel** — BDI extractions persisted to CaseMemoryStore
- **Narrative** — narrative fragments persisted to MindMapStore
- **Engagement events** — persisted via `engagementPersister` callback

On session start, CognitionCore rehydrates from these stores. The avatar
does NOT lose personality on tab close — it loses only the in-flight
conversation history.

### Context refresh mid-session

Domain context changes (new SLA breach, case completion) need to reach the
avatar. `AgentSession.query(String prompt)` takes only the user's message;
there is no `updateSystemPrompt()` method. The system prompt is baked into
`AgentSessionInit` at session creation.

**Mechanism: context prepend to user message.**
When the dirty flag is set, `LifeCognitiveAgentService` prepends the
rebuilt domain context to the user's message before calling `query()`:

```
<domain_context>
Household state as of 2026-11-02T14:30:00Z:
- 3 overdue tasks (highest: boiler repair, 5 days overdue)
- SLA breach on task #42 — contractor response overdue
- 2 active cases: boiler-repair (IN_PROGRESS), school-enrolment (WAITING)
</domain_context>

What tasks are due today?
```

The system prompt (set at session creation) includes the instruction:
"When a `<domain_context>` block appears at the start of a user message,
treat it as updated household state — do not echo it or treat it as
user input."

This is the standard approach for mid-session context injection in
conversation systems. It works with any backend and requires no changes
to the `AgentSession` API.

**Trigger:** significant domain events (SLA_BREACH, CASE_COMPLETED,
CALENDAR_CONFLICT_DETECTED) received via SSE set a dirty flag. The next
user message triggers rebuild + prepend.

**Fallback: session close + reopen.** If the domain context has changed
so substantially that the system prompt itself needs restructuring (e.g.,
new capability domains added), close the session and reopen with a fresh
`AgentSessionInit`. This loses conversation history but is acceptable
since conversation history is transient in Phase A.

### Avatar audit trail

Avatar-initiated actions must be auditable. When the avatar executes
an MCP tool that creates, modifies, or deletes data:
- The action is attributed to the user's principal (they authorized it)
- A `delegatedVia: "avatar"` marker is included in the ledger entry's
  supplementary data, distinguishing avatar-initiated actions from direct
  user actions
- Tool call events (ToolCallComplete, ToolResult) are logged via the
  existing `LifeEventBroadcaster` SSE infrastructure for UI transparency

### Multi-user model

One avatar per tenant, shared across all household members. All three
roles (household-admin, household-member, household-junior) converse
with the same avatar instance. CognitionCore's UserModelOrchestrator
(Phase B) tracks per-user interaction patterns, and MentalModelOrchestrator
builds per-user BDI models. The avatar knows who it's talking to via
CurrentPrincipal.

Oversight gates: when the avatar proposes an action that requires
admin approval, the ConfirmationRequired event is emitted. If the
current user lacks the required role, the avatar explains that admin
approval is needed and does not proceed.

## Data model

No new JPA entities. No Flyway migrations. CognitionCore state (Phase B)
persists via neocortex memory stores (CaseMemoryStore, MindMapStore),
not via new life-specific tables.

## Dependencies

### New Maven dependencies

Phase A:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-eidos-api</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-api</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-session-core</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-openai</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-config-core</artifactId>
</dependency>
```

Phase B (when published — artifact names provisional, to be confirmed
when neocortex Phase 2 publishes):
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-cognition</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-cognition-api</artifactId>
</dependency>
```

### Cross-repo dependencies

- casehub-eidos: AgentDescriptor, AgentDisposition. No changes needed.
- casehub-platform: AgentProvider, AgentSession, AgentEvent,
  SessionPoolRegistry, BackendInstanceRegistry. No changes needed.
- casehub-neocortex (Phase B only): CognitionCore, CognitionCoreFactory,
  cognitive subsystem types. Blocked until Phase 1 extraction completes
  and artifacts are published.

## Configuration

```properties
# Avatar identity
casehub.life.avatar.name=Life
casehub.life.avatar.personality=helpful, proactive, attentive
casehub.life.avatar.model=claude-sonnet-4-5

# Agent backend
casehub.life.avatar.backend=anthropic
casehub.life.avatar.api-key=${ANTHROPIC_API_KEY}

# Session management
casehub.life.avatar.session.idle-timeout=PT30M
casehub.life.avatar.session.max-concurrent=5

# MCP server for tool access
casehub.life.avatar.mcp.server-url=http://localhost:8080/mcp
```

## Testing strategy

### Unit tests
- `LifeDomainContextProviderTest` — verify context assembly, token budget
  cap, dirty flag invalidation, `<domain_context>` block construction
- `AvatarEventTest` — verify complete mapping from AgentEvent (all 6 variants)
  to AvatarEvent (all 8 variants)
- `LifeAvatarDescriptorProviderTest` — verify descriptor configuration
- `ContextPrependTest` — dirty flag lifecycle: set on domain event,
  consumed on next message, prepend format matches spec, no prepend
  when flag is clean
- `PendingConfirmationTest` — storage, retrieval by actionId, timeout
  expiry (5min), lazy cleanup, junior role rejection

### Integration tests (@QuarkusTest)
- `LifeAvatarResourceTest` — REST endpoint:
  - Message send returns SSE stream
  - Session close
  - Role-based access (admin/member/junior all admitted)
  - `POST /avatar/confirm` — approval streams result, rejection
    streams acknowledgment, expired actionId returns 404, junior
    returns 403 on restricted operations
  - `409 Conflict` when in-flight guard blocks concurrent POST
  - Context prepend when dirty flag is set between messages
- `LifeCognitiveAgentServiceTest` — session lifecycle:
  - Multi-turn conversation with canned LLM responses (test backend)
  - Domain context injection on first message
  - Per-session AtomicBoolean guard lifecycle (acquire on query start,
    release on stream completion or error)
  - PendingConfirmation storage and expiry under SessionPipeline idle
    eviction

### Frontend tests
- `life-conversation-panel` — collapsed/expanded rendering, message
  display, streaming token append, confirmation dialog, thinking
  indicator
- `FetchSseReaderTest` — POST+SSE response parsing: `\n\n` splitting,
  `event:` / `data:` field parsing, multi-line data fields, stream
  error and abort handling, incomplete chunk buffering

## Scope boundaries

### In scope (this issue)
- Server-side cognitive agent service (Phase A: without CognitionCore)
- POST + SSE conversation endpoint
- Domain context injection (compact summary + tools for detail)
- MCP tool access via AgentSessionInit
- Conversation panel in life-ui (replacing comms-mock)
- Confirmation flow for consequential actions
- Session lifecycle management
- Eidos-based avatar identity

### Phase B (when neocortex artifacts publish)
- Full CognitionCore wiring (mood, drives, BDI, narrative, user model)
- Dynamic personality evolution
- Cognitive tick after each conversation turn

### Out of scope (subsequent issues)
- Intent classification and three-tier dispatch (#144)
- Avatar-driven UI transformation (#145)
- Voice intake (#120)
- Email/WhatsApp intake (#112, #113)
- Family mind map integration (#121)
- NLP cost optimisation (#137)
- Conversation history persistence across server restarts
- MCP auth wiring on @McpDomain endpoints

## References

- CognitionCore.java — neocortex/cognition/src/main/java/.../core/CognitionCore.java
- AgentProvider.java — platform/agent-api/src/main/java/.../agent/AgentProvider.java
- AgentSession.java — platform/agent-api/src/main/java/.../agent/AgentSession.java
- AgentSessionInit — platform/agent-api (carries mcpServers for tool access)
- SessionPoolRegistry — platform/agent-session-core (pool management)
- BackendInstanceRegistry — platform/agent-session-core (backend wiring)
- Neocortex cognitive architecture roadmap — neocortex/specs/issue-303/
- Epic #139 — digital life assistant roadmap (D1-D9 decisions)
- LifeEventSseResource — existing SSE endpoint pattern
- life-comms-mock.ts — current placeholder panel being replaced
- LifeEventController — existing SSE reactive controller pattern
- @McpDomain — platform annotation generating dual MCP/REST/GraphQL
- HouseholdGroups.java — role constants (ADMIN, MEMBER, JUNIOR)
- LifeAgentDescriptorFactory — existing agent descriptor pattern
- Decision review findings — 15 issues, key revisions: R1-06 (cognitive
  state persistence), R1-10 (context refresh), R1-13 (audit delegation)
- Spec review round 1 — 16 issues, key revisions: phased delivery,
  POST+SSE transport, @RolesAllowed fix, AvatarEvent mapping completeness,
  SessionPipeline vs direct pool management, domain context token budget
