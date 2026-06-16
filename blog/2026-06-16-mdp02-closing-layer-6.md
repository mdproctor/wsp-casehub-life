---
layout: post
title: "Closing Layer 6 — or: the bugs you find when you actually look"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [casehub-ledger, cdi, quarkus, trust-routing, layer-6, testing]
---

Layer 6 — trust routing — has been sitting at "implementation complete" for a couple of sessions with the GitHub issue still open and the ARC42STORIES.MD §9.4 entry half-written. Today I closed it. But before signing off I wanted to actually validate the code rather than just tick the box.

Good thing I did.

I asked Claude to run the Layer 6 tests — `LifeOutcomeAttestationWriterTest`, `LifeTrustRoutingPolicyProviderTest`, `ExternalActorTrustEnrichmentTest`. The first two passed. The third didn't.

```
Expected: not null
  Actual: null

at ExternalActorTrustEnrichmentTest.getExternalActorIncludesTrustProfile(line 32)
```

`trustProfile.globalScore` was null even though the test had seeded a trust score via `trustScoreRepo.upsert()`. The upsert returned normally. No exception. The score just wasn't there.

The immediate suspect was a transaction boundary — maybe the write wasn't committed before the REST call. But the test helper is `@Transactional` and returns before the GET fires. That wasn't it.

The real cause was that `trustScoreRepo` was injecting `NoOpActorTrustScoreRepository` — the `@Default` bean — not `JpaActorTrustScoreRepository`, which is `@Alternative`. Every `upsert()` call was silently doing nothing. Every `currentScore()` call was returning empty. I'd never added the JPA repository to `quarkus.arc.selected-alternatives`.

This is the exact same pattern as `JpaLedgerEntryRepository`, which I'd documented and fixed months ago. The trust score repository is separate and requires its own entry. The casehub-ledger library ships with no-op defaults for everything that requires a JPA datasource — you have to explicitly activate the JPA implementations or you're writing to `/dev/null` and getting no error in return.

The fix was one line in `application.properties`. But it also means the test had been passing for the wrong reason since Layer 6 was first written — the `ExternalActorTrustEnrichmentTest` was asserting on a response that genuinely had no trust data in it, even though I thought I'd seeded some. A false pass.

After that fix, the full test suite turned up another failure — this one HTTP 500 across most integration tests:

```
SQLGrammarException: Column "TENANCY_ID" not found
at LedgerSequenceAllocator.h2NextSequenceNumber()
```

The ledger SNAPSHOT had added `tenancy_id` to `ledger_subject_sequence` and changed the primary key to a composite. The table is plain SQL — not a JPA entity — so Hibernate's `drop-and-create` doesn't touch it. It's created by `import-qhorus.sql`, which still had the old single-column schema. Production was fine because Flyway handles the migration; tests were broken because the SQL file was stale.

Updated `import-qhorus.sql` to match. All the Layer 6 tests pass. Most of the rest of the suite passes too, except for `LifeCaseResourceTest`, which is failing with `NoSuchMethodError` on `CaseHubRuntime.signal()` — the engine SNAPSHOT changed the method signature at some point. Filed that as #36 and moved on.

The §9.4 entry is written, the issue is closed, and the two patterns are documented in CLAUDE.md. One thing worth noting: the `NoOpActorTrustScoreRepository` problem is not going away. Every time casehub-ledger ships a new SNAPSHOT with a new `@Alternative` JPA repository, there's a new silent no-op waiting to be discovered. The only real protection is an integration test that verifies the data actually lands in the database, not just that `upsert()` returns without throwing.
