---
layout: post
title: "Wrong cause, right fix"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [testing, h2, flyway, hibernate, architecture]
---

The handover said eight test classes were failing because `ledger_subject_sequence` was missing from the H2 schema. That was plausible — the table is created by a casehub-ledger Flyway migration that only runs in production. Tests use `drop-and-create` with Flyway disabled, so the table would never appear. We created `import-qhorus.sql` to seed it via `quarkus.hibernate-orm."qhorus".sql-load-script`, wired it into the test config, and ran the suite.

Still 83 failures. Same error pattern throughout: `NULL not allowed for column "TENANCY_ID"`. And `LEDGER_SUBJECT_SEQUENCE` appeared nowhere in any surefire report — the handover had named the wrong cause.

The real failure was in `LifeTestFixtures`. casehub-work V35 had added `tenancy_id NOT NULL DEFAULT '278776f9...'` to `work_item_template` — a platform-wide tenancy column. In production the DEFAULT handles existing rows. In tests, Hibernate generates the column from JPA annotations without the DEFAULT clause, so any insert without an explicit value hits the constraint immediately. The fixture created `WorkItemTemplate` objects without ever setting `tenancyId`. One line fixed it.

The `sql-load-script` fix stayed in — `ledger_subject_sequence` was genuinely missing from H2, just not yet causing failures on active code paths. Both go in together.

The more interesting thing from today was rethinking life#25. It asks for migrating all eight CaseHub worker classes from `Worker.builder().function(lambda)` to `FuncWorkflowBuilder`. I'd been treating this as a straightforward compliance gap — protocol says FuncWorkflowBuilder, workers use raw lambdas, migrate them. But looking at the workers, they return hardcoded maps. Wrapping a single `WorkerResult.of(Map.of(...))` inside a serverless workflow DSL is using a workflow runtime to express "call this function once." That's the wrong tool.

The protocol rationale is sound though. A declared workflow step gives the engine something to inspect — task type, retry policy, risk classification surface. A raw lambda is a black box. The gap is that there's no middle path: no first-class function-worker concept that gives you engine integration without workflow ceremony. Raw lambda gives nothing; FuncWorkflowBuilder single-step gives everything but with the wrong abstraction.

I filed engine#463 to open that design question without foreclosing it. FunctionWorker as a dedicated type, FuncWorkflowBuilder used correctly for genuinely single-step cases, or something else — that needs a brainstorm before casehub-life makes any changes. Life#25 is rescoped to "apply the settled abstraction to first real OpenClaw workers" and blocks on engine#463.

The handover misattribution is worth noting. The symptom was right — eight classes failing — but the cause was guessed rather than verified. A handover that says "probable cause: X" is more honest than one that asserts it.
