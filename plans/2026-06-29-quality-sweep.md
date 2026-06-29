# Quality Sweep Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close 5 quality issues (#30, #31, #41, #42, #43) on a single branch — scattered business logic audit fixes, ledger field cleanup, data-level visibility filter, MCP verification, and CDI exclude-types removal.

**Architecture:** All changes are internal to casehub-life. #43 removes CDI config that platform#112 made redundant. #31 fixes ledger entry fields (remove premature, populate available). #30 eliminates 3 remaining scattered business logic violations via enum methods, dedicated columns, and a threshold key map. #41 adds a visibility policy SPI with a junior-restricting implementation. #42 closes with no code changes after verification.

**Tech Stack:** Java 21, Quarkus 3.32.2, H2 (MODE=PostgreSQL), Flyway, JPA/Panache

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api` then `-pl app`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClassName --batch-mode`
- All commits reference issues: `Refs #30` / `Refs #31` / `Refs #41` / `Refs #42` / `Refs #43`
- Flyway: life domain at `db/life/migration/` (V100+); ledger joins at `db/life/ledger/migration/` (V2100+)
- Ledger entities in `io.casehub.life.app.ledger` (qhorus PU), NOT sub-packages of `io.casehub.life.app.entity`
- api/ module: pure Java only — no Quarkus, no JPA, no `casehub-platform-api`
- Test tenancyId: `278776f9-e1b0-46fb-9032-8bddebdcf9ce`
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all class/reference searches, never bash grep/find

---

### Task 1: Remove CurrentPrincipal exclude-types (#43)

**Files:**
- Modify: `app/src/main/resources/application.properties:96-112`
- Modify: `app/src/test/resources/application.properties:43-64`
- Modify: `docs/protocols/casehub-life/current-principal-cdi-exclusion.md`

**Interfaces:**
- Consumes: nothing
- Produces: clean CDI resolution via `@Alternative @Priority` instead of exclude-types

**Prerequisite:** Pull latest SNAPSHOTs to ensure `OidcCurrentPrincipal @Alternative @Priority(100)` and `FixedCurrentPrincipal @Alternative @Priority(200)` are in local Maven repo.

- [ ] **Step 1: Pull latest SNAPSHOTs**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode dependency:resolve -pl app -U
```

- [ ] **Step 2: Verify OidcCurrentPrincipal is @Alternative @Priority(100)**

Use IntelliJ MCP `ide_find_class` for `OidcCurrentPrincipal` with `scope: project_and_libraries`. Read the decompiled class and verify the annotations.

If NOT found or NOT `@Alternative @Priority(100)`: STOP — platform SNAPSHOT not updated. Report and skip this task.

- [ ] **Step 3: Edit production application.properties**

Remove these 4 CurrentPrincipal entries from `quarkus.arc.exclude-types` (lines 97, 99, 100, 101):
- `io.casehub.platform.mock.MockCurrentPrincipal`
- `io.casehub.qhorus.runtime.identity.QhorusInboundCurrentPrincipal`
- `io.casehub.persistence.memory.DefaultTestPrincipal`
- `io.casehub.work.runtime.service.TenantScopedPrincipal`

Keep `MockGroupMembershipProvider` and all OpenClaw exclusions. Update the comment block (lines 87-95) to explain `@Alternative @Priority` resolution instead of exclude-types.

The resulting `quarkus.arc.exclude-types` should be:
```properties
quarkus.arc.exclude-types=\
  io.casehub.platform.mock.MockGroupMembershipProvider,\
  io.casehub.openclaw.casehub.OpenClawCaseChannelProvider,\
  io.casehub.openclaw.casehub.ReactiveOpenClawCaseChannelProvider,\
  io.casehub.openclaw.casehub.OpenClawWorkerProvisioner,\
  io.casehub.openclaw.casehub.ReactiveOpenClawWorkerProvisioner,\
  io.casehub.openclaw.casehub.OpenClawChannelBackend,\
  io.casehub.openclaw.casehub.ChannelContextWindowObserver,\
  io.casehub.openclaw.casehub.OpenClawAgentRegistry,\
  io.casehub.openclaw.casehub.OpenClawCasehubConfig,\
  io.casehub.openclaw.casehub.OpenClawWorkerStatusListener,\
  io.casehub.openclaw.casehub.OversightGateDispatcher,\
  io.casehub.openclaw.casehub.OversightGateService
