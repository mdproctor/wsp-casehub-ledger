# Unified API Generation — Migrate Work to @McpDomain SPI (Core)

**Issue:** casehubio/work#400
**Parent epic:** casehubio/platform#295 (CLOSED — platform generator done)
**Depends on:** casehubio/engine#1095 (DONE — engine migration, pattern established), casehubio/platform#311 (DONE — @ContextParam annotation)
**Date:** 2026-09-15
**Scope:** Core-first — WorkItemResource (36 endpoints) + WorkItemRelationResource (4 endpoints)

## Summary

Replace hand-written REST endpoints and GraphQL resolvers for the core work
item domain in `casehub-work` with APT-generated code from the platform
`graphql-generator`. Define 5 SPI interfaces in the work `api/` module with
`@McpDomain` + `@PlatformQuery`/`@PlatformMutation`/`@PlatformStream`
annotations. The generator produces REST resources, GraphQL resolvers, and
MCP tool registrations from the same interface.

**Scope:** 36 REST endpoints (WorkItemResource) + 4 REST endpoints
(WorkItemRelationResource), 3 GraphQL resolvers (16 operations), 2 SSE
streams, 2 GraphQL subscriptions → 5 SPI interfaces + 40 methods total.

**Out of scope (follow-up):** WorkItemTemplateResource (7), WorkItemScheduleResource (5),
LabelRuleResource (5), VocabularyResource (2), AuditResource (1),
SpawnGroupResource (1), AsyncApiResource (1), WorkItemBulkResource (1),
WorkItemSpawnResource (3), WorkItemInstancesResource (1).

## Annotation Model

