# Design: OIDC wiring + RBAC-differentiated thresholds

**Issues:** life#40 (OIDC wiring) + life#26 (RBAC thresholds) — closes both  
**Branch:** `issue-40-wire-platform-oidc`  
**Date:** 2026-06-22  
**Deferred:** life#41 — junior data-level task visibility filter (operation gate correct, data gate separate)

---

## Context

`CurrentPrincipal.groups()` always returns empty in all running harnesses for two independent
reasons: (1) `TenantScopedPrincipal @RequestScoped @Unremovable` from casehub-work is the
active production `CurrentPrincipal` and its `groups()` always returns `Set.of()`; (2) no
harness has `casehub-platform-oidc` on the classpath, so `OidcCurrentPrincipal` — which reads
roles from `SecurityIdentity.getRoles()` — has never been activated. Both causes must be
resolved together.

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

**Problem:** adding `OidcCurrentPrincipal @RequestScoped` (no `@DefaultBean`, no `@Alternative`)
to a classpath that already has `TenantScopedPrincipal @RequestScoped @Unremovable` (same
non-default, non-alternative profile) creates two equally-eligible `CurrentPrincipal` beans.
Quarkus throws `AmbiguousResolutionException` at startup.

**Production fix:** add to `quarkus.arc.exclude-types` in production `application.properties`:
```
io.casehub.work.runtime.service.TenantScopedPrincipal
```
The test `application.properties` already excludes it. This mirrors that exclusion to production.
After exclusion, `OidcCurrentPrincipal @RequestScoped` becomes the sole active `CurrentPrincipal`.
`MockCurrentPrincipal @DefaultBean` (already excluded) is moot.

**Tests:** `FixedCurrentPrincipal @Alternative @Priority(1)` (from `casehub-platform-testing`)
globally displaces `OidcCurrentPrincipal @RequestScoped` for `CurrentPrincipal` injection via
standard CDI alternative selection — no new exclusions needed.

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

# Dev profile — disable OIDC and all security enforcement so endpoints are accessible without auth.
# quarkus.security.auth.enabled-in-dev-mode=false activates DevModeDisabledAuthorizationController
# (@Alternative @Priority @Singleton, quarkus-security-runtime-spi 3.32.2) which returns false from
# isAuthorizationEnabled(). Both EagerSecurityHandler (REST enforcement) and StandardSecurityCheckInterceptor
# (CDI path) check this flag and bail out early — @RolesAllowed is not enforced in dev mode.
# quarkus.oidc.enabled=false prevents OIDC from attempting token validation (needed in addition to
# the auth.enabled-in-dev-mode flag to stop OIDC from intercepting requests before security is checked).
%dev.quarkus.security.auth.enabled-in-dev-mode=false
%dev.quarkus.oidc.enabled=false
%dev.quarkus.keycloak.devservices.enabled=false
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

**Request context guard:** `LifeActionRiskClassifier` is `@ApplicationScoped`; `OidcCurrentPrincipal`
is `@RequestScoped`. Calling methods on the injected proxy outside an HTTP request context
throws `ContextNotActiveException`. This happens during async worker execution (scheduler,
QHorus observer). Guard via try/catch at each call site — idiomatic CDI, no Quarkus internal
coupling, testable in plain Mockito by stubbing `groups()` to throw:

```java
private boolean isAdmin() {
    try {
        return principal.groups().contains(HouseholdGroups.ADMIN);
    } catch (ContextNotActiveException e) {
        return false;
    }
}

private boolean isJunior() {
    try {
        // Negative definition is deliberate: unknown/unrecognised roles → always-gate.
        // Fail-secure for a financial-gate system: an unrecognised identity must never
        // act autonomously. When the platform adds a new role (e.g. household-carer),
        // that role will gate until explicitly handled here. The JUNIOR constant is used
        // in @RolesAllowed but not here — junior-gate behaviour is the fallback for any
        // non-admin, non-member identity, which is the correct default for this context.
        return !principal.groups().contains(HouseholdGroups.ADMIN)
            && !principal.groups().contains(HouseholdGroups.MEMBER);
    } catch (ContextNotActiveException e) {
        return false;
    }
}
```

When context is absent: `isAdmin()` and `isJunior()` both return `false` → member threshold
applied. Safe: never autonomously elevates privilege; never always-gates when context is
absent (a background worker with no principal should not require human approval for routine
operations).

**Behavior by role:**

| GatePolicy | ADMIN | MEMBER | JUNIOR / unknown role | No context (async) |
|---|---|---|---|---|
| `NEVER` | Autonomous | Autonomous | Autonomous | Autonomous |
| `ALWAYS` | GateRequired | GateRequired | GateRequired | GateRequired |
| `AMOUNT_THRESHOLD` | admin threshold | member threshold | always GateRequired | member threshold |

