# Handoff — casehub-life Layer 5 closed, on main
2026-06-01

## Last Session

Layer 5 (casehub-engine CasePlanModel workflows) closed. 8 case definitions,
squash-merged to main+upstream. engine#410 filed for integration test blocker.
CDI wiring fix for all 6 engine-persistence-memory alternatives. 199 tests pass.
Issue #6 closed.

## Immediate Next Step

LAYER-LOG.md Layer 5 entry is still a stub (all placeholders). Write it
before starting any new work — layer is not formally complete without it.

## What's Left

- LAYER-LOG.md Layer 5 entry — fill in architectural pattern, key protocols, design refs, key files · XS · Low
- `parent#114` — sync `docs/repos/casehub-life.md` for Layers 3–5 completion · XS · Low
- engine#410 — CaseDefinition forward lookup failure; integration tests `@Disabled` until fixed · blocks integration tests · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #11 | Trust dimension score fields on ExternalActor | S | Med | Needs Layer 6 design |
| #7 | Layer 6: Trust routing — Bayesian Beta scores | L | High | Next layer |
| #20 | Explore ActionRiskClassifier oversight gate | M | High | Research / design |

## References

- Spec: `docs/specs/2026-05-31-layer5-casehub-engine-design.md`
- LAYER-LOG: Layer 5 entry (stub — needs completion)
- Plan: `plans/attic/issue-6-layer5-engine-workflows/2026-05-31-layer5-casehub-engine.md`
- Garden: GE-20260601-8ff52b (Surefire retry + assumeTrue gotcha)
- Blog: `blog/2026-06-01-mdp01-layer5-eight-workflows.md`
