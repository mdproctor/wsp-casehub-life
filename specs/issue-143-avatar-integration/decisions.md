# Decisions — Issue #143: Avatar Integration

## D1: Full CognitionCore integration

**Choice:** Wire the complete neocortex CognitionCore with all cognitive subsystems (mood, drives, user model, mental model, strategy, narrative, goals, memory hygiene, inner life, temporal focus, reflection, consolidation).
**Alternatives:**
- Basic prompt pipeline (eidos identity + domain context only) — simpler but avatar is just a branded chatbot until Phase 2
- Selective subsystems (mood + drives + user model only) — middle ground but arbitrary boundary
**Rationale:** CognitionCore is the differentiator. The avatar should feel alive from day one. Phase 1 "mechanical wiring" means wiring the real cognitive stack, not a placeholder.
**Trade-offs:** Heavier dependency surface. CognitionCore's tick model needs adaptation from simulation-driven to conversation-driven.
**Sources:** Neocortex cognitive architecture roadmap (Phase 1-5), CognitionCore.java, epic #139 D2 (avatar is central intelligence)
**Exploration:** quick
**Status:** captured

## D2: Single AgentProvider for all LLM calls

**Choice:** One AgentProvider instance serves both the main conversation loop and CognitionCore's internal calls (mood appraisal, BDI extraction).
**Alternatives:**
- Separate providers (cheaper model for cognitive calls) — more complex, marginal cost savings
- Platform routing (LifeOpenClawChatModelFactory pattern) — most flexible but adds configuration surface
**Rationale:** Simplicity. CognitionCore's internal calls are cheap side-channel requests. Same model ensures consistent reasoning quality across cognitive and conversational layers.
**Trade-offs:** Can't optimise cost by routing cognitive calls to a cheaper model. Acceptable for a personal-use system.
**Sources:** AgentProvider.java (platform-agent-api), LifeOpenClawChatModelFactory (existing pattern)
**Exploration:** quick
**Status:** captured

## D3: Expandable conversation panel

**Choice:** Default 280px collapsed state showing recent messages. Click to expand to ~480px or overlay/drawer mode for full conversation.
**Alternatives:**
- Replace dock with drawer (400-500px sliding drawer) — more disruptive to grid layout
- Fixed wider panel (change all right dock panels to ~400px) — affects Family, Money, Comms panels
**Rationale:** Best UX without breaking the existing dock layout. The conversation panel needs more space than typical dock panels, but shouldn't force layout changes on other panels.
**Trade-offs:** Two rendering modes to implement (collapsed summary + expanded conversation).
**Sources:** life-ui home-view.ts dock layout (32px dock bar + 280px toggle panel)
**Exploration:** quick
**Status:** captured

## D4: Server-side multi-turn sessions

**Choice:** Use AgentProvider.openSession() for server-side multi-turn sessions. Session persists while the user's browser tab is open. CognitionCore runs tick() and recordInteraction() within the session lifecycle.
**Alternatives:**
- Client-managed history (full history re-sent each call) — expensive, loses cognitive state
- Hybrid with DB persistence (sessions survive browser close) — adds DB schema, doesn't align with CognitionCore's in-memory model
**Rationale:** Natural fit for platform's session model. CognitionCore accumulates cognitive state in-memory across interactions — server-side sessions keep this alive.
**Trade-offs:** Sessions are lost on server restart. Acceptable for Phase 1 — DB-backed session persistence can be added later.
**Sources:** AgentProvider.openSession(), AgentSession.java, CognitionCore.recordInteraction()
**Exploration:** quick
**Status:** captured

## D5: Tool-enabled agent with full MCP access

