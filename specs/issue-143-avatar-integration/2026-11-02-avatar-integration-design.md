# Avatar Integration — Neocortex Cognitive Agent into life-ui

> Design spec for life#143 (P1.4 in epic #139)

## Overview

Wire the neocortex CognitionCore as the primary conversational surface in
casehub-life. The avatar is the central intelligence (D2 from the roadmap) —
it understands intent, knows the user's life state, uses MCP tools to act,
and develops personality through cognitive subsystems.

This is NOT a chatbot with a system prompt. CognitionCore provides mood
evolution (PAD model), drives, user modelling, mental model (BDI extraction),
strategy learning, narrative arc, goal proposals, memory hygiene, inner life,
temporal focus, reflection, and consolidation. The avatar feels alive because
it IS cognitively alive.

## Architecture

### Server-side components

**1. LifeCognitiveAgentService** (`app/service/`)
- `@ApplicationScoped`, manages CognitionCore and AgentSession lifecycle
- One CognitionCore instance per tenant (created via `CognitionCoreFactory`)
- One SessionPipeline per tenant (backed by platform AgentProvider)
- Session lifecycle: create on first user message, maintain while tab is open,
  close on disconnect/timeout

Key methods:
- `createSession(tenantId, principalId)` → session ID
- `sendMessage(sessionId, userMessage)` → `Multi<AvatarEvent>` (streaming)
- `closeSession(sessionId)`

**AgentProvider implementation:** The existing LifeAgentProviderFactory
(OpenClaw wrapper) does NOT support openSession(). This issue requires
a new AgentProvider implementation backed by a direct LLM SDK (e.g.
casehub-platform-agent-openai for Claude/Anthropic). This replaces
the OpenClaw wrapper for the avatar use case — engine workers continue
to use the existing OpenClaw path.

On each message:
1. Assemble system prompt: eidos descriptor + CognitionCore.promptSections() +
   domain context snapshot
2. Send to AgentSession.query() with MCP tools
3. Stream response as AvatarEvent (text deltas, tool calls, structured cards)
4. Call CognitionCore.recordInteraction() (mood appraisal, BDI extraction,
   user model update, strategy learning)

**2. LifeDomainContextProvider** (`app/service/`)
- `@ApplicationScoped`, assembles comprehensive life state snapshot
- Gathers from: DashboardService (briefing), PendingActionsService,
  ExternalActorService (with TrustProfiles), LifeCaseService (active cases),
  LifeCommitmentService (open commitments), LifeCalendarSyncService
  (unresolved conflicts), recent SLA breaches
- Renders as structured text for system prompt injection
- Invalidation: SSE events trigger refresh (WORK_ITEM_UPDATED, SLA_BREACH,
  CASE_STARTED, CALENDAR_CONFLICT_DETECTED)

**3. LifeAvatarDescriptor** (`app/engine/agent/`)
- Provides the base AgentDescriptor for the life avatar
- Identity: name, personality traits (helpful, proactive, attentive to
  household obligations), disposition profile, voice, constraints
- Capability tags: all 8 life domains
- Goals: keep the household running smoothly, surface risks before they
  become crises, help the user make informed decisions
- Config-driven: `casehub.life.avatar.*` properties

**4. LifeAvatarResource** (`app/resource/`)
- REST + WebSocket endpoint for the conversation surface
- `POST /avatar/sessions` — create session → 201 with session ID
- `DELETE /avatar/sessions/{id}` — close session
- WebSocket `/avatar/ws/{sessionId}` — bidirectional: user sends messages,
  receives streamed response events (text deltas, tool call notifications,
  structured cards, completion)
- `@RolesAllowed("household-member")` — all household roles can converse

**5. AvatarEvent** (`api/`)
- Sealed interface: `TextDelta(text)`, `ToolCallStarted(toolName, args)`,
  `ToolCallCompleted(toolName, result)`, `ActionCard(type, data)`,
  `ConfirmationRequired(actionId, description, consequences)`,
  `SessionComplete(usage)`
- Maps from platform AgentEvent to life-specific avatar events
- ActionCard renders structured responses: task created, case started,
  conflict resolved, commitment made

### Frontend components

**6. life-conversation-panel** (`life-ui/src/panels/`)
- Lit 3.x component replacing `life-comms-mock`
- Two modes:
  - **Collapsed** (280px, default): recent messages summary, input field,
    notification count
  - **Expanded** (~480px overlay): full message history, typing indicator,
    structured response cards, action confirmations
