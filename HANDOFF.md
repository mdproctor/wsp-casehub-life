# Handoff — casehub-life Layer 4 done, on main
2026-05-30

## Last Session

Layer 4 (casehub-ledger tamper-evident audit) fully implemented, reviewed, squashed,
and merged to main+upstream. 4 LedgerEntry subclasses, unified LifeLedgerWriter,
LifeDecisionLedgerObserver CDI observer, GDPR Art.17 erasure. 90 tests pass.
Tutorial framing stripped from CLAUDE.md and LAYER-LOG.md. Issues #5, #10, #19 closed.

## Immediate Next Step

Layer 5 (`casehub-engine` CasePlanModel workflows) is the next layer, but engine deps
are removed (engine#379/#380 — now fixed in source, need local build). Run
`mvn install -DskipTests -f ../engine/pom.xml` to install the fixed SNAPSHOT before
starting Layer 5.

## What's Left

- `parent#114` — sync `docs/repos/casehub-life.md` for Layers 3+4 completion (supersedes parent#96) · XS · Low
- `parent#96` — Layer 3 completion update (superseded by parent#114) · XS · Low
- GE-20260530-da427e — garden push blocked by pre-push hook; committed locally, needs push · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #6 | Layer 5: casehub-engine CasePlanModel workflows | L | High | Blocked until engine SNAPSHOT installed locally |
| #11 | Trust dimension score fields on ExternalActor | S | Med | Blocked by Layer 5/6 (needs engine) |
| #7 | Layer 6: Trust routing | L | High | Blocked by Layer 5 |
| #20 | Explore ActionRiskClassifier oversight gate | M | High | Research / design — no blocker |

## References

- Spec: `docs/specs/2026-05-30-layer4-casehub-ledger-design.md`
- LAYER-LOG: Layer 4 entry complete with full key files/wiring/gotchas/pattern to replicate
- Plan: `plans/attic/issue-5-layer4-casehub-ledger/2026-05-30-layer4-casehub-ledger.md`
- Garden: GE-20260530-da427e (multi-PU gotcha), GE-20260511-b6f903 revised (@PrePersist)
