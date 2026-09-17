# HANDOFF — casehub-life

**Date:** 2026-08-10
**Branch:** `issue-85-life-ui-data-wiring`
**Slot:** 99 (life + blocks-ui + ledger)

---

## Last Session

Wired Life UI views to real platform data (epic #85): reused `casehub-ledger-rest` instead of duplicating, added case-scoped tasks endpoint, built `cases-view.ts` (6 tabs) and `journal-view.ts`, added `actionType` to `PendingActionResponse`, applied visibility filtering (#102), fixed metadata/payload field alignment in ledger repo (#101). Created epic #95 with 7 child issues for remaining gaps. Added blocks-ui and ledger repos to slot 99.

## Immediate Next Step

Continue with #96 (routing data endpoint) and #97 (CBR retrieval endpoint). Both need investigation: routing data may come from engine execution records or WorkItem candidateScores; CBR data lives in ephemeral case context — consider re-querying the CBR store or persisting results at case start. Run `/work` to resume.

## Cross-Module

**Enabled:**
- ledger — metadata→payload field rename landed (commit a69b9c8, life#101). Life consumes via `casehub-ledger-rest` dependency.
