# Handoff — casehub-life Layer 6 trust routing complete, on main
2026-06-04

## Last Session

Layer 6 trust routing implemented and merged. Attestation pipeline
(LifeOutcomeAttestationWriter), TrustRoutingPolicyProvider with 8 domain
policies and 32-entry capability→domain mapping, ExternalActor REST
enrichment with ledger-backed TrustProfile. 233 tests pass. Issue #11 closed.

## Immediate Next Step

`/work start 20` — Explore ActionRiskClassifier oversight gate (research/design,
not implementation). Read `docs/specs/life-actor-model.md` for the actor model
and `docs/specs/2026-06-03-layer6-trust-routing.md` for the trust routing context
that #20 builds on.

## What's Left

- `parent#114` — sync `docs/repos/casehub-life.md` for Layers 3–5 completion · XS · Low
- `parent#148` — clarify trust-maturity-model protocol (thresholds in code vs YAML) · XS · Low
- `parent#153` — add engine-ledger + platform-config to Cross-Repo Dependency Map · XS · Low
- `life#22` — minor code review findings (test filters, FQ types, YAML duplication) · XS · Low
- `engine#410` — CaseDefinition forward lookup; L5 integration tests @Disabled · blocks integration tests · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #20 | Explore ActionRiskClassifier oversight gate | M | High | Research/design — builds on Layer 6 trust routing |
| — | Layer 6 ARC42STORIES.MD §9.4 entry | S | Low | Write before closing Layer 6 fully |
| — | Layer 7: OpenClaw as WorkerProvisioner | L | High | Next layer — makes trust routing decisions consequential |

## References

- Spec: `docs/specs/2026-06-03-layer6-trust-routing.md`
- Blog: `blog/2026-06-03-mdp01-layer6-trust-routing.md`
- Plan: `plans/attic/issue-011-trust-routing/2026-06-03-layer6-trust-routing.md`