For `AMOUNT_THRESHOLD` + junior/unknown: apply `GateRequired` regardless of amount (equivalent
to `ALWAYS` policy). Cleaner than threshold=0.

`resolveThreshold()` selects admin or member key per action type. Comment preserved:
"interim until HouseholdRiskRule descriptor pattern lands."

---

## 6. Test plan

### Existing REST test classes — keep green

Add class-level `@TestSecurity(user="household-admin", roles={"household-admin"})` to the
following classes (verified package paths):

| Class | Path |
|---|---|
| `LifeCaseResourceTest` | `app/src/test/java/io/casehub/life/app/resource/LifeCaseResourceTest.java` |
| `ExternalActorResourceTest` | `app/src/test/java/io/casehub/life/app/ExternalActorResourceTest.java` |
| `ExternalActorGdprResourceTest` | `app/src/test/java/io/casehub/life/app/ExternalActorGdprResourceTest.java` |
| `LifeTaskResourceTest` | `app/src/test/java/io/casehub/life/app/LifeTaskResourceTest.java` |
| `LifeCommitmentResourceTest` | `app/src/test/java/io/casehub/life/app/LifeCommitmentResourceTest.java` |

### LifeActionRiskClassifierTest (Mockito) — add RBAC cases

Add `@Mock CurrentPrincipal principal` (field injection via `@InjectMocks`). New cases:
- Admin + amount below `ADMIN_SPEND_THRESHOLD` (500.0) → Autonomous
- Admin + amount at/above `ADMIN_SPEND_THRESHOLD` → GateRequired
- Member + amount at/above `SPEND_THRESHOLD` (100.0) → GateRequired (unchanged behaviour)
- Junior (neither ADMIN nor MEMBER group) + any amount on `AMOUNT_THRESHOLD` type → GateRequired
- **Context inactive:** `groups()` stubbed to throw `ContextNotActiveException`, amount above
  `SPEND_THRESHOLD` → GateRequired (member threshold: amount 100.0 triggers gate)
- **Context inactive + below threshold:** `groups()` stubbed to throw `ContextNotActiveException`,
  amount below `SPEND_THRESHOLD` (e.g. 50.0) → Autonomous — validates that background workers
  are NOT always-gated; they get member threshold, not ALWAYS policy

### LifeActionRiskClassifierQuarkusTest — fix broken tests + add RBAC cases

The existing AMOUNT_THRESHOLD tests break after adding RBAC: `FixedCurrentPrincipal` defaults
to empty groups → `isJunior()` = true → all AMOUNT_THRESHOLD types return GateRequired
regardless of amount. `spendPurchase_belowYamlThreshold_returnsAutonomous` fails; the
at-threshold tests pass but via junior-always-gate, not the threshold path.

**Fix:** inject `FixedCurrentPrincipal` by concrete type and set member groups in `@BeforeEach`:

```java
@Inject
FixedCurrentPrincipal fixedPrincipal;   // @ApplicationScoped — directly injectable

@BeforeEach
void setMemberPrincipal() {
    fixedPrincipal.setGroups(Set.of(HouseholdGroups.MEMBER));
}

@AfterEach
void resetPrincipal() {
    fixedPrincipal.reset();
}
```

The three existing AMOUNT_THRESHOLD tests then correctly exercise the member-threshold path.

**Add RBAC-specific QuarkusTest cases** (end-to-end with YAML loaded):
- Set admin groups: `fixedPrincipal.setGroups(Set.of(HouseholdGroups.ADMIN))` →
  spend amount below `ADMIN_SPEND_THRESHOLD` (500.0) → Autonomous (elevated threshold honoured)
- Set admin groups → spend amount at `ADMIN_SPEND_THRESHOLD` → GateRequired
- Set junior groups (`Set.of(HouseholdGroups.JUNIOR)`) → any AMOUNT_THRESHOLD action → GateRequired
  (validates that junior always-gate is enforced through the full CDI + YAML stack)

### New LifeRestSecurityTest

`@QuarkusTest` covering authorization boundaries via RestAssured + `@TestSecurity`:
- No `@TestSecurity` (unauthenticated) → 401 on guarded endpoints
- `household-junior` → 403 on POST/PUT/DELETE endpoints; not 401, not 403 on `GET /life-tasks/{id}`
- `household-member` → not 401, not 403 on member endpoints; 403 on ADMIN-only (PUT/DELETE actor)
- `household-admin` → not 401, not 403 on all endpoints

---

## 7. Out of scope

- **life#41** — junior data-level task visibility filter (`GET /life-tasks/{id}` returns own tasks only)
- **OIDC provider selection** — deployment-specific; documented as required env vars only
- **HouseholdRiskRule descriptor/handler refactor** — planned in `2026-06-08-business-logic-centralization.md`; tracked separately; the `resolveThreshold()` switch remains interim