```

- [ ] **Step 4: Edit test application.properties**

Remove these 2 CurrentPrincipal entries from `quarkus.arc.exclude-types` (lines 52, 53):
- `io.casehub.qhorus.runtime.identity.QhorusInboundCurrentPrincipal`
- `io.casehub.work.runtime.service.TenantScopedPrincipal`

Update the comment block (lines 43-45). Keep connector, scheduler, and OpenClaw exclusions.

- [ ] **Step 5: Retire the protocol**

Replace `docs/protocols/casehub-life/current-principal-cdi-exclusion.md` content with:

```markdown
---
id: PP-20260615-8ed738
title: "RETIRED — CurrentPrincipal CDI disambiguation"
type: rule
scope: repo
status: retired
retired_date: 2026-06-29
retired_reason: "platform#112 shipped @Alternative @Priority resolution"
---

**Retired.** Since platform#112, `OidcCurrentPrincipal @Alternative @Priority(100)` wins in production and `FixedCurrentPrincipal @Alternative @Priority(200)` wins in tests. No exclude-types entries needed for CurrentPrincipal disambiguation.
```

- [ ] **Step 6: Build and run ALL tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app
```

Expected: all 377 tests pass with no `AmbiguousResolutionException`.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/resources/application.properties app/src/test/resources/application.properties docs/protocols/casehub-life/current-principal-cdi-exclusion.md
git commit -m "fix(#43): remove CurrentPrincipal exclude-types — @Alternative @Priority resolves since platform#112"
```

---

### Task 2: Ledger entry field fixes (#31)

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/ledger/HealthDecisionLedgerEntry.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandler.java`
- Modify: `app/src/test/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandlerTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/ledger/LedgerEntryDomainContentBytesTest.java`
- Create: `app/src/main/resources/db/life/ledger/migration/V2104__drop_health_appointment_date.sql`

**Interfaces:**
- Consumes: `casehub.life.jurisdiction` config property (already exists in `LifeAgentDescriptorFactory`)
- Produces: populated `jurisdiction` on all `LegalActionLedgerEntry` writes; `appointmentDate` removed from `HealthDecisionLedgerEntry`

- [ ] **Step 1: Write failing test — LegalDomainLedgerHandler populates jurisdiction**

In `LegalDomainLedgerHandlerTest.java`, add a test that asserts `entry.jurisdiction` equals the configured value (e.g. `"GB"`). The handler currently sets jurisdiction to null — this test should FAIL.

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LegalDomainLedgerHandlerTest --batch-mode
```

Expected: FAIL — `jurisdiction` is null.

- [ ] **Step 3: Inject jurisdiction config into LegalDomainLedgerHandler**

Add `@ConfigProperty(name = "casehub.life.jurisdiction", defaultValue = "GB") String jurisdiction` to `LegalDomainLedgerHandler`. In the `writeEntry()` method (or equivalent), set `entry.jurisdiction = jurisdiction`.

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LegalDomainLedgerHandlerTest --batch-mode
```

Expected: PASS.

- [ ] **Step 5: Remove appointmentDate from HealthDecisionLedgerEntry**

In `HealthDecisionLedgerEntry.java`:
- Remove `public Instant appointmentDate` field and its `@Column(name = "appointment_date")` annotation
- Update `domainContentBytes()` — remove the `appointmentDate != null ? appointmentDate.toString() : ""` segment from the pipe-delimited string

- [ ] **Step 6: Create migration V2104**

Create `app/src/main/resources/db/life/ledger/migration/V2104__drop_health_appointment_date.sql`:
```sql
ALTER TABLE health_decision_ledger_entry DROP COLUMN appointment_date;
```

- [ ] **Step 7: Update LedgerEntryDomainContentBytesTest**

Update the `HealthDecisionLedgerEntry` hash test to match the new pipe segment count (one fewer). The test constructs an entry with all fields populated and asserts the hash bytes — remove the `appointmentDate` field from setup and adjust the expected pipe-delimited string.

- [ ] **Step 8: Build and run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app
```

Expected: all tests pass.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "fix(#31): remove premature appointmentDate, populate jurisdiction from config"
```

---

### Task 3: HouseholdActionType reasonTemplate + threshold keys (#30 Violation 3)

