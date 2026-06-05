# Handoff — casehub-life #22/#24 closed, on main
2026-06-05

*Updated: parent#148, parent#153 closed — removed from backlog.*

## Last Session

Closed #22 (Layer 6 code review fixes) and #24 (integration tests for 7 case definitions) on one branch. Fixed attestation test filter tautology (`verdict != null` → `trustDimension == null`), removed duplicate test YAML, added cold-start tests, extracted `CaseIntegrationTestSupport` shared utility, wrote 9 new integration test methods across 7 case definition classes. ARC42STORIES.MD synced — Layer 6 status updated to complete.

## Immediate Next Step

`/work start 20` — Explore ActionRiskClassifier oversight gate (research/design). Read `docs/specs/life-actor-model.md` and `docs/specs/2026-06-03-layer6-trust-routing.md`.

## What's Left

- `life#25` — migrate CaseHub workers from Worker.builder to FuncWorkflowBuilder · M · Low
- Pre-existing test failures — `LEDGER_SUBJECT_SEQUENCE` table missing in H2 (11 failures across 7 test classes) · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #20 | Explore ActionRiskClassifier oversight gate | M | High | Research/design — builds on Layer 6 trust routing |
| — | Layer 7: OpenClaw as WorkerProvisioner | L | High | Next layer — makes trust routing decisions consequential |

## References

- Blog: `blog/2026-06-05-mdp01-filter-tested-nothing.md`
- Specs: `docs/specs/2026-06-05-l6-code-review-fixes.md`, `docs/specs/2026-06-05-case-integration-tests.md`
