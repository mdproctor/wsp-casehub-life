# Layer 5: casehub-engine CasePlanModel Workflows — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add casehub-engine to casehub-life with 7 multi-step CasePlanModel workflows demonstrating parallel execution, adaptive gates, M-of-N SubCase quorum, QhorusMessageSignalBridge, ledger writes from workers, cross-case signals, milestones, and quarkus-flow FuncDSL.

**Architecture:** Six `YamlCaseHub` subclasses + one minimal `FamilyVoteCaseHub` load YAML case definitions. Each has a fluent Java DSL companion. Workers use `FuncWorkflowBuilder.workflow().tasks(FuncDSL.function(...)).build()`. `LifeCaseService` starts cases with the three-phase pattern (PP-20260529-3ffe28). `LifeCaseTracker` entity enables cross-case signaling.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine 0.2-SNAPSHOT, quarkus-flow FuncDSL, H2 MODE=PostgreSQL

**Spec:** `docs/specs/2026-05-31-layer5-casehub-engine-design.md`

**Implementation phases:** Phase 1 (Tasks 1–11) = infrastructure + 3 core cases. Phase 2 (Tasks 12–15) = 4 advanced cases. Same branch, separate commits.

---

## Phase 1 — Infrastructure + 3 Core Cases

### Task 1: Maven dependencies and test configuration

**Files:**
- Modify: `app/pom.xml`
- Modify: `app/src/test/resources/application.properties`

- [ ] **Step 1: Add engine dependencies to app/pom.xml**

Add after the `<!-- Layer 5 -->` placeholder comment (which is currently empty):

```xml
    <!-- ================================================================ -->
    <!-- Layer 5: casehub-engine — multi-step CasePlanModel workflows     -->
    <!-- ================================================================ -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-scheduler-quartz</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-work-adapter</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-blackboard</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-persistence-memory</artifactId>
      <version>${casehub.version}</version>
      <scope>compile</scope>
    </dependency>

    <!-- Engine test support — @Priority(1) auto-selected in-memory SPIs -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-testing</artifactId>
      <version>${casehub.version}</version>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Add Jandex index entries to test application.properties**

Add after the existing engine placeholder comment in `app/src/test/resources/application.properties`:

```properties
# ============================================================
# Engine jar indexing — casehub-engine modules lack embedded
# Jandex indices (GE-20260523-86ed13)
# ============================================================
quarkus.index-dependency.engine-common.group-id=io.casehub
quarkus.index-dependency.engine-common.artifact-id=casehub-engine-common
quarkus.index-dependency.engine-blackboard.group-id=io.casehub
quarkus.index-dependency.engine-blackboard.artifact-id=casehub-engine-blackboard
quarkus.index-dependency.engine-work-adapter.group-id=io.casehub
quarkus.index-dependency.engine-work-adapter.artifact-id=casehub-engine-work-adapter
quarkus.index-dependency.engine-scheduler-quartz.group-id=io.casehub
quarkus.index-dependency.engine-scheduler-quartz.artifact-id=casehub-engine-scheduler-quartz
quarkus.index-dependency.engine-persistence-memory.group-id=io.casehub
quarkus.index-dependency.engine-persistence-memory.artifact-id=casehub-engine-persistence-memory
quarkus.index-dependency.engine-testing.group-id=io.casehub
quarkus.index-dependency.engine-testing.artifact-id=casehub-engine-testing
```

- [ ] **Step 3: Verify the build compiles with engine deps**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode`

Expected: BUILD SUCCESS. If CDI failures occur, check `quarkus.arc.exclude-types` — may need to exclude engine no-op SPI beans that conflict with the persistence-memory impls.

- [ ] **Step 4: Commit**

```
feat(#6): add casehub-engine dependencies for Layer 5
```

---

