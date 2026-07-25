# LifeAgent Identity Consolidation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidate scattered agent identity strings across 7 CaseHub classes into a `LifeAgent` enum + `LifeAgentDescriptorFactory`, eliminating 62 string literals and 7 boilerplate descriptor methods.

**Architecture:** `LifeAgent` enum (pure identity data, `app.engine`) defines the 4 OpenClaw agents. `LifeAgentDescriptorFactory` (CDI bean, `app.engine.agent`) owns config→descriptor construction. `LifeOpenClawChatModelFactory.forAgent()` changes from `String` to `LifeAgent`. Each CaseHub declares a `private static final LifeAgent AGENT` constant and delegates to the factory.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-eidos-api (AgentDescriptor)

## Global Constraints

- `casehub.life.tenancy-id` is a required config property (no default). Test value: `278776f9-e1b0-46fb-9032-8bddebdcf9ce`
- `casehub.life.jurisdiction` new property, `defaultValue = "GB"`
- AgentDescriptor.builder() is used instead of the 18-arg constructor
- agentId format: `{MODEL_FAMILY}:{persona}@{MAJOR_VERSION}` per `docs/specs/life-actor-model.md`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl api && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app`
- Single test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ClassName --batch-mode`

---

### Task 1: LifeAgent enum + unit tests

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/LifeAgent.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/LifeAgentTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `LifeAgent.agentId()` → `String`, `LifeAgent.persona()` → `String`, `LifeAgent.displayName()` → `String`, `LifeAgent.slot()` → `String`, `LifeAgent.briefing()` → `String`, `LifeAgent.MODEL_FAMILY` → `"openclaw"`, `LifeAgent.MAJOR_VERSION` → `1`

- [ ] **Step 1: Write the failing test**

```java
// app/src/test/java/io/casehub/life/app/engine/LifeAgentTest.java
package io.casehub.life.app.engine;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import static org.assertj.core.api.Assertions.assertThat;

class LifeAgentTest {

    @Test
    void modelFamilyIsOpenclaw() {
        assertThat(LifeAgent.MODEL_FAMILY).isEqualTo("openclaw");
    }

    @Test
    void majorVersionIsOne() {
        assertThat(LifeAgent.MAJOR_VERSION).isEqualTo(1);
    }

    @ParameterizedTest
    @EnumSource(LifeAgent.class)
    void agentIdFollowsConvention(LifeAgent agent) {
        assertThat(agent.agentId())
                .matches("openclaw:[a-z-]+@1")
                .startsWith(LifeAgent.MODEL_FAMILY + ":")
                .endsWith("@" + LifeAgent.MAJOR_VERSION);
    }

    @ParameterizedTest
    @EnumSource(LifeAgent.class)
    void agentIdContainsPersona(LifeAgent agent) {
        assertThat(agent.agentId()).contains(agent.persona());
    }

    @Test
    void healthIdentity() {
        assertThat(LifeAgent.HEALTH.persona()).isEqualTo("health-agent");
        assertThat(LifeAgent.HEALTH.agentId()).isEqualTo("openclaw:health-agent@1");
        assertThat(LifeAgent.HEALTH.displayName()).isEqualTo("OpenClaw Health Agent");
        assertThat(LifeAgent.HEALTH.slot()).isEqualTo("casehubio/life/health");
        assertThat(LifeAgent.HEALTH.briefing()).isEqualTo("Health domain coordination agent");
    }

    @Test
    void homeIdentity() {
        assertThat(LifeAgent.HOME.persona()).isEqualTo("home-agent");
        assertThat(LifeAgent.HOME.agentId()).isEqualTo("openclaw:home-agent@1");
        assertThat(LifeAgent.HOME.displayName()).isEqualTo("OpenClaw Home Agent");
        assertThat(LifeAgent.HOME.slot()).isEqualTo("casehubio/life/household");
        assertThat(LifeAgent.HOME.briefing()).isEqualTo("Household maintenance agent");
    }

    @Test
    void financeIdentity() {
        assertThat(LifeAgent.FINANCE.persona()).isEqualTo("finance-agent");
        assertThat(LifeAgent.FINANCE.agentId()).isEqualTo("openclaw:finance-agent@1");
        assertThat(LifeAgent.FINANCE.displayName()).isEqualTo("OpenClaw Finance Agent");
        assertThat(LifeAgent.FINANCE.slot()).isEqualTo("casehubio/life/finance");
        assertThat(LifeAgent.FINANCE.briefing()).isEqualTo("Financial review and governance agent");
    }

    @Test
    void travelIdentity() {
        assertThat(LifeAgent.TRAVEL.persona()).isEqualTo("travel-agent");
        assertThat(LifeAgent.TRAVEL.agentId()).isEqualTo("openclaw:travel-agent@1");
        assertThat(LifeAgent.TRAVEL.displayName()).isEqualTo("OpenClaw Travel Agent");
        assertThat(LifeAgent.TRAVEL.slot()).isEqualTo("casehubio/life/travel");
        assertThat(LifeAgent.TRAVEL.briefing()).isEqualTo("Travel planning and booking agent");
    }

    @Test
    void fourAgents() {
        assertThat(LifeAgent.values()).hasSize(4);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeAgentTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `LifeAgent` does not exist

- [ ] **Step 3: Write the enum**

```java
// app/src/main/java/io/casehub/life/app/engine/LifeAgent.java
package io.casehub.life.app.engine;

