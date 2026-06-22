# OIDC Wiring + RBAC-Differentiated Thresholds Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `casehub-platform-oidc` into `casehub-life` so `CurrentPrincipal.groups()` reflects real OIDC roles, add `@RolesAllowed` to all REST resources, and differentiate risk thresholds by household role.

**Architecture:** `casehub-platform-oidc` adds `OidcCurrentPrincipal @RequestScoped` which bridges `SecurityIdentity.getRoles()` → `CurrentPrincipal.groups()`. `@RolesAllowed` on REST resources uses `SecurityIdentity` directly (Quarkus security layer). The risk classifier injects `CurrentPrincipal` and applies admin/member/junior threshold tiers. Tests use `@TestSecurity` for HTTP-layer auth and `FixedCurrentPrincipal` (already active via `@Alternative @Priority(1)`) for CDI-layer group checks.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, `casehub-platform-oidc`, `quarkus-test-security`, Mockito, RestAssured.

**Spec:** `specs/2026-06-22-oidc-rbac-auth-design.md`
**Branch:** `issue-40-wire-platform-oidc`
**Closes:** life#40 (OIDC wiring), life#26 (RBAC thresholds)

---

## File Map

| Action | File | What changes |
|--------|------|--------------|
| Modify | `app/pom.xml` | Add 2 deps |
| Modify | `app/src/main/resources/application.properties` | Exclude TenantScopedPrincipal; add OIDC prod + dev config |
| Modify | `app/src/test/resources/application.properties` | Add OIDC test config |
| Modify | `app/src/main/java/io/casehub/life/app/routing/LifeRiskPolicyKeys.java` | Add 3 admin threshold keys |
| Modify | `app/src/main/resources/casehub/life/risk-policy.yaml` | Add 3 admin threshold values |
| Modify | `app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierTest.java` | Add `@Mock CurrentPrincipal`, stub member groups, add 6 RBAC test cases |
| Modify | `app/src/main/java/io/casehub/life/app/routing/LifeActionRiskClassifier.java` | Inject `CurrentPrincipal`; add `isAdmin()`/`isJunior()`; update `classifyKnownType()` and `resolveThreshold()` |
| Modify | `app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierQuarkusTest.java` | Inject `FixedCurrentPrincipal`; set member groups in `@BeforeEach`; add admin/junior cases |
| Create | `app/src/test/java/io/casehub/life/app/security/LifeRestSecurityTest.java` | New security boundary test |
| Modify | `app/src/main/java/io/casehub/life/app/resource/LifeTaskResource.java` | Add `@RolesAllowed` |
| Modify | `app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java` | Add `@RolesAllowed` |
| Modify | `app/src/main/java/io/casehub/life/app/resource/LifeCaseResource.java` | Add `@RolesAllowed` |
| Modify | `app/src/main/java/io/casehub/life/app/resource/LifeOversightGateResource.java` | Add `@RolesAllowed` |
| Modify | `app/src/main/java/io/casehub/life/app/resource/LifeCommitmentResource.java` | Add `@RolesAllowed` |
| Modify | `app/src/test/java/io/casehub/life/app/ExternalActorResourceTest.java` | Add class-level `@TestSecurity` |
| Modify | `app/src/test/java/io/casehub/life/app/ExternalActorGdprResourceTest.java` | Add class-level `@TestSecurity` |
| Modify | `app/src/test/java/io/casehub/life/app/LifeTaskResourceTest.java` | Add class-level `@TestSecurity` |
| Modify | `app/src/test/java/io/casehub/life/app/LifeCommitmentResourceTest.java` | Add class-level `@TestSecurity` |
| Modify | `app/src/test/java/io/casehub/life/app/resource/LifeCaseResourceTest.java` | Add class-level `@TestSecurity` |

---

## Task 1: Add Dependencies to pom.xml

**Files:**
- Modify: `app/pom.xml`

- [ ] **Step 1: Add `casehub-platform-oidc` compile dep**

In `app/pom.xml`, inside the `<!-- Platform -->` comment block (after `casehub-platform-config`), add:

```xml
<!-- OidcCurrentPrincipal @RequestScoped — displaces MockCurrentPrincipal @DefaultBean;
     brings quarkus-oidc transitively. See life#40 spec for CDI wiring details. -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-oidc</artifactId>
</dependency>
```

- [ ] **Step 2: Add `quarkus-test-security` test dep**

In `app/pom.xml`, inside the `<!-- Test -->` block (after `quarkus-junit`), add:

```xml
<!-- @TestSecurity — controls SecurityIdentity in @QuarkusTest without a real OIDC server -->
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-test-security</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 3: Verify compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api -q && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl app -q
```

Expected: `BUILD SUCCESS` with no errors.

---

