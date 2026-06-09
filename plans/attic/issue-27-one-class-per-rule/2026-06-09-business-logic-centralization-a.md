# Business Logic Centralization — Plan A
# Domain Descriptors · Ledger Handlers · Trust Routing · SLA Policy

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Centralize domain business logic into POJO descriptors and CDI handler beans, eliminating all switch statements and static maps from service/observer classes.

**Architecture:** Each `LifeDomain` enum value carries a `LifeDomainDescriptor` POJO (pure Java, no framework) encoding capability, routing policy, SLA policy, and worker capabilities. CDI `DomainLedgerHandler` beans supplement descriptors with ledger-writing behaviour. Service classes (`LifeDecisionLedgerObserver`, `LifeTaskService`, `LifeTrustRoutingPolicyProvider`, `LifeSlaBreachPolicy`, etc.) become thin dispatchers with no domain knowledge.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, Mockito, Maven (`mvn -pl api`, `mvn -pl app`)

**Spec:** `docs/specs/2026-06-08-business-logic-centralization.md`
**Issue:** casehubio/life#27
**Branch:** `issue-27-one-class-per-rule`

---

## File Map

### New files — `api/`
```
api/src/main/java/io/casehub/life/api/LifeSlaPolicy.java
api/src/main/java/io/casehub/life/api/LifeDomainDescriptor.java
api/src/main/java/io/casehub/life/api/LifeRoutingPolicy.java         ← moved from app/routing/
api/src/main/java/io/casehub/life/api/descriptor/HealthDomainDescriptor.java
api/src/main/java/io/casehub/life/api/descriptor/LegalDomainDescriptor.java
api/src/main/java/io/casehub/life/api/descriptor/FinanceDomainDescriptor.java
api/src/main/java/io/casehub/life/api/descriptor/HouseholdDomainDescriptor.java
api/src/main/java/io/casehub/life/api/descriptor/FamilySchedulingDomainDescriptor.java
api/src/main/java/io/casehub/life/api/descriptor/TravelDomainDescriptor.java
api/src/main/java/io/casehub/life/api/descriptor/ContractorCoordinationDomainDescriptor.java
api/src/main/java/io/casehub/life/api/descriptor/ElderCareDomainDescriptor.java
```

### New test files — `api/`
```
api/src/test/java/io/casehub/life/api/descriptor/HealthDomainDescriptorTest.java
api/src/test/java/io/casehub/life/api/descriptor/AllDomainDescriptorsTest.java
```

### Modified files — `api/`
```
api/src/main/java/io/casehub/life/api/LifeDomain.java               ← add descriptor(), fromCategory()
api/src/main/java/io/casehub/life/api/HouseholdActionType.java       ← remove ThresholdCategory
api/src/test/java/io/casehub/life/api/HouseholdActionTypeTest.java   ← remove ThresholdCategory tests
api/src/test/java/io/casehub/life/api/LifeDomainTest.java            ← add fromCategory() tests (new or update)
```

### Deleted files — `app/`
```
app/src/main/java/io/casehub/life/app/routing/LifeRoutingPolicy.java  ← moved to api/
```

### New files — `app/service/ledger/`
```
app/src/main/java/io/casehub/life/app/service/ledger/DomainLedgerHandler.java
app/src/main/java/io/casehub/life/app/service/ledger/HealthDomainLedgerHandler.java
app/src/main/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandler.java
app/src/main/java/io/casehub/life/app/service/ledger/FinanceDomainLedgerHandler.java
```

### New test files — `app/`
```
app/src/test/java/io/casehub/life/app/service/ledger/HealthDomainLedgerHandlerTest.java
app/src/test/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandlerTest.java
app/src/test/java/io/casehub/life/app/service/ledger/FinanceDomainLedgerHandlerTest.java
```

### Modified files — `app/`
```
app/src/main/java/io/casehub/life/app/service/ledger/LifeLedgerWriter.java
app/src/main/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriter.java
app/src/main/java/io/casehub/life/app/observer/LifeDecisionLedgerObserver.java
app/src/main/java/io/casehub/life/app/service/LifeTaskService.java
app/src/main/java/io/casehub/life/app/commitment/OversightGateStrategy.java
app/src/main/java/io/casehub/life/app/observer/LifeWatchdogAlertObserver.java
app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProvider.java
app/src/main/java/io/casehub/life/app/spi/LifeSlaBreachPolicy.java
app/src/test/java/io/casehub/life/app/service/ledger/LifeLedgerWriterTest.java
app/src/test/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriterTest.java
app/src/test/java/io/casehub/life/app/observer/LifeWatchdogAlertObserverTest.java
```

---

## Build commands (run from project root)

```bash
# api/ only
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api

# app/ only (requires api/ installed first)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app

# Single test class in app/
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=ClassName \
  -Dsurefire.failIfNoSpecifiedTests=false --batch-mode
```

---

## Task 1: `LifeSlaPolicy` and `LifeDomainDescriptor` — api/ foundation types

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/LifeSlaPolicy.java`
- Create: `api/src/main/java/io/casehub/life/api/LifeDomainDescriptor.java`
- Move: `app/src/main/java/io/casehub/life/app/routing/LifeRoutingPolicy.java` → `api/src/main/java/io/casehub/life/api/LifeRoutingPolicy.java`

- [ ] **Step 1: Create `LifeSlaPolicy`**

```java
package io.casehub.life.api;

import java.time.Duration;

public record LifeSlaPolicy(String escalationGroup, Duration escalationDeadline) {}
```

- [ ] **Step 2: Move `LifeRoutingPolicy` to `api/`**

Copy `app/src/main/java/io/casehub/life/app/routing/LifeRoutingPolicy.java` to
`api/src/main/java/io/casehub/life/api/LifeRoutingPolicy.java` with updated package:

```java
package io.casehub.life.api;

import java.util.Objects;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;

public record LifeRoutingPolicy(
    OptionalDouble threshold,
    OptionalInt minimumObservations,
    OptionalDouble borderlineMargin,
    Optional<String> fallbackType,
    String rationale
) {
    public LifeRoutingPolicy {
        Objects.requireNonNull(threshold, "threshold must not be null");
        Objects.requireNonNull(minimumObservations, "minimumObservations must not be null");
        Objects.requireNonNull(borderlineMargin, "borderlineMargin must not be null");
        Objects.requireNonNull(fallbackType, "fallbackType must not be null");
        Objects.requireNonNull(rationale, "rationale must not be null");
    }
}
```

Delete `app/src/main/java/io/casehub/life/app/routing/LifeRoutingPolicy.java`.

- [ ] **Step 3: Create `LifeDomainDescriptor`**

```java
package io.casehub.life.api;

import java.util.Set;

public interface LifeDomainDescriptor {
    String capability();
    String templateCategory();
    LifeRoutingPolicy routingPolicy();
    Set<String> workerCapabilities();
    LifeSlaPolicy slaPolicy();
}
```

- [ ] **Step 4: Build api/ to verify it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl api
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Fix `LifeTrustRoutingPolicyProvider` import**

In `app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProvider.java`, update import:
```java
// Remove:
// import io.casehub.life.app.routing.LifeRoutingPolicy;
// Add:
import io.casehub.life.api.LifeRoutingPolicy;
```

- [ ] **Step 6: Build app/ to verify it still compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl app
```

Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/life/api/LifeSlaPolicy.java \
        api/src/main/java/io/casehub/life/api/LifeDomainDescriptor.java \
        api/src/main/java/io/casehub/life/api/LifeRoutingPolicy.java \
        app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProvider.java
git rm app/src/main/java/io/casehub/life/app/routing/LifeRoutingPolicy.java
git commit -m "refactor(#27): move LifeRoutingPolicy to api/, add LifeSlaPolicy and LifeDomainDescriptor"
```

---

## Task 2: Eight domain descriptor POJOs + `LifeDomain` enum update

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/descriptor/` (8 files)
- Modify: `api/src/main/java/io/casehub/life/api/LifeDomain.java`
- Modify: `api/src/main/java/io/casehub/life/api/HouseholdActionType.java`
- Create: `api/src/test/java/io/casehub/life/api/descriptor/HealthDomainDescriptorTest.java`
- Create: `api/src/test/java/io/casehub/life/api/descriptor/AllDomainDescriptorsTest.java`
- Modify: `api/src/test/java/io/casehub/life/api/HouseholdActionTypeTest.java`

- [ ] **Step 1: Write failing test for `HealthDomainDescriptor`**

