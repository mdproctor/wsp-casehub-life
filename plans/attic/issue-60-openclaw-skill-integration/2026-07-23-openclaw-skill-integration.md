# OpenClaw Skill Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #60 — OpenClaw skill integration — banking, calendar, Home Assistant, messaging
**Issue group:** #60

**Goal:** Transform 32 LLM-backed workers and 7 sentinel heartbeats from reasoning-only
agents into tool-using agents that interact with external services via MCP tools during
OpenClaw `/hooks/agent` turns.

**Architecture:** Two-tier skill model — NATIVE (CaseHub MCP tools with RBAC, audit, trust)
and OPENCLAW (community skills with turn-level accountability). All agents get full MCP
tool access. System prompts guide tool usage; RBAC/ACL gates execution.
See `docs/specs/2026-07-23-openclaw-skill-integration-design.md`.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-iot-api (DeviceEntity types)

## Global Constraints

- All schema changes are additive — new fields are nullable. Existing fields unchanged.
- `List<String> toolsUsed` added to every schema uniformly — LLM self-reported, convenience only (not authoritative for audit).
- System prompt changes must preserve the key substring phrases that `TestLifeOpenClawChatModelFactory` matches on (case-insensitive). Add tool references; do not remove existing descriptive text.
- IoT device types use `DeviceEntity` from `casehub-iot-api` (polymorphic, Jackson `@JsonTypeInfo`). Calendar and banking types use `String`/`Map<String, Object>` until connectors delivers typed SPIs.
- CareEpisodeCaseHub extends `YamlCaseHub` directly (not `LifeTypedCaseHub`) — no `agentWorker()` helper, manual Agent construction.
- FamilyVoteCaseHub has no workers — no changes needed.
- Use `ide_edit_member` to replace record declarations and `ide_replace_member` for method bodies. Use `ide_create_file` for new files.

---

### Task 1: Infrastructure — add casehub-iot-api dependency and config

**Files:**
- Modify: `app/pom.xml`
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/resources/application.properties`
- Test: `app/src/test/java/io/casehub/life/app/engine/agent/ToolAwareSchemaTest.java`

**Interfaces:**
- Consumes: `io.casehub.iot.api.DeviceEntity` (from casehub-iot-api on classpath)
- Produces: `casehub-iot-api` available for response schema imports; skill tier config properties

- [ ] **Step 1: Add casehub-iot-api dependency to app/pom.xml**

Add to `<dependencies>` in `app/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-iot-api</artifactId>
    <scope>compile</scope>
</dependency>
```

Version is managed by the parent POM's `<dependencyManagement>`. If not present in the parent, use the latest SNAPSHOT version matching the IoT repo.

- [ ] **Step 2: Add skill tier config to application.properties**

Append to `app/src/main/resources/application.properties`:

```properties
# Skill tier declarations (§6.2 of design spec)
casehub.life.skills.messaging.tier=native
casehub.life.skills.iot.tier=native
casehub.life.skills.calendar.tier=native
casehub.life.skills.banking.tier=openclaw
```

Add the same to `app/src/test/resources/application.properties`.

- [ ] **Step 3: Verify build compiles with new dependency**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode -Denforcer.skip=true`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add app/pom.xml app/src/main/resources/application.properties app/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#60): add casehub-iot-api dependency and skill tier config

Adds casehub-iot-api for DeviceEntity types in tool-enriched response schemas.
Adds skill tier declarations (messaging=native, iot=native, calendar=native,
banking=openclaw).

Refs #60"
```

---

### Task 2: Home domain — schemas, prompts, sentinels, test factory

Updates HomeMaintenanceCaseHub (5 workers), ContractorCoordinationCaseHub (5 workers),
maintenance-sentinel, and contractor-sentinel.

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/ScheduleInspectionResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/GetQuotesResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/IssueCommitmentResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/MonitorJobResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/RecordCompletionResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/RequestQuoteResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/WatchdogEscalationResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/QuoteReceivedResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/JobMonitoringResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/RecordPaymentResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/MaintenanceSentinelReport.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/ContractorSentinelReport.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/HomeMaintenanceCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/ContractorCoordinationCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java` (home sentinels only)
- Modify: `app/src/test/java/io/casehub/life/app/engine/agent/TestLifeOpenClawChatModelFactory.java` (home entries only)
- Test: `app/src/test/java/io/casehub/life/app/engine/HomeMaintenanceToolAwareTest.java`

**Interfaces:**
- Consumes: `io.casehub.iot.api.DeviceEntity` (Task 1)
- Produces: tool-enriched home domain schemas and prompts

- [ ] **Step 1: Write failing test for tool-aware schemas**

Create `HomeMaintenanceToolAwareTest.java`:

```java
package io.casehub.life.app.engine;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.life.app.engine.agent.ScheduleInspectionResult;
import io.casehub.life.app.engine.agent.MaintenanceSentinelReport;
import io.casehub.life.app.engine.agent.ContractorSentinelReport;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class HomeMaintenanceToolAwareTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void scheduleInspectionResult_hasToolFields() throws Exception {
        var json = """
                {"inspected":true,"condition":"good","inspectionDate":"2026-07-01",
                 "calendarEventId":"evt_123","sensorReadings":null,
                 "toolsUsed":["iot_get_state","calendar_create_event"]}""";
        var result = mapper.readValue(json, ScheduleInspectionResult.class);
        assertEquals("evt_123", result.calendarEventId());
        assertEquals(List.of("iot_get_state", "calendar_create_event"), result.toolsUsed());
    }

    @Test
    void maintenanceSentinelReport_hasToolFields() throws Exception {
        var json = """
                {"progressPercent":75,"status":"on-track","concerns":null,
                 "recommendedAction":"continue","escalationRequired":false,
                 "notificationMessageId":"msg_456",
                 "toolsUsed":["iot_get_state","send_chat"]}""";
        var result = mapper.readValue(json, MaintenanceSentinelReport.class);
        assertEquals("msg_456", result.notificationMessageId());
        assertEquals(List.of("iot_get_state", "send_chat"), result.toolsUsed());
    }

    @Test
    void contractorSentinelReport_hasToolFields() throws Exception {
        var json = """
                {"progressPercent":50,"status":"delayed","concerns":"No response",
                 "recommendedAction":"escalate","escalationRequired":true,
                 "notificationMessageId":"msg_789",
                 "toolsUsed":["send_chat"]}""";
        var result = mapper.readValue(json, ContractorSentinelReport.class);
        assertEquals("msg_789", result.notificationMessageId());
    }

    @Test
    void homeMaintenancePrompt_referencesTools() {
        var hub = new HomeMaintenanceCaseHub();
        // The configureCase method is called during augment — we verify prompt content
        // by inspecting the system prompt constant patterns in the CaseHub source
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=HomeMaintenanceToolAwareTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: FAIL — `calendarEventId` field does not exist on `ScheduleInspectionResult`

- [ ] **Step 3: Update all 10 worker response schemas**

Use `ide_edit_member` to replace each record. The new records add tool-derived fields and `toolsUsed`:

**ScheduleInspectionResult:**
```java
public record ScheduleInspectionResult(
        boolean inspected, String condition, String inspectionDate,
        String calendarEventId, List<String> sensorReadings,
        List<String> toolsUsed) {}