## Task 2: Production application.properties — CDI Exclusion + OIDC Config

**Files:**
- Modify: `app/src/main/resources/application.properties`

- [ ] **Step 1: Add TenantScopedPrincipal to quarkus.arc.exclude-types**

The existing `quarkus.arc.exclude-types` block ends with `io.casehub.persistence.memory.DefaultTestPrincipal`. Extend it to also exclude `TenantScopedPrincipal`:

```properties
quarkus.arc.exclude-types=\
  io.casehub.platform.mock.MockCurrentPrincipal,\
  io.casehub.platform.mock.MockGroupMembershipProvider,\
  io.casehub.qhorus.runtime.identity.QhorusInboundCurrentPrincipal,\
  io.casehub.persistence.memory.DefaultTestPrincipal,\
  io.casehub.work.runtime.service.TenantScopedPrincipal
```

- [ ] **Step 2: Add OIDC configuration section**

Append after the OpenClaw config block at the end of the file:

```properties
# ============================================================
# OIDC — casehub-platform-oidc (life#40)
# Required deployment env vars (do NOT set values here — empty strings bypass ConfigException):
#   QUARKUS_OIDC_AUTH_SERVER_URL — e.g. https://auth.example.com/realms/casehub
#   QUARKUS_OIDC_CLIENT_ID       — e.g. casehub-life
# ============================================================
quarkus.oidc.application-type=service

# Dev profile — disable OIDC and all security enforcement.
# quarkus.security.auth.enabled-in-dev-mode=false activates DevModeDisabledAuthorizationController
# which makes isAuthorizationEnabled()=false, bypassing EagerSecurityHandler and the CDI interceptor.
# quarkus.oidc.enabled=false prevents OIDC from attempting token validation.
%dev.quarkus.security.auth.enabled-in-dev-mode=false
%dev.quarkus.oidc.enabled=false
%dev.quarkus.keycloak.devservices.enabled=false
```

---

## Task 3: Test application.properties — OIDC Test Config

**Files:**
- Modify: `app/src/test/resources/application.properties`

- [ ] **Step 1: Add OIDC test configuration block**

Append after the OpenClaw test config block at the end of the file:

```properties
# ============================================================
# OIDC test config (life#40)
# GE-20260521-f50602: discovery-disabled requires jwks-path (lazy-loaded, never fetched with @TestSecurity)
# GE-20260601-08a351: devservices disabled — Keycloak container startup suppressed
# SecurityIdentity for @RolesAllowed is controlled by @TestSecurity.
# CurrentPrincipal in business logic is controlled by FixedCurrentPrincipal @Alternative @Priority(1).
# ============================================================
quarkus.oidc.auth-server-url=http://localhost:8180/realms/test
quarkus.oidc.discovery-enabled=false
quarkus.oidc.jwks-path=protocol/openid-connect/certs
quarkus.keycloak.devservices.enabled=false
```

- [ ] **Step 2: Verify compile still succeeds**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl app -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit config + dependency tasks**

```bash
git -C /Users/mdproctor/claude/casehub/life add \
  app/pom.xml \
  app/src/main/resources/application.properties \
  app/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/life commit -m "chore(#40): add casehub-platform-oidc dep + OIDC config for production, dev, and tests"
```

---

## Task 4: LifeRiskPolicyKeys — Add Admin Threshold Constants

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/routing/LifeRiskPolicyKeys.java`

- [ ] **Step 1: Add three admin threshold constants**

The current file ends with `APPROVAL_EXPIRES_HOURS` and then `private LifeRiskPolicyKeys() {}`. Insert the three new constants before the private constructor:

```java
/** Spend threshold for household-admin — higher authority, less friction. Default: 500.0. */
public static final PreferenceKey<DoublePreference> ADMIN_SPEND_THRESHOLD =
    new PreferenceKey<>(NS, "admin.spend.threshold", DoublePreference.of(500.0), DoublePreference::parse);

/** Contractor threshold for household-admin. Default: 500.0. */
public static final PreferenceKey<DoublePreference> ADMIN_CONTRACTOR_THRESHOLD =
    new PreferenceKey<>(NS, "admin.contractor.threshold", DoublePreference.of(500.0), DoublePreference::parse);

/** Booking threshold for household-admin. Default: 300.0. */
public static final PreferenceKey<DoublePreference> ADMIN_BOOKING_THRESHOLD =
    new PreferenceKey<>(NS, "admin.booking.threshold", DoublePreference.of(300.0), DoublePreference::parse);

