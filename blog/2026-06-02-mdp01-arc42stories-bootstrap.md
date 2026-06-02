---
layout: post
title: "ARC42STORIES.MD: Five Layers into One Document"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [casehub, arc42stories, documentation, migration]
---

casehub-life now has an ARC42STORIES.MD — 1175 lines covering §1 through §13 plus a self-assessment. The document absorbs what was previously spread across LAYER-LOG.md (layer entries), the workspace DESIGN.md (cross-cutting decisions), and the vertical slice index (chapter sequencing). One file, one architecture record.

The prerequisite that made this possible was completing the Layer 5 LAYER-LOG entry. It had been a stub since the engine branch closed — all placeholder fields unfilled. The design spec, blog entry, and git log provided enough to fill in all 18 elements: 8 YamlCaseHub subclasses, 8 DSL companions, the scope retrofit, engine memory @Alternative wiring, and three gotchas including the Surefire retry interaction with `assumeTrue()` that masks real test failures.

The migration itself was straightforward but tedious. Five complete layer entries, each with 18 elements, each section written to a declared mode from the arc42stories spec's mode map. "What it adds" sections use explanation/comparative mode — Before:/After: contrast, no personal voice, hard length cap. Gotchas use how-to/diagnostic mode — Symptom/Cause/Fix labels, exact actions not directions. "Pattern to replicate" uses tutorial mode — numbered imperative steps, domain-agnostic language. The mode map is the discipline; without it every section drifts toward the same prose texture.

Claude ran two parallel verification agents before any content was written — one inventoried all Layer 5 classes (confirming 8 YamlCaseHub subclasses, not the 7 the design spec header claims), the other verified every class from Layers 1–4 still exists or was correctly removed. The 8th CaseHub is `CareEpisodeCaseHub` — a child case for care-coordination that the design spec describes in prose but doesn't count in its header. CLAUDE.md had the right number all along.

The quality gate from the issue specified 9 checks. The one worth noting: the anti-slop scan found zero banned words across 1175 lines. Mode-first generation prevents them — when each section is written to its declared mode's structural constraints, the AI-sounding filler words never appear because the mode doesn't call for the prose patterns that produce them. The anti-slop rules are a safety net, not the mechanism.

§8 Anti-patterns captures the four failure modes most likely when extending casehub-life: wrapper entities duplicating foundation primitives (the Layer 1 mistake that Layer 2 corrected), multi-PU prefix matching (ledger entity package placement), engine memory @Alternative beans silently missing, and CDI observer firing before domain supplement persisted. Each in Symptom/Cause/Fix format — actionable from a scan.

Two gaps remain visible: no dedicated blog entries for Layers 2 and 4 (noted in §9.3 and §12), and engine#410 blocking integration tests (still open, no comments). LAYER-LOG.md stays as the source-of-truth draft — not deleted, just marked as migrated.