All SPI interfaces use platform annotations from `io.casehub.platform.api.mcp`.
Tenant-scoped parameters use `@ContextParam("tenancyId")` — the generator
resolves these from `CurrentPrincipal` server-side instead of exposing them
as REST query parameters or GraphQL arguments (platform#311).

```java
@McpDomain("work/items")
public interface WorkItemApi {

    @PlatformQuery("List work items with optional filtering")
    @PaginatedResponse
    WorkItemPage listAll(WorkItemStatus status, WorkItemPriority priority,
                         String label, String outcome,
                         @ContextParam("tenancyId") String tenancyId,
                         Integer offset, Integer limit);

    @PlatformQuery("Get a work item by ID")
    WorkItemView getById(@PathParam UUID workItemId,
                         @ContextParam("tenancyId") String tenancyId);

    @PlatformMutation("Create a new work item")
    @RestStatus(201)
    WorkItemView create(WorkItemCreateRequest request,
                        @ContextParam("tenancyId") String tenancyId);
}
```

## SPI Interfaces

Five interfaces organised by caller intent:

### WorkItemApi (`work/items`) — 10 methods

| Method | Type | Description |
|---|---|---|
| `listAll` | Query | List items with status/priority/label/outcome filtering + pagination |
| `getById` | Query | Get item by ID (with audit trail) |
| `create` | Mutation | Create a new work item |
| `clone` | Mutation | Clone an existing work item |
| `inboxSummary` | Query | Aggregated inbox summary (counts by status/priority) |
| `inbox` | Query | Inbox view with root items and post-filtering |
| `addLabel` | Mutation | Add a label to a work item |
| `removeLabel` | Mutation | Remove a label from a work item |
| `streamEvents` | Stream | Global work item event stream (SSE) |
| `streamWorkItemEvents` | Stream | Events for a specific work item (SSE) |

### WorkItemLifecycleApi (`work/lifecycle`) — 17 methods

| Method | Type | Description |
|---|---|---|
| `claim` | Mutation | Claim a work item |
| `start` | Mutation | Start a claimed work item |
| `complete` | Mutation | Complete a work item |
| `reject` | Mutation | Reject a work item |
| `delegate` | Mutation | Delegate a work item |
| `acceptDelegation` | Mutation | Accept a delegation |
| `declineDelegation` | Mutation | Decline a delegation |
| `release` | Mutation | Release a claimed work item |
| `suspend` | Mutation | Suspend a work item |
| `resume` | Mutation | Resume a suspended work item |
| `cancel` | Mutation | Cancel a work item |
| `fault` | Mutation | Mark a work item as faulted |
| `obsolete` | Mutation | Mark a work item as obsolete |
| `escalate` | Mutation | Escalate a work item |
| `extend` | Mutation | Extend a work item deadline |
| `updateDeadline` | Mutation | Update deadline |
| `compensate` | Mutation | Create a compensating work item |

### WorkItemNoteApi (`work/notes`) — 4 methods

| Method | Type | Description |
|---|---|---|
| `addNote` | Mutation | Add a note to a work item |
| `listNotes` | Query | List notes for a work item |
| `editNote` | Mutation | Edit an existing note |
| `deleteNote` | Mutation | Delete a note |

### WorkItemLinkApi (`work/links`) — 3 methods

| Method | Type | Description |
|---|---|---|
| `addLink` | Mutation | Add an external link to a work item |
| `listLinks` | Query | List links for a work item |
| `deleteLink` | Mutation | Delete a link |

### WorkItemRelationApi (`work/relations`) — 6 methods

| Method | Type | Description |
|---|---|---|
| `addRelation` | Mutation | Add a relation (with cycle detection) |
| `listOutgoing` | Query | List outgoing relations |
| `listIncoming` | Query | List incoming relations |
| `deleteRelation` | Mutation | Delete a relation |
| `children` | Query | Get child work items |
| `parent` | Query | Get parent work item |

## View Records

WorkItem (57 fields) deliberately curates to WorkItemResponse (37 fields) —
20 internal fields excluded, 3 reshaped. New view records mirror this curation.

### New records in `io.casehub.work.api.view`:

| Record | Fields from | Replaces |
|---|---|---|
| `WorkItemView` | WorkItemResponse (37 fields) | `WorkItemResponse` (REST), `WorkItemType` (GraphQL) |
| `WorkItemWithAuditView` | WorkItemView + audit trail | `WorkItemWithAuditResponse` (REST) |
| `WorkItemPage` | items, totalCount, hasMore | `PagedResponse<WorkItemResponse>` |
| `WorkItemNoteView` | id, author, content, createdAt, updatedAt | `WorkItemNote` (runtime model) |
| `WorkItemLinkView` | id, url, title, type, createdAt | `WorkItemLink` (runtime model) |
| `WorkItemRelationView` | id, sourceId, targetId, relationType | `WorkItemRelation` (runtime model) |
| `AuditEntryView` | timestamp, actorId, action, details | `AuditEntry` (runtime model) |
| `WorkItemLabelView` | name, value | `WorkItemLabelResponse` (REST) |
| `AddNoteRequest` | content | inline body parsing |
| `AddLinkRequest` | url, title, type | inline body parsing |
| `AddRelationRequest` | targetId, relationType | inline body parsing |

### Reused from `io.casehub.work.api`:

| Type | Used by |
|---|---|
| `WorkItemCreateRequest` | WorkItemApi.create |
| `WorkItemStatus` | WorkItemApi.listAll (filter param) |
| `WorkItemPriority` | WorkItemApi.listAll (filter param) |
| `WorkItemSummary` | WorkItemApi.inboxSummary |
| `WorkItemRootView` | WorkItemApi.inbox |
| `WorkItemLifecycleEvent` | WorkItemApi.streamEvents/streamWorkItemEvents |
| `WorkItemRelationType` | WorkItemRelationApi (param) |

### Lifecycle request types (reused from existing REST DTOs):

Lifecycle methods take request bodies with optional fields (reason, outcome,
comment). These are simple records — check if existing REST DTOs
(`CompleteRequest`, `RejectRequest`, etc.) can be moved to api/view or if
new records are needed.

## SPI Interface Location

SPI interfaces live in `api/src/main/java/io/casehub/work/api/spi/`
(existing package — already has `WorkItemOperations`, `WorkItemStore`).
View records in `api/src/main/java/io/casehub/work/api/view/`.

The `api/` module already has Jandex indexing and `casehub-platform-api`
as a dependency. Add `<parameters>true</parameters>` to the compiler
plugin config.

## SPI Implementation Beans

Five `@ApplicationScoped` CDI beans in `rest/src/main/java/io/casehub/work/rest/service/`:

| Bean | Implements | Delegates to |
|---|---|---|
| `DefaultWorkItemApi` | `WorkItemApi` | `WorkItemOperations`, `WorkItemStore`, `AuditEntryStore`, `WorkItemEventBroadcaster` |
| `DefaultWorkItemLifecycleApi` | `WorkItemLifecycleApi` | `WorkItemOperations` (thin delegates for all lifecycle methods) |
| `DefaultWorkItemNoteApi` | `WorkItemNoteApi` | `WorkItemNoteStore` |
| `DefaultWorkItemLinkApi` | `WorkItemLinkApi` | `WorkItemLinkStore` |
| `DefaultWorkItemRelationApi` | `WorkItemRelationApi` | `WorkItemRelationStore`, `WorkItemStore` |

### Inline logic migration

| Method | Lines | Target bean | Nature |
|---|---|---|---|
| `listAll` | ~15 | DefaultWorkItemApi | Query building + label/outcome filtering |
| `inbox` | ~20 | DefaultWorkItemApi | scanRoots() + 5 post-filters |
| `addNote` | ~12 | DefaultWorkItemNoteApi | Validation + entity construction |
| `editNote` | ~10 | DefaultWorkItemNoteApi | Find + ownership check + update |
| `addLink` | ~12 | DefaultWorkItemLinkApi | Validation + entity construction |
| `addRelation` | ~40 | DefaultWorkItemRelationApi | BFS cycle detection + validation |
| `compensate` | ~10 | DefaultWorkItemLifecycleApi | Build WorkItemCreateRequest |

## APT Configuration

### rest/pom.xml

```xml
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <annotationProcessorPaths>
      <path>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-graphql-generator</artifactId>
        <version>${version.io.casehub}</version>
      </path>
      <path>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-work-api</artifactId>
        <version>${project.version}</version>
      </path>
    </annotationProcessorPaths>
    <compilerArgs>
      <arg>-Xlint:unchecked</arg>
      <arg>-AdomainFilter=work/items,work/lifecycle,work/notes,work/links,work/relations</arg>
      <arg>-AgenerateGraphQL=false</arg>
    </compilerArgs>
  </configuration>
</plugin>
```

### graphql/pom.xml

Same `annotationProcessorPaths`, with:
```xml
<compilerArgs>
  <arg>-Xlint:unchecked</arg>
  <arg>-AdomainFilter=work/items,work/lifecycle,work/notes,work/links,work/relations</arg>
  <arg>-AgenerateRest=false</arg>
</compilerArgs>
```

## What Gets Deleted

### rest/ module
- `WorkItemResource.java` (823 lines) → generated across 5 resources
- `WorkItemRelationResource.java` → generated `GeneratedWorkRelationsResource`
- `CreateWorkItemRequest.java` → moved to api/view or reused from api
- `WorkItemResponse.java` → `api/view/WorkItemView`
- `WorkItemWithAuditResponse.java` → `api/view/WorkItemWithAuditView`
- `WorkItemLabelResponse.java` → `api/view/WorkItemLabelView`
- `WorkItemMapper.java` → replaced by mapping in impl beans
- Lifecycle request DTOs (`CancelRequest`, `CompleteRequest`, etc.) — evaluate individually

### graphql/ module
- `WorkItemQueryResolver.java` → generated resolvers
- `WorkItemMutationResolver.java` → generated resolvers
- `WorkItemSubscriptionResolver.java` → generated resolvers
- All `graphql/dto/*.java` (5 files)

### Kept (not generated)
- `WorkItemEventBroadcaster.java` — CDI event bridge for SSE (stays in rest/)
- Exception mappers — cross-cutting `@Provider` classes
- `WorkItemMapper.java` — evaluate: may be obsoleted by view record constructors

## Testing Strategy

1. **API module tests** — pure JUnit: verify view records, request types
2. **SPI implementation tests** — unit tests with mocks for repositories:
   verify `DefaultWorkItemApi` delegation, mapping, inline logic
3. **Generated endpoint tests** — update existing REST and GraphQL tests
   to hit generated endpoints (new paths)
4. **Streaming tests** — verify `@PlatformStream` methods produce SSE
5. **MCP dispatch test** — verify generated resolvers discoverable by
   `GraphQLModelScanner` (technique from GE-20260818-c2f072)

## Pre-existing Issue

REST module QuarkusTests may have datasource configuration issues
similar to engine (if ledger JPA entities are on the classpath). Pure
unit tests and API module tests should be the primary verification.

## References

- casehubio/platform#295 — parent epic
- casehubio/work#400 — this issue
- casehubio/engine#1095 — completed migration (reference pattern)
- `2026-09-15-engine-api-generation-design.md` — engine spec (same directory)
- `WorkItemResource.java` — primary REST resource (823 lines)
- `WorkItemRelationResource.java` — relation resource
- `WorkItem.java` — api record (57 fields)
- `WorkItemResponse.java` — REST DTO (37 fields, deliberate curation)
- GE-20260914-6077b8 — APT isolated classpath
- GE-20260914-3854b8 — APT domain filtering
- GE-20260914-f53be7 — SPI must be in dependency JAR for Jandex APT
- GE-20260818-c2f072 — testing MCP dispatch without CDI
