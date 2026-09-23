# Engine @McpDomain SPI Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/engine#1095 — feat: unified API generation — migrate to @McpDomain SPI
**Issue group:** casehubio/engine#1095 (covers casehubio/platform#300)

**Goal:** Replace hand-written REST endpoints and GraphQL resolvers in casehub-engine with APT-generated code from the platform graphql-generator.

**Architecture:** Define 5 SPI interfaces in engine `api/` module with `@McpDomain` + platform annotations. Create implementation beans in `rest/service/` that delegate to existing engine services. Wire the platform APT generator in `rest/` and `graphql/` modules. Delete hand-written endpoints.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-graphql-generator APT, SmallRye GraphQL, RESTEasy Reactive

## Global Constraints

- All SPI interfaces use `io.casehub.platform.api.mcp` annotations — never JAX-RS on SPIs
- All tenant-scoped SPI methods take explicit `String tenancyId` parameter
- View records in `io.casehub.api.view` (engine api module)
- SPI interfaces in `io.casehub.api.spi` (engine api module)
- SPI implementation beans in `io.casehub.engine.rest.service` (engine rest module)
- APT domain filter: `engine/cases,engine/control,engine/definitions,engine/events,engine/plan`
- `-parameters` flag required on api module compiler config
- Engine project root: `/Users/mdproctor/claude/casehub/slots/194/engine`
- Platform API dep version: `${casehub-platform.version}` from parent POM

---

## Batch 1: SPI Interfaces + View Records

After this batch: api/ module has all SPI interfaces and view records. Compiles. Tests pass. No functional changes yet — REST and GraphQL are untouched.

### Task 1: Create view records and request types in api/

Create the shared view records that replace both REST DTOs and GraphQL types. These are plain Java records with no framework annotations.

**Files:**
- Create: `api/src/main/java/io/casehub/api/view/CaseInstanceView.java`
- Create: `api/src/main/java/io/casehub/api/view/CasePage.java`
- Create: `api/src/main/java/io/casehub/api/view/CaseDefinitionView.java`
- Create: `api/src/main/java/io/casehub/api/view/CaseDefinitionPage.java`
- Create: `api/src/main/java/io/casehub/api/view/CaseControlView.java`
- Create: `api/src/main/java/io/casehub/api/view/SignalResultView.java`
- Create: `api/src/main/java/io/casehub/api/view/EventLogEntryView.java`
- Create: `api/src/main/java/io/casehub/api/view/EventLogPage.java`
- Create: `api/src/main/java/io/casehub/api/view/PlanItemView.java`
- Create: `api/src/main/java/io/casehub/api/view/GoalEvaluationView.java`
- Create: `api/src/main/java/io/casehub/api/view/GoalStatusView.java`
- Create: `api/src/main/java/io/casehub/api/view/CompletionSummaryView.java`
- Create: `api/src/main/java/io/casehub/api/view/CaseStreamEventView.java`
- Create: `api/src/main/java/io/casehub/api/view/CaseLifecycleEventView.java`
- Create: `api/src/main/java/io/casehub/api/view/CaseContextChangeEventView.java`
- Create: `api/src/main/java/io/casehub/api/view/StartCaseRequest.java`
- Create: `api/src/main/java/io/casehub/api/view/CaseControlRequest.java`
- Create: `api/src/main/java/io/casehub/api/view/SendSignalRequest.java`
- Test: `api/src/test/java/io/casehub/api/view/ViewRecordTest.java`

**Interfaces:**
- Consumes: `CaseStatus` from `io.casehub.api.model`, `CaseInstance`/`CaseMetaModel` from `io.casehub.engine.common.internal.model`
- Produces: All view records used by SPI interfaces (Task 2) and SPI implementations (Task 4)

- [ ] **Step 1: Write view record unit test**

