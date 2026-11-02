# HANDOFF — casehub-life

**Last updated:** 2026-11-02
**Branch:** main
**State:** #143 landed, #144/#145 need rethinking

---

## What Happened This Session

Designed, implemented, and landed #143 (avatar integration — P1.4).
Then discovered architectural boundary issues that reshape #144 and #145.

### #143 — Avatar integration (landed)

Full brainstorming cycle (9 decisions, 31 review findings across two
adversarial reviews), implementation plan (8 tasks, 4 batches), and
execution. Squashed to single commit on main.

**Backend (5 classes):**
- `AvatarEvent` — sealed interface, 8 variants in api/
- `LifeCognitiveAgentService` — per-user AgentSession management
- `LifeAvatarResource` — POST+SSE conversation endpoint
- `LifeDomainContextProvider` — life state summary for system prompt
- `LifeAvatarDescriptorProvider` — eidos identity for the avatar
- `DemoAvatarAgentProvider` — demo-mode multi-turn sessions
- `AvatarEventMapper` — AgentEvent → AvatarEvent (filters ToolCallDelta)

**Frontend:**
- `life-conversation-panel` — Lit 3.x, SSE streaming, markdown, dock panel
- Replaces comms-mock in right dock
- Vite config: missing aliases fixed, /avatar proxy added

**Key design decisions:**
- POST+SSE, not WebSocket (matches existing LifeEventSseResource pattern)
- Phase A without CognitionCore (artifacts not published yet)
- All three household roles admitted (@RolesAllowed)
- delegatedVia: "avatar" audit marker for avatar-initiated actions

**Blockers discovered (pre-existing, not caused by #143):**
- @QuarkusTest broken: casehub-work-engine NotificationTemplate constructor
  changed upstream. Needs casehub-work CI rebuild.
- Vite build broken: pages-schema zod export, mosaic-core applyTheme
  export. Needs pages and mosaic CI rebuilds.

### Architectural boundary corrections

Post-implementation conversation surfaced two rules:

1. **Cognitive → neocortex.** Intent classification, BDI extraction,
   drive-based reasoning belong in neocortex. Life configures dispatch
   rules and domain context, not cognitive capabilities.

2. **Reusable UI → mosaic.** The conversation panel, action cards,
   confirmation dialogs belong in mosaic's avatar package — not in
   life-ui. Mosaic already has a full avatar panel with 3D, TTS/STT.
   Pages has dock workbench, milkdown, PushSource, MCP tool adapter.

**Issues filed:**
- life#146 — migration tracking (move conversation panel to mosaic)
- mosaic#301 — epic: avatar text-conversation support
- mosaic#302-307 — six gap issues (tool visibility, action cards,
  confirmation flow, theme tokens, text-only mode, streaming)

### #144 needs splitting

Currently scoped as "intent resolution" all in life. Should split:
- **neocortex issue** — intent classification (cognitive: free text →
  structured intent, informed by drives/goals/mental model)
- **life issue** — dispatch wiring (domain: intent → tier selection →
  MCP/playbook/case, parameter extraction resolving life entities)

### #145 blocked

Avatar-driven UI is blocked until mosaic#301 lands. The dashboard
transformation should use mosaic components, not build new ones in
life-ui.

## Key Decisions

- Cognitive capabilities → neocortex, not app repos
- Reusable UI → mosaic, not app UIs
- #144 needs neocortex/life split before brainstorming
- #145 blocked on mosaic#301 (avatar text-conversation epic)
- CognitionCore deferred to Phase B (artifacts not published)

## Next Action

1. Split #144 into neocortex + life issues
2. Fix SNAPSHOT mismatches (work, pages, mosaic CI rebuilds)
3. Work mosaic#301 epic (avatar text-conversation support)
4. Then resume life epic #139 with corrected issue scoping
