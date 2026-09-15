# Design: casehub-iot @McpDomain SPI Migration

**Issue:** casehubio/iot#105
**Slot:** 194 (queue position 4/17)
**Date:** 2026-09-15

## Summary

Migrate casehub-iot's hand-written REST endpoints to the @McpDomain SPI
generation pattern. Define SPI interfaces in `webapp-api`, evolve existing
DTOs into view records, generate REST + GraphQL + MCP endpoints from the
SPIs. Same pattern as ledger#207, engine#1095, work#400.

## Scope

**In scope:**
- 5 SPI interfaces (28 methods) covering devices, situations, suppressions,
  cases/resolution queue, and operations
- View records evolved from existing webapp-api DTOs
- Impl beans in `webapp/` delegating to existing services
- APT wiring in webapp pom.xml
- `NotImplementedException` + exception mapper for placeholder endpoints
- MCP coexistence strategy (generated alongside curated)

**Out of scope:**
- SSE/streaming endpoints (stay hand-written per issue #105)
- KPI endpoints (thin aggregations, path doesn't fit one domain — stay hand-written)
- WorkItem endpoints (placeholder, real API is in casehub-work)
- Generator changes (established annotation model, no JAX-RS uplift)

**Endpoints staying hand-written (6):**
- `KpiResource` — GET /api/devices/kpi, GET /api/health/kpi
- `WorkItemResource` — GET /api/workitems, POST /{id}/claim, POST /{id}/complete, GET /{id}/prediction
- `DeviceSseResource` — GET /api/devices/stream (SSE)

## Module Placement

SPI interfaces and view records go in **`webapp-api`** (`io.casehub.iot.webapp`).

Rationale: the SPI interfaces define the webapp's generated REST/GraphQL/MCP
surface — a webapp concern, not a foundation concern. ARC42STORIES boundary
rules are clear: `api/` owns device type vocabulary, provider SPI, and event
model. The webapp API shape is not in that list. `webapp-api` already has
Jandex configured and depends on `platform-api` transitively.

The `api/` module stays focused on device types, provider SPIs, and event
model — the foundation layer consumed by casehub-life and future apps.

## Annotation Model

Established platform annotations per platform#295 decisions:

```java
@McpDomain("iot/devices")
public interface IoTDeviceApi {

    @PlatformQuery("List devices with optional filtering")
    List<DeviceView> listDevices(
        String deviceClass, String providerId, Boolean available,
        @ContextParam("tenancyId") String tenancyId);

    @PlatformQuery("Get device by ID")
    DeviceView getDevice(
        @PathParam UUID deviceId,
        @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Dispatch a command to a device")
    @RestStatus(201)
    CommandView dispatchCommand(
        @PathParam UUID deviceId,
        CommandRequest command,
        @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Delete a situation definition")
    @RestMethod(HttpMethod.DELETE)
    @RestPath("/{situationId}")
    void deleteDefinition(
        @PathParam String situationId,
        @ContextParam("tenancyId") String tenancyId);
}
```

JAX-RS annotations (`@GET`, `@POST`, `@Path`) are NOT used on SPI interfaces.
Quarkus discovers and processes JAX-RS annotations on interfaces and their
implementing CDI beans, causing path conflicts with generated resources.
Platform annotations are invisible to the runtime scanner.

## SPI Interfaces

### 1. IoTDeviceApi — `iot/devices` (4 methods)

| Method | Type | Description |
|--------|------|-------------|
| `listDevices(deviceClass, providerId, available, tenancyId)` | query | List devices with optional filtering |
| `getDevice(deviceId, tenancyId)` | query | Get device by ID |
| `dispatchCommand(deviceId, command, tenancyId)` | mutation | Dispatch a command to a device |
| `getDeviceHistory(deviceId, from, to, limit, tenancyId)` | query | Get device state history |

### 2. IoTSituationApi — `iot/situations` (7 methods)

| Method | Type | REST override | Description |
|--------|------|---------------|-------------|
| `listDefinitions(tenancyId)` | query | | List situation definitions |
| `createDefinition(request, tenancyId)` | mutation | | Create a situation definition |
| `updateDefinition(situationId, request, tenancyId)` | mutation | PUT | Update a situation definition |
| `deleteDefinition(situationId, tenancyId)` | mutation | DELETE | Delete a situation definition |
| `listActive(tenancyId)` | query | | List active situations (placeholder — default method) |
| `getSuggestions(situationId, tenancyId)` | query | | Get resolution suggestions for a situation |
| `dismissSituation(correlationKey, request, tenancyId)` | mutation | | Dismiss an active situation |

### 3. IoTSuppressionApi — `iot/situations/suppressions` (3 methods)

| Method | Type | Description |
|--------|------|-------------|
| `listSuppressions(situationId, since, includeOverridden, tenancyId)` | query | List suppression history |
| `overrideSuppression(id, tenancyId)` | mutation | Override a suppression |
| `getSuppressionStats(situationId, tenancyId)` | query | Get suppression statistics |

### 4. IoTCaseApi — `iot/cases` (6 methods)

| Method | Type | Description |
|--------|------|-------------|
| `listCases(status, situationId, from, to, tenancyId, offset, limit)` | query | List cases (placeholder — default method) |
| `getCase(caseId, tenancyId)` | query | Get case by ID (placeholder — default method) |
| `getCaseSuggestions(caseId, tenancyId)` | query | Get case resolution suggestions |
| `acceptSuggestion(caseId, pastCaseId, tenancyId)` | mutation | Accept a resolution suggestion |
| `listResolutionQueue(view, status, tenancyId)` | query | List resolution queue entries |
| `getResolutionQueueEntry(entryId, tenancyId)` | query | Get resolution queue entry detail |

### 5. IoTOperationsApi — `iot/ops` (7 methods)

| Method | Type | Description |
|--------|------|-------------|
| `listProviders(tenancyId)` | query | List IoT providers with status |
| `getProvider(providerId, tenancyId)` | query | Get provider by ID |
| `refreshAllProviders(tenancyId)` | mutation | Refresh all providers |
| `refreshProvider(providerId, tenancyId)` | mutation | Refresh a specific provider |
| `getBridgeConnections(tenancyId)` | query | List bridge connections |
| `getBridgeAudit(eventType, deviceId, correlationId, from, to, offset, limit, tenancyId)` | query | Query bridge audit trail |
| `getHealthOverview(tenancyId)` | query | Get system health overview |

## View Records

Evolve existing webapp-api DTOs in-place. No parallel type hierarchy.

**Existing DTOs to evolve (in `webapp-api`):**
- `DeviceResponse` → curate fields for generated API (rename if needed)
- `CommandRequest` / `CommandResponse` → keep as-is (already clean)
- `HealthOverviewResponse` → curate nested types
- `SituationDefinitionRequest` → keep as-is
- `SituationSuggestionsResponse` → flatten nested `CaseSuggestions`
- `DismissRequest` → keep as-is
- `SuppressionHistoryResponse` / `SuppressionStatsResponse` → keep as-is
- `QueueEntrySummary` / `QueueEntryDetail` → keep as-is

**New view records needed:**
- `StateHistoryView` — for `getDeviceHistory()` return type (currently inline `StateHistoryResponse` inner record in DeviceResource)
- `SituationDefinitionView` — for definition query returns (currently inline inner record)
- `ActiveSituationView` — for `listActive()` return type (currently inline inner record)
- `ProviderStatusView` — for provider query returns (currently inline inner record)
- `RefreshResultView` — for refresh mutation returns (currently inline inner record)
- `BridgeConnectionsView` / `AuditTrailView` — for bridge query returns (currently inline inner records)
- `CaseSummaryView` / `CaseDetailView` — for case query returns (currently inline inner records, placeholder)
- `SuggestionView` — for case suggestion query returns (currently a separate file `SuggestionResponse`)

Inner records in resource classes are promoted to top-level records in
`webapp-api`. This is the curation step — fields are reviewed and
internal/operational fields are excluded.

## Placeholder Endpoints — NotImplementedException

Three methods are placeholder (TODO in current resource code):
- `IoTCaseApi.listCases()` — returns `List.of()`
- `IoTCaseApi.getCase()` — throws `NotFoundException`
- `IoTSituationApi.listActive()` — returns `List.of()`

These use Java `default` methods that throw `NotImplementedException`:

```java
@McpDomain("iot/cases")
public interface IoTCaseApi {

    // Implemented — abstract, impl bean MUST provide
    @PlatformQuery("Get case resolution suggestions")
    SuggestionView getCaseSuggestions(@PathParam UUID caseId,
                                     @ContextParam("tenancyId") String tenancyId);

    // Not yet implemented — default, impl bean can skip
    @PlatformQuery("List cases with optional filtering")
    @PaginatedResponse
    default CasePage listCases(String status, String situationId,
                               Instant from, Instant to,
                               @ContextParam("tenancyId") String tenancyId,
                               Integer offset, Integer limit) {
        throw new NotImplementedException("listCases");
    }
}
```

`NotImplementedException` goes in `platform-api` (platform#300 scope).
A JAX-RS exception mapper converts it to HTTP 501 with a standard
JSON body: `{"error": "Not implemented", "operation": "listCases"}`.

When the implementation is ready: override the default method in the
impl bean. One change, correct compiler enforcement (abstract = must
implement, default = can defer).

## Impl Beans

5 `@ApplicationScoped` CDI beans in `webapp/src/.../app/service/`:

| Bean | SPI | Delegates to |
|------|-----|-------------|
| `DefaultIoTDeviceApi` | `IoTDeviceApi` | `DeviceRegistry`, `DeviceProvider`, `DeviceStateHistoryProvider` |
| `DefaultIoTSituationApi` | `IoTSituationApi` | `EntityManager` (JPQL), `CaseInstanceCache`, `IoTCbrRetrievalService`, `SituationStore`, `DismissalRecorder` |
| `DefaultIoTSuppressionApi` | `IoTSuppressionApi` | `EntityManager` (JPQL) |
| `DefaultIoTCaseApi` | `IoTCaseApi` | `CaseInstanceCache`, `IoTCbrRetrievalService`, `CaseQueueService`, `CaseQueueEntryStore` |
| `DefaultIoTOperationsApi` | `IoTOperationsApi` | `Instance<DeviceProvider>`, `DeviceRegistry`, `BridgeConnectionRegistry`, `BridgeAuditStore`, `EntityManager` |

Business logic moves from resource classes to impl beans. Resources are
deleted after APT wiring is verified.

## APT Wiring

`webapp/pom.xml` — `maven-compiler-plugin` configuration:

```xml
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
```

`webapp-api/pom.xml` — compiler flag for parameter names:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <parameters>true</parameters>
    </configuration>
</plugin>
```

**Key gotchas (from garden):**
- GE-20260914-f53be7: SPI interfaces must be in a pre-compiled dependency JAR (webapp-api is compiled before webapp — correct)
- GE-20260914-3854b8: `-AdomainFilter` required to prevent generating for other domains on classpath
- GE-20260914-714a71: `-parameters` flag required on the SPI module so APT sees real parameter names

## MCP Surface Strategy

Coexistence then replacement. The existing curated MCP module
(`casehub-iot-mcp` with `IoTDeviceMcpTool`) remains the production
MCP surface during migration. The generated MCP surface from SPI
interfaces is deployed alongside for verification. Once verified,
the curated module is deprecated.

Tool name disambiguation: curated tools use flat names
(`iot_get_devices`), generated tools use domain-scoped names
(`iot/devices/listDevices`).

## Deleted After Migration

| File | Lines | Reason |
|------|-------|--------|
| `DeviceResource.java` | ~120 | Replaced by generated `iot/devices` resource |
| `CaseResource.java` | ~80 | Replaced by generated `iot/cases` resource |
| `SituationResource.java` | ~200 | Replaced by generated `iot/situations` + `iot/situations/suppressions` resources |
| `ProviderResource.java` | ~60 | Replaced by generated `iot/ops` resource |
| `BridgeResource.java` | ~70 | Replaced by generated `iot/ops` resource |
| `HealthResource.java` | ~40 | Replaced by generated `iot/ops` resource |
| `ResolutionQueueResource.java` | ~50 | Replaced by generated `iot/cases` resource |
| `SuggestionResponse.java` | ~15 | Replaced by `SuggestionView` in webapp-api |

**Kept:**
- `KpiResource.java` — excluded from SPI (thin aggregation)
- `WorkItemResource.java` — excluded from SPI (placeholder, real API in casehub-work)
- `DeviceSseResource.java` — excluded from SPI (SSE stays hand-written)

## Batch Structure

Same 3-batch structure as engine/work:

**Batch 1: View records + SPI interfaces (webapp-api)**
- Promote inline inner records to top-level view records
- Curate existing DTOs (field review)
- Define 5 SPI interfaces with `@McpDomain` annotations
- Add `-parameters` compiler flag
- Ensure Jandex indexing is configured

**Batch 2: Impl beans (webapp)**
- 5 `@ApplicationScoped` CDI beans
- Move business logic from resource classes
- Entity → view record mapping

**Batch 3: APT wiring + switchover (webapp)**
- `maven-compiler-plugin` with `annotationProcessorPaths`
- `-AdomainFilter` for iot domains
- Delete 7 hand-written resource classes + 1 DTO file
- Verify generated endpoints match existing API shape
- Update any remaining references to deleted types

## Platform Dependencies (platform#300)

1. `NotImplementedException` in `platform-api` — new class
2. `NotImplementedExceptionMapper` in platform runtime support — JAX-RS exception mapper returning 501

These may be created as part of the iot migration if platform#300
hasn't delivered them yet. Temporary location: `webapp-api` for the
exception, `webapp` for the mapper. Promoted to platform later.

## References

- platform#295 decisions (annotation model: D1, D3, D8)
- GE-20260914-f53be7 — Jandex-based APT can't scan own module's index
- GE-20260914-3854b8 — APT domain filtering required in multi-module
- GE-20260914-714a71 — `-parameters` flag required on SPI module
- GE-20260612-4f9a47 — Quarkus discovers interfaces with JAX-RS annotations
- GE-20260625-88c860 — DeviceEntity.capabilities() null values
- GE-20260804-8b0fd6 — BroadcastProcessor SSE pattern
- engine api/view/ and work api/view/ — established view record pattern
- `McpDomainJandexScanner.java` (generator-common, slot 192) — shared scanner
- `GraphQLResolverProcessor.java` (graphql-generator) — Quarkus APT