public enum LifeAgent {
    HEALTH("health-agent", "OpenClaw Health Agent",
            "casehubio/life/health", "Health domain coordination agent"),
    HOME("home-agent", "OpenClaw Home Agent",
            "casehubio/life/household", "Household maintenance agent"),
    FINANCE("finance-agent", "OpenClaw Finance Agent",
            "casehubio/life/finance", "Financial review and governance agent"),
    TRAVEL("travel-agent", "OpenClaw Travel Agent",
            "casehubio/life/travel", "Travel planning and booking agent");

    public static final String MODEL_FAMILY = "openclaw";
    public static final int MAJOR_VERSION = 1;

    private final String persona;
    private final String displayName;
    private final String slot;
    private final String briefing;

    LifeAgent(String persona, String displayName, String slot, String briefing) {
        this.persona = persona;
        this.displayName = displayName;
        this.slot = slot;
        this.briefing = briefing;
    }

    public String agentId() {
        return MODEL_FAMILY + ":" + persona + "@" + MAJOR_VERSION;
    }

    public String persona() {
        return persona;
    }

    public String displayName() {
        return displayName;
    }

    public String slot() {
        return slot;
    }

    public String briefing() {
        return briefing;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeAgentTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 8 tests PASS

- [ ] **Step 5: Commit**

```
git add app/src/main/java/io/casehub/life/app/engine/LifeAgent.java app/src/test/java/io/casehub/life/app/engine/LifeAgentTest.java
git commit -m "feat(#46): add LifeAgent enum — agent identity constants

Refs #46"
```

---

### Task 2: LifeAgentDescriptorFactory + unit tests

**Files:**
- Create: `app/src/main/java/io/casehub/life/app/engine/agent/LifeAgentDescriptorFactory.java`
- Create: `app/src/test/java/io/casehub/life/app/engine/agent/LifeAgentDescriptorFactoryTest.java`

**Interfaces:**
- Consumes: `LifeAgent` (Task 1)
- Produces: `LifeAgentDescriptorFactory.descriptorFor(LifeAgent)` → `AgentDescriptor`

- [ ] **Step 1: Write the failing test**

```java
// app/src/test/java/io/casehub/life/app/engine/agent/LifeAgentDescriptorFactoryTest.java
package io.casehub.life.app.engine.agent;

import io.casehub.life.app.engine.LifeAgent;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import static org.assertj.core.api.Assertions.assertThat;

class LifeAgentDescriptorFactoryTest {

    private final LifeAgentDescriptorFactory factory =
            new LifeAgentDescriptorFactory("test-tenant-id", "GB");

    @ParameterizedTest
    @EnumSource(LifeAgent.class)
    void descriptorMatchesAgentIdentity(LifeAgent agent) {
        var descriptor = factory.descriptorFor(agent);

        assertThat(descriptor.agentId()).isEqualTo(agent.agentId());
        assertThat(descriptor.name()).isEqualTo(agent.displayName());
        assertThat(descriptor.slot()).isEqualTo(agent.slot());
        assertThat(descriptor.briefing()).isEqualTo(agent.briefing());
    }

    @ParameterizedTest
    @EnumSource(LifeAgent.class)
    void descriptorUsesModelFamilyConstants(LifeAgent agent) {
        var descriptor = factory.descriptorFor(agent);

        assertThat(descriptor.version()).isEqualTo(String.valueOf(LifeAgent.MAJOR_VERSION));
        assertThat(descriptor.provider()).isEqualTo(LifeAgent.MODEL_FAMILY);
        assertThat(descriptor.modelFamily()).isEqualTo(LifeAgent.MODEL_FAMILY);
    }

    @ParameterizedTest
    @EnumSource(LifeAgent.class)
    void descriptorUsesInjectedConfig(LifeAgent agent) {
        var descriptor = factory.descriptorFor(agent);

        assertThat(descriptor.tenancyId()).isEqualTo("test-tenant-id");
        assertThat(descriptor.jurisdiction()).isEqualTo("GB");
    }

    @ParameterizedTest
    @EnumSource(LifeAgent.class)
    void descriptorLeavesOptionalFieldsNull(LifeAgent agent) {
        var descriptor = factory.descriptorFor(agent);

        assertThat(descriptor.modelVersion()).isNull();
        assertThat(descriptor.weightsFingerprint()).isNull();
        assertThat(descriptor.domainVocabulary()).isNull();
        assertThat(descriptor.slotVocabulary()).isNull();
        assertThat(descriptor.dispositionVocabulary()).isNull();
        assertThat(descriptor.axisVocabularies()).isNull();
        assertThat(descriptor.capabilities()).isEmpty();
        assertThat(descriptor.disposition()).isNull();
        assertThat(descriptor.dataHandlingPolicy()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeAgentDescriptorFactoryTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `LifeAgentDescriptorFactory` does not exist

- [ ] **Step 3: Write the factory**

```java
// app/src/main/java/io/casehub/life/app/engine/agent/LifeAgentDescriptorFactory.java
package io.casehub.life.app.engine.agent;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.life.app.engine.LifeAgent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class LifeAgentDescriptorFactory {

    private final String tenancyId;
    private final String jurisdiction;

    @Inject
    public LifeAgentDescriptorFactory(
            @ConfigProperty(name = "casehub.life.tenancy-id") String tenancyId,
            @ConfigProperty(name = "casehub.life.jurisdiction", defaultValue = "GB") String jurisdiction) {
        this.tenancyId = tenancyId;
        this.jurisdiction = jurisdiction;
    }

    public AgentDescriptor descriptorFor(LifeAgent agent) {
        return AgentDescriptor.builder()
                .agentId(agent.agentId())
                .name(agent.displayName())
                .version(String.valueOf(LifeAgent.MAJOR_VERSION))
                .provider(LifeAgent.MODEL_FAMILY)
                .modelFamily(LifeAgent.MODEL_FAMILY)
                .slot(agent.slot())
                .jurisdiction(jurisdiction)
                .tenancyId(tenancyId)
                .briefing(agent.briefing())
                .build();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=LifeAgentDescriptorFactoryTest --batch-mode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 4 tests (parameterized × 4 agents = 16 invocations) PASS

- [ ] **Step 5: Commit**

```
git add app/src/main/java/io/casehub/life/app/engine/agent/LifeAgentDescriptorFactory.java app/src/test/java/io/casehub/life/app/engine/agent/LifeAgentDescriptorFactoryTest.java
git commit -m "feat(#46): add LifeAgentDescriptorFactory — config-to-descriptor construction

Refs #46"
```

---

### Task 3: Factory signature change + CaseHub migration + test updates

This task is compilation-atomic: changing `forAgent(String)` to `forAgent(LifeAgent)` breaks all 32 call sites in 7 CaseHubs. All changes must be applied before the project compiles.

**Files:**
- Modify: `app/src/main/java/io/casehub/life/app/engine/agent/LifeOpenClawChatModelFactory.java`
- Modify: `app/src/test/java/io/casehub/life/app/engine/agent/TestLifeOpenClawChatModelFactory.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/AppointmentCycleCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/CareCoordinationCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/CareEpisodeCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/HomeMaintenanceCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/ContractorCoordinationCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/FinancialReviewCaseHub.java`
- Modify: `app/src/main/java/io/casehub/life/app/engine/TravelPlanCaseHub.java`
- Modify: `app/src/test/java/io/casehub/life/app/engine/AppointmentCycleCaseHubTest.java`

**Interfaces:**
- Consumes: `LifeAgent` (Task 1), `LifeAgentDescriptorFactory` (Task 2)
- Produces: updated `LifeOpenClawChatModelFactory.forAgent(LifeAgent)` → `ChatModelProvider`

- [ ] **Step 1: Change `LifeOpenClawChatModelFactory.forAgent` signature**

In `app/src/main/java/io/casehub/life/app/engine/agent/LifeOpenClawChatModelFactory.java`:

Change the import section to add:
```java
import io.casehub.life.app.engine.LifeAgent;
```

Change the `forAgent` method (lines 69-85):
```java
public ChatModelProvider forAgent(LifeAgent agent) {
    var provider = new OpenClawAgentProvider(
            bridge, hookClient, agent.persona(), deliveryBaseUrl);
    var chatModel = new OpenClawChatModel(
            provider, Duration.ofSeconds(timeoutSeconds));
    return new ChatModelProvider() {
        @Override
        public ModelType type() {
            return ModelType.OPENAI;
        }

        @Override
        public dev.langchain4j.model.chat.ChatModel get() {
            return chatModel;
        }
    };
}
```

Update the javadoc on `forAgent` — change `@param openClawAgentId the agent identifier (e.g., "health-agent", "home-agent")` to `@param agent the life agent constant`.

- [ ] **Step 2: Update `TestLifeOpenClawChatModelFactory.forAgent` override**

In `app/src/test/java/io/casehub/life/app/engine/agent/TestLifeOpenClawChatModelFactory.java`:

Add import:
```java
import io.casehub.life.app.engine.LifeAgent;
```

Change the `forAgent` override (line 143) from:
```java
public ChatModelProvider forAgent(String openClawAgentId) {
```
to:
```java
public ChatModelProvider forAgent(LifeAgent agent) {
```

- [ ] **Step 3: Migrate AppointmentCycleCaseHub**

In `app/src/main/java/io/casehub/life/app/engine/AppointmentCycleCaseHub.java`:

Add imports:
```java
import io.casehub.life.app.engine.agent.LifeAgentDescriptorFactory;
```
(LifeAgent is same package, no import needed)

Remove import:
```java
import io.casehub.eidos.api.AgentDescriptor;
```

Remove the `@ConfigProperty` tenancyId field (lines 61-62):
```java
@ConfigProperty(name = "casehub.life.tenancy-id")
String tenancyId;
```

Add after the `openClawFactory` field:
```java
@Inject
LifeAgentDescriptorFactory descriptorFactory;
```

Add as first line in the class body (after the fields):
```java
private static final LifeAgent AGENT = LifeAgent.HEALTH;
```

Replace the inline `AgentDescriptor descriptor = new AgentDescriptor(...)` block (lines 83-102) and the `setAgentDescriptors` call with:
```java
yaml.setAgentDescriptors(Map.of(
        AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)));
```

Replace all `openClawFactory.forAgent("health-agent")` calls (5 occurrences at lines 133, 159, 182, 204, 226) with:
```java
openClawFactory.forAgent(AGENT)
```

Remove the `import org.eclipse.microprofile.config.inject.ConfigProperty;` if no longer used.

- [ ] **Step 4: Migrate CareCoordinationCaseHub (HEALTH)**

Same pattern as Step 3. In `app/src/main/java/io/casehub/life/app/engine/CareCoordinationCaseHub.java`:

- Remove `@ConfigProperty tenancyId` field (lines 58-59)
- Remove `import io.casehub.eidos.api.AgentDescriptor;` and `import org.eclipse.microprofile.config.inject.ConfigProperty;`
- Add `@Inject LifeAgentDescriptorFactory descriptorFactory;` and import
- Add `private static final LifeAgent AGENT = LifeAgent.HEALTH;`
- Replace `yaml.setAgentDescriptors(Map.of("openclaw:health-agent@1", healthDescriptor()));` with `yaml.setAgentDescriptors(Map.of(AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)));`
- Replace 3 `openClawFactory.forAgent("health-agent")` calls (lines 101, 123, 145) with `openClawFactory.forAgent(AGENT)`
- Delete `private AgentDescriptor healthDescriptor()` method (lines 159-180)

- [ ] **Step 5: Migrate CareEpisodeCaseHub (HEALTH)**

Same pattern. In `app/src/main/java/io/casehub/life/app/engine/CareEpisodeCaseHub.java`:

- Remove `@ConfigProperty tenancyId` field (lines 53-54)
- Remove AgentDescriptor and ConfigProperty imports
- Add `@Inject LifeAgentDescriptorFactory descriptorFactory;` and import
- Add `private static final LifeAgent AGENT = LifeAgent.HEALTH;`
- Replace `yaml.setAgentDescriptors(Map.of("openclaw:health-agent@1", healthDescriptor()));` with `yaml.setAgentDescriptors(Map.of(AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)));`
- Replace 2 `openClawFactory.forAgent("health-agent")` calls (lines 95, 117) with `openClawFactory.forAgent(AGENT)`
- Delete `private AgentDescriptor healthDescriptor()` method (lines 131-152)

- [ ] **Step 6: Migrate HomeMaintenanceCaseHub (HOME)**

In `app/src/main/java/io/casehub/life/app/engine/HomeMaintenanceCaseHub.java`:

- Remove `@ConfigProperty tenancyId` field (lines 57-58)
- Remove AgentDescriptor and ConfigProperty imports
- Add `@Inject LifeAgentDescriptorFactory descriptorFactory;` and import
- Add `private static final LifeAgent AGENT = LifeAgent.HOME;`
- Replace `yaml.setAgentDescriptors(Map.of("openclaw:home-agent@1", homeDescriptor()));` with `yaml.setAgentDescriptors(Map.of(AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)));`
- Replace 5 `openClawFactory.forAgent("home-agent")` calls (lines 102, 124, 146, 168, 190) with `openClawFactory.forAgent(AGENT)`
- Delete `private AgentDescriptor homeDescriptor()` method (lines 204-225)

- [ ] **Step 7: Migrate ContractorCoordinationCaseHub (HOME)**

In `app/src/main/java/io/casehub/life/app/engine/ContractorCoordinationCaseHub.java`:

- Remove `@ConfigProperty tenancyId` field (lines 62-63)
- Remove AgentDescriptor and ConfigProperty imports
- Add `@Inject LifeAgentDescriptorFactory descriptorFactory;` and import
- Add `private static final LifeAgent AGENT = LifeAgent.HOME;`
- Replace `yaml.setAgentDescriptors(Map.of("openclaw:home-agent@1", homeDescriptor()));` with `yaml.setAgentDescriptors(Map.of(AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)));`
- Replace 5 `openClawFactory.forAgent("home-agent")` calls (lines 107, 129, 151, 173, 195) with `openClawFactory.forAgent(AGENT)`
- Delete `private AgentDescriptor homeDescriptor()` method (lines 209-230)

- [ ] **Step 8: Migrate FinancialReviewCaseHub (FINANCE)**

In `app/src/main/java/io/casehub/life/app/engine/FinancialReviewCaseHub.java`:

- Remove `@ConfigProperty tenancyId` field (lines 66-67)
- Remove AgentDescriptor and ConfigProperty imports
- Add `@Inject LifeAgentDescriptorFactory descriptorFactory;` and import
- Add `private static final LifeAgent AGENT = LifeAgent.FINANCE;`
- Replace `yaml.setAgentDescriptors(Map.of("openclaw:finance-agent@1", financeDescriptor()));` with `yaml.setAgentDescriptors(Map.of(AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)));`
- Replace 5 `openClawFactory.forAgent("finance-agent")` calls (lines 112, 135, 161, 185, 213) with `openClawFactory.forAgent(AGENT)`
- Delete `private AgentDescriptor financeDescriptor()` method (lines 227-248)

- [ ] **Step 9: Migrate TravelPlanCaseHub (TRAVEL)**

In `app/src/main/java/io/casehub/life/app/engine/TravelPlanCaseHub.java`:

- Remove `@ConfigProperty tenancyId` field (lines 66-67)
- Remove AgentDescriptor and ConfigProperty imports
- Add `@Inject LifeAgentDescriptorFactory descriptorFactory;` and import
- Add `private static final LifeAgent AGENT = LifeAgent.TRAVEL;`
- Replace `yaml.setAgentDescriptors(Map.of("openclaw:travel-agent@1", travelDescriptor()));` with `yaml.setAgentDescriptors(Map.of(AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)));`
- Replace 7 `openClawFactory.forAgent("travel-agent")` calls (lines 146, 168, 190, 212, 234, 256, 278) with `openClawFactory.forAgent(AGENT)`
- Delete `private AgentDescriptor travelDescriptor()` method (lines 292-313)

- [ ] **Step 10: Update AppointmentCycleCaseHubTest**

In `app/src/test/java/io/casehub/life/app/engine/AppointmentCycleCaseHubTest.java`:

Add import:
```java
import io.casehub.life.app.engine.LifeAgent;
```

Replace all 4 occurrences of `"openclaw:health-agent@1"` (lines 119, 120, 123, 126) with `LifeAgent.HEALTH.agentId()`.

The test at line 119-128 becomes:
```java
assertThat(def.agentDescriptorFor(LifeAgent.HEALTH.agentId()))
        .as("CaseDefinition must have agentDescriptor for " + LifeAgent.HEALTH.agentId())
        .isPresent();

final var descriptor = def.agentDescriptorFor(LifeAgent.HEALTH.agentId()).orElseThrow();
assertThat(descriptor.agentId())
        .as("agentId must follow {model-family}:{persona}@{major} convention")
        .isEqualTo(LifeAgent.HEALTH.agentId());
assertThat(descriptor.provider()).isEqualTo(LifeAgent.MODEL_FAMILY);
assertThat(descriptor.slot()).isEqualTo(LifeAgent.HEALTH.slot());
```

- [ ] **Step 11: Compile check**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,app --batch-mode`
Expected: BUILD SUCCESS — all files compile with new signatures

- [ ] **Step 12: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode`
Expected: all tests PASS — no behavioural change, only API surface changed

- [ ] **Step 13: Commit**

```
git add -A
git commit -m "feat(#46): migrate CaseHubs to LifeAgent enum + descriptor factory

forAgent(String) → forAgent(LifeAgent). 7 CaseHubs migrated.
7 descriptor methods deleted. 32 persona strings eliminated.
7 tenancyId injections removed. 7 hardcoded 'GB' eliminated.

Refs #46"
```

---

### Task 4: Protocol and documentation updates

**Files:**
- Modify: `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`

**Interfaces:**
- Consumes: Tasks 1-3 (complete implementation)
- Produces: updated protocol, CLAUDE.md update deferred to commit skill

- [ ] **Step 1: Update PP-20260618-openclaw-agent**

In `docs/protocols/casehub-life/openclaw-agent-worker-pattern.md`, update the following sections:

Replace the factory pattern description (lines 48-51) to reference `LifeAgent`:
```
**Factory pattern:** `LifeOpenClawChatModelFactory.forAgent(LifeAgent)` creates a
per-agent `ChatModelProvider`. Each worker gets its own `Agent` with its own system prompt
and response schema, all routed through the same OpenClaw agent. The factory injects
`DirectCallBridge` and `OpenClawHookClient` from `casehub-openclaw-casehub`/`casehub-openclaw-core`.
```

Replace the AgentDescriptor registration description (lines 37-41) to reference `LifeAgentDescriptorFactory`:
```
**AgentDescriptor is registered on CaseDefinition (not Worker)** per engine#543.
In `augment()`, after adding workers:
```java
yaml.setAgentDescriptors(Map.of(
        AGENT.agentId(), descriptorFactory.descriptorFor(AGENT)));
```
`LifeAgentDescriptorFactory` (CDI bean, `app.engine.agent`) owns config→descriptor
construction. `LifeAgent` enum (`app.engine`) defines the 4 agent identity constants.
`CaseDefinition.agentDescriptorFor(agentId)` returns `Optional<AgentDescriptor>`.
```

Add a line referencing life#46 to the `refs:` list in the frontmatter.

- [ ] **Step 2: Commit**

```
git add docs/protocols/casehub-life/openclaw-agent-worker-pattern.md
git commit -m "docs(#46): update PP-20260618-openclaw-agent for LifeAgent

Refs #46"
```

- [ ] **Step 3: CLAUDE.md Layer 7 additions update**

Add to the "Layer 7 additions" section in CLAUDE.md, after the `LifeOpenClawChatModelFactory` bullet:
```
- `LifeAgent` — `app/engine/` enum: 4 agent identity constants (HEALTH, HOME, FINANCE, TRAVEL).
  `agentId()` derives `{MODEL_FAMILY}:{persona}@{MAJOR_VERSION}`. `persona()` returns the bare
  persona for `LifeOpenClawChatModelFactory`. Separate from `LifeDomain` — mapping is not 1:1.
- `LifeAgentDescriptorFactory` — `app/engine/agent/` `@ApplicationScoped`; constructor-injected
  `tenancyId` + `jurisdiction` (new config, default "GB"). `descriptorFor(LifeAgent)` builds
  `AgentDescriptor` via builder. Eliminates 7 per-CaseHub descriptor methods and config injections.
```

- [ ] **Step 4: Commit**

```
git add CLAUDE.md
git commit -m "docs(#46): update CLAUDE.md Layer 7 additions for LifeAgent

Refs #46"
```