```java
package io.casehub.api.view;

import io.casehub.api.model.CaseStatus;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class ViewRecordTest {

    @Test
    void caseInstanceView_createsFromFields() {
        var view = new CaseInstanceView(
            UUID.randomUUID(), CaseStatus.RUNNING,
            "acme", "order", "1.0.0",
            Instant.now(), "alice");
        assertEquals(CaseStatus.RUNNING, view.status());
        assertEquals("acme", view.namespace());
    }

    @Test
    void casePage_wrapsItemsWithPaginationInfo() {
        var items = List.of(new CaseInstanceView(
            UUID.randomUUID(), CaseStatus.RUNNING,
            "ns", "n", "1.0", Instant.now(), "a"));
        var page = new CasePage(items, 42, false);
        assertEquals(42, page.totalCount());
        assertFalse(page.hasMore());
    }

    @Test
    void goalEvaluationView_holdsGoalsAndSummary() {
        var goal = new GoalStatusView("g1", "SIMPLE", true, ".done");
        var summary = new CompletionSummaryView(false, 1, 3, "GOAL_BASED");
        var eval = new GoalEvaluationView(List.of(goal), summary);
        assertEquals(1, eval.goals().size());
        assertFalse(eval.completionSummary().complete());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ViewRecordTest -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: Compilation failure — view records don't exist yet

- [ ] **Step 3: Create all view records**

Each record is a plain Java record. Use `from()` static factory methods where mapping from engine domain objects. Reference existing REST DTOs (`rest/dto/CaseInstanceResponse.java`) for field lists. Records to create:

```java
// CaseInstanceView.java
package io.casehub.api.view;
import io.casehub.api.model.CaseStatus;
import java.time.Instant;
import java.util.UUID;
public record CaseInstanceView(UUID caseId, CaseStatus status,
    String namespace, String name, String version,
    Instant createdAt, String actorId) {}

// CasePage.java
package io.casehub.api.view;
import java.util.List;
public record CasePage(List<CaseInstanceView> items,
    long totalCount, boolean hasMore) {}

// CaseDefinitionView.java
package io.casehub.api.view;
import java.util.List;
public record CaseDefinitionView(String namespace, String name,
    String version, String title, String summary,
    List<String> capabilities) {}

// CaseDefinitionPage.java
package io.casehub.api.view;
import java.util.List;
public record CaseDefinitionPage(List<CaseDefinitionView> items,
    long totalCount, boolean hasMore) {}

// CaseControlView.java
package io.casehub.api.view;
import io.casehub.api.model.CaseStatus;
import java.util.UUID;
public record CaseControlView(UUID caseId, CaseStatus status) {}

// SignalResultView.java
package io.casehub.api.view;
import java.util.UUID;
public record SignalResultView(UUID caseId, boolean accepted) {}

// EventLogEntryView.java
package io.casehub.api.view;
import java.time.Instant;
import java.util.Map;
public record EventLogEntryView(String eventType, String streamType,
    Instant timestamp, Map<String, Object> payload) {}

// EventLogPage.java
package io.casehub.api.view;
import java.util.List;
public record EventLogPage(List<EventLogEntryView> items,
    long totalCount, boolean hasMore) {}

// PlanItemView.java
package io.casehub.api.view;
import io.casehub.api.model.TaskStatus;
import java.util.UUID;
public record PlanItemView(UUID id, String name, TaskStatus status,
    String type, UUID parentId) {}

// GoalEvaluationView.java
package io.casehub.api.view;
import java.util.List;
public record GoalEvaluationView(List<GoalStatusView> goals,
    CompletionSummaryView completionSummary) {}

// GoalStatusView.java
package io.casehub.api.view;
public record GoalStatusView(String name, String kind,
    boolean reached, String condition) {}

// CompletionSummaryView.java
package io.casehub.api.view;
public record CompletionSummaryView(boolean complete, int satisfied,
    int total, String kind) {}

// CaseStreamEventView.java
package io.casehub.api.view;
import java.util.Map;
import java.util.UUID;
public record CaseStreamEventView(UUID caseId, String type,
    Map<String, String> data) {}

// CaseLifecycleEventView.java
package io.casehub.api.view;
import java.util.UUID;
public record CaseLifecycleEventView(UUID caseId, String eventType,
    String commandType, String caseStatus, String actorId,
    String actorRole, String caseDefinitionName, String namespace,
    String satisfiedGoalName, String satisfiedGoalKind) {}

