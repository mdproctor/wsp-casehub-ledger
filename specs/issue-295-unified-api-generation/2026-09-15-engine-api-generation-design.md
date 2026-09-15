# Unified API Generation — Migrate Engine to @McpDomain SPI

**Issue:** casehubio/engine#1095
**Parent epic:** casehubio/platform#295 (CLOSED — platform generator done)
**Depends on:** casehubio/ledger#207 (CLOSED — ledger migration complete, pattern established)
**Date:** 2026-09-15

## Summary

Replace hand-written REST endpoints and GraphQL resolvers in `casehub-engine`
with generated code from the platform `graphql-generator` APT. Define SPI
interfaces with `@McpDomain` + `@PlatformQuery`/`@PlatformMutation`/`@PlatformStream`
annotations. The generator produces REST resources, GraphQL resolvers, and MCP
tool registrations from the same interface.

**Scope:** 8 REST resources (24 endpoints), 3 GraphQL resolvers (13 methods),
2 SSE streams, 2 GraphQL subscriptions → 5 SPI interfaces + 4 streaming methods.

## Annotation Model

Identical to ledger migration (D1). All SPI interfaces use platform-specific
annotations from `io.casehub.platform.api.mcp`:

```java
@McpDomain("engine/cases")
public interface EngineCaseApi {

    @PlatformQuery("List case instances with optional filtering")
    @PaginatedResponse
    CasePage listCases(CaseStatus status, String namespace, String name,
                       Integer offset, Integer limit);

    @PlatformQuery("Get a case instance by ID")
    CaseInstanceView getCaseById(@PathParam UUID caseId);

    @PlatformMutation("Start a new case instance")
    @RestStatus(201)
    CaseInstanceView startCase(StartCaseRequest request);

    @PlatformStream("Live case event stream")
    Multi<CaseStreamEventView> caseStream(@PathParam UUID caseId);
}
```

## SPI Interfaces

Five interfaces organised by caller intent:

### EngineCaseApi (`engine/cases`) — 10 methods

| Method | Type | Description | Parameters |
|---|---|---|---|
| `listCases` | Query | List instances with filtering/pagination | `CaseStatus status`, `String namespace`, `String name`, `Integer offset`, `Integer limit` |
| `getCaseById` | Query | Get instance by ID | `@PathParam UUID caseId` |
| `startCase` | Mutation | Start a new case | `StartCaseRequest request` |
| `getCaseContext` | Query | Full case context | `@PathParam UUID caseId` |
| `getCaseContextPath` | Query | Context at a specific path | `@PathParam UUID caseId`, `String path` |
| `getPlanItems` | Query | Plan items for a case | `@PathParam UUID caseId` |
| `getGoals` | Query | Evaluate goals against live context | `@PathParam UUID caseId` |
| `caseStream` | Stream | SSE stream of case events | `@PathParam UUID caseId` |
| `caseLifecycle` | Stream | Live lifecycle events | `@PathParam UUID caseId` |
| `caseContextChange` | Stream | Live context change events | `@PathParam UUID caseId` |

### EngineCaseControlApi (`engine/control`) — 4 methods

| Method | Type | Description | Parameters |
|---|---|---|---|
| `suspendCase` | Mutation | Suspend a running case | `@PathParam UUID caseId`, `CaseControlRequest request` |
| `resumeCase` | Mutation | Resume a suspended case | `@PathParam UUID caseId`, `CaseControlRequest request` |
| `cancelCase` | Mutation | Cancel a case | `@PathParam UUID caseId`, `CaseControlRequest request` |
| `sendSignal` | Mutation | Send a signal to a case | `@PathParam UUID caseId`, `SendSignalRequest request` |

### EngineCaseDefinitionApi (`engine/definitions`) — 3 methods

| Method | Type | Description | Parameters |
|---|---|---|---|
| `listDefinitions` | Query | List registered definitions | `Integer offset`, `Integer limit` |
| `getDefinitionsByName` | Query | Definitions by namespace/name | `@PathParam String namespace`, `@PathParam String name` |
| `getDefinitionByKey` | Query | Specific definition by key | `@PathParam String namespace`, `@PathParam String name`, `@PathParam String version` |

### EngineEventLogApi (`engine/events`) — 1 method