- Uses `EventConnection` from pages-data for WebSocket transport
- Message rendering: user bubbles (right), avatar bubbles (left), action
  cards (full-width), confirmation dialogs (modal)
- Markdown rendering via `marked` + `dompurify` (already in deps)
- Streaming: tokens arrive via WebSocket, append to current avatar bubble
  with typing animation

**7. Integration with home-view**
- Replace `life-comms-mock` registration in right dock with
  `life-conversation-panel`
- Expand/collapse toggle: dock bar icon switches panel mode
- Notification badge: unread avatar messages via LifeEventController

### CognitionCore wiring

**8. CognitionCore configuration**
- All subsystems enabled (CognitionConfig.all())
- SessionPipeline: backed by platform AgentProvider via SessionLifecycleManager
- MoodOrchestrator: PAD model with life-calibrated defaults
- DriveOrchestrator: life-specific drives (household efficiency, family
  wellbeing, financial prudence, health vigilance)
- UserModelOrchestrator: builds model of the household member over time
- MentalModelOrchestrator: BDI extraction from conversations
- NarrativeOrchestrator: tracks ongoing life narratives ("the boiler saga",
  "Ella's school year")
- GoalProposalOrchestrator: proposes goals from patterns (proactive)
- MemoryHygieneOrchestrator: manages memory store pruning
- InnerLifeOrchestrator: generates internal thoughts between interactions
- TemporalFocusOrchestrator: awareness of time-sensitive obligations
- ReflectionRetrievalOrchestrator: learns from past conversations

**9. Cognitive tick model**
- CognitionCore.tick() called after each conversation turn (not on a timer)
- This differs from simulation-driven agents where tick() runs on a clock
- The tick processes: mood decay, drive updates, attention shifts, narrative
  evolution, goal priority adjustments
- CognitionCore.recordInteraction() called with user message + agent response
  after each turn — drives mood appraisal and BDI extraction via SessionPipeline

### MCP tool access

**10. Tool registration**
- AgentSessionInit.mcpServers configured at session-open time
- Points at the platform's MCP server endpoint (HTTP MCP transport)
- All 6 @McpDomain services available: life/tasks, life/actors, life/cases,
  life/dashboard, life/analytics, life/events
- Playbook execution: via life/cases domain (start case with template) or
  new life/playbooks McpDomain for direct playbook dispatch
- Tool descriptions enriched with domain context for the LLM

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
- Future: HouseholdActionType thresholds from risk-policy.yaml can inform
  the avatar's confirmation rules (not wired in this issue)

## Dependencies

### New Maven dependencies

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-cognition</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-cognition-api</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-cognitive-api</artifactId>
</dependency>
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

Most of these are likely already transitive via existing deps (check
casehub-engine, casehub-openclaw-core). Only add what's not already on
the classpath.

### Cross-repo dependencies

- casehub-neocortex: CognitionCore, CognitionCoreFactory, all cognitive
  subsystem types. No changes needed in neocortex — we consume the existing API.
- casehub-eidos: AgentDescriptor, DispositionProfile. No changes needed.
- casehub-platform: AgentProvider, AgentSession, AgentEvent, SessionPipeline,
  SessionLifecycleManager. No changes needed.
- casehub-pages: EventConnection, EventStream, PushSource (WebSocket backend).
  No changes needed — already consumed by life-ui.

### No foundation gates

This issue doesn't require new foundation capabilities. All platform
primitives needed (AgentProvider, MCP tools, SSE events) are already
available and wired.

### Cognitive state persistence

CognitionCore accumulates state in-memory during a session, but key
subsystems persist across sessions via neocortex stores:
- **MoodState** — persisted via `moodPersister` callback → CaseMemoryStore
- **UserModel** — persisted via UserModelOrchestrator → CaseMemoryStore
- **MentalModel** — BDI extractions persisted to CaseMemoryStore
- **Narrative** — narrative fragments persisted to MindMapStore
- **Engagement events** — persisted via `engagementPersister` callback

On session start, CognitionCore rehydrates from these stores. The avatar
does NOT lose personality on tab close — it loses only the in-flight
conversation history. Mood, learned user preferences, mental model, and
narrative arc persist across sessions.

Conversation history itself is transient (lost on session close). This is
acceptable for Phase 1. If conversation persistence is needed later, it
belongs in neocortex (session serialization), not in a life-specific table.

### Context refresh mid-session

AgentSession.query() takes only a user prompt — there is no
updateSystemPrompt() method. Domain context changes (new SLA breach,
case completion) cannot be injected into an active session's system
prompt directly.