// CaseContextChangeEventView.java
package io.casehub.api.view;
import java.util.Map;
import java.util.UUID;
public record CaseContextChangeEventView(UUID caseId,
    String changedLayer, Map<String, Object> contextSnapshot) {}

// StartCaseRequest.java
package io.casehub.api.view;
import java.util.Map;
public record StartCaseRequest(String namespace, String name,
    String version, Map<String, Object> context) {}

// CaseControlRequest.java
package io.casehub.api.view;
public record CaseControlRequest(String reason) {}

// SendSignalRequest.java
package io.casehub.api.view;
public record SendSignalRequest(String path, String value) {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ViewRecordTest -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/engine add api/src/
git -C /Users/mdproctor/claude/casehub/slots/194/engine commit -m "feat(#1095): add view records and request types for SPI migration Refs #1095"
```

### Task 2: Create SPI interfaces + build config

Define the 5 SPI interfaces with platform annotations. Add `-parameters` compiler flag.

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/EngineCaseApi.java`
- Create: `api/src/main/java/io/casehub/api/spi/EngineCaseControlApi.java`
- Create: `api/src/main/java/io/casehub/api/spi/EngineCaseDefinitionApi.java`
- Create: `api/src/main/java/io/casehub/api/spi/EngineEventLogApi.java`
- Create: `api/src/main/java/io/casehub/api/spi/EnginePlanApi.java`
- Modify: `api/pom.xml` — add `<parameters>true</parameters>`
- Test: `api/src/test/java/io/casehub/api/spi/SpiInterfaceTest.java`

**Interfaces:**
- Consumes: View records from Task 1
- Produces: SPI interfaces consumed by Task 4 (impl beans) and Task 5 (APT generator)

- [ ] **Step 1: Write SPI interface verification test**

```java
package io.casehub.api.spi;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformStream;
import org.junit.jupiter.api.Test;
import java.lang.reflect.Method;
import static org.junit.jupiter.api.Assertions.*;

class SpiInterfaceTest {

    @Test
    void engineCaseApi_hasCorrectDomain() {
        var domain = EngineCaseApi.class.getAnnotation(McpDomain.class);
        assertNotNull(domain);
        assertEquals("engine/cases", domain.value());
    }

    @Test
    void engineCaseApi_hasQueryAndMutationAndStreamMethods() {
        long queries = java.util.Arrays.stream(EngineCaseApi.class.getMethods())
            .filter(m -> m.isAnnotationPresent(PlatformQuery.class)).count();
        long mutations = java.util.Arrays.stream(EngineCaseApi.class.getMethods())
            .filter(m -> m.isAnnotationPresent(PlatformMutation.class)).count();
        long streams = java.util.Arrays.stream(EngineCaseApi.class.getMethods())
            .filter(m -> m.isAnnotationPresent(PlatformStream.class)).count();
        assertEquals(7, queries, "EngineCaseApi should have 7 queries");
        assertEquals(1, mutations, "EngineCaseApi should have 1 mutation");
        assertEquals(2, streams, "EngineCaseApi should have 2 streams");
    }

    @Test
    void allFiveInterfacesHaveMcpDomain() {
        assertNotNull(EngineCaseApi.class.getAnnotation(McpDomain.class));
        assertNotNull(EngineCaseControlApi.class.getAnnotation(McpDomain.class));
        assertNotNull(EngineCaseDefinitionApi.class.getAnnotation(McpDomain.class));
        assertNotNull(EngineEventLogApi.class.getAnnotation(McpDomain.class));
        assertNotNull(EnginePlanApi.class.getAnnotation(McpDomain.class));
    }

    @Test
    void engineCaseApi_parameterNamesPreserved() throws Exception {
        Method m = EngineCaseApi.class.getMethod("getCaseById",
            java.util.UUID.class, String.class);
        assertEquals("caseId", m.getParameters()[0].getName(),
            "-parameters flag must be set on api module");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=SpiInterfaceTest -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: Compilation failure — SPI interfaces don't exist

- [ ] **Step 3: Add `-parameters` flag to api/pom.xml**

Find the `maven-compiler-plugin` configuration in `api/pom.xml` and add `<parameters>true</parameters>` inside `<configuration>`. If no compiler plugin config exists, add one:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <parameters>true</parameters>
  </configuration>
</plugin>
```

Also add `casehub-platform-api` as a dependency in `api/pom.xml` if not already present:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-api</artifactId>
</dependency>
```

- [ ] **Step 4: Create all 5 SPI interfaces**

```java
// EngineCaseApi.java
package io.casehub.api.spi;

import io.casehub.api.view.*;
import io.casehub.api.model.CaseStatus;
import io.casehub.platform.api.mcp.*;
import io.smallrye.mutiny.Multi;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@McpDomain("engine/cases")
public interface EngineCaseApi {
    @PlatformQuery("List case instances with optional filtering")
    @PaginatedResponse
    CasePage listCases(CaseStatus status, String namespace, String name,
                       String tenancyId, Integer offset, Integer limit);