private LifeRiskPolicyKeys() {}
```

---

## Task 5: risk-policy.yaml — Add Admin Threshold Values

**Files:**
- Modify: `app/src/main/resources/casehub/life/risk-policy.yaml`

- [ ] **Step 1: Add three admin threshold entries**

The current file is:
```yaml
entries:
  - scope: casehubio/life/risk-policy
    casehubio.life.risk-policy.spend.threshold: "100.0"
    casehubio.life.risk-policy.contractor.threshold: "200.0"
    casehubio.life.risk-policy.booking.threshold: "150.0"
    casehubio.life.risk-policy.approval.expires-hours: "24.0"
```

Replace with:
```yaml
entries:
  - scope: casehubio/life/risk-policy
    casehubio.life.risk-policy.spend.threshold: "100.0"
    casehubio.life.risk-policy.contractor.threshold: "200.0"
    casehubio.life.risk-policy.booking.threshold: "150.0"
    casehubio.life.risk-policy.approval.expires-hours: "24.0"
    casehubio.life.risk-policy.admin.spend.threshold: "500.0"
    casehubio.life.risk-policy.admin.contractor.threshold: "500.0"
    casehubio.life.risk-policy.admin.booking.threshold: "300.0"
```

- [ ] **Step 2: Commit keys + YAML**

```bash
git -C /Users/mdproctor/claude/casehub/life add \
  app/src/main/java/io/casehub/life/app/routing/LifeRiskPolicyKeys.java \
  app/src/main/resources/casehub/life/risk-policy.yaml
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#26): add admin threshold PreferenceKeys and risk-policy.yaml entries"
```

---

## Task 6: LifeActionRiskClassifierTest — Write Failing RBAC Tests (RED)

**Files:**
- Modify: `app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierTest.java`

- [ ] **Step 1: Add `@Mock CurrentPrincipal` field and extend `@BeforeEach`**

Add the import at the top of the file:
```java
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.enterprise.context.ContextNotActiveException;
```

Add `@Mock` field alongside `preferenceProvider`:
```java
@Mock
private CurrentPrincipal principal;
```

In `setUp()`, add two lenient stubs after the existing ones (member groups by default):
```java
lenient().when(principal.groups()).thenReturn(Set.of(HouseholdGroups.MEMBER));
lenient().when(prefs.get(LifeRiskPolicyKeys.ADMIN_SPEND_THRESHOLD)).thenReturn(DoublePreference.of(500.0));
lenient().when(prefs.get(LifeRiskPolicyKeys.ADMIN_CONTRACTOR_THRESHOLD)).thenReturn(DoublePreference.of(500.0));
lenient().when(prefs.get(LifeRiskPolicyKeys.ADMIN_BOOKING_THRESHOLD)).thenReturn(DoublePreference.of(300.0));
```

The existing tests set up `Set.of(HouseholdGroups.MEMBER)` as default, so existing AMOUNT_THRESHOLD tests continue to exercise the member path (unchanged semantics).

- [ ] **Step 2: Add RBAC test cases**

Append these test methods to the class (before the closing `}`):

```java
// --- RBAC: admin elevated threshold ---

@Test
void admin_spendPurchase_belowAdminThreshold_returnsAutonomous() {
    when(principal.groups()).thenReturn(Set.of(HouseholdGroups.ADMIN));
    // 400.0 is below admin threshold (500.0) but above member threshold (100.0)
    assertInstanceOf(Autonomous.class, classifier.classify(actionWithAmount(SPEND_PURCHASE, 400.0)));
}

@Test
void admin_spendPurchase_atAdminThreshold_returnsGateRequired() {
    when(principal.groups()).thenReturn(Set.of(HouseholdGroups.ADMIN));
    assertInstanceOf(GateRequired.class, classifier.classify(actionWithAmount(SPEND_PURCHASE, 500.0)));
}

@Test
void admin_contractorEngage_belowAdminThreshold_returnsAutonomous() {
    when(principal.groups()).thenReturn(Set.of(HouseholdGroups.ADMIN));
    // 400.0 is below admin contractor threshold (500.0) but above member threshold (200.0)
    assertInstanceOf(Autonomous.class, classifier.classify(actionWithAmount(CONTRACTOR_ENGAGE, 400.0)));
}

// --- RBAC: junior always gates on AMOUNT_THRESHOLD ---

@Test
void junior_spendPurchase_belowMemberThreshold_returnsGateRequired() {
    when(principal.groups()).thenReturn(Set.of(HouseholdGroups.JUNIOR));
    // 50.0 is below member threshold (100.0) — junior always gates regardless
    assertInstanceOf(GateRequired.class, classifier.classify(actionWithAmount(SPEND_PURCHASE, 50.0)));
}

@Test
void junior_contractorEngage_belowMemberThreshold_returnsGateRequired() {
    when(principal.groups()).thenReturn(Set.of(HouseholdGroups.JUNIOR));
    assertInstanceOf(GateRequired.class, classifier.classify(actionWithAmount(CONTRACTOR_ENGAGE, 10.0)));
}

