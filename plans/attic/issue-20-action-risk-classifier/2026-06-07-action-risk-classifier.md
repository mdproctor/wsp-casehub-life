# ActionRiskClassifier Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `LifeActionRiskClassifier` — the life-specific `ActionRiskClassifier` that gates consequential household agent actions behind a human approval step, driven by a configurable YAML risk policy.

**Architecture:** `HouseholdActionType` enum in `api/` owns the full action taxonomy (gate policy, threshold category, reversible flag, candidate groups). `LifeActionRiskClassifier` in `app/routing/` is a thin dispatcher: parse action type → enum → switch on gate policy → check YAML threshold if needed → return `GateRequired` or `Autonomous`. The engine's `ChainedReactiveActionRiskClassifier` discovers it via `@Inject @RiskClassifier Instance<ActionRiskClassifier>` and takes the most restrictive result across all classifiers.

**Tech Stack:** Java 21, Quarkus 3.32.2, `casehub-engine-api` (`ActionRiskClassifier`, `PlannedAction`, `RiskDecision`, `@RiskClassifier`), `casehub-platform-config` (YAML-backed `PreferenceProvider`), JUnit 5 + Mockito (unit), `@QuarkusTest` + H2 (integration).

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `api/src/main/java/io/casehub/life/api/HouseholdGroups.java` | Create | Group name string constants |
| `api/src/main/java/io/casehub/life/api/HouseholdActionType.java` | Create | Action type enum — full taxonomy |
| `api/src/test/java/io/casehub/life/api/HouseholdActionTypeTest.java` | Create | Enum unit tests |
| `app/src/main/java/io/casehub/life/app/routing/LifeRiskPolicyKeys.java` | Create | `PreferenceKey` constants |
| `app/src/main/resources/casehub/life/risk-policy.yaml` | Create | Household thresholds + approval expiry |
| `app/src/main/resources/application.properties` | Modify | Add `risk-policy.yaml` to `casehub.platform.config.files` |
| `app/src/test/resources/application.properties` | Modify | Same — test profile must load the YAML too |
| `app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierTest.java` | Create | Unit tests (no Quarkus) |
| `app/src/main/java/io/casehub/life/app/routing/LifeActionRiskClassifier.java` | Create | Classifier implementation |
| `app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierQuarkusTest.java` | Create | CDI + YAML wiring test |

---

## Task 1: `HouseholdGroups` — group name constants

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/HouseholdGroups.java`

- [ ] **Create `HouseholdGroups.java`**

```java
package io.casehub.life.api;

/**
 * CDI group names used in household approval routing.
 * Matched against candidateGroups in RiskDecision.GateRequired.
 */
public final class HouseholdGroups {
    public static final String ADMIN  = "household-admin";
    public static final String MEMBER = "household-member";
    public static final String JUNIOR = "household-junior";

    private HouseholdGroups() {}
}
```

- [ ] **Build api to verify it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api
```

Expected: `BUILD SUCCESS`

- [ ] **Commit**

```bash
git add api/src/main/java/io/casehub/life/api/HouseholdGroups.java
git commit -m "feat(#20): HouseholdGroups — household approval group name constants

Refs #20"
```

---

## Task 2: `HouseholdActionType` enum + tests

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/HouseholdActionType.java`
- Create: `api/src/test/java/io/casehub/life/api/HouseholdActionTypeTest.java`

- [ ] **Write the failing test**

```java
package io.casehub.life.api;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import java.util.Optional;

import static io.casehub.life.api.HouseholdActionType.*;
import static org.junit.jupiter.api.Assertions.*;

class HouseholdActionTypeTest {

    @ParameterizedTest
    @EnumSource(HouseholdActionType.class)
    void actionType_roundTrips(HouseholdActionType type) {
        String s = type.actionType();
        Optional<HouseholdActionType> parsed = HouseholdActionType.fromActionType(s);
        assertTrue(parsed.isPresent(), "round-trip failed for: " + s);
        assertEquals(type, parsed.get());
    }

    @Test
    void fromActionType_null_returnsEmpty() {
        assertTrue(HouseholdActionType.fromActionType(null).isEmpty());
    }