```

**GetQuotesResult:**
```java
public record GetQuotesResult(int quoteCount, List<QuoteItem> quotes,
        String notificationMessageId, List<String> toolsUsed) {
    public record QuoteItem(String contractor, int amount, boolean available) {}
}
```

**IssueCommitmentResult:**
```java
public record IssueCommitmentResult(boolean commitmentIssued, String channel,
        String notificationMessageId, List<String> toolsUsed) {}
```

**MonitorJobResult:**
```java
public record MonitorJobResult(String progress, String estimatedCompletion,
        List<String> sensorReadings, String notificationMessageId,
        List<String> toolsUsed) {}
```

**RecordCompletionResult:**
```java
public record RecordCompletionResult(boolean recorded, String ledgerEntryId,
        String notificationMessageId, List<String> toolsUsed) {}
```

**RequestQuoteResult:**
```java
public record RequestQuoteResult(boolean quoteRequested, String channel,
        boolean deadlinePassed, String notificationMessageId,
        List<String> toolsUsed) {}
```

**WatchdogEscalationResult:**
```java
public record WatchdogEscalationResult(boolean escalated, boolean reminderSent,
        String notificationMessageId, List<String> toolsUsed) {}
```

**QuoteReceivedResult:**
```java
public record QuoteReceivedResult(int quoteAmount, String contractor,
        String validUntil, List<String> toolsUsed) {}
```

**JobMonitoringResult:**
```java
public record JobMonitoringResult(String progress, String estimatedCompletion,
        String notificationMessageId, List<String> toolsUsed) {}
```

**RecordPaymentResult:**
```java
public record RecordPaymentResult(boolean paymentRecorded, int amount,
        String ledgerEntryId, String crossCaseSignal,
        String notificationMessageId, List<String> toolsUsed) {}
```

Add `import java.util.List;` to any schema file that doesn't already have it.

- [ ] **Step 4: Update 2 sentinel report schemas**

**MaintenanceSentinelReport:**
```java
public record MaintenanceSentinelReport(
        int progressPercent, String status, String concerns,
        String recommendedAction, boolean escalationRequired,
        List<String> sensorReadings, String notificationMessageId,
        List<String> toolsUsed) {}
```

**ContractorSentinelReport:**
```java
public record ContractorSentinelReport(
        int progressPercent, String status, String concerns,
        String recommendedAction, boolean escalationRequired,
        String notificationMessageId, List<String> toolsUsed) {}
```

- [ ] **Step 5: Update HomeMaintenanceCaseHub prompts**

Replace the `configureCase` method body with tool-aware prompts. Each prompt preserves
the key substring the test factory matches on, then adds tool instructions:

```java
@Override
protected void configureCase(CaseDefinition definition) {
    definition.getWorkers().add(agentWorker("schedule-inspection", """
            You are a home maintenance agent. Schedule a property inspection,
            assess the condition, and report findings.
            Use iot_get_state to read current sensor data for the property.
            Use calendar_create_event to schedule the inspection appointment.
            Include the calendar event ID and sensor readings in your response.
            If sensors show anomalies (temperature, humidity), flag them.
            If cbrCalibration is provided, use featureStats for historical
            maintenance duration and severity patterns.""", ScheduleInspectionResult.class));
    definition.getWorkers().add(agentWorker("get-quotes", """
            You are a home maintenance agent. Gather contractor quotes for the
            required maintenance work.
            Use send_chat to contact contractors for quotes.
            Include the message ID from send_chat in your response.
            If cbrCalibration is provided, use featureStats.estimatedCost for
            historical cost ranges to assess quote reasonableness.""", GetQuotesResult.class));
    definition.getWorkers().add(agentWorker("issue-commitment", """
            You are a home maintenance agent. Issue a commitment to the selected
            contractor for the approved work.
            Use send_chat to notify the contractor of the accepted quote.
            Include the notification message ID in your response.""", IssueCommitmentResult.class));
    definition.getWorkers().add(agentWorker("monitor-job", """
            You are a home maintenance agent. Monitor job progress and report
            estimated completion.
            Use iot_get_state to check property sensors for work progress indicators.
            Use send_chat to request a status update from the contractor if needed.
            Include sensor readings and any notification message ID in your response.""", MonitorJobResult.class));
    definition.getWorkers().add(agentWorker("record-completion", """
            You are a home maintenance agent. Record job completion to the
            tamper-evident ledger.
            Use send_chat to send completion confirmation to the household.
            Include the notification message ID in your response.""", RecordCompletionResult.class));
}
```

- [ ] **Step 6: Update ContractorCoordinationCaseHub prompts**

Replace the `configureCase` method body:

```java
@Override
protected void configureCase(CaseDefinition definition) {
    definition.getWorkers().add(agentWorker("request-quote", """
            You are a contractor coordination agent. Request a quote from the
            contractor via the appropriate messaging channel.
            Use send_chat to send the quote request to the contractor.
            Include the notification message ID in your response.
            If cbrCalibration is provided, use featureStats.estimatedCost for
            typical cost ranges in similar jobs.""", RequestQuoteResult.class));
    definition.getWorkers().add(agentWorker("watchdog-escalation", """
            You are a contractor coordination agent. Escalate an overdue
            contractor commitment by sending a reminder.
            Use send_chat to send the escalation reminder to the contractor.
            Include the notification message ID in your response.""", WatchdogEscalationResult.class));
    definition.getWorkers().add(agentWorker("quote-received", """
            You are a contractor coordination agent. Process a received quote,
            extracting amount, contractor details, and validity period.""", QuoteReceivedResult.class));
    definition.getWorkers().add(agentWorker("job-monitoring", """
            You are a contractor coordination agent. Monitor an active contractor
            job and report progress.
            Use send_chat to request a progress update from the contractor.
            Include the notification message ID in your response.""", JobMonitoringResult.class));
    definition.getWorkers().add(agentWorker("record-payment", """
            You are a contractor coordination agent. Record a contractor payment
            to the tamper-evident ledger and emit a cross-case signal.
            Use send_chat to send payment confirmation to the contractor.
            Include the notification message ID in your response.""", RecordPaymentResult.class));
}
```

- [ ] **Step 7: Update sentinel prompts in LifeHeartbeatJob (home sentinels)**

In the `sentinelSystemPrompt` method, update the `contractor-sentinel` and `maintenance-sentinel` cases:

```java
case "contractor-sentinel" -> """
        You are a contractor progress monitoring agent for a UK household.
        Check on the status of the active contractor job for this case.
        Use send_chat to contact the contractor for a status update if no recent
        communication exists. Report current progress, status (on-track/delayed/stalled),
        any concerns, and recommended actions. Include any notification message ID
        in your response.""";
case "maintenance-sentinel" -> """
        You are a home maintenance progress monitoring agent for a UK household.
        Check on the status of the active maintenance job for this case.
        Use iot_get_state to read current property sensor data.
        Use send_chat to notify the household if sensors show anomalies.
        Report current progress, status (on-track/delayed/stalled),
        any concerns, and recommended actions. Include sensor readings and
        any notification message ID in your response.""";
