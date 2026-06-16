---
layout: post
title: "What selected-alternatives doesn't tell you"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [quarkus, cdi, ci, snapshots]
---

CI was red — not from a push, but from a `repository_dispatch`. A casehub-qhorus SNAPSHOT had published and triggered a rebuild on `casehubio/life`. Twenty-seven identical errors: "Ambiguous dependencies for type `CurrentPrincipal`."

Three beans were competing: `QhorusInboundCurrentPrincipal` (new in the SNAPSHOT), `TenantScopedPrincipal` from casehub-work, and `DefaultTestPrincipal` from casehub-engine-persistence-memory. The obvious fix was `quarkus.arc.selected-alternatives`. I added `DefaultTestPrincipal` to the list. Built again. Still 27 errors.

Claude decompiled the class. The bytecode showed `@ApplicationScoped` — no `@Alternative` annotation. That's the issue: `selected-alternatives` only acts on beans marked `@Alternative`. A plain `@Default` bean is already active and needs no selection; Quarkus accepts the config entry silently, does nothing, and the ambiguity stands. Nothing in the docs or the error output hints at this. We went through the bytecode of all three competing beans before the picture was clear.

The actual fix is `quarkus.arc.exclude-types`. Exclude the beans you don't want; the one you want wins by surviving. Production config excludes `QhorusInboundCurrentPrincipal` and `DefaultTestPrincipal`; tests exclude `QhorusInboundCurrentPrincipal` and `TenantScopedPrincipal` — leaving `DefaultTestPrincipal` as the test winner. It matters because `DefaultTestPrincipal.tenancyId()` returns a fixed UUID that every test fixture depends on.

There was a second failure behind the first. The casehub-ledger SNAPSHOT had added a build-time validator: any `LedgerEntry` subclass with persistent fields must override `domainContentBytes()`, or the build refuses. We had four subclasses missing it. The pattern comes from `CaseLedgerEntry` in casehub-engine-ledger: join all domain fields with a `|` separator, return as UTF-8 bytes. The bytes feed into the Merkle chain. Four overrides later, the build was clean.

There was also a wrong-remote confusion. This project has `origin` (my fork at `mdproctor/life`) and `upstream` (the org repo at `casehubio/life`). The `repository_dispatch` CI runs against `casehubio/life`. A push to `origin` does nothing for CI — and the headSha in the failing run pointed to commits that didn't exist locally, because they were on the org's main, not the fork's. `git fetch upstream` found them immediately.

`selected-alternatives` silently accepting non-`@Alternative` beans is the kind of thing you only discover by decompiling — there's no error, no warning, and the config looks correct until you check the annotation.