    @Test
    void fromActionType_unknown_returnsEmpty() {
        assertTrue(HouseholdActionType.fromActionType("foo.bar").isEmpty());
    }

    @Test
    void spendPurchase_actionTypeString() {
        assertEquals("spend.purchase", SPEND_PURCHASE.actionType());
    }

    @Test
    void bookingNonrefundable_actionTypeString() {
        assertEquals("booking.nonrefundable", BOOKING_NONREFUNDABLE.actionType());
    }

    @Test
    void healthMedicationFlag_actionTypeString() {
        assertEquals("health.medication.flag", HEALTH_MEDICATION_FLAG.actionType());
    }

    @Test
    void bookingNonrefundable_isIrreversible() {
        assertFalse(BOOKING_NONREFUNDABLE.reversible());
    }

    @Test
    void legalDocumentSubmit_isIrreversible() {
        assertFalse(LEGAL_DOCUMENT_SUBMIT.reversible());
    }

    @Test
    void healthMedicationFlag_isIrreversible() {
        assertFalse(HEALTH_MEDICATION_FLAG.reversible());
    }

    @Test
    void healthMedicationFlag_candidateGroupsIncludesMember() {
        assertTrue(HEALTH_MEDICATION_FLAG.candidateGroups().contains(HouseholdGroups.MEMBER));
    }

    @Test
    void elderCareDecision_candidateGroupsIncludesMember() {
        assertTrue(ELDER_CARE_DECISION.candidateGroups().contains(HouseholdGroups.MEMBER));
    }

    @Test
    void healthAppointmentGp_isNeverGated() {
        assertEquals(HouseholdActionType.GatePolicy.NEVER, HEALTH_APPOINTMENT_GP.gatePolicy());
    }

    @Test
    void spendPurchase_usesSpendThresholdCategory() {
        assertEquals(HouseholdActionType.ThresholdCategory.SPEND, SPEND_PURCHASE.thresholdCategory());
    }

    @Test
    void contractorEngage_usesContractorThresholdCategory() {
        assertEquals(HouseholdActionType.ThresholdCategory.CONTRACTOR, CONTRACTOR_ENGAGE.thresholdCategory());
    }

    @Test
    void bookingRefundable_usesBookingThresholdCategory() {
        assertEquals(HouseholdActionType.ThresholdCategory.BOOKING, BOOKING_REFUNDABLE.thresholdCategory());
    }

    @Test
    void spendSubscriptionModify_usesSpendThresholdCategory() {
        assertEquals(HouseholdActionType.ThresholdCategory.SPEND, SPEND_SUBSCRIPTION_MODIFY.thresholdCategory());
    }
}
```

- [ ] **Run test to confirm it fails (class missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api --batch-mode -Dtest=HouseholdActionTypeTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5
```

Expected: `COMPILATION ERROR` or `BUILD FAILURE`

- [ ] **Create `HouseholdActionType.java`**