**Choice:** Avatar uses MCP tools autonomously within conversation. Full access to @McpDomain-generated endpoints (WorkItems, cases, actors, commitments, calendar, etc.) plus playbook execution via MCP. Confirmation flow for consequential actions.
**Alternatives:**
- Suggestion-only (avatar suggests, user clicks buttons) — defeats central intelligence vision (D2)
- Phased: suggest now, tools in #144 — clean separation but two integration passes
**Rationale:** The avatar IS the execution surface (epic #139 D2, D3). The three execution tiers (direct MCP, playbooks, engine cases) all work through MCP tools. Future security gating will allow/deny specific tools.
**Trade-offs:** Consequential actions need confirmation flow to prevent unintended execution. Security model for tool access is deferred (not part of this issue).
**Sources:** @McpDomain annotation, CoordinationTemplateCatalog (playbook MCP), epic #139 D3 (three execution tiers)
**Exploration:** quick
**Status:** captured

## D6: Transport via pages EventStream/EventConnection

**Choice:** Use the pages-data unified transport layer. EventStream/EventConnection with PushSource abstraction — supports both SSE and WebSocket backends. EventConnection provides bidirectional send/listen for WebSocket.
**Alternatives:**
- Custom SSE-only (LifeEventController pattern) — one-directional, can't send messages via same channel
- Raw WebSocket — reinvents what pages-data already provides
**Rationale:** Pages already solved transport unification. EventConnection.send() supports bidirectional communication. PushSource abstraction makes SSE/WebSocket interchangeable.
**Trade-offs:** None significant — this is using existing platform infrastructure.
**Sources:** pages-data EventStream, EventConnection, PushSource, SSEManager
**Exploration:** quick
**Status:** captured

## D7: Full state snapshot for domain context

**Choice:** On session open, inject comprehensive life state summary into the system prompt. CognitionCore's promptSections() add cognitive state on top. Refresh on SSE events (case status change, SLA breach).
**Alternatives:**
- Lazy retrieval via tools (avatar calls MCP tools when it needs context) — lighter prompt but avatar can't be proactive
- Layered: summary + tools (compact summary in prompt, tools for detail) — balances prompt size against proactivity
**Rationale:** The avatar must always know what's happening. Proactive suggestions ("Bob's ETA is overdue — want me to follow up?") require the avatar to have full state awareness without being asked.
**Trade-offs:** Larger system prompt. Token cost increases with life complexity. May need summarisation for users with many active cases.
**Sources:** DashboardService.buildBriefing(), PendingActionsService, ExternalActorService, LifeCaseService
**Exploration:** quick
**Status:** captured

## D8: Dynamic identity via CognitionCore

**Choice:** AgentDescriptor provides base identity. CognitionCore evolves it: mood affects tone, drives shape priorities, narrative builds over time. The avatar develops personality through interaction.
**Alternatives:**
- Static YAML descriptor — fixed personality, predictable but lifeless
- Base + evolved layers (D8 pattern from neocortex roadmap Phase 4) — most architecturally sound but Phase 4 isn't complete
**Rationale:** This IS what makes CognitionCore valuable. The avatar isn't just a persona — it develops. CognitionCore's prompt sections dynamically modify the system prompt based on accumulated cognitive state.
**Trade-offs:** Personality drift if cognitive state isn't well-calibrated. May need reset/recalibration mechanisms. Phase 4 (adaptive brief) will supersede this with a more structured approach.
**Sources:** CognitionCore.promptSections(), eidos AgentDescriptor, neocortex cognitive roadmap D8
**Exploration:** quick
**Status:** captured

## D9: Life Cognitive Agent Service architecture

**Choice:** Approach A — dedicated LifeCognitiveAgentService in app/ that composes CognitionCore + AgentProvider + domain context + MCP tools directly in casehub-life.
**Alternatives:**
- Thin wrapper over blocks AvatarCognition — adds blocks dependency, violates layering rule, AvatarCognition being refactored
- Direct LLM without CognitionCore — contradicts D1
**Rationale:** Clean ownership. CognitionCore is a library, not a framework. casehub-life owns its domain. No unnecessary coupling to blocks. AgentProvider SPI handles LLM mechanics.
**Trade-offs:** Main engineering challenge is adapting CognitionCore's tick model from simulation-driven to conversation-driven. One-time integration pattern.
**Depends on:** D1 (full CognitionCore), D2 (single AgentProvider), D4 (server-side sessions)
**Sources:** CognitionCore.java, CognitionCoreFactory.java, AgentProvider.java, LifeOpenClawChatModelFactory (existing pattern)
**Exploration:** quick
**Status:** captured
