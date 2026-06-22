# Design: OIDC wiring + RBAC-differentiated thresholds

**Issues:** life#40 (OIDC wiring) + life#26 (RBAC thresholds) — closes both  
**Branch:** `issue-40-wire-platform-oidc`  
**Date:** 2026-06-22  
**Deferred:** life#41 — junior data-level task visibility filter (operation gate correct, data gate separate)

---

## Context

`CurrentPrincipal.groups()` always returns empty in all running harnesses because no harness
has `casehub-platform-oidc` on the classpath yet. `casehub-platform-oidc` ships
`OidcCurrentPrincipal @RequestScoped` which reads roles from `SecurityIdentity.getRoles()`.
Activation requires: (1) classpath dep, (2) Quarkus OIDC config, (3) `@RolesAllowed` on REST
resources, (4) business logic that reads `groups()` (the risk classifier).

life#26 (RBAC-differentiated risk thresholds) is the first consumer of real groups — no point
implementing it before groups work, so both issues land together.

---

## 1. Dependencies (app/pom.xml)

**Add compile dep:**
```xml
<!-- casehub-platform-oidc — OidcCurrentPrincipal @RequestScoped; brings quarkus-oidc transitively -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-oidc</artifactId>
</dependency>
```

**Add test dep:**
```xml
<!-- @TestSecurity — controls SecurityIdentity in @QuarkusTest without a real OIDC server -->
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-test-security</artifactId>
  <scope>test</scope>
</dependency>
```

`quarkus-oidc` is transitive through `casehub-platform-oidc`; no explicit declaration needed.

---

## 2. CDI wiring

**Production:** `OidcCurrentPrincipal @RequestScoped` becomes the sole `CurrentPrincipal`.
`MockCurrentPrincipal @DefaultBean` is already excluded from production `application.properties`
(written in anticipation of this landing). No new exclusions needed.

**Tests:** `FixedCurrentPrincipal @Alternative @Priority(1)` (from `casehub-platform-testing`)
globally displaces `OidcCurrentPrincipal @RequestScoped` for `CurrentPrincipal` injection via
standard CDI alternative selection. No new exclusions needed.

**Two separate paths in production:** `SecurityIdentity` → `@RolesAllowed` enforcement;
`OidcCurrentPrincipal.groups()` bridges `SecurityIdentity.getRoles()` → `CurrentPrincipal.groups()`
for business logic. In tests, `@TestSecurity` controls the former and `FixedCurrentPrincipal`
controls the latter — the bridge itself is tested in `casehub-platform-oidc` unit tests.

---

## 3. Configuration

### Production application.properties

```properties
# casehub-platform-oidc (life#40)
# Required deployment env vars — NOT set here (same pattern as openclaw config):
#   QUARKUS_OIDC_AUTH_SERVER_URL — e.g. https://auth.example.com/realms/casehub
#   QUARKUS_OIDC_CLIENT_ID       — e.g. casehub-life
quarkus.oidc.application-type=service
```

### Test application.properties

```properties
# OIDC test config (life#40)
# GE-20260521-f50602: discovery-disabled requires jwks-path (lazy-loaded, never fetched with @TestSecurity)
# GE-20260601-08a351: devservices disabled — Keycloak container startup suppressed
quarkus.oidc.auth-server-url=http://localhost:8180/realms/test
quarkus.oidc.discovery-enabled=false
quarkus.oidc.jwks-path=protocol/openid-connect/certs
quarkus.keycloak.devservices.enabled=false
```

---

## 4. @RolesAllowed mapping

Method-level throughout for explicitness. Uses `HouseholdGroups` constants (`ADMIN`, `MEMBER`,
`JUNIOR`).

| Resource | Method | Path | Roles |
|---|---|---|---|
| `LifeTaskResource` | POST | `/life-tasks` | ADMIN, MEMBER |
| `LifeTaskResource` | GET | `/life-tasks/{id}` | ADMIN, MEMBER, JUNIOR |
| `ExternalActorResource` | POST | `/external-actors` | ADMIN, MEMBER |
| `ExternalActorResource` | GET | `/external-actors` | ADMIN, MEMBER |
| `ExternalActorResource` | GET | `/external-actors/{id}` | ADMIN, MEMBER |
| `ExternalActorResource` | PUT | `/external-actors/{id}` | ADMIN only |
| `ExternalActorResource` | DELETE | `/external-actors/{id}` | ADMIN only |
| `ExternalActorResource` | DELETE | `/external-actors/{id}/personal-data` | ADMIN only |
| `ExternalActorResource` | GET | `/external-actors/{id}/tasks` | ADMIN, MEMBER |
| `LifeCaseResource` | POST | `/life-cases` | ADMIN, MEMBER |
| `LifeOversightGateResource` | POST | `/life-oversight-gates` | ADMIN, MEMBER |
| `LifeCommitmentResource` | POST | `/life-tasks/{id}/commit` | ADMIN, MEMBER |

