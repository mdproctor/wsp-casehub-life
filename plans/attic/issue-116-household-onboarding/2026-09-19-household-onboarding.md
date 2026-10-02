# Household Onboarding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #116 — Household onboarding — family setup, role assignment, template customisation
**Issue group:** #116

**Goal:** A new user can set up a household in under 5 minutes via a wizard UI, backed by real Keycloak OIDC in dev mode.

**Architecture:** Keycloak Dev Services provides OIDC in dev mode. A realm-config.json defines roles and client config. New `Household` and `HouseholdMember` entities (Household.id = tenancyId). `KeycloakAdminService` wraps the Admin REST API for user CRUD. Frontend: onboarding wizard (5 steps) and settings view (4 tabs).

**Tech Stack:** Java 21 / Quarkus 3.32.2, Keycloak Admin Client, Lit 3.x, H2 (test/demo), PostgreSQL (prod)

## Global Constraints

- Flyway migrations start at V113 (V112 is latest)
- API response records in `api/src/.../response/` as Java records
- REST resources: `@Blocking @ApplicationScoped`, class-level `@Produces`/`@Consumes`
- Roles: `HouseholdGroups.ADMIN`, `HouseholdGroups.MEMBER`, `HouseholdGroups.JUNIOR`
- Tenancy scoping: all queries filter by `CurrentPrincipal.tenancyId()`
- Test tenancy ID: `278776f9-e1b0-46fb-9032-8bddebdcf9ce`
- `@TestSecurity` + `FixedCurrentPrincipal` for tests (not Keycloak container)
- IntelliJ MCP for all code navigation and structural editing

---

## Batch 1: Backend Foundation — entities, migration, Keycloak config

### Task 1: Household and HouseholdMember entities + migration

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/MemberRelationship.java`
- Create: `app/src/main/java/io/casehub/life/app/entity/Household.java`
- Create: `app/src/main/java/io/casehub/life/app/entity/HouseholdMember.java`
- Create: `app/src/main/resources/db/life/migration/V113__household_and_members.sql`
- Test: `app/src/test/java/io/casehub/life/app/entity/HouseholdEntityTest.java`

**Interfaces:**
- Produces: `Household` entity with `findByTenancyId(UUID)`, `HouseholdMember` entity with `findByHouseholdId(UUID)`, `findByKeycloakUserId(String)`, `MemberRelationship` enum

- [ ] **Step 1: Create MemberRelationship enum in api/**

```java
package io.casehub.life.api;

public enum MemberRelationship {
    PARENT, CHILD, SPOUSE, GUARDIAN, OTHER
}
```

Use `ide_create_file` to create at `api/src/main/java/io/casehub/life/api/MemberRelationship.java`.

- [ ] **Step 2: Create Household entity**

```java
package io.casehub.life.app.entity;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

@Entity
@Table(name = "household")
public class Household extends PanacheEntityBase {
    @Id
    public UUID id;
    @Column(nullable = false)
    public String name;
    @Column(nullable = false, length = 64)
    public String timezone;
    @Column(length = 10)
    public String jurisdiction;
    @Column(name = "created_at", nullable = false, updatable = false)
    public Instant createdAt;

    @PrePersist
    void onPersist() {
        if (id == null) id = UUID.randomUUID();
        if (createdAt == null) createdAt = Instant.now();
        if (timezone == null) timezone = "UTC";
    }

    public static Optional<Household> findByTenancyId(UUID tenancyId) {
        return findByIdOptional(tenancyId);
    }
}
```

Use `ide_create_file`.

- [ ] **Step 3: Create HouseholdMember entity**

```java
package io.casehub.life.app.entity;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.MemberRelationship;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

@Entity
@Table(name = "household_member")
public class HouseholdMember extends PanacheEntityBase {
    @Id
    public UUID id;
    @Column(name = "household_id", nullable = false)
    public UUID householdId;
    @Column(name = "keycloak_user_id", nullable = false)
    public String keycloakUserId;
    @Column(nullable = false)
    public String name;
    public String email;
    @Column(nullable = false, length = 32)
    public String role;
    @Enumerated(EnumType.STRING)
    @Column(length = 32)
    public MemberRelationship relationship;
    @Column(name = "related_to")
    public UUID relatedTo;
    @Column(name = "notification_channel", length = 32)
    public String notificationChannel;
    @Column(name = "notification_value")
    public String notificationValue;
    @Column(name = "joined_at", nullable = false, updatable = false)
    public Instant joinedAt;
    @Column(name = "deactivated_at")
    public Instant deactivatedAt;

    @PrePersist
    void onPersist() {
        if (id == null) id = UUID.randomUUID();
        if (joinedAt == null) joinedAt = Instant.now();
        if (!Set.of(HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR).contains(role)) {
            throw new IllegalArgumentException("Invalid role: " + role);
        }
    }

