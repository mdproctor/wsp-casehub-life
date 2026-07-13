# API Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #62 — ExternalActor search, filter, trust history, and activity timeline
**Issue group:** #62, #63, #64

**Goal:** Add read-side API surface to casehub-life: ExternalActor search/trust/activity, pending actions, and case outcome analytics.

**Architecture:** Three concerns, three homes. ExternalActor entity-level queries extend the existing resource. Pending actions and analytics are cross-cutting queries with dedicated resources. All features query existing data stores — no new entities or migrations.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache (Active Record), JAX-RS, H2 (test), casehub-ledger (qhorus PU for LedgerAttestation/ActorTrustScore)

## Global Constraints

- `@Blocking @ApplicationScoped` on all REST resources (GE-20260518-e4fa52)
- Class-level `@Produces/@Consumes(APPLICATION_JSON)` (PP-20260526-d0b921)
- `@RolesAllowed` on every endpoint
- `@Transactional` on service methods only (PP-20260526-75d9c9) — required on paginated methods (count+page consistency), not needed on read-only analytics
- Use Panache `find().page(Page.of(page, size))` — never `list()` for paginated queries (GE-20260523-06e8b6)
- No switch/if-else on LifeDomain/LifeCaseType in services (descriptor-handler-no-domain-switches protocol)
- `tenancyId` must be `"278776f9-e1b0-46fb-9032-8bddebdcf9ce"` on all test WorkItems (V35 NOT NULL)
- api/ records: pure Java, zero framework imports, no JPA
- IntelliJ MCP: use `ide_insert_member`, `ide_replace_member`, `ide_edit_member` for existing files; `Write` for new files only

---

### Task 1: api/ response records, PagedResponse, and Urgency enum

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/response/PagedResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/response/TrustHistoryEntry.java`
- Create: `api/src/main/java/io/casehub/life/api/response/ActorActivityEntry.java`
- Create: `api/src/main/java/io/casehub/life/api/response/PendingActionResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/Urgency.java`
- Create: `api/src/main/java/io/casehub/life/api/response/CaseStatisticsResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/response/SlaComplianceResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/response/TrustAnalyticsResponse.java`
- Test: `api/src/test/java/io/casehub/life/api/UrgencyTest.java`

**Interfaces:**
- Produces: `PagedResponse<T>`, `TrustHistoryEntry`, `ActorActivityEntry`, `PendingActionResponse`, `Urgency`, `CaseStatisticsResponse` (with `CaseTypeStats`), `SlaComplianceResponse` (with `DomainSlaStats`), `TrustAnalyticsResponse` (with `ActorScoreSummary`)

- [ ] **Step 1: Write Urgency enum with classify() method**

```java
// api/src/main/java/io/casehub/life/api/Urgency.java
package io.casehub.life.api;

import java.time.Duration;
import java.time.Instant;

public enum Urgency {
    OVERDUE, DUE_SOON, NORMAL, NO_DEADLINE;

    public static Urgency classify(Instant expiresAt, Instant now, int dueSoonHours) {
        if (expiresAt == null) return NO_DEADLINE;
        if (expiresAt.isBefore(now)) return OVERDUE;
        if (Duration.between(now, expiresAt).toHours() <= dueSoonHours) return DUE_SOON;
        return NORMAL;
    }

    public static Long daysOverdue(Instant expiresAt, Instant now) {
        if (expiresAt == null || !expiresAt.isBefore(now)) return null;
        return Duration.between(expiresAt, now).toDays();
    }
}
```

- [ ] **Step 2: Write failing test for Urgency.classify()**

```java
// api/src/test/java/io/casehub/life/api/UrgencyTest.java
package io.casehub.life.api;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class UrgencyTest {

    private static final Instant NOW = Instant.parse("2026-07-12T12:00:00Z");

    @Test
    void classify_nullExpiresAt_returnsNoDeadline() {
        assertEquals(Urgency.NO_DEADLINE, Urgency.classify(null, NOW, 24));
    }

    @Test
    void classify_pastDeadline_returnsOverdue() {
        Instant expired = NOW.minusSeconds(3600);
        assertEquals(Urgency.OVERDUE, Urgency.classify(expired, NOW, 24));
    }

    @Test
    void classify_within24Hours_returnsDueSoon() {
        Instant soonExpires = NOW.plusSeconds(3600 * 12);
        assertEquals(Urgency.DUE_SOON, Urgency.classify(soonExpires, NOW, 24));
    }

    @Test
    void classify_beyondWindow_returnsNormal() {
        Instant farExpires = NOW.plusSeconds(3600 * 48);
        assertEquals(Urgency.NORMAL, Urgency.classify(farExpires, NOW, 24));
    }

    @Test
    void classify_customDueSoonHours() {
        Instant in6Hours = NOW.plusSeconds(3600 * 6);
        assertEquals(Urgency.DUE_SOON, Urgency.classify(in6Hours, NOW, 8));
        assertEquals(Urgency.NORMAL, Urgency.classify(in6Hours, NOW, 4));
    }

    @Test
    void classify_exactlyAtDeadline_returnsOverdue() {
        assertEquals(Urgency.OVERDUE, Urgency.classify(NOW, NOW, 24));
    }

    @Test
    void classify_exactlyAtWindowBoundary_returnsDueSoon() {
        Instant exactly24h = NOW.plusSeconds(3600 * 24);
        assertEquals(Urgency.DUE_SOON, Urgency.classify(exactly24h, NOW, 24));
    }

    @Test
    void daysOverdue_nullExpires_returnsNull() {
        assertNull(Urgency.daysOverdue(null, NOW));
    }

    @Test
    void daysOverdue_futureDeadline_returnsNull() {
        assertNull(Urgency.daysOverdue(NOW.plusSeconds(3600), NOW));
    }

    @Test
    void daysOverdue_3DaysOverdue_returns3() {
        Instant expired = NOW.minusSeconds(3600 * 24 * 3);
        assertEquals(3L, Urgency.daysOverdue(expired, NOW));
    }
}
```

- [ ] **Step 3: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=UrgencyTest --batch-mode`
Expected: PASS

- [ ] **Step 4: Write all response records**

```java
// api/src/main/java/io/casehub/life/api/response/PagedResponse.java
package io.casehub.life.api.response;

import java.util.List;

public record PagedResponse<T>(List<T> items, int page, int size, long totalCount) {}
```

```java
// api/src/main/java/io/casehub/life/api/response/TrustHistoryEntry.java
package io.casehub.life.api.response;

import java.time.Instant;

public record TrustHistoryEntry(
        Instant occurredAt,
        String capabilityTag,
        String dimension,
        Double score,
        String verdict) {}
```

```java
// api/src/main/java/io/casehub/life/api/response/ActorActivityEntry.java
package io.casehub.life.api.response;

import io.casehub.life.api.LifeDomain;
import java.time.Instant;
import java.util.UUID;

public record ActorActivityEntry(
        UUID workItemId,
        String title,
        LifeDomain domain,
        String status,
        String scope,
        Instant createdAt,
        Instant completedAt,
        String outcome) {}
```

