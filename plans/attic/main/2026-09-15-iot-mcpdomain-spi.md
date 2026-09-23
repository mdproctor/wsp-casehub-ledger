# IoT @McpDomain SPI Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/iot#105 — migrate iot to @McpDomain SPI
**Issue group:** casehubio/platform#300

**Goal:** Replace 7 hand-written REST resources with 5 @McpDomain SPI
interfaces, generating REST + GraphQL + MCP endpoints from the SPIs.

**Architecture:** Define SPI interfaces in `webapp-api` with platform
annotations (`@PlatformQuery`, `@PlatformMutation`, `@ContextParam`).
Impl beans in `webapp` delegate to existing services. APT generates
endpoint classes. 3 resources stay hand-written (KPI, WorkItem, SSE).

**Tech Stack:** Java 21, Quarkus 3.32.2, platform-api MCP annotations,
`casehub-platform-graphql-generator` APT

## Global Constraints

- Annotation model: `@PlatformQuery`/`@PlatformMutation` + overrides (`@RestMethod`, `@RestPath`). NO JAX-RS annotations on SPI interfaces.
- `@ContextParam("tenancyId")` on all tenant-scoped methods
- `-parameters` compiler flag on webapp-api (GE-20260914-714a71)
- `-AdomainFilter=iot/devices,iot/situations,iot/situations/suppressions,iot/cases,iot/ops` (GE-20260914-3854b8)
- SPI interfaces in webapp-api, impl beans in webapp (D1)
- Placeholder methods use `default` + `NotImplementedException` (D5)
- MCP coexistence: curated `casehub-iot-mcp` stays alongside generated surface (D7)
- All commits reference `Refs casehubio/iot#105`

---

## Batch 1: View Records + SPI Interfaces (webapp-api)

### Task 1: View records, NotImplementedException, and compiler setup

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/NotImplementedException.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/StateHistoryView.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/SituationDefinitionView.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/ActiveSituationView.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/ProviderStatusView.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/RefreshResultView.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/BridgeConnectionsView.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/AuditTrailView.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/CaseSummaryView.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/CaseDetailView.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/view/SuggestionView.java`
- Modify: `webapp-api/pom.xml` (add `-parameters` compiler flag)
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/view/ViewRecordTest.java`

**Interfaces:**
- Produces: `NotImplementedException(String operation)` — used by SPI default methods (Task 2)
- Produces: All view records — used by SPI return types (Task 2) and impl beans (Tasks 3-5)

- [ ] **Step 1: Create NotImplementedException**

```java
package io.casehub.iot.webapp;

public class NotImplementedException extends RuntimeException {
    private final String operation;

    public NotImplementedException(String operation) {
        super("Not implemented: " + operation);
        this.operation = operation;
    }

    public String operation() { return operation; }
}
```

Temporary location — promoted to `platform-api` when platform#300 delivers it.

- [ ] **Step 2: Create view records — device domain**

`StateHistoryView.java`:
```java
package io.casehub.iot.webapp.view;

import java.time.Instant;
import java.util.List;

public record StateHistoryView(
    String deviceId,
    String deviceClass,
    Object stateSnapshot,
    List<String> changedCapabilities,
    Instant occurredAt) {}
```

- [ ] **Step 3: Create view records — situation domain**

`SituationDefinitionView.java`:
```java
package io.casehub.iot.webapp.view;

import java.time.Instant;

public record SituationDefinitionView(
    String situationId,
    String tenancyId,
    Object definition,
    Instant createdAt,
    Instant updatedAt,
    String source) {}
```

`ActiveSituationView.java`:
```java
package io.casehub.iot.webapp.view;

import java.time.Instant;

public record ActiveSituationView(
    String situationId,
    String correlationKey,
    double confidence,
    int signalCount,
    Instant firstSignal,
    Instant lastSignal) {}
```

- [ ] **Step 4: Create view records — operations domain**

`ProviderStatusView.java`:
```java
package io.casehub.iot.webapp.view;

public record ProviderStatusView(
    String providerId,
    String status,
    int deviceCount) {}
```

`RefreshResultView.java`:
```java
package io.casehub.iot.webapp.view;

public record RefreshResultView(String message) {}
```

`BridgeConnectionsView.java`:
```java
package io.casehub.iot.webapp.view;

import java.time.Instant;
import java.util.List;

public record BridgeConnectionsView(
    boolean connected,
    List<TenancyConnection> tenancies) {

    public record TenancyConnection(
        String tenancyId,
        Instant connectedSince) {}
}
```

`AuditTrailView.java`:
```java
package io.casehub.iot.webapp.view;

import java.time.Instant;
import java.util.List;

public record AuditTrailView(
    List<AuditRecord> records,
    int totalCount,
    int offset,
    int limit) {

    public record AuditRecord(
        String eventType,
        String deviceId,
        String correlationId,
        Object payload,
        Instant occurredAt) {}
}
```

- [ ] **Step 5: Create view records — case domain**

`CaseSummaryView.java`:
```java
package io.casehub.iot.webapp.view;

import java.time.Instant;
import java.util.UUID;

public record CaseSummaryView(
    UUID caseId,
    String caseType,
    String status,
    String situationId,
    int pendingActionsCount,
    Instant createdAt) {}
```

