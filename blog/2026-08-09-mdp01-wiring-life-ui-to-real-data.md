---
layout: post
title: "Wiring Life UI to Real Data"
date: 2026-08-09
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [life-ui, blocks-ui, ledger, cases-view, data-wiring]
---

The Life UI scaffolding from the previous session gave us an app shell, a dashboard, and an inbox — all rendering blocks-ui components but mostly against placeholder data. This session was about making it real: backend endpoints that serve actual domain data, and frontend views that consume them.

The first decision was the most interesting. Issue #86 asked for a `LifeLedgerResource` wrapping `LedgerEntryRepository` — a thin REST resource exposing ledger queries. I went looking at what the platform already provides and found `casehub-ledger-rest`, a separate module in the ledger repo that ships the exact endpoints `<blocks-audit-trail-viewer>` expects: `/api/v1/ledger/entries`, attestations, Merkle verification. Same URL paths, same response shape. Adding the Maven dependency, a Jandex index entry, and an HTTP auth policy gave us the complete ledger API with zero life-specific Java code.

That discovery set the tone. Platform coherence isn't just a rule — it's a genuine time saver when you check before building.

The case-scoped tasks endpoint was straightforward: `GET /life-cases/{id}/tasks` queries WorkItems by `callerRef` prefix matching on the engine case ID. The pattern was already established in `LifeChannelContextProvider` — I just needed a REST surface for it. Added visibility filtering so juniors only see tasks where their group appears in `candidateGroups`.

With the backend ready, we built `cases-view.ts` — a split-pane layout with a case list on the left and a 6-tab detail panel on the right. Overview, Tasks, Audit, Routing, CBR, Commitments. The Audit tab wires `<blocks-audit-trail-viewer>` to the case's `engineCaseId` as `subjectId`. Tasks fetches from the new endpoint. Routing and CBR render their respective blocks-ui components pointed at endpoints that don't exist yet — they'll show empty states until we build those backends.

The `actionType` field on `PendingActionResponse` was a clean addition: derive from `LifeCommitmentRecord.mode` — OVERSIGHT becomes `OVERSIGHT_GATE`, DELEGATION stays `DELEGATION`, everything else is `WORK_ITEM`. Three tests, one switch expression. The inbox frontend doesn't discriminate on it yet, but the data is there.

One cross-repo fix: the `<blocks-audit-trail-viewer>` reads `entry.payload` but `casehub-ledger-rest` serialises the field as `metadata`. Entry expansion showed null. A one-field rename in the ledger DTO fixed it — the underlying `LedgerEntry.metadata` JPA field is unchanged, only the JSON wire name changes.

The remaining gaps are the harder pieces. The Routing and CBR tabs need backend endpoints that either read from the engine's ephemeral case context or re-query the CBR store — neither is a simple database query. The Channels tab needs `MessageStore` exposed as REST. The Inbox needs frontend actionType discrimination that may require blocks-ui component changes. I filed epic #95 with 7 child issues covering all of it, and added `blocks-ui` and `ledger` to the slot for cross-repo work.
