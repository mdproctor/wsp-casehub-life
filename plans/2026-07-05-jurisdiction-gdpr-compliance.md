# Jurisdiction, GDPR Compliance & CaseService Refactor — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-action jurisdiction to legal ledger entries, eliminate LifeCaseService switch, extract a dedicated GDPR erasure service integrating LedgerErasureService tokenisation, and surface erasure results to callers.

**Architecture:** Four issues on one branch. #48 adds jurisdiction to the task creation flow with fallback to tenant-wide config. #51 replaces a 6-arm switch with CDI Instance lookup. #49 extracts `LifeGdprErasureService` integrating `LedgerErasureService.erase()` for actor ID tokenisation. #50 wires the resource to return `ErasureResponse`. Implementation order: `#48 → #51 → #49 → #50`.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-ledger 0.2-SNAPSHOT, H2 (test), Flyway

## Global Constraints

- Java source level 21 on Java 26 JVM (`JAVA_HOME=$(/usr/libexec/java_home -v 26)`)
- Build: `mvn --batch-mode install -pl api` then `mvn --batch-mode install -pl app`
- Test single class: `mvn test -pl app -Dtest=ClassName --batch-mode` (requires `api` installed first)
- Flyway: default DS migrations at `db/life/migration/` (V110+), qhorus DS at `db/life/ledger/migration/` (V2106+)
- `@Transactional` on service methods only, never resource methods (PP-20260526-75d9c9)
- REST resources: `@Blocking @ApplicationScoped`, class-level `@Produces/@Consumes(APPLICATION_JSON)` (PP-20260526-d0b921)
- No switch/if-else on LifeDomain, LifeCaseType, or HouseholdActionType in service/observer classes (PP-20260609-bd9d27)
- Ledger subclass entities in `io.casehub.life.app.ledger` package (qhorus PU)
- Every commit references an issue: `Refs #N` or `Closes #N`
- All issue numbers: #48, #51, #49, #50

---

### Task 1: Per-Action Jurisdiction on LegalActionLedgerEntry (#48)

**Files:**
- Modify: `api/src/main/java/io/casehub/life/api/request/CreateLifeTaskRequest.java`
- Modify: `api/src/main/java/io/casehub/life/api/response/LifeTaskContextResponse.java`
- Modify: `app/src/main/java/io/casehub/life/app/entity/LifeTaskContext.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/LifeTaskService.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandler.java`
- Modify: `app/src/main/java/io/casehub/life/app/ledger/LegalActionLedgerEntry.java`
- Create: `app/src/main/resources/db/life/migration/V110__life_task_context_jurisdiction.sql`
- Create: `app/src/main/resources/db/life/ledger/migration/V2106__jurisdiction_and_erasure_alignment.sql`
- Test: `app/src/test/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandlerTest.java`

**Interfaces:**
- Consumes: nothing (first task)
- Produces: `CreateLifeTaskRequest.jurisdiction()` (String, nullable), `LifeTaskContext.jurisdiction` (String field), `LifeTaskContextResponse.jurisdiction()` (String)

- [ ] **Step 1: Write failing test — jurisdiction fallback in LegalDomainLedgerHandler**

Add a test to `LegalDomainLedgerHandlerTest.java` that verifies: when `LifeTaskContext.jurisdiction` is set, it is used instead of the config default.

