# Handover — Slot 194

## Branch
`main` — all repos on main, no active feature branch.

## Active Issue
`casehubio/engine#1095` — migrate engine REST/GraphQL to @McpDomain SPI.
Queue position 1/17 (ledger done, engine next).

## Session Summary

Two work items completed:

1. **Ledger #207 closed** — hand-written REST/GraphQL replaced with APT-generated
   endpoints. 4 squashed commits merged to main, pushed. Full work-end cycle
   (review, branch audit, forage, squash, land).

2. **Platform #300 completed** — 10 generator improvements: response codes (201),
   null→404, @PlatformStream (SSE + subscriptions), @Operation from description,
   @RestName, @RolesAllowed pass-through, @PaginatedResponse, thread dispatch
   awareness. 51 tests. Merged to main, pushed. Spring session's shared scan
   model + graphql-spring-generator rewrite rebased in.

## Engine Brainstorming State

Exploration started but not committed to spec. Findings so far:
- 6 standard REST resources migratable (CaseInstance, CaseControl, CaseDefinition,
  Plan, EventLog, Signal)
- 2 SSE resources now migratable with @PlatformStream (CaseStream, ExecutionState)
- 2 GraphQL query/mutation resolvers migratable
- 1 GraphQL subscription resolver now migratable with @PlatformStream
- CaseInstanceResource has inline business logic (goal evaluation, completion
  computation) — needs extraction to service layer before migration
- CaseService in rest/service/ is a partial service layer

## Standing Instructions

1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Generator JARs updated in slot .m2 — platform-api, graphql-generator,
   generator-common all at latest.

## Known Issues

- Ledger main has a revert of #207 changes (upstream). The slot's fork has
  the work landed. This needs reconciliation if ledger re-migration is needed.
- Platform `persistence-jpa` module broken (Panache removal WIP from Spring
  session) — skip when building platform full.

## References

| Artifact | Path |
|----------|------|
| Slot plan | `slots/194/.plan` |
| Generator spec | `wsp-casehub-ledger/specs/issue-300-generator-improvements/` |
| Generator plan | `wsp-casehub-ledger/plans/2026-09-15-generator-improvements.md` |
| Diary entry | `wsp-casehub-ledger/blog/2026-09-14-mdp01-one-spi-three-surfaces.md` |
| Garden entries | GE-20260914-638e46 (null→200 regression), GE-20260914-714a71 (APT param names) |