```java
package io.casehub.life.api;

import java.util.List;
import java.util.Optional;

/**
 * Typed taxonomy of consequential household actions declared by workers before execution.
 * Workers use actionType() when constructing PlannedAction; fromActionType() reverses the mapping.
 * Each constant encodes its inherent domain properties — gatePolicy, thresholdCategory,
 * reversible, candidateGroups — so all logic for a type lives here.
 */
public enum HouseholdActionType {

    SPEND_PURCHASE(
        GatePolicy.AMOUNT_THRESHOLD, ThresholdCategory.SPEND, true,
        List.of(HouseholdGroups.ADMIN)),

    SPEND_SUBSCRIPTION_CANCEL(
        GatePolicy.ALWAYS, null, true,
        List.of(HouseholdGroups.ADMIN)),

    SPEND_SUBSCRIPTION_MODIFY(
        GatePolicy.AMOUNT_THRESHOLD, ThresholdCategory.SPEND, true,
        List.of(HouseholdGroups.ADMIN)),

    BOOKING_NONREFUNDABLE(
        GatePolicy.ALWAYS, null, false,
        List.of(HouseholdGroups.ADMIN)),

    BOOKING_REFUNDABLE(
        GatePolicy.AMOUNT_THRESHOLD, ThresholdCategory.BOOKING, true,
        List.of(HouseholdGroups.ADMIN)),

    HEALTH_APPOINTMENT_SPECIALIST(
        GatePolicy.ALWAYS, null, true,
        List.of(HouseholdGroups.ADMIN)),

    /** Routine GP booking — no gate required. */
    HEALTH_APPOINTMENT_GP(
        GatePolicy.NEVER, null, true,
        List.of()),

    /** Medication interaction — irreversible safety concern; any adult can approve (speed matters). */
    HEALTH_MEDICATION_FLAG(
        GatePolicy.ALWAYS, null, false,
        List.of(HouseholdGroups.ADMIN, HouseholdGroups.MEMBER)),

    CONTRACTOR_ENGAGE(
        GatePolicy.AMOUNT_THRESHOLD, ThresholdCategory.CONTRACTOR, true,
        List.of(HouseholdGroups.ADMIN)),

    LEGAL_DOCUMENT_SUBMIT(
        GatePolicy.ALWAYS, null, false,
        List.of(HouseholdGroups.ADMIN)),

    /** Care decision for a dependent — any adult can approve (urgency matters). */
    ELDER_CARE_DECISION(
        GatePolicy.ALWAYS, null, true,
        List.of(HouseholdGroups.ADMIN, HouseholdGroups.MEMBER));

    public enum GatePolicy {
        ALWAYS,           // unconditional gate
        AMOUNT_THRESHOLD, // gate when context["amount"] >= configured threshold
        NEVER             // always autonomous
    }

    /**
     * Maps AMOUNT_THRESHOLD types to their threshold config category.
     * Null for ALWAYS and NEVER types — no threshold applies.
     */
    public enum ThresholdCategory { SPEND, BOOKING, CONTRACTOR }

    private final GatePolicy gatePolicy;
    private final ThresholdCategory thresholdCategory;
    private final boolean reversible;
    private final List<String> candidateGroups;

    HouseholdActionType(GatePolicy gatePolicy, ThresholdCategory thresholdCategory,
                        boolean reversible, List<String> candidateGroups) {
        this.gatePolicy = gatePolicy;
        this.thresholdCategory = thresholdCategory;
        this.reversible = reversible;
        this.candidateGroups = List.copyOf(candidateGroups);
    }

    public GatePolicy gatePolicy() { return gatePolicy; }

    /** Null for ALWAYS and NEVER types. */
    public ThresholdCategory thresholdCategory() { return thresholdCategory; }

    public boolean reversible() { return reversible; }

    public List<String> candidateGroups() { return candidateGroups; }

    /** The actionType string for PlannedAction.actionType(). e.g. SPEND_PURCHASE → "spend.purchase" */
    public String actionType() {
        return name().toLowerCase().replace('_', '.');
    }

    /** Parse a PlannedAction.actionType() string back to enum. Empty if unknown. */
    public static Optional<HouseholdActionType> fromActionType(String actionType) {
        if (actionType == null) return Optional.empty();
        try {
            return Optional.of(valueOf(actionType.toUpperCase().replace('.', '_')));
        } catch (IllegalArgumentException e) {
            return Optional.empty();
        }
    }
}
```

- [ ] **Run tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api --batch-mode -Dtest=HouseholdActionTypeTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`, `Tests run: 16, Failures: 0`

- [ ] **Install api so app can compile against it**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api
```

Expected: `BUILD SUCCESS`

- [ ] **Commit**

```bash
git add api/src/main/java/io/casehub/life/api/HouseholdActionType.java \
        api/src/test/java/io/casehub/life/api/HouseholdActionTypeTest.java
git commit -m "feat(#20): HouseholdActionType enum — household risk action taxonomy

gatePolicy / thresholdCategory / reversible / candidateGroups per action type.
actionType() / fromActionType() provide type-safe PlannedAction construction.

Refs #20"
```

---

## Task 3: `LifeRiskPolicyKeys`, `risk-policy.yaml`, `application.properties`

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/routing/LifeRiskPolicyKeys.java`
- Create: `app/src/main/resources/casehub/life/risk-policy.yaml`
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/resources/application.properties`

- [ ] **Create `LifeRiskPolicyKeys.java`**

