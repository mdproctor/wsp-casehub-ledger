# Handover — Slot 194

## Branch
Work repo: `issue-400-work-mcpdomain-spi` (pushed, 5 commits ahead of main).
Engine repo: `issue-1095-engine-mcpdomain-spi` (pushed, from prior session).
Ledger repo: `main`.
Platform repo: `issue-311-context-param` (local only — NOT on origin/main).

## Active Issue
`casehubio/iot#105` — migrate iot to @McpDomain SPI.
Queue position 4/17 (ledger done, engine done, work done, iot active).

## Session Summary

### Work #400 — Completed
Full @McpDomain SPI migration for casehub-work core domain. 5 commits:

1. **View records + request types** — 22 records in `io.casehub.work.api.view`
   (8 view records mirroring curated REST DTOs, 14 lifecycle request types).
   WorkItemView has 43 fields matching WorkItemResponse.

2. **SPI interfaces** — 5 interfaces with `@McpDomain` annotations:
   - WorkItemApi (`work/items`, 10 methods: 4 queries, 4 mutations, 2 streams)
   - WorkItemLifecycleApi (`work/lifecycle`, 17 mutations)
   - WorkItemNoteApi (`work/notes`, 4 methods)
   - WorkItemLinkApi (`work/links`, 3 methods)
   - WorkItemRelationApi (`work/relations`, 6 methods)
   All use `@ContextParam("tenancyId")` from platform#311. Added Mutiny
   dependency and `-parameters` compiler flag to api/pom.xml.

3. **SPI impl beans** — 5 `@ApplicationScoped` CDI beans in `rest/service/`:
   DefaultWorkItemApi, DefaultWorkItemLifecycleApi, DefaultWorkItemNoteApi,
   DefaultWorkItemLinkApi, DefaultWorkItemRelationApi. Shared `ViewMapper`
   handles WorkItem→WorkItemView mapping. Relation bean has BFS cycle detection.

4. **APT wiring** — `maven-compiler-plugin` with `annotationProcessorPaths`
   in rest/pom.xml and graphql/pom.xml. Domain filter:
   `work/items,work/lifecycle,work/notes,work/links,work/relations`.

5. **Endpoint switchover** — deleted WorkItemResource (823 lines),
   WorkItemRelationResource, 11 REST DTOs, WorkItemMapper, 3 GraphQL
   resolvers, 5 GraphQL DTOs. Updated WorkItemInstancesResource and
   WorkItemTemplateResource to use ViewMapper. Net -1993/+1485 lines.

Branch pushed to origin. PR not yet created.

## Resume Point

**Start casehubio/iot#105 — migrate iot to @McpDomain SPI.**

Steps:
1. Create feature branch in iot repo: `git -C /Users/mdproctor/claude/casehub/slots/194/iot checkout -b issue-105-iot-mcpdomain-spi`
2. Brainstorm the iot SPI design (same pattern as engine/work: identify endpoints, define SPI interfaces, view records)
3. Write spec and plan
4. Execute plan (same 3-batch structure: view records + SPIs, impl beans, APT + switchover)

## Key Discoveries

**Platform @ContextParam requires slot .m2 install.** The work project uses
`.mvn/maven.config` with `-Dmaven.repo.local=/Users/mdproctor/claude/casehub/slots/194/.m2`.
Platform modules must be installed to the slot .m2, not the global `~/.m2`. Command:
```
mvn install -pl platform-api,generator-common,graphql-generator -DskipTests -q \
  -Dmaven.repo.local=/Users/mdproctor/claude/casehub/slots/194/.m2 \
  -f /Users/mdproctor/claude/casehub/slots/194/platform/pom.xml
```

**WorkItemStatus uses PENDING not CREATED.** LabelPersistence uses MANUAL not PERMANENT.

**WorkItemQuery.Builder uses `labelPattern()` not `label()`.** Caught at compile time.

**Out-of-scope resources reference deleted types.** WorkItemInstancesResource and
WorkItemTemplateResource both referenced WorkItemMapper — updated to use ViewMapper.
Check all kept resources for similar references when doing the next migration.

**GraphQL QuarkusTests fail on deployment artifact resolution.** Pre-existing issue
unrelated to the migration — `casehub-work-deployment` not in slot .m2.

## Standing Instructions

1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Platform JARs with @ContextParam installed to slot .m2 (platform-api, generator-common, graphql-generator).
4. IntelliJ needs the slot project opened for plan execution.

## References

| Artifact | Path |
|----------|------|
| Slot plan | `slots/194/.plan` |
| Work spec | `wsp-casehub-ledger/specs/issue-400-work-mcpdomain-spi/2026-09-15-work-api-generation-design.md` |
| Work plan | `wsp-casehub-ledger/plans/2026-09-15-work-api-generation.md` |
| Engine spec | `wsp-casehub-ledger/specs/issue-295-unified-api-generation/2026-09-15-engine-api-generation-design.md` |
| Work decisions | `wsp-casehub-ledger/specs/issue-400-work-mcpdomain-spi/decisions.md` |
