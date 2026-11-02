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

The separation is clean: LifeCognitiveAgentService takes an optional
CognitionCore. Phase A passes null; Phase B passes the wired instance.
CognitionCore's constructor already accepts nullable subsystems.

## Architecture

### Server-side components

**1. LifeCognitiveAgentService** (`app/service/`)
- `@ApplicationScoped`, manages AgentSession lifecycle per user
- Optional CognitionCore (nullable — Phase A works without it)
- Uses `AgentProvider.openSession()` for multi-turn sessions
- Sessions keyed by `tenantId:principalId`, one per user

Key methods:
- `sendMessage(principalId, userMessage)` → `Multi<AvatarEvent>` (streaming)
- `closeSession(principalId)`

**AgentProvider implementation:** The existing LifeAgentProviderFactory
(OpenClaw wrapper) does NOT support openSession(). This issue requires
wiring a real `AgentBackend` implementation (casehub-platform-agent-openai
for Claude/Anthropic API) via `SessionPoolRegistry` and
`BackendInstanceRegistry`. Configuration:
- `SessionPoolRegistry`: one pool, keyed "life-avatar"
- `PoolDeclaration`: max concurrent sessions = 5, idle timeout = 30m
- `BackendInstanceRegistry`: register Anthropic backend with model config

Engine workers continue to use the existing OpenClaw invoke() path.

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
  stream. Uses the `SSEManager` pattern from LifeEventController for
  parsing SSE events from a fetch response.
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
- SessionPipeline NOT used per-tenant — SessionPipeline is fleet management
  infrastructure, overkill for a single avatar. Instead: CognitionCore's
  internal LLM calls (mood appraisal, BDI) use `AgentProvider.invoke()`
  directly (single-shot, not multi-turn) via a thin SessionPipeline adapter.
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

**10. Tool registration**
- AgentSessionInit.mcpServers configured at session-open time
- Points at the platform's MCP server endpoint (HTTP MCP transport)
- All 6 @McpDomain services available: life/tasks, life/actors, life/cases,
  life/dashboard, life/analytics, life/events
- Playbook execution: via life/cases domain (start case with template)
- Tool descriptions enriched with domain context for the LLM
- **Security note:** @McpDomain-generated REST endpoints currently lack
  @RolesAllowed (LifeRestSecurityTest is disabled). MCP tool access in
  this issue is gated by the avatar session's own authentication —
  only authenticated household members can converse with the avatar.
  Full MCP auth wiring is tracked separately.

**11. Confirmation flow**
- The avatar's confirmation flow is SEPARATE from the engine's
  ActionRiskClassifier (which gates worker actions during case execution).
  The avatar is a conversational agent — it decides when to confirm based
  on the action type and its system prompt instructions.
- System prompt includes confirmation rules: "Always confirm before starting
  a case, creating a financial commitment, or approving an oversight gate.
  Direct lookups, status queries, and low-risk task creation can proceed
  without confirmation."
- Avatar emits ConfirmationRequired event for consequential actions
- UI renders confirmation dialog with description + consequences
- User confirms → avatar executes via MCP tool
- User declines → avatar acknowledges and suggests alternatives
- Junior users: cannot confirm consequential actions. Avatar explains
  that admin or member approval is needed.

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
avatar. Two mechanisms:

1. **Dirty flag + rebuild:** Significant domain events set a dirty flag.
   Next user message triggers domain context rebuild, which is injected
   as a system-level context update message (not via clear()).
2. **Fallback: session close + reopen:** If the backend's clear() is a
   no-op (platform default), fall back to closing and reopening the session.
   This loses conversation history but ensures fresh context.

Note: `AgentSession.clear()` default implementation is empty. The spec
does NOT rely on it — the dirty flag + context update message approach
works regardless of the backend's clear() implementation.

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
```

Phase B (when published):
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
  cap, dirty flag invalidation
- `AvatarEventTest` — verify complete mapping from AgentEvent (all 6 variants)
  to AvatarEvent (all 8 variants)
- `LifeAvatarDescriptorProviderTest` — verify descriptor configuration

### Integration tests (@QuarkusTest)
- `LifeAvatarResourceTest` — REST endpoint: message send returns SSE stream,
  session close, role-based access (admin/member/junior all admitted)
- `LifeCognitiveAgentServiceTest` — session lifecycle, multi-turn conversation,
  domain context injection. Uses canned LLM responses (test backend).

### Frontend tests
- `life-conversation-panel` — collapsed/expanded rendering, message display,
  streaming token append, confirmation dialog, thinking indicator

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