```java
// api/src/main/java/io/casehub/life/api/response/PendingActionResponse.java
package io.casehub.life.api.response;

import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.Urgency;
import java.time.Instant;
import java.util.UUID;

public record PendingActionResponse(
        UUID workItemId,
        String title,
        String description,
        String status,
        LifeDomain domain,
        String candidateGroups,
        Instant createdAt,
        Instant expiresAt,
        Urgency urgency,
        Long daysOverdue) {}
```

```java
// api/src/main/java/io/casehub/life/api/response/CaseStatisticsResponse.java
package io.casehub.life.api.response;

import java.util.List;

public record CaseStatisticsResponse(List<CaseTypeStats> entries) {

    public record CaseTypeStats(
            String caseType,
            long total,
            long active,
            long completed,
            long failed,
            Double avgResolutionHours,
            Double p50ResolutionHours,
            Double p95ResolutionHours,
            Double completionRate) {}
}
```

```java
// api/src/main/java/io/casehub/life/api/response/SlaComplianceResponse.java
package io.casehub.life.api.response;

import java.util.List;

public record SlaComplianceResponse(List<DomainSlaStats> entries) {

    public record DomainSlaStats(
            String domain,
            long totalWithSla,
            long breachedCount,
            Double complianceRate,
            Double avgBreachLatencyHours) {}
}
```

```java
// api/src/main/java/io/casehub/life/api/response/TrustAnalyticsResponse.java
package io.casehub.life.api.response;

import java.util.List;
import java.util.Map;
import java.util.UUID;

public record TrustAnalyticsResponse(
        int actorCount,
        Double avgGlobalScore,
        Map<String, Double> dimensionAverages,
        List<ActorScoreSummary> lowestScoreActors) {

    public record ActorScoreSummary(UUID actorId, String name, Double globalScore) {}
}
```

- [ ] **Step 5: Run full api build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add api/src/main/java/io/casehub/life/api/Urgency.java api/src/main/java/io/casehub/life/api/response/PagedResponse.java api/src/main/java/io/casehub/life/api/response/TrustHistoryEntry.java api/src/main/java/io/casehub/life/api/response/ActorActivityEntry.java api/src/main/java/io/casehub/life/api/response/PendingActionResponse.java api/src/main/java/io/casehub/life/api/response/CaseStatisticsResponse.java api/src/main/java/io/casehub/life/api/response/SlaComplianceResponse.java api/src/main/java/io/casehub/life/api/response/TrustAnalyticsResponse.java api/src/test/java/io/casehub/life/api/UrgencyTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#62): api/ response records, PagedResponse, and Urgency enum with tests"
```

---

### Task 2: ExternalActor search and pagination

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/service/ExternalActorService.java` — replace `list()` with paginated search
- Modify: `app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java` — update list endpoint signature, add JUNIOR to GET @RolesAllowed
- Create: `app/src/test/java/io/casehub/life/app/ExternalActorSearchTest.java`

**Interfaces:**
- Consumes: `PagedResponse<ExternalActorResponse>` from Task 1
- Produces: `ExternalActorService.search(name, actorType, contactMethod, erasedOnly, page, size)` returning `PagedResponse<ExternalActorResponse>`

- [ ] **Step 1: Write failing integration test for search**

```java
// app/src/test/java/io/casehub/life/app/ExternalActorSearchTest.java
package io.casehub.life.app;

import io.casehub.life.api.LifeActorType;
import io.casehub.life.app.entity.ExternalActor;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class ExternalActorSearchTest {

    @BeforeEach
    @Transactional
    void seedActors() {
        ExternalActor.deleteAll();
        createActor("Alice Plumbing", LifeActorType.CONTRACTOR, "phone", "07700900001", false);
        createActor("Bob Electric", LifeActorType.CONTRACTOR, "email", "bob@electric.co.uk", false);
        createActor("Dr Carol Smith", LifeActorType.DOCTOR, "email", "carol@nhs.uk", false);
        createActor("Erased Corp", LifeActorType.INSTITUTION, "email", "erased@corp.com", true);
    }

    @Test
    void search_noFilters_returnsPagedResponse() {
        given().contentType(ContentType.JSON)
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("items", hasSize(4))
                .body("page", equalTo(0))
                .body("size", equalTo(20))
                .body("totalCount", equalTo(4));
    }

    @Test
    void search_byName_caseInsensitive() {
        given().contentType(ContentType.JSON)
                .queryParam("name", "alice")
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("items", hasSize(1))
                .body("items[0].name", equalTo("Alice Plumbing"));
    }

    @Test
    void search_byActorType() {
        given().contentType(ContentType.JSON)
                .queryParam("actorType", "CONTRACTOR")
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("items", hasSize(2))
                .body("totalCount", equalTo(2));
    }

    @Test
    void search_byContactMethod() {
        given().contentType(ContentType.JSON)
                .queryParam("contactMethod", "email")
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("items", hasSize(3));
    }

    @Test
    void search_erasedOnly() {
        given().contentType(ContentType.JSON)
                .queryParam("erasedOnly", true)
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("items", hasSize(1))
                .body("items[0].name", nullValue());
    }

    @Test
    void search_pagination_page0Size2() {
        given().contentType(ContentType.JSON)
                .queryParam("page", 0)
                .queryParam("size", 2)
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("items", hasSize(2))
                .body("totalCount", equalTo(4))
                .body("page", equalTo(0))
                .body("size", equalTo(2));
    }

    @Test
    void search_pagination_page1Size2() {
        given().contentType(ContentType.JSON)
                .queryParam("page", 1)
                .queryParam("size", 2)
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("items", hasSize(2))
                .body("totalCount", equalTo(4));
    }

    @Test
    void search_pagination_beyondLastPage_returnsEmpty() {
        given().contentType(ContentType.JSON)
                .queryParam("page", 10)
                .queryParam("size", 20)
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("items", hasSize(0))
                .body("totalCount", equalTo(4));
    }

    @Test
    void search_combinedFilters() {
        given().contentType(ContentType.JSON)
                .queryParam("actorType", "CONTRACTOR")
                .queryParam("contactMethod", "email")
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("items", hasSize(1))
                .body("items[0].name", equalTo("Bob Electric"));
    }

    @Test
    void search_sizeCappedAt100() {
        given().contentType(ContentType.JSON)
                .queryParam("size", 500)
                .when().get("/external-actors")
                .then().statusCode(200)
                .body("size", equalTo(100));
    }

    private void createActor(String name, LifeActorType type, String method, String value, boolean erased) {
        ExternalActor a = new ExternalActor();
        a.name = erased ? null : name;
        a.actorType = type;
        a.contactMethod = method;
        a.contactValue = value;
        if (erased) a.gdprErasedAt = java.time.Instant.now();
        a.persist();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=ExternalActorSearchTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: FAIL — `list()` returns `List` not `PagedResponse`

- [ ] **Step 3: Implement ExternalActorService.search()**

Use `ide_replace_member` to replace the `list` method on `ExternalActorService`:

```java
@Transactional
public PagedResponse<ExternalActorResponse> search(
        String name, LifeActorType actorType, String contactMethod,
        boolean erasedOnly, int page, int size) {
    size = Math.min(size, 100);
    var params = new HashMap<String, Object>();
    var conditions = new ArrayList<String>();

    if (name != null && !name.isBlank()) {
        conditions.add("LOWER(name) LIKE LOWER(:name)");
        params.put("name", "%" + name + "%");
    }
    if (actorType != null) {
        conditions.add("actorType = :actorType");
        params.put("actorType", actorType);
    }
    if (contactMethod != null && !contactMethod.isBlank()) {
        conditions.add("contactMethod = :contactMethod");
        params.put("contactMethod", contactMethod);
    }
    if (erasedOnly) {
        conditions.add("gdprErasedAt IS NOT NULL");
    }

    String query = conditions.isEmpty() ? "" : String.join(" AND ", conditions);
    var panacheQuery = query.isEmpty()
            ? ExternalActor.<ExternalActor>findAll()
            : ExternalActor.<ExternalActor>find(query, params);

    long total = panacheQuery.count();
    List<ExternalActor> actors = panacheQuery.page(Page.of(page, size)).list();
    List<ExternalActorResponse> items = actors.stream().map(this::toResponse).toList();
    return new PagedResponse<>(items, page, size, total);
}
```

Add imports: `io.casehub.life.api.response.PagedResponse`, `io.quarkus.panache.common.Page`, `java.util.ArrayList`, `java.util.HashMap`.

- [ ] **Step 4: Update ExternalActorResource.list() to delegate to search()**

Use `ide_replace_member` to replace the `list` method:

```java
@GET
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
public PagedResponse<ExternalActorResponse> list(
        @QueryParam("actorType") final LifeActorType actorType,
        @QueryParam("name") final String name,
        @QueryParam("contactMethod") final String contactMethod,
        @QueryParam("erasedOnly") @DefaultValue("false") final boolean erasedOnly,
        @QueryParam("page") @DefaultValue("0") final int page,
        @QueryParam("size") @DefaultValue("20") final int size) {
    return service.search(name, actorType, contactMethod, erasedOnly, page, size);
}
```

Add import: `jakarta.ws.rs.DefaultValue`.

Also update `@RolesAllowed` on `get()` method to add `HouseholdGroups.JUNIOR`:

```java
@GET
@Path("/{id}")
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=ExternalActorSearchTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS

