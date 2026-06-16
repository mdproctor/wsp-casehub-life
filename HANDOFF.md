# Handoff — 2026-06-16
CI fixed — two SNAPSHOT-induced failures resolved, code reviewed, docs synced

## Last Session

Fixed CI broken by casehub-qhorus and casehub-ledger SNAPSHOTs. Two root causes: (1) `QhorusInboundCurrentPrincipal` introduced as `@Default` causing 27 CDI ambiguity errors — resolved via `quarkus.arc.exclude-types` in both prod and test configs; (2) casehub-ledger added a build-time validator requiring `domainContentBytes()` on all `LedgerEntry` subclasses — added to all four life ledger entries. Code review found comment style nit + missing test; both fixed. CLAUDE.md updated with both patterns. Protocol `PP-20260615-8ed738` captured. Created `casehubio/parent#251` (auth retrofit platform epic). 3 garden entries submitted.

## Immediate Next Step

Start `engine#463` in the engine session — function-as-worker abstraction design. This unblocks OpenClaw WorkerProvisioner (casehub-openclaw) and then `life#25`.

## What's Left

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

- Blog: `blog/2026-06-16-mdp01-snapshot-silent-alternatives.md`
- Protocol: `docs/protocols/casehub-life/current-principal-cdi-exclusion.md` (PP-20260615-8ed738)
- Auth retrofit epic: `casehubio/parent#251`
- Garden: GE-20260616-d70e7e, GE-20260616-716524, GE-20260616-fa89ff
