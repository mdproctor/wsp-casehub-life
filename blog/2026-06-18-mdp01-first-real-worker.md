---
title: "The first real worker (and why it's not quite Layer 7)"
date: 2026-06-18
series: casehub-life
entry: mdp01
tags: [casehub-life, layer-7, openClaw, langchain4j, quarkus, testing]
---

The `book-appointment-agent` now calls an actual LLM. That's the headline.

The more interesting story is what I discovered about how casehub-engine actually executes workers — and why "OpenClaw as LLM backend" and "OpenClaw as WorkerProvisioner" are two completely different things that I had confused for most of the design phase.

---

I came into this session thinking life#25 was about wiring `OpenClawWorkerProvisioner`. The CLAUDE.md Layer 7 entry said "casehub-openclaw as the WorkerProvisioner", and the issue was titled "apply the settled function-as-worker abstraction". I'd connected these incorrectly: I thought engine#463's settled abstraction *was* the provisioner path.

It isn't. There are two distinct execution paths in the engine and they don't compose.

`WorkerFunction.AgentExec(Agent)` runs when a binding fires and an *inline* worker with matching capabilities is found in the case definition. The engine routes it to `DefaultWorkerExecutor.executeSync(agent::execute)` on a virtual thread. The result is synchronous — `Agent.execute()` blocks, returns a `WorkerResult`, and the case context is updated immediately.

`WorkerProvisioner` is the *fallback* — called only when no inline worker matches. It registers an external agent and returns immediately. The result arrives asynchronously via a Qhorus channel delivery: DONE signal → `QhorusMessageSignalBridge` → `CaseHubRuntime.signal()`. The case context is updated through the signal, not through a `WorkerResult`.

You cannot substitute one for the other. If you want a synchronous LLM call through `Agent.execute()`, you're on the `AgentExec` path. If you want OpenClaw to autonomously execute and deliver results via its heartbeat mechanism, you're on the `WorkerProvisioner` path. The two paths touch different code, have different delivery guarantees, and would require different case definitions to work with.

What life#25 delivered is the first path: `WorkerFunction.AgentExec(Agent)` with OpenClaw's `/v1/chat/completions` as the LLM backend. The booking agent sends a synchronous HTTP request to OpenClaw's OpenAI-compatible endpoint, receives structured JSON (`BookingResult`), and returns a `WorkerResult`. The case proceeds normally from there. OpenClaw in this mode is a smart LLM server — it answers the question using its model, but it's not routing to any skills, not using its calendar integration, not interacting with Home Assistant.

That's not Layer 7 in the sense ARC42STORIES describes. ARC42STORIES §9.4 says Layer 7 is "banking API aggregation, Google Calendar integration, Home Assistant smart home control, WhatsApp/SMS follow-up." That requires the `WorkerProvisioner` path, filed as life#37 and life#38. What landed today is the pattern validation: AgentExec wired end-to-end, the `AgentDescriptor` identity convention (`openclaw:health-agent@1` per `{model-family}:{persona}@{major}`) established, the `TestLifeOpenClawChatModelProvider` CDI alternative in place.

---

The CDI testing story was the unexpected complication.

My first instinct for the integration test was `@InjectMock` — the standard Quarkus test double mechanism. It worked for the first test class that ran but broke everything else. The symptom was `BlackboardEventCodecRegistrar.onStart()` throwing "Already a default codec registered". I spent a while looking at codec registration before I understood what was actually happening.

`@InjectMock` changes the CDI bean configuration for the annotated test class. When Quarkus switches from a test class without `@InjectMock` to one with it, the CDI profile hash changes and Quarkus *restarts the application*. During restart, `BlackboardEventCodecRegistrar` tries to register Vert.x default codecs again — but Vert.x's `CodecManager` throws if a codec is already registered from the previous start. All subsequent `@QuarkusTest` classes then fail to start with the same error.

The fix: an `@Alternative @Priority(10) @ApplicationScoped` CDI test bean registered in `quarkus.arc.selected-alternatives` in test config. This is part of the *base* CDI profile — no profile change occurs between test classes, no Quarkus restart, no codec conflict. The test bean implements `doChat(ChatRequest)` directly (because `AiMessage` and `ChatResponse` can't be Mockito-mocked — final methods) and detects "unavailable" in the rendered user message to serve the right response for each test path.

One other thing I confirmed while reading source: `ChatModel.chat(ChatRequest)` is a *default* method that calls `doChat(ChatRequest)`. The override point for test doubles is `doChat()`, not `chat()`. This isn't in any LangChain4J documentation I could find — I had to read the source to discover it.

There's a broader issue underneath this. All `@QuarkusTest` classes in the project are currently skipping because Quarkus's augmentation phase fires `StartupEvent` observers to validate CDI wiring — this registers the codecs — and then the test execution fires them again. The engine fix is to make `BlackboardEventCodecRegistrar.onStart()` idempotent (`try/catch` on the registration). Filed as engine#536. Until that lands, unit tests and non-Quarkus tests pass, integration tests skip.

---

The `AgentDescriptor` detail is worth noting explicitly because it connects to Layer 6's attestation work.

`Worker.Builder.build()` doesn't enforce `agentDescriptor` — the field is silently nullable. Omitting it compiles cleanly. But without it, the trust system has no identity to score, and the attestation pipeline can't attribute outcomes. I added a test (`bookAppointmentWorkerHasAgentDescriptor()` in `AppointmentCycleCaseHubTest`) specifically because there's no build-time guard. The test enforces what the builder won't.

The agentId `"openclaw:health-agent@1"` follows the `{model-family}:{persona}@{major}` convention already documented in `docs/specs/life-actor-model.md`. The same value needs to be the config map key in casehub-openclaw-casehub when `WorkerProvisioner` is wired — `resolveAgentId()` returns the config map key and registers it in the trust system by that name. If the key is `"health-agent"` there but the AgentDescriptor says `"openclaw:health-agent@1"` here, the trust system can't correlate the executing agent with its descriptor. Getting that right now costs nothing; fixing it after trust data has accumulated would be a migration.

---

Six garden entries this session, four in casehub-engine domain and two in jvm:

- `Agent.build()` bakes `ChatModel` once — `@InjectMock` is silently ignored after first `getDefinition()` call
- `@InjectMock` → Quarkus CDI restart → `BlackboardEventCodecRegistrar` double-registration → all subsequent `@QuarkusTest` fail
- `AiMessage` and `ChatResponse` can't be Mockito-mocked (final methods) — use real constructors
- `@Alternative @Priority(10)` CDI test bean as the correct alternative to `@InjectMock`
- `ChatModel.doChat(ChatRequest)` is the override point, not `chat(ChatRequest)` (default method)
- `WorkerFunction.AgentExec` and `WorkerProvisioner` are completely different engine execution paths

The `@InjectMock` → codec restart chain is the one I'd most want other casehub developers to know before they spend three hours on it.

Next: engine#536 (idempotent codecs) would unblock the integration tests. Life#37 (WorkerProvisioner wiring) or life#38 (`/hooks/agent` direct-call bridge) would move the booking worker from "LLM server" to "real skill execution". Either is a meaningful next step for Layer 7.