    @PlatformQuery("Get a case instance by ID")
    CaseInstanceView getCaseById(@PathParam UUID caseId, String tenancyId);

    @PlatformMutation("Start a new case instance")
    @RestStatus(201)
    CaseInstanceView startCase(StartCaseRequest request, String tenancyId);

    @PlatformQuery("Get full case context as JSON")
    Map<String, Object> getCaseContext(@PathParam UUID caseId, String tenancyId);

    @PlatformQuery("Get case context at a specific path")
    Map<String, Object> getCaseContextPath(@PathParam UUID caseId,
                                           String path, String tenancyId);

    @PlatformQuery("Get plan items for a case")
    List<PlanItemView> getPlanItems(@PathParam UUID caseId, String tenancyId);

    @PlatformQuery("Evaluate goals against live case context")
    GoalEvaluationView getGoals(@PathParam UUID caseId, String tenancyId);

    @PlatformStream("Live case event stream")
    Multi<CaseStreamEventView> caseStream(@PathParam UUID caseId);

    @PlatformStream("Live case lifecycle events")
    Multi<CaseLifecycleEventView> caseLifecycle(@PathParam UUID caseId);
}
```

```java
// EngineCaseControlApi.java
package io.casehub.api.spi;

import io.casehub.api.view.*;
import io.casehub.platform.api.mcp.*;
import java.util.UUID;

@McpDomain("engine/control")
public interface EngineCaseControlApi {
    @PlatformMutation("Suspend a running case")
    CaseControlView suspendCase(@PathParam UUID caseId,
                                CaseControlRequest request, String tenancyId);

    @PlatformMutation("Resume a suspended case")
    CaseControlView resumeCase(@PathParam UUID caseId,
                               CaseControlRequest request, String tenancyId);

    @PlatformMutation("Cancel a case")
    CaseControlView cancelCase(@PathParam UUID caseId,
                               CaseControlRequest request, String tenancyId);

    @PlatformMutation("Send a signal to a case")
    SignalResultView sendSignal(@PathParam UUID caseId,
                               SendSignalRequest request, String tenancyId);
}
```

```java
// EngineCaseDefinitionApi.java
package io.casehub.api.spi;

import io.casehub.api.view.*;
import io.casehub.platform.api.mcp.*;
import java.util.List;

@McpDomain("engine/definitions")
public interface EngineCaseDefinitionApi {
    @PlatformQuery("List registered case definitions")
    @PaginatedResponse
    CaseDefinitionPage listDefinitions(String tenancyId,
                                       Integer offset, Integer limit);

    @PlatformQuery("Get definitions by namespace and name")
    List<CaseDefinitionView> getDefinitionsByName(
        @PathParam String namespace, @PathParam String name,
        String tenancyId);

