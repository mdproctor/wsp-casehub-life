# Handoff — 2026-06-29

Closed #34 (GDPR CaseMemoryStore erasure). Wired `eraseEntity()` into existing erasure flow, added `memoryRecordsErased` audit field, fixed hardcoded `erasedBy` to use `CurrentPrincipal.actorId()`. Added `quarkus-junit5-mockito` for `@InjectMock` testing. 401 tests green.

## Last Session

Rescoped #34 from "build full erasure endpoint" (already done in Layer 4) to "wire CaseMemoryStore.eraseEntity() into existing flow". Adversarial design review caught three issues that shaped implementation: `domainContentBytes()` Merkle hash exclusion, JTA transaction boundary with external memory stores, `erasedBy` identity flow from `CurrentPrincipal`. Discovered Panache mockStatic limitation (GE-20260629-74fc65 — garden entry). Code review drove addition of `ExternalActorServiceTest` with `@InjectMock` and `LOG.debugf` on the catch path.

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
| life#48 | Per-action jurisdiction on LegalActionLedgerEntry | S | Med | Tenant-wide config is interim |

## References

- Spec: `docs/specs/2026-06-29-gdpr-casememorystore-erasure-design.md`
- Garden: GE-20260629-74fc65 (Panache mockStatic limitation)
- Blog: `blog/2026-06-29-mdp01-memory-gap-gdpr-erasure.md`