| Method | Type | Description | Parameters |
|---|---|---|---|
| `getEventLog` | Query | Paginated/filtered event log | `@PathParam UUID caseId`, `Integer offset`, `Integer limit`, `List<String> eventTypes`, `List<String> streamTypes` |

### EnginePlanApi (`engine/plan`) — 7 methods

| Method | Type | Description | Parameters |
|---|---|---|---|
| `getPlanModel` | Query | Live case plan model snapshot | `@PathParam UUID caseId` |
| `getPlanDefinitions` | Query | Plan item definition hierarchy | `@PathParam UUID caseId` |
| `getDecomposition` | Query | HTN decomposition tree | `@PathParam UUID caseId` |
| `getDagPlan` | Query | DAG plan snapshot | `@PathParam UUID caseId` |
| `getDagResult` | Query | DAG execution result | `@PathParam UUID caseId` |
| `getExecutionState` | Query | Composed execution state | `@PathParam UUID caseId` |
| `executionStateStream` | Stream | SSE stream of execution state | `@PathParam UUID caseId` |

## Pre-Migration Cleanup

Before SPI interfaces can be thin delegation, inline business logic must be
extracted to the service layer:

### 1. Goal evaluation extraction (CaseInstanceResource.getGoals)

~60 lines of inline logic: iterates goals from definition, evaluates JQ
conditions via `ExpressionEngineRegistry`, collects reached goals, builds
`CompletionSummary` with pattern matching on `GoalBasedCompletion`/
`PredicateBasedCompletion`.

**Extract to:** `CaseService.evaluateGoals(UUID caseId)` → `GoalEvaluationView`.
CaseService already injects `CaseDefinitionRegistry`, `ExpressionEngineRegistry`,
`CaseHubRuntime`. The private `buildCompletionSummary()` method moves with it.

### 2. Execution state composition (PlanResource.getExecutionState)

Reads from 3 snapshot sources (`CasePlanModelSnapshotProvider`,
`ExecutionSnapshotStore.getDagPlan/getDagResult`), resolves definition, calls
`ExecutionStateSnapshot.compose()`.

**Extract to:** `ExecutionStateBroadcaster.composeInitial(UUID caseId)` already
has this logic. Consolidate — the SPI impl calls
`executionStateBroadcaster.composeInitial(caseId)` for the REST polling endpoint.

### 3. ACL filtering

`CaseInstanceResource.listCases()` and `CaseDefinitionResource.listAll()` do
post-query filtering via `AccessControlProvider.canAccess()`. This is cross-cutting
and can stay in the SPI implementation — the impl calls `caseService.requireCaseAccess()`
or filters via `AccessControlProvider` after the query, same as today.

## Return Types

New view records in `engine-api` module (`io.casehub.engine.api.view`):

| Record | Fields | Replaces |
|---|---|---|
| `CaseInstanceView` | caseId, status, namespace, name, version, createdAt, actorId | `CaseInstanceResponse` (REST), `CaseInstanceType` (GraphQL) |
| `CasePage` | items (List), totalCount, hasMore | `PagedResponse<CaseInstanceResponse>` (REST), `CasePage` (GraphQL) |
| `CaseDefinitionView` | namespace, name, version, title, summary, capabilities | `CaseDefinition` (REST response), `CaseDefinitionType` (GraphQL) |
| `CaseDefinitionPage` | items (List), totalCount, hasMore | `PagedResponse<CaseDefinition>` (REST), `CaseDefinitionPage` (GraphQL) |
| `CaseControlView` | caseId, status | `CaseControlResponse` (REST), `CaseControl` (GraphQL) |
| `SignalResultView` | caseId, accepted | `SignalResponse` (REST), `SignalResult` (GraphQL) |
| `EventLogEntryView` | eventType, streamType, timestamp, payload | `EventLogEntryResponse` (REST), `EventLogEntry` (GraphQL) |
| `EventLogPage` | items (List), totalCount, hasMore | `PagedResponse<EventLogEntryResponse>` (REST), `EventLogPage` (GraphQL) |
| `PlanItemView` | id, name, status, type, parentId | `PlanItemResponse` (REST) |
| `GoalEvaluationView` | goals (List), completionSummary | `GoalEvaluationResponse` (REST) |
| `GoalStatusView` | name, kind, reached, condition | `GoalStatusResponse` (REST) |
| `CompletionSummaryView` | complete, satisfied, total, kind | `CompletionSummary` (REST) |
| `CaseStreamEventView` | caseId, type, data | `CaseStreamEvent` (REST) |
| `CaseLifecycleEventView` | caseId, eventType, commandType, ... | `CaseLifecycleEventType` (GraphQL) |
| `CaseContextChangeEventView` | caseId, changedLayer, contextSnapshot | `CaseContextChangeEventType` (GraphQL) |

