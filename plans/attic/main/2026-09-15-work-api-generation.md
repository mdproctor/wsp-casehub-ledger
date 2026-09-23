# Work @McpDomain SPI Migration (Core) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/work#400 — feat: unified API generation — migrate to @McpDomain SPI
**Issue group:** casehubio/work#400 (covers casehubio/platform#300)

**Goal:** Replace hand-written REST endpoints and GraphQL resolvers for the core work item domain with APT-generated code from the platform graphql-generator.

**Architecture:** Define 5 SPI interfaces in work `api/` module with `@McpDomain` + platform annotations. Create implementation beans in `rest/service/` that delegate to existing `WorkItemOperations` and stores. Wire the platform APT generator in `rest/` and `graphql/` modules. Delete hand-written endpoints.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-graphql-generator APT, SmallRye GraphQL, RESTEasy Reactive

## Global Constraints

- All SPI interfaces use `io.casehub.platform.api.mcp` annotations — never JAX-RS on SPIs
- All tenant-scoped SPI methods use `@ContextParam("tenancyId")` — never expose tenancyId as a query param
- View records in `io.casehub.work.api.view` (work api module)
- SPI interfaces in `io.casehub.work.api.spi` (work api module, existing package)
- SPI implementation beans in `io.casehub.work.rest.service` (work rest module, new package)
- APT domain filter: `work/items,work/lifecycle,work/notes,work/links,work/relations`
- `-parameters` flag required on api module compiler config
- Work project root: `/Users/mdproctor/claude/casehub/slots/194/work`
- Platform version: `${version.io.casehub}` from parent POM
- Install updated platform JARs (with `@ContextParam`) to slot .m2 before starting

---

## Batch 1: View Records + SPI Interfaces

After this batch: api/ module has all SPI interfaces and view records. Compiles. Tests pass. No functional changes yet — REST and GraphQL are untouched.

### Task 1: Create view records and request types in api/

Create the view records that replace both REST DTOs and GraphQL types. Also create request records for note/link/relation operations. Move lifecycle request DTOs to api/view.

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/view/WorkItemView.java`
- Create: `api/src/main/java/io/casehub/work/api/view/WorkItemWithAuditView.java`
- Create: `api/src/main/java/io/casehub/work/api/view/WorkItemPage.java`
- Create: `api/src/main/java/io/casehub/work/api/view/WorkItemNoteView.java`
- Create: `api/src/main/java/io/casehub/work/api/view/WorkItemLinkView.java`
- Create: `api/src/main/java/io/casehub/work/api/view/WorkItemRelationView.java`
- Create: `api/src/main/java/io/casehub/work/api/view/AuditEntryView.java`
- Create: `api/src/main/java/io/casehub/work/api/view/WorkItemLabelView.java`
- Create: `api/src/main/java/io/casehub/work/api/view/AddNoteRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/AddLinkRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/AddRelationRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/CompleteRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/RejectRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/CancelRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/DelegateRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/SuspendRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/FaultRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/ObsoleteRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/EscalateRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/ExtendRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/UpdateDeadlineRequest.java`
- Create: `api/src/main/java/io/casehub/work/api/view/CompensateRequest.java`
- Test: `api/src/test/java/io/casehub/work/api/view/ViewRecordTest.java`

**Interfaces:**
- Consumes: `WorkItemStatus`, `WorkItemPriority`, `WorkItemRelationType` from `io.casehub.work.api`
- Produces: All view/request records used by SPI interfaces (Task 2) and SPI implementations (Task 3)

- [ ] **Step 1: Write view record unit test**

Create `ViewRecordTest.java` verifying key records: `WorkItemView` construction, `WorkItemPage` pagination, `WorkItemNoteView` fields, `WorkItemRelationView` fields. Reference `WorkItemResponse.java` (rest module) for the 37 curated field names — the test confirms `WorkItemView` has the same fields.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ViewRecordTest -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`
Expected: Compilation failure — view records don't exist yet

- [ ] **Step 3: Create all view records**

