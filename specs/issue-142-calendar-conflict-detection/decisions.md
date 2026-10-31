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

**Choice:** Store pushed field values — add `pushedSummary`, `pushedStartTime`, `pushedEndTime` columns to `JpaLifeCalendarMapping`. When sync returns a changed event, compare returned values against stored pushed values. If different, it's an external edit.
**Alternatives:**
- Hash of last-pushed state — compact (single column) but vulnerable to false positives from provider normalization (whitespace trimming, timezone canonicalization, duration rounding); cannot show field-level diff
- ETag/version from provider — zero false positives but provider-dependent; not all providers support it
**Rationale:** Stored field values avoid hash collision and normalization false positives. Comparison uses the values as returned by the provider (not as computed before push), eliminating provider normalization artifacts. Enables field-level diff at query time ("start changed from Tue to Thu"). The mapping table has one row per event — the additional storage is negligible.
**Trade-offs:** Three additional columns on the mapping table (V116 migration). Slightly more complex comparison logic than a single hash comparison.
**Sources:** LifeCalendarMapping schema (V115), CalendarPlatform SyncResult, R1-03 decision review finding (provider normalization false positives)
**Exploration:** quick → revised after decision review
**Status:** revised

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

## D10: Conflict data model

**Choice:** New JPA entity `LifeCalendarConflict` with its own lifecycle, behind a `LifeCalendarConflictStore` SPI in api/
**Alternatives:**
- Status field on LifeCalendarMapping — conflates mapping lifecycle with conflict lifecycle; a mapping can have zero or many conflicts over its lifetime
- Transient object — no persistence means no foreign key for ledger entries, no queryable conflict history, and conflicts lost on restart
**Rationale:** A conflict has its own identity (detection time, conflict type, external state snapshot, resolution), its own lifecycle (DETECTED → ACCEPTED/DISMISSED), and is referenced by ledger entries. This warrants a dedicated entity. Fields: `{id, mappingId, conflictType (MODIFIED/DELETED), detectedAt, externalSummary, externalStartTime, externalEndTime, status (DETECTED/ACCEPTED/DISMISSED), resolvedAt, resolvedBy}`. SPI in api/ follows LifeCalendarMappingStore pattern.
**Trade-offs:** Additional entity, migration (V117), and SPI. Justified by the conflict lifecycle requirements and audit trail foreign key.
**Depends on:** D5 (stored field values enable field-level diff in conflict record)
**Sources:** R1-05 decision review finding, LifeCalendarMappingStore pattern
**Exploration:** quick (gap identified by decision review)
**Status:** captured

## D11: Single-calendar scope

**Choice:** Single calendar (default platformId + calendarId) for this issue. Data model supports multi-calendar (LifeCalendarSyncState is per-platformId/calendarId pair) but the sync pipeline and UI assume one calendar.
**Alternatives:**
- Multi-calendar now — support multiple calendars from the start; adds configuration UI, per-calendar sync scheduling, and domain→calendar routing. Significantly larger scope.
**Rationale:** The existing infrastructure from #114 is single-calendar. Extending the data model to be multi-calendar-ready (D6 already is) while keeping the pipeline single-calendar avoids premature complexity. Multi-calendar can be added as a follow-up.
**Trade-offs:** Users with domain-specific calendars (health on one, family on another) must consolidate to a single calendar for now.
**Sources:** R1-10 decision review finding, LifeCalendarSyncService config keys, CalendarPlatform.listEventsSync(calendarId)
**Exploration:** quick (implicit assumption made explicit by decision review)
**Status:** captured
