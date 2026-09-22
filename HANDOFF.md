# Handover — Slot 194

## Active Issue
`casehubio/ledger#210` — closed this session (APT generator wired).
Queue position 26/31 — advance needed to activate next issue.

## Context

The @McpDomain migration (24/24 original queue items) and **hardening phase** are progressing. Two issues closed this session:

1. **platform#378** (closed) — 119 Object-returning @McpDomain methods replaced with typed records across 5 repos
2. **ledger#210** (closed) — APT generator wired for @McpDomain classes in ledger rest module

## What Was Done

### platform#378 — Type Safety (119 methods across 5 repos)

All `.getEntity()` delegation eliminated. Services injected directly with typed returns:

| Repo | Methods Fixed | Commit | Branch |
|------|--------------|--------|--------|
| chat-app | 5 | `38c5ac5` | `issue-42-ux-overhaul` |
| life | 17 | `4efc47d` | `issue-118-mcpdomain-spi` |
| ops | 29 | `8dc59e2` | `issue-90-mcpdomain-spi` |
| claudony | 26 + fix | `c4ae2cd`, `1a778c2` | `issue-204-mcpdomain-spi` |
| fsitrading | 42 | `0105e23` | `issue-47-mcpdomain-spi` |

All repos compile clean (only pre-existing errors in unrelated files).

### ledger#210 — APT Generator Wiring

- Moved 4 @McpDomain classes from `runtime/service/api/` to `rest/api/` (matches engine pattern — extension runtime jars can't have `@RunOnVirtualThread` endpoints)
- Added `casehub-platform-graphql-generator` APT to `rest/pom.xml`
- Domain filter: `ledger/entries,ledger/attestations,ledger/trust,ledger/verification`
- Deleted 4 hand-written REST resource classes + 3 tests (generated endpoints replace them)
- All 13 modules pass

## Queue (platform#377 children)

1. ~~**platform#378**~~ done — Replace 104 Object returns with typed records
2. ~~**ledger#210**~~ done — Wire APT generator for 5 existing @McpDomain classes
3. **work#405** — @McpDomain for 25 bare REST resources (queues, federation, bulk ops)
4. **platform#379** — Generator bugs: @PaginatedResponse shadowing, @Valid dep, @DefaultValue
5. **platform#380** — @HandWrittenEndpoint or delete old REST across 13 repos
6. **platform#381** — Consolidation: shared ApiResult, merge single-method classes

## Standing Rules

- **Type safety is non-negotiable** — no `Object` returns, no raw types, proper generics always
- Inject services directly — never delegate to REST resources via `.getEntity()`
- `List<T>`, `Map<K,V>`, `Optional<T>` — always parameterised
- `Map<String,Object>` → create a typed record when the structure is known

## Notes for Next Session

- The .plan still shows ledger#210 as active — run `work next` to advance to work#405
- Commits are on feature branches in each repo, not on main — need work-end per repo to land them
- claudony in IntelliJ workspace is from slot 202, not 194 — verify before editing