```java
package io.casehub.life.app.routing;

import io.casehub.platform.api.preferences.PreferenceKey;

/**
 * PreferenceKey constants for household risk policy configuration.
 * Namespace: casehubio.life.risk-policy
 * All amounts are in the household's local currency (default GBP).
 */
public final class LifeRiskPolicyKeys {

    private static final String NS = "casehubio.life.risk-policy";

    /** Purchases and subscription modifications at or above this amount require approval. Default: 100.0. */
    public static final PreferenceKey<DoublePreference> SPEND_THRESHOLD =
        new PreferenceKey<>(NS, "spend.threshold", DoublePreference.of(100.0), DoublePreference::parse);

    /** Contractor work instructions at or above this estimated cost require approval. Default: 200.0. */
    public static final PreferenceKey<DoublePreference> CONTRACTOR_THRESHOLD =
        new PreferenceKey<>(NS, "contractor.threshold", DoublePreference.of(200.0), DoublePreference::parse);

    /** Refundable bookings at or above this amount require approval. Default: 150.0. */
    public static final PreferenceKey<DoublePreference> BOOKING_THRESHOLD =
        new PreferenceKey<>(NS, "booking.threshold", DoublePreference.of(150.0), DoublePreference::parse);

    /** Hours before an unanswered approval gate expires. Default: 24.0. */
    public static final PreferenceKey<DoublePreference> APPROVAL_EXPIRES_HOURS =
        new PreferenceKey<>(NS, "approval.expires-hours", DoublePreference.of(24.0), DoublePreference::parse);

    private LifeRiskPolicyKeys() {}
}
```

- [ ] **Create `risk-policy.yaml`**

```yaml
entries:
  - scope: casehubio/life/risk-policy
    casehubio.life.risk-policy.spend.threshold: "100.0"
    casehubio.life.risk-policy.contractor.threshold: "200.0"
    casehubio.life.risk-policy.booking.threshold: "150.0"
    casehubio.life.risk-policy.approval.expires-hours: "24.0"
```

- [ ] **Update `app/src/main/resources/application.properties`**

Find the line:
```
casehub.platform.config.files=classpath:casehub/life/trust-routing.yaml
```

Replace with:
```
casehub.platform.config.files=classpath:casehub/life/trust-routing.yaml,classpath:casehub/life/risk-policy.yaml
```

- [ ] **Update `app/src/test/resources/application.properties`**

Same change — find and replace the `casehub.platform.config.files` line:
```
casehub.platform.config.files=classpath:casehub/life/trust-routing.yaml,classpath:casehub/life/risk-policy.yaml
```

- [ ] **Compile app to verify no errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app --batch-mode 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`

- [ ] **Commit**

```bash
git add app/src/main/java/io/casehub/life/app/routing/LifeRiskPolicyKeys.java \
        app/src/main/resources/casehub/life/risk-policy.yaml \
        app/src/main/resources/application.properties \
        app/src/test/resources/application.properties
git commit -m "feat(#20): LifeRiskPolicyKeys + risk-policy.yaml — household approval thresholds

spend=100, contractor=200, booking=150, approval-expiry=24h.
Registered in casehub.platform.config.files for both main and test profiles.

Refs #20"
```

---

## Task 4: Write failing unit tests for `LifeActionRiskClassifier`

**Files:**
- Create: `app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierTest.java`

- [ ] **Create `LifeActionRiskClassifierTest.java`**

```java
package io.casehub.life.app.routing;

import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.RiskDecision;
import io.casehub.api.spi.RiskDecision.Autonomous;
import io.casehub.api.spi.RiskDecision.GateRequired;
import io.casehub.life.api.HouseholdActionType;
import io.casehub.life.api.HouseholdGroups;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Duration;
import java.util.Map;