Create `api/src/test/java/io/casehub/life/api/descriptor/HealthDomainDescriptorTest.java`:

```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeCapabilities;
import org.junit.jupiter.api.Test;
import java.time.Duration;

import static org.junit.jupiter.api.Assertions.*;

class HealthDomainDescriptorTest {

    private final HealthDomainDescriptor descriptor = new HealthDomainDescriptor();

    @Test void capability_isHealthCoordination() {
        assertEquals(LifeCapabilities.HEALTH_COORDINATION, descriptor.capability());
    }

    @Test void templateCategory_isHealth() {
        assertEquals("health", descriptor.templateCategory());
    }

    @Test void routingPolicy_threshold075() {
        assertEquals(0.75, descriptor.routingPolicy().threshold().getAsDouble());
    }

    @Test void routingPolicy_minObservations10() {
        assertEquals(10, descriptor.routingPolicy().minimumObservations().getAsInt());
    }

    @Test void routingPolicy_hasFallback() {
        assertTrue(descriptor.routingPolicy().fallbackType().isPresent());
    }

    @Test void workerCapabilities_containsBookAppointment() {
        assertTrue(descriptor.workerCapabilities().contains("book-appointment"));
    }

    @Test void workerCapabilities_fiveEntries() {
        assertEquals(5, descriptor.workerCapabilities().size());
    }

    @Test void slaPolicy_escalationGroup_isAdmin() {
        assertEquals("household-admin", descriptor.slaPolicy().escalationGroup());
    }

    @Test void slaPolicy_deadline_is24Hours() {
        assertEquals(Duration.ofHours(24), descriptor.slaPolicy().escalationDeadline());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api \
  -Dtest=HealthDomainDescriptorTest --batch-mode
```

Expected: FAIL — `HealthDomainDescriptor` does not exist

- [ ] **Step 3: Create `HealthDomainDescriptor`**

```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomainDescriptor;
import io.casehub.life.api.LifeRoutingPolicy;
import io.casehub.life.api.LifeSlaPolicy;

import java.time.Duration;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import java.util.Set;

public final class HealthDomainDescriptor implements LifeDomainDescriptor {
    @Override public String capability()       { return LifeCapabilities.HEALTH_COORDINATION; }
    @Override public String templateCategory() { return "health"; }

    @Override public Set<String> workerCapabilities() {
        return Set.of("book-appointment", "find-alternative", "confirm-appointment",
                      "pre-visit-prep", "record-health-decision");
    }

    @Override public LifeRoutingPolicy routingPolicy() {
        return new LifeRoutingPolicy(OptionalDouble.of(0.75), OptionalInt.of(10),
                OptionalDouble.of(0.05), Optional.of("household-admin"),
                "High reliability required for health appointments and follow-ups");
    }

    @Override public LifeSlaPolicy slaPolicy() {
        return new LifeSlaPolicy("household-admin", Duration.ofHours(24));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api \
  -Dtest=HealthDomainDescriptorTest --batch-mode
```

Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 5: Create the remaining 7 domain descriptors**

Create each file in `api/src/main/java/io/casehub/life/api/descriptor/`:

**`LegalDomainDescriptor.java`**
```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomainDescriptor;
import io.casehub.life.api.LifeRoutingPolicy;
import io.casehub.life.api.LifeSlaPolicy;

import java.time.Duration;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import java.util.Set;

public final class LegalDomainDescriptor implements LifeDomainDescriptor {
    @Override public String capability()       { return LifeCapabilities.LEGAL_DEADLINE; }
    @Override public String templateCategory() { return "legal"; }
    @Override public Set<String> workerCapabilities() { return Set.of(); }

    @Override public LifeRoutingPolicy routingPolicy() {
        return new LifeRoutingPolicy(OptionalDouble.of(0.80), OptionalInt.of(12),
                OptionalDouble.of(0.05), Optional.of("household-admin"),
                "Critical deadlines with legal consequences require highest reliability");
    }

    @Override public LifeSlaPolicy slaPolicy() {
        return new LifeSlaPolicy("household-admin", Duration.ofHours(12));
    }
}
```

**`FinanceDomainDescriptor.java`**
```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomainDescriptor;
import io.casehub.life.api.LifeRoutingPolicy;
import io.casehub.life.api.LifeSlaPolicy;

import java.time.Duration;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import java.util.Set;

public final class FinanceDomainDescriptor implements LifeDomainDescriptor {
    @Override public String capability()       { return LifeCapabilities.FINANCIAL_PLANNING; }
    @Override public String templateCategory() { return "finance"; }

    @Override public Set<String> workerCapabilities() {
        return Set.of("gather-data", "analyse-anomalies", "escalate-anomalies",
                      "oversight-response", "produce-report");
    }

    @Override public LifeRoutingPolicy routingPolicy() {
        return new LifeRoutingPolicy(OptionalDouble.of(0.70), OptionalInt.of(10),
                OptionalDouble.of(0.10), Optional.of("household-admin"),
                "Financial decisions require cost accuracy but tolerate wider margin");
    }

    @Override public LifeSlaPolicy slaPolicy() {
        return new LifeSlaPolicy("household-admin", Duration.ofHours(48));
    }
}
```

**`HouseholdDomainDescriptor.java`**
```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomainDescriptor;
import io.casehub.life.api.LifeRoutingPolicy;
import io.casehub.life.api.LifeSlaPolicy;

import java.time.Duration;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import java.util.Set;

public final class HouseholdDomainDescriptor implements LifeDomainDescriptor {
    @Override public String capability()       { return LifeCapabilities.HOUSEHOLD_MANAGEMENT; }
    @Override public String templateCategory() { return "household"; }

    @Override public Set<String> workerCapabilities() {
        return Set.of("schedule-inspection", "get-quotes", "issue-commitment",
                      "monitor-job", "record-completion");
    }

    @Override public LifeRoutingPolicy routingPolicy() {
        return new LifeRoutingPolicy(OptionalDouble.of(0.50), OptionalInt.of(5),
                OptionalDouble.empty(), Optional.empty(),
                "Routine household tasks tolerate lower threshold, no escalation");
    }

    @Override public LifeSlaPolicy slaPolicy() {
        return new LifeSlaPolicy("household-admin", Duration.ofHours(48));
    }
}
```

**`FamilySchedulingDomainDescriptor.java`**
```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomainDescriptor;
import io.casehub.life.api.LifeRoutingPolicy;
import io.casehub.life.api.LifeSlaPolicy;

import java.time.Duration;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import java.util.Set;

public final class FamilySchedulingDomainDescriptor implements LifeDomainDescriptor {
    @Override public String capability()       { return LifeCapabilities.FAMILY_SCHEDULING; }
    @Override public String templateCategory() { return "family"; }
    @Override public Set<String> workerCapabilities() { return Set.of(); }

    @Override public LifeRoutingPolicy routingPolicy() {
        return new LifeRoutingPolicy(OptionalDouble.of(0.50), OptionalInt.of(5),
                OptionalDouble.empty(), Optional.empty(),
                "Family calendar coordination is low-stakes");
    }

    @Override public LifeSlaPolicy slaPolicy() {
        return new LifeSlaPolicy("household-admin", Duration.ofHours(48));
    }
}
```

**`TravelDomainDescriptor.java`**
```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomainDescriptor;
import io.casehub.life.api.LifeRoutingPolicy;
import io.casehub.life.api.LifeSlaPolicy;

import java.time.Duration;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import java.util.Set;

public final class TravelDomainDescriptor implements LifeDomainDescriptor {
    @Override public String capability()       { return LifeCapabilities.TRAVEL_PLANNING; }
    @Override public String templateCategory() { return "travel"; }

    @Override public Set<String> workerCapabilities() {
        return Set.of("destination-research", "flight-search", "hotel-search",
                      "budget-assessment", "booking", "rebooking", "confirmation");
    }

    @Override public LifeRoutingPolicy routingPolicy() {
        return new LifeRoutingPolicy(OptionalDouble.of(0.55), OptionalInt.of(6),
                OptionalDouble.of(0.05), Optional.of("household-admin"),
                "Travel research and booking require moderate reliability");
    }

    @Override public LifeSlaPolicy slaPolicy() {
        return new LifeSlaPolicy("household-admin", Duration.ofHours(48));
    }
}
```