// --- RBAC: context inactive (async/scheduler execution) — member threshold fallback ---

@Test
void contextInactive_aboveThreshold_returnsGateRequired() {
    when(principal.groups()).thenThrow(ContextNotActiveException.class);
    // No context → member fallback: 100.0 >= 100.0 → gate
    assertInstanceOf(GateRequired.class, classifier.classify(actionWithAmount(SPEND_PURCHASE, 100.0)));
}

@Test
void contextInactive_belowThreshold_returnsAutonomous() {
    when(principal.groups()).thenThrow(ContextNotActiveException.class);
    // No context → member fallback: 50.0 < 100.0 → autonomous (NOT always-gate)
    assertInstanceOf(Autonomous.class, classifier.classify(actionWithAmount(SPEND_PURCHASE, 50.0)));
}
```

- [ ] **Step 3: Run tests — verify new tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app \
  -Dtest=LifeActionRiskClassifierTest \
  -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: Tests compile. The admin and junior tests **FAIL** (classifier has no RBAC logic yet). The context-inactive tests may pass (existing member-threshold behavior already matches) — that is acceptable; they verify the fallback is preserved post-implementation.

---

## Task 7: LifeActionRiskClassifier — Implement RBAC Changes (GREEN)

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/routing/LifeActionRiskClassifier.java`

- [ ] **Step 1: Replace the full file**

The complete modified file (all changes marked with comments):

```java
package io.casehub.life.app.routing;

import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.RiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.casehub.life.api.HouseholdActionType;
import io.casehub.life.api.HouseholdGroups;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.context.ContextNotActiveException;
import jakarta.inject.Inject;

import java.time.Duration;
import java.util.Map;

/**
 * Life-specific ActionRiskClassifier. Discovered by the engine via @RiskClassifier CDI qualifier.
 * Classifies consequential household agent actions as Autonomous or GateRequired
 * using HouseholdActionType policy plus YAML-configured thresholds.
 *
 * Scope: casehubio/life/oversight (verify mapping against engine#437).
 */
@ApplicationScoped
@RiskClassifier
public class LifeActionRiskClassifier implements ActionRiskClassifier {

    private static final String OVERSIGHT_SCOPE = "casehubio/life/oversight";
    private static final SettingsScope RISK_POLICY_SCOPE =
        SettingsScope.of("casehubio", "life", "risk-policy");

    @Inject PreferenceProvider preferenceProvider;
    @Inject CurrentPrincipal principal;

    @Override
    public RiskDecision classify(PlannedAction action) {
        return HouseholdActionType.fromActionType(action.actionType())
            .map(type -> classifyKnownType(type, action))
            .orElse(new RiskDecision.Autonomous());
    }

    private RiskDecision classifyKnownType(HouseholdActionType type, PlannedAction action) {
        return switch (type.gatePolicy()) {
            case ALWAYS           -> buildGate(type, action, preferenceProvider.resolve(RISK_POLICY_SCOPE));
            case NEVER            -> new RiskDecision.Autonomous();
            case AMOUNT_THRESHOLD -> isJunior()
                ? buildGate(type, action, preferenceProvider.resolve(RISK_POLICY_SCOPE))
                : classifyByAmount(type, action);
        };
    }

    private RiskDecision classifyByAmount(HouseholdActionType type, PlannedAction action) {
        Object raw = action.context().get("amount");
        if (raw == null) return new RiskDecision.Autonomous();
        double amount;
        try {
            amount = Double.parseDouble(raw.toString());
        } catch (NumberFormatException e) {
            return new RiskDecision.Autonomous();
        }
        Preferences prefs = preferenceProvider.resolve(RISK_POLICY_SCOPE);
        double threshold = resolveThreshold(type, prefs);
        return amount >= threshold ? buildGate(type, action, prefs) : new RiskDecision.Autonomous();
    }

    // ThresholdCategory removed from HouseholdActionType in #27 — switch directly on type.
    // This method is interim: replaced by per-type HouseholdRiskRule implementations in Plan B.
    private double resolveThreshold(HouseholdActionType type, Preferences prefs) {
        boolean admin = isAdmin();
        return switch (type) {
            case SPEND_PURCHASE, SPEND_SUBSCRIPTION_MODIFY ->
                prefs.get(admin ? LifeRiskPolicyKeys.ADMIN_SPEND_THRESHOLD
                                : LifeRiskPolicyKeys.SPEND_THRESHOLD).value();
            case BOOKING_REFUNDABLE ->
                prefs.get(admin ? LifeRiskPolicyKeys.ADMIN_BOOKING_THRESHOLD
                                : LifeRiskPolicyKeys.BOOKING_THRESHOLD).value();
            case CONTRACTOR_ENGAGE ->
                prefs.get(admin ? LifeRiskPolicyKeys.ADMIN_CONTRACTOR_THRESHOLD
                                : LifeRiskPolicyKeys.CONTRACTOR_THRESHOLD).value();
            default -> throw new IllegalStateException(
                "resolveThreshold called for non-AMOUNT_THRESHOLD type: " + type);
        };
    }

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
            // act autonomously. The JUNIOR constant is used in @RolesAllowed; the classifier
            // uses the negative form so any non-admin, non-member identity gets the same
            // restrictive treatment.
            return !principal.groups().contains(HouseholdGroups.ADMIN)
                && !principal.groups().contains(HouseholdGroups.MEMBER);
        } catch (ContextNotActiveException e) {
            return false;
        }
    }

    private RiskDecision.GateRequired buildGate(HouseholdActionType type, PlannedAction action, Preferences prefs) {
        long hours = (long) prefs.get(LifeRiskPolicyKeys.APPROVAL_EXPIRES_HOURS).value();
        return new RiskDecision.GateRequired(
            buildReason(type, action),
            type.reversible(),
            type.candidateGroups(),
            Duration.ofHours(hours),
            OVERSIGHT_SCOPE
        );
    }

    private String buildReason(HouseholdActionType type, PlannedAction action) {
        String amt = formatAmount(action.context());
        return switch (type) {
            case SPEND_PURCHASE, SPEND_SUBSCRIPTION_MODIFY ->
                "Spend of " + amt + " requires household approval";
            case SPEND_SUBSCRIPTION_CANCEL ->
                "Subscription cancellation — confirm before proceeding";
            case BOOKING_NONREFUNDABLE -> {
                String amtStr = action.context().containsKey("amount") ? " of " + amt : "";
                yield "Non-refundable booking" + amtStr + " — cannot be undone once confirmed";
            }
            case BOOKING_REFUNDABLE ->
                "Refundable booking of " + amt + " requires household approval";
            case HEALTH_APPOINTMENT_SPECIALIST ->
                "Specialist appointment referral — confirm before booking";
            case HEALTH_APPOINTMENT_GP ->
                throw new IllegalStateException("buildReason called for non-gated type: " + type);
            case HEALTH_MEDICATION_FLAG ->
                "Medication concern — family awareness required before any action";
            case CONTRACTOR_ENGAGE ->
                "Contractor instruction estimated at " + amt + " — approval required";
            case LEGAL_DOCUMENT_SUBMIT ->
                "Legal document submission — confirm before filing (irreversible)";
            case ELDER_CARE_DECISION ->
                "Care decision for dependent — family approval required";
        };
    }

    private String formatAmount(Map<String, Object> context) {
        Object amount   = context.get("amount");
        Object currency = context.getOrDefault("currency", "GBP");
        return amount != null ? currency + " " + amount : "unspecified amount";
    }
}
```

