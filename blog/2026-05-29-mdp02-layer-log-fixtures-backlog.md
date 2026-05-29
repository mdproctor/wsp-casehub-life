---
layout: post
title: "Mapping slices and fixing the seeding tax"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [casehub, layer-log, vertical-slices, test-fixtures, quarkus]
---

Before starting Layer 4 — casehub-ledger integration — I wanted to close three
backlog items that had been accumulating since Layer 3. None were blocking, but
they were the kind of drift that compounds: inconsistent documentation, seeding
code copied six times across test classes, no slice plan for a project now three
layers deep.

## Navigation headers, finally consistent

LAYER-LOG.md had two separate conventions for navigation metadata. Layer 1 had
`**Issue:**` and `**Navigation:**` inline in the header block. Layers 2 and 3 had
`### Navigation` subsections buried at the bottom of each entry. All seven layers
are now standardised on the header format, and the stale Layer 3 "Open follow-up"
section — still listing issues #16, #17, and #18 from last session, all closed —
got removed.

Trivial individually. Collectively they make the log navigable.

## Defining the vertical slices

The SIAL protocol requires a Vertical Slice Index at the top of LAYER-LOG.md —
user-visible capabilities, sequenced by dependency. I'd been building layers without
one, which meant the log told you *how* each integration was done but not *what* the
system could actually do at each milestone.

Six slices for casehub-life:

| Slice | Capability |
|-------|-----------|
| S1 | Tasks with SLA deadlines; household-admin escalated on breach (Layers 1–2) |
| S2 | Formal COMMAND/RESPONSE commitments with Watchdog follow-up (+ Layer 3) |
| S3 | Tamper-evident Merkle audit + GDPR Art.17 erasure (+ Layer 4) |
| S4 | Multi-step CasePlanModel workflows (+ Layer 5) |
| S5 | Trust-weighted agent routing (+ Layer 6) |
| S6 | OpenClaw skill integration (+ Layer 7) |

Writing the capability description for S2 forced me to be precise about something
I'd been glossing over. Three commitment modes — family delegation, contractor
follow-up, oversight gate — look similar in the docs but have genuinely different
mechanics. Delegation requires acknowledged RESPONSE. Contractor gets Watchdog
follow-up if no response arrives. Oversight gates are different: no WorkItem exists
until the household-admin RESPONSE arrives. There is nothing to block — creation is
deferred, not held up.

Claude caught this during code review. I'd written "financial-oversight gates block
WorkItem creation until household-admin approves." That's wrong — "block" implies
something in flight. The correct description is "defer WorkItem creation — no task
exists until RESPONSE is received." One word change, but it shifts how you'd
implement the pattern.

The arch pattern column for pending slices (S3–S6) presented a similar question.
I wanted to label S3 as `+ Interceptor` since casehub-ledger uses
`ProvenanceCaptureInterceptor` internally. But the pattern the *consumer* demonstrates
is calling a port — Hexagonal — not implementing the interceptor. The interceptor is a
ledger-internal concern. The right pattern label for S3 will be clear once Layer 4 is
built. For now the cell shows `🔲`. Honest placeholder.

## The seeding tax

Six `@QuarkusTest` classes all had their own `seedTemplates()` + `seedIfAbsent()`
implementations. Same four templates, same field values, five different UUID ranges,
and a priority discrepancy: `life-escalation` was MEDIUM in five classes and HIGH in
one.

We extracted it into `LifeTestFixtures` — a final utility class with two static
methods: `seedStandardTemplates()` and `seedEscalationTemplate()`. Static methods
inherit the active JTA transaction from `@BeforeEach @Transactional`, so no CDI
injection is needed. Canonical UUIDs 001–004. Idempotency guard by template name.
`life-escalation` standardised to HIGH, which is what the escalation semantic actually
requires.

During review Claude flagged the `t.createdAt = Instant.now()` line in the fixture
helper — cargo code we'd been copying across three layers. `WorkItemTemplate` has
a `@PrePersist` method that sets `createdAt` (and `id`, if null) automatically. Every
fixture had been setting `createdAt` explicitly for nothing. The only reason to set
`id` explicitly is to get deterministic UUIDs for test analysis — which we still do.
The `createdAt` line was just noise from the first fixture ever written, faithfully
copied ever since.

61 tests pass. The CLAUDE.md testing table now references `LifeTestFixtures` instead
of the old per-class `seedIfAbsent` pattern.