- [ ] **Step 6: Run existing ExternalActorResourceTest to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=ExternalActorResourceTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS (may need adjustment if old test expects `List` instead of `PagedResponse`)

Fix any broken assertions — the list endpoint now returns `PagedResponse` wrapper instead of raw `List`.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/service/ExternalActorService.java app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java app/src/test/java/io/casehub/life/app/ExternalActorSearchTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#62): ExternalActor search, filter, and pagination"
```

---

### Task 3: Trust history and activity timeline

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/ExternalActorHistoryService.java`
- Modify: `app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java` — add trust-history and activity endpoints
- Create: `app/src/test/java/io/casehub/life/app/ExternalActorHistoryTest.java`

**Interfaces:**
- Consumes: `PagedResponse<T>`, `TrustHistoryEntry`, `ActorActivityEntry` from Task 1
- Produces: `ExternalActorHistoryService.trustHistory(UUID actorId, int page, int size)`, `ExternalActorHistoryService.activityTimeline(UUID actorId, int page, int size)`

- [ ] **Step 1: Write failing integration test for trust history**

```java
// app/src/test/java/io/casehub/life/app/ExternalActorHistoryTest.java
package io.casehub.life.app;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.life.api.LifeActorType;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.entity.ExternalActor;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.api.WorkItemStatus;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class ExternalActorHistoryTest {

    private static final UUID ACTOR_ID = UUID.fromString("aaaaaaaa-0000-0000-0000-000000000001");

    @Inject
    @io.quarkus.hibernate.orm.PersistenceUnit("qhorus")
    EntityManager qhorusEm;

    @BeforeEach
    @Transactional
    void seed() {
        ExternalActor.deleteAll();
        WorkItem.deleteAll();
        LifeTaskContext.deleteAll();

        ExternalActor actor = new ExternalActor();
        actor.id = ACTOR_ID;
        actor.name = "Test Contractor";
        actor.actorType = LifeActorType.CONTRACTOR;
        actor.contactMethod = "phone";
        actor.contactValue = "07700900001";
        actor.persist();

        LifeTestFixtures.seedStandardTemplates();
    }

    @Test
    void trustHistory_returnsAttestations() {
        seedAttestation(ACTOR_ID, "household-management", "deadline-reliability", 0.85, AttestationVerdict.SOUND);
        seedAttestation(ACTOR_ID, "contractor-coordination", null, null, AttestationVerdict.FLAGGED);

        given().contentType(ContentType.JSON)
                .when().get("/external-actors/" + ACTOR_ID + "/trust-history")
                .then().statusCode(200)
                .body("items", hasSize(2))
                .body("items[0].capabilityTag", notNullValue())
                .body("items[0].verdict", notNullValue());
    }

    @Test
    void trustHistory_unknownActor_returns404() {
        given().contentType(ContentType.JSON)
                .when().get("/external-actors/" + UUID.randomUUID() + "/trust-history")
                .then().statusCode(404);
    }

    @Test
    void trustHistory_paginated() {
        for (int i = 0; i < 5; i++) {
            seedAttestation(ACTOR_ID, "household-management", "deadline-reliability", 0.8 + i * 0.02, AttestationVerdict.SOUND);
        }

        given().contentType(ContentType.JSON)
                .queryParam("page", 0).queryParam("size", 2)
                .when().get("/external-actors/" + ACTOR_ID + "/trust-history")
                .then().statusCode(200)
                .body("items", hasSize(2))
                .body("totalCount", equalTo(5));
    }

    @Test
    void activity_returnsWorkItemsForActor() {
        UUID wiId = seedWorkItemWithContext(ACTOR_ID, "Fix boiler", LifeDomain.HOUSEHOLD, WorkItemStatus.COMPLETED);

        given().contentType(ContentType.JSON)
                .when().get("/external-actors/" + ACTOR_ID + "/activity")
                .then().statusCode(200)
                .body("items", hasSize(1))
                .body("items[0].workItemId", equalTo(wiId.toString()))
                .body("items[0].title", equalTo("Fix boiler"))
                .body("items[0].domain", equalTo("HOUSEHOLD"));
    }

    @Test
    void activity_unknownActor_returns404() {
        given().contentType(ContentType.JSON)
                .when().get("/external-actors/" + UUID.randomUUID() + "/activity")
                .then().statusCode(404);
    }

    @Test
    void activity_noWorkItems_returnsEmptyList() {
        given().contentType(ContentType.JSON)
                .when().get("/external-actors/" + ACTOR_ID + "/activity")
                .then().statusCode(200)
                .body("items", hasSize(0));
    }

    @Transactional
    void seedAttestation(UUID subjectId, String capabilityTag, String dimension, Double score, AttestationVerdict verdict) {
        LedgerAttestation a = new LedgerAttestation();
        a.subjectId = subjectId;
        a.attestorId = "life-system";
        a.attestorType = ActorType.AGENT;
        a.capabilityTag = capabilityTag;
        a.trustDimension = dimension;
        a.dimensionScore = score;
        a.verdict = verdict;
        a.confidence = 1.0;
        a.ledgerEntryId = UUID.randomUUID();
        qhorusEm.persist(a);
    }

    @Transactional
    UUID seedWorkItemWithContext(UUID actorId, String title, LifeDomain domain, WorkItemStatus status) {
        WorkItem wi = new WorkItem();
        wi.title = title;
        wi.status = status;
        wi.scope = "casehubio/life/" + domain.descriptor().templateCategory();
        wi.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        if (status == WorkItemStatus.COMPLETED) wi.completedAt = Instant.now();
        wi.persist();

        LifeTaskContext ctx = new LifeTaskContext();
        ctx.workItemId = wi.id;
        ctx.domain = domain;
        ctx.externalActorId = actorId;
        ctx.persist();

        return wi.id;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=ExternalActorHistoryTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: FAIL — endpoints don't exist

- [ ] **Step 3: Implement ExternalActorHistoryService**

```java
// app/src/main/java/io/casehub/life/app/service/ExternalActorHistoryService.java
package io.casehub.life.app.service;

