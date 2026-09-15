# Handover — Slot 194

## Branch
Engine repo: `issue-1095-engine-mcpdomain-spi` (feature branch, 5 commits ahead of main).
Ledger repo: `main`.

## Active Issue
`casehubio/engine#1095` — migrate engine REST/GraphQL to @McpDomain SPI.
Queue position 1/17 (ledger done, engine active).

## Session Summary

Completed Batch 2 execution (Tasks 3-4) for engine #1095:

1. **Task 3: Extract goal evaluation** — moved ~75 lines of inline goal
   evaluation logic from `CaseInstanceResource.getGoals()` to
   `CaseService.evaluateGoals(UUID, String)`. Includes `buildCompletionSummary()`
   private helper. `ExpressionEngineRegistry` injection added to CaseService.
   CaseInstanceResource now delegates with a single line.

2. **Task 4: Create 5 SPI implementation beans** — all in
   `io.casehub.engine.rest.service`:
   - `DefaultEngineCaseApi` (10 methods): CRUD, context, goals, plan items, 3 streams
   - `DefaultEngineCaseControlApi` (4 methods): suspend, resume, cancel, signal
   - `DefaultEngineCaseDefinitionApi` (3 methods): list, by-name, by-key
   - `DefaultEngineEventLogApi` (1 method): paginated+filtered event log
   - `DefaultEnginePlanApi` (7 methods): plan model, definitions, decomposition,
     DAG, execution state, execution state stream

   Each bean delegates to existing services and maps to `api/view/` records.
   All compile clean via IntelliJ diagnostics.

## Resume Point

**Batch 3, Task 5: Wire APT generator in rest/ and graphql/ modules.**

Remaining tasks:
- Task 5: Add `casehub-platform-graphql-generator` as annotation processor in
  rest/pom.xml and graphql/pom.xml with domain filtering
- Task 6: Delete hand-written REST resources, GraphQL resolvers, and old DTOs;
  update tests to hit generated endpoint paths

## Key Discoveries During Execution

**CaseMetaModelRepository uses CaseDefinitionQuery, not CaseInstanceQuery.**
The definition API's query method takes `CaseDefinitionQuery` (in
`io.casehub.engine.common.spi.query`) which has namespace/name/page/size — not
`CaseInstanceQuery` which also has status filtering.

**Capability is a record** — `io.casehub.worker.api.Capability` is a Java record,
so accessor is `name()` not `getName()`.

**GoalEvaluationResponse → GoalEvaluationView mapping is lossy.** The existing DTO
has a rich `CompletionSummary` with per-kind `byKind` map. The new
`CompletionSummaryView` is simpler (complete/satisfied/total/kind). The SPI impl
maps by counting satisfied kinds for goal-based, or using the boolean for
predicate-based completion.

**Lifecycle and contextChange streams not wired.** `DefaultEngineCaseApi` returns
`Multi.empty()` for `caseLifecycle` and `caseContextChange` streams — no
broadcasters exist for these in rest/. The graphql module's `CaseEventPublisher`
handles these via CDI events. Wiring can happen in Task 6 when the generated
endpoints replace hand-written ones.

**Rest module QuarkusTest runtime broken** (pre-existing). Ledger JPA entities on
the classpath require a datasource not configured in the engine rest test profile.
The `CaseInstanceGoalsResourceTest` can't run via `mvn test -pl rest`. IntelliJ
diagnostics confirm compilation is clean. This needs a test config fix (add
`quarkus.hibernate-orm.packages` exclusion or a datasource) before Task 6.

## Standing Instructions

1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Generator JARs updated in slot .m2 — platform-api rebuilt prior session.
4. IntelliJ now has the slot engine project open — use `project_path=/Users/mdproctor/claude/casehub/slots/194/engine`.
5. Pre-existing build errors in `schema/` module (Worker/Agent class) — unrelated to SPI migration.

## References

| Artifact | Path |
|----------|------|
| Slot plan | `slots/194/.plan` |
| Engine spec | `wsp-casehub-ledger/specs/issue-295-unified-api-generation/2026-09-15-engine-api-generation-design.md` |
| Engine plan | `wsp-casehub-ledger/plans/2026-09-15-engine-api-generation.md` |
| Ledger spec (reference) | `wsp-casehub-ledger/specs/issue-295-unified-api-generation/2026-09-14-unified-api-generation-design.md` |
| Decisions | `wsp-casehub-ledger/specs/issue-295-unified-api-generation/decisions.md` (D1-D9) |
| Garden entries | GE-20260914-638e46 (null→200), GE-20260914-3854b8 (domain filter), GE-20260914-f53be7 (Jandex APT), GE-20260914-714a71 (param names), GE-20260804-8b0fd6 (BroadcastProcessor), GE-20260818-c2f072 (MCP dispatch test) |