Each record mirrors the curated REST DTO fields. Read `WorkItemResponse.java` for `WorkItemView` fields (37 fields — the curated subset of WorkItem's 57). Read `WorkItemWithAuditResponse.java` for `WorkItemWithAuditView`. Read runtime model classes (`WorkItemNote`, `WorkItemLink`) for note/link/relation view fields.

Lifecycle request records are simple — copy field signatures from existing rest DTOs:
- `CompleteRequest(String resolution, String outcome)`
- `RejectRequest(String reason, String outcome)`
- `CancelRequest(String reason)`
- `DelegateRequest(String delegateTo, String reason)`
- `SuspendRequest(String reason)`
- `FaultRequest(String errorDetail)`
- `ObsoleteRequest(String reason)`
- `EscalateRequest(String reason)`
- `ExtendRequest(String reason, java.time.Instant newDeadline)`
- `UpdateDeadlineRequest(java.time.Instant deadline)`
- `CompensateRequest(String namespace, String name, String version, java.util.Map<String,Object> context)`

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ViewRecordTest -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/work add api/src/
git -C /Users/mdproctor/claude/casehub/slots/194/work commit -m "feat(#400): add view records and request types for SPI migration Refs #400"
```

### Task 2: Create SPI interfaces + build config

Define the 5 SPI interfaces with platform annotations. Add `-parameters` compiler flag to api module.

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/spi/WorkItemApi.java`
- Create: `api/src/main/java/io/casehub/work/api/spi/WorkItemLifecycleApi.java`
- Create: `api/src/main/java/io/casehub/work/api/spi/WorkItemNoteApi.java`
- Create: `api/src/main/java/io/casehub/work/api/spi/WorkItemLinkApi.java`
- Create: `api/src/main/java/io/casehub/work/api/spi/WorkItemRelationApi.java`
- Modify: `api/pom.xml` — add `<parameters>true</parameters>` if not present
- Test: `api/src/test/java/io/casehub/work/api/spi/SpiInterfaceTest.java`

**Interfaces:**
- Consumes: View records from Task 1, existing api types
- Produces: SPI interfaces consumed by Task 3 (impl beans) and Task 4 (APT generator)

- [ ] **Step 1: Write SPI interface verification test**

Test that: all 5 interfaces have `@McpDomain`, correct domain strings, correct method counts per annotation type, parameter names preserved via `-parameters` flag.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=SpiInterfaceTest -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`
Expected: Compilation failure — SPI interfaces don't exist

- [ ] **Step 3: Add `-parameters` flag to api/pom.xml**

Check if the parent POM already has this configured. If not, add `<parameters>true</parameters>` to `maven-compiler-plugin` configuration in `api/pom.xml`.

- [ ] **Step 4: Create all 5 SPI interfaces**

Follow the spec's annotation model exactly. Key patterns:
- `@ContextParam("tenancyId")` on every tenant-scoped parameter
- `@PathParam` on UUID path parameters (workItemId, noteId, linkId, relId)
- `@PaginatedResponse` on listAll and inbox
- `@PlatformStream` on streamEvents and streamWorkItemEvents
- `@RestStatus(201)` on create and clone
- Stream methods return `Multi<WorkItemLifecycleEvent>` (reused from api)

**WorkItemApi** (`work/items`): 10 methods — listAll, getById, create, clone, inboxSummary, inbox, addLabel, removeLabel, streamEvents, streamWorkItemEvents

**WorkItemLifecycleApi** (`work/lifecycle`): 17 methods — claim, start, complete(CompleteRequest), reject(RejectRequest), delegate(DelegateRequest), acceptDelegation, declineDelegation, release, suspend(SuspendRequest), resume, cancel(CancelRequest), fault(FaultRequest), obsolete(ObsoleteRequest), escalate(EscalateRequest), extend(ExtendRequest), updateDeadline(UpdateDeadlineRequest), compensate(CompensateRequest)

**WorkItemNoteApi** (`work/notes`): 4 methods — addNote(AddNoteRequest), listNotes, editNote(AddNoteRequest), deleteNote

**WorkItemLinkApi** (`work/links`): 3 methods — addLink(AddLinkRequest), listLinks, deleteLink

**WorkItemRelationApi** (`work/relations`): 6 methods — addRelation(AddRelationRequest), listOutgoing, listIncoming, deleteRelation, children, parent

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=SpiInterfaceTest -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`
Expected: PASS

- [ ] **Step 6: Run full api module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`
Expected: All existing + new tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/work add api/
git -C /Users/mdproctor/claude/casehub/slots/194/work commit -m "feat(#400): add 5 SPI interfaces with @McpDomain annotations Refs #400"
```

---

## Batch 2: SPI Implementations

After this batch: 5 SPI implementation beans exist, delegate to existing services. All existing endpoints still work unchanged.

### Task 3: Create SPI implementation beans

Create the 5 `@ApplicationScoped` CDI beans in `rest/service/` that implement the SPI interfaces by delegating to `WorkItemOperations` and stores.

**Files:**
- Create: `rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemApi.java`
- Create: `rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemLifecycleApi.java`
- Create: `rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemNoteApi.java`
- Create: `rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemLinkApi.java`
- Create: `rest/src/main/java/io/casehub/work/rest/service/DefaultWorkItemRelationApi.java`

**Interfaces:**
- Consumes: SPI interfaces from Task 2, view records from Task 1, existing `WorkItemOperations`, `WorkItemStore`, `WorkItemNoteStore`, `WorkItemLinkStore`, `WorkItemRelationStore`, `AuditEntryStore`, `WorkItemEventBroadcaster`
- Produces: CDI beans that the APT-generated code will inject via `@Inject WorkItemApi` etc.

- [ ] **Step 1: Create DefaultWorkItemLifecycleApi (thin delegates)**

All 17 methods delegate to `WorkItemOperations`. Most are one-liners. This bean is the simplest — start here to establish the pattern. Inject `WorkItemOperations` and `CurrentPrincipal`. Each method resolves tenancyId from the `@ContextParam` parameter, calls `workItemOperations.{method}()`, maps result to `WorkItemView`.

- [ ] **Step 2: Create DefaultWorkItemApi (CRUD + query + labels + streaming)**

10 methods. Inject `WorkItemOperations`, `WorkItemStore`, `AuditEntryStore`, `WorkItemEventBroadcaster`, `CurrentPrincipal`. Key inline logic migrations:
- `listAll`: build `WorkItemQuery` from filter params, delegate to `workItemStore.query()`, post-filter labels/outcomes, map results to `WorkItemPage`
- `inbox`: call `workItemStore.scanRoots()`, apply 5 post-filters, map to `WorkItemPage`
- `getById`: load work item + audit trail, map to `WorkItemWithAuditView`
- `create`/`clone`: delegate to `workItemOperations`, map result
- `addLabel`/`removeLabel`: delegate to `workItemOperations`
- `streamEvents`/`streamWorkItemEvents`: delegate to `broadcaster.stream()`

- [ ] **Step 3: Create DefaultWorkItemNoteApi (note CRUD)**

4 methods. Inject `WorkItemNoteStore`, `CurrentPrincipal`, `WorkItemOperations` (for access check). Migrate inline logic from `WorkItemResource`:
- `addNote`: validate content, construct `WorkItemNote`, persist via store
- `listNotes`: delegate to `noteStore.findByWorkItemId()`
- `editNote`: find note, verify ownership, update content
- `deleteNote`: find note, verify ownership, delete

- [ ] **Step 4: Create DefaultWorkItemLinkApi (link CRUD)**

3 methods. Inject `WorkItemLinkStore`, `CurrentPrincipal`, `WorkItemOperations`. Migrate inline logic:
- `addLink`: validate fields, construct `WorkItemLink`, persist
- `listLinks`: delegate to `linkStore.findByWorkItemId()`
- `deleteLink`: find link, verify ownership, delete

- [ ] **Step 5: Create DefaultWorkItemRelationApi (relations + children/parent)**

6 methods. Inject `WorkItemRelationStore`, `WorkItemStore`, `CurrentPrincipal`, `WorkItemOperations`. Contains the heaviest inline logic migration:
- `addRelation`: validate, BFS cycle detection (~40 lines from `WorkItemRelationResource`), duplicate check, construct relation entity, persist
- `listOutgoing`/`listIncoming`: delegate to store
- `deleteRelation`: find, verify ownership, delete
- `children`/`parent`: relation lookup + work item resolution (from `WorkItemResource`)

- [ ] **Step 6: Compile rest module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl rest -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`
Expected: Compiles clean — new beans don't conflict with existing endpoints

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/work add rest/src/main/java/io/casehub/work/rest/service/
git -C /Users/mdproctor/claude/casehub/slots/194/work commit -m "feat(#400): add 5 SPI implementation beans delegating to work services Refs #400"
```

---

## Batch 3: APT Wiring + Switchover

After this batch: hand-written REST resources and GraphQL resolvers deleted, APT-generated code serves all core endpoints. Tests updated and passing.

### Task 4: Wire APT generator in rest/ and graphql/ modules

Configure the platform graphql-generator as an annotation processor in both modules with domain filtering.

**Files:**
- Modify: `rest/pom.xml` — add annotationProcessorPaths + compilerArgs
- Modify: `graphql/pom.xml` — add annotationProcessorPaths + compilerArgs

**Interfaces:**
- Consumes: SPI interfaces from Task 2 (Jandex-indexed in api/ JAR)
- Produces: Generated `GeneratedWork*Resource.java` and `GeneratedWork*Resolver.java` classes

- [ ] **Step 1: Install api module to local repo**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -DskipTests -q -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`

- [ ] **Step 2: Add APT config to rest/pom.xml**

Add `maven-compiler-plugin` with `annotationProcessorPaths` (generator + work-api JARs) and `compilerArgs` (`-AdomainFilter=...`, `-AgenerateGraphQL=false`, `-Xlint:unchecked`). Follow the spec's exact XML.

- [ ] **Step 3: Add APT config to graphql/pom.xml**

Same `annotationProcessorPaths`, with `-AgenerateRest=false` instead.

- [ ] **Step 4: Compile to verify generation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl rest,graphql -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`
Expected: BUILD SUCCESS. Check `rest/target/generated-sources/annotations/` for 5 REST resources. Check `graphql/target/generated-sources/annotations/` for 5 GraphQL resolvers.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/work add rest/pom.xml graphql/pom.xml
git -C /Users/mdproctor/claude/casehub/slots/194/work commit -m "feat(#400): wire APT generator with domain filtering Refs #400"
```

### Task 5: Delete hand-written endpoints + update tests

Remove hand-written REST resources, GraphQL resolvers, and old DTOs. Update references in kept files.

**Files:**
- Delete: `rest/.../WorkItemResource.java` (823 lines)
- Delete: `rest/.../WorkItemRelationResource.java`
- Delete: `rest/.../WorkItemResponse.java`, `WorkItemWithAuditResponse.java`, `WorkItemLabelResponse.java`, `WorkItemMapper.java`
- Delete: `rest/.../CreateWorkItemRequest.java` (rest module copy — api has its own)
- Delete: `rest/.../CompleteRequest.java`, `RejectRequest.java`, `CancelRequest.java`, `DelegateRequest.java`, `SuspendRequest.java`, `FaultRequest.java`, `ObsoleteRequest.java`, `EscalateRequest.java`, `ExtendRequest.java`, `UpdateDeadlineRequest.java`
- Delete: `graphql/.../WorkItemQueryResolver.java`, `WorkItemMutationResolver.java`, `WorkItemSubscriptionResolver.java`
- Delete: All `graphql/.../dto/*.java` (5 files)
- Keep: `WorkItemEventBroadcaster.java`, exception mappers, resources out of scope (Template, Schedule, LabelRule, Vocabulary, Audit, SpawnGroup, AsyncApi, Bulk, Spawn, Instances)
- Update: `WorkItemEventBroadcaster.java` — switch from any old event types to api types if needed
- Delete: REST tests for deleted resources
- Delete: GraphQL tests for deleted resolvers

**Interfaces:**
- Consumes: Generated endpoint classes from Task 4
- Produces: Clean codebase with only generated endpoints for core domain

- [ ] **Step 1: Check references from kept files to deleted DTOs**

Search for imports of `WorkItemResponse`, `WorkItemLabelResponse`, `WorkItemMapper`, lifecycle request DTOs in kept files (WorkItemEventBroadcaster, exception mappers, out-of-scope resources). Update any references.

- [ ] **Step 2: Delete hand-written REST resources and DTOs**

Delete WorkItemResource (823 lines), WorkItemRelationResource, and all REST DTOs being replaced. Keep exception mappers and out-of-scope resources.

- [ ] **Step 3: Delete hand-written GraphQL resolvers and DTOs**

Delete 3 resolver classes and all 5 graphql/dto/ files. Keep CaseEventPublisher (if exists) and WorkModelEnricher.

- [ ] **Step 4: Compile to verify no broken references**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl rest,graphql -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`
Expected: BUILD SUCCESS with generated code replacing deleted classes

- [ ] **Step 5: Delete tests for deleted classes, run remaining tests**

Delete REST tests that test old endpoints. Delete GraphQL tests for deleted resolvers. Run remaining tests to verify no regressions.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/slots/194/work/pom.xml`
Run non-QuarkusTest tests in rest and graphql modules individually.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/194/work add -A
git -C /Users/mdproctor/claude/casehub/slots/194/work commit -m "feat(#400): delete hand-written endpoints, switch to APT-generated code Closes #400"
```

## References

- [2026-09-15-work-api-generation-design.md] — design spec this plan implements
- [2026-09-15-engine-api-generation-design.md] — engine spec (reference pattern)
- [2026-09-15-engine-api-generation.md] — engine plan (template)
- [WorkItemResource.java] — primary REST resource (823 lines)
- [WorkItemRelationResource.java] — relation resource
- [WorkItem.java] — api record (57 fields)
- [WorkItemResponse.java] — REST DTO (37 curated fields)
- [WorkItemOperations.java] — existing service layer (31 methods)
- [GE-20260914-6077b8] — APT isolated classpath
- [GE-20260914-3854b8] — APT domain filtering
- [GE-20260914-f53be7] — SPI must be in dependency JAR for Jandex APT
- [GE-20260818-c2f072] — testing MCP dispatch without CDI
- [casehubio/work#400] — focal issue
- [casehubio/platform#311] — @ContextParam annotation
