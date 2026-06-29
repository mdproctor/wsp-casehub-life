---
layout: post
title: "The Memory Gap in GDPR Erasure"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [gdpr, erasure, casememorystore, testing]
---

# CaseHub Life — The Memory Gap in GDPR Erasure

**Date:** 2026-06-29
**Type:** phase-update

---

## What I was trying to achieve: wire CaseMemoryStore into the existing GDPR erasure flow

casehub-life's `DELETE /external-actors/{id}/personal-data` endpoint already nullifies PII and writes a tamper-evident ledger entry. But it doesn't touch `CaseMemoryStore` — any memory records referencing the erased actor survive the erasure. devtown and clinical both integrate `eraseEntity()` alongside their erasure flows. casehub-life should do the same.

## What we believed going in: issue #34 was mostly done

The issue was filed before Layer 4 landed. When I pulled it up, the description said "No erasure endpoint exists yet" — but it does, and has since Layer 4. The real gap was narrower than the issue described: a missing `CaseMemoryStore.eraseEntity()` call and no `erasedBy` identity flow from the authenticated principal.

There's a further wrinkle: casehub-life has no memory adapter on the classpath. `NoOpCaseMemoryStore` is active, returning 0 for everything. The call is a guarded no-op today — but when OpenClaw agents start storing case context, the erasure path will already be wired.

## The design review that sharpened three edges

The adversarial design review ran three rounds and surfaced 10 issues. Three shaped the final implementation:

**The Merkle hash question.** Should `memoryRecordsErased` be included in `domainContentBytes()`? Adding it would break hash verification of existing entries whose digests were computed without the field. The reviewer caught this — it's operational metadata, not compliance-critical content. The authoritative erasure proof is the entry's existence and its existing fields. The new field stays out of the hash.

**The transaction boundary.** `ExternalActorService.erase()` is `@Transactional`. PII nullification and the ledger write participate in JTA. But `CaseMemoryStore.eraseEntity()` for external backends (Mem0, Graphiti, SQLite) does NOT. If memory erasure succeeds but JTA rolls back, memory records are gone with no ledger proof. This limitation is shared with devtown and clinical — inherent in distributed memory stores, documented as a known gap to address when a real adapter arrives.

**The `erasedBy` identity flow.** The old code hardcoded `"household-admin"` as the eraser identity. The review pushed for `CurrentPrincipal.actorId()` at the resource layer, with the service accepting it as a parameter — keeping auth logic out of the service layer per the auth-retrofit-readiness protocol.

## The Panache mockStatic trap

The one genuinely non-obvious discovery: Mockito's `mockStatic(ExternalActor.class)` cannot intercept `findByIdOptional()`. The method is inherited from `PanacheEntityBase` — Java compiles the call as `PanacheEntityBase.findByIdOptional()`, and Mockito only intercepts methods whose declaring class matches the mock. The error is misleading: `MissingMethodInvocationException` suggesting the call isn't on a mock, when the real issue is bytecode dispatch to the parent class.

The fix: add `quarkus-junit5-mockito` and use `@InjectMock` with `@QuarkusTest` instead of pure Mockito. This went into the garden as GE-20260629-74fc65 — anyone testing Panache Active Record services will hit the same wall.

## Where this leaves us

Nine files changed, four new tests, one Flyway migration. The erasure flow now calls `memoryStore.eraseEntity()` with a narrow `MemoryCapabilityException` catch — real I/O failures propagate and roll back the transaction. `memoryRecordsErased` is persisted on the ledger entry for the audit trail. Today that count is always 0; when a memory adapter arrives with life#8, it won't be.