import static io.casehub.life.api.HouseholdActionType.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class LifeActionRiskClassifierTest {

    @Mock
    private PreferenceProvider preferenceProvider;

    @InjectMocks
    private LifeActionRiskClassifier classifier;

    @BeforeEach
    void setUp() {
        Preferences prefs = mock(Preferences.class);
        when(preferenceProvider.resolve(any(SettingsScope.class))).thenReturn(prefs);
        when(prefs.get(LifeRiskPolicyKeys.SPEND_THRESHOLD)).thenReturn(DoublePreference.of(100.0));
        when(prefs.get(LifeRiskPolicyKeys.CONTRACTOR_THRESHOLD)).thenReturn(DoublePreference.of(200.0));
        when(prefs.get(LifeRiskPolicyKeys.BOOKING_THRESHOLD)).thenReturn(DoublePreference.of(150.0));
        when(prefs.get(LifeRiskPolicyKeys.APPROVAL_EXPIRES_HOURS)).thenReturn(DoublePreference.of(24.0));
    }

    // --- helpers ---

    private PlannedAction action(HouseholdActionType type) {
        return PlannedAction.of("test", type.actionType(), Map.of());
    }

    private PlannedAction actionWithAmount(HouseholdActionType type, double amount) {
        return PlannedAction.of("test", type.actionType(), Map.of("amount", String.valueOf(amount)));
    }

    // --- ALWAYS gate types ---

    @Test
    void spendSubscriptionCancel_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(action(SPEND_SUBSCRIPTION_CANCEL)));
    }

    @Test
    void bookingNonrefundable_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(action(BOOKING_NONREFUNDABLE)));
    }

    @Test
    void healthAppointmentSpecialist_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(action(HEALTH_APPOINTMENT_SPECIALIST)));
    }

    @Test
    void healthMedicationFlag_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(action(HEALTH_MEDICATION_FLAG)));
    }

    @Test
    void legalDocumentSubmit_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(action(LEGAL_DOCUMENT_SUBMIT)));
    }

    @Test
    void elderCareDecision_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(action(ELDER_CARE_DECISION)));
    }

    // --- NEVER gate ---

    @Test
    void healthAppointmentGp_returnsAutonomous() {
        assertInstanceOf(Autonomous.class, classifier.classify(action(HEALTH_APPOINTMENT_GP)));
    }

    // --- reversible ---

    @Test
    void bookingNonrefundable_isIrreversible() {
        GateRequired result = (GateRequired) classifier.classify(action(BOOKING_NONREFUNDABLE));
        assertFalse(result.reversible());
    }

    @Test
    void legalDocumentSubmit_isIrreversible() {
        GateRequired result = (GateRequired) classifier.classify(action(LEGAL_DOCUMENT_SUBMIT));
        assertFalse(result.reversible());
    }

    @Test
    void healthMedicationFlag_isIrreversible() {
        GateRequired result = (GateRequired) classifier.classify(action(HEALTH_MEDICATION_FLAG));
        assertFalse(result.reversible());
    }

    // --- candidateGroups ---

    @Test
    void healthMedicationFlag_candidateGroupsIncludesMember() {
        GateRequired result = (GateRequired) classifier.classify(action(HEALTH_MEDICATION_FLAG));
        assertTrue(result.candidateGroups().contains(HouseholdGroups.MEMBER));
    }

    @Test
    void elderCareDecision_candidateGroupsIncludesMember() {
        GateRequired result = (GateRequired) classifier.classify(action(ELDER_CARE_DECISION));
        assertTrue(result.candidateGroups().contains(HouseholdGroups.MEMBER));
    }

    @Test
    void bookingNonrefundable_candidateGroupsAdminOnly() {
        GateRequired result = (GateRequired) classifier.classify(action(BOOKING_NONREFUNDABLE));
        assertEquals(1, result.candidateGroups().size());
        assertEquals(HouseholdGroups.ADMIN, result.candidateGroups().get(0));
    }

    // --- scope and expiry ---

    @Test
    void gateRequired_scopeIsOversightScope() {
        GateRequired result = (GateRequired) classifier.classify(action(BOOKING_NONREFUNDABLE));
        assertEquals("casehubio/life/oversight", result.scope());
    }

    @Test
    void gateRequired_expiresIn24Hours() {
        GateRequired result = (GateRequired) classifier.classify(action(BOOKING_NONREFUNDABLE));
        assertEquals(Duration.ofHours(24), result.expiresIn());
    }

    // --- AMOUNT_THRESHOLD: spend ---

    @Test
    void spendPurchase_belowThreshold_returnsAutonomous() {
        assertInstanceOf(Autonomous.class, classifier.classify(actionWithAmount(SPEND_PURCHASE, 99.99)));
    }

    @Test
    void spendPurchase_atThreshold_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(actionWithAmount(SPEND_PURCHASE, 100.0)));
    }

    @Test
    void spendSubscriptionModify_atThreshold_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(actionWithAmount(SPEND_SUBSCRIPTION_MODIFY, 100.0)));
    }

    // --- AMOUNT_THRESHOLD: contractor ---

    @Test
    void contractorEngage_belowThreshold_returnsAutonomous() {
        assertInstanceOf(Autonomous.class, classifier.classify(actionWithAmount(CONTRACTOR_ENGAGE, 199.99)));
    }

    @Test
    void contractorEngage_atThreshold_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(actionWithAmount(CONTRACTOR_ENGAGE, 200.0)));
    }

    // --- AMOUNT_THRESHOLD: booking ---

    @Test
    void bookingRefundable_belowThreshold_returnsAutonomous() {
        assertInstanceOf(Autonomous.class, classifier.classify(actionWithAmount(BOOKING_REFUNDABLE, 149.99)));
    }

    @Test
    void bookingRefundable_atThreshold_returnsGateRequired() {
        assertInstanceOf(GateRequired.class, classifier.classify(actionWithAmount(BOOKING_REFUNDABLE, 150.0)));
    }

    // --- missing / bad amount ---

    @Test
    void spendPurchase_missingAmount_returnsAutonomous() {
        assertInstanceOf(Autonomous.class, classifier.classify(action(SPEND_PURCHASE)));
    }

    @Test
    void spendPurchase_unparsableAmount_returnsAutonomous() {
        PlannedAction bad = PlannedAction.of("test", SPEND_PURCHASE.actionType(),
            Map.of("amount", "not-a-number"));
        assertInstanceOf(Autonomous.class, classifier.classify(bad));
    }

    // --- unknown / null actionType ---

    @Test
    void unknownActionType_returnsAutonomous() {
        PlannedAction unknown = PlannedAction.of("test", "foo.bar", Map.of());
        assertInstanceOf(Autonomous.class, classifier.classify(unknown));
    }

    @Test
    void nullActionType_returnsAutonomous() {
        PlannedAction nullType = PlannedAction.of("test", null, Map.of());
        assertInstanceOf(Autonomous.class, classifier.classify(nullType));
    }
}
```

- [ ] **Run tests to confirm they fail (class missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode \
  -Dtest=LifeActionRiskClassifierTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5
```