```java
@Test
void writeEntry_usesContextJurisdiction_whenPresent() {
    LifeTaskContext ctx = new LifeTaskContext();
    ctx.workItemId = WORK_ITEM_ID;
    ctx.domain = LifeDomain.LEGAL;
    ctx.jurisdiction = "US-CA";

    when(LifeTaskContext.findByIdOptional(WORK_ITEM_ID)).thenReturn(Optional.of(ctx));

    handler.writeEntry(LifeDecisionEventType.COMPLETED, WORK_ITEM_ID, workItem);

    var entry = captureLegalEntry();
    assertThat(entry.jurisdiction).isEqualTo("US-CA");
}

@Test
void writeEntry_fallsBackToConfig_whenContextJurisdictionNull() {
    LifeTaskContext ctx = new LifeTaskContext();
    ctx.workItemId = WORK_ITEM_ID;
    ctx.domain = LifeDomain.LEGAL;
    ctx.jurisdiction = null;

    when(LifeTaskContext.findByIdOptional(WORK_ITEM_ID)).thenReturn(Optional.of(ctx));

    handler.writeEntry(LifeDecisionEventType.COMPLETED, WORK_ITEM_ID, workItem);

    var entry = captureLegalEntry();
    assertThat(entry.jurisdiction).isEqualTo("GB");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LegalDomainLedgerHandlerTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `LifeTaskContext` has no `jurisdiction` field yet.

- [ ] **Step 3: Add jurisdiction field to api/ records**

`api/src/main/java/io/casehub/life/api/request/CreateLifeTaskRequest.java`:
```java
public record CreateLifeTaskRequest(
        @NotBlank String templateRef,
        @NotBlank String title,
        UUID externalActorId,
        Instant deadline,
        @Pattern(regexp = "[A-Z]{2}(-[A-Z0-9]{1,6})?")
        String jurisdiction
) {}
```

Add `import jakarta.validation.constraints.Pattern;`.

`api/src/main/java/io/casehub/life/api/response/LifeTaskContextResponse.java`:
```java
public record LifeTaskContextResponse(
        UUID workItemId,
        LifeDomain domain,
        UUID externalActorId,
        String recurrence,
        String jurisdiction
) {}
```

- [ ] **Step 4: Add jurisdiction column to LifeTaskContext entity**

`app/src/main/java/io/casehub/life/app/entity/LifeTaskContext.java` — add after the `recurrence` field:
```java
@Column(length = 10)
public String jurisdiction;
```

- [ ] **Step 5: Update LifeTaskService to pass jurisdiction through**

`app/src/main/java/io/casehub/life/app/service/LifeTaskService.java` — in `create()`, after `ctx.externalActorId = req.externalActorId();` add:
```java
ctx.jurisdiction = req.jurisdiction();
```

In the `listTasks()` method of `ExternalActorService.java`, update the `LifeTaskContextResponse` constructor call to include `c.jurisdiction`:
```java
.map(c -> new LifeTaskContextResponse(c.workItemId, c.domain, c.externalActorId, c.recurrence, c.jurisdiction))
```

- [ ] **Step 6: Update LegalDomainLedgerHandler to prefer context jurisdiction**

`app/src/main/java/io/casehub/life/app/service/ledger/LegalDomainLedgerHandler.java` — in `writeEntry()`, change:
```java
entry.jurisdiction   = jurisdiction;
```
to:
```java
entry.jurisdiction   = ctx.jurisdiction != null ? ctx.jurisdiction : jurisdiction;
```

- [ ] **Step 7: Align LegalActionLedgerEntry jurisdiction column length**

`app/src/main/java/io/casehub/life/app/ledger/LegalActionLedgerEntry.java` — change:
```java
@Column(name = "jurisdiction", length = 100)
```
to:
```java
@Column(name = "jurisdiction", length = 10)
```

- [ ] **Step 8: Create Flyway migrations**

`app/src/main/resources/db/life/migration/V110__life_task_context_jurisdiction.sql`:
```sql
ALTER TABLE life_task_context ADD COLUMN jurisdiction VARCHAR(10);
```

`app/src/main/resources/db/life/ledger/migration/V2106__jurisdiction_and_erasure_alignment.sql`:
```sql
-- Precondition: no existing erasure entries (domainContentBytes() change is chain-breaking)
DO $$
BEGIN
  IF (SELECT COUNT(*) FROM external_actor_erasure_ledger_entry) > 0 THEN
    RAISE EXCEPTION 'Cannot migrate: existing ExternalActorErasureLedgerEntry records would have invalidated Merkle digests';
  END IF;
END $$;

-- Align jurisdiction column to ISO 3166-1/2 length (was 100, now 10)
ALTER TABLE legal_action_ledger_entry ALTER COLUMN jurisdiction TYPE VARCHAR(10);

