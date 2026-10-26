# HANDOFF — casehub-life

**Last updated:** 2026-10-26
**Branch:** main
**State:** Between issues — ready for next batch

---

## Current State

Project is on `main`. All prior issues closed. One unpushed commit on main:
- `217ce1d` — `protocol(PP-20261026-8a9dce): no-openclaw-dependencies`

Recent refactoring landed on main:
- `cb492b6` — Removed OpenClaw dependencies, replaced with platform `AgentProvider`
- `LifeOpenClawChatModelFactory` → `LifeAgentProviderFactory` (no-op until platform routing wired)
- `LifeAgent.MODEL_FAMILY` changed from `"openclaw"` to `"life"`
- 11 OpenClaw CDI exclusions removed from application.properties
- `b425955`/`b42e690`/`886bb05` — blocks-ui → casehub-mosaic → casehub-agency rename chain

## Next Work — Connector Wiring Batch (#131–134)

Four issues, same mechanical pattern. Recommended as a single branch.

### What each issue adds

| Issue | Connector | MCP tools | Workers affected |
|-------|-----------|-----------|-----------------|
| #131 | ContactsPlatform | `list_contacts`, `search_contacts` | request-quote, job-monitoring, book-appointment, find-alternative, needs-assessment, care-plan, get-quotes (7) |
| #132 | EmailPlatform | `send_email` | pre-visit-prep, produce-report, confirmation, record-completion (4) |
| #133 | DocumentPlatform | `sync_documents` | produce-report, care-plan, confirmation, record-completion (4) |
| #134 | LocationPlatform | place search, directions, geocoding | destination-research, find-alternative, get-quotes (3) |

### What changes per issue (same pattern)

1. Add MCP tool name references to system prompt text blocks in CaseHub classes
2. Add corresponding fields to response schema records (e.g. `contactId`, `emailMessageId`, `documentId`, `address`/`distance`/`travelTime`)
3. Update tests for new schema fields (TestLifeAgentProviderFactory canned responses)

### Wiring verified

- `LifeTypedCaseHub.agentWorker()` pipeline unchanged by refactor
- `LifeAgentProviderFactory.invoke()` is no-op — tool references are forward-looking (same pattern as existing `send_chat`, `calendar_create_event`, `iot_get_state`, `bank_get_transactions`)
- All 8 CaseHub classes and 39 response schema records are intact
- Sentinel prompts in `LifeHeartbeatJob` also candidates for tool additions

### Files to touch

- `ContractorCoordinationCaseHub.java` — #131 (request-quote, job-monitoring)
- `AppointmentCycleCaseHub.java` — #131 (book-appointment, find-alternative), #132 (pre-visit-prep)
- `CareCoordinationCaseHub.java` — #131 (needs-assessment, care-plan), #133 (care-plan)
- `HomeMaintenanceCaseHub.java` — #131 (get-quotes), #133 (record-completion), #134 (get-quotes)
- `FinancialReviewCaseHub.java` — #132 (produce-report), #133 (produce-report)
- `TravelPlanCaseHub.java` — #132 (confirmation), #133 (confirmation), #134 (destination-research)
- Response schema records in `app/engine/agent/` — add fields per worker
- `TestLifeAgentProviderFactory.java` — update canned responses for new fields
- `LifeHeartbeatJob.java` — sentinel prompts may gain tool references

### After the batch

- #136 (BankPlatform PISP) — M/Med, extends record-payment with payment initiation
- #135 (CommercePlatform) — L/High, needs new case definitions (procurement/gifting)

## Stale References Cleaned

- Issue #238 (`issue-238-rename-mosaic`) — ghost reference in work DB. Branch never existed. Issue doesn't exist on GitHub. Ignored.
- Issue #126 branch — closed and stamped (`1309e22`). Workspace was stale on this branch; switched back to main.