Expected: `COMPILATION ERROR` — `LifeActionRiskClassifier` does not exist yet.

- [ ] **Commit the failing tests**

```bash
git add app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierTest.java
git commit -m "test(#20): LifeActionRiskClassifier unit tests — all failing (red)

Refs #20"
```

---

## Task 5: Implement `LifeActionRiskClassifier` — make tests pass

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/routing/LifeActionRiskClassifier.java`

- [ ] **Create `LifeActionRiskClassifier.java`**

```java
package io.casehub.life.app.routing;

import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.RiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.casehub.life.api.HouseholdActionType;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import jakarta.enterprise.context.ApplicationScoped;
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

    @Inject
    PreferenceProvider preferenceProvider;

    @Override
    public RiskDecision classify(PlannedAction action) {
        return HouseholdActionType.fromActionType(action.actionType())
            .map(type -> classifyKnownType(type, action))
            .orElse(new RiskDecision.Autonomous());
    }

    private RiskDecision classifyKnownType(HouseholdActionType type, PlannedAction action) {
        return switch (type.gatePolicy()) {
            case ALWAYS           -> buildGate(type, action);
            case NEVER            -> new RiskDecision.Autonomous();
            case AMOUNT_THRESHOLD -> classifyByAmount(type, action);
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
        double threshold = resolveThreshold(type.thresholdCategory(), prefs);
        return amount >= threshold ? buildGate(type, action) : new RiskDecision.Autonomous();
    }

    private double resolveThreshold(HouseholdActionType.ThresholdCategory category, Preferences prefs) {
        return switch (category) {
            case SPEND      -> prefs.get(LifeRiskPolicyKeys.SPEND_THRESHOLD).value();
            case BOOKING    -> prefs.get(LifeRiskPolicyKeys.BOOKING_THRESHOLD).value();
            case CONTRACTOR -> prefs.get(LifeRiskPolicyKeys.CONTRACTOR_THRESHOLD).value();
        };
    }

    private RiskDecision.GateRequired buildGate(HouseholdActionType type, PlannedAction action) {
        Preferences prefs = preferenceProvider.resolve(RISK_POLICY_SCOPE);
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
            case BOOKING_NONREFUNDABLE ->
                "Non-refundable booking of " + amt + " — cannot be undone once confirmed";
            case BOOKING_REFUNDABLE ->
                "Refundable booking of " + amt + " requires household approval";
            case HEALTH_APPOINTMENT_SPECIALIST ->
                "Specialist appointment referral — confirm before booking";
            case HEALTH_APPOINTMENT_GP ->
                "";  // NEVER policy — this branch is unreachable
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

- [ ] **Run unit tests to confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode \
  -Dtest=LifeActionRiskClassifierTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -8
```

Expected: `BUILD SUCCESS`, `Tests run: 27, Failures: 0, Errors: 0`

- [ ] **Commit**

```bash
git add app/src/main/java/io/casehub/life/app/routing/LifeActionRiskClassifier.java
git commit -m "feat(#20): LifeActionRiskClassifier — @RiskClassifier household action gate

ALWAYS types: subscription cancel, nonrefundable booking, specialist appointment,
medication flag, legal filing, elder care.
AMOUNT_THRESHOLD types: spend purchase/modify (£100), refundable booking (£150),
contractor engage (£200). YAML thresholds via casehub-platform-config.

Refs #20"
```

---

## Task 6: `@QuarkusTest` — CDI and YAML wiring verification

**Files:**
- Create: `app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierQuarkusTest.java`

- [ ] **Create `LifeActionRiskClassifierQuarkusTest.java`**

```java
package io.casehub.life.app.routing;

import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.RiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static io.casehub.life.api.HouseholdActionType.*;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class LifeActionRiskClassifierQuarkusTest {

    @Inject
    LifeActionRiskClassifier classifier;

    @Inject
    @RiskClassifier
    Instance<ActionRiskClassifier> riskClassifiers;

    @Test
    void riskClassifierInstance_isSatisfied() {
        assertFalse(riskClassifiers.isUnsatisfied(),
            "@RiskClassifier Instance<ActionRiskClassifier> must not be empty");
    }

    @Test
    void alwaysGateType_returnsGateRequired() {
        PlannedAction action = PlannedAction.of(
            "book specialist", HEALTH_APPOINTMENT_SPECIALIST.actionType(), Map.of());
        assertInstanceOf(RiskDecision.GateRequired.class, classifier.classify(action));
    }

    @Test
    void spendPurchase_belowYamlThreshold_returnsAutonomous() {
        // Confirms risk-policy.yaml loaded: threshold is 100.0
        PlannedAction action = PlannedAction.of(
            "buy groceries", SPEND_PURCHASE.actionType(), Map.of("amount", "99.0"));
        assertInstanceOf(RiskDecision.Autonomous.class, classifier.classify(action));
    }

    @Test
    void spendPurchase_atYamlThreshold_returnsGateRequired() {
        PlannedAction action = PlannedAction.of(
            "buy groceries", SPEND_PURCHASE.actionType(), Map.of("amount", "100.0"));
        assertInstanceOf(RiskDecision.GateRequired.class, classifier.classify(action));
    }

    @Test
    void contractorEngage_atYamlThreshold_returnsGateRequired() {
        PlannedAction action = PlannedAction.of(
            "hire plumber", CONTRACTOR_ENGAGE.actionType(), Map.of("amount", "200.0"));
        assertInstanceOf(RiskDecision.GateRequired.class, classifier.classify(action));
    }
}
```

- [ ] **Run integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode \
  -Dtest=LifeActionRiskClassifierQuarkusTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -8
```

Expected: `BUILD SUCCESS`, `Tests run: 5, Failures: 0`

- [ ] **Run the full app test suite to check for regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`. Any pre-existing `LEDGER_SUBJECT_SEQUENCE` failures are tracked in the backlog — confirm no new failures.

- [ ] **Commit**

```bash
git add app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierQuarkusTest.java
git commit -m "test(#20): LifeActionRiskClassifierQuarkusTest — CDI qualifier + YAML wiring

Refs #20"
```

---

## Task 7: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (project repo)

- [ ] **Add Layer 7 additions to CLAUDE.md**

In `CLAUDE.md`, find the `Layer 6:` block inside **Foundation Layers**. Update Layer 6 status line from `🔲 PENDING — implementation complete, not yet merged` to `✅ COMPLETE` (it merged when #22 closed). Then add the Layer 7 block after it:

```
Layer 7 (partial): Action risk classification — LifeActionRiskClassifier intercepts
         consequential worker actions before execution. @RiskClassifier CDI qualifier
         activates via ChainedReactiveActionRiskClassifier. HouseholdActionType enum
         (api/) owns the full action taxonomy: 11 types across 3 gate policies
         (ALWAYS / AMOUNT_THRESHOLD / NEVER). YAML thresholds in risk-policy.yaml
         via casehub-platform-config. RBAC-differentiated thresholds deferred (life#26,
         blocked on auth retrofit). Full Layer 7 = + casehub-openclaw as WorkerProvisioner.
         ✅ COMPLETE (risk classification)  🔲 PENDING (OpenClaw integration)
```

In the **What This Project Owns** section, under **Layer 6 additions**, add a **Layer 7 additions** subsection:

```
**Layer 7 additions (partial — risk classification):**
- `HouseholdActionType` — `api/` enum: 11 action types, `GatePolicy` (ALWAYS/AMOUNT_THRESHOLD/NEVER),
  `ThresholdCategory` (SPEND/BOOKING/CONTRACTOR), `reversible`, `candidateGroups`.
  `actionType()` / `fromActionType()` provide type-safe `PlannedAction` construction.
- `HouseholdGroups` — `api/` string constants: `household-admin`, `household-member`, `household-junior`.
- `LifeRiskPolicyKeys` — `app/routing/` `PreferenceKey` constants. Namespace: `casehubio.life.risk-policy`.
  spend.threshold (100.0), contractor.threshold (200.0), booking.threshold (150.0), approval.expires-hours (24.0).
- `LifeActionRiskClassifier` — `app/routing/` `@ApplicationScoped @RiskClassifier`; implements
  `ActionRiskClassifier` from casehub-engine-api. Discovered by engine's `ChainedReactiveActionRiskClassifier`
  via `@Inject @RiskClassifier Instance<ActionRiskClassifier>`.
- `risk-policy.yaml` — YAML config at `casehub/life/risk-policy.yaml`; single scope `casehubio/life/risk-policy`.
- scope convention: `"casehubio/life/oversight"` — verify against engine#437 once engine docs clarify scope→channel mapping.
```

- [ ] **Build to confirm CLAUDE.md change doesn't break anything**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api 2>&1 | tail -3
```

Expected: `BUILD SUCCESS`

- [ ] **Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#20): update CLAUDE.md — Layer 6 complete, Layer 7 risk classification partial

Refs #20"
```

---

## Self-Review Checklist

- [x] **Spec coverage:** `HouseholdGroups` ✅ Task 1. `HouseholdActionType` ✅ Task 2. `LifeRiskPolicyKeys` + YAML ✅ Task 3. Unit tests ✅ Task 4. Classifier ✅ Task 5. QuarkusTest ✅ Task 6. CLAUDE.md ✅ Task 7.
- [x] **No placeholders:** All code blocks are complete. No TBDs.
- [x] **Type consistency:** `DoublePreference` reused from `app/routing/` (same package, no import issues). `SettingsScope.of(String...)` matches decompiled signature. `@RiskClassifier` qualifier from `io.casehub.api.spi.RiskClassifier`. `PlannedAction.of(description, actionType, context)` matches decompiled factory.
- [x] **application.properties:** Both main and test files updated in Task 3.
- [x] **Pre-existing test failures:** `LEDGER_SUBJECT_SEQUENCE` failures are backlog items — Task 6 explicitly notes to confirm no new failures appear.
