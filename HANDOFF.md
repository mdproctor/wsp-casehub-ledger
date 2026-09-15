# Handover — Slot 194

## Branch
Engine repo: `issue-1095-engine-mcpdomain-spi` (pushed, 6 commits ahead of main).
Work repo: `main` (no feature branch yet — plan written, execution not started).
Ledger repo: `main`.
Platform repo: `issue-311-context-param` (2 commits, landed on local original main).

## Active Issue
`casehubio/work#400` — migrate work to @McpDomain SPI.
Queue position 3/17 (ledger done, engine done, work active).

## Session Summary

### Engine #1095 — Completed
Completed Batch 3 (Tasks 5-6):
1. **Task 5: Wire APT generator** — added `annotationProcessorPaths` + domain filtering
   to rest/pom.xml and graphql/pom.xml. 5 REST resources and 5 GraphQL resolvers generated.
2. **Task 6: Delete hand-written endpoints** — deleted 8 REST resources, 14 REST DTOs,
   3 GraphQL resolvers, 13 GraphQL DTOs, 9 test files. Updated CaseService.evaluateGoals()
   to return new view records directly. Updated CaseStreamBroadcaster to emit
   CaseStreamEventView. 54 files changed, +217/-3654 lines. All tests pass.

Branch pushed to origin. PR not yet created.

### Platform #311 — @ContextParam Annotation (NEW)
Created `@ContextParam` annotation for server-side parameter resolution. Parameters
annotated with `@ContextParam("tenancyId")` are skipped in generated REST/GraphQL
signatures — the generator injects `CurrentPrincipal` and resolves the value.

Changes across 4 modules:
- `platform-api`: new `ContextParam.java` annotation
- `generator-common`: `ResolvedParam` + `McpDomainJandexScanner` updated
- `graphql-generator`: REST + GraphQL code generation skips context params
- `graphql-spring-generator`: Spring controller + REST controller writers updated

65 tests pass. Landed on local original platform main. NOT yet on GitHub origin/main.
Spring session told to pick it up from `issue-311-context-param` branch.

### Work #400 — Design + Plan Complete
Brainstorming completed with 3 decisions:
- D1: 5 SPI interfaces (work/items, work/lifecycle, work/notes, work/links, work/relations)
- D2: SPI impl beans delegate to WorkItemOperations (existing service layer)
- D3: New view records needed — WorkItem(57 fields) curated to WorkItemView(37 fields)

Design spec: `wsp-casehub-ledger/specs/issue-400-work-mcpdomain-spi/2026-09-15-work-api-generation-design.md`
Implementation plan: `wsp-casehub-ledger/plans/2026-09-15-work-api-generation.md`
3 batches, 5 tasks. Light reviews ran for decisions and spec.

## Resume Point

**Execute the work#400 plan starting at Batch 1, Task 1.**

Prerequisites before starting:
1. Create feature branch in work repo: `git -C /path/to/work checkout -b issue-400-work-mcpdomain-spi`
2. Ensure updated platform JARs (with @ContextParam) are in slot .m2
3. Open the slot work project in IntelliJ: `ide_open_project` with `/Users/mdproctor/claude/casehub/slots/194/work`

## Key Discoveries

**WorkItem field curation is deliberate.** WorkItem(57 fields) vs WorkItemResponse(37 fields)
— 20 internal fields excluded, 3 reshaped. This drove the D3 decision to create new view
records rather than reuse api types.

**WorkItemOperations IS the existing service layer.** 31-method interface in api/spi. The
decision review caught the incorrect claim that "work has no service layer." SPI impl beans
delegate to WorkItemOperations for lifecycle methods, inject stores directly for notes/links/relations.

**@ContextParam resolves tenancyId from authentication context.** End-user apps never see
tenancyId in REST query params or GraphQL arguments. Internal Java callers still pass it
explicitly via the SPI interface. Built-in keys: tenancyId, actorId.

**Spring migration feedback aligned with our approach.** The Spring session's briefing about
Pattern 1 vs Pattern 2 confirmed our SPI interface approach IS Pattern 2. No conflict.

## Standing Instructions

1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Platform JARs updated with @ContextParam — installed to both `~/.m2` and slot.
4. IntelliJ needs the slot work project opened for plan execution.

## References

| Artifact | Path |
|----------|------|
| Slot plan | `slots/194/.plan` |
| Work spec | `wsp-casehub-ledger/specs/issue-400-work-mcpdomain-spi/2026-09-15-work-api-generation-design.md` |
| Work plan | `wsp-casehub-ledger/plans/2026-09-15-work-api-generation.md` |
| Engine spec | `wsp-casehub-ledger/specs/issue-295-unified-api-generation/2026-09-15-engine-api-generation-design.md` |
| Decisions | `wsp-casehub-ledger/specs/issue-400-work-mcpdomain-spi/decisions.md` (D1-D3) |
| Decision review | `~/reviews/casehub-slots/issue-400-work-mcpdomain-decision-20260915-134058/` |
| Spec review | `~/reviews/casehub-slots/issue-400-work-mcpdomain-spec-*` |