**Files:**
- Modify: `api/src/main/java/io/casehub/life/api/HouseholdActionType.java`
- Create: `app/src/main/java/io/casehub/life/app/routing/HouseholdActionThresholdKeys.java`
- Modify: `app/src/main/java/io/casehub/life/app/routing/LifeActionRiskClassifier.java`
- Modify: `app/src/test/java/io/casehub/life/app/routing/LifeActionRiskClassifierTest.java` (or equivalent)

**Interfaces:**
- Consumes: `LifeRiskPolicyKeys` preference keys (existing)
- Produces: `HouseholdActionType.reasonTemplate()` — `@Nullable String`, `HouseholdActionThresholdKeys.forType()` — `ThresholdKeyPair`

- [ ] **Step 1: Add reasonTemplate() to HouseholdActionType enum**

Add a `@Nullable String reasonTemplate` field and constructor parameter. Each constant gets its template string. `HEALTH_APPOINTMENT_GP` gets `null` (NEVER gated — `buildReason()` unreachable). See spec §4 Violation 3 Part A for the complete template list.

Constructor changes from `(GatePolicy, boolean, List<String>)` to `(GatePolicy, boolean, List<String>, String)`. Add `@Nullable public String reasonTemplate() { return reasonTemplate; }`.

- [ ] **Step 2: Install api/ so app/ can see the new method**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api
```

- [ ] **Step 3: Create HouseholdActionThresholdKeys in app/routing/**

Create `app/src/main/java/io/casehub/life/app/routing/HouseholdActionThresholdKeys.java`:

```java
package io.casehub.life.app.routing;

import io.casehub.life.api.HouseholdActionType;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;

import java.util.Map;
import java.util.Objects;

import static io.casehub.life.api.HouseholdActionType.*;

public final class HouseholdActionThresholdKeys {

    public record ThresholdKeyPair(
            io.casehub.platform.api.preferences.PreferenceKey<DoublePreference> member,
            io.casehub.platform.api.preferences.PreferenceKey<DoublePreference> admin) {

        public double resolve(Preferences prefs, boolean isAdmin) {
            return prefs.get(isAdmin ? admin : member).value();
        }
    }

    private static final Map<HouseholdActionType, ThresholdKeyPair> KEYS = Map.of(
        SPEND_PURCHASE,            new ThresholdKeyPair(LifeRiskPolicyKeys.SPEND_THRESHOLD, LifeRiskPolicyKeys.ADMIN_SPEND_THRESHOLD),
        SPEND_SUBSCRIPTION_MODIFY, new ThresholdKeyPair(LifeRiskPolicyKeys.SPEND_THRESHOLD, LifeRiskPolicyKeys.ADMIN_SPEND_THRESHOLD),
        BOOKING_REFUNDABLE,        new ThresholdKeyPair(LifeRiskPolicyKeys.BOOKING_THRESHOLD, LifeRiskPolicyKeys.ADMIN_BOOKING_THRESHOLD),
        CONTRACTOR_ENGAGE,         new ThresholdKeyPair(LifeRiskPolicyKeys.CONTRACTOR_THRESHOLD, LifeRiskPolicyKeys.ADMIN_CONTRACTOR_THRESHOLD)
    );

    public static ThresholdKeyPair forType(HouseholdActionType type) {
        return Objects.requireNonNull(KEYS.get(type),
            "No threshold keys for non-AMOUNT_THRESHOLD type: " + type);
    }

    private HouseholdActionThresholdKeys() {}
}
```

- [ ] **Step 4: Refactor LifeActionRiskClassifier**

Replace `resolveThreshold()` switch with:
```java
private double resolveThreshold(HouseholdActionType type, Preferences prefs, boolean admin) {
    return HouseholdActionThresholdKeys.forType(type).resolve(prefs, admin);
}
```

Replace `buildReason()` switch with:
```java
private String buildReason(HouseholdActionType type, PlannedAction action) {
    return type.reasonTemplate().formatted(formatAmount(action.parameters()));
}
```

Note: templates without `%s` (like `SPEND_SUBSCRIPTION_CANCEL`) will ignore the format arg — `String.formatted()` silently ignores extra args.

- [ ] **Step 5: Run existing tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeActionRiskClassifierTest --batch-mode
```

Expected: all existing risk classifier tests pass — behavior is unchanged.