    public static List<HouseholdMember> findByHouseholdId(UUID householdId) {
        return list("householdId = ?1 AND deactivatedAt IS NULL", householdId);
    }

    public static Optional<HouseholdMember> findByKeycloakUserId(String keycloakUserId) {
        return find("keycloakUserId", keycloakUserId).firstResultOptional();
    }

    public boolean isActive() {
        return deactivatedAt == null;
    }
}
```

Use `ide_create_file`.

- [ ] **Step 4: Create Flyway migration V113**

```sql
CREATE TABLE household (
    id            UUID PRIMARY KEY,
    name          VARCHAR(255) NOT NULL,
    timezone      VARCHAR(64) NOT NULL DEFAULT 'UTC',
    jurisdiction  VARCHAR(10),
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE household_member (
    id                   UUID PRIMARY KEY,
    household_id         UUID NOT NULL REFERENCES household(id),
    keycloak_user_id     VARCHAR(255) NOT NULL,
    name                 VARCHAR(255) NOT NULL,
    email                VARCHAR(255),
    role                 VARCHAR(32) NOT NULL,
    relationship         VARCHAR(32),
    related_to           UUID REFERENCES household_member(id),
    notification_channel VARCHAR(32),
    notification_value   VARCHAR(255),
    joined_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deactivated_at       TIMESTAMP
);

CREATE INDEX idx_household_member_household ON household_member(household_id);
```

Write to `app/src/main/resources/db/life/migration/V113__household_and_members.sql`.

- [ ] **Step 5: Write entity test**

```java
@QuarkusTest
@TestSecurity(user = "admin", roles = {"household-admin"})
class HouseholdEntityTest {
    @Inject FixedCurrentPrincipal fixedPrincipal;

    @BeforeEach @Transactional void setup() {
        fixedPrincipal.setGroups(Set.of(HouseholdGroups.ADMIN));
        HouseholdMember.deleteAll();
        Household.deleteAll();
    }

    @Test @Transactional void createHousehold_persistsWithDefaults() {
        Household h = new Household();
        h.id = UUID.fromString("278776f9-e1b0-46fb-9032-8bddebdcf9ce");
        h.name = "Test Family";
        h.persist();

        assertThat(h.createdAt).isNotNull();
        assertThat(h.timezone).isEqualTo("UTC");
        assertThat(Household.findByTenancyId(h.id)).isPresent();
    }

    @Test @Transactional void createMember_validatesRole() {
        // seed household first
        Household h = new Household();
        h.id = UUID.fromString("278776f9-e1b0-46fb-9032-8bddebdcf9ce");
        h.name = "Test";
        h.persist();

        HouseholdMember m = new HouseholdMember();
        m.householdId = h.id;
        m.keycloakUserId = "kc-001";
        m.name = "Mark";
        m.role = "invalid-role";
        assertThatThrownBy(m::persist).hasMessageContaining("Invalid role");
    }

    @Test @Transactional void createMember_validRole_persists() {
        Household h = new Household();
        h.id = UUID.fromString("278776f9-e1b0-46fb-9032-8bddebdcf9ce");
        h.name = "Test";
        h.persist();

        HouseholdMember m = new HouseholdMember();
        m.householdId = h.id;
        m.keycloakUserId = "kc-001";
        m.name = "Mark";
        m.role = HouseholdGroups.ADMIN;
        m.relationship = MemberRelationship.PARENT;
        m.persist();

        List<HouseholdMember> members = HouseholdMember.findByHouseholdId(h.id);
        assertThat(members).hasSize(1);
        assertThat(members.get(0).name).isEqualTo("Mark");
    }
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=HouseholdEntityTest -am --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 3 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C $PROJECT add api/src/main/java/io/casehub/life/api/MemberRelationship.java app/src/main/java/io/casehub/life/app/entity/Household.java app/src/main/java/io/casehub/life/app/entity/HouseholdMember.java app/src/main/resources/db/life/migration/V113__household_and_members.sql app/src/test/java/io/casehub/life/app/entity/HouseholdEntityTest.java
git -C $PROJECT commit -m "feat(#116): Household + HouseholdMember entities and V113 migration Refs #116"
```

---

### Task 2: API response/request records + VoiceEnrollmentService SPI stub

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/response/HouseholdResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/response/HouseholdMemberResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/request/CreateHouseholdRequest.java`
- Create: `api/src/main/java/io/casehub/life/api/request/CreateMemberRequest.java`
- Create: `api/src/main/java/io/casehub/life/api/response/OnboardingStatusResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/response/HouseholdCapabilitiesResponse.java`
- Create: `api/src/main/java/io/casehub/life/api/spi/VoiceEnrollmentService.java`
- Create: `app/src/main/java/io/casehub/life/app/spi/NoOpVoiceEnrollmentService.java`

**Interfaces:**
- Consumes: `Household`, `HouseholdMember`, `MemberRelationship` from Task 1
- Produces: All request/response records, `VoiceEnrollmentService` SPI

- [ ] **Step 1: Create request records**

```java
// CreateHouseholdRequest.java
package io.casehub.life.api.request;
public record CreateHouseholdRequest(String name, String timezone, String jurisdiction) {}

// CreateMemberRequest.java
package io.casehub.life.api.request;
import io.casehub.life.api.MemberRelationship;
public record CreateMemberRequest(
    String name, String email, String role,
    MemberRelationship relationship, java.util.UUID relatedTo,
    String notificationChannel, String notificationValue
) {}
```

Use `ide_create_file` for each.

- [ ] **Step 2: Create response records**

```java
// HouseholdResponse.java
package io.casehub.life.api.response;
import java.time.Instant;
import java.util.UUID;
public record HouseholdResponse(UUID id, String name, String timezone, String jurisdiction, Instant createdAt) {}

// HouseholdMemberResponse.java
package io.casehub.life.api.response;
import io.casehub.life.api.MemberRelationship;
import java.time.Instant;
import java.util.UUID;
public record HouseholdMemberResponse(
    UUID id, String name, String email, String role,
    MemberRelationship relationship, UUID relatedTo,
    String notificationChannel, String notificationValue,
    Instant joinedAt, boolean active
) {}

// OnboardingStatusResponse.java
package io.casehub.life.api.response;
public record OnboardingStatusResponse(boolean needsOnboarding) {}

// HouseholdCapabilitiesResponse.java
package io.casehub.life.api.response;
public record HouseholdCapabilitiesResponse(boolean voiceEnrollment) {}
```

Use `ide_create_file` for each.

- [ ] **Step 3: Create VoiceEnrollmentService SPI + NoOp impl**

```java
// api/ — VoiceEnrollmentService.java
package io.casehub.life.api.spi;
import java.util.Optional;
import java.util.UUID;
public interface VoiceEnrollmentService {
    void enrollVoice(UUID memberId, byte[] voiceSample);
    boolean isEnrolled(UUID memberId);
    Optional<UUID> identifyByVoice(byte[] voiceSample);
}

// app/ — NoOpVoiceEnrollmentService.java
package io.casehub.life.app.spi;
import io.casehub.life.api.spi.VoiceEnrollmentService;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Optional;
import java.util.UUID;

@DefaultBean
@ApplicationScoped
public class NoOpVoiceEnrollmentService implements VoiceEnrollmentService {
    @Override public void enrollVoice(UUID memberId, byte[] voiceSample) {}
    @Override public boolean isEnrolled(UUID memberId) { return false; }
    @Override public Optional<UUID> identifyByVoice(byte[] voiceSample) { return Optional.empty(); }
}
```

- [ ] **Step 4: Install api module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api --batch-mode -q`

- [ ] **Step 5: Compile to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode -q`
Expected: clean compile

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add api/src app/src/main/java/io/casehub/life/app/spi/NoOpVoiceEnrollmentService.java
git -C $PROJECT commit -m "feat(#116): API records + VoiceEnrollmentService SPI stub Refs #116"
```

---

### Task 3: Keycloak realm config + application.properties changes

**Files:**
- Create: `app/src/main/resources/realm-config.json`
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/pom.xml` (add keycloak-admin-client dependency)

**Interfaces:**
- Produces: Working Keycloak Dev Services in dev profile, `quarkus-keycloak-admin-client` on classpath

- [ ] **Step 1: Add keycloak-admin-client dependency to pom.xml**

Add to `app/pom.xml` `<dependencies>` section:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-keycloak-admin-client</artifactId>
</dependency>
```

Use `ide_replace_text_in_file` to insert after an existing dependency.

- [ ] **Step 2: Create realm-config.json**

Write to `app/src/main/resources/realm-config.json`:

```json
{
  "realm": "casehub-life",
  "enabled": true,
  "registrationAllowed": false,
  "roles": {
    "realm": [
      { "name": "household-admin", "description": "Full household authority" },
      { "name": "household-member", "description": "Standard household member" },
      { "name": "household-junior", "description": "Restricted access" }
    ]
  },
  "users": [
    {
      "username": "setup-admin",
      "enabled": true,
      "credentials": [{ "type": "password", "value": "setup", "temporary": true }],
      "realmRoles": ["household-admin"]
    }
  ],
  "clients": [
    {
      "clientId": "casehub-life",
      "enabled": true,
      "publicClient": true,
      "directAccessGrantsEnabled": true,
      "redirectUris": ["*"],
      "webOrigins": ["*"],
      "protocolMappers": [
        {
          "name": "tenancyId",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-attribute-mapper",
          "config": {
            "user.attribute": "tenancyId",
            "claim.name": "tenancyId",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "jsonType.label": "String"
          }
        }
      ]
    }
  ]
}
```

- [ ] **Step 3: Update application.properties — dev profile**

Replace the dev profile OIDC lines. Use `ide_replace_text_in_file`:

Find:
```
%dev.quarkus.oidc.enabled=false
%dev.quarkus.keycloak.devservices.enabled=false
```

Replace with:
```
%dev.quarkus.keycloak.devservices.realm-path=realm-config.json
```

- [ ] **Step 4: Compile to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app --batch-mode -q`
Expected: clean compile (keycloak-admin-client resolves)

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add app/pom.xml app/src/main/resources/realm-config.json app/src/main/resources/application.properties
git -C $PROJECT commit -m "feat(#116): Keycloak realm config + dev services enabled + admin client dep Refs #116"
```

---

## Batch 2: Onboarding Flow — service, resource, UI

### Task 4: KeycloakAdminService

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/KeycloakAdminService.java`
- Test: `app/src/test/java/io/casehub/life/app/service/KeycloakAdminServiceTest.java`

**Interfaces:**
- Consumes: `quarkus-keycloak-admin-client` from Task 3
- Produces: `createUser(name, email, password, role, tenancyId) → String keycloakUserId`, `updateUserRole(keycloakUserId, newRole)`, `deactivateUser(keycloakUserId)`

- [ ] **Step 1: Write failing test**

```java
@QuarkusTest
@TestSecurity(user = "admin", roles = {"household-admin"})
class KeycloakAdminServiceTest {
    @Inject KeycloakAdminService keycloakAdminService;

    @Test
    void serviceIsInjectable() {
        assertThat(keycloakAdminService).isNotNull();
    }
}
```

- [ ] **Step 2: Run test — expect FAIL (class not found)**

- [ ] **Step 3: Implement KeycloakAdminService**

```java
package io.casehub.life.app.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.keycloak.admin.client.Keycloak;
import org.keycloak.representations.idm.CredentialRepresentation;
import org.keycloak.representations.idm.UserRepresentation;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class KeycloakAdminService {

    private static final String REALM = "casehub-life";

    @Inject Keycloak keycloak;

    public String createUser(String name, String email, String password,
                             String role, String tenancyId) {
        UserRepresentation user = new UserRepresentation();
        user.setUsername(email);
        user.setEmail(email);
        user.setFirstName(name);
        user.setEnabled(true);
        user.setAttributes(Map.of("tenancyId", List.of(tenancyId)));

        CredentialRepresentation cred = new CredentialRepresentation();
        cred.setType(CredentialRepresentation.PASSWORD);
        cred.setValue(password);
        cred.setTemporary(false);
        user.setCredentials(List.of(cred));

        var response = keycloak.realm(REALM).users().create(user);
        String userId = extractUserId(response);

        keycloak.realm(REALM).users().get(userId)
                .roles().realmLevel()
                .add(List.of(keycloak.realm(REALM).roles().get(role).toRepresentation()));

        return userId;
    }

    public void updateUserRole(String keycloakUserId, String oldRole, String newRole) {
        var userResource = keycloak.realm(REALM).users().get(keycloakUserId);
        userResource.roles().realmLevel()
                .remove(List.of(keycloak.realm(REALM).roles().get(oldRole).toRepresentation()));
        userResource.roles().realmLevel()
                .add(List.of(keycloak.realm(REALM).roles().get(newRole).toRepresentation()));
    }

    public void deactivateUser(String keycloakUserId) {
        var userResource = keycloak.realm(REALM).users().get(keycloakUserId);
        UserRepresentation user = userResource.toRepresentation();
        user.setEnabled(false);
        userResource.update(user);
    }

    private String extractUserId(jakarta.ws.rs.core.Response response) {
        String location = response.getHeaderString("Location");
        return location.substring(location.lastIndexOf('/') + 1);
    }
}
```

- [ ] **Step 4: Run test — expect PASS**

Note: this test requires Keycloak Dev Services (Docker). If CI doesn't have Docker, mark test with `@Tag("keycloak")` and skip in CI.

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add app/src/main/java/io/casehub/life/app/service/KeycloakAdminService.java app/src/test/java/io/casehub/life/app/service/KeycloakAdminServiceTest.java
git -C $PROJECT commit -m "feat(#116): KeycloakAdminService — Keycloak Admin REST API wrapper Refs #116"
```

---

### Task 5: HouseholdService + OnboardingResource

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/service/HouseholdService.java`
- Create: `app/src/main/java/io/casehub/life/app/resource/OnboardingResource.java`
- Test: `app/src/test/java/io/casehub/life/app/resource/OnboardingResourceTest.java`

**Interfaces:**
- Consumes: `Household`, `HouseholdMember` (Task 1), API records (Task 2), `KeycloakAdminService` (Task 4)
- Produces: `HouseholdService` (createHousehold, addMember, removeMember, getOnboardingStatus), `OnboardingResource` (status, create, members, templates)

- [ ] **Step 1: Write failing test for onboarding status**

```java
@QuarkusTest
@TestSecurity(user = "admin", roles = {"household-admin"})
class OnboardingResourceTest {
    @Inject FixedCurrentPrincipal fixedPrincipal;

    @BeforeEach @Transactional void setup() {
        fixedPrincipal.setGroups(Set.of(HouseholdGroups.ADMIN));
        fixedPrincipal.setTenancyId("278776f9-e1b0-46fb-9032-8bddebdcf9ce");
        HouseholdMember.deleteAll();
        Household.deleteAll();
    }

    @Test
    void status_noHousehold_needsOnboarding() {
        given()
            .when().get("/onboarding/status")
            .then().statusCode(200)
            .body("needsOnboarding", equalTo(true));
    }

    @Test @Transactional
    void status_withHousehold_doesNotNeedOnboarding() {
        Household h = new Household();
        h.id = UUID.fromString("278776f9-e1b0-46fb-9032-8bddebdcf9ce");
        h.name = "Test";
        h.persist();

        given()
            .when().get("/onboarding/status")
            .then().statusCode(200)
            .body("needsOnboarding", equalTo(false));
    }

    @Test
    void createHousehold_returnsHouseholdResponse() {
        given()
            .contentType("application/json")
            .body("""
                {"name":"The Proctors","timezone":"Europe/London","jurisdiction":"GB"}
            """)
            .when().post("/onboarding/household")
            .then().statusCode(201)
            .body("name", equalTo("The Proctors"))
            .body("timezone", equalTo("Europe/London"));
    }
}
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Implement HouseholdService**

```java
package io.casehub.life.app.service;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.response.*;
import io.casehub.life.api.request.*;
import io.casehub.life.api.spi.VoiceEnrollmentService;
import io.casehub.life.app.entity.Household;
import io.casehub.life.app.entity.HouseholdMember;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class HouseholdService {

    @Inject CurrentPrincipal currentPrincipal;
    @Inject VoiceEnrollmentService voiceEnrollmentService;

    public OnboardingStatusResponse getOnboardingStatus() {
        String tenancyId = currentPrincipal.tenancyId();
        boolean exists = Household.findByTenancyId(UUID.fromString(tenancyId)).isPresent();
        return new OnboardingStatusResponse(!exists);
    }

    @Transactional
    public HouseholdResponse createHousehold(CreateHouseholdRequest req) {
        Household h = new Household();
        h.id = UUID.fromString(currentPrincipal.tenancyId());
        h.name = req.name();
        h.timezone = req.timezone();
        h.jurisdiction = req.jurisdiction();
        h.persist();
        return toResponse(h);
    }

    @Transactional
    public HouseholdMemberResponse addMember(CreateMemberRequest req) {
        UUID householdId = UUID.fromString(currentPrincipal.tenancyId());
        HouseholdMember m = new HouseholdMember();
        m.householdId = householdId;
        m.keycloakUserId = "pending-" + UUID.randomUUID();
        m.name = req.name();
        m.email = req.email();
        m.role = req.role();
        m.relationship = req.relationship();
        m.relatedTo = req.relatedTo();
        m.notificationChannel = req.notificationChannel();
        m.notificationValue = req.notificationValue();
        m.persist();
        return toMemberResponse(m);
    }

    public List<HouseholdMemberResponse> listMembers() {
        UUID householdId = UUID.fromString(currentPrincipal.tenancyId());
        return HouseholdMember.findByHouseholdId(householdId).stream()
                .map(this::toMemberResponse).toList();
    }

    public HouseholdCapabilitiesResponse getCapabilities() {
        return new HouseholdCapabilitiesResponse(
                !(voiceEnrollmentService instanceof io.casehub.life.app.spi.NoOpVoiceEnrollmentService));
    }

    private HouseholdResponse toResponse(Household h) {
        return new HouseholdResponse(h.id, h.name, h.timezone, h.jurisdiction, h.createdAt);
    }

    private HouseholdMemberResponse toMemberResponse(HouseholdMember m) {
        return new HouseholdMemberResponse(
                m.id, m.name, m.email, m.role,
                m.relationship, m.relatedTo,
                m.notificationChannel, m.notificationValue,
                m.joinedAt, m.isActive());
    }
}
```

- [ ] **Step 4: Implement OnboardingResource**

```java
package io.casehub.life.app.resource;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.request.CreateHouseholdRequest;
import io.casehub.life.api.request.CreateMemberRequest;
import io.casehub.life.api.response.*;
import io.casehub.life.app.service.HouseholdService;
import io.smallrye.common.annotation.Blocking;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.List;

@Blocking
@ApplicationScoped
@Path("/onboarding")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class OnboardingResource {

    @Inject HouseholdService householdService;

    @GET
    @Path("/status")
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
    public OnboardingStatusResponse status() {
        return householdService.getOnboardingStatus();
    }

    @POST
    @Path("/household")
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
    public Response createHousehold(CreateHouseholdRequest req) {
        HouseholdResponse response = householdService.createHousehold(req);
        return Response.status(Response.Status.CREATED).entity(response).build();
    }

    @POST
    @Path("/members")
    @RolesAllowed({HouseholdGroups.ADMIN})
    public Response addMembers(List<CreateMemberRequest> members) {
        List<HouseholdMemberResponse> responses = members.stream()
                .map(householdService::addMember).toList();
        return Response.status(Response.Status.CREATED).entity(responses).build();
    }
}
```

- [ ] **Step 5: Run tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=OnboardingResourceTest -am --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add app/src/main/java/io/casehub/life/app/service/HouseholdService.java app/src/main/java/io/casehub/life/app/resource/OnboardingResource.java app/src/test/java/io/casehub/life/app/resource/OnboardingResourceTest.java
git -C $PROJECT commit -m "feat(#116): HouseholdService + OnboardingResource — onboarding flow backend Refs #116"
```

---

### Task 6: HouseholdResource (settings CRUD)

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/resource/HouseholdResource.java`
- Test: `app/src/test/java/io/casehub/life/app/resource/HouseholdResourceTest.java`

**Interfaces:**
- Consumes: `HouseholdService` (Task 5)
- Produces: `/household` CRUD endpoints, `/household/members` CRUD, `/household/capabilities`

- [ ] **Step 1: Write failing test**

```java
@QuarkusTest
@TestSecurity(user = "admin", roles = {"household-admin"})
class HouseholdResourceTest {
    @Inject FixedCurrentPrincipal fixedPrincipal;

    @BeforeEach @Transactional void setup() {
        fixedPrincipal.setGroups(Set.of(HouseholdGroups.ADMIN));
        fixedPrincipal.setTenancyId("278776f9-e1b0-46fb-9032-8bddebdcf9ce");
        HouseholdMember.deleteAll();
        Household.deleteAll();

        Household h = new Household();
        h.id = UUID.fromString("278776f9-e1b0-46fb-9032-8bddebdcf9ce");
        h.name = "Test Family";
        h.timezone = "Europe/London";
        h.jurisdiction = "GB";
        h.persist();
    }

    @Test void getHousehold_returnsDetails() {
        given().when().get("/household")
            .then().statusCode(200)
            .body("name", equalTo("Test Family"));
    }

    @Test void getMembers_returnsEmptyList() {
        given().when().get("/household/members")
            .then().statusCode(200)
            .body("$", hasSize(0));
    }

    @Test void getCapabilities_voiceIsFalse() {
        given().when().get("/household/capabilities")
            .then().statusCode(200)
            .body("voiceEnrollment", equalTo(false));
    }

    @Test @TestSecurity(user = "junior", roles = {"household-junior"})
    void juniorCannotAddMembers() {
        fixedPrincipal.setGroups(Set.of(HouseholdGroups.JUNIOR));
        given().contentType("application/json")
            .body("""
                {"name":"Test","email":"test@example.uk","role":"household-member"}
            """)
            .when().post("/household/members")
            .then().statusCode(403);
    }
}
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Implement HouseholdResource**

```java
package io.casehub.life.app.resource;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.request.CreateMemberRequest;
import io.casehub.life.api.response.*;
import io.casehub.life.app.service.HouseholdService;
import io.smallrye.common.annotation.Blocking;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.List;

@Blocking
@ApplicationScoped
@Path("/household")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class HouseholdResource {

    @Inject HouseholdService householdService;

    @GET
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
    public HouseholdResponse getHousehold() {
        return householdService.getHousehold();
    }

    @GET @Path("/members")
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER})
    public List<HouseholdMemberResponse> listMembers() {
        return householdService.listMembers();
    }

    @POST @Path("/members")
    @RolesAllowed({HouseholdGroups.ADMIN})
    public Response addMember(CreateMemberRequest req) {
        HouseholdMemberResponse response = householdService.addMember(req);
        return Response.status(Response.Status.CREATED).entity(response).build();
    }

    @GET @Path("/capabilities")
    @RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
    public HouseholdCapabilitiesResponse capabilities() {
        return householdService.getCapabilities();
    }
}
```

Note: `getHousehold()` needs to be added to HouseholdService:

```java
public HouseholdResponse getHousehold() {
    UUID householdId = UUID.fromString(currentPrincipal.tenancyId());
    return Household.findByTenancyId(householdId)
            .map(this::toResponse)
            .orElseThrow(() -> new NotFoundException("Household not found"));
}
```

Use `ide_insert_member` to add after `getOnboardingStatus()`.

- [ ] **Step 4: Run tests — expect PASS**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=HouseholdResourceTest -am --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add app/src/main/java/io/casehub/life/app/resource/HouseholdResource.java app/src/main/java/io/casehub/life/app/service/HouseholdService.java app/src/test/java/io/casehub/life/app/resource/HouseholdResourceTest.java
git -C $PROJECT commit -m "feat(#116): HouseholdResource — household settings CRUD Refs #116"
```

---

## Batch 3: Frontend — onboarding wizard + settings view

### Task 7: Onboarding wizard view

**Files:**
- Create: `life-ui/src/views/onboarding-view.ts`
- Modify: `life-ui/src/shell/app-shell.ts` (add route, onboarding redirect, settings icon)
- Modify: `life-ui/src/index.ts` (register new components)

**Interfaces:**
- Consumes: `GET /onboarding/status`, `POST /onboarding/household`, `POST /onboarding/members`, `POST /external-actors`, `POST /life-tasks`

- [ ] **Step 1: Create onboarding-view.ts**

A multi-step wizard component with 5 steps. Each step commits to backend before advancing. Uses Lit 3.x with `@state()` for step tracking and form data.

Key structure:
- `_currentStep` state (1-5)
- `_renderStep1()` through `_renderStep5()`
- Each step's "Next" button calls the backend, then advances `_currentStep`
- Progress indicator at top
- Back/Next navigation

Code is substantial (~300 lines) — implement following the pattern in `people-view.ts` (form inputs, fetch calls, CSS using pages tokens).

- [ ] **Step 2: Update app-shell.ts — add routes and onboarding check**

Add to the `View` type: `'onboarding' | 'settings'`

Add onboarding redirect in `connectedCallback()`:
```typescript
fetch('/onboarding/status')
    .then(r => r.json())
    .then(data => {
        if (data.needsOnboarding) this._view = 'onboarding';
    })
    .catch(() => {});
```

Add gear icon in toolbar (before notification badge).

Add route cases for `#onboarding` and `#settings`.

- [ ] **Step 3: Update index.ts — register new components**

```typescript
import './views/onboarding-view.js';
import './views/settings-view.js';
```

- [ ] **Step 4: Build frontend**

Run: `npm run build --prefix life-ui`
Expected: clean build

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add life-ui/src/
git -C $PROJECT commit -m "feat(#116): onboarding wizard view + app shell routing Refs #116"
```

---

### Task 8: Settings view

**Files:**
- Create: `life-ui/src/views/settings-view.ts`

**Interfaces:**
- Consumes: `GET /household`, `PUT /household`, `GET /household/members`, `POST /household/members`, `DELETE /household/members/{id}`, `GET /household/capabilities`

- [ ] **Step 1: Create settings-view.ts**

Tabbed view with 4 tabs: Members, Household, Templates, Voice.

Key structure:
- `_activeTab` state ('members' | 'household' | 'templates' | 'voice')
- Tab bar at top
- Members tab: list + add/edit/deactivate
- Household tab: edit form (name, timezone, jurisdiction)
- Templates tab: toggle list
- Voice tab: checks `/household/capabilities`, shows placeholder or enrollment UI

Follow the tabbed pattern from `people-view.ts` (tab buttons, conditional rendering).

- [ ] **Step 2: Build frontend**

Run: `npm run build --prefix life-ui`
Expected: clean build

- [ ] **Step 3: Commit**

```bash
git -C $PROJECT add life-ui/src/views/settings-view.ts
git -C $PROJECT commit -m "feat(#116): settings view — household management UI Refs #116"
```

---

## Batch 4: Demo Profile + Polish

### Task 9: Demo profile seed data

**Files:**
- Modify: `app/src/main/resources/import-demo.sql`

**Interfaces:**
- Consumes: `Household`, `HouseholdMember` entities (Task 1)

- [ ] **Step 1: Add household + member seeds to import-demo.sql**

Append to `import-demo.sql`:

```sql
-- Household
INSERT INTO household (id, name, timezone, jurisdiction, created_at)
VALUES ('278776f9-e1b0-46fb-9032-8bddebdcf9ce', 'Demo Household', 'Europe/London', 'GB', CURRENT_TIMESTAMP);

-- Household Members
INSERT INTO household_member (id, household_id, keycloak_user_id, name, email, role, relationship, related_to, notification_channel, notification_value, joined_at)
VALUES
  ('a0000001-0000-0000-0000-000000000001', '278776f9-e1b0-46fb-9032-8bddebdcf9ce', 'demo-admin', 'Mark', 'mark@example.uk', 'household-admin', 'PARENT', NULL, 'sms', '+447700000001', CURRENT_TIMESTAMP),
  ('a0000001-0000-0000-0000-000000000002', '278776f9-e1b0-46fb-9032-8bddebdcf9ce', 'demo-member', 'Sarah', 'sarah@example.uk', 'household-member', 'PARENT', NULL, 'email', 'sarah@example.uk', CURRENT_TIMESTAMP),
  ('a0000001-0000-0000-0000-000000000003', '278776f9-e1b0-46fb-9032-8bddebdcf9ce', 'demo-junior-1', 'Ella', 'ella@example.uk', 'household-junior', 'CHILD', 'a0000001-0000-0000-0000-000000000001', NULL, NULL, CURRENT_TIMESTAMP),
  ('a0000001-0000-0000-0000-000000000004', '278776f9-e1b0-46fb-9032-8bddebdcf9ce', 'demo-junior-2', 'Tom', 'tom@example.uk', 'household-junior', 'CHILD', 'a0000001-0000-0000-0000-000000000001', NULL, NULL, CURRENT_TIMESTAMP);
```

- [ ] **Step 2: Run demo mode to verify**

Start Quarkus in demo mode, verify dashboard loads with household data:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app -Dquarkus.profile=demo
```

- [ ] **Step 3: Commit**

```bash
git -C $PROJECT add app/src/main/resources/import-demo.sql
git -C $PROJECT commit -m "feat(#116): demo profile household + member seed data Refs #116"
```

---

### Task 10: Full integration test + CLAUDE.md update

**Files:**
- Test: `app/src/test/java/io/casehub/life/app/OnboardingIntegrationTest.java`
- Modify: `CLAUDE.md` (add household onboarding docs)

**Interfaces:**
- Consumes: All previous tasks

- [ ] **Step 1: Write end-to-end onboarding test**

```java
@QuarkusTest
@TestSecurity(user = "admin", roles = {"household-admin"})
class OnboardingIntegrationTest {
    @Inject FixedCurrentPrincipal fixedPrincipal;

    @BeforeEach @Transactional void setup() {
        fixedPrincipal.setGroups(Set.of(HouseholdGroups.ADMIN));
        fixedPrincipal.setTenancyId("278776f9-e1b0-46fb-9032-8bddebdcf9ce");
        HouseholdMember.deleteAll();
        Household.deleteAll();
    }

    @Test void fullOnboardingFlow() {
        // Step 1: check status
        given().when().get("/onboarding/status")
            .then().statusCode(200).body("needsOnboarding", equalTo(true));

        // Step 2: create household
        given().contentType("application/json")
            .body("""
                {"name":"The Proctors","timezone":"Europe/London","jurisdiction":"GB"}
            """)
            .when().post("/onboarding/household")
            .then().statusCode(201);

        // Step 3: status changes
        given().when().get("/onboarding/status")
            .then().statusCode(200).body("needsOnboarding", equalTo(false));

        // Step 4: add members
        given().contentType("application/json")
            .body("""
                [{"name":"Sarah","email":"sarah@example.uk","role":"household-member",
                  "relationship":"PARENT","notificationChannel":"email","notificationValue":"sarah@example.uk"}]
            """)
            .when().post("/onboarding/members")
            .then().statusCode(201);

        // Step 5: verify via settings
        given().when().get("/household")
            .then().statusCode(200).body("name", equalTo("The Proctors"));

        given().when().get("/household/members")
            .then().statusCode(200).body("$", hasSize(1));

        given().when().get("/household/capabilities")
            .then().statusCode(200).body("voiceEnrollment", equalTo(false));
    }
}
```

- [ ] **Step 2: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am --batch-mode`
Expected: all tests pass (existing + new)

- [ ] **Step 3: Update CLAUDE.md — add onboarding section**

Add under "What This Project Owns → Domain Model":

```
**Onboarding + household management:**
- `Household` — one per tenancy: `{id (= tenancyId), name, timezone, jurisdiction, createdAt}`
- `HouseholdMember` — family members: `{id, householdId, keycloakUserId, name, email, role, relationship, relatedTo, notificationChannel, notificationValue}`
- `MemberRelationship` enum: PARENT, CHILD, SPOUSE, GUARDIAN, OTHER
- `VoiceEnrollmentService` SPI — `@DefaultBean NoOpVoiceEnrollmentService`; #120 provides real impl
- Keycloak Dev Services in dev profile (`realm-config.json` — roles, bootstrap admin, tenancyId claim)
- `OnboardingResource` — `/onboarding/status`, `/onboarding/household`, `/onboarding/members`
- `HouseholdResource` — `/household`, `/household/members`, `/household/capabilities`
```

- [ ] **Step 4: Commit**

```bash
git -C $PROJECT add app/src/test/java/io/casehub/life/app/OnboardingIntegrationTest.java CLAUDE.md
git -C $PROJECT commit -m "feat(#116): integration test + CLAUDE.md update Closes #116"
```

---

## References

- `specs/issue-116-household-onboarding/2026-09-19-household-onboarding-design.md` — design spec
- `specs/issue-116-household-onboarding/decisions.md` — D1–D5 design decisions
- `app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java` — CRUD pattern reference
- `api/src/main/java/io/casehub/life/api/response/ExternalActorResponse.java` — response record pattern
- `api/src/main/java/io/casehub/life/api/HouseholdGroups.java` — role constants
- `app/src/main/resources/import-demo.sql` — demo seed pattern
- `app/src/main/resources/application.properties:163-197` — existing OIDC config
- Quarkus Keycloak Dev Services — https://quarkus.io/guides/security-openid-connect-dev-services
- casehubio/life#120 — voice intake (blocked by this)
- casehubio/life#121 — family mind map (blocked by this)
