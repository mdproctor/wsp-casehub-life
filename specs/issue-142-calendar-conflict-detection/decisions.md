## D1: Conflict resolution semantics

**Choice:** Detect + notify — external edits surface as conflicts; WorkItem deadline stays unchanged until a human explicitly updates it through CaseHub
**Alternatives:**
- Auto-reconcile inbound — automatically update WorkItem to match external edit; treats calendar as authoritative, undermines accountability model
- CaseHub-wins override — always push back CaseHub's version; ignores legitimate rescheduling
- Configurable per domain — different strategies per LifeDomain; adds complexity without clear benefit when detect+notify covers all cases
**Rationale:** CaseHub is an accountability system — deadlines and SLAs are the authoritative record. External calendar edits are signals, not commands. The human decides whether an external reschedule should change the WorkItem.
**Trade-offs:** User must explicitly resolve every conflict. Unresolved conflicts accumulate if ignored.
**Sources:** issue #142 body, LifeCalendarSyncService (outbound-only today), casehub-work SLA model
**Exploration:** quick
**Status:** captured

## D2: Sync trigger mechanism

**Choice:** Both scheduled polling (Quartz job at configurable interval) and on-demand manual refresh in the UI
**Alternatives:**
- Scheduled polling only — slight delay in detection, no way for user to force immediate check
- On-demand only — conflicts not detected until someone opens the calendar panel
**Rationale:** Background polling ensures timely detection for SLA-sensitive events. On-demand refresh gives users immediate feedback when they know something changed.
**Trade-offs:** Two trigger paths to maintain. Quartz job adds background load (mitigated by incremental sync tokens — only deltas fetched).
**Sources:** CalendarPlatform.listEventsSync() SPI, SyncRequest/SyncResult in casehub-connectors
**Exploration:** quick
**Status:** captured

## D3: Resolution actions

**Choice:** Accept or dismiss — two actions covering the two meaningful outcomes
**Alternatives:**
- Accept, dismiss, or defer — adds a 'defer' that leaves both sides diverged; risks forgotten conflicts
- Accept only — never push back to Google; user must manually fix calendar if they want CaseHub's version
**Rationale:** Accept updates the WorkItem deadline to match the external edit. Dismiss pushes CaseHub's deadline back to Google. These cover every resolution scenario without ambiguity.
**Trade-offs:** No 'defer' means the user must decide immediately (or leave the conflict unresolved, which is effectively deferring).
**Sources:** issue #142 scope
**Exploration:** quick
**Status:** captured

## D4: Audit trail

**Choice:** Tamper-evident ledger entry (CalendarConflictLedgerEntry) for each detected conflict and its resolution
**Alternatives:**
- Application log only — lighter weight but no tamper-evident trail; compliance-relevant events (health appointment rescheduling, legal deadline changes) would lack formal audit
**Rationale:** External calendar edits to health appointments or legal deadlines are compliance-relevant. The ledger provides tamper-evident proof that deadline changes were detected and acknowledged.
**Trade-offs:** New LedgerEntry subclass (JOINED inheritance, V2101+ migration). Additional write per conflict detection and resolution.
**Sources:** LifeDecisionLedgerObserver pattern, casehub-ledger JOINED inheritance, dual-trail-audit-pattern.md
**Exploration:** quick
**Status:** captured

## D5: Change detection mechanism

**Choice:** Hash of last-pushed state — store SHA-256 of (summary, startTime, endTime) that CaseHub last pushed to the calendar provider
**Alternatives:**
- Full snapshot of last-pushed fields — richer field-level diff but more storage and schema complexity
- ETag/version from provider — zero false positives but provider-dependent; not all providers support it
**Rationale:** Hash is compact (single column), provider-agnostic, and sufficient to distinguish "our update echoed back" from "someone else changed it". Field-level diff isn't needed — the conflict UI shows the current external state vs CaseHub state, which is derivable from the WorkItem.
**Trade-offs:** Cannot show "what specifically changed" from hash alone — but the current external state is visible in the sync result, and the CaseHub state is on the WorkItem, so a diff is constructable at display time.
**Sources:** LifeCalendarMapping schema (V115), CalendarPlatform SyncResult
**Exploration:** quick
**Status:** captured

## D6: Sync token storage

**Choice:** New JPA entity (LifeCalendarSyncState) behind an SPI (LifeCalendarSyncStateStore in api/)
**Alternatives:**
- Extend LifeCalendarMapping — conflates per-event mapping with per-calendar sync state (one sync token covers all events in a calendar, not individual events)
**Rationale:** Sync token is per-(platformId, calendarId), not per-event. Clean separation from the mapping table. SPI in api/ follows the same pattern as LifeCalendarMappingStore.
**Trade-offs:** Additional entity and migration (V116). Additional SPI interface.
**Sources:** CalendarPlatform.listEventsSync() SyncRequest.syncToken, LifeCalendarMappingStore pattern
**Exploration:** quick
**Status:** captured

## D7: UI surfacing

**Choice:** Conflict badge + inline indicator in the existing calendar panel — warning icon on conflicted events, conflict count badge on nav item, click to see diff and accept/dismiss actions
**Alternatives:**
- Dedicated conflicts view — more prominent but adds a new view to the SPA for a secondary concern
- Notification-only — conflicts via SSE/notification badge; no calendar panel changes; less discoverable
**Rationale:** Conflicts are calendar-context-specific — surfacing them inline where the user is already looking at calendar events is the most natural UX. Badge provides at-a-glance awareness.
**Trade-offs:** Calendar panel grows more complex. If many conflicts accumulate, inline display may become noisy.
**Sources:** life-calendar-panel (existing Lit component), LifeEventBroadcaster SSE pattern
**Exploration:** quick
**Status:** captured

## D8: External deletion handling

**Choice:** Treat as conflict — surface a "deleted externally" conflict with the same accept/dismiss flow
**Alternatives:**
- Auto-recreate silently — CaseHub recreates the event immediately; no conflict surfaced; may frustrate user who intentionally deleted
- Remove mapping silently — clean up mapping with no notification; user loses calendar link without knowing
**Rationale:** Deletion of a CaseHub-managed event is a significant action — it could be intentional (user moved to a different calendar app) or accidental. Treating it as a conflict ensures the user is aware and can decide. Accept = remove mapping (WorkItem keeps deadline). Dismiss = recreate event in Google.
**Trade-offs:** User sees a conflict for intentional deletions. Acceptable given the accountability model.
**Sources:** CalendarPlatform SyncResult.deletedIds, LifeCalendarOutboundObserver removal pattern
**Exploration:** quick
**Status:** captured

## D9: Pipeline architecture

**Choice:** Event-driven pipeline with separate components: SyncJob → ConflictDetector → CDI events → LedgerObserver
**Alternatives:**
- Inline in existing service — extend LifeCalendarSyncService with inbound sync, conflict state on LifeCalendarMapping; fewer classes but conflates outbound push with inbound sync and overloads the mapping entity
**Rationale:** Clean separation of concerns matches existing casehub-life patterns (observers, CDI events, dedicated services). Conflict lifecycle is independent from mapping lifecycle. Each component is testable in isolation.
**Trade-offs:** More classes than the inline approach. Justified by the complexity of the conflict state machine (detected → accepted/dismissed) and the audit requirements.
**Depends on:** D1 (detect+notify semantics), D6 (sync state storage)
**Sources:** LifeDecisionLedgerObserver pattern, LifeCalendarOutboundObserver pattern, CDI event patterns throughout casehub-life
**Exploration:** quick
**Status:** captured