- [ ] **Step 6: Full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app
```

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "refactor(#30): move reasonTemplate to HouseholdActionType, threshold keys to descriptor map"
```

---

### Task 4: CommitmentMode escalationTemplate (#30 Violation 2b)

**Files:**
- Modify: `api/src/main/java/io/casehub/life/api/commitment/CommitmentMode.java`
- Modify: `app/src/main/java/io/casehub/life/app/observer/LifeWatchdogAlertObserver.java`
- Modify: `app/src/test/java/io/casehub/life/app/LifeWatchdogAlertObserverTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `CommitmentMode.escalationTemplate()` — `String`

- [ ] **Step 1: Add escalationTemplate() to CommitmentMode**

```java
public enum CommitmentMode {
    DELEGATION("%s has not confirmed — action required"),
    CONTRACTOR("Contractor has not confirmed by deadline"),
    OVERSIGHT("Oversight gate expired — request not approved");

    private final String escalationTemplate;
    CommitmentMode(String escalationTemplate) { this.escalationTemplate = escalationTemplate; }
    public String escalationTemplate() { return escalationTemplate; }
}
```

- [ ] **Step 2: Install api/**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api
```

- [ ] **Step 3: Refactor LifeWatchdogAlertObserver.createEscalationTask()**

Replace the `switch (record.mode)` block with:

```java
private void createEscalationTask(final LifeCommitmentRecord record) {
    final String title = record.mode == CommitmentMode.DELEGATION
        ? record.mode.escalationTemplate().formatted(
            record.delegateTo != null ? record.delegateTo : "Unknown")
        : record.mode.escalationTemplate();
    lifeTaskService.create(new CreateLifeTaskRequest("life-escalation", title, null, null));
}
```

- [ ] **Step 4: Run observer tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeWatchdogAlertObserverTest --batch-mode
```

Expected: PASS — same escalation messages produced.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "refactor(#30): move escalation templates to CommitmentMode enum"
```

---

### Task 5: LifeCommitmentRecord domain + oversightKey columns (#30 Violations 1, 2, 2c)

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/entity/LifeCommitmentRecord.java`
- Modify: `api/src/main/java/io/casehub/life/api/request/OversightGateRequest.java`
- Modify: `app/src/main/java/io/casehub/life/app/commitment/OversightGateStrategy.java`
- Modify: `app/src/main/java/io/casehub/life/app/commitment/DelegationCommitmentStrategy.java`
- Modify: `app/src/main/java/io/casehub/life/app/commitment/ContractorCommitmentStrategy.java`
- Modify: `app/src/main/java/io/casehub/life/app/observer/LifeWatchdogAlertObserver.java`
- Create: `app/src/main/resources/db/life/migration/V108__commitment_record_domain.sql`
- Create: `app/src/main/resources/db/life/migration/V109__commitment_record_oversight_key.sql`
- Modify: tests for strategies and observer

**Interfaces:**
- Consumes: `LifeDomain` enum, `DomainLedgerHandler` CDI instances (existing)
- Produces: `LifeCommitmentRecord.domain` (non-null on new records), `LifeCommitmentRecord.oversightKey` (OVERSIGHT only), `OversightGateRequest.domain()` (required field)

- [ ] **Step 1: Add domain and oversightKey to LifeCommitmentRecord**

Add two fields to the entity:

```java
@Enumerated(EnumType.STRING)
@Column(length = 50)
public LifeDomain domain;

@Column(name = "oversight_key", length = 255)
public String oversightKey;
```

Add import for `io.casehub.life.api.LifeDomain`.

- [ ] **Step 2: Create migrations**

`app/src/main/resources/db/life/migration/V108__commitment_record_domain.sql`:
```sql
ALTER TABLE life_commitment_record ADD COLUMN domain VARCHAR(50);
```

`app/src/main/resources/db/life/migration/V109__commitment_record_oversight_key.sql`:
```sql
ALTER TABLE life_commitment_record ADD COLUMN oversight_key VARCHAR(255);
```

- [ ] **Step 3: Add domain to OversightGateRequest**

```java
public record OversightGateRequest(
        @NotNull LifeDomain domain,
        @NotNull Instant deadline,
        @NotNull @Valid CreateLifeTaskRequest pendingTask,
        @NotNull BigDecimal amountThreshold,
        @NotBlank String purchaseCategory
) {}
```

Add import for `io.casehub.life.api.LifeDomain`. Install api/: `mvn --batch-mode install -pl api`.

- [ ] **Step 4: Write failing tests for domain population**

For each strategy test: assert `record.domain` is set after `execute()`. For OversightGateStrategy: assert `record.oversightKey` is set and `record.delegateTo` is null. These should FAIL before implementation.

- [ ] **Step 5: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=OversightGateStrategyTest,DelegationCommitmentStrategyTest,ContractorCommitmentStrategyTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false
```