### Request Types

| Record | Fields | Used by |
|---|---|---|
| `StartCaseRequest` | namespace, name, version, context (Map) | `EngineCaseApi.startCase` |
| `CaseControlRequest` | reason (optional) | `EngineCaseControlApi.suspend/resume/cancel` |
| `SendSignalRequest` | path, value | `EngineCaseControlApi.sendSignal` |

## SPI Implementation Beans

Five `@ApplicationScoped` CDI beans in `rest/service/`:

| Bean | Implements | Delegates to |
|---|---|---|
| `DefaultEngineCaseApi` | `EngineCaseApi` | `CaseService`, `CaseHubRuntime`, `CaseInstanceRepository`, `PlanItemStore`, `CaseStreamBroadcaster`, `CaseEventPublisher` |
| `DefaultEngineCaseControlApi` | `EngineCaseControlApi` | `CaseService`, `CaseHubRuntime` |
| `DefaultEngineCaseDefinitionApi` | `EngineCaseDefinitionApi` | `CaseMetaModelRepository`, `CaseDefinitionRegistry`, `AccessControlProvider` |
| `DefaultEngineEventLogApi` | `EngineEventLogApi` | `EventLogRepository`, `CaseService` |
| `DefaultEnginePlanApi` | `EnginePlanApi` | `CasePlanModelSnapshotProvider`, `ExecutionSnapshotStore`, `ExecutionStateBroadcaster`, `CaseService` |

Each bean:
- Delegates to existing services — no new business logic
- Maps between entity/service types and `api/view/` return types
- Handles ACL enforcement via `CaseService.requireCaseAccess()`
- Is `@ApplicationScoped` (not `@DefaultBean` — engine is the only provider)

## SPI Interface Location

SPI interfaces and view records live in the engine `api/` module
(`io.casehub.engine.api`). This follows ledger D4 — `api/` is the public
contract module, no JPA deps, no framework deps.

Note: the engine `api/` module is `casehub-engine-api`. It already has Jandex
indexing configured. The APT in `rest/` and `graphql/` will see the SPI
interfaces from the `api/` JAR's Jandex index (per GE-20260914-f53be7).

## APT Configuration

### rest/pom.xml

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

### graphql/pom.xml

Same `annotationProcessorPaths`, with:
```xml
<compilerArgs>
  <arg>-AdomainFilter=engine/cases,engine/control,engine/definitions,engine/events,engine/plan</arg>
  <arg>-AgenerateRest=false</arg>
</compilerArgs>
```

## What Gets Deleted

### rest/ module
- `CaseInstanceResource.java` → generated `GeneratedEngineCasesResource`
- `CaseControlResource.java` → generated `GeneratedEngineControlResource`
- `CaseDefinitionResource.java` → generated `GeneratedEngineDefinitionsResource`
- `EventLogResource.java` → generated `GeneratedEngineEventsResource`
- `SignalResource.java` → folded into `GeneratedEngineControlResource`
- `PlanResource.java` → generated `GeneratedEnginePlanResource`
- `CaseStreamResource.java` → generated (stream method on `GeneratedEngineCasesResource`)
- `ExecutionStateResource.java` → generated (stream method on `GeneratedEnginePlanResource`)
- `dto/CaseInstanceResponse.java` → `api/view/CaseInstanceView`
- `dto/CaseControlResponse.java` → `api/view/CaseControlView`
- `dto/SignalResponse.java` → `api/view/SignalResultView`
- `dto/EventLogEntryResponse.java` → `api/view/EventLogEntryView`
- `dto/PlanItemResponse.java` → `api/view/PlanItemView`
- `dto/GoalEvaluationResponse.java` → `api/view/GoalEvaluationView`
- `dto/GoalStatusResponse.java` → `api/view/GoalStatusView`
- `dto/CompletionSummary.java` → `api/view/CompletionSummaryView`
- `dto/CaseStreamEvent.java` → `api/view/CaseStreamEventView`
- `dto/StartCaseRequest.java` → `api/view/StartCaseRequest`
- `dto/CaseControlRequest.java` → `api/view/CaseControlRequest`
- `dto/SendSignalRequest.java` → `api/view/SendSignalRequest`
- `dto/PagedResponse.java` → `api/view/CasePage`/`EventLogPage` per-domain
- `dto/ProblemDetail.java` → kept (error format)
- `dto/ExecutionStateSnapshot.java` → kept (used by broadcaster, not a REST DTO)