Resolution: use `AgentSession.clear()` to reset context, then send a
fresh system prompt on the next query. This preserves the session
(no close/reopen needed) while allowing the system prompt to be
rebuilt with current domain context. CognitionCore's in-memory
cognitive state is unaffected by clear() since it lives outside the
AgentSession.

Trigger: significant domain events (SLA_BREACH, CASE_COMPLETED,
CALENDAR_CONFLICT_DETECTED) received via SSE trigger a context
invalidation flag. The next user message causes a clear+rebuild
before sending to the LLM.

### Avatar audit trail

Avatar-initiated actions must be auditable. When the avatar executes
an MCP tool that creates, modifies, or deletes data (task creation,
case start, commitment, GDPR erasure), the tool execution is
attributed to the user's principal (not the avatar). The avatar is
an agent acting on behalf of the user — the user's actorId appears
in ledger entries, not a synthetic avatar actorId.

Tool call events (ToolCallComplete) are logged via the existing
`LifeEventBroadcaster` SSE infrastructure for UI transparency.
Full audit of avatar actions can be reconstructed from the ledger
(same as any other platform action).

### Multi-user model

One avatar per tenant, shared across all household members. All three
roles (household-admin, household-member, household-junior) converse
with the same avatar instance. CognitionCore's UserModelOrchestrator
tracks per-user interaction patterns, and MentalModelOrchestrator
builds per-user BDI models. The avatar knows who it's talking to via
CurrentPrincipal.

Oversight gates: when the avatar proposes an action that requires
admin approval, the ConfirmationRequired event is emitted. If the
current user lacks the required role, the avatar explains that admin
approval is needed and does not proceed.

## Data model

No new JPA entities. No Flyway migrations. CognitionCore state
persists via neocortex memory stores (CaseMemoryStore, MindMapStore),
not via new life-specific tables.

## Configuration

```properties
# Avatar identity
casehub.life.avatar.name=Life
casehub.life.avatar.personality=helpful, proactive, attentive
casehub.life.avatar.model=claude-sonnet-4-5

# Session management
casehub.life.avatar.session.idle-timeout=PT30M
casehub.life.avatar.session.max-concurrent=5

# MCP server for tool access
casehub.life.avatar.mcp.server-url=http://localhost:8080/mcp
```

## Testing strategy

### Unit tests
- `LifeDomainContextProviderTest` — verify context assembly from mocked services
- `AvatarEventTest` — verify mapping from AgentEvent to AvatarEvent
- `LifeAvatarDescriptorTest` — verify descriptor configuration

### Integration tests (@QuarkusTest)
- `LifeAvatarResourceTest` — REST endpoint: session create/close, message send
- `LifeCognitiveAgentServiceTest` — CognitionCore integration: verify
  tick/recordInteraction called, prompt sections injected. Uses
  TestLifeOpenClawChatModelFactory pattern for canned LLM responses.

### Frontend tests
- `life-conversation-panel` — collapsed/expanded rendering, message display,
  streaming token append, confirmation dialog

## Scope boundaries

### In scope (this issue)
- Server-side cognitive agent service with full CognitionCore
- REST + WebSocket conversation endpoint
- Domain context injection (full state snapshot)
- MCP tool access via AgentSessionInit
- Conversation panel in life-ui (replacing comms-mock)
- Confirmation flow for consequential actions
- Session lifecycle management

### Out of scope (subsequent issues)
- Intent classification and three-tier dispatch (#144)
- Avatar-driven UI transformation (#145)
- Voice intake (#120)
- Email/WhatsApp intake (#112, #113)
- Family mind map integration (#121)
- NLP cost optimisation (#137)
- Session persistence across server restarts

## References

- CognitionCore.java — neocortex/cognition/src/main/java/.../core/CognitionCore.java
- AgentProvider.java — platform/agent-api/src/main/java/.../agent/AgentProvider.java
- AgentSession.java — platform/agent-api/src/main/java/.../agent/AgentSession.java
- SessionPipeline — platform/agent-session-core (concrete class, not interface)
- AgentSessionInit — platform/agent-api (carries mcpServers for tool access)
- Neocortex cognitive architecture roadmap — neocortex/specs/issue-303/
- Epic #139 — digital life assistant roadmap (D1-D9 decisions)
- EventConnection — pages-data (unified SSE/WebSocket transport)
- life-comms-mock.ts — current placeholder panel being replaced
- LifeEventController — existing SSE reactive controller pattern
- @McpDomain — platform annotation generating dual MCP/REST/GraphQL