JUNIOR has operation-level access to `GET /life-tasks/{id}`. Data-level "own tasks only"
restriction is life#41 (blocked on life#40).

---

## 5. RBAC-differentiated thresholds (life#26)

### New PreferenceKeys in LifeRiskPolicyKeys

```
ADMIN_SPEND_THRESHOLD       casehubio.life.risk-policy.admin.spend.threshold       500.0
ADMIN_BOOKING_THRESHOLD     casehubio.life.risk-policy.admin.booking.threshold      300.0
ADMIN_CONTRACTOR_THRESHOLD  casehubio.life.risk-policy.admin.contractor.threshold   500.0
```

Existing member keys (`SPEND_THRESHOLD`, `BOOKING_THRESHOLD`, `CONTRACTOR_THRESHOLD`,
`APPROVAL_EXPIRES_HOURS`) unchanged.

### risk-policy.yaml additions

Three admin threshold entries appended to the existing scope entry.

### LifeActionRiskClassifier changes

Inject `@Inject CurrentPrincipal principal`.

**Request context guard** — the classifier is called during case worker execution, which may be
async (scheduler, QHorus observer) with no HTTP request context active. `OidcCurrentPrincipal`
is `@RequestScoped`; accessing it outside a request context throws `ContextNotActiveException`.
Safe fallback: treat as member threshold when context is absent (never grants elevated privileges).

```java
private boolean requestContextActive() {
    return Arc.container().requestContext().isActive();
}
private boolean isAdmin() {
    return requestContextActive() && principal.groups().contains(HouseholdGroups.ADMIN);
}
private boolean isJunior() {
    return requestContextActive()
        && !principal.groups().contains(HouseholdGroups.ADMIN)
        && !principal.groups().contains(HouseholdGroups.MEMBER);
}
```

**Threshold behavior by role:**

| GatePolicy | ADMIN | MEMBER | JUNIOR or no context |
|---|---|---|---|
| `NEVER` | Autonomous | Autonomous | Autonomous |
| `ALWAYS` | GateRequired | GateRequired | GateRequired |
| `AMOUNT_THRESHOLD` | elevated threshold | standard threshold | always GateRequired |

For `AMOUNT_THRESHOLD` + junior: apply `ALWAYS`-equivalent (build gate regardless of amount,
same gate structure as `ALWAYS` types). Cleaner than threshold=0.

`resolveThreshold()` switches on admin vs member key per action type — same structure as
current but selecting from two key sets. Comment preserved: "interim until HouseholdRiskRule
descriptor pattern lands" (tracked separately in `2026-06-08-business-logic-centralization.md`).

---

## 6. Test plan

### Existing test classes — keep green

Add class-level `@TestSecurity(user="household-admin", roles={"household-admin"})` to:
- `ExternalActorResourceTest`
- `ExternalActorGdprResourceTest`
- `LifeTaskResourceTest`
- `LifeCommitmentResourceTest`
- `LifeCaseResourceTest`

This gives all existing test methods a valid authenticated admin principal without changing
their logic. `FixedCurrentPrincipal` remains active for `CurrentPrincipal` injection in
those tests.

### LifeActionRiskClassifierTest — add RBAC cases

Add `@Mock CurrentPrincipal principal` to the existing mock-based test. New cases:
- Admin + amount below `ADMIN_SPEND_THRESHOLD` → Autonomous
- Admin + amount above `ADMIN_SPEND_THRESHOLD` → GateRequired
- Member + amount above `SPEND_THRESHOLD` → GateRequired (unchanged behaviour)
- Junior (neither group) + any amount on AMOUNT_THRESHOLD type → GateRequired
- Context inactive (no request context) + amount above member threshold → GateRequired (member fallback)

### New LifeRestSecurityTest

`@QuarkusTest` covering authorization boundaries via RestAssured + `@TestSecurity`:
- No `@TestSecurity` (unauthenticated) → 401 on guarded endpoints
- `household-junior` → 403 on POST/PUT/DELETE endpoints; 200 (or appropriate 4xx for empty data) on `GET /life-tasks/{id}`
- `household-member` → 200 on member endpoints; 403 on ADMIN-only (PUT/DELETE actor)
- `household-admin` → 200 on all endpoints

---

## 7. Out of scope

- **life#41** — junior data-level task visibility filter (`GET /life-tasks/{id}` returns own tasks only)
- **OIDC provider selection** — deployment-specific; documented as required env vars only
- **HouseholdRiskRule descriptor/handler refactor** — planned in `2026-06-08-business-logic-centralization.md`; tracked separately; the `resolveThreshold()` switch remains interim