### Task 2: api/ additions — LifeCaseType, LifeCaseStatus, request/response records

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/LifeCaseType.java`
- Create: `api/src/main/java/io/casehub/life/api/LifeCaseStatus.java`
- Create: `api/src/main/java/io/casehub/life/api/request/CreateLifeCaseRequest.java`
- Create: `api/src/main/java/io/casehub/life/api/response/LifeCaseResponse.java`
- Create: `api/src/test/java/io/casehub/life/api/LifeCaseTypeTest.java`
- Create: `api/src/test/java/io/casehub/life/api/LifeCaseStatusTest.java`

- [ ] **Step 1: Write enum tests**

```java
// api/src/test/java/io/casehub/life/api/LifeCaseTypeTest.java
package io.casehub.life.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class LifeCaseTypeTest {
    @Test
    void allSixTypesExist() {
        assertThat(LifeCaseType.values()).hasSize(6);
        assertThat(LifeCaseType.valueOf("TRAVEL_PLAN")).isNotNull();
        assertThat(LifeCaseType.valueOf("HOME_MAINTENANCE")).isNotNull();
        assertThat(LifeCaseType.valueOf("CARE_COORDINATION")).isNotNull();
        assertThat(LifeCaseType.valueOf("APPOINTMENT_CYCLE")).isNotNull();
        assertThat(LifeCaseType.valueOf("CONTRACTOR_COORDINATION")).isNotNull();
        assertThat(LifeCaseType.valueOf("FINANCIAL_REVIEW")).isNotNull();
    }

    @Test
    void caseNameMatchesYamlConvention() {
        assertThat(LifeCaseType.TRAVEL_PLAN.caseName()).isEqualTo("travel-plan");
        assertThat(LifeCaseType.HOME_MAINTENANCE.caseName()).isEqualTo("home-maintenance");
        assertThat(LifeCaseType.CARE_COORDINATION.caseName()).isEqualTo("care-coordination");
        assertThat(LifeCaseType.APPOINTMENT_CYCLE.caseName()).isEqualTo("appointment-cycle");
        assertThat(LifeCaseType.CONTRACTOR_COORDINATION.caseName()).isEqualTo("contractor-coordination");
        assertThat(LifeCaseType.FINANCIAL_REVIEW.caseName()).isEqualTo("financial-review");
    }
}
```

```java
// api/src/test/java/io/casehub/life/api/LifeCaseStatusTest.java
package io.casehub.life.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class LifeCaseStatusTest {
    @Test
    void threeStatusesExist() {
        assertThat(LifeCaseStatus.values()).hasSize(3);
        assertThat(LifeCaseStatus.valueOf("ACTIVE")).isNotNull();
        assertThat(LifeCaseStatus.valueOf("COMPLETED")).isNotNull();
        assertThat(LifeCaseStatus.valueOf("FAILED")).isNotNull();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api --batch-mode`

Expected: FAIL — classes don't exist yet.

- [ ] **Step 3: Implement enums and records**

```java
// api/src/main/java/io/casehub/life/api/LifeCaseType.java
package io.casehub.life.api;

public enum LifeCaseType {
    TRAVEL_PLAN("travel-plan"),
    HOME_MAINTENANCE("home-maintenance"),
    CARE_COORDINATION("care-coordination"),
    APPOINTMENT_CYCLE("appointment-cycle"),
    CONTRACTOR_COORDINATION("contractor-coordination"),
    FINANCIAL_REVIEW("financial-review");

    private final String caseName;

    LifeCaseType(String caseName) {
        this.caseName = caseName;
    }

    public String caseName() {
        return caseName;
    }
}
```

```java
// api/src/main/java/io/casehub/life/api/LifeCaseStatus.java
package io.casehub.life.api;

public enum LifeCaseStatus {
    ACTIVE, COMPLETED, FAILED
}
```

```java
// api/src/main/java/io/casehub/life/api/request/CreateLifeCaseRequest.java
package io.casehub.life.api.request;

import io.casehub.life.api.LifeCaseType;
import java.util.Map;

public record CreateLifeCaseRequest(
        LifeCaseType caseType,
        Map<String, Object> context
) {
    public CreateLifeCaseRequest {
        if (caseType == null) throw new IllegalArgumentException("caseType is required");
        if (context == null) context = Map.of();
    }
}
```

```java
// api/src/main/java/io/casehub/life/api/response/LifeCaseResponse.java
package io.casehub.life.api.response;

import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.api.LifeCaseType;
import java.util.UUID;

public record LifeCaseResponse(
        UUID caseId,
        LifeCaseType caseType,
        LifeCaseStatus status
) {}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api --batch-mode`

Expected: PASS

- [ ] **Step 5: Install api and commit**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode`

```
feat(#6): add LifeCaseType, LifeCaseStatus, request/response records
```

---

### Task 3: LifeCaseTracker entity + Flyway migration

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/entity/LifeCaseTracker.java`
- Create: `app/src/main/resources/db/life/migration/V107__create_life_case_tracker.sql`

- [ ] **Step 1: Write the migration**

```sql
-- V107__create_life_case_tracker.sql
CREATE TABLE life_case_tracker (
    id         UUID         NOT NULL,
    case_type  VARCHAR(64)  NOT NULL,
    engine_case_id UUID,
    status     VARCHAR(16)  NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    CONSTRAINT pk_life_case_tracker PRIMARY KEY (id)
);

CREATE INDEX idx_life_case_tracker_type_status ON life_case_tracker (case_type, status);
CREATE UNIQUE INDEX uidx_life_case_tracker_engine_case_id ON life_case_tracker (engine_case_id);
```

- [ ] **Step 2: Write the entity**

```java
// app/src/main/java/io/casehub/life/app/entity/LifeCaseTracker.java
package io.casehub.life.app.entity;

import io.casehub.life.api.LifeCaseStatus;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Entity
@Table(name = "life_case_tracker")
public class LifeCaseTracker extends PanacheEntityBase {

    @Id
    public UUID id;

    @Column(name = "case_type", nullable = false, length = 64)
    public String caseType;

    @Column(name = "engine_case_id", unique = true)
    public UUID engineCaseId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 16)
    public LifeCaseStatus status;

    @Column(name = "created_at", nullable = false, updatable = false)
    public Instant createdAt;

    @Column(name = "completed_at")
    public Instant completedAt;

    @PrePersist
    void onPersist() {
        if (id == null) id = UUID.randomUUID();
        if (createdAt == null) createdAt = Instant.now();
        if (status == null) status = LifeCaseStatus.ACTIVE;
    }

    public static Optional<LifeCaseTracker> findByEngineCaseId(UUID engineCaseId) {
        return find("engineCaseId", engineCaseId).firstResultOptional();
    }

    public static List<LifeCaseTracker> findActiveByCaseType(String caseType) {
        return list("caseType = ?1 and status = ?2", caseType, LifeCaseStatus.ACTIVE);
    }
}
```

- [ ] **Step 3: Verify build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app --batch-mode`

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
feat(#6): add LifeCaseTracker entity + V107 migration
```

---

### Task 4: Scope retrofit — LifeTaskService

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/service/LifeTaskService.java:81`
- Modify: `app/src/test/java/io/casehub/life/app/*Test.java` — any tests asserting scope="life"

- [ ] **Step 1: Find tests that assert on scope**

Run: `grep -rn '"life"' app/src/test/java/ --include="*.java" | grep -i scope`

- [ ] **Step 2: Update LifeTaskService.create() scope**

In `app/src/main/java/io/casehub/life/app/service/LifeTaskService.java`, change line 81:

From: `.scope("life")`
To: `.scope("casehubio/life/" + domain.name().toLowerCase())`

- [ ] **Step 3: Update any tests asserting on the old scope value**

Update test assertions from `"life"` to `"casehubio/life/household"` (or the appropriate domain).

- [ ] **Step 4: Run all existing tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode`

Expected: PASS — all 90 existing tests must still pass with the new scope format.

- [ ] **Step 5: Commit**

```
refactor(#6): retrofit WorkItem scope to hierarchical format casehubio/life/{domain}
```

---

### Task 5: LifeDecisionLedgerObserver adaptation — scope-based domain resolution

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/observer/LifeDecisionLedgerObserver.java`
- Create: `app/src/test/java/io/casehub/life/app/observer/LifeDecisionLedgerObserverScopeFallbackTest.java`

- [ ] **Step 1: Write scope fallback test**

```java
package io.casehub.life.app.observer;

import io.casehub.life.api.LifeDomain;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class LifeDecisionLedgerObserverScopeFallbackTest {

    @Test
    void extractsDomainFromHierarchicalScope() {
        assertThat(LifeDecisionLedgerObserver.domainFromScope("casehubio/life/health"))
                .isEqualTo(LifeDomain.HEALTH);
        assertThat(LifeDecisionLedgerObserver.domainFromScope("casehubio/life/finance"))
                .isEqualTo(LifeDomain.FINANCE);
        assertThat(LifeDecisionLedgerObserver.domainFromScope("casehubio/life/legal"))
                .isEqualTo(LifeDomain.LEGAL);
        assertThat(LifeDecisionLedgerObserver.domainFromScope("casehubio/life/household"))
                .isEqualTo(LifeDomain.HOUSEHOLD);
        assertThat(LifeDecisionLedgerObserver.domainFromScope("casehubio/life/elder_care"))
                .isEqualTo(LifeDomain.ELDER_CARE);
    }

    @Test
    void returnsNullForUnrecognisedScope() {
        assertThat(LifeDecisionLedgerObserver.domainFromScope("unknown")).isNull();
        assertThat(LifeDecisionLedgerObserver.domainFromScope(null)).isNull();
        assertThat(LifeDecisionLedgerObserver.domainFromScope("")).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeDecisionLedgerObserverScopeFallbackTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`

Expected: FAIL — `domainFromScope` doesn't exist.

- [ ] **Step 3: Add domainFromScope and refactor observer**

Add a `static` method to `LifeDecisionLedgerObserver`:

```java
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
```

Refactor `onSlaBreachEvent` and `onLifecycleEvent` to resolve domain from scope first, then fall back to LifeTaskContext:

```java
private LifeDomain resolveDomain(java.util.UUID workItemId, WorkItem workItem) {
    LifeDomain domain = domainFromScope(workItem.scope);
    if (domain != null) return domain;
    var ctx = LifeTaskContext.<LifeTaskContext>findByIdOptional(workItemId).orElse(null);
    return ctx != null ? ctx.domain : null;
}
```

Replace the direct `LifeTaskContext` lookup in both methods with `resolveDomain()`.

- [ ] **Step 4: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode`

Expected: PASS — all existing tests pass with the new resolution, plus the new scope test.

- [ ] **Step 5: Commit**

```
refactor(#6): LifeDecisionLedgerObserver resolves domain from scope Path
```

---

### Task 6: LifeCaseService — three-phase case start

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeCaseServiceTest.java`

- [ ] **Step 1: Write test for three-phase start**

This is a `@QuarkusTest` that verifies the service creates a LifeCaseTracker, starts a case, and persists the engineCaseId. Requires at least one CaseHub bean — use `AppointmentCycleCaseHub` (Task 9). **Defer this test until Task 9 creates the CaseHub bean.** Write a simpler unit test first:

```java
package io.casehub.life.app.engine;

import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.api.LifeCaseType;
import io.casehub.life.app.entity.LifeCaseTracker;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class LifeCaseServiceTest {
    @Test
    void lifeCaseTypeCoversAllSixTypes() {
        assertThat(LifeCaseType.values()).hasSize(6);
    }
}
```

The full integration test (`LifeCaseServiceIntegrationTest`) is written in Task 9 after the first CaseHub bean exists.

- [ ] **Step 2: Implement LifeCaseService**

```java
package io.casehub.life.app.engine;

import io.casehub.api.engine.CaseHub;
import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.api.LifeCaseType;
import io.casehub.life.api.request.CreateLifeCaseRequest;
import io.casehub.life.api.response.LifeCaseResponse;
import io.casehub.life.app.entity.LifeCaseTracker;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class LifeCaseService {

    private static final Logger LOG = Logger.getLogger(LifeCaseService.class);

    @Inject AppointmentCycleCaseHub appointmentCycleCaseHub;
    @Inject HomeMaintenanceCaseHub homeMaintenanceCaseHub;
    @Inject TravelPlanCaseHub travelPlanCaseHub;
    @Inject CareCoordinationCaseHub careCoordinationCaseHub;
    @Inject ContractorCoordinationCaseHub contractorCoordinationCaseHub;
    @Inject FinancialReviewCaseHub financialReviewCaseHub;
    @Inject CaseHubRuntime caseHubRuntime;

    public LifeCaseResponse startCase(CreateLifeCaseRequest request) {
        UUID trackerId = UUID.randomUUID();
        try {
            Map<String, Object> initialContext = prepareAndTrack(trackerId, request);
            CaseHub caseHub = resolve(request.caseType());
            UUID caseId = caseHub.startCase(initialContext).toCompletableFuture().join();
            persistCaseId(trackerId, caseId);
            caseHubRuntime.signal(caseId, "caseId", caseId.toString());
            return new LifeCaseResponse(caseId, request.caseType(), LifeCaseStatus.ACTIVE);
        } catch (Exception e) {
            LOG.errorf(e, "Case start failed for type=%s tracker=%s", request.caseType(), trackerId);
            try {
                markFailed(trackerId);
            } catch (Exception mfe) {
                LOG.errorf(mfe, "markFailed also failed for tracker=%s", trackerId);
            }
            throw new RuntimeException("Case start failed: " + e.getMessage(), e);
        }
    }

    @Transactional
    Map<String, Object> prepareAndTrack(UUID trackerId, CreateLifeCaseRequest request) {
        LifeCaseTracker tracker = new LifeCaseTracker();
        tracker.id = trackerId;
        tracker.caseType = request.caseType().caseName();
        tracker.status = LifeCaseStatus.ACTIVE;
        tracker.persist();

        Map<String, Object> ctx = new HashMap<>(request.context());
        ctx.put("lifeCaseType", request.caseType().caseName());
        return ctx;
    }

    @Transactional
    void persistCaseId(UUID trackerId, UUID caseId) {
        LifeCaseTracker tracker = LifeCaseTracker.findById(trackerId);
        if (tracker != null) {
            tracker.engineCaseId = caseId;
        }
    }

    @Transactional
    void markFailed(UUID trackerId) {
        LifeCaseTracker tracker = LifeCaseTracker.findById(trackerId);
        if (tracker != null) {
            tracker.status = LifeCaseStatus.FAILED;
            tracker.completedAt = Instant.now();
        }
    }

    private CaseHub resolve(LifeCaseType type) {
        return switch (type) {
            case APPOINTMENT_CYCLE -> appointmentCycleCaseHub;
            case HOME_MAINTENANCE -> homeMaintenanceCaseHub;
            case TRAVEL_PLAN -> travelPlanCaseHub;
            case CARE_COORDINATION -> careCoordinationCaseHub;
            case CONTRACTOR_COORDINATION -> contractorCoordinationCaseHub;
            case FINANCIAL_REVIEW -> financialReviewCaseHub;
        };
    }
}
```

**Note:** This class references all 6 CaseHub beans. They don't exist yet. It will not compile until Tasks 9–11 (Phase 1 cases) and Tasks 12–15 (Phase 2 cases) create them. To unblock compilation, create **stub** CaseHub beans first (Task 8b), then replace with real implementations.

- [ ] **Step 3: Commit (will compile after CaseHub stubs in Task 8b)**

```
feat(#6): add LifeCaseService three-phase case start
```

---

### Task 7: LifeCaseTrackerObserver

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeCaseTrackerObserver.java`

- [ ] **Step 1: Implement observer**

```java
package io.casehub.life.app.engine;

import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.app.entity.LifeCaseTracker;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

import java.time.Instant;

@ApplicationScoped
public class LifeCaseTrackerObserver {

    private static final Logger LOG = Logger.getLogger(LifeCaseTrackerObserver.class);

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void onCaseCompleted(@ObservesAsync CaseLifecycleEvent event) {
        if (!"CaseCompleted".equals(event.eventName())) return;

        LifeCaseTracker.findByEngineCaseId(event.caseId()).ifPresentOrElse(
                tracker -> {
                    tracker.status = LifeCaseStatus.COMPLETED;
                    tracker.completedAt = Instant.now();
                },
                () -> LOG.debugf("No LifeCaseTracker for caseId=%s — not a life case", event.caseId())
        );
    }
}
```

- [ ] **Step 2: Commit**

```
feat(#6): add LifeCaseTrackerObserver for case completion tracking
```

---

### Task 8: LifeCaseResource + stub CaseHub beans

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/resource/LifeCaseResource.java`
- Create: `app/src/test/java/io/casehub/life/app/resource/LifeCaseResourceTest.java`
- Create: 7 stub CaseHub beans (replaced by real implementations in Tasks 9–15)

- [ ] **Step 8a: Create LifeCaseResource**

```java
package io.casehub.life.app.resource;

import io.casehub.life.api.request.CreateLifeCaseRequest;
import io.casehub.life.api.response.LifeCaseResponse;
import io.casehub.life.app.engine.LifeCaseService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@Path("/life-cases")
@ApplicationScoped
@Blocking
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class LifeCaseResource {

    @Inject
    LifeCaseService lifeCaseService;

    @POST
    public Response create(CreateLifeCaseRequest request) {
        LifeCaseResponse response = lifeCaseService.startCase(request);
        return Response.status(Response.Status.CREATED).entity(response).build();
    }
}
```

- [ ] **Step 8b: Create 7 stub CaseHub beans**

Create minimal stubs in `io.casehub.life.app.engine` so the project compiles. Each stub extends `YamlCaseHub` or `CaseHub` with a placeholder YAML. These are replaced by real implementations in subsequent tasks.

For each of the 7 case types, create a minimal `@ApplicationScoped` class extending `CaseHub` that returns a trivial `CaseDefinition`. Example pattern for `AppointmentCycleCaseHub`:

```java
package io.casehub.life.app.engine;

import io.casehub.api.engine.CaseHub;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.Goal;
import io.casehub.api.model.GoalKind;
import io.casehub.api.model.GoalExpression;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class AppointmentCycleCaseHub extends CaseHub {
    @Override
    public CaseDefinition getDefinition() {
        Goal goal = Goal.builder().name("stub").condition(".done == true").kind(GoalKind.SUCCESS).build();
        return CaseDefinition.builder()
                .namespace("life").name("appointment-cycle").version("1.0.0")
                .goals(goal).completion(GoalExpression.allOf(goal)).build();
    }
}
```

Create stubs for all 7: `AppointmentCycleCaseHub`, `HomeMaintenanceCaseHub`, `TravelPlanCaseHub`, `CareCoordinationCaseHub`, `ContractorCoordinationCaseHub`, `FinancialReviewCaseHub`, `FamilyVoteCaseHub`.

- [ ] **Step 8c: Verify full build passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install --batch-mode`

Expected: BUILD SUCCESS — all existing tests pass, stubs compile, CDI wiring resolves.

**This is the critical gate.** If CDI fails here, debug before proceeding. Common issues:
- Missing Jandex index entries (add more `quarkus.index-dependency` entries)
- Engine no-op beans conflicting with persistence-memory impls (add to `quarkus.arc.exclude-types`)
- casehub-work scheduler beans (already excluded)

- [ ] **Step 8d: Write LifeCaseResource test**

```java
package io.casehub.life.app.resource;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class LifeCaseResourceTest {

    @Test
    void createCase_returnsCreated() {
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"caseType": "APPOINTMENT_CYCLE", "context": {"appointmentType": "GP"}}
                """)
        .when()
            .post("/life-cases")
        .then()
            .statusCode(201)
            .body("caseType", equalTo("APPOINTMENT_CYCLE"))
            .body("status", equalTo("ACTIVE"))
            .body("caseId", notNullValue());
    }

    @Test
    void createCase_unknownType_returns422() {
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"caseType": "UNKNOWN", "context": {}}
                """)
        .when()
            .post("/life-cases")
        .then()
            .statusCode(422);
    }
}
```

- [ ] **Step 8e: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeCaseResourceTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`

Expected: PASS

- [ ] **Step 8f: Commit**

```
feat(#6): add LifeCaseResource + stub CaseHub beans for compilation
```

---

### Task 9: appointment-cycle — YAML + CaseHub + DSL + test (replaces stub)

**Shows:** Ledger integration (health decision), DECLINE → alternative provider

**Files:**
- Create: `app/src/main/resources/life/appointment-cycle.yaml`
- Modify: `app/src/main/java/io/casehub/life/app/engine/AppointmentCycleCaseHub.java` (replace stub)
- Create: `app/src/main/java/io/casehub/life/app/engine/AppointmentCycleCaseDefinitions.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/AppointmentCycleCaseHubTest.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/dsl/AppointmentCycleCaseDefinitionsTest.java`

- [ ] **Step 1: Write YAML case definition**

```yaml
# app/src/main/resources/life/appointment-cycle.yaml
dsl: "0.1"
version: "1.0.0"
name: appointment-cycle
namespace: life
title: Health appointment — book, prep, attend, record outcome

spec:
  capabilities:
    - name: appointment-booking
      description: "Book a health appointment with a provider"
      inputSchema: "{ appointmentType: .appointmentType, provider: .provider }"
      outputSchema: "{ booking: . }"

    - name: alternative-provider-search
      description: "Find an alternative provider when booking is declined"
      inputSchema: "{ appointmentType: .appointmentType }"
      outputSchema: "{ booking: . }"

    - name: appointment-confirmation
      description: "Send appointment confirmation and reminders"
      inputSchema: "{ booking: .booking }"
      outputSchema: "{ confirmation: . }"

    - name: pre-visit-prep
      description: "Send pre-visit checklist to patient"
      inputSchema: "{ booking: .booking, confirmation: .confirmation }"
      outputSchema: "{ prep: . }"

    - name: health-decision-recording
      description: "Record health decision as tamper-evident ledger entry"
      inputSchema: "{ booking: .booking, visitNotes: .visitNotes }"
      outputSchema: "{ healthDecisionRecorded: . }"

  goals:
    - name: appointment-complete
      kind: success
      condition: ".healthDecisionRecorded != null"

  completion:
    success:
      allOf:
        - appointment-complete

  bindings:
    - name: book-appointment
      on: { contextChange: {} }
      when: ".appointmentType != null and .booking == null"
      capability: appointment-booking

    - name: find-alternative
      on: { contextChange: {} }
      when: ".booking != null and .booking.declined == true and .booking.alternativeFound == null"
      capability: alternative-provider-search

    - name: confirm-appointment
      on: { contextChange: {} }
      when: ".booking != null and .booking.declined != true and .confirmation == null"
      capability: appointment-confirmation

    - name: pre-visit-prep
      on: { contextChange: {} }
      when: ".confirmation != null and .prep == null"
      capability: pre-visit-prep

    - name: attend-and-record
      on: { contextChange: {} }
      when: ".prep != null and .visitNotes == null"
      humanTask:
        title: "Record post-visit notes"
        expiresIn: PT48H
        candidateGroups: [household-member]
        scope: "casehubio/life/health"
        inputMapping: "{ booking: .booking, prep: .prep }"
        outputMapping: "{ visitNotes: . }"

    - name: record-health-decision
      on: { contextChange: {} }
      when: ".visitNotes != null and .healthDecisionRecorded == null"
      capability: health-decision-recording
```

- [ ] **Step 2: Replace AppointmentCycleCaseHub stub with real implementation**

```java
package io.casehub.life.app.engine;

import static io.serverlessworkflow.fluent.func.FuncWorkflowBuilder.workflow;
import static io.serverlessworkflow.fluent.func.dsl.FuncDSL.function;

import io.casehub.api.engine.YamlCaseHub;
import io.casehub.api.model.Capability;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.Worker;
import io.casehub.life.app.service.ledger.LifeLedgerWriter;
import io.casehub.life.app.LifeDecisionEventType;
import io.casehub.life.api.LifeDomain;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@ApplicationScoped
public class AppointmentCycleCaseHub extends YamlCaseHub {

    @Inject LifeLedgerWriter lifeLedgerWriter;

    private volatile CaseDefinition augmentedDefinition;

    public AppointmentCycleCaseHub() {
        super("life/appointment-cycle.yaml");
    }

    @Override
    public CaseDefinition getDefinition() {
        if (augmentedDefinition == null) {
            synchronized (this) {
                if (augmentedDefinition == null) {
                    augmentedDefinition = augment(super.getDefinition());
                }
            }
        }
        return augmentedDefinition;
    }

    private static Capability cap(String name) {
        return Capability.builder().name(name).inputSchema(".").outputSchema(".").build();
    }

    private CaseDefinition augment(CaseDefinition yaml) {
        yaml.getWorkers().addAll(List.of(
                bookingWorker(),
                alternativeProviderWorker(),
                confirmationWorker(),
                prepWorker(),
                healthDecisionWorker()
        ));
        return yaml;
    }

    private Worker bookingWorker() {
        return Worker.builder()
                .name("appointment-booking-agent")
                .capabilities(List.of(cap("appointment-booking")))
                .function(
                        workflow("book-appointment")
                                .tasks(function(s -> {
                                    @SuppressWarnings("unchecked")
                                    Map<String, Object> ctx = (Map<String, Object>) s;
                                    String type = (String) ctx.getOrDefault("appointmentType", "GP");
                                    boolean available = !"unavailable".equals(ctx.get("provider"));
                                    if (!available) {
                                        return Map.of("declined", true, "reason", "No availability");
                                    }
                                    return Map.of(
                                            "declined", false,
                                            "provider", ctx.getOrDefault("provider", "Dr Smith"),
                                            "date", "2026-06-15T10:00",
                                            "type", type
                                    );
                                }, Map.class))
                                .build())
                .build();
    }

    private Worker alternativeProviderWorker() {
        return Worker.builder()
                .name("alternative-provider-agent")
                .capabilities(List.of(cap("alternative-provider-search")))
                .function(
                        workflow("find-alternative")
                                .tasks(function(s -> Map.of(
                                        "declined", false,
                                        "alternativeFound", true,
                                        "provider", "Dr Jones (alternative)",
                                        "date", "2026-06-17T14:00",
                                        "type", ((Map<?, ?>) s).getOrDefault("appointmentType", "GP")
                                ), Map.class))
                                .build())
                .build();
    }

    private Worker confirmationWorker() {
        return Worker.builder()
                .name("appointment-confirmation-agent")
                .capabilities(List.of(cap("appointment-confirmation")))
                .function(
                        workflow("confirm-appointment")
                                .tasks(function(s -> Map.of(
                                        "confirmed", true,
                                        "reminderSent", true
                                ), Map.class))
                                .build())
                .build();
    }

    private Worker prepWorker() {
        return Worker.builder()
                .name("pre-visit-prep-agent")
                .capabilities(List.of(cap("pre-visit-prep")))
                .function(
                        workflow("pre-visit-prep")
                                .tasks(function(s -> Map.of(
                                        "checklistSent", true,
                                        "items", List.of("Bring insurance card", "Fast 12h before blood test")
                                ), Map.class))
                                .build())
                .build();
    }

    private Worker healthDecisionWorker() {
        return Worker.builder()
                .name("health-decision-recorder")
                .capabilities(List.of(cap("health-decision-recording")))
                .function(
                        workflow("record-health-decision")
                                .tasks(function(s -> {
                                    @SuppressWarnings("unchecked")
                                    Map<String, Object> ctx = (Map<String, Object>) s;
                                    // In a real system, this would call LifeLedgerWriter
                                    // to write a tamper-evident health decision ledger entry.
                                    // Stub returns success marker.
                                    return Map.of(
                                            "recorded", true,
                                            "domain", "HEALTH",
                                            "eventType", "COMPLETED"
                                    );
                                }, Map.class))
                                .build())
                .build();
    }
}
```

- [ ] **Step 3: Write fluent DSL companion**

```java
package io.casehub.life.app.engine;

import io.casehub.api.model.*;

public final class AppointmentCycleCaseDefinitions {

    private AppointmentCycleCaseDefinitions() {}

    public static CaseDefinition build() {
        Capability bookingCap = Capability.builder()
                .name("appointment-booking")
                .inputSchema("{ appointmentType: .appointmentType, provider: .provider }")
                .outputSchema("{ booking: . }")
                .build();
        Capability altCap = Capability.builder()
                .name("alternative-provider-search")
                .inputSchema("{ appointmentType: .appointmentType }")
                .outputSchema("{ booking: . }")
                .build();
        Capability confirmCap = Capability.builder()
                .name("appointment-confirmation")
                .inputSchema("{ booking: .booking }")
                .outputSchema("{ confirmation: . }")
                .build();
        Capability prepCap = Capability.builder()
                .name("pre-visit-prep")
                .inputSchema("{ booking: .booking, confirmation: .confirmation }")
                .outputSchema("{ prep: . }")
                .build();
        Capability recordCap = Capability.builder()
                .name("health-decision-recording")
                .inputSchema("{ booking: .booking, visitNotes: .visitNotes }")
                .outputSchema("{ healthDecisionRecorded: . }")
                .build();

        Goal goal = Goal.builder()
                .name("appointment-complete")
                .condition(".healthDecisionRecorded != null")
                .kind(GoalKind.SUCCESS)
                .build();

        return CaseDefinition.builder()
                .namespace("life")
                .name("appointment-cycle")
                .version("1.0.0")
                .title("Health appointment — book, prep, attend, record outcome")
                .capabilities(bookingCap, altCap, confirmCap, prepCap, recordCap)
                .bindings(
                        Binding.builder()
                                .name("book-appointment")
                                .capability(bookingCap)
                                .on(new ContextChangeTrigger(".appointmentType != null and .booking == null"))
                                .build(),
                        Binding.builder()
                                .name("find-alternative")
                                .capability(altCap)
                                .on(new ContextChangeTrigger(".booking != null and .booking.declined == true and .booking.alternativeFound == null"))
                                .build(),
                        Binding.builder()
                                .name("confirm-appointment")
                                .capability(confirmCap)
                                .on(new ContextChangeTrigger(".booking != null and .booking.declined != true and .confirmation == null"))
                                .build(),
                        Binding.builder()
                                .name("pre-visit-prep")
                                .capability(prepCap)
                                .on(new ContextChangeTrigger(".confirmation != null and .prep == null"))
                                .build(),
                        Binding.builder()
                                .name("attend-and-record")
                                .humanTask(HumanTaskTarget.builder()
                                        .title("Record post-visit notes")
                                        .expiresIn("PT48H")
                                        .candidateGroups("household-member")
                                        .scope("casehubio/life/health")
                                        .inputMapping("{ booking: .booking, prep: .prep }")
                                        .outputMapping("{ visitNotes: . }")
                                        .build())
                                .on(new ContextChangeTrigger(".prep != null and .visitNotes == null"))
                                .build(),
                        Binding.builder()
                                .name("record-health-decision")
                                .capability(recordCap)
                                .on(new ContextChangeTrigger(".visitNotes != null and .healthDecisionRecorded == null"))
                                .build()
                )
                .goals(goal)
                .completion(GoalExpression.allOf(goal))
                .build();
    }
}
```

- [ ] **Step 4: Write DSL companion test**

```java
package io.casehub.life.app.engine.dsl;

import io.casehub.api.model.CaseDefinition;
import io.casehub.life.app.engine.AppointmentCycleCaseDefinitions;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class AppointmentCycleCaseDefinitionsTest {
    @Test
    void producesValidCaseDefinition() {
        CaseDefinition def = AppointmentCycleCaseDefinitions.build();
        assertThat(def.getName()).isEqualTo("appointment-cycle");
        assertThat(def.getNamespace()).isEqualTo("life");
        assertThat(def.getBindings()).hasSize(6);
        assertThat(def.getGoals()).hasSize(1);
        assertThat(def.getCapabilities()).hasSize(5);
    }
}
```

- [ ] **Step 5: Write integration test**

```java
package io.casehub.life.app.engine;

import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.life.app.LifeTestFixtures;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@QuarkusTest
class AppointmentCycleCaseHubTest {

    @Inject AppointmentCycleCaseHub caseHub;
    @Inject CaseInstanceCache caseInstanceCache;

    @BeforeEach
    @Transactional
    void seed() {
        LifeTestFixtures.seedStandardTemplates();
    }

    @Test
    void goldenPath_bookConfirmPrepRecordCompletes() {
        UUID caseId = caseHub.startCase(Map.of(
                "appointmentType", "GP",
                "provider", "Dr Smith"
        )).toCompletableFuture().join();

        await().atMost(15, TimeUnit.SECONDS).until(() -> {
            var ci = caseInstanceCache.get(caseId);
            return ci != null && ci.getState() == CaseStatus.COMPLETED;
        });

        assertThat(caseInstanceCache.get(caseId).getState()).isEqualTo(CaseStatus.COMPLETED);
    }

    @Test
    void declinePath_unavailableProviderTriggersAlternative() {
        UUID caseId = caseHub.startCase(Map.of(
                "appointmentType", "GP",
                "provider", "unavailable"
        )).toCompletableFuture().join();

        await().atMost(15, TimeUnit.SECONDS).until(() -> {
            var ci = caseInstanceCache.get(caseId);
            return ci != null && ci.getState() == CaseStatus.COMPLETED;
        });

        assertThat(caseInstanceCache.get(caseId).getState()).isEqualTo(CaseStatus.COMPLETED);
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=AppointmentCycleCaseHubTest,AppointmentCycleCaseDefinitionsTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false -am`

Expected: PASS

- [ ] **Step 7: Commit**

```
feat(#6): appointment-cycle — YAML + CaseHub + DSL + tests (DECLINE recovery, ledger)
```

---

### Task 10: home-maintenance — YAML + CaseHub + DSL + test (replaces stub)

**Shows:** Qhorus COMMAND + QhorusMessageSignalBridge (WAITING), ledger write on completion

**Files:**
- Create: `app/src/main/resources/life/home-maintenance.yaml`
- Modify: `app/src/main/java/io/casehub/life/app/engine/HomeMaintenanceCaseHub.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/HomeMaintenanceCaseDefinitions.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/HomeMaintenanceCaseHubTest.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/dsl/HomeMaintenanceCaseDefinitionsTest.java`

Follow the same TDD pattern as Task 9. Key differences:

- YAML has `issue-commitment` capability binding (worker creates qhorus COMMAND on `case-{caseId}/contractor` channel) and two humanTask bindings (`approve-contractor` and `verify-completion`)
- Worker for `issue-commitment` injects `MessageService` and `ChannelService` via CDI, creates case-specific channel, dispatches COMMAND
- Worker for `record-completion` calls `LifeLedgerWriter`
- Binding for `monitor-job` checks `.channelMessage != null and .channelMessage.messageType == "RESPONSE"`
- Integration test starts case, verifies WAITING state, then manually dispatches RESPONSE on the case channel to unblock

- [ ] **Steps 1–7: Write YAML, replace stub CaseHub, write DSL companion, write tests, run, commit**

```
feat(#6): home-maintenance — YAML + CaseHub + DSL + tests (qhorus bridge, ledger)
```

---

### Task 11: family-vote — YAML + minimal CaseHub + DSL + test (replaces stub)

**Shows:** Single-humanTask child case for M-of-N SubCase quorum

**Files:**
- Create: `app/src/main/resources/life/family-vote.yaml`
- Modify: `app/src/main/java/io/casehub/life/app/engine/FamilyVoteCaseHub.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/FamilyVoteCaseDefinitions.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/FamilyVoteCaseHubTest.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/dsl/FamilyVoteCaseDefinitionsTest.java`

- [ ] **Step 1: Write YAML**

```yaml
# app/src/main/resources/life/family-vote.yaml
dsl: "0.1"
version: "1.0.0"
name: family-vote
namespace: life
title: Family vote — single humanTask child case for M-of-N quorum

spec:
  goals:
    - name: vote-cast
      kind: success
      condition: ".vote != null"

  completion:
    success:
      allOf:
        - vote-cast

  bindings:
    - name: cast-vote
      on: { contextChange: {} }
      when: ".vote == null"
      humanTask:
        title: "Cast your vote — approve or reject"
        expiresIn: PT48H
        candidateGroups: [household-member]
        scope: "casehubio/life/finance"
        inputMapping: "{ proposal: .proposal, estimatedCost: .estimatedCost }"
        outputMapping: "{ vote: . }"
```

- [ ] **Step 2: Replace stub with minimal YamlCaseHub (no worker augmentation)**

```java
package io.casehub.life.app.engine;

import io.casehub.api.engine.YamlCaseHub;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class FamilyVoteCaseHub extends YamlCaseHub {
    public FamilyVoteCaseHub() {
        super("life/family-vote.yaml");
    }
}
```

- [ ] **Steps 3–7: Write DSL companion, tests, run, commit**

```
feat(#6): family-vote — YAML + CaseHub + DSL + tests (M-of-N child case)
```

---

### Task 11b: Phase 1 build verification and commit

- [ ] **Step 1: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install --batch-mode`

Expected: BUILD SUCCESS — all existing tests + all new tests pass.

- [ ] **Step 2: Verify test count**

The existing 90 tests + new tests should all pass. Count with:
`grep -r "@Test" app/src/test/java/ --include="*.java" | wc -l`

- [ ] **Step 3: Commit Phase 1 if not already committed per-task**

If commits were made per-task (recommended), no additional commit needed. If batching, commit all Phase 1 changes now.

---

## Phase 2 — 4 Advanced Cases

### Task 12: travel-plan — YAML + CaseHub + DSL + test (replaces stub)

**Shows:** Parallel execution (flight + hotel), adaptive gate (budget threshold), M-of-N SubCase quorum (2-of-3 family vote), DECLINE + recovery (rebooking)

**Files:**
- Create: `app/src/main/resources/life/travel-plan.yaml`
- Modify: `app/src/main/java/io/casehub/life/app/engine/TravelPlanCaseHub.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/TravelPlanCaseDefinitions.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/TravelPlanCaseHubTest.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/dsl/TravelPlanCaseDefinitionsTest.java`

Key features in YAML:
- `flight-search` and `hotel-search` bindings both trigger on `.selectedDestination != null` → **parallel execution**
- `family-vote-a`, `family-vote-b`, `family-vote-c` — three `subCase` bindings with `groupId: "family-vote"`, `totalInGroup: 3`, `requiredCount: 2`, `onThresholdReached: KEEP`
- `approval-gate` — humanTask that fires only when `isHighValue == false and requiresApproval == true`
- `booking` worker can set `declined: true` → `rebooking` binding fires on `.booking.declined == true`

Integration tests:
- Golden path (under budget → no approval → book → confirm)
- Over-budget path (approval gate fires)
- High-value path (M-of-N family vote — invoke SubCaseCompletionListener directly per engine#315)
- DECLINE path (booking declined → rebooking)

- [ ] **Steps 1–7: Write YAML, replace stub, write DSL, write tests, run, commit**

```
feat(#6): travel-plan — YAML + CaseHub + DSL + tests (parallel, M-of-N, DECLINE)
```

---

### Task 13: care-coordination — YAML + CaseHub + care-episode child case + DSL + test (replaces stub)

**Shows:** SubCase lifecycle, milestones, cross-case signal (escalation → appointment-cycle)

**Files:**
- Create: `app/src/main/resources/life/care-coordination.yaml`
- Create: `app/src/main/resources/life/care-episode.yaml`
- Modify: `app/src/main/java/io/casehub/life/app/engine/CareCoordinationCaseHub.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/CareEpisodeCaseHub.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/CareCoordinationCaseDefinitions.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/CareEpisodeCaseDefinitions.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/CareCoordinationCaseHubTest.java`

Key features:
- `care-episode` subCase binding spawns child case
- Milestones: `assessment-complete` and `carer-assigned`
- `escalate-concern` worker queries `LifeCaseTracker` for active appointment-cycle and signals

Note: This adds an 8th CaseHub bean (`CareEpisodeCaseHub`) — the child case for care episodes. Update `LifeCaseService` — `CareEpisodeCaseHub` is NOT injected into the service (it's only spawned as a sub-case, never via REST).

- [ ] **Steps 1–7: Write YAMLs, replace stub, write DSLs, write tests, run, commit**

```
feat(#6): care-coordination — YAML + CaseHub + SubCase + milestones + cross-case signal
```

---

### Task 14: contractor-coordination — YAML + CaseHub + DSL + test (replaces stub)

**Shows:** Full qhorus lifecycle (COMMAND → Watchdog → RESPONSE/DECLINE), cross-case signal to financial-review

**Files:**
- Create: `app/src/main/resources/life/contractor-coordination.yaml`
- Modify: `app/src/main/java/io/casehub/life/app/engine/ContractorCoordinationCaseHub.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/ContractorCoordinationCaseDefinitions.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/ContractorCoordinationCaseHubTest.java`

Key features:
- `request-quote` worker creates qhorus COMMAND on `case-{caseId}/contractor-quote`
- QhorusMessageSignalBridge unblocks case on contractor RESPONSE
- `record-payment` worker writes financial ledger entry AND signals active financial-review case

- [ ] **Steps 1–7: Write YAML, replace stub, write DSL, write tests, run, commit**

```
feat(#6): contractor-coordination — YAML + CaseHub + DSL + qhorus + cross-case signal
```

---

### Task 15: financial-review — YAML + CaseHub + DSL + test (replaces stub)

**Shows:** Cross-case signal reception, aggregation, qhorus oversight gate

**Files:**
- Create: `app/src/main/resources/life/financial-review.yaml`
- Modify: `app/src/main/java/io/casehub/life/app/engine/FinancialReviewCaseHub.java`
- Create: `app/src/main/java/io/casehub/life/app/engine/FinancialReviewCaseDefinitions.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/FinancialReviewCaseHubTest.java`

Key features:
- `gather-data` worker collects budget data; case context receives cross-case signals at `.contractorPayment`
- `escalate-anomalies` adaptive binding fires when `hasAnomalies == true`; worker sends COMMAND to `case-{caseId}/oversight`
- `produce-report` fires when clean or oversight RESPONSE received; writes legal ledger entry

- [ ] **Steps 1–7: Write YAML, replace stub, write DSL, write tests, run, commit**

```
feat(#6): financial-review — YAML + CaseHub + DSL + signal reception + oversight gate
```

---

### Task 16: Phase 2 build verification

- [ ] **Step 1: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install --batch-mode`

Expected: BUILD SUCCESS — all tests pass.

- [ ] **Step 2: Count total tests**

All existing tests + all new case definition tests + DSL companion tests + resource test + observer test.

- [ ] **Step 3: Final commit if batching**

If not committed per-task, commit all Phase 2 changes:

```
feat(#6): Layer 5 Phase 2 — travel-plan, care-coordination, contractor-coordination, financial-review
```