import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.response.ActorActivityEntry;
import io.casehub.life.api.response.PagedResponse;
import io.casehub.life.api.response.TrustHistoryEntry;
import io.casehub.life.app.entity.ExternalActor;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.TypedQuery;
import jakarta.transaction.Transactional;

import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class ExternalActorHistoryService {

    @Inject
    @io.quarkus.hibernate.orm.PersistenceUnit("qhorus")
    EntityManager qhorusEm;

    @Transactional
    public PagedResponse<TrustHistoryEntry> trustHistory(UUID actorId, int page, int size) {
        size = Math.min(size, 100);

        TypedQuery<Long> countQuery = qhorusEm.createQuery(
                "SELECT COUNT(a) FROM LedgerAttestation a WHERE a.subjectId = :subjectId",
                Long.class);
        countQuery.setParameter("subjectId", actorId);
        long total = countQuery.getSingleResult();

        TypedQuery<LedgerAttestation> query = qhorusEm.createQuery(
                "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :subjectId ORDER BY a.occurredAt ASC",
                LedgerAttestation.class);
        query.setParameter("subjectId", actorId);
        query.setFirstResult(page * size);
        query.setMaxResults(size);

        List<TrustHistoryEntry> items = query.getResultList().stream()
                .map(a -> new TrustHistoryEntry(
                        a.occurredAt, a.capabilityTag, a.trustDimension,
                        a.dimensionScore, a.verdict.name()))
                .toList();

        return new PagedResponse<>(items, page, size, total);
    }

    @Transactional
    public PagedResponse<ActorActivityEntry> activityTimeline(UUID actorId, int page, int size) {
        size = Math.min(size, 100);

        long total = LifeTaskContext.count("externalActorId", actorId);

        List<LifeTaskContext> contexts = LifeTaskContext.<LifeTaskContext>find(
                        "externalActorId", actorId)
                .page(io.quarkus.panache.common.Page.of(page, size))
                .list();

        List<ActorActivityEntry> items = contexts.stream()
                .map(ctx -> {
                    WorkItem wi = WorkItem.findById(ctx.workItemId);
                    if (wi == null) return null;
                    return new ActorActivityEntry(
                            wi.id, wi.title, ctx.domain,
                            wi.status != null ? wi.status.name() : null,
                            wi.scope, wi.createdAt, wi.completedAt, wi.outcome);
                })
                .filter(java.util.Objects::nonNull)
                .toList();

        return new PagedResponse<>(items, page, size, total);
    }

    public boolean actorExists(UUID id) {
        return ExternalActor.findByIdOptional(id).isPresent();
    }
}
```

- [ ] **Step 4: Add trust-history and activity endpoints to ExternalActorResource**

Use `ide_insert_member` to add two methods after the existing `listTasks` method:

```java
@GET
@Path("/{id}/trust-history")
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
public Response trustHistory(
        @PathParam("id") final UUID id,
        @QueryParam("page") @DefaultValue("0") final int page,
        @QueryParam("size") @DefaultValue("20") final int size) {
    if (!historyService.actorExists(id)) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
    return Response.ok(historyService.trustHistory(id, page, size)).build();
}

@GET
@Path("/{id}/activity")
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
public Response activityTimeline(
        @PathParam("id") final UUID id,
        @QueryParam("page") @DefaultValue("0") final int page,
        @QueryParam("size") @DefaultValue("20") final int size) {
    if (!historyService.actorExists(id)) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
    return Response.ok(historyService.activityTimeline(id, page, size)).build();
}
```

Add field injection to ExternalActorResource:
```java
@Inject
ExternalActorHistoryService historyService;
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=ExternalActorHistoryTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/service/ExternalActorHistoryService.java app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java app/src/test/java/io/casehub/life/app/ExternalActorHistoryTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#62): trust history and activity timeline endpoints"
```

---

### Task 4: Pending actions

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/PendingActionsService.java`
- Create: `app/src/main/java/io/casehub/life/app/resource/PendingActionsResource.java`
- Create: `app/src/test/java/io/casehub/life/app/PendingActionsTest.java`

**Interfaces:**
- Consumes: `PagedResponse<PendingActionResponse>`, `Urgency`, `LifeDomain.fromCategory()` from api/
- Produces: `PendingActionsService.findPendingActions(domain, candidateGroup, dueSoonHours, page, size)`

- [ ] **Step 1: Write failing integration test**