    @PlatformQuery("Get a specific definition by namespace, name, and version")
    CaseDefinitionView getDefinitionByKey(
        @PathParam String namespace, @PathParam String name,
        @PathParam String version, String tenancyId);
}
```

```java
// EngineEventLogApi.java
package io.casehub.api.spi;

import io.casehub.api.view.EventLogPage;
import io.casehub.platform.api.mcp.*;
import java.util.List;
import java.util.UUID;

@McpDomain("engine/events")
public interface EngineEventLogApi {
    @PlatformQuery("Get paginated and filtered event log for a case")
    @PaginatedResponse
    EventLogPage getEventLog(@PathParam UUID caseId, String tenancyId,
                             Integer offset, Integer limit,
                             List<String> eventTypes,
                             List<String> streamTypes);
}
```

```java
// EnginePlanApi.java
package io.casehub.api.spi;

import io.casehub.api.view.CaseContextChangeEventView;
import io.casehub.engine.plan.snapshot.DagPlanSnapshot;
import io.casehub.engine.plan.execution.DagResultSnapshot;
import io.casehub.engine.plan.snapshot.DecompositionSnapshot;
import io.casehub.engine.plan.snapshot.PlanItemDefinitionSnapshot;
import io.casehub.engine.rest.dto.ExecutionStateSnapshot;
import io.casehub.engine.plan.execution.CasePlanModelSnapshot;
import io.casehub.platform.api.mcp.*;
import io.smallrye.mutiny.Multi;
import java.util.List;
import java.util.UUID;

@McpDomain("engine/plan")
public interface EnginePlanApi {
    @PlatformQuery("Get live case plan model snapshot")
    CasePlanModelSnapshot getPlanModel(@PathParam UUID caseId, String tenancyId);

    @PlatformQuery("Get plan item definition hierarchy")
    List<PlanItemDefinitionSnapshot> getPlanDefinitions(
        @PathParam UUID caseId, String tenancyId);

    @PlatformQuery("Get HTN decomposition tree snapshot")
    DecompositionSnapshot getDecomposition(@PathParam UUID caseId,
                                           String tenancyId);

    @PlatformQuery("Get DAG plan snapshot")
    DagPlanSnapshot getDagPlan(@PathParam UUID caseId, String tenancyId);

    @PlatformQuery("Get DAG execution result snapshot")
    DagResultSnapshot getDagResult(@PathParam UUID caseId, String tenancyId);

    @PlatformQuery("Get composed execution state snapshot")
    ExecutionStateSnapshot getExecutionState(@PathParam UUID caseId,
                                             String tenancyId);

