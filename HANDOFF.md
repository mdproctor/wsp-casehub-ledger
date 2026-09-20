# Handover — Slot 194

## Branch
SOC on branch `issue-55-mcpdomain-spi` (9 commits, 474 tests pass, issue closed).
Connectors on branch `issue-100-mcpdomain-full-parity` (2 commits, 608 tests pass, issue closed).
All other repos on main.

## Active Issue
`casehubio/life#118` — migrate life to @McpDomain SPI.
Queue position 8/24 (8 done, 16 remaining).

## Session Summary

### Completed This Session

| Issue | Repo | Result |
|---|---|---|
| soc#55 | casehub-soc | 7 @McpDomain classes, 24 DTOs, 474 tests green |
| connectors#100 | casehub-connectors | 4 @McpDomain classes (18 ops), mcp/ module deleted, 608 tests green |

### CDI Fixes (cross-repo)
- engine: `@Vetoed` on ActorState, CDI ambiguity fixes (pushed to origin)
- eidos: removed `@DefaultBean` from `DefaultCapabilityHealth`
- neocortex: trust imports fixed (upstream deleted the file)

### Key Lessons
- APT generator must be in pom.xml compiler plugin (`annotationProcessorPaths`)
- `@PaginatedResponse` has a variable shadowing bug — avoid until fixed
- Connector test configs needed: twilio, whatsapp, slack, teams in `application.properties`
- `@Tool` MCP classes are the old approach — replace with `@McpDomain` for tri-channel parity

## What's Next
`work continue` → life#118 (migrate life to @McpDomain SPI)

## Standing Instructions
1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` — all slot repos installed.
3. `casehub-platform-graphql-generator` is the APT processor.
