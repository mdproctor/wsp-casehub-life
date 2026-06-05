---
layout: post
title: "The Filter That Tested Nothing"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [testing, attestation, code-review]
---

# casehub-life — The Filter That Tested Nothing

**Date:** 2026-06-05
**Type:** phase-update

---

## What I was trying to achieve: close the Layer 6 code review backlog

Two minor issues sat in the queue from the Layer 6 trust routing code review — #22 (four findings from the review itself) and #24 (integration tests for the seven case definitions that only had definition-level tests). Neither was architecturally significant. Both were about making sure the test suite actually tests what it claims to test.

## The filter that tested nothing

Finding #1 in the code review turned out to be more interesting than "Minor" suggests. The `LifeOutcomeAttestationWriterTest` was filtering attestations with `.filter(a -> a.verdict != null)` to isolate verdict-only attestations from dimension attestations. The problem: every attestation has a non-null verdict. Verdict-only attestations set `verdict`. Dimension attestations set `verdict` AND `trustDimension`. The filter was a tautology — it matched everything and distinguished nothing.

The fix is `.filter(a -> a.trustDimension == null)`, which correctly selects attestations that carry a verdict but no dimension. The tests still passed after the change, which is exactly right — the underlying production code was correct. The filter just wasn't doing any filtering.

This is the same category of invisible logic error as the binary-else pattern that produced FLAGGED attestations from CREATED events (GE-20260603-753526). Both pass through values that look reasonable. Both are only caught by asking "what does this filter actually exclude?" rather than "does the test pass?"

## Seven integration tests, one shared helper

The case definition integration tests followed the pattern from `AppointmentCycleIntegrationTest` — start a case, poll with Awaitility until workers fire, complete humanTask WorkItems, verify COMPLETED state. Writing seven copies of the same infrastructure made the case for extracting `CaseIntegrationTestSupport` as a static utility: `startCase`, `awaitWorker`, `completeHumanTask`, `awaitCaseCompleted`.

Three of the seven cases complete naturally — family-vote (pure humanTask), care-episode (two workers plus a humanTask), and travel-plan (parallel searches, budget gate, approval humanTask, booking, confirmation). The other four block at QhorusMessageSignalBridge dependencies where `.channelMessage` needs an external signal. Those tests verify the non-bridge portion runs correctly and stop at the bridge boundary. Full bridge testing is infrastructure scope, not case definition scope.

## Where this leaves us

The Layer 6 review backlog is clear. Twelve integration tests now exercise actual case workflows end-to-end — three that complete naturally, four that verify up to the bridge boundary. The next piece of real work is #20 — the ActionRiskClassifier oversight gate.