    @PlatformStream("Live execution state updates")
    Multi<ExecutionStateSnapshot> executionStateStream(@PathParam UUID caseId);
}
```

Note: `EnginePlanApi` returns existing snapshot types directly (not new view records) — these types are already Jackson-serializable and match the blocks-ui TypeScript contracts. `ExecutionStateSnapshot` stays in `rest/dto/` for now since it's used by the broadcaster.

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=SpiInterfaceTest -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: PASS — all 4 tests green. The parameter name test confirms `-parameters` flag works.

- [ ] **Step 6: Run full api module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: All existing + new tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/engine add api/
git -C /Users/mdproctor/claude/casehub/slots/194/engine commit -m "feat(#1095): add 5 SPI interfaces with @McpDomain annotations Refs #1095"
```

---

## Batch 2: Pre-Migration Cleanup + SPI Implementations

After this batch: goal evaluation extracted to CaseService, 5 SPI implementation beans exist and are tested, all existing endpoints still work unchanged.

### Task 3: Extract goal evaluation to CaseService

Move the ~60 lines of inline goal evaluation logic from `CaseInstanceResource.getGoals()` to `CaseService`. This is a prerequisite — the SPI impl needs a service method to delegate to.

**Files:**
- Modify: `rest/src/main/java/io/casehub/engine/rest/service/CaseService.java` — add `evaluateGoals()` + `ExpressionEngineRegistry` injection
- Modify: `rest/src/main/java/io/casehub/engine/rest/CaseInstanceResource.java` — replace inline logic with `caseService.evaluateGoals()` delegation
- Test: `rest/src/test/java/io/casehub/engine/rest/CaseInstanceGoalsResourceTest.java` (existing test — verify it still passes)

**Interfaces:**
- Consumes: `CaseDefinitionRegistry`, `ExpressionEngineRegistry`, `CaseHubRuntime`, `CaseInstance`
- Produces: `CaseService.evaluateGoals(UUID caseId, String tenancyId)` method used by Task 4

- [ ] **Step 1: Read the existing getGoals() implementation**

Read `CaseInstanceResource.java` to understand the full goal evaluation logic, including `buildCompletionSummary()`.

- [ ] **Step 2: Add ExpressionEngineRegistry injection to CaseService**

Use `ide_insert_member` to add:
```java
@Inject ExpressionEngineRegistry expressionEngineRegistry;
```

- [ ] **Step 3: Move evaluateGoals() and buildCompletionSummary() to CaseService**

Extract the body of `getGoals()` from `CaseInstanceResource` into a new method on `CaseService`:

```java
public GoalEvaluationResponse evaluateGoals(UUID caseId, String tenancyId) {
    // ... moved logic from CaseInstanceResource.getGoals()
    // Uses: requireCaseAccess(), definitionRegistry, expressionEngineRegistry, runtime
}

private CompletionSummary buildCompletionSummary(/* params */) {
    // ... moved from CaseInstanceResource
}
```

- [ ] **Step 4: Update CaseInstanceResource.getGoals() to delegate**

Replace the inline logic with:
```java
public GoalEvaluationResponse getGoals(@PathParam("caseId") UUID caseId) {
    return caseService.evaluateGoals(caseId, currentPrincipal.tenancyId());
}
```

- [ ] **Step 5: Run existing goals test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -Dtest=CaseInstanceGoalsResourceTest -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: PASS — behavior unchanged

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/engine add rest/src/
git -C /Users/mdproctor/claude/casehub/slots/194/engine commit -m "refactor(#1095): extract goal evaluation to CaseService Refs #1095"
```

### Task 4: Create SPI implementation beans

Create the 5 `@ApplicationScoped` CDI beans in `rest/service/` that implement the SPI interfaces by delegating to existing engine services.

**Files:**
- Create: `rest/src/main/java/io/casehub/engine/rest/service/DefaultEngineCaseApi.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/service/DefaultEngineCaseControlApi.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/service/DefaultEngineCaseDefinitionApi.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/service/DefaultEngineEventLogApi.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/service/DefaultEnginePlanApi.java`
- Test: `rest/src/test/java/io/casehub/engine/rest/service/DefaultEngineCaseApiTest.java`
- Test: `rest/src/test/java/io/casehub/engine/rest/service/DefaultEngineCaseControlApiTest.java`

**Interfaces:**
- Consumes: SPI interfaces from Task 2, view records from Task 1, `CaseService.evaluateGoals()` from Task 3
- Produces: CDI beans that the APT-generated code will inject via `@Inject EngineCaseApi`

- [ ] **Step 1: Write test for DefaultEngineCaseApi**

Test key delegation paths — getCaseById returns correct view, startCase delegates to CaseService, getGoals delegates to extracted method:

```java
package io.casehub.engine.rest.service;

import io.casehub.api.model.CaseStatus;
import io.casehub.api.spi.EngineCaseApi;
import io.casehub.api.view.CaseInstanceView;
// ... test with Mockito mocks for CaseService, runtime, repos
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class DefaultEngineCaseApiTest {
    // Mock CaseService, CaseHubRuntime, CaseInstanceRepository, etc.
    // Verify delegation: getCaseById → caseService.requireCaseAccess()
    // Verify mapping: CaseInstance → CaseInstanceView fields match
    // Verify getGoals → caseService.evaluateGoals()
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: Compilation failure — impl classes don't exist

- [ ] **Step 3: Implement all 5 SPI beans**

Each bean follows the same pattern — inject existing services, delegate, map to view records:

```java
@ApplicationScoped
public class DefaultEngineCaseApi implements EngineCaseApi {
    @Inject CaseService caseService;
    @Inject CaseHubRuntime runtime;
    @Inject CaseInstanceRepository instanceRepository;
    @Inject PlanItemStore planItemStore;
    @Inject CurrentPrincipal currentPrincipal;
    @Inject CaseStreamBroadcaster caseStreamBroadcaster;
    @Inject CaseEventPublisher caseEventPublisher;

    @Override
    public CaseInstanceView getCaseById(UUID caseId, String tenancyId) {
        var instance = caseService.requireCaseAccess(caseId, AclAction.VIEW);
        return mapToView(instance);
    }
    // ... other methods delegate similarly
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -Dtest="DefaultEngine*Test" -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Run full rest module tests (regression check)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: All tests pass — new beans don't interfere with existing endpoints

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/engine add rest/src/
git -C /Users/mdproctor/claude/casehub/slots/194/engine commit -m "feat(#1095): add 5 SPI implementation beans delegating to engine services Refs #1095"
```

---

## Batch 3: APT Wiring + Switchover

After this batch: hand-written REST resources and GraphQL resolvers deleted, APT-generated code serves all endpoints. Tests updated and passing.

### Task 5: Wire APT generator in rest/ and graphql/ modules

Configure the platform graphql-generator as an annotation processor in both modules with domain filtering.

**Files:**
- Modify: `rest/pom.xml` — add annotationProcessorPaths + compilerArgs
- Modify: `graphql/pom.xml` — add annotationProcessorPaths + compilerArgs

**Interfaces:**
- Consumes: SPI interfaces from Task 2 (Jandex-indexed in api/ JAR)
- Produces: Generated `GeneratedEngine*Resource.java` and `GeneratedEngine*Resolver.java` classes

- [ ] **Step 1: Add APT config to rest/pom.xml**

Add inside `<build><plugins>`:
```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <annotationProcessorPaths>
      <path>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-graphql-generator</artifactId>
        <version>${casehub-platform.version}</version>
      </path>
      <path>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-engine-api</artifactId>
        <version>${project.version}</version>
      </path>
    </annotationProcessorPaths>
    <compilerArgs>
      <arg>-AdomainFilter=engine/cases,engine/control,engine/definitions,engine/events,engine/plan</arg>
      <arg>-AgenerateGraphQL=false</arg>
    </compilerArgs>
  </configuration>
</plugin>
```

- [ ] **Step 2: Add APT config to graphql/pom.xml**

Same annotationProcessorPaths, different compilerArgs:
```xml
<compilerArgs>
  <arg>-AdomainFilter=engine/cases,engine/control,engine/definitions,engine/events,engine/plan</arg>
  <arg>-AgenerateRest=false</arg>
</compilerArgs>
```

- [ ] **Step 3: Build to verify generated code compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl rest,graphql -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: Compilation succeeds. Check `rest/target/generated-sources/annotations/` for `GeneratedEngine*Resource.java` files. Check `graphql/target/generated-sources/annotations/` for `GeneratedEngine*Resolver.java` files.

- [ ] **Step 4: Verify generated classes exist**

```bash
ls /Users/mdproctor/claude/casehub/slots/194/engine/rest/target/generated-sources/annotations/io/casehub/platform/rest/generated/
ls /Users/mdproctor/claude/casehub/slots/194/engine/graphql/target/generated-sources/annotations/io/casehub/platform/graphql/generated/
```

Expected: 5 REST resources and 5 GraphQL resolvers generated.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/engine add rest/pom.xml graphql/pom.xml
git -C /Users/mdproctor/claude/casehub/slots/194/engine commit -m "feat(#1095): wire APT generator with domain filtering Refs #1095"
```

### Task 6: Delete hand-written endpoints + update tests

Remove hand-written REST resources, GraphQL resolvers, and old DTOs. Update tests to hit generated endpoint paths.

**Files:**
- Delete: `rest/src/main/java/io/casehub/engine/rest/CaseInstanceResource.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/CaseControlResource.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/CaseDefinitionResource.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/EventLogResource.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/SignalResource.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/PlanResource.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/CaseStreamResource.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/ExecutionStateResource.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/CaseInstanceResponse.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/CaseControlResponse.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/SignalResponse.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/EventLogEntryResponse.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/PlanItemResponse.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/GoalEvaluationResponse.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/GoalStatusResponse.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/CompletionSummary.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/CompletionStatus.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/CaseStreamEvent.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/StartCaseRequest.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/SendSignalRequest.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/CaseControlRequest.java`
- Delete: `rest/src/main/java/io/casehub/engine/rest/dto/PagedResponse.java`
- Delete: `graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java`
- Delete: `graphql/src/main/java/io/casehub/engine/graphql/CaseMutationResolver.java`
- Delete: `graphql/src/main/java/io/casehub/engine/graphql/CaseSubscriptionResolver.java`
- Delete: All `graphql/src/main/java/io/casehub/engine/graphql/dto/*.java` (13 files)
- Keep: `CaseStreamBroadcaster.java`, `ExecutionStateBroadcaster.java`, `CaseEventPublisher.java`, `EngineModelEnricher.java`, exception mappers, `CaseService.java`, `ExecutionStateSnapshot.java`, `ProblemDetail.java`
- Modify: All existing REST/GraphQL test files — update paths to generated endpoint paths

**Interfaces:**
- Consumes: Generated endpoint classes from Task 5
- Produces: Clean codebase with only generated endpoints

- [ ] **Step 1: Delete hand-written REST resources**

Use `ide_refactor_safe_delete` for each resource class. This ensures no stale references remain.

- [ ] **Step 2: Delete old REST DTOs**

Delete all REST DTOs listed above except `ExecutionStateSnapshot.java` and `ProblemDetail.java` (kept).

- [ ] **Step 3: Delete hand-written GraphQL resolvers + DTOs**

Delete the 3 resolver classes and all 13 DTO files in `graphql/dto/`.

- [ ] **Step 4: Update CaseStreamBroadcaster**

The broadcaster currently creates `CaseStreamEvent` (old DTO). Update it to create `CaseStreamEventView` (new view record). Same fields, different class.

- [ ] **Step 5: Compile to check for missing references**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl rest,graphql -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: Compilation succeeds with generated code replacing deleted classes.

Fix any remaining references to deleted DTOs — they should point to view records in `io.casehub.api.view` instead.

- [ ] **Step 6: Update existing REST tests**

Generated endpoints use different paths (kebab-cased from method names). Update test URL paths. For example:
- Old: `GET /api/v1/cases` → New: `GET /api/engine/cases/list-cases`
- Old: `GET /api/v1/cases/{id}` → New: `GET /api/engine/cases/get-case-by-id/{caseId}`
- Old: `POST /api/v1/cases/{id}/suspend` → New: `POST /api/engine/control/suspend-case/{caseId}`

Read the generated resource files in `target/generated-sources/annotations/` to confirm exact paths.

- [ ] **Step 7: Update existing GraphQL tests**

GraphQL operation names may change. Update test queries to match generated resolver method names.

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest,graphql -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: All tests pass

- [ ] **Step 9: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/slots/194/engine/pom.xml`
Expected: Full build succeeds

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/engine add -A
git -C /Users/mdproctor/claude/casehub/slots/194/engine commit -m "feat(#1095): replace hand-written endpoints with APT-generated code Closes #1095"
```

## References

- [2026-09-15-engine-api-generation-design.md] — design spec this plan implements
- [2026-09-14-unified-api-generation-design.md] — ledger spec (reference pattern)
- [rest/src/main/java/io/casehub/engine/rest/CaseInstanceResource.java] — primary REST resource (goal eval inline logic)
- [rest/src/main/java/io/casehub/engine/rest/service/CaseService.java] — existing service layer
- [graphql/src/main/java/io/casehub/engine/graphql/CaseQueryResolver.java] — primary GraphQL resolver
- [GE-20260914-638e46] — null→200 regression in generated endpoints
- [GE-20260914-3854b8] — APT domain filtering for multi-module
- [GE-20260914-f53be7] — SPI must be in dependency JAR for Jandex APT
- [GE-20260914-714a71] — `-parameters` flag needed on SPI module
- [GE-20260804-8b0fd6] — BroadcastProcessor CDI→Multi bridge
- [GE-20260818-c2f072] — testing MCP dispatch without CDI
- [GE-20260420-05dca8] — REST Assured hangs on SSE
- [casehubio/engine#1095] — focal issue
- [casehubio/platform#300] — generator improvements
