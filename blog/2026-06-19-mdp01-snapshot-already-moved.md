---
layout: post
title: "The snapshot had already moved"
date: 2026-06-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [testing, engine, snapshots]
---

The @QuarkusTest suite for casehub-life is fully green. 286 tests run, 0 fail,
0 skip — including all eight integration test classes that were blocked while
engine#536 and engine#537 were open.

life#36 was on the backlog: `LifeCaseResourceTest` failing with a
`NoSuchMethodError` on `CaseHubRuntime.signal(UUID, String, Object)`. The issue
was filed because the engine SNAPSHOT had changed the method signature. The
obvious next step was to update the call site in `LifeCaseService`.

I brought Claude in to work through it. Before writing anything, I had Claude
check the installed SNAPSHOT jar directly with `javap`. The method was there —
same signature `LifeCaseService` was already using. We ran the test. It passed
first time, no changes needed. The SNAPSHOT had moved again between when the
issue was filed and when we picked it up.

This happens with SNAPSHOT dependencies. The artifact in your local Maven cache
reflects the state at the last successful update, not the current one. Between
filing an issue and picking it up, the upstream may already have changed. The
right first move for a `NoSuchMethodError` against a SNAPSHOT is to check the jar
before writing any fix code.

We cleaned up the aftermath: removed the "@QuarkusTest status (2026-06-18)"
warning row from CLAUDE.md, and updated ARC42STORIES.MD's improvement refs to
mark both engine issues resolved. The status note had been accurate — it
documented a real block — but once the upstream fixes landed it was just
misleading noise waiting to ambush someone reading the file cold.