**`ContractorCoordinationDomainDescriptor.java`**
```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomainDescriptor;
import io.casehub.life.api.LifeRoutingPolicy;
import io.casehub.life.api.LifeSlaPolicy;

import java.time.Duration;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import java.util.Set;

public final class ContractorCoordinationDomainDescriptor implements LifeDomainDescriptor {
    @Override public String capability()       { return LifeCapabilities.CONTRACTOR_COORDINATION; }
    @Override public String templateCategory() { return "contractor"; }

    @Override public Set<String> workerCapabilities() {
        return Set.of("request-quote", "watchdog-escalation", "quote-received",
                      "job-monitoring", "record-payment");
    }

    @Override public LifeRoutingPolicy routingPolicy() {
        return new LifeRoutingPolicy(OptionalDouble.of(0.65), OptionalInt.of(8),
                OptionalDouble.of(0.05), Optional.of("household-admin"),
                "Contractor follow-up balances deadline reliability and cost accuracy");
    }

    @Override public LifeSlaPolicy slaPolicy() {
        return new LifeSlaPolicy("household-admin", Duration.ofHours(48));
    }
}
```

**`ElderCareDomainDescriptor.java`**
```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomainDescriptor;
import io.casehub.life.api.LifeRoutingPolicy;
import io.casehub.life.api.LifeSlaPolicy;

import java.time.Duration;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import java.util.Set;

public final class ElderCareDomainDescriptor implements LifeDomainDescriptor {
    @Override public String capability()       { return LifeCapabilities.ELDER_CARE; }
    @Override public String templateCategory() { return "elder-care"; }

    @Override public Set<String> workerCapabilities() {
        return Set.of("needs-assessment", "care-plan", "health-check",
                      "assess-patient", "provide-care");
    }

    @Override public LifeRoutingPolicy routingPolicy() {
        return new LifeRoutingPolicy(OptionalDouble.of(0.75), OptionalInt.of(10),
                OptionalDouble.of(0.05), Optional.of("household-admin"),
                "Care coordination requires high reliability and proactive alerting");
    }

    @Override public LifeSlaPolicy slaPolicy() {
        return new LifeSlaPolicy("household-admin", Duration.ofHours(12));
    }
}
```

- [ ] **Step 6: Write `AllDomainDescriptorsTest` — contract tests for all 8 descriptors**

Create `api/src/test/java/io/casehub/life/api/descriptor/AllDomainDescriptorsTest.java`:

```java
package io.casehub.life.api.descriptor;

import io.casehub.life.api.LifeDomainDescriptor;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;

import java.util.stream.Stream;

import static org.junit.jupiter.api.Assertions.*;

class AllDomainDescriptorsTest {

    static Stream<LifeDomainDescriptor> allDescriptors() {
        return Stream.of(
            new HealthDomainDescriptor(),
            new LegalDomainDescriptor(),
            new FinanceDomainDescriptor(),
            new HouseholdDomainDescriptor(),
            new FamilySchedulingDomainDescriptor(),
            new TravelDomainDescriptor(),
            new ContractorCoordinationDomainDescriptor(),
            new ElderCareDomainDescriptor()
        );
    }

    @ParameterizedTest @MethodSource("allDescriptors")
    void capability_notNull(LifeDomainDescriptor d) {
        assertNotNull(d.capability());
        assertFalse(d.capability().isBlank());
    }

    @ParameterizedTest @MethodSource("allDescriptors")
    void templateCategory_notNull(LifeDomainDescriptor d) {
        assertNotNull(d.templateCategory());
        assertFalse(d.templateCategory().isBlank());
    }

    @ParameterizedTest @MethodSource("allDescriptors")
    void workerCapabilities_notNull(LifeDomainDescriptor d) {
        assertNotNull(d.workerCapabilities());
    }

    @ParameterizedTest @MethodSource("allDescriptors")
    void routingPolicy_notNull(LifeDomainDescriptor d) {
        assertNotNull(d.routingPolicy());
        assertTrue(d.routingPolicy().threshold().isPresent());
        assertTrue(d.routingPolicy().minimumObservations().isPresent());
    }

    @ParameterizedTest @MethodSource("allDescriptors")
    void slaPolicy_notNull(LifeDomainDescriptor d) {
        assertNotNull(d.slaPolicy());
        assertNotNull(d.slaPolicy().escalationGroup());
        assertNotNull(d.slaPolicy().escalationDeadline());
    }

    @ParameterizedTest @MethodSource("allDescriptors")
    void householdAndFamilyScheduling_emptyMarginAndFallback(LifeDomainDescriptor d) {
        // HOUSEHOLD and FAMILY_SCHEDULING have no fallback escalation path
        if (d instanceof HouseholdDomainDescriptor || d instanceof FamilySchedulingDomainDescriptor) {
            assertTrue(d.routingPolicy().borderlineMargin().isEmpty());
            assertTrue(d.routingPolicy().fallbackType().isEmpty());
        }
    }
}
```

- [ ] **Step 7: Run `AllDomainDescriptorsTest`**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api \
  -Dtest=AllDomainDescriptorsTest --batch-mode
```

Expected: BUILD SUCCESS

- [ ] **Step 8: Update `LifeDomain` enum**

Replace the entire content of `api/src/main/java/io/casehub/life/api/LifeDomain.java`:

```java
package io.casehub.life.api;

import io.casehub.life.api.descriptor.*;

import java.util.Arrays;
import java.util.Optional;

public enum LifeDomain {
    HEALTH(new HealthDomainDescriptor()),
    FINANCE(new FinanceDomainDescriptor()),
    FAMILY_SCHEDULING(new FamilySchedulingDomainDescriptor()),
    TRAVEL(new TravelDomainDescriptor()),
    LEGAL(new LegalDomainDescriptor()),
    CONTRACTOR_COORDINATION(new ContractorCoordinationDomainDescriptor()),
    ELDER_CARE(new ElderCareDomainDescriptor()),
    HOUSEHOLD(new HouseholdDomainDescriptor());

    private final LifeDomainDescriptor descriptor;

    LifeDomain(LifeDomainDescriptor descriptor) {
        this.descriptor = descriptor;
    }

    public LifeDomainDescriptor descriptor() {
        return descriptor;
    }

    public static Optional<LifeDomain> fromCategory(String category) {
        if (category == null) return Optional.empty();
        return Arrays.stream(values())
                .filter(d -> d.descriptor().templateCategory().equals(category))
                .findFirst();
    }
}
```

- [ ] **Step 9: Write `LifeDomainTest` — test `fromCategory()`**

Create or update `api/src/test/java/io/casehub/life/api/LifeDomainTest.java`:

```java
package io.casehub.life.api;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

class LifeDomainTest {

    @ParameterizedTest
    @EnumSource(LifeDomain.class)
    void allDomains_haveDescriptor(LifeDomain domain) {
        assertNotNull(domain.descriptor());
    }

    @ParameterizedTest
    @EnumSource(LifeDomain.class)
    void fromCategory_roundTrips(LifeDomain domain) {
        Optional<LifeDomain> result = LifeDomain.fromCategory(domain.descriptor().templateCategory());
        assertTrue(result.isPresent(), "fromCategory() should find: " + domain);
        assertEquals(domain, result.get());
    }

    @Test void fromCategory_null_returnsEmpty() {
        assertTrue(LifeDomain.fromCategory(null).isEmpty());
    }

    @Test void fromCategory_unknown_returnsEmpty() {
        assertTrue(LifeDomain.fromCategory("xyz-unknown").isEmpty());
    }

    @Test void fromCategory_health_returnsHealth() {
        assertEquals(Optional.of(LifeDomain.HEALTH), LifeDomain.fromCategory("health"));
    }

