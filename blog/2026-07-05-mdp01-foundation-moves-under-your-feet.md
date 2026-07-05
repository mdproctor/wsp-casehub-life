---
layout: post
title: "The Foundation Moves Under Your Feet"
date: 2026-07-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [gdpr, jurisdiction, cdi, snapshot-migration]
series: issue-48-jurisdiction-gdpr-refine
---

*Part of a series. Previous: [Duplication Points Upward](2026-06-30-mdp01-duplication-points-upward.md).*

## When the Ground Shifts

I'd planned a clean run: four issues, one branch, straightforward implementations. Per-action jurisdiction on legal ledger entries, a switch elimination, GDPR erasure service extraction, compliance report wiring. None of these were architecturally novel — they were refinements to patterns already established in Layers 4–7.

What I hadn't planned for was the foundation moving while we worked.

Three upstream SNAPSHOTs had shipped since the last session. Engine-api removed `ListEvaluator` and replaced it with `CandidateSetSpec` — a sealed interface with `Inline` and `Named` variants. Qhorus converted `Channel`, `Watchdog`, and all store types from JPA entities to records, moving them from `runtime` to `api` packages. Casehub-work added a NOT NULL `tenancyId` column to `WorkItemTemplate`.

None of these were documented. None had migration guides. We discovered each one when the compiler told us.

## The CDI Ambiguity Trap

The worst was a 35-error CDI ambiguity between `casehub-qhorus-testing` and `casehub-qhorus-persistence-memory`. Both jars publish identical `@Alternative @Priority(1)` store beans — `InMemoryChannelStore`, `InMemoryMessageStore`, and twelve others. CDI can't disambiguate same-priority alternatives.

The natural fix — `quarkus.arc.exclude-types` for one package — doesn't work. The reactive store wrappers inject the blocking stores by concrete class, not by interface. Exclude the blocking stores, the reactive wrappers fail with `UnsatisfiedResolutionException`. Exclude the other package's stores, the same thing happens from that side.

The actual fix was one Maven `<exclusion>` line: exclude `casehub-qhorus-persistence-memory` from the `casehub-qhorus-testing` dependency. The testing jar's stores are self-contained. The persistence-memory jar was the redundant one.

## The Design Review Earned Its Keep

We ran a two-round adversarial design review before implementing. It caught three things I'd have shipped wrong:

The `ExternalActorErasureLedgerEntry` didn't include `ledgerEntriesAffected` in `domainContentBytes()`. Without it, the erasure proof isn't self-contained in the Merkle chain — an auditor would need to cross-reference the foundation's `ErasureReceiptLedgerEntry` to know how many entries were tokenised. Adding it means changing the hash function, which is a chain-breaking operation. The review forced us to add a migration precondition guard: assert zero existing rows before allowing the schema change.

The implementation order was wrong. The spec said `#48 → #50 → #51 → #49`, but #50's code uses `LifeGdprErasureService` which is created in #49. Corrected to `#48 → #51 → #49 → #50`.

The jurisdiction column was 10 characters on `LifeTaskContext` but 100 on `LegalActionLedgerEntry` — same conceptual value, different column widths. Aligned both to 10 with ISO 3166-1/2 validation at the API boundary.

## What Actually Shipped

The jurisdiction flow is clean: optional field on `CreateLifeTaskRequest`, persisted to `LifeTaskContext`, `LegalDomainLedgerHandler` prefers it over the tenant-wide config default. A UK resident filing US immigration paperwork now gets `jurisdiction=US` on the legal ledger entry, not `GB`.

`LifeCaseService.resolve()` lost its 6-arm switch. CDI `Instance<LifeTypedCaseHub>` with `lifeCaseType()` matching — adding a new case type is now one class, zero service changes.

`LifeGdprErasureService` extracts the full GDPR pipeline: PII nullification → memory erasure → `LedgerErasureService.erase()` for actor ID tokenisation → erasure ledger entry. The erasure endpoint returns `ErasureResponse` with `ledgerEntriesAffected` and `tokenisationEnabled` — the compliance auditor sees everything in one response.

The upstream migration work consumed roughly half the session. The feature work, once the foundation stabilised, was mechanical.