```java
// app/src/test/java/io/casehub/life/app/PendingActionsTest.java
package io.casehub.life.app;

import io.casehub.life.api.LifeDomain;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.runtime.model.WorkItem;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class PendingActionsTest {

    @BeforeEach
    @Transactional
    void seed() {
        LifeTaskContext.deleteAll();
        WorkItem.deleteAll();
        LifeTestFixtures.seedStandardTemplates();
    }

    @Test
    void pendingActions_returnsActiveLifeWorkItems() {
        seedWorkItem("Approve quote", WorkItemStatus.PENDING, "casehubio/life/household",
                "household-admin", Instant.now().plus(2, ChronoUnit.DAYS), LifeDomain.HOUSEHOLD);

        given().contentType(ContentType.JSON)
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items", hasSize(1))
                .body("items[0].title", equalTo("Approve quote"))
                .body("items[0].urgency", equalTo("NORMAL"));
    }

    @Test
    void pendingActions_excludesSuspended() {
        seedWorkItem("Suspended task", WorkItemStatus.SUSPENDED, "casehubio/life/household",
                "household-admin", Instant.now().plus(1, ChronoUnit.DAYS), LifeDomain.HOUSEHOLD);

        given().contentType(ContentType.JSON)
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items", hasSize(0));
    }

    @Test
    void pendingActions_excludesCompleted() {
        seedWorkItem("Done task", WorkItemStatus.COMPLETED, "casehubio/life/household",
                "household-admin", null, LifeDomain.HOUSEHOLD);

        given().contentType(ContentType.JSON)
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items", hasSize(0));
    }

    @Test
    void pendingActions_excludesNonLifeScope() {
        seedWorkItem("Other scope", WorkItemStatus.PENDING, "casehubio/other/thing",
                "admin", null, null);

        given().contentType(ContentType.JSON)
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items", hasSize(0));
    }

    @Test
    void pendingActions_overdueClassification() {
        seedWorkItem("Overdue task", WorkItemStatus.PENDING, "casehubio/life/health",
                "household-member", Instant.now().minus(3, ChronoUnit.DAYS), LifeDomain.HEALTH);

        given().contentType(ContentType.JSON)
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items[0].urgency", equalTo("OVERDUE"))
                .body("items[0].daysOverdue", greaterThanOrEqualTo(2));
    }

    @Test
    void pendingActions_dueSoonClassification() {
        seedWorkItem("Soon task", WorkItemStatus.PENDING, "casehubio/life/finance",
                "household-admin", Instant.now().plus(6, ChronoUnit.HOURS), LifeDomain.FINANCE);

        given().contentType(ContentType.JSON)
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items[0].urgency", equalTo("DUE_SOON"));
    }

    @Test
    void pendingActions_filterByDomain() {
        seedWorkItem("Health task", WorkItemStatus.PENDING, "casehubio/life/health",
                "household-member", null, LifeDomain.HEALTH);
        seedWorkItem("Finance task", WorkItemStatus.PENDING, "casehubio/life/finance",
                "household-admin", null, LifeDomain.FINANCE);

        given().contentType(ContentType.JSON)
                .queryParam("domain", "HEALTH")
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items", hasSize(1))
                .body("items[0].title", equalTo("Health task"));
    }

    @Test
    void pendingActions_filterByCandidateGroup() {
        seedWorkItem("Admin task", WorkItemStatus.PENDING, "casehubio/life/household",
                "household-admin", null, LifeDomain.HOUSEHOLD);
        seedWorkItem("Member task", WorkItemStatus.PENDING, "casehubio/life/household",
                "household-member", null, LifeDomain.HOUSEHOLD);

        given().contentType(ContentType.JSON)
                .queryParam("candidateGroup", "household-admin")
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items", hasSize(1))
                .body("items[0].title", equalTo("Admin task"));
    }

    @Test
    void pendingActions_customDueSoonHours() {
        seedWorkItem("Task", WorkItemStatus.PENDING, "casehubio/life/household",
                "household-admin", Instant.now().plus(6, ChronoUnit.HOURS), LifeDomain.HOUSEHOLD);

        given().contentType(ContentType.JSON)
                .queryParam("dueSoonHours", 4)
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items[0].urgency", equalTo("NORMAL"));

        given().contentType(ContentType.JSON)
                .queryParam("dueSoonHours", 8)
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items[0].urgency", equalTo("DUE_SOON"));
    }

    @Test
    void pendingActions_sortOrder_overdueFirst() {
        seedWorkItem("Overdue", WorkItemStatus.PENDING, "casehubio/life/household",
                "household-admin", Instant.now().minus(1, ChronoUnit.DAYS), LifeDomain.HOUSEHOLD);
        seedWorkItem("Normal", WorkItemStatus.PENDING, "casehubio/life/household",
                "household-admin", Instant.now().plus(3, ChronoUnit.DAYS), LifeDomain.HOUSEHOLD);

        given().contentType(ContentType.JSON)
                .when().get("/pending-actions")
                .then().statusCode(200)
                .body("items[0].title", equalTo("Overdue"))
                .body("items[1].title", equalTo("Normal"));
    }

    @Transactional
    void seedWorkItem(String title, WorkItemStatus status, String scope,
                      String candidateGroups, Instant expiresAt, LifeDomain domain) {
        WorkItem wi = new WorkItem();
        wi.title = title;
        wi.status = status;
        wi.scope = scope;
        wi.candidateGroups = candidateGroups;
        wi.expiresAt = expiresAt;
        wi.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        wi.persist();

        if (domain != null) {
            LifeTaskContext ctx = new LifeTaskContext();
            ctx.workItemId = wi.id;
            ctx.domain = domain;
            ctx.persist();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=PendingActionsTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: FAIL — endpoint not found

- [ ] **Step 3: Implement PendingActionsService**

```java
// app/src/main/java/io/casehub/life/app/service/PendingActionsService.java
package io.casehub.life.app.service;