- [ ] **Step 2: Run LifeActionRiskClassifierTest — verify all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app \
  -Dtest=LifeActionRiskClassifierTest \
  -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All tests pass including the 7 new RBAC cases.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add \
  app/src/main/java/io/casehub/life/app/routing/LifeActionRiskClassifier.java \
  app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#26): RBAC-differentiated thresholds in LifeActionRiskClassifier — admin elevated, junior always-gate, context-inactive fallback"
```

---

## Task 8: LifeActionRiskClassifierQuarkusTest — Fix + Add End-to-End RBAC Cases

**Files:**
- Modify: `app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierQuarkusTest.java`

Without this fix: `FixedCurrentPrincipal` defaults to empty groups → `isJunior()=true` → `spendPurchase_belowYamlThreshold_returnsAutonomous` receives `GateRequired` → test fails.

- [ ] **Step 1: Add FixedCurrentPrincipal injection and @BeforeEach/@AfterEach**

Add import:
```java
import io.casehub.life.api.HouseholdGroups;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
```

Add field injection and lifecycle methods:
```java
@Inject
FixedCurrentPrincipal fixedPrincipal;  // @ApplicationScoped — directly injectable

@BeforeEach
void setMemberPrincipal() {
    fixedPrincipal.setGroups(Set.of(HouseholdGroups.MEMBER));
}

@AfterEach
void resetPrincipal() {
    fixedPrincipal.reset();
}
```

Also add `import java.util.Set;` if not already present.

- [ ] **Step 2: Add RBAC-specific end-to-end test cases**

Append to the class (before the closing `}`):

```java
// --- RBAC: admin elevated threshold (end-to-end through YAML) ---

