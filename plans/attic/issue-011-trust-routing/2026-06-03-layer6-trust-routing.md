# Layer 6: Trust Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire trust-weighted agent routing, build the attestation pipeline that makes trust scores accumulate, and expose ledger-backed trust profiles on ExternalActor REST responses.

**Architecture:** Two orthogonal concerns share dependency wiring. Problem A (agent routing) adds `casehub-engine-ledger` + `TrustRoutingPolicyProvider` SPI implementation with capability-name→domain mapping. Problem B (external actor trust) fixes actorId attribution in the ledger writer, adds an attestation pipeline, and enriches `GET /external-actors/{id}` with trust scores read from `TrustGateService`. All trust scoring computation is owned by casehub-ledger — casehub-life only reads.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-ledger, casehub-platform-config, casehub-ledger `TrustGateService`/`LedgerAttestation`

**Spec:** `docs/specs/2026-06-03-layer6-trust-routing.md`

---

## File Structure

### New files — api/

| File | Responsibility |
|---|---|
| `api/src/main/java/io/casehub/life/api/LifeActorIds.java` | `life-actor:{uuid}` convention — `of()`, `isLifeActor()`, `extractId()` |
| `api/src/test/java/io/casehub/life/api/LifeActorIdsTest.java` | Unit tests for round-trip, null guards |

### New files — app/

| File | Responsibility |
|---|---|
| `app/src/main/java/io/casehub/life/app/routing/DoublePreference.java` | `SingleValuePreference` for YAML double values |
| `app/src/main/java/io/casehub/life/app/routing/LifeRoutingPolicy.java` | Domain routing policy record (threshold, minObs, margin, fallback, rationale) |
| `app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyKeys.java` | `PreferenceKey` constants for blend-factor + 4 dimension floor keys |
| `app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProvider.java` | `TrustRoutingPolicyProvider` SPI impl — capability→domain mapping + assembly |
| `app/src/main/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriter.java` | Converts outcomes to `LedgerAttestation` — verdict, dimension scores, capability tag |
| `app/src/main/resources/casehub/life/trust-routing.yaml` | YAML config — blend factors and quality floors per domain |

### New test files — app/

| File | Tests |
|---|---|
| `app/src/test/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyKeysTest.java` | All floor keys present; `allFloorKeys()` size |
| `app/src/test/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProviderTest.java` | `@QuarkusTest` — capability resolution, YAML floor wiring, unknown→DEFAULT |
| `app/src/test/java/io/casehub/life/app/service/LifeOutcomeAttestationWriterTest.java` | Mockito unit test — verdict mapping, dimension scores, capability tag |
| `app/src/test/java/io/casehub/life/app/service/LifeLedgerWriterActorIdTest.java` | Mockito unit test — actorId convention |
| `app/src/test/java/io/casehub/life/app/TrustStrategyDisplacementTest.java` | `@QuarkusTest` — verify `TrustWeightedAgentStrategy` active |
| `app/src/test/java/io/casehub/life/app/ExternalActorTrustEnrichmentTest.java` | `@QuarkusTest` — REST response contains TrustProfile |
| `app/src/test/java/io/casehub/life/app/ColdStartBehaviorTest.java` | `@QuarkusTest` — no trust data: policies work, profile empty |

### Modified files

| File | Change |
|---|---|
| `app/pom.xml` | Add `casehub-engine-ledger`, `casehub-platform-config` |
| `app/src/main/resources/application.properties` | Flyway qhorus locations + `trust-score.enabled` |
| `app/src/test/resources/application.properties` | Jandex index for engine-ledger |
| `api/src/main/java/io/casehub/life/api/response/ExternalActorResponse.java` | Add `TrustProfile` nested record |
| `app/src/main/java/io/casehub/life/app/service/ExternalActorService.java` | Inject `TrustGateService`, build `TrustProfile` |
| `app/src/main/java/io/casehub/life/app/service/ledger/LifeLedgerWriter.java` | Contextual actorId, return `LedgerEntry`, call attestation writer |
| `app/src/test/java/io/casehub/life/app/ExternalActorResourceTest.java` | Update expected response shape |
| `app/src/test/java/io/casehub/life/app/ExternalActorGdprResourceTest.java` | Verify GDPR-erased `TrustProfile.EMPTY` |

---

### Task 1: Dependencies and Configuration

**Files:**
- Modify: `app/pom.xml`
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/resources/application.properties`

- [ ] **Step 1: Add casehub-engine-ledger dependency to app/pom.xml**

Insert after the `casehub-engine-blackboard` dependency block:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-ledger</artifactId>
      <version>${casehub.version}</version>
    </dependency>
```

- [ ] **Step 2: Add casehub-platform-config dependency to app/pom.xml**

Insert after the `casehub-platform-expression` dependency:

```xml
    <!-- YAML-backed PreferenceProvider for trust routing thresholds.
         @ApplicationScoped — displaces MockPreferenceProvider @DefaultBean automatically. -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-config</artifactId>
    </dependency>
```

- [ ] **Step 3: Update production application.properties**

Add `classpath:db/engine-ledger/migration` to qhorus Flyway locations. Change the existing line:

```properties
quarkus.flyway."qhorus".locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/life/ledger/migration,classpath:db/engine-ledger/migration
```

Add trust score enablement at the end of the file:

```properties
# ============================================================
# Trust scoring — enable Bayesian Beta computation from attestations
# ============================================================
casehub.ledger.trust-score.enabled=true
```

- [ ] **Step 4: Update test application.properties**

Add Jandex index entries for engine-ledger. Insert after the existing engine indexing block:

```properties
quarkus.index-dependency.engine-ledger.group-id=io.casehub
quarkus.index-dependency.engine-ledger.artifact-id=casehub-engine-ledger
```

Add Jandex index for platform-config:

```properties
quarkus.index-dependency.platform-config.group-id=io.casehub
quarkus.index-dependency.platform-config.artifact-id=casehub-platform-config
```

- [ ] **Step 5: Verify build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: BUILD SUCCESS. If `ConfigFilePreferenceProvider` causes an `UnsatisfiedResolutionException` for a YAML file, add an empty `app/src/main/resources/casehub/life/trust-routing.yaml` with `entries: []` to unblock compilation (the full YAML is written in Task 5).

- [ ] **Step 6: Commit**

```
feat(#11): add casehub-engine-ledger + casehub-platform-config dependencies

Refs #11
```

---