import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.Urgency;
import io.casehub.life.api.response.PagedResponse;
import io.casehub.life.api.response.PendingActionResponse;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.work.runtime.model.WorkItem;
import io.quarkus.panache.common.Page;
import io.quarkus.panache.common.Sort;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class PendingActionsService {

    private static final String LIFE_SCOPE_PREFIX = "casehubio/life/";
    private static final List<String> ACTIONABLE_STATUSES = List.of(
            "PENDING", "ASSIGNED", "IN_PROGRESS", "DELEGATED");

    @Transactional
    public PagedResponse<PendingActionResponse> findPendingActions(
            LifeDomain domain, String candidateGroup, int dueSoonHours, int page, int size) {
        size = Math.min(size, 100);

        var params = new HashMap<String, Object>();
        var conditions = new ArrayList<String>();
        conditions.add("scope LIKE :scopePrefix");
        params.put("scopePrefix", LIFE_SCOPE_PREFIX + "%");
        conditions.add("status IN :statuses");
        params.put("statuses", ACTIONABLE_STATUSES.stream()
                .map(io.casehub.work.api.WorkItemStatus::valueOf).toList());

        if (domain != null) {
            conditions.add("scope LIKE :domainScope");
            params.put("domainScope", LIFE_SCOPE_PREFIX + domain.descriptor().templateCategory() + "%");
        }
        if (candidateGroup != null && !candidateGroup.isBlank()) {
            conditions.add("candidateGroups LIKE :candidateGroup");
            params.put("candidateGroup", "%" + candidateGroup + "%");
        }

        String query = String.join(" AND ", conditions);
        long total = WorkItem.count(query, params);
        List<WorkItem> items = WorkItem.<WorkItem>find(query, params)
                .page(Page.of(page, size)).list();

        Instant now = Instant.now();
        List<PendingActionResponse> responses = items.stream()
                .map(wi -> toPendingAction(wi, now, dueSoonHours))
                .sorted(urgencyComparator())
                .toList();

        return new PagedResponse<>(responses, page, size, total);
    }

    private PendingActionResponse toPendingAction(WorkItem wi, Instant now, int dueSoonHours) {
        LifeDomain domain = resolveDomain(wi);
        Urgency urgency = Urgency.classify(wi.expiresAt, now, dueSoonHours);
        Long daysOverdue = Urgency.daysOverdue(wi.expiresAt, now);

        return new PendingActionResponse(
                wi.id, wi.title, wi.description,
                wi.status != null ? wi.status.name() : null,
                domain, wi.candidateGroups,
                wi.createdAt, wi.expiresAt, urgency, daysOverdue);
    }

    private LifeDomain resolveDomain(WorkItem wi) {
        return LifeTaskContext.<LifeTaskContext>findByIdOptional(wi.id)
                .map(ctx -> ctx.domain)
                .orElseGet(() -> domainFromScope(wi.scope));
    }

    private LifeDomain domainFromScope(String scope) {
        if (scope == null || !scope.startsWith(LIFE_SCOPE_PREFIX)) return null;
        String segment = scope.substring(LIFE_SCOPE_PREFIX.length());
        int slash = segment.indexOf('/');
        if (slash > 0) segment = segment.substring(0, slash);
        return LifeDomain.fromCategory(segment).orElse(null);
    }

    private static Comparator<PendingActionResponse> urgencyComparator() {
        return Comparator.comparing(PendingActionResponse::urgency)
                .thenComparing(r -> r.expiresAt() != null ? r.expiresAt() : Instant.MAX)
                .thenComparing(r -> r.createdAt() != null ? r.createdAt() : Instant.MAX);
    }
}
```

- [ ] **Step 4: Implement PendingActionsResource**

```java
// app/src/main/java/io/casehub/life/app/resource/PendingActionsResource.java
package io.casehub.life.app.resource;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.response.PagedResponse;
import io.casehub.life.api.response.PendingActionResponse;
import io.casehub.life.app.service.PendingActionsService;
import io.smallrye.common.annotation.Blocking;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.DefaultValue;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;

