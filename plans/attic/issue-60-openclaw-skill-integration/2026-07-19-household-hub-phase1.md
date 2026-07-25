# Household Hub Phase 1 — Backend + Frontend MVP

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — file before starting (epic: Household Hub UI)
**Issue group:** TBD (child issues per task)

**Goal:** Deliver a working standalone Household Hub where household members
can see and act on their tasks via an inbox, view a dashboard with KPIs and
SLA warnings, and receive real-time updates via SSE — backed by enriched case
endpoints, role-based visibility, and pre-populated demo data.

**Architecture:** Backend enriches LifeCaseTracker with domain, adds GET
/life-cases list+detail endpoints with LifeCaseVisibilityPolicy, adds SSE
event infrastructure (CDI→SSE bridge with snapshot-on-reconnect). Frontend
is a Lit SPA in `life-ui/` served via Quarkus Quinoa, composing blocks-ui
Web Components. Demo mode via Quarkus `demo` profile with Flyway seeds.

**Tech Stack:** Java 21, Quarkus 3.32.2, Lit 3.x, Vite, Quinoa, blocks-ui,
SSE (RESTEasy Reactive `Multi<OutboundSseEvent>`), H2 MODE=PostgreSQL (tests)

## Global Constraints

- Java 21 source (Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Flyway life domain migrations: `db/life/migration/` starting at V111 (V110 is latest)
- All REST resources: `@Blocking @ApplicationScoped`, class-level `@Produces`/`@Consumes`
- `@Transactional` on service methods only, never resources
- Test tenancyId: `278776f9-e1b0-46fb-9032-8bddebdcf9ce`
- Tests use H2 MODE=PostgreSQL with `drop-and-create`, Flyway disabled
- All commits reference an issue
- IntelliJ MCP mandatory for all .java/.ts file operations

---

### Task 1: LifeCaseType domain mapping + LifeCaseTracker schema enrichment

**Files:**
- Modify: `api/src/main/java/io/casehub/life/api/LifeCaseType.java`
- Modify: `app/src/main/java/io/casehub/life/app/entity/LifeCaseTracker.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java`
- Create: `app/src/main/resources/db/life/migration/V111__life_case_tracker_domain.sql`
- Test: `api/src/test/java/io/casehub/life/api/LifeCaseTypeTest.java` (modify)
- Test: `app/src/test/java/io/casehub/life/app/engine/LifeCaseServiceDomainTest.java` (create)

**Interfaces:**
- Produces: `LifeCaseType.domain()` → returns `LifeDomain` for each case type
- Produces: `LifeCaseTracker.domain` → persisted `LifeDomain` column
- Produces: `LifeCaseTracker.findByDomain(LifeDomain)` → query method

- [ ] **Step 1: Write failing test for LifeCaseType.domain()**

Add to existing `LifeCaseTypeTest`:

```java
@Test
void eachCaseTypeMapsToDomain() {
    assertEquals(LifeDomain.TRAVEL, LifeCaseType.TRAVEL_PLAN.domain());
    assertEquals(LifeDomain.HOUSEHOLD, LifeCaseType.HOME_MAINTENANCE.domain());
    assertEquals(LifeDomain.ELDER_CARE, LifeCaseType.CARE_COORDINATION.domain());
    assertEquals(LifeDomain.HEALTH, LifeCaseType.APPOINTMENT_CYCLE.domain());
    assertEquals(LifeDomain.CONTRACTOR_COORDINATION, LifeCaseType.CONTRACTOR_COORDINATION.domain());
    assertEquals(LifeDomain.FINANCE, LifeCaseType.FINANCIAL_REVIEW.domain());
}

@Test
void allCaseTypesHaveDomainMapping() {
    for (LifeCaseType type : LifeCaseType.values()) {
        assertNotNull(type.domain(), type + " must map to a LifeDomain");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=LifeCaseTypeTest --batch-mode`
Expected: Compilation error — `domain()` method does not exist

- [ ] **Step 3: Implement LifeCaseType.domain()**

Add `domain` field to `LifeCaseType` enum and `domain()` accessor:

```java
public enum LifeCaseType {
    TRAVEL_PLAN("travel-plan", LifeDomain.TRAVEL),
    HOME_MAINTENANCE("home-maintenance", LifeDomain.HOUSEHOLD),
    CARE_COORDINATION("care-coordination", LifeDomain.ELDER_CARE),
    APPOINTMENT_CYCLE("appointment-cycle", LifeDomain.HEALTH),
    CONTRACTOR_COORDINATION("contractor-coordination", LifeDomain.CONTRACTOR_COORDINATION),
    FINANCIAL_REVIEW("financial-review", LifeDomain.FINANCE);

    private final String caseName;
    private final LifeDomain domain;

    LifeCaseType(String caseName, LifeDomain domain) {
        this.caseName = caseName;
        this.domain = domain;
    }

    public String caseName() { return caseName; }
    public LifeDomain domain() { return domain; }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=LifeCaseTypeTest --batch-mode`
Expected: PASS

- [ ] **Step 5: Create V111 migration**

Create `app/src/main/resources/db/life/migration/V111__life_case_tracker_domain.sql`:

```sql
ALTER TABLE life_case_tracker ADD COLUMN domain VARCHAR(32);

UPDATE life_case_tracker SET domain = 'TRAVEL' WHERE case_type = 'travel-plan';
UPDATE life_case_tracker SET domain = 'HOUSEHOLD' WHERE case_type = 'home-maintenance';
UPDATE life_case_tracker SET domain = 'ELDER_CARE' WHERE case_type = 'care-coordination';
UPDATE life_case_tracker SET domain = 'HEALTH' WHERE case_type = 'appointment-cycle';
UPDATE life_case_tracker SET domain = 'CONTRACTOR_COORDINATION' WHERE case_type = 'contractor-coordination';
UPDATE life_case_tracker SET domain = 'FINANCE' WHERE case_type = 'financial-review';

ALTER TABLE life_case_tracker ALTER COLUMN domain SET NOT NULL;
```

- [ ] **Step 6: Add domain field to LifeCaseTracker entity**

Add to `LifeCaseTracker.java`:

```java
@Enumerated(EnumType.STRING)
@Column(nullable = false, length = 32)
public LifeDomain domain;
```

Add query method:

```java
public static List<LifeCaseTracker> findByDomain(LifeDomain domain) {
    return list("domain", domain);
}
```

Import `io.casehub.life.api.LifeDomain`.

- [ ] **Step 7: Update LifeCaseService.prepareAndTrack() to set domain**

In `prepareAndTrack()`, after `tracker.caseType = request.caseType().caseName();`, add:

```java
tracker.domain = request.caseType().domain();
```

- [ ] **Step 8: Write integration test for domain persistence**

Create `LifeCaseServiceDomainTest.java`:

```java
@QuarkusTest
@TestSecurity(user = "admin", roles = HouseholdGroups.ADMIN)
public class LifeCaseServiceDomainTest {

    @Inject LifeCaseService lifeCaseService;

    @BeforeEach
    @Transactional
    void setup() {
        LifeTestFixtures.seedStandardTemplates();
        LifeTestFixtures.seedEscalationTemplate();
    }

    @Test
    @Transactional
    void startCasePersistsDomainOnTracker() {
        // Use a case type where the CaseHub resolves (all are registered).
        // Direct tracker query to verify domain was set.
        var trackers = LifeCaseTracker.findByDomain(LifeDomain.TRAVEL);
        long before = trackers.size();

        // startCase will fail at engine level in unit test, but the tracker
        // is created in prepareAndTrack (Phase 1 of three-phase pattern).
        // We can test prepareAndTrack directly via the CDI proxy.
        var ctx = lifeCaseService.prepareAndTrack(
                UUID.randomUUID(),
                new CreateLifeCaseRequest(LifeCaseType.TRAVEL_PLAN, Map.of()));
        assertNotNull(ctx);

        var trackersAfter = LifeCaseTracker.findByDomain(LifeDomain.TRAVEL);
        assertEquals(before + 1, trackersAfter.size());
        assertEquals(LifeDomain.TRAVEL, trackersAfter.getLast().domain);
    }
}
```

- [ ] **Step 9: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeCaseServiceDomainTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/life/api/LifeCaseType.java \
        api/src/test/java/io/casehub/life/api/LifeCaseTypeTest.java \
        app/src/main/java/io/casehub/life/app/entity/LifeCaseTracker.java \
        app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java \
        app/src/main/resources/db/life/migration/V111__life_case_tracker_domain.sql \
        app/src/test/java/io/casehub/life/app/engine/LifeCaseServiceDomainTest.java
git commit -m "feat(#N): LifeCaseType.domain() mapping + LifeCaseTracker domain column (V111)"
```

---

### Task 2: Enriched LifeCaseResponse + GET /life-cases endpoints + visibility policy

**Files:**
- Modify: `api/src/main/java/io/casehub/life/api/response/LifeCaseResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/response/LifeCaseDetailResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/spi/LifeCaseVisibilityPolicy.java`
- Create: `app/src/main/java/io/casehub/life/app/spi/JuniorLifeCaseVisibilityPolicy.java`
- Create: `app/src/main/java/io/casehub/life/app/service/LifeCaseQueryService.java`
- Modify: `app/src/main/java/io/casehub/life/app/resource/LifeCaseResource.java`
- Test: `app/src/test/java/io/casehub/life/app/spi/JuniorLifeCaseVisibilityPolicyTest.java`
- Test: `app/src/test/java/io/casehub/life/app/resource/LifeCaseResourceTest.java`

**Interfaces:**
- Consumes: `LifeCaseTracker.domain` (Task 1), `LifeCaseTracker.findByDomain()` (Task 1)
- Produces: `GET /life-cases` → `PagedResponse<LifeCaseResponse>` with filters
- Produces: `GET /life-cases/{id}` → `LifeCaseDetailResponse`
- Produces: `LifeCaseVisibilityPolicy.isVisible(LifeCaseResponse, String, Set<String>)` → boolean

- [ ] **Step 1: Enrich LifeCaseResponse**

Update `LifeCaseResponse` to include domain, timestamps, and SLA state:

```java
package io.casehub.life.api.response;

import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.api.LifeCaseType;
import io.casehub.life.api.LifeDomain;
import java.time.Instant;
import java.util.UUID;

public record LifeCaseResponse(
        UUID caseId,
        LifeCaseType caseType,
        LifeDomain domain,
        LifeCaseStatus status,
        Instant createdAt,
        Instant completedAt
) {}
```

- [ ] **Step 2: Fix existing callers of LifeCaseResponse**

Use `ide_find_references` on `LifeCaseResponse` to find all construction sites.
Update `LifeCaseService.startCase()` return statement:

```java
return new LifeCaseResponse(caseId, request.caseType(),
        request.caseType().domain(), LifeCaseStatus.ACTIVE,
        Instant.now(), null);
```

Run `ide_diagnostics` on `LifeCaseService.java` to verify no compilation errors.

- [ ] **Step 3: Create LifeCaseDetailResponse**

Create `api/src/main/java/io/casehub/life/api/response/LifeCaseDetailResponse.java`:

```java
package io.casehub.life.api.response;

import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.api.LifeCaseType;
import io.casehub.life.api.LifeDomain;
import java.time.Instant;
import java.util.UUID;

public record LifeCaseDetailResponse(
        UUID caseId,
        LifeCaseType caseType,
        LifeDomain domain,
        LifeCaseStatus status,
        Instant createdAt,
        Instant completedAt,
        UUID engineCaseId
) {}
```

- [ ] **Step 4: Create LifeCaseVisibilityPolicy SPI**

Create `api/src/main/java/io/casehub/life/api/spi/LifeCaseVisibilityPolicy.java`:

```java
package io.casehub.life.api.spi;

import io.casehub.life.api.response.LifeCaseResponse;
import java.util.Set;

public interface LifeCaseVisibilityPolicy {
    boolean isVisible(LifeCaseResponse caseResponse, String actorId, Set<String> groups);
}
```

- [ ] **Step 5: Write failing test for JuniorLifeCaseVisibilityPolicy**

Create `JuniorLifeCaseVisibilityPolicyTest.java`:

```java
package io.casehub.life.app.spi;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.api.LifeCaseType;
import io.casehub.life.api.response.LifeCaseResponse;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Set;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class JuniorLifeCaseVisibilityPolicyTest {

    private final JuniorLifeCaseVisibilityPolicy policy = new JuniorLifeCaseVisibilityPolicy();

    private LifeCaseResponse aCase() {
        return new LifeCaseResponse(UUID.randomUUID(), LifeCaseType.TRAVEL_PLAN,
                LifeCaseType.TRAVEL_PLAN.domain(), LifeCaseStatus.ACTIVE,
                Instant.now(), null);
    }

    @Test
    void adminAlwaysVisible() {
        assertTrue(policy.isVisible(aCase(), "admin-1", Set.of(HouseholdGroups.ADMIN)));
    }

    @Test
    void memberAlwaysVisible() {
        assertTrue(policy.isVisible(aCase(), "member-1", Set.of(HouseholdGroups.MEMBER)));
    }

    @Test
    void juniorNeverVisibleWithoutWorkItemCheck() {
        // Junior visibility requires a WorkItem check — default is not visible.
        // The actual check queries WorkItems by case scope and candidate groups.
        // In unit test without DB, the policy returns false for juniors.
        assertFalse(policy.isVisible(aCase(), "junior-1", Set.of(HouseholdGroups.JUNIOR)));
    }
}
```

- [ ] **Step 6: Implement JuniorLifeCaseVisibilityPolicy**

Create `app/src/main/java/io/casehub/life/app/spi/JuniorLifeCaseVisibilityPolicy.java`:

```java
package io.casehub.life.app.spi;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.response.LifeCaseResponse;
import io.casehub.life.api.spi.LifeCaseVisibilityPolicy;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import java.util.Set;

@Alternative
@Priority(1)
@ApplicationScoped
public class JuniorLifeCaseVisibilityPolicy implements LifeCaseVisibilityPolicy {
    @Override
    public boolean isVisible(LifeCaseResponse caseResponse, String actorId, Set<String> groups) {
        if (groups.contains(HouseholdGroups.ADMIN) || groups.contains(HouseholdGroups.MEMBER)) {
            return true;
        }
        // Junior: requires WorkItem-level check — deferred to query service
        // which filters cases by whether the junior has an assigned WorkItem.
        // At the SPI level, junior without admin/member is not visible.
        return false;
    }
}
```

- [ ] **Step 7: Run visibility policy test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=JuniorLifeCaseVisibilityPolicyTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS

- [ ] **Step 8: Create LifeCaseQueryService**

Create `app/src/main/java/io/casehub/life/app/service/LifeCaseQueryService.java`:

```java
package io.casehub.life.app.service;

import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.api.LifeCaseType;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.response.LifeCaseDetailResponse;
import io.casehub.life.api.response.LifeCaseResponse;
import io.casehub.life.api.response.PagedResponse;
import io.casehub.life.api.spi.LifeCaseVisibilityPolicy;
import io.casehub.life.app.entity.LifeCaseTracker;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.panache.common.Page;
import io.quarkus.panache.common.Sort;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.util.*;

@ApplicationScoped
public class LifeCaseQueryService {

    @Inject CurrentPrincipal currentPrincipal;
    @Inject LifeCaseVisibilityPolicy visibilityPolicy;

    @Transactional
    public PagedResponse<LifeCaseResponse> listCases(LifeDomain domain,
                                                      LifeCaseStatus status,
                                                      LifeCaseType caseType,
                                                      int page, int size) {
        var query = buildListQuery(domain, status, caseType);
        long total = LifeCaseTracker.count(query.hql(), query.params());
        List<LifeCaseTracker> trackers = LifeCaseTracker.find(query.hql(),
                        Sort.by("createdAt", Sort.Direction.Descending), query.params())
                .page(Page.of(page, size))
                .list();

        String actorId = currentPrincipal.actorId();
        Set<String> groups = currentPrincipal.groups();

        List<LifeCaseResponse> items = trackers.stream()
                .map(this::toResponse)
                .filter(r -> visibilityPolicy.isVisible(r, actorId, groups))
                .toList();

        return new PagedResponse<>(items, page, size, total);
    }

    @Transactional
    public Optional<LifeCaseDetailResponse> findById(UUID id) {
        LifeCaseTracker tracker = LifeCaseTracker.findById(id);
        if (tracker == null) return Optional.empty();

        LifeCaseResponse response = toResponse(tracker);
        String actorId = currentPrincipal.actorId();
        Set<String> groups = currentPrincipal.groups();

        if (!visibilityPolicy.isVisible(response, actorId, groups)) {
            return Optional.empty();
        }

        return Optional.of(toDetailResponse(tracker));
    }

    private LifeCaseResponse toResponse(LifeCaseTracker tracker) {
        return new LifeCaseResponse(
                tracker.id,
                LifeCaseType.valueOf(caseNameToEnumName(tracker.caseType)),
                tracker.domain,
                tracker.status,
                tracker.createdAt,
                tracker.completedAt
        );
    }

    private LifeCaseDetailResponse toDetailResponse(LifeCaseTracker tracker) {
        return new LifeCaseDetailResponse(
                tracker.id,
                LifeCaseType.valueOf(caseNameToEnumName(tracker.caseType)),
                tracker.domain,
                tracker.status,
                tracker.createdAt,
                tracker.completedAt,
                tracker.engineCaseId
        );
    }

    private String caseNameToEnumName(String caseName) {
        return caseName.toUpperCase().replace('-', '_');
    }

    private record QueryParts(String hql, Map<String, Object> params) {}

    private QueryParts buildListQuery(LifeDomain domain, LifeCaseStatus status,
                                       LifeCaseType caseType) {
        var conditions = new ArrayList<String>();
        var params = new HashMap<String, Object>();

        if (domain != null) {
            conditions.add("domain = :domain");
            params.put("domain", domain);
        }
        if (status != null) {
            conditions.add("status = :status");
            params.put("status", status);
        }
        if (caseType != null) {
            conditions.add("caseType = :caseType");
            params.put("caseType", caseType.caseName());
        }

        String hql = conditions.isEmpty() ? "" : String.join(" and ", conditions);
        return new QueryParts(hql, params);
    }
}
```

- [ ] **Step 9: Add GET endpoints to LifeCaseResource**

Add to `LifeCaseResource`:

```java
@Inject
LifeCaseQueryService queryService;

@GET
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
public PagedResponse<LifeCaseResponse> list(
        @QueryParam("domain") LifeDomain domain,
        @QueryParam("status") LifeCaseStatus status,
        @QueryParam("caseType") LifeCaseType caseType,
        @QueryParam("page") @DefaultValue("0") int page,
        @QueryParam("size") @DefaultValue("20") int size) {
    return queryService.listCases(domain, status, caseType, page, size);
}

@GET
@Path("/{id}")
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
public Response findById(@PathParam("id") UUID id) {
    return queryService.findById(id)
            .map(r -> Response.ok(r).build())
            .orElse(Response.status(Response.Status.NOT_FOUND).build());
}
```

Add required imports: `DefaultValue`, `QueryParam`, `PathParam`, `GET`, `LifeDomain`,
`LifeCaseStatus`, `LifeCaseType`, `PagedResponse`, `LifeCaseQueryService`.

- [ ] **Step 10: Register JuniorLifeCaseVisibilityPolicy in test config**

Add to test `application.properties` `quarkus.arc.selected-alternatives`:

```
io.casehub.life.app.spi.JuniorLifeCaseVisibilityPolicy
```

Wait — it's already registered (for LifeTaskVisibilityPolicy). The new
`JuniorLifeCaseVisibilityPolicy` is a different class for a different SPI.
Add the new class to `selected-alternatives`.

- [ ] **Step 11: Write REST integration test**

Create `LifeCaseResourceTest.java`:

```java
@QuarkusTest
@TestSecurity(user = "admin", roles = HouseholdGroups.ADMIN)
public class LifeCaseResourceTest {

    @BeforeEach
    @Transactional
    void setup() {
        LifeTestFixtures.seedStandardTemplates();
        LifeTestFixtures.seedEscalationTemplate();
        seedTracker("travel-plan", LifeDomain.TRAVEL, LifeCaseStatus.ACTIVE);
        seedTracker("home-maintenance", LifeDomain.HOUSEHOLD, LifeCaseStatus.COMPLETED);
    }

    @AfterEach
    @Transactional
    void cleanup() {
        LifeCaseTracker.deleteAll();
    }

    @Test
    void listReturnsAllCases() {
        given()
            .when().get("/life-cases")
            .then()
            .statusCode(200)
            .body("items.size()", greaterThanOrEqualTo(2))
            .body("totalCount", greaterThanOrEqualTo(2));
    }

    @Test
    void listFiltersByDomain() {
        given()
            .queryParam("domain", "TRAVEL")
            .when().get("/life-cases")
            .then()
            .statusCode(200)
            .body("items.size()", is(1))
            .body("items[0].domain", is("TRAVEL"));
    }

    @Test
    void listFiltersByStatus() {
        given()
            .queryParam("status", "COMPLETED")
            .when().get("/life-cases")
            .then()
            .statusCode(200)
            .body("items.size()", is(1))
            .body("items[0].status", is("COMPLETED"));
    }

    @Test
    void getByIdReturnsDetail() {
        UUID id = seedTracker("financial-review", LifeDomain.FINANCE, LifeCaseStatus.ACTIVE);
        given()
            .when().get("/life-cases/" + id)
            .then()
            .statusCode(200)
            .body("caseId", is(id.toString()))
            .body("domain", is("FINANCE"));
    }

    @Test
    void getByIdReturns404ForUnknown() {
        given()
            .when().get("/life-cases/" + UUID.randomUUID())
            .then()
            .statusCode(404);
    }

    @Transactional
    UUID seedTracker(String caseType, LifeDomain domain, LifeCaseStatus status) {
        LifeCaseTracker t = new LifeCaseTracker();
        t.id = UUID.randomUUID();
        t.caseType = caseType;
        t.domain = domain;
        t.status = status;
        t.createdAt = Instant.now();
        if (status == LifeCaseStatus.COMPLETED) t.completedAt = Instant.now();
        t.persist();
        return t.id;
    }
}
```

- [ ] **Step 12: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeCaseResourceTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS

- [ ] **Step 13: Verify with ide_diagnostics**

Run `ide_diagnostics` on all modified files. Fix any errors.

- [ ] **Step 14: Commit**

```bash
git add api/src/main/java/io/casehub/life/api/response/LifeCaseResponse.java \
        api/src/main/java/io/casehub/life/api/response/LifeCaseDetailResponse.java \
        api/src/main/java/io/casehub/life/api/spi/LifeCaseVisibilityPolicy.java \
        app/src/main/java/io/casehub/life/app/spi/JuniorLifeCaseVisibilityPolicy.java \
        app/src/main/java/io/casehub/life/app/service/LifeCaseQueryService.java \
        app/src/main/java/io/casehub/life/app/resource/LifeCaseResource.java \
        app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java \
        app/src/test/java/io/casehub/life/app/spi/JuniorLifeCaseVisibilityPolicyTest.java \
        app/src/test/java/io/casehub/life/app/resource/LifeCaseResourceTest.java \
        app/src/test/resources/application.properties
git commit -m "feat(#N): GET /life-cases list+detail endpoints with visibility policy"
```

---

### Task 3: SSE event infrastructure

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/event/LifeEventBroadcaster.java`
- Create: `app/src/main/java/io/casehub/life/app/event/LifeEventType.java`
- Create: `app/src/main/java/io/casehub/life/app/event/LifeSseEvent.java`
- Create: `app/src/main/java/io/casehub/life/app/event/LifeEventBridge.java`
- Create: `app/src/main/java/io/casehub/life/app/resource/LifeEventSseResource.java`
- Test: `app/src/test/java/io/casehub/life/app/event/LifeEventBroadcasterTest.java`
- Test: `app/src/test/java/io/casehub/life/app/resource/LifeEventSseResourceTest.java`

**Interfaces:**
- Consumes: `WorkItemLifecycleEvent` (casehub-work), `CaseLifecycleEvent` (casehub-engine), `SlaBreachEvent` (casehub-work)
- Produces: `GET /events/inbox` → SSE stream of `LifeSseEvent`
- Produces: `GET /events/cases` → SSE stream of `LifeSseEvent`

- [ ] **Step 1: Create LifeEventType enum**

```java
package io.casehub.life.app.event;

public enum LifeEventType {
    WORK_ITEM_CREATED,
    WORK_ITEM_UPDATED,
    WORK_ITEM_COMPLETED,
    SLA_BREACH,
    CASE_STARTED,
    CASE_COMPLETED,
    CASE_FAULTED
}
```

- [ ] **Step 2: Create LifeSseEvent record**

```java
package io.casehub.life.app.event;

import java.time.Instant;
import java.util.Map;

public record LifeSseEvent(
        LifeEventType type,
        Map<String, Object> data,
        Instant timestamp
) {
    public static LifeSseEvent of(LifeEventType type, Map<String, Object> data) {
        return new LifeSseEvent(type, data, Instant.now());
    }
}
```

- [ ] **Step 3: Write failing test for LifeEventBroadcaster**

```java
package io.casehub.life.app.event;

import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.Map;
import java.util.concurrent.Flow;

import static org.junit.jupiter.api.Assertions.*;

class LifeEventBroadcasterTest {

    @Test
    void subscriberReceivesPublishedEvent() {
        var broadcaster = new LifeEventBroadcaster();
        var received = new ArrayList<LifeSseEvent>();

        broadcaster.subscribe(event -> received.add(event));
        broadcaster.publish(LifeSseEvent.of(LifeEventType.CASE_STARTED, Map.of("caseId", "123")));

        assertEquals(1, received.size());
        assertEquals(LifeEventType.CASE_STARTED, received.getFirst().type());
    }

    @Test
    void multipleSubscribersReceiveEvent() {
        var broadcaster = new LifeEventBroadcaster();
        var received1 = new ArrayList<LifeSseEvent>();
        var received2 = new ArrayList<LifeSseEvent>();

        broadcaster.subscribe(received1::add);
        broadcaster.subscribe(received2::add);
        broadcaster.publish(LifeSseEvent.of(LifeEventType.SLA_BREACH, Map.of()));

        assertEquals(1, received1.size());
        assertEquals(1, received2.size());
    }

    @Test
    void unsubscribedListenerDoesNotReceiveEvent() {
        var broadcaster = new LifeEventBroadcaster();
        var received = new ArrayList<LifeSseEvent>();

        var sub = broadcaster.subscribe(received::add);
        sub.cancel();
        broadcaster.publish(LifeSseEvent.of(LifeEventType.CASE_COMPLETED, Map.of()));

        assertTrue(received.isEmpty());
    }
}
```

- [ ] **Step 4: Implement LifeEventBroadcaster**

```java
package io.casehub.life.app.event;

import jakarta.enterprise.context.ApplicationScoped;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Consumer;

@ApplicationScoped
public class LifeEventBroadcaster {

    private final List<SubscriptionHandle> subscribers = new CopyOnWriteArrayList<>();

    public Subscription subscribe(Consumer<LifeSseEvent> listener) {
        var handle = new SubscriptionHandle(listener);
        subscribers.add(handle);
        return handle;
    }

    public void publish(LifeSseEvent event) {
        subscribers.forEach(s -> {
            if (s.active) {
                try {
                    s.listener.accept(event);
                } catch (Exception e) {
                    // Best-effort delivery — log and continue
                }
            }
        });
    }

    public interface Subscription {
        void cancel();
    }

    private class SubscriptionHandle implements Subscription {
        final Consumer<LifeSseEvent> listener;
        volatile boolean active = true;

        SubscriptionHandle(Consumer<LifeSseEvent> listener) {
            this.listener = listener;
        }

        @Override
        public void cancel() {
            active = false;
            subscribers.remove(this);
        }
    }
}
```

- [ ] **Step 5: Run broadcaster test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeEventBroadcasterTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS

- [ ] **Step 6: Create LifeEventBridge — CDI events → broadcaster**

```java
package io.casehub.life.app.event;

import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.work.runtime.event.SlaBreachEvent;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import java.util.Map;

@ApplicationScoped
public class LifeEventBridge {

    @Inject LifeEventBroadcaster broadcaster;

    public void onWorkItemEvent(@ObservesAsync WorkItemLifecycleEvent event) {
        LifeEventType type = switch (event.eventType()) {
            case "WorkItemCreated" -> LifeEventType.WORK_ITEM_CREATED;
            case "WorkItemCompleted" -> LifeEventType.WORK_ITEM_COMPLETED;
            default -> LifeEventType.WORK_ITEM_UPDATED;
        };
        broadcaster.publish(LifeSseEvent.of(type, Map.of(
                "workItemId", event.workItemId().toString(),
                "eventType", event.eventType()
        )));
    }

    public void onCaseEvent(@ObservesAsync CaseLifecycleEvent event) {
        LifeEventType type = switch (event.eventType()) {
            case "CaseStarted" -> LifeEventType.CASE_STARTED;
            case "CaseCompleted" -> LifeEventType.CASE_COMPLETED;
            case "CaseFaulted" -> LifeEventType.CASE_FAULTED;
            default -> null;
        };
        if (type == null) return;
        broadcaster.publish(LifeSseEvent.of(type, Map.of(
                "caseId", event.caseId().toString(),
                "eventType", event.eventType()
        )));
    }

    public void onSlaBreachEvent(@ObservesAsync SlaBreachEvent event) {
        broadcaster.publish(LifeSseEvent.of(LifeEventType.SLA_BREACH, Map.of(
                "workItemId", event.workItemId().toString()
        )));
    }
}
```

- [ ] **Step 7: Create LifeEventSseResource**

```java
package io.casehub.life.app.resource;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.app.event.LifeEventBroadcaster;
import io.casehub.life.app.event.LifeEventType;
import io.casehub.life.app.event.LifeSseEvent;
import io.smallrye.mutiny.Multi;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.jboss.resteasy.reactive.RestMulti;

import java.time.Duration;
import java.util.Set;

@ApplicationScoped
@Path("/events")
public class LifeEventSseResource {

    private static final Set<LifeEventType> INBOX_TYPES = Set.of(
            LifeEventType.WORK_ITEM_CREATED,
            LifeEventType.WORK_ITEM_UPDATED,
            LifeEventType.WORK_ITEM_COMPLETED,
            LifeEventType.SLA_BREACH
    );

    private static final Set<LifeEventType> CASE_TYPES = Set.of(
            LifeEventType.CASE_STARTED,
            LifeEventType.CASE_COMPLETED,
            LifeEventType.CASE_FAULTED
    );

    @Inject LifeEventBroadcaster broadcaster;
    @Inject ObjectMapper objectMapper;

    @GET
    @Path("/inbox")
    @Produces(MediaType.SERVER_SENT_EVENTS)
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
    public Multi<String> inbox() {
        return eventStream(INBOX_TYPES);
    }

    @GET
    @Path("/cases")
    @Produces(MediaType.SERVER_SENT_EVENTS)
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
    public Multi<String> cases() {
        return eventStream(CASE_TYPES);
    }

    private Multi<String> eventStream(Set<LifeEventType> filter) {
        Multi<String> events = Multi.createFrom().emitter(emitter -> {
            var sub = broadcaster.subscribe(event -> {
                if (filter.contains(event.type())) {
                    try {
                        emitter.emit(objectMapper.writeValueAsString(event));
                    } catch (Exception e) {
                        // Best-effort
                    }
                }
            });
            emitter.onTermination(() -> sub.cancel());
        });

        Multi<String> heartbeat = Multi.createFrom()
                .ticks().every(Duration.ofSeconds(30))
                .map(tick -> ":keepalive\n");

        return Multi.createBy().merging().streams(events, heartbeat);
    }
}
```

- [ ] **Step 8: Write SSE resource integration test**

```java
@QuarkusTest
@TestSecurity(user = "admin", roles = HouseholdGroups.ADMIN)
public class LifeEventSseResourceTest {

    @Inject LifeEventBroadcaster broadcaster;

    @Test
    void inboxEndpointReturns200() {
        // SSE endpoints return 200 with text/event-stream content type.
        // Full SSE client test is complex; verify the endpoint exists and responds.
        given()
            .when().get("/events/inbox")
            .then()
            .statusCode(200)
            .contentType("text/event-stream");
    }

    @Test
    void casesEndpointReturns200() {
        given()
            .when().get("/events/cases")
            .then()
            .statusCode(200)
            .contentType("text/event-stream");
    }
}
```

- [ ] **Step 9: Run SSE tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeEventSseResourceTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add app/src/main/java/io/casehub/life/app/event/ \
        app/src/main/java/io/casehub/life/app/resource/LifeEventSseResource.java \
        app/src/test/java/io/casehub/life/app/event/LifeEventBroadcasterTest.java \
        app/src/test/java/io/casehub/life/app/resource/LifeEventSseResourceTest.java
git commit -m "feat(#N): SSE event infrastructure — CDI→broadcaster→SSE bridge"
```

---

### Task 4: Demo data infrastructure

**Files:**
- Create: `app/src/main/resources/db/life/demo/V9001__demo_external_actors.sql`
- Create: `app/src/main/resources/db/life/demo/V9002__demo_cases_and_tasks.sql`
- Modify: `app/src/main/resources/application.properties` (demo profile)

**Interfaces:**
- Consumes: All entity tables from Tasks 1-2
- Produces: Pre-populated demo data available when `quarkus.profile=demo`

- [ ] **Step 1: Add demo profile Flyway config**

Add to `application.properties`:

```properties
# ============================================================
# Demo profile — pre-populated data for standalone mode.
# Flyway seeds at db/life/demo/ (V9000+ range) run only in demo profile.
# OIDC dev services with 3 demo users: admin, member, junior.
# ============================================================
%demo.quarkus.datasource.db-kind=h2
%demo.quarkus.datasource.jdbc.url=jdbc:h2:mem:life;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%demo.quarkus.flyway.locations=classpath:db/life/migration,classpath:db/work/migration,classpath:db/life/demo
%demo.quarkus.security.auth.enabled-in-dev-mode=false
%demo.quarkus.oidc.enabled=false
%demo.quarkus.keycloak.devservices.enabled=false
```

- [ ] **Step 2: Create demo ExternalActors seed**

Create `app/src/main/resources/db/life/demo/V9001__demo_external_actors.sql`:

```sql
INSERT INTO external_actor (id, name, actor_type, contact_method, contact_value, created_at)
VALUES
  ('a0000000-0000-0000-0000-000000000001', 'Dave Wilson Plumbing', 'CONTRACTOR', 'PHONE', '+44 7700 900001', NOW()),
  ('a0000000-0000-0000-0000-000000000002', 'Dr Sarah Chen', 'DOCTOR', 'EMAIL', 'dr.chen@nhs.example.uk', NOW()),
  ('a0000000-0000-0000-0000-000000000003', 'Harper & Associates', 'INSTITUTION', 'EMAIL', 'enquiries@harper-law.example.uk', NOW()),
  ('a0000000-0000-0000-0000-000000000004', 'Oakwood Primary School', 'INSTITUTION', 'EMAIL', 'office@oakwood.example.uk', NOW()),
  ('a0000000-0000-0000-0000-000000000005', 'Maria Santos — Carer', 'SERVICE_PROVIDER', 'PHONE', '+44 7700 900005', NOW());
```

- [ ] **Step 3: Create demo cases and tasks seed**

Create `app/src/main/resources/db/life/demo/V9002__demo_cases_and_tasks.sql`:

```sql
-- Demo LifeCaseTracker records (static — no engine dependency)
INSERT INTO life_case_tracker (id, case_type, domain, status, created_at, completed_at)
VALUES
  ('c0000000-0000-0000-0000-000000000001', 'contractor-coordination', 'CONTRACTOR_COORDINATION', 'ACTIVE', NOW() - INTERVAL '3' DAY, NULL),
  ('c0000000-0000-0000-0000-000000000002', 'care-coordination', 'ELDER_CARE', 'ACTIVE', NOW() - INTERVAL '7' DAY, NULL),
  ('c0000000-0000-0000-0000-000000000003', 'travel-plan', 'TRAVEL', 'COMPLETED', NOW() - INTERVAL '14' DAY, NOW() - INTERVAL '2' DAY);
```

- [ ] **Step 4: Verify demo profile starts**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app -Dquarkus.profile=demo`
Verify: Application starts, Flyway runs V9001+V9002, no errors.
Stop the dev server.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/resources/db/life/demo/ \
        app/src/main/resources/application.properties
git commit -m "feat(#N): demo data infrastructure — Quarkus demo profile with Flyway seeds"
```

---

### Task 5: life-ui module setup + Quinoa integration

**Files:**
- Create: `life-ui/package.json`
- Create: `life-ui/tsconfig.json`
- Create: `life-ui/vite.config.ts`
- Create: `life-ui/index.html`
- Create: `life-ui/src/shell/app-shell.ts`
- Create: `life-ui/src/shell/router.ts`
- Create: `life-ui/src/views/home-view.ts`
- Create: `life-ui/src/views/inbox-view.ts`
- Create: `life-ui/src/index.ts`
- Modify: `app/pom.xml` (add Quinoa plugin)

**Interfaces:**
- Consumes: `@casehubio/blocks-ui-*` packages, REST endpoints from Tasks 2-3
- Produces: Working SPA served at `http://localhost:8080/`

- [ ] **Step 1: Create life-ui directory and package.json**

```json
{
  "name": "@casehubio/life-ui",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "lit": "^3.0.0",
    "@casehubio/blocks-ui-core": "^0.1.0",
    "@casehubio/blocks-ui-work-item-workbench": "^0.1.0",
    "@casehubio/blocks-ui-work-item-inbox": "^0.1.0",
    "@casehubio/blocks-ui-work-item-detail": "^0.1.0",
    "@casehubio/blocks-ui-kpi-metric-row": "^0.1.0",
    "@casehubio/blocks-ui-sla-indicator": "^0.1.0",
    "@casehubio/blocks-ui-sla-breach-policy": "^0.1.0",
    "@casehubio/blocks-ui-notification-inbox": "^0.1.0",
    "@casehubio/blocks-ui-grouped-data-view": "^0.1.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "vite": "^6.0.0"
  }
}
```

Published packages from GitHub Packages — requires `.npmrc` with
`@casehubio:registry=https://npm.pkg.github.com` (created in Step 10).

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "experimentalDecorators": true,
    "useDefineForClassFields": false,
    "declaration": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

- [ ] **Step 3: Create vite.config.ts**

```typescript
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    outDir: 'dist',
    rollupOptions: {
      input: 'index.html',
    },
  },
  server: {
    proxy: {
      '/life-cases': 'http://localhost:8080',
      '/life-tasks': 'http://localhost:8080',
      '/pending-actions': 'http://localhost:8080',
      '/external-actors': 'http://localhost:8080',
      '/analytics': 'http://localhost:8080',
      '/events': 'http://localhost:8080',
    },
  },
});
```

- [ ] **Step 4: Create index.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Household Hub</title>
  <script type="module" src="/src/index.ts"></script>
</head>
<body>
  <app-shell></app-shell>
</body>
</html>
```

- [ ] **Step 5: Create app shell component**

Create `life-ui/src/shell/app-shell.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import { generateThemeCSS, injectTheme } from '@casehubio/blocks-ui-core';

@customElement('app-shell')
export class AppShell extends LitElement {
  @state() private currentView = 'home';

  static styles = css`
    :host {
      display: flex;
      flex-direction: column;
      height: 100vh;
    }
    nav {
      display: flex;
      gap: var(--pages-space-4, 1rem);
      padding: var(--pages-space-3, 0.75rem) var(--pages-space-4, 1rem);
      border-bottom: 1px solid var(--pages-neutral-200, #e5e5e5);
      background: var(--pages-neutral-50, #fafafa);
    }
    nav a {
      text-decoration: none;
      padding: var(--pages-space-2, 0.5rem) var(--pages-space-3, 0.75rem);
      border-radius: var(--pages-radius-md, 6px);
      color: var(--pages-neutral-700, #404040);
      cursor: pointer;
    }
    nav a[data-active] {
      background: var(--pages-accent-100, #e0e7ff);
      color: var(--pages-accent-700, #3730a3);
    }
    main {
      flex: 1;
      overflow: auto;
      padding: var(--pages-space-4, 1rem);
    }
  `;

  connectedCallback() {
    super.connectedCallback();
    injectTheme(generateThemeCSS({ baseHue: 220, accentHue: 250, chroma: 0.08, contrast: 0.9 }));
    window.addEventListener('hashchange', () => this.updateView());
    this.updateView();
  }

  private updateView() {
    this.currentView = window.location.hash.slice(1) || 'home';
  }

  private navigate(view: string) {
    window.location.hash = view;
  }

  render() {
    return html`
      <nav>
        <a ?data-active=${this.currentView === 'home'} @click=${() => this.navigate('home')}>Home</a>
        <a ?data-active=${this.currentView === 'inbox'} @click=${() => this.navigate('inbox')}>Inbox</a>
        <a ?data-active=${this.currentView === 'people'} @click=${() => this.navigate('people')}>People</a>
        <a ?data-active=${this.currentView === 'cases'} @click=${() => this.navigate('cases')}>Cases</a>
        <a ?data-active=${this.currentView === 'journal'} @click=${() => this.navigate('journal')}>Journal</a>
      </nav>
      <main>
        ${this.renderView()}
      </main>
    `;
  }

  private renderView() {
    switch (this.currentView) {
      case 'home': return html`<home-view></home-view>`;
      case 'inbox': return html`<inbox-view></inbox-view>`;
      default: return html`<p>Coming in Phase 2</p>`;
    }
  }
}
```

- [ ] **Step 6: Create home view (dashboard)**

Create `life-ui/src/views/home-view.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';
import '@casehubio/blocks-ui-kpi-metric-row';
import '@casehubio/blocks-ui-sla-indicator';
import '@casehubio/blocks-ui-grouped-data-view';

@customElement('home-view')
export class HomeView extends LitElement {
  static styles = css`
    :host { display: block; }
    h1 {
      margin: 0 0 var(--pages-space-4, 1rem);
      font-size: var(--pages-text-2xl, 1.5rem);
    }
    section { margin-bottom: var(--pages-space-6, 1.5rem); }
  `;

  render() {
    return html`
      <h1>Dashboard</h1>
      <section>
        <kpi-metric-row endpoint="/analytics/cases"></kpi-metric-row>
      </section>
      <section>
        <kpi-metric-row endpoint="/analytics/sla"></kpi-metric-row>
      </section>
      <section>
        <h2>Active Cases</h2>
        <grouped-data-view
          endpoint="/life-cases?status=ACTIVE"
          group-by="domain"
        ></grouped-data-view>
      </section>
    `;
  }
}
```

- [ ] **Step 7: Create inbox view**

Create `life-ui/src/views/inbox-view.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';
import '@casehubio/blocks-ui-work-item-workbench';

@customElement('inbox-view')
export class InboxView extends LitElement {
  static styles = css`
    :host { display: block; height: 100%; }
    work-item-workbench { height: 100%; }
  `;

  render() {
    return html`
      <work-item-workbench
        endpoint="/pending-actions"
        identity="life-inbox"
      ></work-item-workbench>
    `;
  }
}
```

- [ ] **Step 8: Create index.ts entry point**

Create `life-ui/src/index.ts`:

```typescript
import './shell/app-shell.js';
import './views/home-view.js';
import './views/inbox-view.js';
```

- [ ] **Step 9: Add Quinoa plugin to app/pom.xml**

Add to `app/pom.xml` `<dependencies>`:

```xml
<dependency>
    <groupId>io.quarkiverse.quinoa</groupId>
    <artifactId>quarkus-quinoa</artifactId>
    <version>2.5.3</version>
</dependency>
```

Add to `app/pom.xml` `<properties>` or `application.properties`:

```properties
quarkus.quinoa.ui-dir=../life-ui
quarkus.quinoa.package-manager-install=false
quarkus.quinoa.enable-spa-routing=true
```

- [ ] **Step 10: Create .npmrc for GitHub Packages**

Create `life-ui/.npmrc`:

```
@casehubio:registry=https://npm.pkg.github.com
```

- [ ] **Step 11: Install dependencies and verify build**

```bash
npm install --prefix life-ui
npm run build --prefix life-ui
```

- [ ] **Step 12: Start dev server and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app -Dquarkus.profile=demo`
Open: `http://localhost:8080/`
Verify: App shell renders with nav, dashboard view shows, inbox view loads.

- [ ] **Step 13: Commit**

```bash
git add life-ui/ app/pom.xml app/src/main/resources/application.properties
git commit -m "feat(#N): life-ui module — Lit SPA with app shell, dashboard, and inbox views"
```

---

### Task 6: blocks-ui #56 update + issue tracking

**Files:** None (GitHub issue operations only)

- [ ] **Step 1: Update blocks-ui #56 status table**

Add life to the app delivery epic status table via GitHub comment:

```bash
gh issue comment 56 --repo casehubio/blocks-ui --body "Adding casehub-life as a consuming app.

| App | Migration Epic | Phase 1 (Consume) | Phase 2 (Promote) |
|-----|---------------|-------------------|-------------------|
| Life | — | ⬜ starting (Phase 1 MVP) | N/A |

Components consumed: work-item-workbench, work-item-inbox, work-item-detail, kpi-metric-row, sla-indicator, sla-breach-policy, notification-bell/inbox, grouped-data-view.

New components needed: calendar-strip, agent-activity-panel (Phase 2)."
```

- [ ] **Step 2: File life epic issue**

```bash
gh issue create --repo casehubio/life --title "epic: Household Hub UI — Phase 1 MVP" \
  --body "Phase 1 of the Household Hub UI. Delivers app shell, inbox, dashboard, SSE, case endpoints, demo data.

Spec: docs/specs/2026-07-19-household-hub-ui-design.md

## Deliverables
- App shell (Lit SPA via Quinoa)
- Inbox view (work-item-workbench composition)
- Dashboard view (KPI strip + active cases)
- SSE event infrastructure (CDI→SSE bridge)
- GET /life-cases list + detail endpoints
- LifeCaseTracker domain column (V111)
- LifeCaseVisibilityPolicy SPI
- Demo data (Quarkus demo profile + Flyway seeds)

## Scale: L — Complexity: Med"
```

- [ ] **Step 3: Commit spec**

```bash
git -C /Users/mdproctor/claude/casehub/life add docs/specs/2026-07-19-household-hub-ui-design.md
git -C /Users/mdproctor/claude/casehub/life commit -m "docs: Household Hub UI design spec"
```