-- Add ledger_entries_affected for self-contained erasure proof (#49)
ALTER TABLE external_actor_erasure_ledger_entry ADD COLUMN ledger_entries_affected BIGINT NOT NULL DEFAULT 0;
```

- [ ] **Step 9: Install api, run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LegalDomainLedgerHandlerTest --batch-mode`
Expected: PASS — both jurisdiction tests pass.

- [ ] **Step 10: Commit**

```
feat(#48): per-action jurisdiction on LegalActionLedgerEntry

Adds optional jurisdiction field to CreateLifeTaskRequest and
LifeTaskContext. LegalDomainLedgerHandler prefers task-level jurisdiction,
falls back to tenant-wide config default. ISO 3166-1/2 validation via
@Pattern. Column length aligned to 10 on both entities.

Refs #48
```

---

### Task 2: LifeCaseService Switch Elimination (#51)

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java`
- Modify: `app/src/test/java/io/casehub/life/app/engine/LifeTypedCaseHubTest.java` (add resolution test)

**Interfaces:**
- Consumes: `LifeTypedCaseHub.lifeCaseType()` (from existing codebase — returns `LifeCaseType`)
- Produces: `LifeCaseService.resolve(LifeCaseType)` now uses `Instance<LifeTypedCaseHub>` (internal — no public API change)

- [ ] **Step 1: Write failing test — Instance-based CaseHub resolution**

Add to `app/src/test/java/io/casehub/life/app/engine/LifeTypedCaseHubTest.java`:

```java
@Test
void lifeCaseType_allSubclassesReturnDistinctType() {
    // Verify each LifeTypedCaseHub subclass returns a distinct LifeCaseType.
    // This is the contract LifeCaseService.resolve() depends on.
    var types = List.of(
        new AppointmentCycleCaseHub(),
        new HomeMaintenanceCaseHub(),
        new TravelPlanCaseHub(),
        new CareCoordinationCaseHub(),
        new ContractorCoordinationCaseHub(),
        new FinancialReviewCaseHub()
    );

    var caseTypes = types.stream().map(LifeTypedCaseHub::lifeCaseType).toList();
    assertThat(caseTypes).doesNotHaveDuplicates();
    assertThat(caseTypes).containsExactlyInAnyOrder(LifeCaseType.values());
}
```

Note: The CaseHub constructors call `super(path, agent)` — if they fail to construct without CDI, use reflection or check if a simpler test approach is needed. The key verification is that every `LifeCaseType` has exactly one matching CaseHub.

- [ ] **Step 2: Run test to verify it compiles and passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeTypedCaseHubTest --batch-mode`
Expected: PASS (this test validates the precondition, not the refactor)

- [ ] **Step 3: Refactor LifeCaseService — replace switch with Instance**

`app/src/main/java/io/casehub/life/app/engine/LifeCaseService.java`:

Remove the 6 individual `@Inject` fields:
```java
@Inject AppointmentCycleCaseHub appointmentCycleCaseHub;
@Inject HomeMaintenanceCaseHub homeMaintenanceCaseHub;
@Inject TravelPlanCaseHub travelPlanCaseHub;
@Inject CareCoordinationCaseHub careCoordinationCaseHub;
@Inject ContractorCoordinationCaseHub contractorCoordinationCaseHub;
@Inject FinancialReviewCaseHub financialReviewCaseHub;
```

Replace with:
```java
@Inject @Any
Instance<LifeTypedCaseHub> caseHubs;
```

Add imports:
```java
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
```

Replace the `resolve()` method:
```java
private CaseHub resolve(LifeCaseType type) {
    return caseHubs.stream()
            .filter(hub -> hub.lifeCaseType() == type)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                    "No CaseHub registered for type: " + type));
}
```

Remove unused imports for the 6 CaseHub classes.

- [ ] **Step 4: Build and run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode`
Expected: PASS — all existing tests pass with Instance-based resolution.

- [ ] **Step 5: Commit**

```
refactor(#51): eliminate LifeCaseService.resolve() switch via Instance<LifeTypedCaseHub>

Replaces 6 direct @Inject fields and switch statement with CDI Instance
lookup using lifeCaseType() matching. Adding a new case type now requires
only a new LifeTypedCaseHub subclass — zero changes to LifeCaseService.

Satisfies protocol PP-20260609-bd9d27.

Closes #51
```

---

### Task 3: Extract LifeGdprErasureService + LedgerErasureService Integration (#49)

**Files:**
- Create: `api/src/main/java/io/casehub/life/api/response/ErasureResponse.java`
- Modify: `app/src/main/java/io/casehub/life/app/ledger/ExternalActorErasureLedgerEntry.java`
- Modify: `app/src/main/java/io/casehub/life/app/service/ledger/LifeLedgerWriter.java`
- Create: `app/src/main/java/io/casehub/life/app/service/LifeGdprErasureService.java`
- Modify: `app/src/test/resources/application.properties`
- Create: `app/src/test/java/io/casehub/life/app/service/LifeGdprErasureServiceTest.java`
- Modify: `app/src/test/java/io/casehub/life/app/service/ledger/LifeLedgerWriterTest.java`

**Interfaces:**
- Consumes: `LedgerErasureService.erase(String, ErasureReason)` → `ErasureResult` (from casehub-ledger), `LifeLedgerWriter.writeErasureEntry(ExternalActor, String, int)` (existing), `CaseMemoryStore.eraseEntity(String, String)` (from platform), `LedgerConfig.identity().tokenisation().enabled()` (from casehub-ledger config)
- Produces: `ErasureResponse(UUID erasedActorId, Instant erasedAt, int memoryRecordsErased, long ledgerEntriesAffected, boolean tokenisationEnabled)`, `LifeGdprErasureService.erase(UUID id, String erasedBy)` → `ErasureResponse`

- [ ] **Step 1: Create ErasureResponse record in api/**

`api/src/main/java/io/casehub/life/api/response/ErasureResponse.java`:
```java
package io.casehub.life.api.response;

import java.time.Instant;
import java.util.UUID;

public record ErasureResponse(
        UUID erasedActorId,
        Instant erasedAt,
        int memoryRecordsErased,
        long ledgerEntriesAffected,
        boolean tokenisationEnabled
) {}
```

- [ ] **Step 2: Add ledgerEntriesAffected to ExternalActorErasureLedgerEntry**

`app/src/main/java/io/casehub/life/app/ledger/ExternalActorErasureLedgerEntry.java` — add field after `memoryRecordsErased`:
```java
@Column(name = "ledger_entries_affected", nullable = false)
public long ledgerEntriesAffected;
```

Update `domainContentBytes()`:
```java
@Override
protected byte[] domainContentBytes() {
    return String.join("|",
        erasedActorId != null ? erasedActorId.toString() : "",
        contactMethod != null ? contactMethod : "",
        erasedBy != null ? erasedBy : "",
        String.valueOf(memoryRecordsErased),
        String.valueOf(ledgerEntriesAffected)
    ).getBytes(StandardCharsets.UTF_8);
}
```

- [ ] **Step 3: Update LifeLedgerWriter to accept ledgerEntriesAffected**

`app/src/main/java/io/casehub/life/app/service/ledger/LifeLedgerWriter.java` — change `writeErasureEntry` signature:
```java
public void writeErasureEntry(final ExternalActor actor, final String erasedBy,
                               final int memoryRecordsErased, final long ledgerEntriesAffected) {
    ExternalActorErasureLedgerEntry entry = new ExternalActorErasureLedgerEntry();
    populateBase(entry, actor.id, erasedBy, ActorType.HUMAN, "GdprDataController");
    entry.erasedActorId = actor.id;
    entry.contactMethod = actor.contactMethod;
    entry.erasedBy = erasedBy;
    entry.memoryRecordsErased = memoryRecordsErased;
    entry.ledgerEntriesAffected = ledgerEntriesAffected;
    ledgerRepository.save(entry, TenancyConstants.DEFAULT_TENANT_ID);
}
```

- [ ] **Step 4: Update LifeLedgerWriterTest for new signature**

`app/src/test/java/io/casehub/life/app/service/ledger/LifeLedgerWriterTest.java`:

Update the existing calls from `writeErasureEntry(actor, "household-admin", 0)` to `writeErasureEntry(actor, "household-admin", 0, 0)`.

Add a test:
```java
@Test
void writeErasureEntry_setsLedgerEntriesAffected() {
    writer.writeErasureEntry(externalActor(), "household-admin", 3, 12);

    var entry = captureErasure();
    assertThat(entry.memoryRecordsErased).isEqualTo(3);
    assertThat(entry.ledgerEntriesAffected).isEqualTo(12);
}
```

- [ ] **Step 5: Write failing test — LifeGdprErasureService**

`app/src/test/java/io/casehub/life/app/service/LifeGdprErasureServiceTest.java`:
```java
package io.casehub.life.app.service;

import io.casehub.ledger.api.model.ErasureReason;
import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.privacy.LedgerErasureService;
import io.casehub.life.api.LifeActorIds;
import io.casehub.life.api.LifeActorType;
import io.casehub.life.api.response.ErasureResponse;
import io.casehub.life.app.entity.ExternalActor;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.app.service.ledger.LifeLedgerWriter;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.MemoryCapabilityException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class LifeGdprErasureServiceTest {

    @Mock LedgerErasureService ledgerErasureService;
    @Mock CaseMemoryStore memoryStore;
    @Mock LifeLedgerWriter lifeLedgerWriter;
    @Mock LedgerConfig ledgerConfig;
    @Mock LedgerConfig.IdentityConfig identityConfig;
    @Mock LedgerConfig.IdentityConfig.TokenisationConfig tokenisationConfig;

    @InjectMocks LifeGdprErasureService service;

    private static final UUID ACTOR_ID = UUID.randomUUID();

    @BeforeEach
    void setUp() {
        lenient().when(ledgerConfig.identity()).thenReturn(identityConfig);
        lenient().when(identityConfig.tokenisation()).thenReturn(tokenisationConfig);
        lenient().when(tokenisationConfig.enabled()).thenReturn(true);
        lenient().when(ledgerErasureService.erase(any(), any()))
                .thenReturn(new LedgerErasureService.ErasureResult(
                        LifeActorIds.of(ACTOR_ID), true, 5, Optional.empty()));
    }

    @Test
    void erase_nullifiesPiiAndWritesLedgerEntry() {
        ExternalActor actor = actor();

        ErasureResponse response = service.erase(ACTOR_ID, "jane.admin");

        assertThat(actor.name).isEqualTo("[ERASED]");
        assertThat(actor.contactValue).isEqualTo("[ERASED]");
        assertThat(actor.gdprErasedAt).isNotNull();
        assertThat(response.erasedActorId()).isEqualTo(ACTOR_ID);
        assertThat(response.ledgerEntriesAffected()).isEqualTo(5);
        assertThat(response.tokenisationEnabled()).isTrue();
        verify(lifeLedgerWriter).writeErasureEntry(eq(actor), eq("jane.admin"), eq(0), eq(5L));
    }

    @Test
    void erase_callsLedgerErasureService() {
        actor();
        service.erase(ACTOR_ID, "jane.admin");
        verify(ledgerErasureService).erase(
                eq(LifeActorIds.of(ACTOR_ID)),
                eq(ErasureReason.GDPR_ART_17_REQUEST));
    }

    @Test
    void erase_callsMemoryStoreErase() {
        actor();
        service.erase(ACTOR_ID, "jane.admin");
        verify(memoryStore).eraseEntity(
                eq(LifeActorIds.of(ACTOR_ID)),
                eq(TenancyConstants.DEFAULT_TENANT_ID));
    }

    @Test
    void erase_handlesMemoryCapabilityException() {
        actor();
        when(memoryStore.eraseEntity(any(), any()))
                .thenThrow(new MemoryCapabilityException("not supported"));

        ErasureResponse response = service.erase(ACTOR_ID, "jane.admin");
        assertThat(response.memoryRecordsErased()).isZero();
    }

    @Test
    void erase_throws404_whenActorNotFound() {
        // No actor seeded — findByIdOptional returns empty
        assertThatThrownBy(() -> service.erase(UUID.randomUUID(), "jane.admin"))
                .isInstanceOf(jakarta.ws.rs.NotFoundException.class);
    }

    @Test
    void erase_throws409_whenAlreadyErased() {
        ExternalActor a = actor();
        a.gdprErasedAt = java.time.Instant.now();

        assertThatThrownBy(() -> service.erase(ACTOR_ID, "jane.admin"))
                .isInstanceOf(jakarta.ws.rs.WebApplicationException.class)
                .hasMessageContaining("already erased");
    }

    // Note: active-task guard test requires Panache static methods (findByIdOptional),
    // which cannot be mocked in unit tests. Covered by ExternalActorGdprResourceTest (@QuarkusTest).

    private ExternalActor actor() {
        // This test requires Panache mocking or static method interception.
        // If ExternalActor.findByIdOptional cannot be mocked, this test class
        // should use @QuarkusTest with real persistence instead.
        // For now, the service constructor-injects its dependencies
        // and the actor lookup is a protected method that can be overridden in test.
        // SEE STEP 6 for the actual implementation approach.
        return null; // placeholder — replaced in Step 6
    }
}
```

**Note:** The above test skeleton shows the assertions. The actual Panache static method (`ExternalActor.findByIdOptional`) cannot be mocked with Mockito (GE-20260629-74fc65). Step 6 addresses this by making the actor lookup a package-visible method that the test can override, matching the pattern used by `LegalDomainLedgerHandler.findContext()`.

- [ ] **Step 6: Create LifeGdprErasureService**

`app/src/main/java/io/casehub/life/app/service/LifeGdprErasureService.java`:
```java
package io.casehub.life.app.service;

import io.casehub.ledger.api.model.ErasureReason;
import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.privacy.LedgerErasureService;
import io.casehub.life.api.LifeActorIds;
import io.casehub.life.api.response.ErasureResponse;
import io.casehub.life.app.entity.ExternalActor;
import io.casehub.life.app.entity.LifeTaskContext;
import io.casehub.life.app.service.ledger.LifeLedgerWriter;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.MemoryCapabilityException;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.NotFoundException;
import jakarta.ws.rs.WebApplicationException;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.UUID;

@ApplicationScoped
public class LifeGdprErasureService {

    private static final Logger LOG = Logger.getLogger(LifeGdprErasureService.class);

    @Inject LedgerErasureService ledgerErasureService;
    @Inject CaseMemoryStore memoryStore;
    @Inject LifeLedgerWriter lifeLedgerWriter;
    @Inject LedgerConfig ledgerConfig;

    LifeGdprErasureService() {}

    LifeGdprErasureService(LedgerErasureService ledgerErasureService,
                            CaseMemoryStore memoryStore,
                            LifeLedgerWriter lifeLedgerWriter,
                            LedgerConfig ledgerConfig) {
        this.ledgerErasureService = ledgerErasureService;
        this.memoryStore = memoryStore;
        this.lifeLedgerWriter = lifeLedgerWriter;
        this.ledgerConfig = ledgerConfig;
    }

    @Transactional
    public ErasureResponse erase(final UUID id, final String erasedBy) {
        ExternalActor actor = findActor(id);

        if (actor.gdprErasedAt != null) {
            throw new WebApplicationException(
                    "ExternalActor already erased at " + actor.gdprErasedAt, 409);
        }

        long activeTasks = LifeTaskContext.<LifeTaskContext>list("externalActorId", id)
                .stream()
                .filter(ctx -> {
                    var wi = WorkItem.<WorkItem>findByIdOptional(ctx.workItemId).orElse(null);
                    return wi != null && wi.status.isActive();
                })
                .count();
        if (activeTasks > 0) {
            throw new WebApplicationException(
                    "ExternalActor has " + activeTasks + " active task(s) — close before erasure", 409);
        }

        actor.name = "[ERASED]";
        actor.contactValue = "[ERASED]";
        actor.gdprErasedAt = Instant.now();

        int memoryRecordsErased;
        try {
            memoryRecordsErased = memoryStore.eraseEntity(
                    LifeActorIds.of(id), TenancyConstants.DEFAULT_TENANT_ID);
        } catch (MemoryCapabilityException e) {
            LOG.debugf("Memory store does not support eraseEntity: %s", e.getMessage());
            memoryRecordsErased = 0;
        }

        var erasureResult = ledgerErasureService.erase(
                LifeActorIds.of(id), ErasureReason.GDPR_ART_17_REQUEST);

        lifeLedgerWriter.writeErasureEntry(actor, erasedBy,
                memoryRecordsErased, erasureResult.affectedEntryCount());

        return new ErasureResponse(
                id,
                actor.gdprErasedAt,
                memoryRecordsErased,
                erasureResult.affectedEntryCount(),
                ledgerConfig.identity().tokenisation().enabled());
    }

    protected ExternalActor findActor(UUID id) {
        return ExternalActor.<ExternalActor>findByIdOptional(id)
                .orElseThrow(NotFoundException::new);
    }
}
```

- [ ] **Step 7: Finalise LifeGdprErasureServiceTest with testable pattern**

Update the test to use the `findActor()` override pattern (matching `LegalDomainLedgerHandler.findContext()`):

Replace the `actor()` placeholder method and add an inner subclass:
```java
private ExternalActor seedActor() {
    var a = new ExternalActor();
    a.id = ACTOR_ID;
    a.name = "Bob Contractor";
    a.contactMethod = "phone";
    a.contactValue = "+44-7700-900001";
    a.actorType = LifeActorType.EXTERNAL_HUMAN;
    return a;
}

// Override findActor() to bypass Panache static methods
private LifeGdprErasureService createServiceWithActor(ExternalActor actor) {
    return new LifeGdprErasureService(ledgerErasureService, memoryStore, lifeLedgerWriter, ledgerConfig) {
        @Override
        protected ExternalActor findActor(UUID id) {
            if (actor == null || !actor.id.equals(id)) {
                throw new jakarta.ws.rs.NotFoundException();
            }
            return actor;
        }
    };
}
```

Update each test to use `createServiceWithActor(seedActor())` instead of `@InjectMocks`.

- [ ] **Step 8: Add tokenisation config to test application.properties**

Add to `app/src/test/resources/application.properties`:
```properties
# ============================================================
# Ledger tokenisation + erasure receipts (#49)
# GE-20260531-46f8ab: required for LedgerErasureService.erase() to work
# ============================================================
casehub.ledger.identity.tokenisation.enabled=true
casehub.ledger.erasure-receipt.enabled=true
```

- [ ] **Step 9: Install api, run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode`
Expected: PASS — all tests including new LifeGdprErasureServiceTest and updated LifeLedgerWriterTest.

If any existing test fails due to tokenisation being enabled (actorId assertions), update those assertions: replace `assertThat(entry.actorId).isEqualTo("raw-value")` with `assertThat(entry.actorId).isNotNull()`. Based on codebase scan, `LifeLedgerWriterTest` (Mockito, bypasses identity provider) and `ExternalActorGdprResourceTest` (asserts on `erasedActorId` UUID, not `actorId`) should be unaffected.

- [ ] **Step 10: Commit**

```
feat(#49): extract LifeGdprErasureService, integrate LedgerErasureService

Creates LifeGdprErasureService orchestrating the full GDPR erasure pipeline:
PII nullification → memory erasure → ledger tokenisation → erasure ledger
entry. Adds ledgerEntriesAffected to ExternalActorErasureLedgerEntry for
self-contained erasure proof. Enables tokenisation and erasure receipts in
test config (GE-20260531-46f8ab).

Refs #49
```

---

### Task 4: Wire Compliance Report + Remove Old Erasure Path (#50, #49 close)

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/service/ExternalActorService.java`
- Modify: `app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java`
- Modify: `app/src/test/java/io/casehub/life/app/ExternalActorGdprResourceTest.java`

**Interfaces:**
- Consumes: `LifeGdprErasureService.erase(UUID, String)` → `ErasureResponse` (from Task 3)
- Produces: `DELETE /external-actors/{id}/personal-data` now returns 200 with `ErasureResponse` body

- [ ] **Step 1: Write failing test — erasure returns 200 with ErasureResponse**

Update `ExternalActorGdprResourceTest.java` — change the `eraseActor_204_and_piiNulled` test:
```java
@Test
void eraseActor_200_and_piiNulled() {
    final UUID actorId = createActor();

    given()
            .when().delete("/external-actors/" + actorId + "/personal-data")
            .then()
            .statusCode(200)
            .body("erasedActorId", equalTo(actorId.toString()))
            .body("erasedAt", notNullValue())
            .body("memoryRecordsErased", equalTo(0))
            .body("ledgerEntriesAffected", notNullValue())
            .body("tokenisationEnabled", equalTo(true));

    final ExternalActor persisted = ExternalActor.findById(actorId);
    assertThat(persisted.name).isEqualTo("[ERASED]");
    assertThat(persisted.contactValue).isEqualTo("[ERASED]");
    assertThat(persisted.gdprErasedAt).isNotNull();
}
```

Update all other tests that assert `statusCode(204)` to `statusCode(200)` for the erasure endpoint. The 409 and 404 tests keep their status codes.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ExternalActorGdprResourceTest --batch-mode`
Expected: FAIL — endpoint still returns 204.

- [ ] **Step 3: Remove erase() from ExternalActorService**

`app/src/main/java/io/casehub/life/app/service/ExternalActorService.java`:

Delete the entire `erase()` method (lines ~92-128). Also remove the now-unused `lifeLedgerWriter` and `memoryStore` inject fields if they are no longer referenced by any other method in this class. Keep `trustGateService` — it's used by `toResponse()`.

Remove unused imports: `LifeActorIds`, `LifeLedgerWriter`, `CaseMemoryStore`, `MemoryCapabilityException`, `LifeTaskContext`, `WorkItem`, `Instant`.

Verify remaining imports are still needed by scanning the class for their usage.

- [ ] **Step 4: Update ExternalActorResource to use LifeGdprErasureService**

`app/src/main/java/io/casehub/life/app/resource/ExternalActorResource.java`:

Add inject:
```java
@Inject
LifeGdprErasureService gdprErasureService;
```

Add import:
```java
import io.casehub.life.app.service.LifeGdprErasureService;
import io.casehub.life.api.response.ErasureResponse;
```

Change `erasePersonalData()`:
```java
@DELETE
@Path("/{id}/personal-data")
@RolesAllowed(HouseholdGroups.ADMIN)
public Response erasePersonalData(@PathParam("id") final UUID id) {
    ErasureResponse result = gdprErasureService.erase(id, currentPrincipal.actorId());
    return Response.ok(result).build();
}
```

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode`
Expected: PASS — all tests including updated ExternalActorGdprResourceTest.

- [ ] **Step 6: Commit**

```
feat(#50): wire compliance report — erasure returns ErasureResponse

Changes DELETE /external-actors/{id}/personal-data from 204 to 200 with
ErasureResponse body containing erasedActorId, erasedAt,
memoryRecordsErased, ledgerEntriesAffected, and tokenisationEnabled.
Removes erase() from ExternalActorService — GDPR erasure is now solely
owned by LifeGdprErasureService.

Closes #49, Closes #50
```

---

### Task 5: Full Build Verification + Close #48

- [ ] **Step 1: Full build from clean**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode clean install`
Expected: BUILD SUCCESS with all tests passing.

- [ ] **Step 2: Verify no compilation warnings for unused imports**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl api,app 2>&1 | grep -i "warning" | head -20`
Expected: no unused import warnings from files modified in this branch.

- [ ] **Step 3: Commit close marker for #48**

Task 1 used `Refs #48` — now that all jurisdiction work is complete:
```
chore(#48): close — per-action jurisdiction complete

Closes #48
```