@Blocking
@ApplicationScoped
@Path("/pending-actions")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class PendingActionsResource {

    @Inject
    PendingActionsService service;

    @GET
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
    public PagedResponse<PendingActionResponse> list(
            @QueryParam("domain") final LifeDomain domain,
            @QueryParam("candidateGroup") final String candidateGroup,
            @QueryParam("dueSoonHours") @DefaultValue("24") final int dueSoonHours,
            @QueryParam("page") @DefaultValue("0") final int page,
            @QueryParam("size") @DefaultValue("20") final int size) {
        return service.findPendingActions(domain, candidateGroup, dueSoonHours, page, size);
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=PendingActionsTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/service/PendingActionsService.java app/src/main/java/io/casehub/life/app/resource/PendingActionsResource.java app/src/test/java/io/casehub/life/app/PendingActionsTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#63): pending actions API — surface what needs attention"
```

---

### Task 5: Case outcome analytics

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/LifeAnalyticsService.java`
- Create: `app/src/main/java/io/casehub/life/app/resource/LifeAnalyticsResource.java`
- Create: `app/src/test/java/io/casehub/life/app/LifeAnalyticsTest.java`

**Interfaces:**
- Consumes: `CaseStatisticsResponse`, `SlaComplianceResponse`, `TrustAnalyticsResponse` from Task 1. `LifeActorIds.of()` from api/. `LifeCaseTracker` entity. `ActorTrustScore` via qhorus EntityManager.
- Produces: `LifeAnalyticsService.caseStatistics(caseType)`, `LifeAnalyticsService.slaCompliance(domain)`, `LifeAnalyticsService.trustAnalytics()`

- [ ] **Step 1: Write failing integration test**

```java
// app/src/test/java/io/casehub/life/app/LifeAnalyticsTest.java
package io.casehub.life.app;

import io.casehub.ledger.api.model.ScoreType;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.life.api.LifeActorType;
import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.api.LifeActorIds;
import io.casehub.life.app.entity.ExternalActor;
import io.casehub.life.app.entity.LifeCaseTracker;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.runtime.model.WorkItem;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class LifeAnalyticsTest {

    @Inject
    @io.quarkus.hibernate.orm.PersistenceUnit("qhorus")
    EntityManager qhorusEm;

    @BeforeEach
    @Transactional
    void seed() {
        LifeCaseTracker.deleteAll();
        WorkItem.deleteAll();
        ExternalActor.deleteAll();
        LifeTestFixtures.seedStandardTemplates();
    }

    @Test
    void caseStatistics_groupsByCaseType() {
        seedTracker("travel-plan", LifeCaseStatus.COMPLETED, 48);
        seedTracker("travel-plan", LifeCaseStatus.COMPLETED, 72);
        seedTracker("travel-plan", LifeCaseStatus.ACTIVE, null);
        seedTracker("home-maintenance", LifeCaseStatus.COMPLETED, 24);

        given().contentType(ContentType.JSON)
                .when().get("/analytics/cases")
                .then().statusCode(200)
                .body("entries", hasSize(2))
                .body("entries.find { it.caseType == 'travel-plan' }.total", equalTo(3))
                .body("entries.find { it.caseType == 'travel-plan' }.completed", equalTo(2))
                .body("entries.find { it.caseType == 'travel-plan' }.active", equalTo(1));
    }

    @Test
    void caseStatistics_filterByCaseType() {
        seedTracker("travel-plan", LifeCaseStatus.COMPLETED, 48);
        seedTracker("home-maintenance", LifeCaseStatus.COMPLETED, 24);

        given().contentType(ContentType.JSON)
                .queryParam("caseType", "travel-plan")
                .when().get("/analytics/cases")
                .then().statusCode(200)
                .body("entries", hasSize(1))
                .body("entries[0].caseType", equalTo("travel-plan"));
    }

    @Test
    void caseStatistics_emptyResult() {
        given().contentType(ContentType.JSON)
                .when().get("/analytics/cases")
                .then().statusCode(200)
                .body("entries", hasSize(0));
    }

    @Test
    void slaCompliance_computesBreach() {
        Instant now = Instant.now();
        seedWorkItemWithSla("On time", "casehubio/life/household",
                WorkItemStatus.COMPLETED, now.minus(2, ChronoUnit.DAYS),
                now.plus(1, ChronoUnit.DAYS), now);
        seedWorkItemWithSla("Breached", "casehubio/life/household",
                WorkItemStatus.COMPLETED, now.minus(5, ChronoUnit.DAYS),
                now.minus(3, ChronoUnit.DAYS), now.minus(1, ChronoUnit.DAYS));

        given().contentType(ContentType.JSON)
                .when().get("/analytics/sla")
                .then().statusCode(200)
                .body("entries", hasSize(1))
                .body("entries[0].totalWithSla", equalTo(2))
                .body("entries[0].breachedCount", equalTo(1));
    }

    @Test
    void slaCompliance_filterByDomain() {
        Instant now = Instant.now();
        seedWorkItemWithSla("Health", "casehubio/life/health",
                WorkItemStatus.COMPLETED, now.minus(1, ChronoUnit.DAYS),
                now.plus(1, ChronoUnit.DAYS), now);
        seedWorkItemWithSla("Finance", "casehubio/life/finance",
                WorkItemStatus.COMPLETED, now.minus(1, ChronoUnit.DAYS),
                now.plus(1, ChronoUnit.DAYS), now);

        given().contentType(ContentType.JSON)
                .queryParam("domain", "HEALTH")
                .when().get("/analytics/sla")
                .then().statusCode(200)
                .body("entries", hasSize(1))
                .body("entries[0].domain", equalTo("health"));
    }

    @Test
    void trustAnalytics_aggregatesScores() {
        UUID actorId = seedActor("Contractor A");
        seedGlobalTrustScore(LifeActorIds.of(actorId), 0.85);

        given().contentType(ContentType.JSON)
                .when().get("/analytics/trust")
                .then().statusCode(200)
                .body("actorCount", equalTo(1))
                .body("avgGlobalScore", closeTo(0.85, 0.01));
    }

    @Test
    void trustAnalytics_emptyWhenNoActors() {
        given().contentType(ContentType.JSON)
                .when().get("/analytics/trust")
                .then().statusCode(200)
                .body("actorCount", equalTo(0))
                .body("avgGlobalScore", nullValue());
    }

    @Transactional
    void seedTracker(String caseType, LifeCaseStatus status, Integer resolutionHours) {
        LifeCaseTracker t = new LifeCaseTracker();
        t.caseType = caseType;
        t.status = status;
        t.engineCaseId = UUID.randomUUID();
        if (status == LifeCaseStatus.COMPLETED && resolutionHours != null) {
            t.createdAt = Instant.now().minus(resolutionHours, ChronoUnit.HOURS);
            t.completedAt = Instant.now();
        }
        t.persist();
    }

    @Transactional
    void seedWorkItemWithSla(String title, String scope, WorkItemStatus status,
                             Instant createdAt, Instant expiresAt, Instant completedAt) {
        WorkItem wi = new WorkItem();
        wi.title = title;
        wi.scope = scope;
        wi.status = status;
        wi.createdAt = createdAt;
        wi.expiresAt = expiresAt;
        wi.completedAt = completedAt;
        wi.tenancyId = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
        wi.persist();
    }

    @Transactional
    UUID seedActor(String name) {
        ExternalActor a = new ExternalActor();
        a.name = name;
        a.actorType = LifeActorType.CONTRACTOR;
        a.contactMethod = "phone";
        a.contactValue = "07700900001";
        a.persist();
        return a.id;
    }

    @Transactional
    void seedGlobalTrustScore(String actorId, double score) {
        ActorTrustScore s = new ActorTrustScore();
        s.id = UUID.randomUUID();
        s.actorId = actorId;
        s.scoreType = ScoreType.GLOBAL;
        s.trustScore = score;
        s.globalTrustScore = score;
        s.decisionCount = 5;
        s.lastComputedAt = Instant.now();
        qhorusEm.persist(s);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeAnalyticsTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: FAIL — endpoints don't exist

- [ ] **Step 3: Implement LifeAnalyticsService**

```java
// app/src/main/java/io/casehub/life/app/service/LifeAnalyticsService.java
package io.casehub.life.app.service;

import io.casehub.ledger.api.model.ScoreType;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.life.api.LifeActorIds;
import io.casehub.life.api.LifeCaseStatus;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.response.CaseStatisticsResponse;
import io.casehub.life.api.response.CaseStatisticsResponse.CaseTypeStats;
import io.casehub.life.api.response.SlaComplianceResponse;
import io.casehub.life.api.response.SlaComplianceResponse.DomainSlaStats;
import io.casehub.life.api.response.TrustAnalyticsResponse;
import io.casehub.life.api.response.TrustAnalyticsResponse.ActorScoreSummary;
import io.casehub.life.app.entity.ExternalActor;
import io.casehub.life.app.entity.LifeCaseTracker;
import io.casehub.work.api.WorkItemStatus;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
public class LifeAnalyticsService {

    private static final String LIFE_SCOPE_PREFIX = "casehubio/life/";

    @Inject
    @io.quarkus.hibernate.orm.PersistenceUnit("qhorus")
    EntityManager qhorusEm;

    @Transactional
    public CaseStatisticsResponse caseStatistics(String caseType) {
        List<LifeCaseTracker> trackers = caseType != null
                ? LifeCaseTracker.list("caseType", caseType)
                : LifeCaseTracker.listAll();

        Map<String, List<LifeCaseTracker>> grouped = trackers.stream()
                .collect(Collectors.groupingBy(t -> t.caseType, LinkedHashMap::new, Collectors.toList()));

        List<CaseTypeStats> entries = grouped.entrySet().stream()
                .map(e -> buildCaseTypeStats(e.getKey(), e.getValue()))
                .toList();

        return new CaseStatisticsResponse(entries);
    }

    private CaseTypeStats buildCaseTypeStats(String caseType, List<LifeCaseTracker> trackers) {
        long total = trackers.size();
        long active = trackers.stream().filter(t -> t.status == LifeCaseStatus.ACTIVE).count();
        long completed = trackers.stream().filter(t -> t.status == LifeCaseStatus.COMPLETED).count();
        long failed = trackers.stream().filter(t -> t.status == LifeCaseStatus.FAILED).count();

        List<Double> resolutionHours = trackers.stream()
                .filter(t -> t.status == LifeCaseStatus.COMPLETED && t.completedAt != null && t.createdAt != null)
                .map(t -> Duration.between(t.createdAt, t.completedAt).toMillis() / 3_600_000.0)
                .sorted()
                .toList();

        Double avg = resolutionHours.isEmpty() ? null
                : resolutionHours.stream().mapToDouble(Double::doubleValue).average().orElse(0);
        Double p50 = percentile(resolutionHours, 0.50);
        Double p95 = percentile(resolutionHours, 0.95);

        long terminal = completed + failed;
        Double completionRate = terminal > 0 ? (double) completed / terminal : null;

        return new CaseTypeStats(caseType, total, active, completed, failed, avg, p50, p95, completionRate);
    }

    private static Double percentile(List<Double> sorted, double p) {
        if (sorted.isEmpty()) return null;
        double index = p * (sorted.size() - 1);
        int lower = (int) Math.floor(index);
        int upper = (int) Math.ceil(index);
        if (lower == upper) return sorted.get(lower);
        double weight = index - lower;
        return sorted.get(lower) * (1 - weight) + sorted.get(upper) * weight;
    }

    @Transactional
    public SlaComplianceResponse slaCompliance(LifeDomain domain) {
        var params = new java.util.HashMap<String, Object>();
        var conditions = new ArrayList<String>();
        conditions.add("scope LIKE :prefix");
        params.put("prefix", LIFE_SCOPE_PREFIX + "%");
        conditions.add("expiresAt IS NOT NULL");

        if (domain != null) {
            conditions.add("scope LIKE :domainScope");
            params.put("domainScope", LIFE_SCOPE_PREFIX + domain.descriptor().templateCategory() + "%");
        }

        String query = String.join(" AND ", conditions);
        List<WorkItem> items = WorkItem.list(query, params);

        Instant now = Instant.now();
        Map<String, List<WorkItem>> byDomain = items.stream()
                .collect(Collectors.groupingBy(
                        wi -> extractDomainSegment(wi.scope),
                        LinkedHashMap::new, Collectors.toList()));

        List<DomainSlaStats> entries = byDomain.entrySet().stream()
                .map(e -> buildDomainSlaStats(e.getKey(), e.getValue(), now))
                .toList();

        return new SlaComplianceResponse(entries);
    }

    private String extractDomainSegment(String scope) {
        if (scope == null || !scope.startsWith(LIFE_SCOPE_PREFIX)) return "unknown";
        String remainder = scope.substring(LIFE_SCOPE_PREFIX.length());
        int slash = remainder.indexOf('/');
        return slash > 0 ? remainder.substring(0, slash) : remainder;
    }

    private DomainSlaStats buildDomainSlaStats(String domain, List<WorkItem> items, Instant now) {
        long total = items.size();
        long breached = items.stream().filter(wi -> isSlaBreached(wi, now)).count();
        Double complianceRate = total > 0 ? (double) (total - breached) / total : null;

        List<Double> breachLatencies = items.stream()
                .filter(wi -> wi.completedAt != null && wi.expiresAt != null
                        && wi.completedAt.isAfter(wi.expiresAt))
                .map(wi -> Duration.between(wi.expiresAt, wi.completedAt).toMillis() / 3_600_000.0)
                .toList();

        Double avgBreachLatency = breachLatencies.isEmpty() ? null
                : breachLatencies.stream().mapToDouble(Double::doubleValue).average().orElse(0);

        return new DomainSlaStats(domain, total, breached, complianceRate, avgBreachLatency);
    }

    private boolean isSlaBreached(WorkItem wi, Instant now) {
        if (wi.status == WorkItemStatus.ESCALATED || wi.status == WorkItemStatus.EXPIRED) return true;
        if (wi.completedAt != null && wi.expiresAt != null && wi.completedAt.isAfter(wi.expiresAt)) return true;
        if (wi.expiresAt != null && wi.expiresAt.isBefore(now) && wi.status != null && wi.status.isActive()) return true;
        return false;
    }

    @Transactional
    public TrustAnalyticsResponse trustAnalytics() {
        List<ExternalActor> actors = ExternalActor.list("gdprErasedAt IS NULL");
        if (actors.isEmpty()) {
            return new TrustAnalyticsResponse(0, null, Map.of(), List.of());
        }

        List<String> actorIds = actors.stream()
                .map(a -> LifeActorIds.of(a.id))
                .toList();

        List<ActorTrustScore> globalScores = qhorusEm.createQuery(
                        "SELECT s FROM ActorTrustScore s WHERE s.actorId IN :actorIds AND s.scoreType = :scoreType AND s.capabilityKey IS NULL AND s.dimensionKey IS NULL",
                        ActorTrustScore.class)
                .setParameter("actorIds", actorIds)
                .setParameter("scoreType", ScoreType.GLOBAL)
                .getResultList();

        List<ActorTrustScore> dimensionScores = qhorusEm.createQuery(
                        "SELECT s FROM ActorTrustScore s WHERE s.actorId IN :actorIds AND s.scoreType = :scoreType AND s.dimensionKey IS NOT NULL",
                        ActorTrustScore.class)
                .setParameter("actorIds", actorIds)
                .setParameter("scoreType", ScoreType.GLOBAL)
                .getResultList();

        Map<String, UUID> actorIdToUuid = actors.stream()
                .collect(Collectors.toMap(a -> LifeActorIds.of(a.id), a -> a.id));
        Map<UUID, String> uuidToName = actors.stream()
                .collect(Collectors.toMap(a -> a.id, a -> a.name != null ? a.name : ""));

        Double avgGlobal = globalScores.isEmpty() ? null
                : globalScores.stream().mapToDouble(s -> s.trustScore).average().orElse(0);

        Map<String, Double> dimAverages = dimensionScores.stream()
                .collect(Collectors.groupingBy(
                        s -> s.dimensionKey,
                        Collectors.averagingDouble(s -> s.trustScore)));

        List<ActorScoreSummary> lowest = globalScores.stream()
                .sorted(Comparator.comparingDouble(s -> s.trustScore))
                .limit(5)
                .map(s -> {
                    UUID uuid = actorIdToUuid.get(s.actorId);
                    return new ActorScoreSummary(uuid, uuidToName.getOrDefault(uuid, ""), s.trustScore);
                })
                .toList();

        return new TrustAnalyticsResponse(globalScores.size(), avgGlobal, dimAverages, lowest);
    }
}
```

- [ ] **Step 4: Implement LifeAnalyticsResource**

```java
// app/src/main/java/io/casehub/life/app/resource/LifeAnalyticsResource.java
package io.casehub.life.app.resource;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.response.CaseStatisticsResponse;
import io.casehub.life.api.response.SlaComplianceResponse;
import io.casehub.life.api.response.TrustAnalyticsResponse;
import io.casehub.life.app.service.LifeAnalyticsService;
import io.smallrye.common.annotation.Blocking;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;

@Blocking
@ApplicationScoped
@Path("/analytics")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class LifeAnalyticsResource {

    @Inject
    LifeAnalyticsService service;

    @GET
    @Path("/cases")
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
    public CaseStatisticsResponse caseStatistics(@QueryParam("caseType") final String caseType) {
        return service.caseStatistics(caseType);
    }

    @GET
    @Path("/sla")
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
    public SlaComplianceResponse slaCompliance(@QueryParam("domain") final LifeDomain domain) {
        return service.slaCompliance(domain);
    }

    @GET
    @Path("/trust")
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
    public TrustAnalyticsResponse trustAnalytics() {
        return service.trustAnalytics();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=LifeAnalyticsTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: PASS

- [ ] **Step 6: Run full test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/src/main/java/io/casehub/life/app/service/LifeAnalyticsService.java app/src/main/java/io/casehub/life/app/resource/LifeAnalyticsResource.java app/src/test/java/io/casehub/life/app/LifeAnalyticsTest.java
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#64): case outcome analytics — cases, SLA compliance, trust aggregates"
```