`CaseDetailView.java`:
```java
package io.casehub.iot.webapp.view;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record CaseDetailView(
    UUID caseId,
    String caseType,
    String status,
    String situationId,
    List<CaseEventView> events,
    List<WorkerResultView> workerResults,
    List<PlannedActionView> plannedActions,
    Instant createdAt,
    Instant completedAt) {

    public record CaseEventView(String type, String detail, Instant occurredAt) {}
    public record WorkerResultView(String workerId, String outcome, Object data) {}
    public record PlannedActionView(String action, String status, Object parameters) {}
}
```

`SuggestionView.java`:
```java
package io.casehub.iot.webapp.view;

import io.casehub.ras.api.cbr.ResolutionSuggestion;
import java.util.List;
import java.util.UUID;

public record SuggestionView(
    UUID caseId,
    String caseType,
    int suggestionCount,
    List<ResolutionSuggestion> suggestions) {}
```

- [ ] **Step 6: Add `-parameters` compiler flag to webapp-api**

In `webapp-api/pom.xml`, add to the `maven-compiler-plugin` configuration:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <parameters>true</parameters>
    </configuration>
</plugin>
```

- [ ] **Step 7: Write compilation test**

`webapp-api/src/test/java/io/casehub/iot/webapp/view/ViewRecordTest.java`:
```java
package io.casehub.iot.webapp.view;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class ViewRecordTest {

    @Test
    void stateHistoryView_roundTrip() {
        var view = new StateHistoryView("d1", "Switch", null, List.of("on"), Instant.now());
        assertEquals("d1", view.deviceId());
        assertEquals("Switch", view.deviceClass());
    }

    @Test
    void situationDefinitionView_roundTrip() {
        var view = new SituationDefinitionView("temp-rise", "t1", null, Instant.now(), Instant.now(), "config");
        assertEquals("temp-rise", view.situationId());
    }

    @Test
    void caseSummaryView_roundTrip() {
        var view = new CaseSummaryView(UUID.randomUUID(), "incident", "OPEN", "temp-rise", 3, Instant.now());
        assertEquals("incident", view.caseType());
    }

    @Test
    void bridgeConnectionsView_nestedRecords() {
        var conn = new BridgeConnectionsView.TenancyConnection("t1", Instant.now());
        var view = new BridgeConnectionsView(true, List.of(conn));
        assertTrue(view.connected());
        assertEquals(1, view.tenancies().size());
    }
}
```

- [ ] **Step 8: Build and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webapp-api -f /Users/mdproctor/claude/casehub/slots/194/iot/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```
feat(#105): add view records and NotImplementedException for @McpDomain SPI

Promote inline inner records from REST resources to top-level view
records in webapp-api. Add NotImplementedException for placeholder
endpoints. Add -parameters compiler flag for APT parameter names.

Refs casehubio/iot#105
```

---

### Task 2: Define 5 SPI interfaces

**Files:**
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/spi/IoTDeviceApi.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/spi/IoTSituationApi.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/spi/IoTSuppressionApi.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/spi/IoTCaseApi.java`
- Create: `webapp-api/src/main/java/io/casehub/iot/webapp/spi/IoTOperationsApi.java`
- Test: `webapp-api/src/test/java/io/casehub/iot/webapp/spi/SpiCompilationTest.java`

**Interfaces:**
- Consumes: All view records from Task 1
- Consumes: `NotImplementedException` from Task 1
- Consumes: Existing DTOs: `DeviceResponse`, `CommandRequest`, `CommandResponse`, `SituationDefinitionRequest`, `SituationSuggestionsResponse`, `DismissRequest`, `SuppressionHistoryResponse`, `SuppressionStatsResponse`, `HealthOverviewResponse`, `QueueEntrySummary`, `QueueEntryDetail`
- Produces: 5 SPI interfaces — consumed by impl beans (Tasks 3-5) and APT generator (Task 6)

- [ ] **Step 1: Create IoTDeviceApi**

```java
package io.casehub.iot.webapp.spi;

import io.casehub.iot.webapp.rest.CommandRequest;
import io.casehub.iot.webapp.rest.CommandResponse;
import io.casehub.iot.webapp.rest.DeviceResponse;
import io.casehub.iot.webapp.view.StateHistoryView;
import io.casehub.platform.api.mcp.ContextParam;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestStatus;

import java.time.Instant;
import java.util.List;

@McpDomain("iot/devices")
public interface IoTDeviceApi {

    @PlatformQuery("List devices with optional filtering")
    List<DeviceResponse> listDevices(String deviceClass, String providerId, Boolean available,
                                     @ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("Get device by ID")
    DeviceResponse getDevice(@PathParam String deviceId,
                             @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Dispatch a command to a device")
    @RestStatus(201)
    CommandResponse dispatchCommand(@PathParam String deviceId,
                                    CommandRequest command,
                                    @ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("Get device state history")
    List<StateHistoryView> getDeviceHistory(@PathParam String deviceId,
                                            Instant from, Instant to, Integer limit,
                                            @ContextParam("tenancyId") String tenancyId);
}
```

- [ ] **Step 2: Create IoTSituationApi**

```java
package io.casehub.iot.webapp.spi;

import io.casehub.iot.webapp.NotImplementedException;
import io.casehub.iot.webapp.rest.DismissRequest;
import io.casehub.iot.webapp.rest.SituationDefinitionRequest;
import io.casehub.iot.webapp.rest.SituationSuggestionsResponse;
import io.casehub.iot.webapp.view.ActiveSituationView;
import io.casehub.iot.webapp.view.SituationDefinitionView;
import io.casehub.platform.api.mcp.ContextParam;
import io.casehub.platform.api.mcp.HttpMethod;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestMethod;
import io.casehub.platform.api.mcp.RestStatus;

import java.util.List;

@McpDomain("iot/situations")
public interface IoTSituationApi {

    @PlatformQuery("List situation definitions")
    List<SituationDefinitionView> listDefinitions(@ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Create a situation definition")
    @RestStatus(201)
    SituationDefinitionView createDefinition(SituationDefinitionRequest request,
                                              @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Update a situation definition")
    @RestMethod(HttpMethod.PUT)
    SituationDefinitionView updateDefinition(@PathParam String situationId,
                                              SituationDefinitionRequest request,
                                              @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Delete a situation definition")
    @RestMethod(HttpMethod.DELETE)
    void deleteDefinition(@PathParam String situationId,
                          @ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("List active situations")
    default List<ActiveSituationView> listActive(@ContextParam("tenancyId") String tenancyId) {
        throw new NotImplementedException("listActive");
    }

    @PlatformQuery("Get resolution suggestions for a situation")
    SituationSuggestionsResponse getSuggestions(@PathParam String situationId,
                                                @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Dismiss an active situation")
    void dismissSituation(@PathParam String correlationKey,
                          DismissRequest request,
                          @ContextParam("tenancyId") String tenancyId);
}
```

- [ ] **Step 3: Create IoTSuppressionApi**

```java
package io.casehub.iot.webapp.spi;

import io.casehub.iot.webapp.rest.SuppressionHistoryResponse;
import io.casehub.iot.webapp.rest.SuppressionStatsResponse;
import io.casehub.platform.api.mcp.ContextParam;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

@McpDomain("iot/situations/suppressions")
public interface IoTSuppressionApi {

    @PlatformQuery("List suppression history")
    List<SuppressionHistoryResponse> listSuppressions(String situationId, Instant since,
                                                       Boolean includeOverridden,
                                                       @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Override a suppression")
    void overrideSuppression(@PathParam UUID id,
                             @ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("Get suppression statistics for a situation")
    SuppressionStatsResponse getSuppressionStats(@PathParam String situationId,
                                                  @ContextParam("tenancyId") String tenancyId);
}
```

- [ ] **Step 4: Create IoTCaseApi**

```java
package io.casehub.iot.webapp.spi;

import io.casehub.iot.webapp.NotImplementedException;
import io.casehub.iot.webapp.resolution.QueueEntryDetail;
import io.casehub.iot.webapp.resolution.QueueEntrySummary;
import io.casehub.iot.webapp.view.CaseDetailView;
import io.casehub.iot.webapp.view.CaseSummaryView;
import io.casehub.iot.webapp.view.SuggestionView;
import io.casehub.platform.api.mcp.ContextParam;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

@McpDomain("iot/cases")
public interface IoTCaseApi {

    @PlatformQuery("List cases with optional filtering")
    default List<CaseSummaryView> listCases(String status, String situationId,
                                             Instant from, Instant to,
                                             @ContextParam("tenancyId") String tenancyId) {
        throw new NotImplementedException("listCases");
    }

    @PlatformQuery("Get case by ID")
    default CaseDetailView getCase(@PathParam UUID caseId,
                                    @ContextParam("tenancyId") String tenancyId) {
        throw new NotImplementedException("getCase");
    }

    @PlatformQuery("Get case resolution suggestions")
    SuggestionView getCaseSuggestions(@PathParam UUID caseId,
                                      @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Accept a resolution suggestion for a case")
    void acceptSuggestion(@PathParam UUID caseId, @PathParam UUID pastCaseId,
                          @ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("List resolution queue entries")
    List<QueueEntrySummary> listResolutionQueue(String view, String status,
                                                 @ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("Get resolution queue entry detail")
    QueueEntryDetail getResolutionQueueEntry(@PathParam UUID entryId,
                                              @ContextParam("tenancyId") String tenancyId);
}
```

- [ ] **Step 5: Create IoTOperationsApi**

```java
package io.casehub.iot.webapp.spi;

import io.casehub.iot.webapp.rest.HealthOverviewResponse;
import io.casehub.iot.webapp.view.AuditTrailView;
import io.casehub.iot.webapp.view.BridgeConnectionsView;
import io.casehub.iot.webapp.view.ProviderStatusView;
import io.casehub.iot.webapp.view.RefreshResultView;
import io.casehub.platform.api.mcp.ContextParam;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;

import java.time.Instant;
import java.util.List;

@McpDomain("iot/ops")
public interface IoTOperationsApi {

    @PlatformQuery("List IoT providers with status")
    List<ProviderStatusView> listProviders(@ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("Get provider by ID")
    ProviderStatusView getProvider(@PathParam String providerId,
                                   @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Refresh all providers")
    RefreshResultView refreshAllProviders(@ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Refresh a specific provider")
    RefreshResultView refreshProvider(@PathParam String providerId,
                                      @ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("List bridge connections")
    BridgeConnectionsView getBridgeConnections(@ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("Query bridge audit trail")
    AuditTrailView getBridgeAudit(String eventType, String deviceId, String correlationId,
                                   Instant from, Instant to, Integer offset, Integer limit,
                                   @ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("Get system health overview")
    HealthOverviewResponse getHealthOverview(@ContextParam("tenancyId") String tenancyId);
}
```

- [ ] **Step 6: Write SPI compilation test**

`webapp-api/src/test/java/io/casehub/iot/webapp/spi/SpiCompilationTest.java`:
```java
package io.casehub.iot.webapp.spi;

import io.casehub.platform.api.mcp.McpDomain;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class SpiCompilationTest {

    @Test
    void allSpisHaveMcpDomainAnnotation() {
        assertNotNull(IoTDeviceApi.class.getAnnotation(McpDomain.class));
        assertNotNull(IoTSituationApi.class.getAnnotation(McpDomain.class));
        assertNotNull(IoTSuppressionApi.class.getAnnotation(McpDomain.class));
        assertNotNull(IoTCaseApi.class.getAnnotation(McpDomain.class));
        assertNotNull(IoTOperationsApi.class.getAnnotation(McpDomain.class));
    }

    @Test
    void domainNamesAreCorrect() {
        assertEquals("iot/devices", IoTDeviceApi.class.getAnnotation(McpDomain.class).value());
        assertEquals("iot/situations", IoTSituationApi.class.getAnnotation(McpDomain.class).value());
        assertEquals("iot/situations/suppressions", IoTSuppressionApi.class.getAnnotation(McpDomain.class).value());
        assertEquals("iot/cases", IoTCaseApi.class.getAnnotation(McpDomain.class).value());
        assertEquals("iot/ops", IoTOperationsApi.class.getAnnotation(McpDomain.class).value());
    }

    @Test
    void parameterNamesPreserved() throws NoSuchMethodException {
        var params = IoTDeviceApi.class.getMethod("listDevices", String.class, String.class, Boolean.class, String.class)
                .getParameters();
        assertEquals("deviceClass", params[0].getName());
        assertEquals("providerId", params[1].getName());
        assertEquals("tenancyId", params[3].getName());
    }
}
```

- [ ] **Step 7: Build and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webapp-api -f /Users/mdproctor/claude/casehub/slots/194/iot/pom.xml`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 8: Commit**

```
feat(#105): define 5 @McpDomain SPI interfaces for iot webapp

IoTDeviceApi (4 methods), IoTSituationApi (7 methods),
IoTSuppressionApi (3 methods), IoTCaseApi (6 methods),
IoTOperationsApi (7 methods). Platform annotations per #295.
Placeholder methods use default + NotImplementedException.

Refs casehubio/iot#105
```

---

## Batch 2: Impl Beans (webapp)

### Task 3: DefaultIoTDeviceApi + DefaultIoTOperationsApi

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTDeviceApi.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTOperationsApi.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/service/DefaultIoTDeviceApiTest.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/service/DefaultIoTOperationsApiTest.java`

**Interfaces:**
- Consumes: `IoTDeviceApi`, `IoTOperationsApi` from Task 2
- Consumes: View records from Task 1
- Consumes: `DeviceRegistry`, `DeviceProvider`, `DeviceStateHistoryProvider`, `BridgeConnectionRegistry`, `BridgeAuditStore`, `CurrentPrincipal` from existing codebase
- Produces: CDI beans injected by generated REST resources (Task 6)

- [ ] **Step 1: Write failing test for DefaultIoTDeviceApi**

The test uses `@QuarkusTest` with the in-memory providers. Check existing test
infrastructure in `webapp/src/test/` for the test profile and available mock beans.
The test should verify that `listDevices()` filters by tenancy and maps to `DeviceResponse`.

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.webapp.rest.DeviceResponse;
import io.casehub.iot.webapp.spi.IoTDeviceApi;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class DefaultIoTDeviceApiTest {

    @Inject
    IoTDeviceApi deviceApi;

    @Test
    void listDevices_returnsFilteredList() {
        var devices = deviceApi.listDevices(null, null, null, "default");
        assertNotNull(devices);
    }

    @Test
    void getDevice_notFound_throws() {
        assertThrows(Exception.class, () -> deviceApi.getDevice("nonexistent", "default"));
    }
}
```

- [ ] **Step 2: Implement DefaultIoTDeviceApi**

Move business logic from `DeviceResource` (lines 72-208). Key delegation:
- `listDevices()` → `deviceRegistry.findAll()` + tenancy filter + map to DeviceResponse
- `getDevice()` → `deviceRegistry.findById()` + map to DeviceResponse
- `dispatchCommand()` → `deviceRegistry.findById()` + provider lookup + `provider.dispatch()`
- `getDeviceHistory()` → `historyProvider.findHistory()` + map to StateHistoryView

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.api.DeviceEntity;
import io.casehub.iot.api.DeviceRegistry;
import io.casehub.iot.api.DeviceStateHistoryProvider;
import io.casehub.iot.api.spi.DeviceCommand;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.webapp.rest.CommandRequest;
import io.casehub.iot.webapp.rest.CommandResponse;
import io.casehub.iot.webapp.rest.DeviceResponse;
import io.casehub.iot.webapp.spi.IoTDeviceApi;
import io.casehub.iot.webapp.view.StateHistoryView;
import io.casehub.platform.api.CurrentPrincipal;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class DefaultIoTDeviceApi implements IoTDeviceApi {

    @Inject DeviceRegistry deviceRegistry;
    @Inject Instance<DeviceProvider> providers;
    @Inject CurrentPrincipal principal;
    @Inject DeviceStateHistoryProvider historyProvider;

    @Override
    public List<DeviceResponse> listDevices(String deviceClass, String providerId,
                                             Boolean available, String tenancyId) {
        // Move logic from DeviceResource.list() lines 72-103
        // Filter by tenancyId, deviceClass, providerId, available
        // Map DeviceEntity → DeviceResponse
        return deviceRegistry.findAll().stream()
                .filter(d -> tenancyId == null || tenancyId.equals(d.tenancyId()))
                .filter(d -> deviceClass == null || deviceClass.equals(d.deviceClass()))
                .filter(d -> providerId == null || providerId.equals(d.providerId()))
                .filter(d -> available == null || available.equals(d.available()))
                .map(this::toDeviceResponse)
                .toList();
    }

    @Override
    public DeviceResponse getDevice(String deviceId, String tenancyId) {
        // Move logic from DeviceResource.get() lines 105-134
        return deviceRegistry.findById(deviceId, tenancyId)
                .map(this::toDeviceResponse)
                .orElseThrow(() -> new jakarta.ws.rs.NotFoundException("Device not found: " + deviceId));
    }

    @Override
    public CommandResponse dispatchCommand(String deviceId, CommandRequest command, String tenancyId) {
        // Move logic from DeviceResource.dispatch() lines 136-181
        // Lookup device, find provider, dispatch command, map result
        var device = deviceRegistry.findById(deviceId, tenancyId)
                .orElseThrow(() -> new jakarta.ws.rs.NotFoundException("Device not found: " + deviceId));
        var provider = providers.stream()
                .filter(p -> p.providerId().equals(device.providerId()))
                .findFirst()
                .orElseThrow(() -> new IllegalStateException("No provider for device: " + deviceId));
        var correlationId = UUID.randomUUID().toString();
        var result = provider.dispatch(new DeviceCommand(deviceId, command.action(), command.parameters(), correlationId));
        return new CommandResponse(deviceId, command.action(), result, correlationId);
    }

    @Override
    public List<StateHistoryView> getDeviceHistory(String deviceId, Instant from, Instant to,
                                                     Integer limit, String tenancyId) {
        // Move logic from DeviceResource.history() lines 183-207
        return historyProvider.findHistory(deviceId, tenancyId, from, to, limit != null ? limit : 100)
                .stream()
                .map(h -> new StateHistoryView(h.deviceId(), h.deviceClass(),
                        h.stateSnapshot(), h.changedCapabilities(), h.occurredAt()))
                .toList();
    }

    private DeviceResponse toDeviceResponse(DeviceEntity d) {
        return new DeviceResponse(d.deviceId(), d.providerId(), d.tenancyId(),
                d.deviceClass(), d.name(), d.location(), d.available(),
                d.capabilities(), d.lastUpdated());
    }
}
```

Note: the exact method signatures on `DeviceRegistry`, `DeviceProvider`, `DeviceCommand`,
`DeviceStateHistoryProvider` must be verified against the actual api module classes.
The implementer should use `ide_find_class` + `ide_file_structure` to confirm parameter
names and types before writing the delegation code.

- [ ] **Step 3: Implement DefaultIoTOperationsApi**

Move business logic from `ProviderResource` (105 lines), `BridgeResource` (165 lines),
and `HealthResource` (108 lines). All delegation — no complex business logic.

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.api.DeviceRegistry;
import io.casehub.iot.api.spi.DeviceProvider;
import io.casehub.iot.bridge.server.BridgeConnectionRegistry;
import io.casehub.iot.bridge.persistence.BridgeAuditStore;
import io.casehub.iot.bridge.persistence.BridgeAuditQuery;
import io.casehub.iot.webapp.rest.HealthOverviewResponse;
import io.casehub.iot.webapp.spi.IoTOperationsApi;
import io.casehub.iot.webapp.view.AuditTrailView;
import io.casehub.iot.webapp.view.BridgeConnectionsView;
import io.casehub.iot.webapp.view.ProviderStatusView;
import io.casehub.iot.webapp.view.RefreshResultView;
import io.casehub.platform.api.CurrentPrincipal;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.List;

@ApplicationScoped
public class DefaultIoTOperationsApi implements IoTOperationsApi {

    @Inject Instance<DeviceProvider> providers;
    @Inject DeviceRegistry deviceRegistry;
    @Inject BridgeConnectionRegistry connectionRegistry;
    @Inject BridgeAuditStore auditStore;
    @Inject CurrentPrincipal principal;

    @Override
    public List<ProviderStatusView> listProviders(String tenancyId) {
        // Move from ProviderResource.list() lines 36-54
        return providers.stream()
                .map(p -> new ProviderStatusView(p.providerId(), p.status().name(),
                        (int) deviceRegistry.findAll().stream()
                                .filter(d -> p.providerId().equals(d.providerId())).count()))
                .toList();
    }

    @Override
    public ProviderStatusView getProvider(String providerId, String tenancyId) {
        // Move from ProviderResource.get() lines 56-76
        return providers.stream()
                .filter(p -> p.providerId().equals(providerId))
                .findFirst()
                .map(p -> new ProviderStatusView(p.providerId(), p.status().name(),
                        (int) deviceRegistry.findAll().stream()
                                .filter(d -> p.providerId().equals(d.providerId())).count()))
                .orElseThrow(() -> new jakarta.ws.rs.NotFoundException("Provider not found: " + providerId));
    }

    @Override
    public RefreshResultView refreshAllProviders(String tenancyId) {
        deviceRegistry.refresh();
        return new RefreshResultView("All providers refreshed");
    }

    @Override
    public RefreshResultView refreshProvider(String providerId, String tenancyId) {
        deviceRegistry.refresh(providerId);
        return new RefreshResultView("Provider refreshed: " + providerId);
    }

    @Override
    public BridgeConnectionsView getBridgeConnections(String tenancyId) {
        // Move from BridgeResource.connections() lines 46-76
        var tenancies = connectionRegistry.connectedTenancies().stream()
                .map(t -> new BridgeConnectionsView.TenancyConnection(t, Instant.now()))
                .toList();
        return new BridgeConnectionsView(connectionRegistry.hasAnyConnection(), tenancies);
    }

    @Override
    public AuditTrailView getBridgeAudit(String eventType, String deviceId, String correlationId,
                                          Instant from, Instant to, Integer offset, Integer limit,
                                          String tenancyId) {
        // Move from BridgeResource.audit() lines 78-133
        var query = BridgeAuditQuery.builder()
                .eventType(eventType).deviceId(deviceId).correlationId(correlationId)
                .from(from).to(to)
                .offset(offset != null ? offset : 0)
                .limit(limit != null ? limit : 50)
                .build();
        var result = auditStore.query(query);
        var records = result.records().stream()
                .map(r -> new AuditTrailView.AuditRecord(r.eventType(), r.deviceId(),
                        r.correlationId(), r.payload(), r.occurredAt()))
                .toList();
        return new AuditTrailView(records, result.totalCount(), query.offset(), query.limit());
    }

    @Override
    public HealthOverviewResponse getHealthOverview(String tenancyId) {
        // Move from HealthResource.overview() lines 58-106
        // Aggregate provider status, bridge connections, placeholder counts
        var providerStatuses = providers.stream()
                .map(p -> new HealthOverviewResponse.ProviderStatus(p.providerId(), p.status().name(),
                        (int) deviceRegistry.findAll().stream()
                                .filter(d -> p.providerId().equals(d.providerId())).count()))
                .toList();
        var bridgeConns = connectionRegistry.connectedTenancies().stream()
                .map(t -> new HealthOverviewResponse.BridgeConnection(t, Instant.now()))
                .toList();
        return new HealthOverviewResponse(providerStatuses, bridgeConns, 0, 0, 0);
    }
}
```

Note: verify `BridgeAuditStore.query()` return type, `BridgeAuditQuery.builder()` API,
and `HealthOverviewResponse` nested types against the actual codebase using `ide_file_structure`.

- [ ] **Step 4: Run tests and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webapp -f /Users/mdproctor/claude/casehub/slots/194/iot/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```
feat(#105): add DefaultIoTDeviceApi and DefaultIoTOperationsApi impl beans

Device API delegates to DeviceRegistry, providers, and history provider.
Operations API delegates to providers, bridge registry, bridge audit store.

Refs casehubio/iot#105
```

---

### Task 4: DefaultIoTSituationApi + DefaultIoTSuppressionApi

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTSituationApi.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTSuppressionApi.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/service/DefaultIoTSituationApiTest.java`

**Interfaces:**
- Consumes: `IoTSituationApi`, `IoTSuppressionApi` from Task 2
- Consumes: `EntityManager`, `CaseInstanceCache`, `CaseDefinitionRegistry`, `IoTCbrRetrievalService`, `DismissalRecorder`, `SituationStore`, `CurrentPrincipal`

- [ ] **Step 1: Implement DefaultIoTSituationApi**

Most complex impl — JPQL queries, sealed type mapping (`mapChainMode`, `mapTriggerMode`),
CDI event firing. Move ALL business logic from `SituationResource` lines 81-366.

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.webapp.rest.DismissRequest;
import io.casehub.iot.webapp.rest.SituationDefinitionRequest;
import io.casehub.iot.webapp.rest.SituationSuggestionsResponse;
import io.casehub.iot.webapp.spi.IoTSituationApi;
import io.casehub.iot.webapp.view.SituationDefinitionView;
import io.casehub.platform.api.CurrentPrincipal;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import java.util.List;

@ApplicationScoped
public class DefaultIoTSituationApi implements IoTSituationApi {

    @Inject EntityManager em;
    @Inject CurrentPrincipal principal;
    // Move these injections from SituationResource:
    // @Inject CaseInstanceCache caseInstanceCache;
    // @Inject CaseDefinitionRegistry caseDefinitionRegistry;
    // @Inject IoTCbrRetrievalService retrievalService;
    // @Inject DismissalRecorder dismissalRecorder;
    // @Inject SituationStore situationStore;
    // @Inject Event<SituationChangeEvent> changeEvent;

    // Move ALL method implementations from SituationResource lines 131-366
    // including helper methods: mapRequestToDomain(), mapChainMode(), mapTriggerMode()

    @Override
    public List<SituationDefinitionView> listDefinitions(String tenancyId) {
        // Move from SituationResource.listDefinitions() line 131
        // em.createQuery() JPQL → map to SituationDefinitionView
    }

    @Override
    @Transactional
    public SituationDefinitionView createDefinition(SituationDefinitionRequest request,
                                                     String tenancyId) {
        // Move from SituationResource.createDefinition() line 163
        // mapRequestToDomain() + em.persist()
    }

    @Override
    @Transactional
    public SituationDefinitionView updateDefinition(String situationId,
                                                     SituationDefinitionRequest request,
                                                     String tenancyId) {
        // Move from SituationResource.updateDefinition() line 213
        // em.remove() + em.persist() (immutable entity pattern)
    }

    @Override
    @Transactional
    public void deleteDefinition(String situationId, String tenancyId) {
        // Move from SituationResource.deleteDefinition() line 266
        // em.createQuery() DELETE JPQL
    }

    // listActive() is default method — not overridden (placeholder)

    @Override
    public SituationSuggestionsResponse getSuggestions(String situationId, String tenancyId) {
        // Move from SituationResource.getSuggestions() line 283
        // caseInstanceCache + caseDefinitionRegistry + retrievalService
    }

    @Override
    @Transactional
    public void dismissSituation(String correlationKey, DismissRequest request, String tenancyId) {
        // Move from SituationResource.dismissSituation() line 347
        // situationStore.find() + dismissalRecorder + situationStore.remove() + changeEvent.fireAsync()
    }
}
```

The implementer must read `SituationResource.java` in full and copy/adapt each method.
Use `ide_file_structure` on `SituationResource` to identify all helper methods that need
to move.

- [ ] **Step 2: Implement DefaultIoTSuppressionApi**

Move JPQL queries from `SituationResource` lines 374-465.

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.webapp.rest.SuppressionHistoryResponse;
import io.casehub.iot.webapp.rest.SuppressionStatsResponse;
import io.casehub.iot.webapp.spi.IoTSuppressionApi;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class DefaultIoTSuppressionApi implements IoTSuppressionApi {

    @Inject EntityManager em;
    // @Inject DismissalRecorder dismissalRecorder; (for overrideSuppression)

    @Override
    public List<SuppressionHistoryResponse> listSuppressions(String situationId, Instant since,
                                                              Boolean includeOverridden,
                                                              String tenancyId) {
        // Move from SituationResource.listSuppressions() line 374
        // Dynamic JPQL query with optional filters
    }

    @Override
    @Transactional
    public void overrideSuppression(UUID id, String tenancyId) {
        // Move from SituationResource.overrideSuppression() line 409
        // em.find() + entry.markOverridden() + dismissalRecorder.recordCaseOutcome()
    }

    @Override
    public SuppressionStatsResponse getSuppressionStats(String situationId, String tenancyId) {
        // Move from SituationResource.getSuppressionStats() line 431
        // Multiple COUNT JPQL queries + safetyCritical check
    }
}
```

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webapp -f /Users/mdproctor/claude/casehub/slots/194/iot/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
feat(#105): add DefaultIoTSituationApi and DefaultIoTSuppressionApi impl beans

Situation API handles definition CRUD, suggestions, dismissal.
Suppression API handles history, overrides, stats.
Business logic moved from SituationResource (JPQL, sealed type mapping).

Refs casehubio/iot#105
```

---

### Task 5: DefaultIoTCaseApi

**Files:**
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/service/DefaultIoTCaseApi.java`
- Create: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/NotImplementedExceptionMapper.java`
- Test: `webapp/src/test/java/io/casehub/iot/webapp/app/service/DefaultIoTCaseApiTest.java`

**Interfaces:**
- Consumes: `IoTCaseApi` from Task 2
- Consumes: `CaseInstanceCache`, `CaseDefinitionRegistry`, `IoTCbrRetrievalService`, `CaseQueueService`, `CaseQueueEntryStore`, `SubjectViewStore`, `CurrentPrincipal`

- [ ] **Step 1: Create NotImplementedExceptionMapper**

```java
package io.casehub.iot.webapp.app.rest;

import io.casehub.iot.webapp.NotImplementedException;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;
import java.util.Map;

@Provider
public class NotImplementedExceptionMapper implements ExceptionMapper<NotImplementedException> {
    @Override
    public Response toResponse(NotImplementedException e) {
        return Response.status(501)
                .entity(Map.of("error", "Not implemented", "operation", e.operation()))
                .build();
    }
}
```

- [ ] **Step 2: Implement DefaultIoTCaseApi**

Move logic from `CaseResource` (238 lines) and `ResolutionQueueResource` (223 lines).
`listCases()` and `getCase()` are NOT overridden — inherited defaults throw `NotImplementedException`.

```java
package io.casehub.iot.webapp.app.service;

import io.casehub.iot.webapp.resolution.QueueEntryDetail;
import io.casehub.iot.webapp.resolution.QueueEntrySummary;
import io.casehub.iot.webapp.spi.IoTCaseApi;
import io.casehub.iot.webapp.view.SuggestionView;
import io.casehub.platform.api.CurrentPrincipal;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class DefaultIoTCaseApi implements IoTCaseApi {

    // Move injections from CaseResource + ResolutionQueueResource:
    // @Inject CaseInstanceCache caseInstanceCache;
    // @Inject CaseDefinitionRegistry caseDefinitionRegistry;
    // @Inject IoTCbrRetrievalService retrievalService;
    // @Inject CaseQueueService queueService;
    // @Inject CaseQueueEntryStore entryStore;
    // @Inject SubjectViewStore viewStore;
    @Inject CurrentPrincipal principal;

    // @PostConstruct from ResolutionQueueResource.init() — view name mapping

    // listCases() — NOT overridden (placeholder, inherited default throws NotImplementedException)
    // getCase() — NOT overridden (placeholder, inherited default throws NotImplementedException)

    @Override
    public SuggestionView getCaseSuggestions(UUID caseId, String tenancyId) {
        // Move from CaseResource.getSuggestions() line 101
        // caseInstanceCache.get() + caseDefinitionRegistry + retrievalService + extractFeatures()
    }

    @Override
    public void acceptSuggestion(UUID caseId, UUID pastCaseId, String tenancyId) {
        // Move from CaseResource.acceptSuggestion() line 129
        // Match validation, plan steps extraction, context mutation
    }

    @Override
    public List<QueueEntrySummary> listResolutionQueue(String view, String status, String tenancyId) {
        // Move from ResolutionQueueResource.list() line 73
        // resolveViewIds() + queueService.findPending()/findByView() + filter + toSummary()
    }

    @Override
    public QueueEntryDetail getResolutionQueueEntry(UUID entryId, String tenancyId) {
        // Move from ResolutionQueueResource.detail() line 107
        // entryStore.findById() + caseCache.get() + extractWorkingContext() + retrievalService
    }

    // Move helper methods: extractFeatures(), toSummary(), extractWorkingContext(), resolveViewIds()
}
```

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webapp -f /Users/mdproctor/claude/casehub/slots/194/iot/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
feat(#105): add DefaultIoTCaseApi impl bean and NotImplementedExceptionMapper

Case API handles suggestions, acceptance, and resolution queue.
listCases/getCase remain unimplemented (default methods → 501).
Exception mapper converts NotImplementedException → HTTP 501.

Refs casehubio/iot#105
```

---

## Batch 3: APT Wiring + Switchover (webapp)

### Task 6: APT wiring, verification, and resource deletion

**Files:**
- Modify: `webapp/pom.xml` (add `annotationProcessorPaths` and `compilerArgs`)
- Delete: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/DeviceResource.java` (use `ide_refactor_safe_delete`)
- Delete: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/CaseResource.java`
- Delete: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/SituationResource.java`
- Delete: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/ProviderResource.java`
- Delete: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/BridgeResource.java`
- Delete: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/HealthResource.java`
- Delete: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/ResolutionQueueResource.java`
- Delete: `webapp/src/main/java/io/casehub/iot/webapp/app/rest/SuggestionResponse.java`
- Keep: `KpiResource.java`, `WorkItemResource.java`, `DeviceSseResource.java`

**Interfaces:**
- Consumes: All SPI interfaces (Task 2) via Jandex index
- Consumes: All impl beans (Tasks 3-5) as CDI beans

- [ ] **Step 1: Add APT wiring to webapp pom.xml**

Add to `maven-compiler-plugin` configuration in `webapp/pom.xml`:

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
                <artifactId>casehub-iot-webapp-api</artifactId>
                <version>${project.version}</version>
            </path>
        </annotationProcessorPaths>
        <compilerArgs>
            <arg>-AdomainFilter=iot/devices,iot/situations,iot/situations/suppressions,iot/cases,iot/ops</arg>
            <arg>-AgenerateGraphQL=false</arg>
        </compilerArgs>
    </configuration>
</plugin>
```

- [ ] **Step 2: Build to verify APT generation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl webapp -f /Users/mdproctor/claude/casehub/slots/194/iot/pom.xml`

Check generated sources:
```bash
ls /Users/mdproctor/claude/casehub/slots/194/iot/webapp/target/generated-sources/annotations/
```

Expected: Generated REST resource classes for each domain
(e.g., `IoTDeviceApiResource.java`, `IoTSituationApiResource.java`, etc.)

- [ ] **Step 3: Verify no path conflicts**

Before deleting hand-written resources, build with BOTH present to confirm
the APT skip detection works (hand-written resources have no `@McpDomain`,
so paths may conflict). If paths conflict, delete the hand-written resources
FIRST, then rebuild.

- [ ] **Step 4: Delete hand-written resources**

Use `ide_refactor_safe_delete` for each file to check for references:

Delete in order:
1. `DeviceResource.java`
2. `CaseResource.java`
3. `SituationResource.java`
4. `ProviderResource.java`
5. `BridgeResource.java`
6. `HealthResource.java`
7. `ResolutionQueueResource.java`
8. `SuggestionResponse.java`

Check that kept resources (`KpiResource`, `WorkItemResource`, `DeviceSseResource`)
don't reference any deleted types. If they do, update references.

- [ ] **Step 5: Full build and test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl webapp -f /Users/mdproctor/claude/casehub/slots/194/iot/pom.xml`
Expected: BUILD SUCCESS, all tests pass

If tests reference deleted resources, update them to use the generated
endpoints (same URLs, same request/response shapes).

- [ ] **Step 6: Verify generated endpoint paths match original**

Manually verify (or write a test) that the generated REST endpoints produce
the same URL paths as the deleted hand-written resources. Key paths:
- GET /api/iot/devices → was GET /api/devices
- GET /api/iot/situations/definitions → was GET /api/situations/definitions
- GET /api/iot/cases → was GET /api/cases
- GET /api/iot/ops/providers → was GET /api/providers

Note: generated paths will be `/api/iot/<domain>/<method-kebab-case>`.
These may differ from the original hand-written paths. If path compatibility
is required, use `@RestPath` overrides on SPI methods (Task 2).

- [ ] **Step 7: Commit**

```
feat(#105): wire APT generation, delete hand-written resources

Generated REST endpoints replace 7 hand-written resource classes.
Kept: KpiResource, WorkItemResource, DeviceSseResource.
Net deletion: ~1000 lines of hand-written REST code.

Refs casehubio/iot#105
```

---

## References

- [2026-09-15-iot-mcpdomain-spi-design.md] — design spec this plan implements
- [decisions.md] — 7 design decisions (D1-D7)
- [platform#295 decisions] — annotation model architecture (D1, D3, D8)
- [WorkItemApi.java] — established SPI pattern reference
- [DefaultWorkItemApi.java] — established impl bean pattern reference
- [work rest/pom.xml] — APT wiring reference
- [GE-20260914-f53be7] — SPI must be in dependency JAR
- [GE-20260914-3854b8] — domainFilter required
- [GE-20260914-714a71] — -parameters flag required
- [casehubio/iot#105] — focal issue
- [casehubio/platform#300] — generator improvements (slot scope)