@Test
void admin_spendPurchase_belowAdminYamlThreshold_returnsAutonomous() {
    fixedPrincipal.setGroups(Set.of(HouseholdGroups.ADMIN));
    // Admin threshold from YAML: 500.0. 400.0 is below that.
    PlannedAction action = PlannedAction.of(
        "large purchase", SPEND_PURCHASE.actionType(), Map.of("amount", "400.0"));
    assertInstanceOf(RiskDecision.Autonomous.class, classifier.classify(action));
}

@Test
void admin_spendPurchase_atAdminYamlThreshold_returnsGateRequired() {
    fixedPrincipal.setGroups(Set.of(HouseholdGroups.ADMIN));
    PlannedAction action = PlannedAction.of(
        "large purchase", SPEND_PURCHASE.actionType(), Map.of("amount", "500.0"));
    assertInstanceOf(RiskDecision.GateRequired.class, classifier.classify(action));
}

// --- RBAC: junior always gates (end-to-end) ---

@Test
void junior_spendPurchase_belowMemberYamlThreshold_returnsGateRequired() {
    fixedPrincipal.setGroups(Set.of(HouseholdGroups.JUNIOR));
    // Member threshold from YAML: 100.0. 50.0 is below — junior always gates regardless.
    PlannedAction action = PlannedAction.of(
        "small purchase", SPEND_PURCHASE.actionType(), Map.of("amount", "50.0"));
    assertInstanceOf(RiskDecision.GateRequired.class, classifier.classify(action));
}
```

Also add `import java.util.Map;` if not already present.

- [ ] **Step 3: Run LifeActionRiskClassifierQuarkusTest — verify all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app \
  -Dtest=LifeActionRiskClassifierQuarkusTest \
  -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All 8 tests pass (3 original + 3 new RBAC cases + the `riskClassifierInstance_isSatisfied` wiring test).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add \
  app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierQuarkusTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "test(#26): fix LifeActionRiskClassifierQuarkusTest — set member groups in @BeforeEach; add admin/junior RBAC cases"
```

---

## Task 9: LifeRestSecurityTest — Write Failing Security Boundary Tests (RED)

**Files:**
- Create: `app/src/test/java/io/casehub/life/app/security/LifeRestSecurityTest.java`

- [ ] **Step 1: Create the security test class**

```java
package io.casehub.life.app.security;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.not;
import static org.hamcrest.Matchers.in;
import static java.util.List.of;

/**
 * Security boundary tests for all REST resources.
 * Verifies @RolesAllowed enforcement: 401 for unauthenticated, 403 for wrong role.
 * Does NOT test business logic correctness — just auth/authz boundaries.
 * Response body is ignored; only HTTP status is asserted.
 */
@QuarkusTest
class LifeRestSecurityTest {

    private static final int[] FORBIDDEN = {401, 403};

    // ==============================
    // Unauthenticated → 401
    // ==============================

    @Test
    void unauthenticated_postLifeTasks_returns401() {
        given().contentType(ContentType.JSON).body("{}")
            .when().post("/life-tasks")
            .then().statusCode(401);
    }

    @Test
    void unauthenticated_postExternalActors_returns401() {
        given().contentType(ContentType.JSON).body("{}")
            .when().post("/external-actors")
            .then().statusCode(401);
    }

    @Test
    void unauthenticated_getLifeTask_returns401() {
        given()
            .when().get("/life-tasks/" + UUID.randomUUID())
            .then().statusCode(401);
    }

    @Test
    void unauthenticated_deleteExternalActor_returns401() {
        given()
            .when().delete("/external-actors/" + UUID.randomUUID())
            .then().statusCode(401);
    }

    @Test
    void unauthenticated_postLifeCases_returns401() {
        given().contentType(ContentType.JSON).body("{}")
            .when().post("/life-cases")
            .then().statusCode(401);
    }

    // ==============================
    // household-junior — blocked on create/mutate, allowed on GET /life-tasks/{id}
    // ==============================

    @Test
    @TestSecurity(user = "junior", roles = {"household-junior"})
    void junior_postLifeTasks_returns403() {
        given().contentType(ContentType.JSON).body("{}")
            .when().post("/life-tasks")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "junior", roles = {"household-junior"})
    void junior_postLifeCases_returns403() {
        given().contentType(ContentType.JSON).body("{}")
            .when().post("/life-cases")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "junior", roles = {"household-junior"})
    void junior_deleteExternalActor_returns403() {
        given()
            .when().delete("/external-actors/" + UUID.randomUUID())
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "junior", roles = {"household-junior"})
    void junior_getLifeTask_isNotForbidden() {
        // junior has operation-level access to GET /life-tasks/{id} (data filter is life#41)
        // task not found → 404; 401/403 would indicate auth failure
        given()
            .when().get("/life-tasks/" + UUID.randomUUID())
            .then().statusCode(not(in(of(401, 403))));
    }

    // ==============================
    // household-member — blocked on admin-only, allowed on member endpoints
    // ==============================

    @Test
    @TestSecurity(user = "member", roles = {"household-member"})
    void member_deleteExternalActor_returns403() {
        given()
            .when().delete("/external-actors/" + UUID.randomUUID())
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "member", roles = {"household-member"})
    void member_putExternalActor_returns403() {
        given().contentType(ContentType.JSON).body("{}")
            .when().put("/external-actors/" + UUID.randomUUID())
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "member", roles = {"household-member"})
    void member_eraseExternalActorPersonalData_returns403() {
        given()
            .when().delete("/external-actors/" + UUID.randomUUID() + "/personal-data")
            .then().statusCode(403);
    }

    @Test
    @TestSecurity(user = "member", roles = {"household-member"})
    void member_postLifeTasks_isNotForbidden() {
        given().contentType(ContentType.JSON).body("{}")
            .when().post("/life-tasks")
            .then().statusCode(not(in(of(401, 403))));
    }

    @Test
    @TestSecurity(user = "member", roles = {"household-member"})
    void member_postLifeCases_isNotForbidden() {
        given().contentType(ContentType.JSON).body("{}")
            .when().post("/life-cases")
            .then().statusCode(not(in(of(401, 403))));
    }

    // ==============================
    // household-admin — access to everything
    // ==============================

    @Test
    @TestSecurity(user = "admin", roles = {"household-admin"})
    void admin_deleteExternalActor_isNotForbidden() {
        given()
            .when().delete("/external-actors/" + UUID.randomUUID())
            .then().statusCode(not(in(of(401, 403))));
    }

    @Test
    @TestSecurity(user = "admin", roles = {"household-admin"})
    void admin_eraseExternalActorPersonalData_isNotForbidden() {
        given()
            .when().delete("/external-actors/" + UUID.randomUUID() + "/personal-data")
            .then().statusCode(not(in(of(401, 403))));
    }

    @Test
    @TestSecurity(user = "admin", roles = {"household-admin"})
    void admin_putExternalActor_isNotForbidden() {
        given().contentType(ContentType.JSON).body("{}")
            .when().put("/external-actors/" + UUID.randomUUID())
            .then().statusCode(not(in(of(401, 403))));
    }

    @Test
    @TestSecurity(user = "admin", roles = {"household-admin"})
    void admin_postLifeTasks_isNotForbidden() {
        given().contentType(ContentType.JSON).body("{}")
            .when().post("/life-tasks")
            .then().statusCode(not(in(of(401, 403))));
    }
}
```