- [ ] **Step 6: Update OversightGateStrategy**

1. Set `record.domain = oc.request().domain()` instead of hardcoding
2. Set `record.oversightKey = taskKey` instead of `record.delegateTo = taskKey`
3. Set `record.delegateTo = null`
4. Update dedup query: change `delegateTo` to `oversightKey` field:
   ```java
   final boolean duplicate = LifeCommitmentRecord
       .find("mode = ?1 and status = ?2 and oversightKey = ?3",
           CommitmentMode.OVERSIGHT, CommitmentStatus.PENDING_RESPONSE, taskKey)
       .count() > 0;
   ```
5. Fix ledger handler lookup: replace `h -> h.domain() == LifeDomain.FINANCE` with `h -> h.domain() == oc.request().domain()`

- [ ] **Step 7: Update DelegationCommitmentStrategy and ContractorCommitmentStrategy**

In each strategy, after creating the `LifeCommitmentRecord`, set `record.domain` from the context's `LifeTaskContext.domain`. Both contexts carry a `LifeTaskContext` — find it via `DelegationContext.taskContext()` / `ContractorContext.taskContext()` (or equivalent accessors).

- [ ] **Step 8: Update LifeWatchdogAlertObserver**

1. Fix FINANCE hardcode: replace `h -> h.domain() == LifeDomain.FINANCE` with `h -> h.domain() == record.domain`
2. Fix `delegateTo` guard in DELEGATION escalation: remove the `delegate.contains(":")` check — `delegateTo` is now always a principal ID for DELEGATION records (oversight keys moved to `oversightKey`)

The `createEscalationTask()` method (already refactored in Task 4) should now be:
```java
private void createEscalationTask(final LifeCommitmentRecord record) {
    final String title = record.mode == CommitmentMode.DELEGATION
        ? record.mode.escalationTemplate().formatted(
            record.delegateTo != null ? record.delegateTo : "Unknown")
        : record.mode.escalationTemplate();
    lifeTaskService.create(new CreateLifeTaskRequest("life-escalation", title, null, null));
}
```

No further changes to this method — the `delegateTo` field now correctly holds only principal IDs for DELEGATION.

- [ ] **Step 9: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app
```

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "refactor(#30): add domain + oversightKey to LifeCommitmentRecord, fix FINANCE hardcode and delegateTo overload"
```

---