### graphql/ module
- `CaseQueryResolver.java` → generated resolvers
- `CaseMutationResolver.java` → generated resolvers
- `CaseSubscriptionResolver.java` → generated resolvers
- `CaseEventPublisher.java` → kept (CDI event bridge for streams)
- `EngineModelEnricher.java` → kept (MCP model enricher)
- `dto/CaseInstanceType.java` → `api/view/CaseInstanceView`
- `dto/CasePage.java` → `api/view/CasePage`
- `dto/CaseDefinitionType.java` → `api/view/CaseDefinitionView`
- `dto/CaseDefinitionPage.java` → `api/view/CaseDefinitionPage`
- `dto/CaseControl.java` → `api/view/CaseControlView`
- `dto/SignalResult.java` → `api/view/SignalResultView`
- `dto/EventLogEntry.java` → `api/view/EventLogEntryView`
- `dto/EventLogPage.java` → `api/view/EventLogPage`
- `dto/CaseLifecycleEventType.java` → `api/view/CaseLifecycleEventView`
- `dto/CaseContextChangeEventType.java` → `api/view/CaseContextChangeEventView`
- `dto/StartCaseInput.java` → `api/view/StartCaseRequest`
- `dto/CaseFilterInput.java` → filter params become method parameters
- `dto/SignalInput.java` → unused, delete

### Kept (not generated)
- `CaseStreamBroadcaster.java` — CDI event bridge for SSE (stays in rest/)
- `ExecutionStateBroadcaster.java` — CDI event bridge for execution state SSE (stays in rest/)
- `CaseEventPublisher.java` — CDI event bridge for GraphQL subscriptions (stays in graphql/)
- `EngineModelEnricher.java` — MCP model enricher (stays in graphql/)
- Exception mappers — cross-cutting `@Provider` classes
- `CaseScopeExtractor.java` — request filter
- `AclEnforcementHealthCheck.java` — health check
- `CaseService.java` — service layer (expanded with goal evaluation)

## Out of Scope

- `ActorStateResource` (in `engine-support-core`) — separate domain, separate module,
  not part of the engine REST API surface. Can be migrated later if needed.
- `CaseService.startCase()` inline logic in `CaseMutationResolver` — already extracted
  in the REST layer; the GraphQL resolver's version is a duplicate that will be
  consolidated when both layers delegate to the SPI.

## Testing Strategy

1. **API module tests** — pure JUnit: verify view records, request types
2. **SPI implementation tests** — `@QuarkusTest` with in-memory repos: verify
   `DefaultEngineXxxApi` beans delegate correctly, return correct view types,
   handle ACL enforcement
3. **Generated endpoint tests** — update existing REST and GraphQL tests to hit
   generated endpoints (new paths, same semantics)
4. **Streaming tests** — verify `@PlatformStream` methods produce SSE and
   GraphQL subscription output. Use `java.net.http.HttpClient` for SSE
   (GE-20260420-05dca8), not REST Assured
5. **MCP dispatch test** — verify generated resolvers are discoverable by
   `GraphQLModelScanner` (technique from GE-20260818-c2f072)

## References

- casehubio/platform#295 — parent epic
- casehubio/engine#1095 — this issue
- casehubio/ledger#207 — completed migration (reference pattern)
- `2026-09-14-unified-api-generation-design.md` — ledger spec (same directory)
- `GraphQLResolverProcessor.java` (platform) — the code generator
- GE-20260914-638e46 — null→200 regression in generated endpoints
- GE-20260914-3854b8 — APT domain filtering for multi-module
- GE-20260914-f53be7 — SPI must be in dependency JAR for Jandex APT
- GE-20260914-714a71 — `-parameters` flag needed on SPI module
- GE-20260804-8b0fd6 — BroadcastProcessor for CDI→Multi SSE bridge
- GE-20260818-c2f072 — testing MCP dispatch without CDI
- GE-20260420-05dca8 — REST Assured hangs on SSE, use HttpClient
