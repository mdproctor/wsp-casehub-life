# Handoff — 2026-06-29

*Updated: engine#569 closed — removed from backlog.*

Quality sweep complete. Closed #30 (scattered business logic audit), #31 (ledger fields), #41 (junior visibility), #42 (MCP verification), #43 (CDI exclude-types). All 5 issues merged to main. 517 tests green.

## Last Session

Eliminated all remaining scattered business logic: `reasonTemplate()` on `HouseholdActionType`, `escalationTemplate()` on `CommitmentMode`, `HouseholdActionThresholdKeys` descriptor map, `domain` + `oversightKey` columns on `LifeCommitmentRecord`, FINANCE hardcode fixed, `delegateTo` semantic overload resolved. Added `LifeTaskVisibilityPolicy` SPI with `JuniorLifeTaskVisibilityPolicy` for data-level junior filtering. Removed 6 CurrentPrincipal exclude-types entries (CDI `@Alternative @Priority` resolves since platform#112). Removed premature `appointmentDate` from health ledger, populated `jurisdiction` from config. Verified Qhorus MCP is CDI-only (no external consumers). Adversarial design review caught 10 substantive issues before implementation — notably `OversightGateRequest.domain()` field requirement and the `delegateTo` semantic overload.

## Immediate Next Step

Pick up life#47 (structural CaseHub duplication — augment pattern, cap() helper, double-checked lock).

## What's Left

- life#47 — structural CaseHub duplication (augment pattern, cap() helper) · M · Low
- life#48 — per-action jurisdiction on LegalActionLedgerEntry · S · Med
- (to file on engine) — engine should call terminate() on provisioner at case terminal state
- (to file on engine) — add timer sentry type for periodic binding evaluation
- (to file on engine) — extend bridge to route STATUS messages
- (to file on openclaw) — OpenClawAgentRegistry 1:N support

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| life#47 | Structural CaseHub duplication | M | Low | Extract augment pattern, cap() helper |
| life#48 | Per-action jurisdiction on LegalActionLedgerEntry | S | Med | Filed this session; tenant-wide config is interim |

## References

- Spec: `docs/specs/2026-06-29-quality-sweep-design.md`
- Plan: `plans/attic/issue-30-quality-sweep/2026-06-29-quality-sweep.md`