### Task 6: LifeTaskVisibilityPolicy SPI + junior filter (#41)

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/spi/LifeTaskVisibilityPolicy.java`
- Create: `app/src/main/java/io/casehub/life/app/spi/DefaultLifeTaskVisibilityPolicy.java`
- Create: `app/src/main/java/io/casehub/life/app/spi/JuniorLifeTaskVisibilityPolicy.java`
- Modify: `api/src/main/java/io/casehub/life/api/response/LifeTaskResponse.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/LifeTaskService.java`
- Modify: `app/src/main/java/io/casehub/life/app/resource/LifeTaskResource.java`
- Create: `app/src/test/java/io/casehub/life/app/spi/JuniorLifeTaskVisibilityPolicyTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/LifeTaskResourceTest.java`

**Interfaces:**
- Consumes: `LifeTaskResponse` (existing, extended), `CurrentPrincipal` (existing)
- Produces: `LifeTaskVisibilityPolicy.isVisible(LifeTaskResponse, String, Set<String>)` — `boolean`

- [ ] **Step 1: Write failing unit test for JuniorLifeTaskVisibilityPolicy**

Create `app/src/test/java/io/casehub/life/app/spi/JuniorLifeTaskVisibilityPolicyTest.java`:

```java
package io.casehub.life.app.spi;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.LifeDomain;
import io.casehub.life.api.response.LifeTaskResponse;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class JuniorLifeTaskVisibilityPolicyTest {

    private final JuniorLifeTaskVisibilityPolicy policy = new JuniorLifeTaskVisibilityPolicy();

    @Test
    void adminSeesEverything() {
        var task = taskWithAssignee("other-actor");
        assertTrue(policy.isVisible(task, "admin-actor",
            Set.of(HouseholdGroups.ADMIN)));
    }

    @Test
    void memberSeesEverything() {
        var task = taskWithAssignee("other-actor");
        assertTrue(policy.isVisible(task, "member-actor",
            Set.of(HouseholdGroups.MEMBER)));
    }

    @Test
    void juniorSeesAssignedTask() {
        var task = taskWithAssignee("junior-actor");
        assertTrue(policy.isVisible(task, "junior-actor",
            Set.of(HouseholdGroups.JUNIOR)));
    }

    @Test
    void juniorSeesTaskInCandidatePool() {
        var task = taskWithCandidateGroups(List.of(HouseholdGroups.JUNIOR));
        assertTrue(policy.isVisible(task, "junior-actor",
            Set.of(HouseholdGroups.JUNIOR)));
    }

    @Test
    void juniorCannotSeeOtherTask() {
        var task = taskWithAssignee("admin-actor");
        assertFalse(policy.isVisible(task, "junior-actor",
            Set.of(HouseholdGroups.JUNIOR)));
    }

    @Test
    void juniorCannotSeeTaskWithoutMatchingCandidateGroup() {
        var task = taskWithCandidateGroups(List.of(HouseholdGroups.ADMIN));
        assertFalse(policy.isVisible(task, "junior-actor",
            Set.of(HouseholdGroups.JUNIOR)));
    }

    private LifeTaskResponse taskWithAssignee(String assigneeId) {
        return new LifeTaskResponse(UUID.randomUUID(), "test-template",
            LifeDomain.HOUSEHOLD, "READY", null, Instant.now(),
            null, null, assigneeId, List.of());
    }

    private LifeTaskResponse taskWithCandidateGroups(List<String> groups) {
        return new LifeTaskResponse(UUID.randomUUID(), "test-template",
            LifeDomain.HOUSEHOLD, "READY", null, Instant.now(),
            null, null, null, groups);
    }
}
```

- [ ] **Step 2: Create LifeTaskVisibilityPolicy SPI in api/**

Create `api/src/main/java/io/casehub/life/api/spi/LifeTaskVisibilityPolicy.java`:

```java
package io.casehub.life.api.spi;

import io.casehub.life.api.response.LifeTaskResponse;

import java.util.Set;

public interface LifeTaskVisibilityPolicy {
    boolean isVisible(LifeTaskResponse task, String actorId, Set<String> groups);
}
```

- [ ] **Step 3: Add assigneeId and candidateGroups to LifeTaskResponse**

```java
public record LifeTaskResponse(
        UUID workItemId,
        String templateRef,
        LifeDomain domain,
        String status,
        UUID externalActorId,
        Instant createdAt,
        CommitmentMode commitmentMode,
        CommitmentStatus commitmentStatus,
        String assigneeId,
        List<String> candidateGroups
) {}
```

Add `import java.util.List;`. Install api/: `mvn --batch-mode install -pl api`.

- [ ] **Step 4: Create DefaultLifeTaskVisibilityPolicy**

Create `app/src/main/java/io/casehub/life/app/spi/DefaultLifeTaskVisibilityPolicy.java`:

```java
package io.casehub.life.app.spi;

import io.casehub.life.api.response.LifeTaskResponse;
import io.casehub.life.api.spi.LifeTaskVisibilityPolicy;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Set;

@DefaultBean
@ApplicationScoped
public class DefaultLifeTaskVisibilityPolicy implements LifeTaskVisibilityPolicy {
    @Override
    public boolean isVisible(LifeTaskResponse task, String actorId, Set<String> groups) {
        return true;
    }
}
```

- [ ] **Step 5: Create JuniorLifeTaskVisibilityPolicy**

Create `app/src/main/java/io/casehub/life/app/spi/JuniorLifeTaskVisibilityPolicy.java`:

```java
package io.casehub.life.app.spi;

import io.casehub.life.api.HouseholdGroups;
import io.casehub.life.api.response.LifeTaskResponse;
import io.casehub.life.api.spi.LifeTaskVisibilityPolicy;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import java.util.Set;