    @Test void fromCategory_elderCare_returnsElderCare() {
        assertEquals(Optional.of(LifeDomain.ELDER_CARE), LifeDomain.fromCategory("elder-care"));
    }
}
```

- [ ] **Step 10: Remove `ThresholdCategory` from `HouseholdActionType`**

In `api/src/main/java/io/casehub/life/api/HouseholdActionType.java`:
- Remove the `ThresholdCategory` enum nested type entirely
- Remove the `thresholdCategory` field from each constant
- Remove the `thresholdCategory()` getter method
- Update each constant's constructor call to drop the `ThresholdCategory` argument

The updated enum constants (remove `ThresholdCategory` arg from each):
```java
SPEND_PURCHASE(GatePolicy.AMOUNT_THRESHOLD, true, List.of(HouseholdGroups.ADMIN)),
SPEND_SUBSCRIPTION_CANCEL(GatePolicy.ALWAYS, false, List.of(HouseholdGroups.ADMIN)),
SPEND_SUBSCRIPTION_MODIFY(GatePolicy.AMOUNT_THRESHOLD, true, List.of(HouseholdGroups.ADMIN)),
BOOKING_NONREFUNDABLE(GatePolicy.ALWAYS, false, List.of(HouseholdGroups.ADMIN)),
BOOKING_REFUNDABLE(GatePolicy.AMOUNT_THRESHOLD, true, List.of(HouseholdGroups.ADMIN)),
HEALTH_APPOINTMENT_SPECIALIST(GatePolicy.ALWAYS, true, List.of(HouseholdGroups.ADMIN)),
HEALTH_APPOINTMENT_GP(GatePolicy.NEVER, true, List.of()),
HEALTH_MEDICATION_FLAG(GatePolicy.ALWAYS, false, List.of(HouseholdGroups.ADMIN, HouseholdGroups.MEMBER)),
CONTRACTOR_ENGAGE(GatePolicy.AMOUNT_THRESHOLD, true, List.of(HouseholdGroups.ADMIN)),
LEGAL_DOCUMENT_SUBMIT(GatePolicy.ALWAYS, false, List.of(HouseholdGroups.ADMIN)),
ELDER_CARE_DECISION(GatePolicy.ALWAYS, true, List.of(HouseholdGroups.ADMIN, HouseholdGroups.MEMBER));
```

Updated constructor:
```java
HouseholdActionType(GatePolicy gatePolicy, boolean reversible, List<String> candidateGroups) {
    this.gatePolicy = gatePolicy;
    this.reversible = reversible;
    this.candidateGroups = List.copyOf(candidateGroups);
}
```

Remove `thresholdCategory` field and `thresholdCategory()` method. Remove `ThresholdCategory` enum. Keep `GatePolicy` enum, `gatePolicy()`, `reversible()`, `candidateGroups()`, `actionType()`, `fromActionType()`.

- [ ] **Step 11: Update `HouseholdActionTypeTest` — remove `ThresholdCategory` tests**

In `api/src/test/java/io/casehub/life/api/HouseholdActionTypeTest.java`, delete the four `thresholdCategory` test methods:
- `spendPurchase_usesSpendThresholdCategory()`
- `contractorEngage_usesContractorThresholdCategory()`
- `bookingRefundable_usesBookingThresholdCategory()`
- `spendSubscriptionModify_usesSpendThresholdCategory()`

- [ ] **Step 12: Run all api/ tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api
```

Expected: BUILD SUCCESS — all tests pass including HealthDomainDescriptorTest, AllDomainDescriptorsTest, LifeDomainTest, HouseholdActionTypeTest

- [ ] **Step 13: Commit**

```bash
git add api/src/
git commit -m "refactor(#27): add 8 domain descriptors, update LifeDomain with descriptor()/fromCategory(), remove ThresholdCategory"
```

---

## Task 3: `DomainLedgerHandler` interface and three CDI implementations

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/ledger/DomainLedgerHandler.java`
- Create: `app/src/main/java/io/casehub/life/app/service/ledger/HealthDomainLedgerHandler.java`
- Create: `app/src/main/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandler.java`
- Create: `app/src/main/java/io/casehub/life/app/service/ledger/FinanceDomainLedgerHandler.java`
- Create test files for each handler

- [ ] **Step 1: Create `DomainLedgerHandler` interface**

```java
package io.casehub.life.app.service.ledger;

import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeCommitmentRecord;
import io.casehub.work.runtime.model.WorkItem;

import java.util.UUID;

public interface DomainLedgerHandler {
    LifeDomain domain();

    void writeEntry(LifeDecisionEventType event, UUID workItemId, WorkItem workItem);