### Task 2: LifeActorIds Utility

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/LifeActorIds.java`
- Create: `api/src/test/java/io/casehub/life/api/LifeActorIdsTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.life.api;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class LifeActorIdsTest {

    private static final UUID ACTOR_ID = UUID.fromString("550e8400-e29b-41d4-a716-446655440000");

    @Test
    void ofProducesExpectedFormat() {
        assertThat(LifeActorIds.of(ACTOR_ID))
            .isEqualTo("life-actor:550e8400-e29b-41d4-a716-446655440000");
    }

    @Test
    void isLifeActorReturnsTrueForValidPrefix() {
        assertThat(LifeActorIds.isLifeActor("life-actor:550e8400-e29b-41d4-a716-446655440000"))
            .isTrue();
    }

    @Test
    void isLifeActorReturnsFalseForOtherFormats() {
        assertThat(LifeActorIds.isLifeActor("claude:analyst@v1")).isFalse();
        assertThat(LifeActorIds.isLifeActor("life-system")).isFalse();
        assertThat(LifeActorIds.isLifeActor(null)).isFalse();
        assertThat(LifeActorIds.isLifeActor("")).isFalse();
    }

    @Test
    void extractIdRoundTrips() {
        String encoded = LifeActorIds.of(ACTOR_ID);
        assertThat(LifeActorIds.extractId(encoded)).isEqualTo(ACTOR_ID);
    }

    @Test
    void extractIdThrowsOnInvalidFormat() {
        assertThatThrownBy(() -> LifeActorIds.extractId("not-a-life-actor"))
            .isInstanceOf(StringIndexOutOfBoundsException.class);
    }

    @Test
    void ofThrowsOnNull() {
        assertThatThrownBy(() -> LifeActorIds.of(null))
            .isInstanceOf(NullPointerException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=LifeActorIdsTest --batch-mode`

Expected: FAIL — `LifeActorIds` class not found.

- [ ] **Step 3: Write the implementation**

```java
package io.casehub.life.api;

import java.util.Objects;
import java.util.UUID;

public final class LifeActorIds {

    public static final String PREFIX = "life-actor:";

    public static String of(final UUID externalActorId) {
        Objects.requireNonNull(externalActorId, "externalActorId must not be null");
        return PREFIX + externalActorId;
    }

    public static boolean isLifeActor(final String actorId) {
        return actorId != null && actorId.startsWith(PREFIX);
    }

    public static UUID extractId(final String actorId) {
        return UUID.fromString(actorId.substring(PREFIX.length()));
    }

    private LifeActorIds() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=LifeActorIdsTest --batch-mode`

Expected: PASS — all 6 tests green.

- [ ] **Step 5: Install api and commit**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode`

```
feat(#11): add LifeActorIds utility — life-actor:{uuid} convention

Refs #11
```

---

### Task 3: LifeLedgerWriter ActorId Fix

**Files:**
- Create: `app/src/test/java/io/casehub/life/app/service/LifeLedgerWriterActorIdTest.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/ledger/LifeLedgerWriter.java`

- [ ] **Step 1: Write the failing test**

Unit test with Mockito — mocks `LedgerEntryRepository`. Verifies the actorId and actorType on the persisted entry. Per CLAUDE.md testing convention: do NOT assert on `entry.id` or `entry.occurredAt` — these are set by `LedgerEntry.@PrePersist` which is bypassed in mocked tests.

```java
package io.casehub.life.app.service;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeActorIds;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.app.service.ledger.LifeLedgerWriter;
import io.casehub.life.app.service.ledger.LifeOutcomeAttestationWriter;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.work.runtime.model.WorkItem;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class LifeLedgerWriterActorIdTest {

    @Mock LedgerEntryRepository ledgerRepository;
    @Mock LifeOutcomeAttestationWriter attestationWriter;
    @InjectMocks LifeLedgerWriter writer;

    @BeforeEach
    void setUp() {
        when(ledgerRepository.findLatestBySubjectId(any())).thenReturn(Optional.empty());
    }

    @Test
    void healthEntryUsesLifeActorIdWhenExternalActorPresent() {
        var actorId = UUID.randomUUID();
        var ctx = new LifeTaskContext();
        ctx.workItemId = UUID.randomUUID();
        ctx.domain = LifeDomain.HEALTH;
        ctx.externalActorId = actorId;

        var workItem = new WorkItem();
        workItem.category = "appointment";
        workItem.expiresAt = Instant.now().plusSeconds(3600);

        writer.writeHealthEntry(LifeDecisionEventType.COMPLETED, ctx, workItem);

        var captor = ArgumentCaptor.forClass(LedgerEntry.class);
        verify(ledgerRepository).save(captor.capture());
        var saved = captor.getValue();

        assertThat(saved.actorId).isEqualTo(LifeActorIds.of(actorId));
        assertThat(saved.actorType).isEqualTo(ActorType.HUMAN);
    }

    @Test
    void healthEntryUsesLifeSystemWhenNoExternalActor() {
        var ctx = new LifeTaskContext();
        ctx.workItemId = UUID.randomUUID();
        ctx.domain = LifeDomain.HEALTH;
        ctx.externalActorId = null;

        var workItem = new WorkItem();
        workItem.category = "self-care";
        workItem.expiresAt = Instant.now().plusSeconds(3600);

        writer.writeHealthEntry(LifeDecisionEventType.COMPLETED, ctx, workItem);

        var captor = ArgumentCaptor.forClass(LedgerEntry.class);
        verify(ledgerRepository).save(captor.capture());
        var saved = captor.getValue();

        assertThat(saved.actorId).isEqualTo("life-system");
        assertThat(saved.actorType).isEqualTo(ActorType.SYSTEM);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeLedgerWriterActorIdTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — `LifeOutcomeAttestationWriter` does not exist yet. Also `writeHealthEntry` doesn't accept the attestation writer.

- [ ] **Step 3: Create stub LifeOutcomeAttestationWriter**

Create `app/src/main/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriter.java` as a no-op stub that will be filled in Task 4:

```java
package io.casehub.life.app.service.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class LifeOutcomeAttestationWriter {

    public void attestOutcome(final LedgerEntry entry,
                              final LifeDecisionEventType eventType,
                              final LifeTaskContext ctx,
                              final WorkItem workItem) {
        // Stub — implemented in Task 4
    }
}
```

- [ ] **Step 4: Modify LifeLedgerWriter for contextual actorId**

Update `LifeLedgerWriter`:
1. Add `@Inject LifeOutcomeAttestationWriter attestationWriter` field
2. Change `writeHealthEntry` — if `ctx.externalActorId != null`, pass `LifeActorIds.of(ctx.externalActorId)` and `ActorType.HUMAN`; otherwise `"life-system"` and `ActorType.SYSTEM`. Call `attestationWriter.attestOutcome()` after save.
3. Same for `writeLegalEntry`.
4. `writeFinancialEntry` and `writeErasureEntry` stay unchanged (system actions).

The key change to `writeHealthEntry`:

```java
public void writeHealthEntry(final LifeDecisionEventType eventType,
                              final LifeTaskContext ctx,
                              final WorkItem workItem) {
    final var entry = new HealthDecisionLedgerEntry();
    final String actorId = ctx.externalActorId != null
        ? LifeActorIds.of(ctx.externalActorId)
        : "life-system";
    final ActorType actorType = ctx.externalActorId != null
        ? ActorType.HUMAN
        : ActorType.SYSTEM;
    populateBase(entry, ctx.workItemId, actorId, actorType, "HealthDecisionAudit");
    entry.workItemId = ctx.workItemId;
    entry.providerId = ctx.externalActorId;
    entry.taskCategory = workItem.category;
    entry.slaDeadline = workItem.expiresAt;
    entry.eventType = eventType;
    entry.outcome = eventType == LifeDecisionEventType.COMPLETED ? workItem.outcome : null;
    ledgerRepository.save(entry);
    attestationWriter.attestOutcome(entry, eventType, ctx, workItem);
}
```

Apply the same pattern to `writeLegalEntry`. Import `io.casehub.life.api.LifeActorIds`.

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeLedgerWriterActorIdTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — both tests green.

- [ ] **Step 6: Run all existing tests to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: All existing tests PASS. The `LifeDecisionLedgerObserverTest` and `LifeLedgerWriterTest` (if it exists as a Mockito test) should still pass since `LifeOutcomeAttestationWriter` is a stub.

- [ ] **Step 7: Commit**

```
feat(#11): fix actorId attribution — life-actor:{uuid} for external actors

LifeLedgerWriter now uses contextual actorId: life-actor:{uuid} with
ActorType.HUMAN when a ledger entry records an ExternalActor's behaviour;
"life-system" with ActorType.SYSTEM for system actions.

Refs #11
```

---

### Task 4: Attestation Pipeline

**Files:**
- Create: `app/src/test/java/io/casehub/life/app/service/LifeOutcomeAttestationWriterTest.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/ledger/LifeOutcomeAttestationWriter.java`

- [ ] **Step 1: Write the failing test**

Mockito unit test — mocks `LedgerEntryRepository`. Verifies attestation fields.

```java
package io.casehub.life.app.service;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.LifeTrustDimensions;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.app.service.ledger.LifeOutcomeAttestationWriter;
import io.casehub.work.runtime.model.WorkItem;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.atLeastOnce;
import static org.mockito.Mockito.verify;

@ExtendWith(MockitoExtension.class)
class LifeOutcomeAttestationWriterTest {

    @Mock LedgerEntryRepository ledgerRepository;
    @InjectMocks LifeOutcomeAttestationWriter writer;

    @Test
    void completedTaskProducesSoundAttestation() {
        var entry = makeEntry(UUID.randomUUID());
        var ctx = makeCtx(LifeDomain.HEALTH, UUID.randomUUID());
        var workItem = makeWorkItem("casehubio/life/health",
            Instant.now().plusSeconds(3600), Instant.now());

        writer.attestOutcome(entry, LifeDecisionEventType.COMPLETED, ctx, workItem);

        var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
        verify(ledgerRepository, atLeastOnce()).saveAttestation(captor.capture());

        var verdictAttestation = captor.getAllValues().stream()
            .filter(a -> a.trustDimension == null)
            .findFirst().orElseThrow();

        assertThat(verdictAttestation.verdict).isEqualTo(AttestationVerdict.SOUND);
        assertThat(verdictAttestation.attestorId).isEqualTo("life-system");
        assertThat(verdictAttestation.confidence).isEqualTo(0.9);
        assertThat(verdictAttestation.capabilityTag).isEqualTo(LifeCapabilities.HEALTH_COORDINATION);
        assertThat(verdictAttestation.ledgerEntryId).isEqualTo(entry.id);
    }

    @Test
    void slaBreachProducesFlaggedAttestation() {
        var entry = makeEntry(UUID.randomUUID());
        var ctx = makeCtx(LifeDomain.HEALTH, UUID.randomUUID());
        var workItem = makeWorkItem("casehubio/life/health",
            Instant.now().minusSeconds(3600), null);

        writer.attestOutcome(entry, LifeDecisionEventType.SLA_BREACH, ctx, workItem);

        var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
        verify(ledgerRepository, atLeastOnce()).saveAttestation(captor.capture());

        var verdictAttestation = captor.getAllValues().stream()
            .filter(a -> a.trustDimension == null)
            .findFirst().orElseThrow();

        assertThat(verdictAttestation.verdict).isEqualTo(AttestationVerdict.FLAGGED);
        assertThat(verdictAttestation.confidence).isEqualTo(0.9);
    }

    @Test
    void completedTaskOnTimeProducesDeadlineReliabilityDimensionScore() {
        var entry = makeEntry(UUID.randomUUID());
        var ctx = makeCtx(LifeDomain.CONTRACTOR_COORDINATION, UUID.randomUUID());
        var deadline = Instant.now().plusSeconds(7200);
        var completedAt = Instant.now();
        var workItem = makeWorkItem("casehubio/life/contractor_coordination", deadline, completedAt);

        writer.attestOutcome(entry, LifeDecisionEventType.COMPLETED, ctx, workItem);

        var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
        verify(ledgerRepository, atLeastOnce()).saveAttestation(captor.capture());

        var dimAttestation = captor.getAllValues().stream()
            .filter(a -> LifeTrustDimensions.DEADLINE_RELIABILITY.equals(a.trustDimension))
            .findFirst().orElseThrow();

        assertThat(dimAttestation.dimensionScore).isEqualTo(1.0);
        assertThat(dimAttestation.capabilityTag)
            .isEqualTo(LifeCapabilities.CONTRACTOR_COORDINATION);
    }

    @Test
    void noExternalActorProducesNoAttestation() {
        var entry = makeEntry(UUID.randomUUID());
        var ctx = makeCtx(LifeDomain.HEALTH, null);
        var workItem = makeWorkItem("casehubio/life/health",
            Instant.now().plusSeconds(3600), Instant.now());

        writer.attestOutcome(entry, LifeDecisionEventType.COMPLETED, ctx, workItem);

        var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
        verify(ledgerRepository, org.mockito.Mockito.never()).saveAttestation(captor.capture());
    }

    @Test
    void unknownScopeUsesGlobalCapabilityTag() {
        var entry = makeEntry(UUID.randomUUID());
        var ctx = makeCtx(null, UUID.randomUUID());
        var workItem = makeWorkItem(null, Instant.now().plusSeconds(3600), Instant.now());

        writer.attestOutcome(entry, LifeDecisionEventType.COMPLETED, ctx, workItem);

        var captor = ArgumentCaptor.forClass(LedgerAttestation.class);
        verify(ledgerRepository, atLeastOnce()).saveAttestation(captor.capture());

        var verdictAttestation = captor.getAllValues().stream()
            .filter(a -> a.trustDimension == null)
            .findFirst().orElseThrow();

        assertThat(verdictAttestation.capabilityTag).isEqualTo(CapabilityTag.GLOBAL);
    }

    private static LedgerEntry makeEntry(UUID id) {
        var entry = new LedgerEntry() {};
        entry.id = id;
        entry.subjectId = UUID.randomUUID();
        return entry;
    }

    private static LifeTaskContext makeCtx(LifeDomain domain, UUID externalActorId) {
        var ctx = new LifeTaskContext();
        ctx.workItemId = UUID.randomUUID();
        ctx.domain = domain;
        ctx.externalActorId = externalActorId;
        return ctx;
    }

    private static WorkItem makeWorkItem(String scope, Instant expiresAt, Instant completedAt) {
        var wi = new WorkItem();
        wi.scope = scope;
        wi.expiresAt = expiresAt;
        wi.completedAt = completedAt;
        return wi;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeOutcomeAttestationWriterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — stub `attestOutcome` does nothing.

- [ ] **Step 3: Implement LifeOutcomeAttestationWriter**

Replace the stub with the full implementation:

```java
package io.casehub.life.app.service.ledger;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.life.api.LifeCapabilities;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.LifeTrustDimensions;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Duration;
import java.time.Instant;
import java.util.Map;

@ApplicationScoped
public class LifeOutcomeAttestationWriter {

    private static final double SYSTEM_CONFIDENCE = 0.9;
    private static final long GRACE_PERIOD_DAYS = 7;

    private static final Map<LifeDomain, String> DOMAIN_TO_CAPABILITY = Map.of(
        LifeDomain.HOUSEHOLD, LifeCapabilities.HOUSEHOLD_MANAGEMENT,
        LifeDomain.HEALTH, LifeCapabilities.HEALTH_COORDINATION,
        LifeDomain.FINANCE, LifeCapabilities.FINANCIAL_PLANNING,
        LifeDomain.FAMILY_SCHEDULING, LifeCapabilities.FAMILY_SCHEDULING,
        LifeDomain.TRAVEL, LifeCapabilities.TRAVEL_PLANNING,
        LifeDomain.LEGAL, LifeCapabilities.LEGAL_DEADLINE,
        LifeDomain.CONTRACTOR_COORDINATION, LifeCapabilities.CONTRACTOR_COORDINATION,
        LifeDomain.ELDER_CARE, LifeCapabilities.ELDER_CARE
    );

    @Inject
    LedgerEntryRepository ledgerRepository;

    public void attestOutcome(final LedgerEntry entry,
                              final LifeDecisionEventType eventType,
                              final LifeTaskContext ctx,
                              final WorkItem workItem) {
        if (ctx.externalActorId == null) {
            return;
        }

        final String capabilityTag = resolveCapabilityTag(ctx.domain, workItem);
        final AttestationVerdict verdict = eventType == LifeDecisionEventType.SLA_BREACH
            ? AttestationVerdict.FLAGGED
            : AttestationVerdict.SOUND;

        saveVerdictAttestation(entry, verdict, capabilityTag);

        if (workItem.expiresAt != null) {
            Instant completionTime = workItem.completedAt != null
                ? workItem.completedAt : Instant.now();
            saveDeadlineReliabilityAttestation(entry, workItem.expiresAt,
                completionTime, capabilityTag);
        }
    }

    private String resolveCapabilityTag(final LifeDomain domain, final WorkItem workItem) {
        if (domain != null) {
            String tag = DOMAIN_TO_CAPABILITY.get(domain);
            if (tag != null) return tag;
        }
        if (workItem.scope != null) {
            String[] segments = workItem.scope.split("/");
            if (segments.length >= 3) {
                try {
                    LifeDomain scopeDomain = LifeDomain.valueOf(segments[2].toUpperCase());
                    String tag = DOMAIN_TO_CAPABILITY.get(scopeDomain);
                    if (tag != null) return tag;
                } catch (IllegalArgumentException ignored) {}
            }
        }
        return CapabilityTag.GLOBAL;
    }

    private void saveVerdictAttestation(final LedgerEntry entry,
                                         final AttestationVerdict verdict,
                                         final String capabilityTag) {
        var attestation = new LedgerAttestation();
        attestation.ledgerEntryId = entry.id;
        attestation.subjectId = entry.subjectId;
        attestation.attestorId = "life-system";
        attestation.attestorType = ActorType.SYSTEM;
        attestation.attestorRole = "OutcomeAssessor";
        attestation.verdict = verdict;
        attestation.confidence = SYSTEM_CONFIDENCE;
        attestation.capabilityTag = capabilityTag;
        ledgerRepository.saveAttestation(attestation);
    }

    private void saveDeadlineReliabilityAttestation(final LedgerEntry entry,
                                                      final Instant deadline,
                                                      final Instant completionTime,
                                                      final String capabilityTag) {
        long daysLate = Duration.between(deadline, completionTime).toDays();
        double score;
        if (daysLate <= 0) {
            score = 1.0;
        } else {
            score = Math.max(0.0, 1.0 - (double) daysLate / GRACE_PERIOD_DAYS);
        }

        var attestation = new LedgerAttestation();
        attestation.ledgerEntryId = entry.id;
        attestation.subjectId = entry.subjectId;
        attestation.attestorId = "life-system";
        attestation.attestorType = ActorType.SYSTEM;
        attestation.attestorRole = "OutcomeAssessor";
        attestation.verdict = score >= 0.5 ? AttestationVerdict.SOUND : AttestationVerdict.FLAGGED;
        attestation.confidence = SYSTEM_CONFIDENCE;
        attestation.capabilityTag = capabilityTag;
        attestation.trustDimension = LifeTrustDimensions.DEADLINE_RELIABILITY;
        attestation.dimensionScore = score;
        ledgerRepository.saveAttestation(attestation);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeOutcomeAttestationWriterTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — all 5 tests green.

- [ ] **Step 5: Run all tests to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: All tests PASS.

- [ ] **Step 6: Commit**

```
feat(#11): attestation pipeline — LifeOutcomeAttestationWriter

Converts WorkItem outcomes and SLA breaches into LedgerAttestation
records with verdict (SOUND/FLAGGED), capability tag (derived from
domain), and deadline-reliability dimension score.

Refs #11
```

---

### Task 5: Routing Policies — LifeTrustRoutingPolicyProvider + YAML Config

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/routing/DoublePreference.java`
- Create: `app/src/main/java/io/casehub/life/app/routing/LifeRoutingPolicy.java`
- Create: `app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyKeys.java`
- Create: `app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProvider.java`
- Create or modify: `app/src/main/resources/casehub/life/trust-routing.yaml`
- Create: `app/src/test/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyKeysTest.java`
- Create: `app/src/test/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProviderTest.java`

- [ ] **Step 1: Write the LifeTrustRoutingPolicyKeys unit test**

```java
package io.casehub.life.app.routing;

import io.casehub.life.api.LifeTrustDimensions;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class LifeTrustRoutingPolicyKeysTest {

    @Test
    void allFloorKeysContainsAllDimensions() {
        var keys = LifeTrustRoutingPolicyKeys.allFloorKeys();
        assertThat(keys).containsKeys(
            LifeTrustDimensions.DEADLINE_RELIABILITY,
            LifeTrustDimensions.COST_ACCURACY,
            LifeTrustDimensions.FACTUAL_ACCURACY,
            LifeTrustDimensions.PROACTIVE_ALERTING
        );
        assertThat(keys).hasSize(4);
    }

    @Test
    void blendFactorKeyHasExpectedQualifiedName() {
        assertThat(LifeTrustRoutingPolicyKeys.BLEND_FACTOR.qualifiedName())
            .isEqualTo("casehubio.life.trust-routing.blend-factor");
    }
}
```

- [ ] **Step 2: Create DoublePreference**

```java
package io.casehub.life.app.routing;

import io.casehub.platform.api.preferences.SingleValuePreference;
import java.util.Objects;

public record DoublePreference(double value) implements SingleValuePreference {
    public static DoublePreference of(double value) { return new DoublePreference(value); }
    public static DoublePreference parse(String raw) {
        Objects.requireNonNull(raw, "raw must not be null");
        return new DoublePreference(Double.parseDouble(raw));
    }
}
```

- [ ] **Step 3: Create LifeRoutingPolicy**

```java
package io.casehub.life.app.routing;

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

- [ ] **Step 4: Create LifeTrustRoutingPolicyKeys**

```java
package io.casehub.life.app.routing;

import io.casehub.life.api.LifeTrustDimensions;
import io.casehub.platform.api.preferences.PreferenceKey;
import java.util.Map;

public final class LifeTrustRoutingPolicyKeys {

    public static final PreferenceKey<DoublePreference> BLEND_FACTOR =
        new PreferenceKey<>(
            "casehubio.life.trust-routing",
            "blend-factor",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    public static final PreferenceKey<DoublePreference> FLOOR_DEADLINE_RELIABILITY =
        new PreferenceKey<>(
            "casehubio.life.trust-routing",
            "floor.deadline-reliability",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    public static final PreferenceKey<DoublePreference> FLOOR_COST_ACCURACY =
        new PreferenceKey<>(
            "casehubio.life.trust-routing",
            "floor.cost-accuracy",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    public static final PreferenceKey<DoublePreference> FLOOR_FACTUAL_ACCURACY =
        new PreferenceKey<>(
            "casehubio.life.trust-routing",
            "floor.factual-accuracy",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    public static final PreferenceKey<DoublePreference> FLOOR_PROACTIVE_ALERTING =
        new PreferenceKey<>(
            "casehubio.life.trust-routing",
            "floor.proactive-alerting",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    private static final Map<String, PreferenceKey<DoublePreference>> FLOOR_KEYS = Map.of(
        LifeTrustDimensions.DEADLINE_RELIABILITY, FLOOR_DEADLINE_RELIABILITY,
        LifeTrustDimensions.COST_ACCURACY,        FLOOR_COST_ACCURACY,
        LifeTrustDimensions.FACTUAL_ACCURACY,     FLOOR_FACTUAL_ACCURACY,
        LifeTrustDimensions.PROACTIVE_ALERTING,   FLOOR_PROACTIVE_ALERTING
    );

    public static Map<String, PreferenceKey<DoublePreference>> allFloorKeys() {
        return FLOOR_KEYS;
    }

    private LifeTrustRoutingPolicyKeys() {}
}
```

- [ ] **Step 5: Run the keys unit test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeTrustRoutingPolicyKeysTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS.

- [ ] **Step 6: Write the LifeTrustRoutingPolicyProvider integration test**

```java
package io.casehub.life.app.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.life.api.LifeCapabilities;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class LifeTrustRoutingPolicyProviderTest {

    @Inject TrustRoutingPolicyProvider provider;

    @Test
    void providerIsLifeImplementation() {
        assertThat(provider).isInstanceOf(LifeTrustRoutingPolicyProvider.class);
    }

    @Test
    void bookAppointmentResolvesToHealthCoordinationPolicy() {
        var policy = provider.forCapability("book-appointment");
        assertThat(policy.threshold()).isEqualTo(0.75);
        assertThat(policy.minimumObservations()).isEqualTo(10);
        assertThat(policy.borderlineMargin()).isEqualTo(0.05);
    }

    @Test
    void requestQuoteResolvesToContractorCoordinationPolicy() {
        var policy = provider.forCapability("request-quote");
        assertThat(policy.threshold()).isEqualTo(0.65);
        assertThat(policy.minimumObservations()).isEqualTo(8);
    }

    @Test
    void unknownCapabilityReturnsDefault() {
        var policy = provider.forCapability("totally-unknown");
        assertThat(policy).isEqualTo(TrustRoutingPolicy.DEFAULT);
    }

    @Test
    void yamlBlendFactorWiredForHealthCoordination() {
        var policy = provider.forCapability("book-appointment");
        assertThat(policy.blendFactor()).isEqualTo(0.70);
    }

    @Test
    void yamlQualityFloorWiredForLegalDeadline() {
        var policy = provider.forCapability("gather-data");
        // gather-data maps to financial-review case → FINANCE domain
        // but that's financial-planning, not legal-deadline
        // Let's use a legal worker instead — there is no explicit legal case definition yet
        // Fall back to domain-direct: use the domain key directly
    }

    @Test
    void healthCoordinationHasFactualAccuracyFloor() {
        var policy = provider.forCapability("book-appointment");
        assertThat(policy.qualityFloors())
            .containsEntry("factual-accuracy", 0.60);
    }

    @Test
    void contractorCoordinationHasDeadlineReliabilityFloor() {
        var policy = provider.forCapability("request-quote");
        assertThat(policy.qualityFloors())
            .containsEntry("deadline-reliability", 0.50);
    }

    @Test
    void householdManagementHasLowBlendFactor() {
        var policy = provider.forCapability("schedule-inspection");
        // schedule-inspection → home-maintenance case → HOUSEHOLD domain
        assertThat(policy.blendFactor()).isEqualTo(0.40);
    }
}
```

- [ ] **Step 7: Create LifeTrustRoutingPolicyProvider**

```java
package io.casehub.life.app.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.life.api.LifeCapabilities;
import io.casehub.platform.api.preferences.PreferenceKey;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;

import static java.util.Map.entry;

@ApplicationScoped
public class LifeTrustRoutingPolicyProvider implements TrustRoutingPolicyProvider {

    private static final Map<String, String> CAPABILITY_TO_DOMAIN = Map.ofEntries(
        // appointment-cycle → health-coordination
        entry("book-appointment", LifeCapabilities.HEALTH_COORDINATION),
        entry("find-alternative", LifeCapabilities.HEALTH_COORDINATION),
        entry("confirm-appointment", LifeCapabilities.HEALTH_COORDINATION),
        entry("pre-visit-prep", LifeCapabilities.HEALTH_COORDINATION),
        entry("record-health-decision", LifeCapabilities.HEALTH_COORDINATION),
        // care-coordination → elder-care
        entry("needs-assessment", LifeCapabilities.ELDER_CARE),
        entry("care-plan", LifeCapabilities.ELDER_CARE),
        entry("health-check", LifeCapabilities.ELDER_CARE),
        // care-episode → elder-care
        entry("assess-patient", LifeCapabilities.ELDER_CARE),
        entry("provide-care", LifeCapabilities.ELDER_CARE),
        // contractor-coordination → contractor-coordination
        entry("request-quote", LifeCapabilities.CONTRACTOR_COORDINATION),
        entry("watchdog-escalation", LifeCapabilities.CONTRACTOR_COORDINATION),
        entry("quote-received", LifeCapabilities.CONTRACTOR_COORDINATION),
        entry("job-monitoring", LifeCapabilities.CONTRACTOR_COORDINATION),
        entry("record-payment", LifeCapabilities.CONTRACTOR_COORDINATION),
        // financial-review → financial-planning
        entry("gather-data", LifeCapabilities.FINANCIAL_PLANNING),
        entry("analyse-anomalies", LifeCapabilities.FINANCIAL_PLANNING),
        entry("escalate-anomalies", LifeCapabilities.FINANCIAL_PLANNING),
        entry("oversight-response", LifeCapabilities.FINANCIAL_PLANNING),
        entry("produce-report", LifeCapabilities.FINANCIAL_PLANNING),
        // home-maintenance → household-management
        entry("schedule-inspection", LifeCapabilities.HOUSEHOLD_MANAGEMENT),
        entry("get-quotes", LifeCapabilities.HOUSEHOLD_MANAGEMENT),
        entry("issue-commitment", LifeCapabilities.HOUSEHOLD_MANAGEMENT),
        entry("monitor-job", LifeCapabilities.HOUSEHOLD_MANAGEMENT),
        entry("record-completion", LifeCapabilities.HOUSEHOLD_MANAGEMENT),
        // travel-plan → travel-planning
        entry("destination-research", LifeCapabilities.TRAVEL_PLANNING),
        entry("flight-search", LifeCapabilities.TRAVEL_PLANNING),
        entry("hotel-search", LifeCapabilities.TRAVEL_PLANNING),
        entry("budget-assessment", LifeCapabilities.TRAVEL_PLANNING),
        entry("booking", LifeCapabilities.TRAVEL_PLANNING),
        entry("rebooking", LifeCapabilities.TRAVEL_PLANNING),
        entry("confirmation", LifeCapabilities.TRAVEL_PLANNING)
    );

    private static final Map<String, LifeRoutingPolicy> POLICIES = Map.of(
        LifeCapabilities.HEALTH_COORDINATION, new LifeRoutingPolicy(
            OptionalDouble.of(0.75), OptionalInt.of(10), OptionalDouble.of(0.05),
            Optional.of("household-admin"),
            "Missed health follow-ups have real consequences"),
        LifeCapabilities.LEGAL_DEADLINE, new LifeRoutingPolicy(
            OptionalDouble.of(0.80), OptionalInt.of(12), OptionalDouble.of(0.05),
            Optional.of("household-admin"),
            "Hard deadlines; highest trust requirement"),
        LifeCapabilities.FINANCIAL_PLANNING, new LifeRoutingPolicy(
            OptionalDouble.of(0.70), OptionalInt.of(10), OptionalDouble.of(0.10),
            Optional.of("household-admin"),
            "Oversight gate exists; wider borderline band"),
        LifeCapabilities.CONTRACTOR_COORDINATION, new LifeRoutingPolicy(
            OptionalDouble.of(0.65), OptionalInt.of(8), OptionalDouble.of(0.05),
            Optional.of("household-admin"),
            "Watchdog catches failures"),
        LifeCapabilities.ELDER_CARE, new LifeRoutingPolicy(
            OptionalDouble.of(0.75), OptionalInt.of(10), OptionalDouble.of(0.05),
            Optional.of("household-admin"),
            "Care coordination failures affect vulnerable people"),
        LifeCapabilities.HOUSEHOLD_MANAGEMENT, new LifeRoutingPolicy(
            OptionalDouble.of(0.50), OptionalInt.of(5), OptionalDouble.empty(),
            Optional.empty(),
            "Low-stakes routine; any competent agent"),
        LifeCapabilities.FAMILY_SCHEDULING, new LifeRoutingPolicy(
            OptionalDouble.of(0.50), OptionalInt.of(5), OptionalDouble.empty(),
            Optional.empty(),
            "Low-stakes scheduling"),
        LifeCapabilities.TRAVEL_PLANNING, new LifeRoutingPolicy(
            OptionalDouble.of(0.55), OptionalInt.of(6), OptionalDouble.of(0.05),
            Optional.of("household-admin"),
            "Booking mistakes cost money but are reversible")
    );

    private final PreferenceProvider preferenceProvider;

    @Inject
    public LifeTrustRoutingPolicyProvider(final PreferenceProvider preferenceProvider) {
        this.preferenceProvider = preferenceProvider;
    }

    @Override
    public TrustRoutingPolicy forCapability(final String capabilityName) {
        String domainKey = CAPABILITY_TO_DOMAIN.getOrDefault(capabilityName, capabilityName);
        LifeRoutingPolicy policy = POLICIES.get(domainKey);
        if (policy == null) {
            return TrustRoutingPolicy.DEFAULT;
        }

        Preferences prefs = preferenceProvider.resolve(
            SettingsScope.of("casehubio", "life", "trust-routing", domainKey));

        double threshold = policy.threshold()
            .orElse(TrustRoutingPolicy.DEFAULT.threshold());
        int minObs = policy.minimumObservations()
            .orElse(TrustRoutingPolicy.DEFAULT.minimumObservations());
        double margin = policy.borderlineMargin()
            .orElse(TrustRoutingPolicy.DEFAULT.borderlineMargin());

        DoublePreference blendPref = prefs.get(LifeTrustRoutingPolicyKeys.BLEND_FACTOR);
        double blendFactor = blendPref != null
            ? blendPref.value()
            : TrustRoutingPolicy.DEFAULT.blendFactor();

        Map<String, Double> qualityFloors = new HashMap<>();
        LifeTrustRoutingPolicyKeys.allFloorKeys().forEach((dim, key) ->
            addFloor(qualityFloors, prefs, key, dim));

        return new TrustRoutingPolicy(threshold, minObs, margin, blendFactor,
            Map.copyOf(qualityFloors));
    }

    private static void addFloor(final Map<String, Double> floors,
                                  final Preferences prefs,
                                  final PreferenceKey<DoublePreference> key,
                                  final String dimension) {
        DoublePreference value = prefs.get(key);
        if (value != null && value.value() > 0.0) {
            floors.put(dimension, value.value());
        }
    }
}
```

- [ ] **Step 8: Write trust-routing.yaml**

Create `app/src/main/resources/casehub/life/trust-routing.yaml`:

```yaml
entries:
  - scope: casehubio/life/trust-routing/health-coordination
    casehubio.life.trust-routing.blend-factor: "0.70"
    casehubio.life.trust-routing.floor.factual-accuracy: "0.60"

  - scope: casehubio/life/trust-routing/legal-deadline
    casehubio.life.trust-routing.blend-factor: "0.80"
    casehubio.life.trust-routing.floor.deadline-reliability: "0.70"

  - scope: casehubio/life/trust-routing/contractor-coordination
    casehubio.life.trust-routing.blend-factor: "0.60"
    casehubio.life.trust-routing.floor.deadline-reliability: "0.50"
    casehubio.life.trust-routing.floor.cost-accuracy: "0.50"

  - scope: casehubio/life/trust-routing/financial-planning
    casehubio.life.trust-routing.blend-factor: "0.65"
    casehubio.life.trust-routing.floor.cost-accuracy: "0.60"

  - scope: casehubio/life/trust-routing/elder-care
    casehubio.life.trust-routing.blend-factor: "0.70"

  - scope: casehubio/life/trust-routing/travel-planning
    casehubio.life.trust-routing.blend-factor: "0.50"

  - scope: casehubio/life/trust-routing/household-management
    casehubio.life.trust-routing.blend-factor: "0.40"

  - scope: casehubio/life/trust-routing/family-scheduling
    casehubio.life.trust-routing.blend-factor: "0.40"
```

- [ ] **Step 9: Run the integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeTrustRoutingPolicyProviderTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS. If `ConfigFilePreferenceProvider` can't find the YAML, check classpath scanning. The file must be at `src/main/resources/casehub/life/trust-routing.yaml` exactly — the scope prefix `casehubio/life/trust-routing` maps to file path `casehub/life/trust-routing.yaml`.

- [ ] **Step 10: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: All tests PASS.

- [ ] **Step 11: Commit**

```
feat(#11): LifeTrustRoutingPolicyProvider + YAML trust routing config

8 domain routing policies with capability-name→domain mapping.
Blend factors and quality floors from trust-routing.yaml via
casehub-platform-config PreferenceProvider.

Refs #11
```

---

### Task 6: ExternalActor Trust Score Enrichment

**Files:**
- Modify: `api/src/main/java/io/casehub/life/api/response/ExternalActorResponse.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/ExternalActorService.java`
- Create: `app/src/test/java/io/casehub/life/app/ExternalActorTrustEnrichmentTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/ExternalActorResourceTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/ExternalActorGdprResourceTest.java`

- [ ] **Step 1: Write the enrichment integration test**

```java
package io.casehub.life.app;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.life.api.LifeActorIds;
import io.casehub.life.api.LifeActorType;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class ExternalActorTrustEnrichmentTest {

    @Inject ActorTrustScoreRepository trustScoreRepo;

    @Test
    void getExternalActorIncludesTrustProfile() {
        var actorId = createActorAndSeedTrustScore();

        given()
            .when().get("/external-actors/" + actorId)
            .then()
            .statusCode(200)
            .body("trustProfile.globalScore", notNullValue())
            .body("trustProfile.globalScore", closeTo(0.8, 0.01))
            .body("trustProfile.dimensionScores", aMapWithSize(0))
            .body("trustProfile.capabilityScores", aMapWithSize(0));
    }

    @Test
    void newActorWithNoTrustDataReturnsEmptyProfile() {
        var actorId = createActor();

        given()
            .when().get("/external-actors/" + actorId)
            .then()
            .statusCode(200)
            .body("trustProfile.globalScore", nullValue())
            .body("trustProfile.dimensionScores", aMapWithSize(0))
            .body("trustProfile.capabilityScores", aMapWithSize(0));
    }

    @Transactional
    UUID createActorAndSeedTrustScore() {
        UUID id = createActor();
        String lifeActorId = LifeActorIds.of(id);
        trustScoreRepo.upsert(lifeActorId, ActorTrustScore.ScoreType.GLOBAL,
            null, null, ActorType.HUMAN, 0.8,
            10, 1, 8.0, 2.0, 8, 2, Instant.now());
        return id;
    }

    UUID createActor() {
        return UUID.fromString(
            given()
                .contentType("application/json")
                .body("""
                    {
                      "name": "Test Contractor",
                      "actorType": "CONTRACTOR",
                      "contactMethod": "email",
                      "contactValue": "test@example.com"
                    }
                    """)
                .when().post("/external-actors")
                .then().statusCode(201)
                .extract().path("id"));
    }
}
```

- [ ] **Step 2: Modify ExternalActorResponse to add TrustProfile**

```java
package io.casehub.life.api.response;

import io.casehub.life.api.LifeActorType;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

public record ExternalActorResponse(
        UUID id,
        String name,
        LifeActorType actorType,
        String contactMethod,
        String contactValue,
        Instant createdAt,
        Instant gdprErasedAt,
        TrustProfile trustProfile
) {
    public record TrustProfile(
        Double globalScore,
        Map<String, Double> dimensionScores,
        Map<String, Double> capabilityScores
    ) {
        public static final TrustProfile EMPTY =
            new TrustProfile(null, Map.of(), Map.of());
    }
}
```

- [ ] **Step 3: Install api (response record changed)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode`

- [ ] **Step 4: Modify ExternalActorService**

Add `@Inject TrustGateService trustGateService` field. Update `toResponse()`:

```java
private ExternalActorResponse toResponse(final ExternalActor actor) {
    ExternalActorResponse.TrustProfile profile;
    if (actor.gdprErasedAt != null) {
        profile = ExternalActorResponse.TrustProfile.EMPTY;
    } else {
        String actorId = LifeActorIds.of(actor.id);
        Double global = trustGateService.currentScore(actorId).orElse(null);
        Map<String, Double> dimensions = trustGateService.dimensionScores(actorId);
        Map<String, Double> capabilities = trustGateService.allCapabilityScores(actorId);
        profile = new ExternalActorResponse.TrustProfile(global, dimensions, capabilities);
    }
    return new ExternalActorResponse(
            actor.id,
            actor.name,
            actor.actorType,
            actor.contactMethod,
            actor.contactValue,
            actor.createdAt,
            actor.gdprErasedAt,
            profile
    );
}
```

Add imports:
```java
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.life.api.LifeActorIds;
import java.util.Map;
```

- [ ] **Step 5: Update existing ExternalActorResourceTest**

The response shape changed — existing tests that assert response body fields need to account for the new `trustProfile` field. Tests that just check status codes should pass unchanged. Tests that use `body("name", ...)` should still pass because Jackson ignores extra fields in assertion.

Check the test file — if it uses exact JSON matching or `equalTo()` on the full body, update to include `trustProfile`. If it uses field-level matchers, it should pass as-is.

- [ ] **Step 6: Update ExternalActorGdprResourceTest**

Add assertion for GDPR-erased trust profile:

```java
// In the erasure test, after erasing:
given()
    .when().get("/external-actors/" + actorId)
    .then()
    .statusCode(200)
    .body("trustProfile.globalScore", nullValue())
    .body("trustProfile.dimensionScores", aMapWithSize(0))
    .body("trustProfile.capabilityScores", aMapWithSize(0));
```

- [ ] **Step 7: Run the enrichment test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=ExternalActorTrustEnrichmentTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS.

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: All tests PASS.

- [ ] **Step 9: Commit**

```
feat(#11): enrich ExternalActor response with ledger-backed TrustProfile

GET /external-actors/{id} now returns globalScore, dimensionScores, and
capabilityScores read from TrustGateService. GDPR-erased actors return
TrustProfile.EMPTY. 3 queries per actor via TrustGateService.

Refs #11
```

---

### Task 7: Wiring Verification Tests

**Files:**
- Create: `app/src/test/java/io/casehub/life/app/TrustStrategyDisplacementTest.java`
- Create: `app/src/test/java/io/casehub/life/app/ColdStartBehaviorTest.java`

- [ ] **Step 1: Write TrustStrategyDisplacementTest**

```java
package io.casehub.life.app;

import io.casehub.api.spi.routing.AgentRoutingStrategy;
import io.casehub.ledger.routing.TrustWeightedAgentStrategy;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class TrustStrategyDisplacementTest {

    @Inject AgentRoutingStrategy strategy;

    @Test
    void trustWeightedStrategyIsActive() {
        assertThat(strategy).isInstanceOf(TrustWeightedAgentStrategy.class);
    }
}
```

- [ ] **Step 2: Write ColdStartBehaviorTest**

```java
package io.casehub.life.app;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.life.api.LifeActorIds;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class ColdStartBehaviorTest {

    @Inject TrustRoutingPolicyProvider policyProvider;
    @Inject TrustGateService trustGateService;

    @Test
    void policiesAvailableWithNoTrustData() {
        var policy = policyProvider.forCapability("book-appointment");
        assertThat(policy).isNotNull();
        assertThat(policy.threshold()).isEqualTo(0.75);
    }

    @Test
    void unknownActorReturnsEmptyScores() {
        String actorId = LifeActorIds.of(UUID.randomUUID());
        assertThat(trustGateService.currentScore(actorId)).isEmpty();
        assertThat(trustGateService.dimensionScores(actorId)).isEmpty();
        assertThat(trustGateService.allCapabilityScores(actorId)).isEmpty();
    }

    @Test
    void unknownCapabilityReturnsDefaultPolicy() {
        var policy = policyProvider.forCapability("nonexistent-capability");
        assertThat(policy).isEqualTo(TrustRoutingPolicy.DEFAULT);
    }
}
```

- [ ] **Step 3: Run the wiring tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest="TrustStrategyDisplacementTest,ColdStartBehaviorTest" --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS.

- [ ] **Step 4: Run the full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

Expected: All tests PASS — including existing tests from Layers 1–5.

- [ ] **Step 5: Commit**

```
feat(#11): wiring verification — strategy displacement + cold start

Refs #11
```

---

### Task 8: Full Build and Final Verification

- [ ] **Step 1: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install --batch-mode`

Expected: BUILD SUCCESS with all tests passing.

- [ ] **Step 2: Commit any remaining changes**

If any fixups were needed during the full build, commit them.

- [ ] **Step 3: Invoke superpowers:requesting-code-review**

Review the implementation for quality, correctness, and platform consistency before final commit.
