# Handover — Slot 194

## Branch
Engine repo: `issue-1095-engine-mcpdomain-spi` (feature branch, 2 commits ahead of main).
Ledger repo: `main`.

## Active Issue
`casehubio/engine#1095` — migrate engine REST/GraphQL to @McpDomain SPI.
Queue position 1/17 (ledger done, engine active).

## Session Summary

Completed brainstorming, spec, plan, and Batch 1 execution for engine #1095:

1. **Brainstorming** — full codebase scan (8 REST resources, 3 GraphQL resolvers,
   24 endpoints, 13 methods). 3 engine-specific decisions captured (D7-D9):
   5 SPI interfaces by function, impls in rest/, all streaming via @PlatformStream.

2. **Design spec written** — `2026-09-15-engine-api-generation-design.md`. Self-review
   caught 6 issues (wrong package, missing tenancyId, CaseService injection claim,
   -parameters flag, return types, missing DTO). All fixed.

3. **Implementation plan written** — `2026-09-15-engine-api-generation.md`. 3 batches,
   6 tasks.

4. **Batch 1 executed** — Foundation:
   - 18 view records + 3 request types in `io.casehub.api.view` (api/ module)
   - 5 SPI interfaces in `io.casehub.api.spi` with @McpDomain annotations
   - 1426 api/ module tests passing (16 new)

## Resume Point

**Batch 2, Task 3: Extract goal evaluation to CaseService.**

Remaining tasks:
- Task 3: Extract CaseInstanceResource.getGoals() inline logic → CaseService.evaluateGoals()
- Task 4: Create 5 SPI implementation beans in rest/service/
- Task 5: Wire APT generator in rest/ and graphql/ pom.xml
- Task 6: Delete hand-written endpoints, update tests

## Key Discovery During Execution

**EnginePlanApi return types:** api/ module can't reference types from common-core/
(dependency goes common-core → api/, not reverse). Methods `getPlanModel`,
`getDagResult`, `getExecutionState`, and `executionStateStream` return `Object`
instead of concrete snapshot types. Jackson serializes correctly; GraphQL maps
to JSON scalar. Type safety enforced in the impl, not the SPI interface.

**platform-api needed rebuild:** Slot .m2 had stale platform-api without
@PaginatedResponse, @PlatformStream, @RestStatus. Rebuilt from slot's platform
source and installed to slot .m2.

## Standing Instructions

1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Generator JARs updated in slot .m2 — platform-api rebuilt this session.
4. IntelliJ indexes the main engine repo (`~/claude/casehub/engine`), not the
   slot clone. Use Write tool for slot files, not ide_create_file.

## Known Issues

- Ledger main has a revert of #207 changes (upstream). The slot's fork has
  the work landed. This needs reconciliation if ledger re-migration is needed.
- Platform `persistence-jpa` module broken (Panache removal WIP from Spring
  session) — skip when building platform full.
- Files created via ide_create_file in prior attempts exist in the main engine
  repo at `~/claude/casehub/engine/api/src/main/java/io/casehub/api/view/` and
  `~/claude/casehub/engine/api/src/main/java/io/casehub/api/spi/` — these are
  duplicates of the slot files. Clean up when convenient.

## References

| Artifact | Path |
|----------|------|
| Slot plan | `slots/194/.plan` |
| Engine spec | `wsp-casehub-ledger/specs/issue-295-unified-api-generation/2026-09-15-engine-api-generation-design.md` |
| Engine plan | `wsp-casehub-ledger/plans/2026-09-15-engine-api-generation.md` |
| Ledger spec (reference) | `wsp-casehub-ledger/specs/issue-295-unified-api-generation/2026-09-14-unified-api-generation-design.md` |
| Decisions | `wsp-casehub-ledger/specs/issue-295-unified-api-generation/decisions.md` (D1-D9) |
| Garden entries | GE-20260914-638e46 (null→200), GE-20260914-3854b8 (domain filter), GE-20260914-f53be7 (Jandex APT), GE-20260914-714a71 (param names), GE-20260804-8b0fd6 (BroadcastProcessor), GE-20260818-c2f072 (MCP dispatch test) |
