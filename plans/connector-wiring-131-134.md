# Plan: Connector Wiring Batch (#131–134)

Wire four connector platform MCP tools into existing worker system prompts and response schemas.

**Branch:** `issue-131-connector-wiring-batch`
**Issues:** #131, #132, #133, #134
**Scale:** M (batch of 4 × S)
**Complexity:** Med (mechanical, same pattern repeated)

---

## Pre-flight

- [ ] Read HANDOFF.md for current state and verified wiring details
- [ ] Confirm `LifeAgentProviderFactory` pipeline intact (should be — verified 2026-10-26)
- [ ] Check connector SPI tool names match what issues reference (ContactsPlatform, EmailPlatform, DocumentPlatform, LocationPlatform in casehub-connectors)

## Batch 1 — #131 ContactsPlatform (7 workers)

Add `list_contacts`, `search_contacts` tool references to system prompts. Add `contactId` or similar fields to response schemas where contact lookup is the primary action.

- [ ] `ContractorCoordinationCaseHub` — request-quote, job-monitoring prompts
- [ ] `AppointmentCycleCaseHub` — book-appointment, find-alternative prompts
- [ ] `CareCoordinationCaseHub` — needs-assessment, care-plan prompts
- [ ] `HomeMaintenanceCaseHub` — get-quotes prompt
- [ ] Response schemas: `RequestQuoteResult`, `JobMonitoringResult`, `BookingResult`, `FindAlternativeResult`, `NeedsAssessmentResult`, `CarePlanResult`, `GetQuotesResult`
- [ ] Tests: update `TestLifeAgentProviderFactory` canned responses

## Batch 2 — #132 EmailPlatform (4 workers)

Add `send_email` tool reference. Add `emailMessageId` field to response schemas.

- [ ] `AppointmentCycleCaseHub` — pre-visit-prep prompt
- [ ] `FinancialReviewCaseHub` — produce-report prompt
- [ ] `TravelPlanCaseHub` — confirmation prompt
- [ ] `HomeMaintenanceCaseHub` — record-completion prompt
- [ ] Response schemas: `PreVisitPrepResult`, `ProduceReportResult`, `ConfirmationResult`, `RecordCompletionResult`
- [ ] Tests: update canned responses

## Batch 3 — #133 DocumentPlatform (4 workers)

Add `sync_documents` tool reference. Add `documentId` field to response schemas.

- [ ] `FinancialReviewCaseHub` — produce-report prompt
- [ ] `CareCoordinationCaseHub` — care-plan prompt (or issue-commitment if care plan is stored)
- [ ] `TravelPlanCaseHub` — confirmation prompt
- [ ] `HomeMaintenanceCaseHub` — record-completion prompt
- [ ] Response schemas: `ProduceReportResult`, `CarePlanResult`, `ConfirmationResult`, `RecordCompletionResult`
- [ ] Tests: update canned responses

## Batch 4 — #134 LocationPlatform (3 workers)

Add location-related MCP tool references (place search, directions, geocoding). Add `address`, `distance`, `travelTime` fields where appropriate.

- [ ] `TravelPlanCaseHub` — destination-research prompt
- [ ] `AppointmentCycleCaseHub` — find-alternative prompt
- [ ] `HomeMaintenanceCaseHub` — get-quotes prompt
- [ ] Response schemas: `DestinationResearchResult`, `FindAlternativeResult`, `GetQuotesResult`
- [ ] Tests: update canned responses

## Sentinels (cross-cutting)

Check whether any sentinel prompts in `LifeHeartbeatJob` should gain new tool references (e.g. contractor sentinel → contacts, booking sentinel → location).

- [ ] Review 7 sentinel prompts for tool relevance
- [ ] Update sentinel response schema records if needed

## Verification

- [ ] `mvn install -pl api` — api compiles
- [ ] `mvn test -pl app -am` — all tests pass
- [ ] No `@Disabled` annotations introduced
- [ ] Each issue referenced in commits (`Refs #131`, etc.)