```

- [ ] **Step 8: Update TestLifeOpenClawChatModelFactory (home entries)**

Update the home domain entries in the RESPONSES map to include tool-derived fields:

```java
// --- Home domain (home-agent) ---
Map.entry("schedule a property inspection",
        "{\"inspected\":true,\"condition\":\"good\",\"inspectionDate\":\"2026-07-01\","
        + "\"calendarEventId\":\"evt_MOCK_001\",\"sensorReadings\":[\"temp:21.3\",\"humidity:45\"],"
        + "\"toolsUsed\":[\"iot_get_state\",\"calendar_create_event\"]}"),
Map.entry("gather contractor quotes",
        "{\"quoteCount\":2,\"quotes\":[{\"contractor\":\"ABC\",\"amount\":500,"
        + "\"available\":true},{\"contractor\":\"DEF\",\"amount\":650,\"available\":true}],"
        + "\"notificationMessageId\":\"msg_MOCK_001\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
Map.entry("issue a commitment to the selected contractor",
        "{\"commitmentIssued\":true,\"channel\":\"life/contractor/mock\","
        + "\"notificationMessageId\":\"msg_MOCK_002\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
Map.entry("monitor job progress",
        "{\"progress\":\"75% complete\",\"estimatedCompletion\":\"2026-07-15\","
        + "\"sensorReadings\":[\"temp:22.0\"],\"notificationMessageId\":null,"
        + "\"toolsUsed\":[\"iot_get_state\"]}"),
Map.entry("record job completion",
        "{\"recorded\":true,\"ledgerEntryId\":\"LEDGER-MOCK\","
        + "\"notificationMessageId\":\"msg_MOCK_003\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
Map.entry("request a quote",
        "{\"quoteRequested\":true,\"channel\":\"life/contractor/mock\","
        + "\"deadlinePassed\":false,\"notificationMessageId\":\"msg_MOCK_004\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
Map.entry("escalate an overdue",
        "{\"escalated\":true,\"reminderSent\":true,"
        + "\"notificationMessageId\":\"msg_MOCK_005\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
Map.entry("process a received quote",
        "{\"quoteAmount\":500,\"contractor\":\"ABC Plumbing\","
        + "\"validUntil\":\"2026-07-30\",\"toolsUsed\":[]}"),
Map.entry("monitor an active contractor job",
        "{\"progress\":\"50% complete\",\"estimatedCompletion\":\"2026-07-20\","
        + "\"notificationMessageId\":\"msg_MOCK_006\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
Map.entry("record a contractor payment",
        "{\"paymentRecorded\":true,\"amount\":500,\"ledgerEntryId\":\"LEDGER-MOCK\","
        + "\"crossCaseSignal\":\"payment-complete\","
        + "\"notificationMessageId\":\"msg_MOCK_007\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=HomeMaintenanceToolAwareTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

Then run the existing home domain tests to verify backward compatibility:

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=HomeMaintenanceCaseHubTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS (nullable new fields don't break existing test JSON)

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=HomeMaintenanceCaseDefinitionsTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add -A
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#60): home domain — tool-aware schemas, prompts, sentinels

HomeMaintenanceCaseHub and ContractorCoordinationCaseHub prompts now
reference iot_get_state, calendar_create_event, and send_chat tools.
10 worker schemas + 2 sentinel schemas gain tool-derived fields and
toolsUsed. Test factory returns tool-enriched responses.

Refs #60"
```

---

### Task 3: Health domain — schemas, prompts, sentinels, test factory

Updates AppointmentCycleCaseHub (5 workers), CareCoordinationCaseHub (3 workers),
CareEpisodeCaseHub (2 workers), follow-up-sentinel, care-quality-sentinel,
patient-status-sentinel.

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/BookingResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/FindAlternativeResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/ConfirmAppointmentResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/PreVisitPrepResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/RecordHealthDecisionResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/NeedsAssessmentResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/CarePlanResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/HealthCheckResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/AssessPatientResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/ProvideCareResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/FollowUpSentinelReport.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/CareQualitySentinelReport.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/PatientStatusSentinelReport.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/AppointmentCycleCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/CareCoordinationCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/CareEpisodeCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java` (health sentinels)
- Modify: `app/src/test/java/io/casehub/life/app/engine/agent/TestLifeOpenClawChatModelFactory.java` (health entries)
- Test: `app/src/test/java/io/casehub/life/app/engine/HealthDomainToolAwareTest.java`

**Interfaces:**
- Consumes: `io.casehub.iot.api.DeviceEntity` (Task 1) — for patient monitoring sensor data
- Produces: tool-enriched health domain schemas and prompts

- [ ] **Step 1: Write failing test for health domain tool-aware schemas**

Create `HealthDomainToolAwareTest.java`:

```java
package io.casehub.life.app.engine;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.life.app.engine.agent.BookingResult;
import io.casehub.life.app.engine.agent.ConfirmAppointmentResult;
import io.casehub.life.app.engine.agent.FollowUpSentinelReport;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class HealthDomainToolAwareTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void bookingResult_hasToolFields() throws Exception {
        var json = """
                {"appointmentId":"APT-001","provider":"Dr Smith",
                 "confirmed":false,"declined":null,"reason":null,
                 "calendarEventId":"evt_001",
                 "toolsUsed":["calendar_create_event"]}""";
        var result = mapper.readValue(json, BookingResult.class);
        assertEquals("evt_001", result.calendarEventId());
        assertEquals(List.of("calendar_create_event"), result.toolsUsed());
    }

    @Test
    void confirmAppointmentResult_hasToolFields() throws Exception {
        var json = """
                {"confirmed":true,"reminderSent":true,
                 "calendarEventId":"evt_002","notificationMessageId":"msg_001",
                 "toolsUsed":["calendar_create_event","send_chat"]}""";
        var result = mapper.readValue(json, ConfirmAppointmentResult.class);
        assertEquals("msg_001", result.notificationMessageId());
    }

    @Test
    void followUpSentinelReport_hasToolFields() throws Exception {
        var json = """
                {"pendingActions":["prescription"],"daysOverdue":3,
                 "concerns":"Prescription not collected","escalationRequired":true,
                 "calendarEventId":"evt_003","notificationMessageId":"msg_002",
                 "toolsUsed":["calendar_list_events","send_chat"]}""";
        var result = mapper.readValue(json, FollowUpSentinelReport.class);
        assertEquals("evt_003", result.calendarEventId());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=HealthDomainToolAwareTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: FAIL

- [ ] **Step 3: Update 10 worker response schemas**

**BookingResult:**
```java
public record BookingResult(
        String appointmentId, String provider,
        boolean confirmed, Boolean declined, String reason,
        String calendarEventId, List<String> toolsUsed) {}
```

**FindAlternativeResult:**
```java
public record FindAlternativeResult(
        boolean alternativeFound, String appointmentId,
        String provider, boolean confirmed,
        String calendarEventId, List<String> toolsUsed) {}
```

**ConfirmAppointmentResult:**
```java
public record ConfirmAppointmentResult(
        boolean confirmed, boolean reminderSent,
        String calendarEventId, String notificationMessageId,
        List<String> toolsUsed) {}
```

**PreVisitPrepResult:**
```java
public record PreVisitPrepResult(
        boolean checklistSent, String instructions,
        String notificationMessageId, List<String> toolsUsed) {}
```

**RecordHealthDecisionResult:**
```java
public record RecordHealthDecisionResult(
        boolean recorded, String ledgerEntryId,
        List<String> toolsUsed) {}
```

**NeedsAssessmentResult:**
```java
public record NeedsAssessmentResult(
        String careLevel, String recommendedFrequency,
        List<String> specialRequirements,
        List<String> sensorReadings, List<String> toolsUsed) {}
```

**CarePlanResult:**
```java
public record CarePlanResult(
        List<String> schedule, String duration, List<String> tasks,
        String calendarEventId, List<String> toolsUsed) {}
```

**HealthCheckResult:**
```java
public record HealthCheckResult(
        boolean reviewed, boolean healthConcern, String notes,
        List<String> sensorReadings, String notificationMessageId,
        List<String> toolsUsed) {}
```

**AssessPatientResult:**
```java
public record AssessPatientResult(
        VitalSigns vitalSigns, String mobility, String cognition,
        List<String> sensorReadings, List<String> toolsUsed) {
    public record VitalSigns(String bp, int hr, double temp) {}
}
```

**ProvideCareResult:**
```java
public record ProvideCareResult(
        List<String> tasksCompleted, String duration, String observations,
        String notificationMessageId, List<String> toolsUsed) {}
```

- [ ] **Step 4: Update 3 sentinel report schemas**

**FollowUpSentinelReport:**
```java
public record FollowUpSentinelReport(
        List<String> pendingActions, int daysOverdue,
        String concerns, boolean escalationRequired,
        String calendarEventId, String notificationMessageId,
        List<String> toolsUsed) {}
```

**CareQualitySentinelReport:**
```java
public record CareQualitySentinelReport(
        int sessionsScheduled, int sessionsCompleted, List<String> missedSessions,
        String concerns, boolean escalationRequired,
        String calendarEventId, String notificationMessageId,
        List<String> toolsUsed) {}
```

**PatientStatusSentinelReport:**
```java
public record PatientStatusSentinelReport(
        String conditionSummary, String trend,
        List<String> alerts, boolean escalationRequired,
        List<String> sensorReadings, String notificationMessageId,
        List<String> toolsUsed) {}
```

- [ ] **Step 5: Update AppointmentCycleCaseHub prompts**

Replace `bookAppointmentWorker()` system prompt and `configureCase` method body:

In `bookAppointmentWorker()`, update the system prompt:
```java
.systemPrompt("You are a healthcare appointment booking agent for a UK household. " +
        "Book medical appointments with the requested provider. " +
        "Use calendar_create_event to create the appointment in the calendar. " +
        "Include the calendar event ID in your response. " +
        "If the provider is unavailable, set declined=true and provide a reason. " +
        "If cbrCalibration is provided, use historicalSuccessRate to inform " +
        "booking confidence and featureStats for appointment patterns. " +
        "Respond with valid JSON only — no prose, no explanation. " + CBR_SYSTEM_PROMPT_SUFFIX)
```

In `configureCase`, update the remaining workers:
```java
definition.getWorkers().add(agentWorker("find-alternative", """
        You are a healthcare appointment agent. Find an alternative provider
        after a booking was declined. Search available providers and propose
        an alternative appointment.
        Use calendar_list_events to check availability for alternative dates.
        Use calendar_create_event to book the alternative appointment.
        Include the calendar event ID in your response.""", FindAlternativeResult.class));
definition.getWorkers().add(agentWorker("confirm-appointment", """
        You are a healthcare appointment agent. Send appointment confirmation
        to the patient and schedule a reminder for 24 hours before.
        Use calendar_create_event to set the reminder event.
        Use send_chat to send the confirmation message to the patient.
        Include both the calendar event ID and notification message ID.""", ConfirmAppointmentResult.class));
definition.getWorkers().add(agentWorker("pre-visit-prep", """
        You are a healthcare appointment agent. Send pre-visit preparation
        checklist and instructions to the patient.
        Use send_chat to send the preparation instructions.
        Include the notification message ID in your response.""", PreVisitPrepResult.class));
definition.getWorkers().add(agentWorker("record-health-decision", """
        You are a healthcare records agent. Record health decision outcomes
        to the tamper-evident ledger.""", RecordHealthDecisionResult.class));
```

- [ ] **Step 6: Update CareCoordinationCaseHub prompts**

```java
@Override
protected void configureCase(CaseDefinition definition) {
    definition.getWorkers().add(agentWorker("needs-assessment", """
            You are a care coordination agent. Assess care needs for the patient,
            determining care level, recommended frequency, and any special requirements.
            Use iot_get_state to read patient monitoring sensors if available.
            Include sensor readings in your response.""", NeedsAssessmentResult.class));
    definition.getWorkers().add(agentWorker("care-plan", """
            You are a care coordination agent. Create a care plan with schedule,
            duration, and task list based on the needs assessment.
            Use calendar_create_event to schedule care sessions.
            Include the calendar event ID in your response.
            If cbrCalibration is provided, use featureStats for historical care
            duration and frequency patterns.""", CarePlanResult.class));
    definition.getWorkers().add(agentWorker("health-check", """
            You are a care coordination agent. Perform a periodic health check,
            reviewing the patient's condition and flagging any concerns.
            Use iot_get_state to read patient monitoring sensors.
            Use send_chat to notify carers or family if concerns are found.
            Include sensor readings and any notification message ID.""", HealthCheckResult.class));
}
```

- [ ] **Step 7: Update CareEpisodeCaseHub prompts**

Update `assessPatientWorker()` system prompt:
```java
.systemPrompt("""
        You are a care episode agent. Assess patient condition including
        vital signs, mobility status, and cognitive state.
        Use iot_get_state to read patient monitoring sensors (movement,
        temperature, medical devices) if available.
        Include sensor readings in your response.""")
```

Update `provideCareWorker()` system prompt:
```java
.systemPrompt("""
        You are a care episode agent. Provide care to the patient, completing
        assigned tasks and recording observations.
        Use send_chat to notify the family when care is complete.
        Include the notification message ID in your response.""")
```

- [ ] **Step 8: Update sentinel prompts in LifeHeartbeatJob (health sentinels)**

```java
case "follow-up-sentinel" -> """
        You are a health appointment follow-up agent for a UK household.
        Check whether post-appointment actions have been completed:
        prescriptions collected, referrals booked, test results received.
        Use calendar_list_events to check for upcoming follow-up appointments.
        Use send_chat to send reminders for overdue actions.
        Report pending actions, days overdue, and whether escalation is needed.
        Include any calendar event ID and notification message ID.""";
case "care-quality-sentinel" -> """
        You are a care quality monitoring agent for a UK household.
        Check whether scheduled care sessions have been delivered.
        Use calendar_list_events to verify scheduled vs completed sessions.
        Use send_chat to notify the family of any missed sessions.
        Report sessions scheduled vs completed, any missed sessions,
        concerns, and whether escalation is needed.
        Include any calendar event ID and notification message ID.""";
case "patient-status-sentinel" -> """
        You are a patient status monitoring agent for a UK household.
        Assess the patient's current condition between care episodes.
        Use iot_get_state to read patient monitoring sensors (movement,
        temperature, medical devices).
        Use send_chat to alert carers if condition changes are detected.
        Report condition summary, trend (improving/stable/declining),
        any alerts, and whether escalation is needed.
        Include sensor readings and any notification message ID.""";
```

- [ ] **Step 9: Update TestLifeOpenClawChatModelFactory (health entries)**

Update the health domain entries to include tool-derived fields:

```java
// --- Health domain (health-agent) ---
Map.entry("healthcare appointment booking",
        "{\"appointmentId\":\"APT-MOCK\",\"provider\":\"Dr Smith\","
        + "\"confirmed\":false,\"declined\":null,\"reason\":null,"
        + "\"calendarEventId\":\"evt_MOCK_APT\","
        + "\"toolsUsed\":[\"calendar_create_event\"]}"),
Map.entry("find an alternative",
        "{\"alternativeFound\":true,\"appointmentId\":\"APT-ALT-MOCK\","
        + "\"provider\":\"Dr Alternative\",\"confirmed\":false,"
        + "\"calendarEventId\":\"evt_MOCK_ALT\","
        + "\"toolsUsed\":[\"calendar_list_events\",\"calendar_create_event\"]}"),
Map.entry("send appointment confirmation",
        "{\"confirmed\":true,\"reminderSent\":true,"
        + "\"calendarEventId\":\"evt_MOCK_REM\",\"notificationMessageId\":\"msg_MOCK_CONF\","
        + "\"toolsUsed\":[\"calendar_create_event\",\"send_chat\"]}"),
Map.entry("pre-visit preparation",
        "{\"checklistSent\":true,\"instructions\":\"Bring ID, insurance card\","
        + "\"notificationMessageId\":\"msg_MOCK_PREP\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
Map.entry("record health decision",
        "{\"recorded\":true,\"ledgerEntryId\":\"LEDGER-MOCK\","
        + "\"toolsUsed\":[]}"),
Map.entry("assess care needs",
        "{\"careLevel\":\"moderate\",\"recommendedFrequency\":\"weekly\","
        + "\"specialRequirements\":[\"mobility support\"],"
        + "\"sensorReadings\":[\"movement:detected\",\"temp:36.6\"],"
        + "\"toolsUsed\":[\"iot_get_state\"]}"),
Map.entry("create a care plan",
        "{\"schedule\":[\"Mon 9am\",\"Wed 2pm\"],\"duration\":\"2 hours\","
        + "\"tasks\":[\"medication\",\"mobility exercises\"],"
        + "\"calendarEventId\":\"evt_MOCK_CARE\","
        + "\"toolsUsed\":[\"calendar_create_event\"]}"),
Map.entry("periodic health check",
        "{\"reviewed\":true,\"healthConcern\":false,\"notes\":\"Stable condition\","
        + "\"sensorReadings\":[\"temp:36.5\",\"movement:normal\"],"
        + "\"notificationMessageId\":null,"
        + "\"toolsUsed\":[\"iot_get_state\"]}"),
Map.entry("assess patient condition",
        "{\"vitalSigns\":{\"bp\":\"120/80\",\"hr\":72,\"temp\":36.6},"
        + "\"mobility\":\"assisted\",\"cognition\":\"alert\","
        + "\"sensorReadings\":[\"movement:limited\",\"temp:22.1\"],"
        + "\"toolsUsed\":[\"iot_get_state\"]}"),
Map.entry("provide care",
        "{\"tasksCompleted\":[\"medication\",\"mobility\"],\"duration\":\"90 min\","
        + "\"observations\":\"Patient cooperative\","
        + "\"notificationMessageId\":\"msg_MOCK_CARE\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
```

- [ ] **Step 10: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=HealthDomainToolAwareTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

Run existing health tests:
`JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=AppointmentCycleCaseHubTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/life add -A
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#60): health domain — tool-aware schemas, prompts, sentinels

AppointmentCycleCaseHub, CareCoordinationCaseHub, CareEpisodeCaseHub
prompts now reference calendar_create_event, calendar_list_events,
iot_get_state, and send_chat tools. 10 worker schemas + 3 sentinel
schemas gain tool-derived fields and toolsUsed.

Refs #60"
```

---

### Task 4: Finance domain — schemas, prompts, sentinel, test factory

Updates FinancialReviewCaseHub (5 workers) and anomaly-sentinel.

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/GatherDataResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/AnalyseAnomaliesResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/EscalateAnomaliesResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/OversightResponseResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/ProduceReportResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/AnomalySentinelReport.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/FinancialReviewCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java` (anomaly sentinel)
- Modify: `app/src/test/java/io/casehub/life/app/engine/agent/TestLifeOpenClawChatModelFactory.java` (finance entries)
- Test: `app/src/test/java/io/casehub/life/app/engine/FinanceDomainToolAwareTest.java`

**Interfaces:**
- Consumes: nothing from earlier tasks beyond Task 1
- Produces: tool-enriched finance domain schemas and prompts

- [ ] **Step 1: Write failing test**

Create `FinanceDomainToolAwareTest.java`:

```java
package io.casehub.life.app.engine;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.life.app.engine.agent.GatherDataResult;
import io.casehub.life.app.engine.agent.AnomalySentinelReport;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class FinanceDomainToolAwareTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void gatherDataResult_hasToolFields() throws Exception {
        var json = """
                {"totalSpend":5000,"budgetLimit":4500,
                 "categories":["groceries","utilities"],
                 "transactionSummary":{"totalTransactions":42},
                 "toolsUsed":["bank_get_transactions","bank_get_balances"]}""";
        var result = mapper.readValue(json, GatherDataResult.class);
        assertNotNull(result.transactionSummary());
        assertEquals(List.of("bank_get_transactions", "bank_get_balances"), result.toolsUsed());
    }

    @Test
    void anomalySentinelReport_hasToolFields() throws Exception {
        var json = """
                {"anomalies":["unusual charge"],"severity":"medium",
                 "concerns":"Review needed","escalationRequired":false,
                 "transactionSummary":{"flaggedCount":1},
                 "alertMessageId":"msg_001",
                 "toolsUsed":["bank_get_transactions","send_chat"]}""";
        var result = mapper.readValue(json, AnomalySentinelReport.class);
        assertEquals("msg_001", result.alertMessageId());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=FinanceDomainToolAwareTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: FAIL

- [ ] **Step 3: Update 5 worker response schemas**

**GatherDataResult:**
```java
public record GatherDataResult(int totalSpend, int budgetLimit,
        List<String> categories, Map<String, Object> transactionSummary,
        List<String> toolsUsed) {}
```

**AnalyseAnomaliesResult:**
```java
public record AnalyseAnomaliesResult(boolean hasAnomalies, String anomalyDetails,
        Map<String, Object> transactionSummary, List<String> toolsUsed) {}
```

**EscalateAnomaliesResult:**
```java
public record EscalateAnomaliesResult(boolean escalationSent, String channel,
        String notificationMessageId, List<String> toolsUsed) {}
```

**OversightResponseResult:**
```java
public record OversightResponseResult(boolean approved, String comments,
        List<String> toolsUsed) {}
```

**ProduceReportResult:**
```java
public record ProduceReportResult(boolean reportGenerated, String summary,
        String ledgerEntryId, String notificationMessageId,
        List<String> toolsUsed) {}
```

- [ ] **Step 4: Update sentinel report schema**

**AnomalySentinelReport:**
```java
public record AnomalySentinelReport(
        List<String> anomalies, String severity,
        String concerns, boolean escalationRequired,
        Map<String, Object> transactionSummary, String alertMessageId,
        List<String> toolsUsed) {}
```

- [ ] **Step 5: Update FinancialReviewCaseHub prompts**

```java
@Override
protected void configureCase(CaseDefinition definition) {
    definition.getWorkers().add(agentWorker("gather-data", """
            You are a financial review agent. Gather financial data by aggregating
            transactions across all linked accounts.
            Use bank_get_transactions to pull recent transactions.
            Use bank_get_balances to get current account balances.
            Include the transaction summary in your response.""", GatherDataResult.class));
    definition.getWorkers().add(agentWorker("analyse-anomalies", """
            You are a financial review agent. Analyse spending anomalies by
            comparing current spending patterns against budget limits.
            Use bank_get_transactions to identify unusual patterns.
            Include the transaction details supporting any anomalies found.
            If cbrCalibration is provided, use featureStats.estimatedBudget for
            historical spending patterns and threshold calibration.""", AnalyseAnomaliesResult.class));
    definition.getWorkers().add(agentWorker("escalate-anomalies", """
            You are a financial review agent. Escalate anomalies to the oversight
            channel for human review.
            Use send_chat to notify the household admin about flagged anomalies.
            Include the notification message ID in your response.""", EscalateAnomaliesResult.class));
    definition.getWorkers().add(agentWorker("oversight-response", """
            You are a financial review agent. Process oversight response from
            the household admin regarding flagged anomalies.""", OversightResponseResult.class));
    definition.getWorkers().add(agentWorker("produce-report", """
            You are a financial review agent. Produce a monthly financial report
            summarising spending and recording it to the ledger.
            Use send_chat to distribute the report to household members.
            Include the notification message ID in your response.""", ProduceReportResult.class));
}
```

- [ ] **Step 6: Update anomaly-sentinel prompt in LifeHeartbeatJob**

```java
case "anomaly-sentinel" -> """
        You are a financial anomaly detection agent for a UK household.
        Scan recent transactions for unusual patterns, budget overruns,
        or suspicious activity.
        Use bank_get_transactions to pull recent transaction data.
        Use send_chat to alert the household if anomalies are detected.
        Report anomalies found, severity, and whether escalation is needed.
        Include the transaction summary and any alert message ID.""";
```

- [ ] **Step 7: Update TestLifeOpenClawChatModelFactory (finance entries)**

```java
// --- Finance domain (finance-agent) ---
Map.entry("gather financial data",
        "{\"totalSpend\":5000,\"budgetLimit\":4500,"
        + "\"categories\":[\"groceries\",\"utilities\",\"contractor\"],"
        + "\"transactionSummary\":{\"totalTransactions\":42,\"period\":\"2026-07\"},"
        + "\"toolsUsed\":[\"bank_get_transactions\",\"bank_get_balances\"]}"),
Map.entry("analyse spending anomalies",
        "{\"hasAnomalies\":true,\"anomalyDetails\":\"Spending exceeded budget by $500 (11%)\","
        + "\"transactionSummary\":{\"flaggedCount\":3},"
        + "\"toolsUsed\":[\"bank_get_transactions\"]}"),
Map.entry("escalate anomalies",
        "{\"escalationSent\":true,\"channel\":\"life/oversight\","
        + "\"notificationMessageId\":\"msg_MOCK_ESC\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
Map.entry("process oversight response",
        "{\"approved\":true,\"comments\":\"Approved by household admin\","
        + "\"toolsUsed\":[]}"),
Map.entry("produce a monthly financial report",
        "{\"reportGenerated\":true,\"summary\":\"Within budget\","
        + "\"ledgerEntryId\":\"LEDGER-MOCK\","
        + "\"notificationMessageId\":\"msg_MOCK_RPT\","
        + "\"toolsUsed\":[\"send_chat\"]}"),
```

- [ ] **Step 8: Run tests and commit**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=FinanceDomainToolAwareTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

Run existing: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=FinancialReviewCaseHubTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/life add -A
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#60): finance domain — tool-aware schemas, prompts, sentinel

FinancialReviewCaseHub prompts now reference bank_get_transactions,
bank_get_balances, and send_chat tools. Banking data uses Map<String,Object>
(OPENCLAW tier — no CaseHub type). 5 worker schemas + 1 sentinel schema
gain tool-derived fields and toolsUsed.

Refs #60"
```

---

### Task 5: Travel domain — schemas, prompts, sentinel, test factory

Updates TravelPlanCaseHub (7 workers) and booking-sentinel.

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/DestinationResearchResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/FlightSearchResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/HotelSearchResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/BudgetAssessmentResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/TravelBookingResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/RebookingResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/ConfirmationResult.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/BookingSentinelReport.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/TravelPlanCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/LifeHeartbeatJob.java` (booking sentinel)
- Modify: `app/src/test/java/io/casehub/life/app/engine/agent/TestLifeOpenClawChatModelFactory.java` (travel entries)
- Test: `app/src/test/java/io/casehub/life/app/engine/TravelDomainToolAwareTest.java`

**Interfaces:**
- Consumes: nothing from earlier tasks beyond Task 1
- Produces: tool-enriched travel domain schemas and prompts

- [ ] **Step 1: Write failing test**

Create `TravelDomainToolAwareTest.java`:

```java
package io.casehub.life.app.engine;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.life.app.engine.agent.TravelBookingResult;
import io.casehub.life.app.engine.agent.ConfirmationResult;
import io.casehub.life.app.engine.agent.BookingSentinelReport;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class TravelDomainToolAwareTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void travelBookingResult_hasToolFields() throws Exception {
        var json = """
                {"bookingRef":"BK-001","status":"confirmed",
                 "declined":null,"reason":null,
                 "calendarEventId":"evt_travel_001",
                 "toolsUsed":["calendar_create_event"]}""";
        var result = mapper.readValue(json, TravelBookingResult.class);
        assertEquals("evt_travel_001", result.calendarEventId());
    }

    @Test
    void confirmationResult_hasToolFields() throws Exception {
        var json = """
                {"confirmed":true,"itinerarySent":true,"confirmationRef":"CONF-001",
                 "calendarEventId":"evt_travel_002","notificationMessageId":"msg_trav_001",
                 "toolsUsed":["calendar_create_event","send_chat"]}""";
        var result = mapper.readValue(json, ConfirmationResult.class);
        assertEquals("msg_trav_001", result.notificationMessageId());
    }

    @Test
    void bookingSentinelReport_hasToolFields() throws Exception {
        var json = """
                {"bookingStatus":"confirmed","priceChanged":false,
                 "priceChangeDetail":null,"alerts":[],"escalationRequired":false,
                 "calendarEventId":"evt_check_001","reminderMessageId":"msg_rem_001",
                 "toolsUsed":["calendar_list_events","send_chat"]}""";
        var result = mapper.readValue(json, BookingSentinelReport.class);
        assertEquals("evt_check_001", result.calendarEventId());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TravelDomainToolAwareTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: FAIL

- [ ] **Step 3: Update 7 worker response schemas**

**DestinationResearchResult** — add `toolsUsed`:
```java
public record DestinationResearchResult(List<DestinationOption> options,
        List<String> toolsUsed) {
    public record DestinationOption(String name, int cost, String rating) {}
}
```

**FlightSearchResult:**
```java
public record FlightSearchResult(List<FlightOption> flights,
        List<String> toolsUsed) {
    public record FlightOption(String airline, int price, int stops) {}
}
```

**HotelSearchResult:**
```java
public record HotelSearchResult(List<HotelOption> hotels,
        List<String> toolsUsed) {
    public record HotelOption(String name, int price, double rating) {}
}
```

**BudgetAssessmentResult:**
```java
public record BudgetAssessmentResult(
        int totalCost, boolean requiresApproval, boolean isHighValue,
        List<String> toolsUsed) {}
```

**TravelBookingResult:**
```java
public record TravelBookingResult(
        String bookingRef, String status, Boolean declined, String reason,
        String calendarEventId, List<String> toolsUsed) {}
```

**RebookingResult:**
```java
public record RebookingResult(
        String bookingRef, String status, boolean alternativeDates,
        String calendarEventId, List<String> toolsUsed) {}
```

**ConfirmationResult:**
```java
public record ConfirmationResult(
        boolean confirmed, boolean itinerarySent, String confirmationRef,
        String calendarEventId, String notificationMessageId,
        List<String> toolsUsed) {}
```

- [ ] **Step 4: Update sentinel report schema**

**BookingSentinelReport:**
```java
public record BookingSentinelReport(
        String bookingStatus, boolean priceChanged, String priceChangeDetail,
        List<String> alerts, boolean escalationRequired,
        String calendarEventId, String reminderMessageId,
        List<String> toolsUsed) {}
```

- [ ] **Step 5: Update TravelPlanCaseHub prompts**

```java
@Override
protected void configureCase(CaseDefinition definition) {
    definition.getWorkers().add(agentWorker("destination-research", """
            You are a travel planning agent. Research destination options with
            costs and ratings.
            Use calendar_list_events to check existing commitments that may
            conflict with travel dates.""", DestinationResearchResult.class));
    definition.getWorkers().add(agentWorker("flight-search", """
            You are a travel planning agent. Search for flights with airline,
            price, and number of stops.""", FlightSearchResult.class));
    definition.getWorkers().add(agentWorker("hotel-search", """
            You are a travel planning agent. Search for hotels with name,
            price, and rating.""", HotelSearchResult.class));
    definition.getWorkers().add(agentWorker("budget-assessment", """
            You are a travel planning agent. Assess the total travel budget
            and determine if approval is required.
            If cbrCalibration is provided, use featureStats.budget for typical cost
            ranges and historicalSuccessRate to gauge risk.""", BudgetAssessmentResult.class));
    definition.getWorkers().add(agentWorker("booking", """
            You are a travel planning agent. Book the selected flights and hotels.
            Use calendar_create_event to add the travel dates to the calendar.
            Include the calendar event ID in your response.
            If booking fails, set declined=true with a reason.
            If cbrCalibration is provided, use historicalSuccessRate to inform
            booking confidence.""", TravelBookingResult.class));
    definition.getWorkers().add(agentWorker("rebooking", """
            You are a travel planning agent. Rebook after a declined booking,
            finding alternative dates.
            Use calendar_create_event to add the new travel dates to the calendar.
            Include the calendar event ID in your response.""", RebookingResult.class));
    definition.getWorkers().add(agentWorker("confirmation", """
            You are a travel planning agent. Confirm the travel itinerary and
            send confirmation details.
            Use calendar_create_event to create the final itinerary event.
            Use send_chat to send confirmation to all travellers.
            Include the calendar event ID and notification message ID.""", ConfirmationResult.class));

    definition.getBindings().addAll(List.of(
            familyVoteBinding("family-vote-a"),
            familyVoteBinding("family-vote-b"),
            familyVoteBinding("family-vote-c")
    ));
}
```

- [ ] **Step 6: Update booking-sentinel prompt in LifeHeartbeatJob**

```java
case "booking-sentinel" -> """
        You are a travel booking monitoring agent for a UK household.
        Check booking confirmations, price changes, and availability.
        Use calendar_list_events to verify travel dates are still in the calendar.
        Use send_chat to send reminders for upcoming travel.
        Report booking status, any price changes, alerts, and whether
        escalation is needed.
        Include any calendar event ID and reminder message ID.""";
```

- [ ] **Step 7: Update TestLifeOpenClawChatModelFactory (travel entries)**

```java
// --- Travel domain (travel-agent) ---
Map.entry("research destination options",
        "{\"options\":[{\"name\":\"Paris\",\"cost\":1200,\"rating\":\"4.5\"},"
        + "{\"name\":\"Barcelona\",\"cost\":900,\"rating\":\"4.3\"}],"
        + "\"toolsUsed\":[\"calendar_list_events\"]}"),
Map.entry("search for flights",
        "{\"flights\":[{\"airline\":\"BA\",\"price\":450,\"stops\":0},"
        + "{\"airline\":\"RY\",\"price\":280,\"stops\":1}],"
        + "\"toolsUsed\":[]}"),
Map.entry("search for hotels",
        "{\"hotels\":[{\"name\":\"Grand Hotel\",\"price\":120,\"rating\":4.5},"
        + "{\"name\":\"Budget Inn\",\"price\":60,\"rating\":3.0}],"
        + "\"toolsUsed\":[]}"),
Map.entry("assess the total travel budget",
        "{\"totalCost\":3500,\"requiresApproval\":true,\"isHighValue\":false,"
        + "\"toolsUsed\":[]}"),
Map.entry("book the selected flights and hotels",
        "{\"bookingRef\":\"BK-MOCK\",\"status\":\"confirmed\","
        + "\"declined\":null,\"reason\":null,"
        + "\"calendarEventId\":\"evt_MOCK_TRAVEL\","
        + "\"toolsUsed\":[\"calendar_create_event\"]}"),
Map.entry("rebook after a declined",
        "{\"bookingRef\":\"BK-REBK-MOCK\",\"status\":\"confirmed\","
        + "\"alternativeDates\":true,"
        + "\"calendarEventId\":\"evt_MOCK_REBK\","
        + "\"toolsUsed\":[\"calendar_create_event\"]}"),
Map.entry("confirm the travel itinerary",
        "{\"confirmed\":true,\"itinerarySent\":true,"
        + "\"confirmationRef\":\"CONF-MOCK\","
        + "\"calendarEventId\":\"evt_MOCK_ITIN\","
        + "\"notificationMessageId\":\"msg_MOCK_TRAV\","
        + "\"toolsUsed\":[\"calendar_create_event\",\"send_chat\"]}"),
```

- [ ] **Step 8: Run tests and commit**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TravelDomainToolAwareTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

Run existing: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TravelPlanCaseHubTest --batch-mode -Denforcer.skip=true -Dsurefire.failIfNoSpecifiedTests=false -am`
Expected: PASS

```bash
git -C /Users/mdproctor/claude/casehub/life add -A
git -C /Users/mdproctor/claude/casehub/life commit -m "feat(#60): travel domain — tool-aware schemas, prompts, sentinel

TravelPlanCaseHub prompts now reference calendar_create_event,
calendar_list_events, and send_chat tools. 7 worker schemas +
1 sentinel schema gain tool-derived fields and toolsUsed.

Refs #60"
```

---

### Task 6: Full build verification and cross-repo issues

**Files:**
- No new files — verification task
- GitHub issues filed on connectors and iot repos

**Interfaces:**
- Consumes: all changes from Tasks 1-5
- Produces: green build, filed cross-repo issues

- [ ] **Step 1: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,app --batch-mode -Denforcer.skip=true`
Expected: BUILD SUCCESS with all tests passing

If any test fails, fix the root cause (likely a test factory substring mismatch or
a schema deserialization issue from a missing `@JsonIgnoreProperties(ignoreUnknown = true)`)
and re-run.

- [ ] **Step 2: File cross-repo issues**

File the following issues using `gh issue create`:

**connectors — Calendar SPI:**
```bash
gh issue create --repo casehubio/connectors --title "feat: CalendarPlatform SPI + Google Calendar provider" --body "$(cat <<'EOF'
## Context

casehub-life#60 (OpenClaw skill integration) needs calendar tools for agent access.
Follow the existing `ChatPlatform` SPI pattern in `chat-spi/`.

## Scope

1. `calendar-spi/` — `CalendarPlatform` SPI (listEvents, createEvent, deleteEvent)
2. `calendar-ref/` — reference implementation for testing
3. `calendar-google/` — Google Calendar provider (jgccli wrapper)
4. MCP tools in `mcp/` — `CalendarMcpTool` following `ChatPlatformMcpTool` pattern

## Design

See `casehub-life/docs/specs/2026-07-23-openclaw-skill-integration-design.md` §2.4

## Not in scope

- OpenClaw fallback provider (future)
- Outlook/CalDAV providers (future)
EOF
)"
```

**iot — MCP tool exposure:**
```bash
gh issue create --repo casehubio/iot --title "feat: MCP tool exposure for DeviceProvider operations" --body "$(cat <<'EOF'
## Context

casehub-life#60 (OpenClaw skill integration) needs IoT tools for agent access.
The DeviceProvider SPI and providers (HA, OpenHAB) already exist.

## Scope

Add MCP tools that serve DeviceProvider operations:
- `iot_get_devices` — list devices with type and state
- `iot_get_state` — get current state for a device
- `iot_send_command` — send a command to a device

Follow the `quarkus-mcp-server-http` pattern used in connectors/mcp.

## Design

See `casehub-life/docs/specs/2026-07-23-openclaw-skill-integration-design.md` §2.3
EOF
)"
```

Record the issue numbers in the commit message.

- [ ] **Step 3: Update issue #60 description**

Remove the stale "Blocked on" prereqs and add cross-repo issue references:

```bash
gh issue edit 60 --repo casehubio/life --body "$(cat <<'EOF'
## Context

Split from #8 (Layer 7 epic). The structural wiring is complete — sentinels, heartbeats,
agent execution, provisioner, cleanup. This issue integrates real external capabilities
via a two-tier skill model (NATIVE + OPENCLAW).

## Status

All 32 worker prompts and 7 sentinel prompts upgraded to tool-aware.
Response schemas gain tool-derived fields and toolsUsed.

## Cross-repo prerequisites (non-blocking — life uses mocks until delivered)

- connectors#TBD — CalendarPlatform SPI + Google Calendar provider
- iot#TBD — MCP tool exposure for DeviceProvider operations

## Design

See `docs/specs/2026-07-23-openclaw-skill-integration-design.md`
EOF
)"
```

Replace `#TBD` with actual issue numbers from Step 2.

- [ ] **Step 4: Commit any fixes**

If Step 1 required fixes, commit them:

```bash
git -C /Users/mdproctor/claude/casehub/life add -A
git -C /Users/mdproctor/claude/casehub/life commit -m "fix(#60): test suite fixes from full build verification

Refs #60"
```

---

### Task 7: Documentation — CLAUDE.md, ARC42STORIES.MD, protocol

**Files:**
- Modify: `CLAUDE.md` (project repo)
- Modify: `ARC42STORIES.MD` (project repo)

**Interfaces:**
- Consumes: completed implementation from Tasks 1-6
- Produces: updated documentation reflecting skill integration

- [ ] **Step 1: Update CLAUDE.md Layer 7 status**

In the Foundation Layers section, update Layer 7 to reflect skill integration completion.
Add after the existing Layer 7 entries:

```markdown
Layer 7 (skill integration): + OpenClaw skill ecosystem — two-tier skill model
         (NATIVE/OPENCLAW). All 32 workers + 7 sentinels upgraded to tool-aware prompts.
         Response schemas gain tool-derived fields (calendarEventId, sensorReadings,
         transactionSummary, notificationMessageId) and toolsUsed. Messaging NATIVE
         (connectors ChatPlatform SPI). IoT NATIVE (casehub-iot DeviceProvider SPI).
         Calendar HYBRID (native Google Calendar planned, OpenClaw fallback). Banking
         OPENCLAW only. Full MCP tool discovery for all agents; RBAC/ACL gates execution.
         ✅ COMPLETE (life-side wiring)  🔲 PENDING (connectors calendar SPI, iot MCP tools)
```

- [ ] **Step 2: Update CLAUDE.md "What This Project Owns" section**

Add to the Layer 7 additions:

```markdown
**Layer 7 additions (skill integration, life#60):**
- Two-tier skill model: NATIVE (CaseHub MCP tools, full platform properties) and
  OPENCLAW (community skills, turn-level accountability). Promotion OPENCLAW → NATIVE
  is transparent to agents.
- Tool-aware system prompts for all 32 workers and 7 sentinels reference available MCP
  tools (calendar_create_event, iot_get_state, bank_get_transactions, send_chat).
- Response schemas gain tool-derived fields: calendarEventId, sensorReadings,
  transactionSummary, notificationMessageId, alertMessageId, reminderMessageId.
- `List<String> toolsUsed` on all 39 schemas — LLM self-reported, convenience for UI
  and debugging (not authoritative for audit).
- `casehub-iot-api` dependency for `DeviceEntity` types in sensor readings.
- Banking data uses `Map<String, Object>` (OPENCLAW tier — no CaseHub type).
- Skill tier config: `casehub.life.skills.{domain}.tier` in application.properties.
```

- [ ] **Step 3: Add ARC42STORIES.MD §9.4 layer entry**

Add a layer entry for skill integration under §9.4:

```markdown
### Layer 7d — OpenClaw Skill Integration

**Issue:** #60
**Status:** ✅ Complete (life-side wiring)
**Depends on:** Layer 7a (AgentExec), Layer 7b (risk classification), Layer 7c (WorkerProvisioner)

Two-tier skill model transforms reasoning-only agents into tool-using agents.
32 worker prompts and 7 sentinel prompts reference MCP tools for calendar,
IoT, banking, and messaging. Response schemas capture tool-derived data.
NATIVE tier provides full platform properties (RBAC, audit, trust, GDPR).
OPENCLAW tier provides the long tail via community skills.

Cross-repo prerequisites (non-blocking): connectors CalendarPlatform SPI,
iot MCP tool exposure.
```

- [ ] **Step 4: Commit documentation**

```bash
git -C /Users/mdproctor/claude/casehub/life add CLAUDE.md ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/life commit -m "docs(#60): update CLAUDE.md and ARC42STORIES.MD for skill integration

Adds Layer 7d (skill integration) to layer taxonomy and ARC42STORIES.
Documents two-tier skill model, tool-aware prompts, and cross-repo
prerequisites.

Refs #60"
```
