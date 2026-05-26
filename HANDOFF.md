# Handoff — casehub-life Layer 1 brainstorming (incomplete)
2026-05-26

## Last Session

Ran work-start for issue #2 (Layer 1: domain baseline), completed all brainstorming
clarifying questions, and did significant cross-repo housekeeping (LAYER-LOG format,
terminology, 7 issues filed across 4 repos). No code written yet. Branch
`issue-2-layer1-naive-domain` is live in both repos. All design decisions resolved —
next session presents design, writes spec, writes plan, implements.

## Immediate Next Step

Start a fresh session. Resume brainstorming at design presentation stage — all clarifying
questions are done. Resolve the one open question (ExternalActor FK, see below), present
entity shapes → REST API → Flyway → showcase scenario. Write spec to `docs/specs/`.
Invoke `writing-plans`. Then implement.

**One open question:** Does `ExternalActor` have an FK to `HouseholdTask` in Layer 1
(optional `externalActorId` on task for contractor coordination), or does that
relationship only materialise in Layer 3 when Qhorus commitments are introduced?

## Design Decisions (Layer 1) — Do Not Re-Debate

**Module structure:**
- `api/` = domain vocabulary only — enums, constants, value records. Zero framework imports, zero JPA.
- `app/` = direct Panache Active Record entities + services + REST resources + Flyway.
- No service SPIs in `api/`. No `@DefaultBean`/`@Alternative` substitution pattern.
- Services grow across layers (Layer 2 adds WorkItem call alongside existing `persist()`) — not replaced.

**Naming conventions:**
- No `Naive*` prefix anywhere. Layer 1 code is production quality.
- Gap commentary in LAYER-LOG.md accountability gaps table — not as code comments.
- "Integrate" not "adopt". "Domain baseline" not "naive Java" in docs.

**`api/` types:**
- `LifeDomain` enum — HOUSEHOLD, HEALTH, FINANCE, FAMILY_SCHEDULING, TRAVEL, LEGAL, CONTRACTOR_COORDINATION, ELDER_CARE
- `LifeCapabilities` — capability tag constants class
- `LifeTrustDimensions` — trust dimension constants class
- `ActorType` enum — AI_AGENT, HOUSEHOLD_PRINCIPAL, EXTERNAL_HUMAN
- `model/` package: `HouseholdTaskStatus`, `LifeGoalStatus`, `ExternalActorType` enums

**`app/entity/` types:**
- `HouseholdTask` — domain, title, description, deadline, `slaHours`, status, assignedTo
- `LifeGoal` — domain, title, targetDate, status
- `LifeEvent` — domain, title, occurredAt, description
- `ExternalActor` — name, contactMethod, contactValue, type (life-specific — not in casehub-qhorus-api)

**`slaHours` on HouseholdTask from Layer 1** — correct domain modelling even though unenforced until Layer 2.

**Tests:**
- `ShowcaseScenarioTest` — single `@QuarkusTest` narrative: household week, demonstrates accountability gaps in sequence.
- Pure-Java unit tests for stateless domain logic (status transitions, validation).
- REST-level `@QuarkusTest` per resource.

**Flyway:** V100–V199. casehub-work owns V1–V21+; ledger owns V1000–V1007.

**Platform coherence:** clear — application-tier domain logic, right repo. auth-retrofit-readiness applies (thin REST, no auth types in service layer, injectable filter on list queries).

**Tutorial approach (revised this session):**
- Layer progression retained for human learning narrative (LAYER-LOG.md, blog)
- LLM-replayable layer state via `git log --grep="#N" --oneline` — no explicit tags needed
- Gap commentary in LAYER-LOG.md, not production code
- Production code is production quality at every layer

## Issues Filed This Session

| Issue | Repo | What |
|-------|------|------|
| life#9 | casehubio/life | Layer navigation index in LAYER-LOG.md |
| aml#34 | casehubio/aml | Layer navigation index in LAYER-LOG.md |
| aml#35 | casehubio/aml | Rename Naive* classes, dissolve tutorial/ package, update LAYER-LOG |
| clinical#38 | casehubio/clinical | Layer navigation index in LAYER-LOG.md |
| devtown#48 | casehubio/devtown | Layer navigation index in LAYER-LOG.md |
| devtown#49 | casehubio/devtown | Rename NaivePrReviewService |
| parent#74 | casehubio/parent | Update tutorial-strategy.md + AGENTIC-HARNESS-GUIDE.md — terminology, gap comment policy, augmented LAYER-LOG format, add casehub-life to scope |

## What Was Updated

- `LAYER-LOG.md` — full rewrite: "Domain baseline" not "Naive Java", "integrate" not "adopt", gap comments → accountability gaps table, architectural decisions section, navigation lines on all layers.

## Branch State

- Project repo: `issue-2-layer1-naive-domain` (scaffold only, no code commits)
- Workspace repo: `issue-2-layer1-naive-domain` (scaffold committed and pushed)

## Garden Entries to Keep In Mind

GE-20260420-d99177 (H2 @QuarkusTest contamination), GE-20260508-ce2285 (UUID-suffix keys),
GE-20260512-2c2eff (DOUBLE PRECISION not DOUBLE in Flyway), GE-20260428-096e90 (JPA FK without CASCADE),
GE-20260512-493c90 (*IT.java naming silently reports 0 tests).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #2 | Layer 1: present design → spec → plan → implement | L | Med | Start here — all design decisions resolved |
| #3 | Layer 2: + casehub-work SLA enforcement | M | Med | Blocked by Layer 1 |
| #4 | Layer 3: + casehub-qhorus commitment lifecycle | M | Med | Blocked by Layer 2 |

## References

- Specs: `docs/specs/life-automation.md`, `docs/specs/life-actor-model.md`
- LAYER-LOG: `LAYER-LOG.md` (updated this session)
- Branch: `issue-2-layer1-naive-domain`