@Alternative
@Priority(1)
@ApplicationScoped
public class JuniorLifeTaskVisibilityPolicy implements LifeTaskVisibilityPolicy {
    @Override
    public boolean isVisible(LifeTaskResponse task, String actorId, Set<String> groups) {
        if (!groups.contains(HouseholdGroups.JUNIOR)) return true;
        if (groups.contains(HouseholdGroups.ADMIN) || groups.contains(HouseholdGroups.MEMBER)) return true;
        if (actorId.equals(task.assigneeId())) return true;
        return task.candidateGroups().stream().anyMatch(groups::contains);
    }
}
```

- [ ] **Step 6: Run unit test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=JuniorLifeTaskVisibilityPolicyTest --batch-mode
```

Expected: PASS.

- [ ] **Step 7: Update LifeTaskService.get() to populate new fields**

In `LifeTaskService.get()`, the `WorkItem` is already loaded. Add `assigneeId` and `candidateGroups` to the `LifeTaskResponse` constructor call:

```java
return new LifeTaskResponse(
        workItem.id,
        workItem.callerRef != null ? workItem.callerRef.replace("life:task/", "") : null,
        ctx.domain,
        workItem.status.name(),
        ctx.externalActorId,
        workItem.createdAt,
        mode, status,
        workItem.assigneeId,
        workItem.candidateGroups != null
            ? List.of(workItem.candidateGroups.split(","))
            : List.of()
);
```

Also update the `create()` method's return statement to include `null` and `List.of()` for the two new fields (no assignee or candidates at creation time — the WorkItem was just created).

- [ ] **Step 8: Wire visibility policy into LifeTaskResource**

```java
@Inject LifeTaskVisibilityPolicy visibilityPolicy;
@Inject CurrentPrincipal principal;

@GET
@Path("/{id}")
@RolesAllowed({HouseholdGroups.ADMIN, HouseholdGroups.MEMBER, HouseholdGroups.JUNIOR})
public LifeTaskResponse get(@PathParam("id") final UUID workItemId) {
    final LifeTaskResponse response = service.get(workItemId);
    if (!visibilityPolicy.isVisible(response, principal.actorId(), principal.groups())) {
        throw new WebApplicationException(404);
    }
    return response;
}
```

Add imports for `CurrentPrincipal`, `LifeTaskVisibilityPolicy`, `WebApplicationException`.

- [ ] **Step 9: Write integration test for junior visibility**

In `LifeTaskResourceTest.java`, add a `@QuarkusTest` test:
- Create a task (as admin/member)
- Call `GET /life-tasks/{id}` with a junior principal whose actorId doesn't match the assignee
- Assert 404 response

Use `@TestSecurity(user = "junior-user", roles = "household-junior")` for the junior call.

- [ ] **Step 10: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && \
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app
```

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(#41): add LifeTaskVisibilityPolicy SPI — household-junior sees own tasks only"
```

---

### Task 7: Close #42 + update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (project repo)

**Interfaces:**
- Consumes: nothing
- Produces: updated documentation

- [ ] **Step 1: Close #42 with verification comment**

```bash
gh issue close 42 --repo casehubio/life --comment "Verified: Qhorus is embedded for internal CDI services only. No external MCP clients connect to life. The auto-activated MCP surface (transitive from casehub-qhorus) is benign — exposes tools nobody calls externally. No config changes needed."
```

- [ ] **Step 2: Update CLAUDE.md — CurrentPrincipal disambiguation**

Find the "CurrentPrincipal disambiguation" section in CLAUDE.md. Replace with:

```markdown
**CurrentPrincipal resolution (since platform#112):** `OidcCurrentPrincipal @Alternative @Priority(100)` wins in production; `FixedCurrentPrincipal @Alternative @Priority(200)` wins in tests. No `quarkus.arc.exclude-types` entries needed for CurrentPrincipal — CDI `@Alternative @Priority` handles disambiguation. `DefaultTestPrincipal` is not `@Alternative` and is superseded by both. `TenantScopedPrincipal` and `QhorusInboundCurrentPrincipal` are plain `@Default` beans and are superseded by any `@Alternative`.
```

- [ ] **Step 3: Update CLAUDE.md — #42 MCP verification result**

No new CLAUDE.md section needed — the MCP topology is not a standing convention, just a one-time verification.

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#42,#43): update CLAUDE.md — CurrentPrincipal resolution, close #42 with verification"
```