- [ ] **Step 2: Run security test — verify the auth tests fail (RED)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app \
  -Dtest=LifeRestSecurityTest \
  -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: Tests that check for 401 (unauthenticated) **FAIL** — endpoints currently return 400/404/200 without auth. The 403 tests for junior/member also fail (no auth enforcement yet).

---

## Task 10: Add @RolesAllowed to All REST Resources (GREEN)

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/resource/LifeTaskResource.java`
- Modify: `app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java`
- Modify: `app/src/main/java/io/casehub/life/app/resource/LifeCaseResource.java`
- Modify: `app/src/main/java/io/casehub/life/app/resource/LifeOversightGateResource.java`
- Modify: `app/src/main/java/io/casehub/life/app/resource/LifeCommitmentResource.java`

All resources need this import added:
```java
import io.casehub.life.api.HouseholdGroups;
import jakarta.annotation.security.RolesAllowed;
```

- [ ] **Step 1: LifeTaskResource — add @RolesAllowed**

```java
@POST
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
public Response create(@Valid final CreateLifeTaskRequest req) { ... }

@GET
@Path("/{id}")
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
public LifeTaskResponse get(@PathParam("id") final UUID workItemId) { ... }
```

- [ ] **Step 2: ExternalActorResource — add @RolesAllowed**

```java
@POST
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
public Response create(@Valid final CreateExternalActorRequest req) { ... }

@GET
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
public List<ExternalActorResponse> list(@QueryParam("actorType") final LifeActorType actorType) { ... }

@GET
@Path("/{id}")
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
public Response get(@PathParam("id") final UUID id) { ... }

@PUT
@Path("/{id}")
@RolesAllowed(HouseholdGroups.ADMIN)
public Response update(@PathParam("id") final UUID id, @Valid final UpdateExternalActorRequest req) { ... }

@DELETE
@Path("/{id}")
@RolesAllowed(HouseholdGroups.ADMIN)
public Response delete(@PathParam("id") final UUID id) { ... }

@DELETE
@Path("/{id}/personal-data")
@RolesAllowed(HouseholdGroups.ADMIN)
public Response erasePersonalData(@PathParam("id") final UUID id) { ... }

