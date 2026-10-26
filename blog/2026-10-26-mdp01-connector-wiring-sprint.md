---
layout: post
title: "Four connectors wired in one session — the pattern that made it possible"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/life]
tags: [connectors, mcp-tools, workers, platform-spi]
---

# Four connectors wired in one session — the pattern that made it possible

casehub-life has had platform connector SPIs sitting unused since the skill
integration work in Layer 7. ContactsPlatform, EmailPlatform, DocumentPlatform,
LocationPlatform — each with production implementations (Google Contacts,
Gmail, Google Drive, Google Maps) — but no workers actually calling them.
The workers referenced `send_chat` and `calendar_create_event` and stopped
there.

I wanted to see how fast the remaining connectors could be wired in, given
the architecture we'd built. The answer: four in a single session, touching
18 workers across 6 CaseHubs.

## The pattern

Every connector wiring follows the same three-step change:

1. **System prompt** — add the MCP tool name and a usage instruction to
   the worker's prompt string. "Use search_contacts to look up the
   contractor's contact details."

2. **Response schema** — add a nullable ID field to the worker's result
   record. `String contactId`, `String emailMessageId`, `String documentId`,
   `String placeId` — following the existing `calendarEventId` and
   `notificationMessageId` convention.

3. **Test factory** — update the canned JSON response to include the new
   field and the tool name in `toolsUsed`.

That's it. No Maven dependencies. No CDI beans. No new services. No YAML
case definition changes. The MCP tool bridge means the worker's system
prompt is the entire integration surface — the agent calls the tool at
runtime through the platform's domain model system, and the connector
SPI implementation handles the rest.

![Connector wiring pattern — workers reference MCP tools, which bridge to platform SPIs](images/connector-wiring-pattern.svg)

## What went in

| Issue | Connector | Tool references | Workers |
|-------|-----------|----------------|---------|
| #131 | ContactsPlatform | `search_contacts`, `list_contacts` | 7 — request-quote, job-monitoring, book-appointment, find-alternative, needs-assessment, care-plan, get-quotes |
| #132 | EmailPlatform | `send_email` | 4 — pre-visit-prep, produce-report, confirmation, record-completion |
| #133 | DocumentPlatform | `sync_documents` | 4 — produce-report, care-plan, confirmation, record-completion |
| #134 | LocationPlatform | `search_places`, `get_directions` | 3 — destination-research, find-alternative, get-quotes |

Some workers now stack multiple connector tools. The `confirmation` worker
in TravelPlanCaseHub references `calendar_create_event`, `sync_documents`,
`send_email`, and `send_chat` — four platform SPIs in a single worker
prompt. The agent decides which to use based on the content: formal
itineraries go to email and document storage, quick confirmations go to chat.

## Two paths to the same connectors

The MCP tool calls are the direct path — one agent, one tool invocation,
one result. But the same connector operations are also exposed over
GraphQL through `ConnectorContactsApi` and its siblings, each annotated
with `@McpDomain`. That means playbooks can batch the same operations
that workers call individually.

A worker calling `search_contacts` is making a single lookup mid-task.
A playbook can compose `searchContacts` + `getContact` + `createContact`
into a batch sequence over GraphQL — sync a contractor's details from
Google Contacts, check if they exist locally, create them if not — without
an LLM in the loop. Same SPI, same operations, different execution model.

This matters most for project-oriented work — the kind where a case spans
multiple domains and needs coordination across them. Think about a home
renovation case: it touches contractors (ContactsPlatform), scheduling
(CalendarPlatform), documents (DocumentPlatform for invoices and permits),
location (LocationPlatform for site details), and finance (BankPlatform
for payments). A naive approach gives each worker its own MCP calls and
lets the LLM figure out the coordination. But a lot of that coordination
is mechanical.

A playbook can do the deterministic groundwork up front: pull the
contractor's contact details, fetch their recent invoices from document
storage, check the calendar for scheduling conflicts, geocode the site
address — all batched over GraphQL, no LLM tokens burned. Scripts can
then combine and index those results into a structured context package.
When the LLM finally gets involved, it's not spending its reasoning
budget on lookups. It gets a pre-assembled view of the situation and
focuses on the judgment calls — is this quote reasonable given the
contractor's history, should we escalate the timeline, does the
household admin need to approve this spend.

This is the shape we need to make sure works: playbooks as mechanical
pre-stage, scripts as combiners and indexers, LLM as the judgment layer
that operates on prepared context. The connector SPIs being accessible
through both paths — direct MCP for in-flight agent decisions, GraphQL
batches for orchestrated pre-staging — is what makes this possible
without duplicating the integration work.

## Why prompt-level, not compile-time

When we built the skill integration in Layer 7, the choice was between
compile-time dependency on each connector SPI (import `Contact.class`,
inject `ContactsPlatform`, call methods directly) or prompt-level tool
references (tell the agent which MCP tools exist, let it call them at
runtime).

We went with prompt-level. The workers don't know what a `ContactsPlatform`
is. They know that `search_contacts` exists and returns contact details.
The MCP domain bridge in connectors handles the mapping. This means wiring
a new connector is a string edit, not a dependency graph change.

The trade-off: life can't do compile-time validation of connector
interactions. If `search_contacts` changes its response shape, no
compiler will catch it — the agent's structured output schema is the
contract, and mismatches surface at runtime. For a personal automation
harness where the connectors are stable platform SPIs, that's an
acceptable trade. For a high-throughput system with frequent SPI changes,
it wouldn't be.

## What's left

#135 (CommercePlatform) is the last connector sibling, but it breaks the
pattern. The issue explicitly says "this isn't just wiring into existing
workers" — it needs new case definitions for procurement and household
supply reordering. That's a design session, not a wiring sprint.

#136 (BankPlatform PISP — payment initiation) is also different: it touches
the contractor payment worker specifically, where the financial
accountability properties matter. Payment initiation through an open
banking API is a consequential action that needs the oversight gate and
risk classification to be involved. Not a prompt edit.

The easy wins are done. What remains is genuine feature work.
