# HANDOFF — casehub-life

**Date:** 2026-06-22
**Project:** `/Users/mdproctor/claude/casehub/life`
**Workspace:** `/Users/mdproctor/claude/public/casehub/life`

---

## Last Session

Auth retrofit complete. `casehub-platform-oidc` wired — `OidcCurrentPrincipal @RequestScoped` is the active `CurrentPrincipal` in production. `@RolesAllowed` on all 12 endpoints across 5 REST resources (household-admin/member/junior). RBAC-differentiated risk thresholds in `LifeActionRiskClassifier`: admin elevated (spend/contractor 500, booking 300), junior always-gates on AMOUNT_THRESHOLD, context-inactive falls back to member threshold. Commits squashed 9→3 and pushed to origin/main. Closes life#40 and life#26.

## Immediate Next Step

Pick from What's Next — all unblocked. life#41 (junior data-level task filter) is the direct follow-on if RBAC work continues.

## Cross-Module

**We're enabling:**
- casehub-devtown and casehub-clinical — same OIDC wiring pattern (parent#251); devtown#90 in progress in its own session
- parent#300 filed — PLATFORM.md + casehub-life.md sync needed in casehub-parent (peer repo)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #41 | junior data-level task visibility filter — GET /life-tasks/{id} own-tasks-only | M | Med | Follows life#40 ✅ |
| #103 | Credential resolution — secrets backend for `credentialRef` | M | Med | Unblocked |
| #85 | ScimDIDResolver — synthetic DID from SCIM | M | Med | Unblocked |
| #105 | LangChain4j AgentProvider bridge (provider-agnostic) | M | Med | Unblocked |
| #84 | JwtVCValidator — W3C VC JWT credential validation | M | High | Unblocked |

## References

- Diary: `blog/2026-06-22-mdp01-first-harness-with-real-auth.md`
- Spec: `docs/specs/2026-06-22-oidc-rbac-auth-design.md`
- Protocols: PP-20260622-eb234d (oidc-harness-wiring-checklist), PP-20260622-44f497 (rbac-classifier-context-guard)
- Garden: GE-20260622-580d45 (auth.enabled-in-dev-mode=false technique)