@GET
@Path("/{id}/tasks")
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
public Response listTasks(@PathParam("id") final UUID id) { ... }
```

- [ ] **Step 3: LifeCaseResource — add @RolesAllowed**

```java
@POST
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
public Response create(CreateLifeCaseRequest request) { ... }
```

- [ ] **Step 4: LifeOversightGateResource — add @RolesAllowed**

```java
@POST
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
public Response requestApproval(@Valid final OversightGateRequest request) { ... }
```

- [ ] **Step 5: LifeCommitmentResource — add @RolesAllowed**

```java
@POST
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
public Response commit(
        @PathParam("id") final UUID workItemId,
        @Valid final CommitmentRequest request) { ... }
```

- [ ] **Step 6: Run LifeRestSecurityTest — verify auth boundary tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app \
  -Dtest=LifeRestSecurityTest \
  -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All security tests **PASS**. (Existing REST test classes will now fail — fixed in Task 11.)

---

## Task 11: Add @TestSecurity to Existing REST Test Classes

**Files:**
- Modify: `app/src/test/java/io/casehub/life/app/ExternalActorResourceTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/ExternalActorGdprResourceTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/LifeTaskResourceTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/LifeCommitmentResourceTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/resource/LifeCaseResourceTest.java`

Each class gets a class-level `@TestSecurity` annotation and the matching import. The annotation runs all test methods in the class with an authenticated admin identity — no per-method changes needed.

- [ ] **Step 1: Add import to each class**

```java
import io.quarkus.test.security.TestSecurity;
```

- [ ] **Step 2: Add class-level annotation to each class**

On the test class declaration (after `@QuarkusTest`):
```java
@QuarkusTest
@TestSecurity(user = "household-admin", roles = {"household-admin"})
class ExternalActorResourceTest {
```

Apply the same pattern to all five classes:
- `ExternalActorResourceTest`
- `ExternalActorGdprResourceTest`
- `LifeTaskResourceTest`
- `LifeCommitmentResourceTest`
- `LifeCaseResourceTest`

- [ ] **Step 3: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app
```

Expected: All tests pass. If any test fails, investigate — do not proceed to commit with failures.

- [ ] **Step 4: Commit all resource and test changes**

```bash
git -C /Users/mdproctor/claude/casehub/life add \
  app/src/main/java/io/casehub/life/app/resource/LifeTaskResource.java \
  app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java \
  app/src/main/java/io/casehub/life/app/resource/LifeCaseResource.java \
  app/src/main/java/io/casehub/life/app/resource/LifeOversightGateResource.java \
  app/src/main/java/io/casehub/life/app/resource/LifeCommitmentResource.java \
  app/src/test/java/io/casehub/life/app/security/LifeRestSecurityTest.java \
  app/src/test/java/io/casehub/life/app/ExternalActorResourceTest.java \
  app/src/test/java/io/casehub/life/app/ExternalActorGdprResourceTest.java \
  app/src/test/java/io/casehub/life/app/LifeTaskResourceTest.java \
  app/src/test/java/io/casehub/life/app/LifeCommitmentResourceTest.java \
  app/src/test/java/io/casehub/life/app/resource/LifeCaseResourceTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#40): add @RolesAllowed to all REST resources + @TestSecurity on existing tests"
```

---

## Self-Review: Spec Coverage

| Spec section | Covered by |
|---|---|
| §1 casehub-platform-oidc compile dep | Task 1 |
| §1 quarkus-test-security test dep | Task 1 |
| §2 TenantScopedPrincipal excluded from production | Task 2 |
| §3 quarkus.oidc.application-type=service | Task 2 |
| §3 Dev profile: auth.enabled-in-dev-mode=false, oidc.enabled=false, devservices.disabled | Task 2 |
| §3 Test config: discovery-disabled, jwks-path, devservices disabled | Task 3 |
| §4 @RolesAllowed on 12 endpoints across 5 resources | Tasks 9, 10 |
| §5 ADMIN_SPEND/CONTRACTOR/BOOKING_THRESHOLD keys | Task 4 |
| §5 risk-policy.yaml admin threshold values | Task 5 |
| §5 isAdmin() / isJunior() with ContextNotActiveException | Task 7 |
| §5 classifyKnownType() junior path | Task 7 |
| §5 resolveThreshold() admin/member selection | Task 7 |
| §6 LifeActionRiskClassifierTest: @Mock CurrentPrincipal, 7 new RBAC cases | Task 6 |
| §6 LifeActionRiskClassifierQuarkusTest: FixedCurrentPrincipal fix, @Before/@AfterEach, admin/junior cases | Task 8 |
| §6 Existing REST tests: @TestSecurity | Task 11 |
| §6 New LifeRestSecurityTest | Task 9 |