    default void writeEntry(LifeDecisionEventType event, LifeCommitmentRecord record) {
        // Default no-op: only FinanceDomainLedgerHandler overrides this
    }
}
```

- [ ] **Step 2: Write failing test for `HealthDomainLedgerHandler`**

Create `app/src/test/java/io/casehub/life/app/service/ledger/HealthDomainLedgerHandlerTest.java`:

```java
package io.casehub.life.app.service.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.app.ledger.HealthDecisionLedgerEntry;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class HealthDomainLedgerHandlerTest {

    @Mock LedgerEntryRepository ledgerRepository;
    @Mock LifeOutcomeAttestationWriter attestationWriter;

    HealthDomainLedgerHandler handler;

    @BeforeEach
    void setUp() {
        handler = new HealthDomainLedgerHandler(ledgerRepository, attestationWriter);
    }

    @Test void domain_isHealth() {
        assertEquals(LifeDomain.HEALTH, handler.domain());
    }

    @Test void writeEntry_nullContext_doesNotWrite() {
        // LifeTaskContext.findByIdOptional returns empty — handler returns early
        UUID taskId = UUID.randomUUID();
        WorkItem workItem = new WorkItem();
        workItem.id = taskId;
        workItem.status = WorkItemStatus.COMPLETED;

        // No Panache in unit test — handler must accept null context gracefully
        // Use a testable subclass that lets us inject context
        handler = new HealthDomainLedgerHandler(ledgerRepository, attestationWriter) {
            @Override protected Optional<LifeTaskContext> findContext(UUID id) {
                return Optional.empty();
            }
        };
        handler.writeEntry(LifeDecisionEventType.COMPLETED, taskId, workItem);
        verify(ledgerRepository, never()).save(any());
    }

    @Test void writeEntry_completed_savesEntry() {
        UUID taskId = UUID.randomUUID();
        WorkItem workItem = new WorkItem();
        workItem.id = taskId;
        workItem.status = WorkItemStatus.COMPLETED;
        workItem.outcome = "appointment-attended";

        LifeTaskContext ctx = new LifeTaskContext();
        ctx.workItemId = taskId;
        ctx.domain = LifeDomain.HEALTH;

        handler = new HealthDomainLedgerHandler(ledgerRepository, attestationWriter) {
            @Override protected Optional<LifeTaskContext> findContext(UUID id) {
                return Optional.of(ctx);
            }
        };

        handler.writeEntry(LifeDecisionEventType.COMPLETED, taskId, workItem);

        ArgumentCaptor<LedgerEntry> captor = ArgumentCaptor.forClass(LedgerEntry.class);
        verify(ledgerRepository).save(captor.capture());
        assertInstanceOf(HealthDecisionLedgerEntry.class, captor.getValue());
        HealthDecisionLedgerEntry entry = (HealthDecisionLedgerEntry) captor.getValue();
        assertEquals("appointment-attended", entry.outcome);
        assertEquals(LifeDecisionEventType.COMPLETED, entry.eventType);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am \
  -Dtest=HealthDomainLedgerHandlerTest \
  -Dsurefire.failIfNoSpecifiedTests=false --batch-mode
```

Expected: FAIL — `HealthDomainLedgerHandler` does not exist

- [ ] **Step 4: Create `HealthDomainLedgerHandler`**

```java
package io.casehub.life.app.service.ledger;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeActorIds;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.app.ledger.HealthDecisionLedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
public class HealthDomainLedgerHandler implements DomainLedgerHandler {

    private static final Logger LOG = Logger.getLogger(HealthDomainLedgerHandler.class);

    @Inject LedgerEntryRepository ledgerRepository;
    @Inject LifeOutcomeAttestationWriter attestationWriter;

    // Package-visible constructor for testing with injected deps
    HealthDomainLedgerHandler(LedgerEntryRepository ledgerRepository,
                               LifeOutcomeAttestationWriter attestationWriter) {
        this.ledgerRepository = ledgerRepository;
        this.attestationWriter = attestationWriter;
    }

    // CDI no-arg constructor
    HealthDomainLedgerHandler() {}

    @Override
    public LifeDomain domain() { return LifeDomain.HEALTH; }

    @Override
    public void writeEntry(LifeDecisionEventType event, UUID workItemId, WorkItem workItem) {
        Optional<LifeTaskContext> ctxOpt = findContext(workItemId);
        if (ctxOpt.isEmpty()) {
            LOG.warnf("HealthDomainLedgerHandler: no LifeTaskContext for workItemId=%s — skipping ledger write", workItemId);
            return;
        }
        LifeTaskContext ctx = ctxOpt.get();

        String actorId = ctx.externalActorId != null
                ? LifeActorIds.of(ctx.externalActorId) : "life-system";
        ActorType actorType = ctx.externalActorId != null ? ActorType.HUMAN : ActorType.SYSTEM;

        HealthDecisionLedgerEntry entry = new HealthDecisionLedgerEntry();
        entry.subjectId     = ctx.workItemId;
        entry.sequenceNumber = nextSequenceNumber(ctx.workItemId);
        entry.entryType     = LedgerEntryType.EVENT;
        entry.actorId       = actorId;
        entry.actorType     = actorType;
        entry.actorRole     = "HealthDecisionAudit";
        entry.workItemId    = ctx.workItemId;
        entry.providerId    = ctx.externalActorId;
        entry.taskCategory  = workItem.category;
        entry.slaDeadline   = workItem.expiresAt;
        entry.eventType     = event;
        entry.outcome       = event == LifeDecisionEventType.COMPLETED ? workItem.outcome : null;

        ledgerRepository.save(entry);
        attestationWriter.attestOutcome(entry, event, ctx, workItem);
    }

    protected Optional<LifeTaskContext> findContext(UUID workItemId) {
        return LifeTaskContext.findByIdOptional(workItemId);
    }

    private int nextSequenceNumber(UUID subjectId) {
        return ledgerRepository.findLatestBySubjectId(subjectId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am \
  -Dtest=HealthDomainLedgerHandlerTest \
  -Dsurefire.failIfNoSpecifiedTests=false --batch-mode
```

Expected: BUILD SUCCESS

- [ ] **Step 6: Create `LegalDomainLedgerHandlerTest`**

Create `app/src/test/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandlerTest.java`:

```java
package io.casehub.life.app.service.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.app.ledger.LegalActionLedgerEntry;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class LegalDomainLedgerHandlerTest {

    @Mock LedgerEntryRepository ledgerRepository;
    @Mock LifeOutcomeAttestationWriter attestationWriter;

    LegalDomainLedgerHandler handler;

    @BeforeEach
    void setUp() {
        handler = new LegalDomainLedgerHandler(ledgerRepository, attestationWriter);
    }

    @Test void domain_isLegal() {
        assertEquals(LifeDomain.LEGAL, handler.domain());
    }

    @Test void writeEntry_nullContext_doesNotWrite() {
        UUID taskId = UUID.randomUUID();
        WorkItem workItem = new WorkItem();
        workItem.id = taskId;

        handler = new LegalDomainLedgerHandler(ledgerRepository, attestationWriter) {
            @Override protected Optional<LifeTaskContext> findContext(UUID id) {
                return Optional.empty();
            }
        };
        handler.writeEntry(LifeDecisionEventType.COMPLETED, taskId, workItem);
        verify(ledgerRepository, never()).save(any());
    }

    @Test void writeEntry_slaBreach_savesLegalEntry() {
        UUID taskId = UUID.randomUUID();
        WorkItem workItem = new WorkItem();
        workItem.id = taskId;
        workItem.title = "File tax return";
        workItem.status = WorkItemStatus.EXPIRED;

        LifeTaskContext ctx = new LifeTaskContext();
        ctx.workItemId = taskId;
        ctx.domain = LifeDomain.LEGAL;

        handler = new LegalDomainLedgerHandler(ledgerRepository, attestationWriter) {
            @Override protected Optional<LifeTaskContext> findContext(UUID id) {
                return Optional.of(ctx);
            }
        };

        handler.writeEntry(LifeDecisionEventType.SLA_BREACH, taskId, workItem);

        ArgumentCaptor<LedgerEntry> captor = ArgumentCaptor.forClass(LedgerEntry.class);
        verify(ledgerRepository).save(captor.capture());
        assertInstanceOf(LegalActionLedgerEntry.class, captor.getValue());
        LegalActionLedgerEntry entry = (LegalActionLedgerEntry) captor.getValue();
        assertEquals("File tax return", entry.legalObligation);
        assertEquals(LifeDecisionEventType.SLA_BREACH, entry.eventType);
    }
}
```

- [ ] **Step 7: Create `LegalDomainLedgerHandler`**

```java
package io.casehub.life.app.service.ledger;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeActorIds;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.app.ledger.LegalActionLedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
public class LegalDomainLedgerHandler implements DomainLedgerHandler {

    private static final Logger LOG = Logger.getLogger(LegalDomainLedgerHandler.class);

    @Inject LedgerEntryRepository ledgerRepository;
    @Inject LifeOutcomeAttestationWriter attestationWriter;

    LegalDomainLedgerHandler(LedgerEntryRepository ledgerRepository,
                              LifeOutcomeAttestationWriter attestationWriter) {
        this.ledgerRepository = ledgerRepository;
        this.attestationWriter = attestationWriter;
    }

    LegalDomainLedgerHandler() {}

    @Override public LifeDomain domain() { return LifeDomain.LEGAL; }

    @Override
    public void writeEntry(LifeDecisionEventType event, UUID workItemId, WorkItem workItem) {
        Optional<LifeTaskContext> ctxOpt = findContext(workItemId);
        if (ctxOpt.isEmpty()) {
            LOG.warnf("LegalDomainLedgerHandler: no LifeTaskContext for workItemId=%s — skipping ledger write", workItemId);
            return;
        }
        LifeTaskContext ctx = ctxOpt.get();

        String actorId = ctx.externalActorId != null
                ? LifeActorIds.of(ctx.externalActorId) : "life-system";
        ActorType actorType = ctx.externalActorId != null ? ActorType.HUMAN : ActorType.SYSTEM;

        LegalActionLedgerEntry entry = new LegalActionLedgerEntry();
        entry.subjectId     = ctx.workItemId;
        entry.sequenceNumber = nextSequenceNumber(ctx.workItemId);
        entry.entryType     = LedgerEntryType.EVENT;
        entry.actorId       = actorId;
        entry.actorType     = actorType;
        entry.actorRole     = "LegalActionAudit";
        entry.workItemId    = ctx.workItemId;
        entry.legalObligation = workItem.title;
        entry.filingDeadline  = workItem.expiresAt;
        entry.eventType       = event;
        entry.actionTaken     = event == LifeDecisionEventType.COMPLETED ? workItem.outcome : null;

        ledgerRepository.save(entry);
        attestationWriter.attestOutcome(entry, event, ctx, workItem);
    }

    protected Optional<LifeTaskContext> findContext(UUID workItemId) {
        return LifeTaskContext.findByIdOptional(workItemId);
    }

    private int nextSequenceNumber(UUID subjectId) {
        return ledgerRepository.findLatestBySubjectId(subjectId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

- [ ] **Step 8: Create `FinanceDomainLedgerHandlerTest`**

Create `app/src/test/java/io/casehub/life/app/service/ledger/FinanceDomainLedgerHandlerTest.java`:

```java
package io.casehub.life.app.service.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeCommitmentRecord;
import io.casehub.life.app.ledger.FinancialDecisionLedgerEntry;
import io.casehub.work.runtime.model.WorkItem;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class FinanceDomainLedgerHandlerTest {

    @Mock LedgerEntryRepository ledgerRepository;

    FinanceDomainLedgerHandler handler;

    @BeforeEach
    void setUp() {
        handler = new FinanceDomainLedgerHandler(ledgerRepository);
    }

    @Test void domain_isFinance() {
        assertEquals(LifeDomain.FINANCE, handler.domain());
    }

    @Test void writeEntry_task_created_isNoOp() {
        // FINANCE CREATED entries go via the commitment overload, not task overload
        UUID taskId = UUID.randomUUID();
        handler.writeEntry(LifeDecisionEventType.CREATED, taskId, new WorkItem());
        verify(ledgerRepository, never()).save(any());
    }

    @Test void writeEntry_task_slaBreach_withRecord_savesEntry() {
        UUID taskId = UUID.randomUUID();
        LifeCommitmentRecord record = new LifeCommitmentRecord();
        record.id = UUID.randomUUID();
        record.amountThreshold = BigDecimal.valueOf(500);

        WorkItem workItem = new WorkItem();
        workItem.id = taskId;

        handler = new FinanceDomainLedgerHandler(ledgerRepository) {
            @Override protected Optional<LifeCommitmentRecord> findRecord(UUID id) {
                return Optional.of(record);
            }
        };

        handler.writeEntry(LifeDecisionEventType.SLA_BREACH, taskId, workItem);

        ArgumentCaptor<LedgerEntry> captor = ArgumentCaptor.forClass(LedgerEntry.class);
        verify(ledgerRepository).save(captor.capture());
        assertInstanceOf(FinancialDecisionLedgerEntry.class, captor.getValue());
    }

    @Test void writeEntry_commitment_created_savesEntry() {
        LifeCommitmentRecord record = new LifeCommitmentRecord();
        record.id = UUID.randomUUID();
        record.amountThreshold = BigDecimal.valueOf(1000);
        record.purchaseCategory = "contractor";

        handler.writeEntry(LifeDecisionEventType.CREATED, record);

        ArgumentCaptor<LedgerEntry> captor = ArgumentCaptor.forClass(LedgerEntry.class);
        verify(ledgerRepository).save(captor.capture());
        FinancialDecisionLedgerEntry entry = (FinancialDecisionLedgerEntry) captor.getValue();
        assertEquals(LifeDecisionEventType.CREATED, entry.eventType);
        assertEquals(BigDecimal.valueOf(1000), entry.amountThreshold);
    }

    @Test void writeEntry_commitment_slaBreach_savesEntry() {
        LifeCommitmentRecord record = new LifeCommitmentRecord();
        record.id = UUID.randomUUID();
        record.amountThreshold = BigDecimal.valueOf(750);

        handler.writeEntry(LifeDecisionEventType.SLA_BREACH, record);

        ArgumentCaptor<LedgerEntry> captor = ArgumentCaptor.forClass(LedgerEntry.class);
        verify(ledgerRepository).save(captor.capture());
        FinancialDecisionLedgerEntry entry = (FinancialDecisionLedgerEntry) captor.getValue();
        assertEquals(LifeDecisionEventType.SLA_BREACH, entry.eventType);
    }
}
```

- [ ] **Step 9: Create `FinanceDomainLedgerHandler`**

```java
package io.casehub.life.app.service.ledger;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeCommitmentRecord;
import io.casehub.life.app.ledger.FinancialDecisionLedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
public class FinanceDomainLedgerHandler implements DomainLedgerHandler {

    @Inject LedgerEntryRepository ledgerRepository;

    FinanceDomainLedgerHandler(LedgerEntryRepository ledgerRepository) {
        this.ledgerRepository = ledgerRepository;
    }

    FinanceDomainLedgerHandler() {}

    @Override public LifeDomain domain() { return LifeDomain.FINANCE; }

    @Override
    public void writeEntry(LifeDecisionEventType event, UUID workItemId, WorkItem workItem) {
        // CREATED events are commitment-initiated — use writeEntry(event, record) instead
        if (event == LifeDecisionEventType.CREATED) return;

        findRecord(workItemId).ifPresent(record -> writeFromRecord(event, record, workItemId));
    }

    @Override
    public void writeEntry(LifeDecisionEventType event, LifeCommitmentRecord record) {
        writeFromRecord(event, record, record.workItemId);
    }

    private void writeFromRecord(LifeDecisionEventType event, LifeCommitmentRecord record, UUID workItemId) {
        FinancialDecisionLedgerEntry entry = new FinancialDecisionLedgerEntry();
        entry.subjectId      = record.id;
        entry.sequenceNumber = nextSequenceNumber(record.id);
        entry.entryType      = LedgerEntryType.EVENT;
        entry.actorId        = "life-system";
        entry.actorType      = ActorType.SYSTEM;
        entry.actorRole      = "FinancialDecisionAudit";
        entry.workItemId     = workItemId;
        entry.oversightRef   = record.id;
        entry.amountThreshold  = record.amountThreshold;
        entry.purchaseCategory = record.purchaseCategory;
        entry.approvedBy = event == LifeDecisionEventType.COMPLETED ? record.approvedBy : null;
        entry.eventType  = event;
        ledgerRepository.save(entry);
    }

    protected Optional<LifeCommitmentRecord> findRecord(UUID workItemId) {
        return LifeCommitmentRecord.findByWorkItemId(workItemId);
    }

    private int nextSequenceNumber(UUID subjectId) {
        return ledgerRepository.findLatestBySubjectId(subjectId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

- [ ] **Step 10: Run all three handler tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am \
  -Dtest="HealthDomainLedgerHandlerTest,LegalDomainLedgerHandlerTest,FinanceDomainLedgerHandlerTest" \
  -Dsurefire.failIfNoSpecifiedTests=false --batch-mode
```

Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/service/ledger/DomainLedgerHandler.java \
        app/src/main/java/io/casehub/life/app/service/ledger/HealthDomainLedgerHandler.java \
        app/src/main/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandler.java \
        app/src/main/java/io/casehub/life/app/service/ledger/FinanceDomainLedgerHandler.java \
        app/src/test/java/io/casehub/life/app/service/ledger/
git commit -m "feat(#27): add DomainLedgerHandler interface and three CDI handler implementations"
```

---

## Task 4: Update service classes — remove all domain switches

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/service/ledger/LifeLedgerWriter.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriter.java`
- Modify: `app/src/main/java/io/casehub/life/app/observer/LifeDecisionLedgerObserver.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/LifeTaskService.java`
- Modify: `app/src/main/java/io/casehub/life/app/commitment/OversightGateStrategy.java`
- Modify: `app/src/main/java/io/casehub/life/app/observer/LifeWatchdogAlertObserver.java`
- Modify: `app/src/test/java/io/casehub/life/app/service/ledger/LifeLedgerWriterTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriterTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/observer/LifeWatchdogAlertObserverTest.java`

- [ ] **Step 1: Shrink `LifeLedgerWriter` — remove the three domain write methods**

Replace `app/src/main/java/io/casehub/life/app/service/ledger/LifeLedgerWriter.java` with:

```java
package io.casehub.life.app.service.ledger;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeActorIds;
import io.casehub.life.app.entity.ExternalActor;
import io.casehub.life.app.ledger.ExternalActorErasureLedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.UUID;

@ApplicationScoped
public class LifeLedgerWriter {

    @Inject
    LedgerEntryRepository ledgerRepository;

    public void writeErasureEntry(final ExternalActor actor, final String erasedBy) {
        ExternalActorErasureLedgerEntry entry = new ExternalActorErasureLedgerEntry();
        populateBase(entry, actor.id, erasedBy, ActorType.HUMAN, "GdprDataController");
        entry.erasedActorId = actor.id;
        entry.contactMethod = actor.contactMethod;
        entry.erasedBy = erasedBy;
        ledgerRepository.save(entry);
    }

    public void populateBase(LedgerEntry entry, UUID subjectId,
                              String actorId, ActorType actorType, String actorRole) {
        entry.subjectId      = subjectId;
        entry.sequenceNumber = nextSequenceNumber(subjectId);
        entry.entryType      = LedgerEntryType.EVENT;
        entry.actorId        = actorId;
        entry.actorType      = actorType;
        entry.actorRole      = actorRole;
    }

    private int nextSequenceNumber(UUID subjectId) {
        return ledgerRepository.findLatestBySubjectId(subjectId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

- [ ] **Step 2: Update `LifeLedgerWriterTest` — remove tests for deleted methods**

In `app/src/test/java/io/casehub/life/app/service/ledger/LifeLedgerWriterTest.java`, delete all test methods that call `writeHealthEntry`, `writeFinancialEntry`, or `writeLegalEntry`. Keep only tests for `writeErasureEntry` and `populateBase` if they exist.

- [ ] **Step 3: Update `LifeOutcomeAttestationWriter` — remove DOMAIN_TO_CAPABILITY map**

Replace the static map and `resolveCapabilityTag()` in `LifeOutcomeAttestationWriter.java`.

Remove:
```java
private static final Map<LifeDomain, String> DOMAIN_TO_CAPABILITY = Map.of(...);
```

Update `resolveCapabilityTag()`:
```java
private String resolveCapabilityTag(final LifeTaskContext ctx, final WorkItem workItem) {
    if (ctx.domain != null) {
        return ctx.domain.descriptor().capability();
    }
    if (workItem.scope != null) {
        String[] segments = workItem.scope.split("/");
        if (segments.length >= 3) {
            try {
                return LifeDomain.valueOf(segments[2].toUpperCase()).descriptor().capability();
            } catch (IllegalArgumentException ignored) {}
        }
    }
    return CapabilityTag.GLOBAL;
}
```

Add import: `import io.casehub.life.api.LifeDomain;`
Remove import for `Map` if no longer needed elsewhere.

- [ ] **Step 4: Update `LifeOutcomeAttestationWriterTest` — update capability assertions**

In `LifeOutcomeAttestationWriterTest`, replace any assertions that use `DOMAIN_TO_CAPABILITY` values with direct capability string assertions (e.g. `assertEquals("health-coordination", ...)` or verify via `LifeDomain.HEALTH.descriptor().capability()`). The capability strings themselves are unchanged.

- [ ] **Step 5: Update `LifeDecisionLedgerObserver` — replace switch with handler dispatch**

Replace the entire class body of `LifeDecisionLedgerObserver.java`:

```java
package io.casehub.life.app.observer;

import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.service.ledger.DomainLedgerHandler;
import io.casehub.work.runtime.event.SlaBreachEvent;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.util.UUID;

@ApplicationScoped
public class LifeDecisionLedgerObserver {

    @Inject @Any
    Instance<DomainLedgerHandler> handlers;

    static LifeDomain domainFromScope(String scope) {
        if (scope == null || scope.isEmpty()) return null;
        String[] segments = scope.split("/");
        if (segments.length < 3) return null;
        try {
            return LifeDomain.valueOf(segments[2].toUpperCase());
        } catch (IllegalArgumentException e) {
            return null;
        }
    }

    private LifeDomain resolveDomain(UUID workItemId, WorkItem workItem) {
        LifeDomain domain = domainFromScope(workItem.scope);
        if (domain != null) return domain;
        return io.casehub.life.app.entity.LifeTaskContext
                .<io.casehub.life.app.entity.LifeTaskContext>findByIdOptional(workItemId)
                .map(ctx -> ctx.domain)
                .orElse(null);
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void onSlaBreachEvent(@Observes final SlaBreachEvent event) {
        resolveAndWrite(event.context().task().taskId(), LifeDecisionEventType.SLA_BREACH);
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void onLifecycleEvent(@Observes final WorkItemLifecycleEvent event) {
        if (event.status() != WorkItemStatus.COMPLETED) return;
        resolveAndWrite(event.workItemId(), LifeDecisionEventType.COMPLETED);
    }

    private void resolveAndWrite(UUID workItemId, LifeDecisionEventType eventType) {
        WorkItem workItem = WorkItem.findByIdOptional(workItemId).orElse(null);
        if (workItem == null) return;
        LifeDomain domain = resolveDomain(workItemId, workItem);
        if (domain == null) return;
        handlers.stream()
                .filter(h -> h.domain() == domain)
                .findFirst()
                .ifPresent(h -> h.writeEntry(eventType, workItemId, workItem));
    }
}
```

- [ ] **Step 6: Update `LifeTaskService` — use `LifeDomain.fromCategory()` and handler dispatch**

In `LifeTaskService.java`:

1. Add injection: `@Inject @Any Instance<DomainLedgerHandler> ledgerHandlers;`
2. Remove injection: `@Inject LifeLedgerWriter lifeLedgerWriter;` (keep `WorkItemService` and `WorkItemTemplateService`)
3. Replace `domainFromCategory()` call with `LifeDomain.fromCategory(template.category).orElse(LifeDomain.HOUSEHOLD)`
4. Replace lines 95–98 (HEALTH/LEGAL hardcoded check) with:
```java
final LifeDomain domain = LifeDomain.fromCategory(template.category).orElse(LifeDomain.HOUSEHOLD);
// ... build workReq and workItem as before ...
ledgerHandlers.stream()
        .filter(h -> h.domain() == domain)
        .findFirst()
        .ifPresent(h -> h.writeEntry(LifeDecisionEventType.CREATED, workItem.id, workItem));
```
5. Delete the `domainFromCategory()` method entirely.
6. Add import: `import io.casehub.life.api.LifeDomain;`, `import io.casehub.life.app.service.ledger.DomainLedgerHandler;`, `import jakarta.enterprise.inject.Any;`, `import jakarta.enterprise.inject.Instance;`

- [ ] **Step 7: Update `OversightGateStrategy` — replace LifeLedgerWriter with handler**

In `OversightGateStrategy.java`:
1. Remove `@Inject LifeLedgerWriter lifeLedgerWriter;`
2. Add `@Inject @Any Instance<DomainLedgerHandler> ledgerHandlers;`
3. Replace `lifeLedgerWriter.writeFinancialEntry(LifeDecisionEventType.CREATED, record, null);` with:
```java
ledgerHandlers.stream()
        .filter(h -> h.domain() == LifeDomain.FINANCE)
        .findFirst()
        .ifPresent(h -> h.writeEntry(LifeDecisionEventType.CREATED, record));
```
4. Add imports: `import io.casehub.life.api.LifeDomain;`, `import io.casehub.life.app.service.ledger.DomainLedgerHandler;`, `import jakarta.enterprise.inject.Any;`, `import jakarta.enterprise.inject.Instance;`
5. Remove import for `LifeLedgerWriter`.

- [ ] **Step 8: Update `LifeWatchdogAlertObserver` — replace LifeLedgerWriter with handler (FINANCE write)**

In `LifeWatchdogAlertObserver.java`:
1. Remove `@Inject LifeLedgerWriter lifeLedgerWriter;`
2. Add `@Inject @Any Instance<DomainLedgerHandler> ledgerHandlers;`
3. Replace `lifeLedgerWriter.writeFinancialEntry(LifeDecisionEventType.SLA_BREACH, record, null);` with:
```java
ledgerHandlers.stream()
        .filter(h -> h.domain() == LifeDomain.FINANCE)
        .findFirst()
        .ifPresent(h -> h.writeEntry(LifeDecisionEventType.SLA_BREACH, record));
```
4. Add imports: `import io.casehub.life.api.LifeDomain;`, `import io.casehub.life.app.service.ledger.DomainLedgerHandler;`, `import jakarta.enterprise.inject.Any;`, `import jakarta.enterprise.inject.Instance;`
5. Keep all other LifeWatchdogAlertObserver logic unchanged for now (the escalation title switch will be removed in Plan B, Task commitment-escalation).

- [ ] **Step 9: Update `LifeWatchdogAlertObserverTest` — add handler assertions**

In `LifeWatchdogAlertObserverTest.java`, add three test assertions (or update existing tests):
1. OVERSIGHT records route to `FinanceDomainLedgerHandler.writeEntry(SLA_BREACH, record)` (commitment overload)
2. `LifeLedgerWriter` is not injected or called
3. (The escalation title assertion will be added in Plan B)

- [ ] **Step 10: Build and run all modified tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app
```

Expected: BUILD SUCCESS — all tests pass

- [ ] **Step 11: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/service/ledger/LifeLedgerWriter.java \
        app/src/main/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriter.java \
        app/src/main/java/io/casehub/life/app/observer/LifeDecisionLedgerObserver.java \
        app/src/main/java/io/casehub/life/app/service/LifeTaskService.java \
        app/src/main/java/io/casehub/life/app/commitment/OversightGateStrategy.java \
        app/src/main/java/io/casehub/life/app/observer/LifeWatchdogAlertObserver.java \
        app/src/test/
git commit -m "refactor(#27): replace domain switches with DomainLedgerHandler CDI dispatch in all observers and services"
```

---

## Task 5: `LifeTrustRoutingPolicyProvider` — remove static maps, add `@PostConstruct` index

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProvider.java`
- Test update: existing `LifeTrustRoutingPolicyProviderTest`

- [ ] **Step 1: Rewrite `LifeTrustRoutingPolicyProvider`**

Replace the full class:

```java
package io.casehub.life.app.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.LifeRoutingPolicy;
import io.casehub.platform.api.preferences.PreferenceKey;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

@ApplicationScoped
public class LifeTrustRoutingPolicyProvider implements TrustRoutingPolicyProvider {

    @Inject
    PreferenceProvider preferenceProvider;

    private Map<String, LifeDomain> capabilityIndex;

    @PostConstruct
    void buildCapabilityIndex() {
        Map<String, LifeDomain> index = new HashMap<>();
        for (LifeDomain domain : LifeDomain.values()) {
            for (String cap : domain.descriptor().workerCapabilities()) {
                index.put(cap, domain);
            }
        }
        this.capabilityIndex = Map.copyOf(index);
    }

    @Override
    public TrustRoutingPolicy forCapability(String capabilityName) {
        LifeDomain domain = capabilityIndex.get(capabilityName);
        if (domain == null) {
            return TrustRoutingPolicy.DEFAULT;
        }

        LifeRoutingPolicy base = domain.descriptor().routingPolicy();
        // Scope key = coarse capability string (e.g. "health-coordination") — unchanged from before
        SettingsScope scope = SettingsScope.of("casehubio", "life", "trust-routing",
                domain.descriptor().capability());
        Preferences prefs = preferenceProvider.resolve(scope);

        double threshold = base.threshold().orElse(TrustRoutingPolicy.DEFAULT.threshold());
        int minObs  = base.minimumObservations().orElse(TrustRoutingPolicy.DEFAULT.minimumObservations());
        double margin = base.borderlineMargin().orElse(TrustRoutingPolicy.DEFAULT.borderlineMargin());

        DoublePreference blendPref = prefs.get(LifeTrustRoutingPolicyKeys.BLEND_FACTOR);
        double blendFactor = blendPref != null ? blendPref.value() : TrustRoutingPolicy.DEFAULT.blendFactor();

        Map<String, Double> qualityFloors = buildQualityFloors(prefs);

        return new TrustRoutingPolicy(threshold, minObs, margin, blendFactor,
                Map.copyOf(qualityFloors), false);
    }

    private Map<String, Double> buildQualityFloors(Preferences prefs) {
        Map<String, Double> floors = new HashMap<>();
        for (Map.Entry<String, PreferenceKey<DoublePreference>> entry
                : LifeTrustRoutingPolicyKeys.allFloorKeys().entrySet()) {
            DoublePreference value = prefs.get(entry.getValue());
            if (value != null && value.value() > 0.0) {
                floors.put(entry.getKey(), value.value());
            }
        }
        return floors;
    }
}
```

- [ ] **Step 2: Update `LifeTrustRoutingPolicyProviderTest`**

The test must verify:
- `forCapability("book-appointment")` resolves to the HEALTH routing policy (threshold 0.75)
- `forCapability("unknown-capability")` returns `TrustRoutingPolicy.DEFAULT`
- All 32 worker capabilities resolve to a non-default domain

If the test previously used the POLICIES static map or CAPABILITY_TO_DOMAIN, update assertions to use `LifeDomain.values()` or concrete expected values.

- [ ] **Step 3: Build and run routing tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am \
  -Dtest=LifeTrustRoutingPolicyProviderTest \
  -Dsurefire.failIfNoSpecifiedTests=false --batch-mode
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProvider.java \
        app/src/test/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProviderTest.java
git commit -m "refactor(#27): LifeTrustRoutingPolicyProvider — @PostConstruct capability index from descriptors, remove static maps"
```

---

## Task 6: `LifeSlaBreachPolicy` — thin dispatcher using domain descriptor

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/spi/LifeSlaBreachPolicy.java`

- [ ] **Step 1: Rewrite `LifeSlaBreachPolicy`**

```java
package io.casehub.life.app.spi;

import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.LifeSlaPolicy;
import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.SlaBreachContext;
import io.casehub.work.api.SlaBreachPolicy;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class LifeSlaBreachPolicy implements SlaBreachPolicy {

    @Override
    public BreachDecision onBreach(final SlaBreachContext ctx) {
        LifeDomain domain = LifeDomain.fromCategory(ctx.task().category())
                .orElse(LifeDomain.HOUSEHOLD);
        LifeSlaPolicy policy = domain.descriptor().slaPolicy();

        if (ctx.task().candidateGroups().contains(policy.escalationGroup())) {
            return new BreachDecision.Fail("life-sla-exhausted");
        }
        return BreachDecision.EscalateTo.to(policy.escalationGroup())
                .withDeadline(policy.escalationDeadline());
    }
}
```

- [ ] **Step 2: Verify the integration test for SLA policy**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am \
  -Dtest=LifeSlaBreachPolicyTest \
  -Dsurefire.failIfNoSpecifiedTests=false --batch-mode
```

If `LifeSlaBreachPolicyTest` doesn't exist yet, create it as a `@QuarkusTest`:

```java
package io.casehub.life.app.spi;

import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.SlaBreachContext;
import io.casehub.work.api.SlaBreachPolicy;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class LifeSlaBreachPolicyTest {

    @Inject SlaBreachPolicy policy;

    @Test void healthDomain_firstBreach_escalatesWithin24h() {
        SlaBreachContext ctx = mockContext("health", java.util.List.of("household-member"));
        BreachDecision decision = policy.onBreach(ctx);
        assertInstanceOf(BreachDecision.EscalateTo.class, decision);
        BreachDecision.EscalateTo escalation = (BreachDecision.EscalateTo) decision;
        assertEquals("household-admin", escalation.candidateGroup());
        assertEquals(java.time.Duration.ofHours(24), escalation.deadline());
    }

    @Test void householdDomain_firstBreach_escalatesWithin48h() {
        SlaBreachContext ctx = mockContext("household", java.util.List.of("household-member"));
        BreachDecision decision = policy.onBreach(ctx);
        assertInstanceOf(BreachDecision.EscalateTo.class, decision);
        assertEquals(java.time.Duration.ofHours(48),
                ((BreachDecision.EscalateTo) decision).deadline());
    }

    @Test void anyDomain_secondBreach_fails() {
        SlaBreachContext ctx = mockContext("health", java.util.List.of("household-admin"));
        BreachDecision decision = policy.onBreach(ctx);
        assertInstanceOf(BreachDecision.Fail.class, decision);
    }

    private SlaBreachContext mockContext(String category, java.util.List<String> candidateGroups) {
        // Use Mockito or a test double — implement per existing test helper patterns in the project
        var task = org.mockito.Mockito.mock(io.casehub.work.api.BreachedTask.class);
        org.mockito.Mockito.when(task.category()).thenReturn(category);
        org.mockito.Mockito.when(task.candidateGroups()).thenReturn(candidateGroups);
        var ctx = org.mockito.Mockito.mock(SlaBreachContext.class);
        org.mockito.Mockito.when(ctx.task()).thenReturn(task);
        return ctx;
    }
}
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Run full app test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app
```

Expected: BUILD SUCCESS — all tests pass

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/spi/LifeSlaBreachPolicy.java \
        app/src/test/java/io/casehub/life/app/spi/LifeSlaBreachPolicyTest.java
git commit -m "refactor(#27): LifeSlaBreachPolicy — thin dispatcher via domain descriptor slaPolicy()"
```

---

## Task 7: Final build, push, and workspace sync

- [ ] **Step 1: Full build of both modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: BUILD SUCCESS

- [ ] **Step 2: Push project branch**

```bash
git push
```

- [ ] **Step 3: Commit workspace with plan**

```bash
git -C /Users/mdproctor/claude/public/casehub/life add plans/2026-06-09-business-logic-centralization-a.md
git -C /Users/mdproctor/claude/public/casehub/life commit -m "docs: Plan A — business logic centralization (descriptors, handlers, routing, SLA)"
git -C /Users/mdproctor/claude/public/casehub/life push
```

---

## Self-review

**Spec coverage check:**

| Spec section | Task covering it |
|---|---|
| `LifeSlaPolicy` record | Task 1 |
| `LifeDomainDescriptor` interface | Task 1 |
| `LifeRoutingPolicy` moved to api/ | Task 1 |
| 8 domain descriptor POJOs | Task 2 |
| `LifeDomain.descriptor()` + `fromCategory()` | Task 2 |
| `ThresholdCategory` removed | Task 2 |
| `DomainLedgerHandler` interface (two overloads) | Task 3 |
| `HealthDomainLedgerHandler` | Task 3 |
| `LegalDomainLedgerHandler` | Task 3 |
| `FinanceDomainLedgerHandler` | Task 3 |
| `LifeLedgerWriter` shrinks | Task 4 |
| `LifeOutcomeAttestationWriter` removes DOMAIN_TO_CAPABILITY | Task 4 |
| `LifeDecisionLedgerObserver` no switch | Task 4 |
| `LifeTaskService` removes domainFromCategory() | Task 4 |
| `OversightGateStrategy` replaces LifeLedgerWriter | Task 4 |
| `LifeWatchdogAlertObserver` FINANCE write via handler | Task 4 |
| `LifeTrustRoutingPolicyProvider` @PostConstruct index | Task 5 |
| `LifeSlaBreachPolicy` thin dispatcher | Task 6 |

**Not covered in Plan A (deferred to Plan B):**
- `HouseholdRiskRule` interface + 11 rules + `LifeActionRiskClassifier`
- `LifeCommitmentStrategy.escalationTitle()` + `commitmentMode()`
- `LifeWatchdogAlertObserver` escalation title refactor
- `LifeCaseDescriptor` + `LifeTypedCaseHub` + 6 case descriptors + CDI shells
- Delete `*CaseDefinitions` companion classes

**Placeholder scan:** None found — all steps have complete code.

**Type consistency:**
- `LifeDomain.fromCategory()` returns `Optional<LifeDomain>` — callers use `.orElse(LifeDomain.HOUSEHOLD)` ✅
- `DomainLedgerHandler.writeEntry(event, workItemId, workItem)` — consistent across all call sites ✅
- `DomainLedgerHandler.writeEntry(event, record)` — commitment overload, used in OversightGateStrategy and LifeWatchdogAlertObserver ✅
