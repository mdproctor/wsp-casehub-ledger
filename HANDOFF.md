# Handover — Slot 194

## Active Issue
`casehubio/platform#379` — in progress. Branch `issue-379-generator-bugs` in platform repo (1 commit).
Queue position 28/31.

## Context

The @McpDomain migration (platform#300 epic) continues. This session closed work#405 (merged to main, issue closed) and advanced to platform#379 — three APT generator bugs.

## What Was Done

### work#405 — @McpDomain for 25 bare REST resources (closed)

Squash-merged branch `issue-405-mcpdomain-rest` to main as `db02951b`. Branch stamped, issue closed on GitHub. All 25 production REST resources in casehub-work now have @McpDomain coverage.

### platform#379 — Generator bugs (in progress)

Three fixes across both Quarkus (`GraphQLResolverProcessor`) and Spring (`SpringDomainRestControllerWriter`) REST generators:

1. **`var page` shadowing** — renamed generated variable to `__pageResult` to avoid colliding with method parameters named `page`
2. **`@Valid` conditional** — `@jakarta.validation.Valid` now only propagated when present on the SPI parameter; previously added unconditionally for all body params
3. **`@DefaultValue` propagation** — new `@io.casehub.platform.api.mcp.DefaultValue` annotation; scanned by both Jandex and APT scanners; propagated to `@RequestParam(defaultValue=...)` (Spring) and `@jakarta.ws.rs.DefaultValue` (Quarkus)

Files changed: `ResolvedParam` (both copies), `McpDomainJandexScanner`, `GraphQLResolverProcessor`, `SpringDomainRestControllerWriter`, `DefaultValue.java` (new). Tests added for all three fixes.

Full platform build running — generator-common, graphql-generator, and graphql-spring-generator tests already verified green individually.

## Queue (platform#300 children)

1. ~~**platform#378**~~ done — Replace 104 Object returns with typed records
2. ~~**ledger#210**~~ done — Wire APT generator for 5 existing @McpDomain classes
3. ~~**work#405**~~ done — @McpDomain for 25 bare REST resources
4. **platform#379** — Generator bugs: @PaginatedResponse shadowing, @Valid dep, @DefaultValue ← active
5. **platform#380** — @HandWrittenEndpoint or delete old REST across 13 repos
6. **platform#381** — Consolidation: shared ApiResult, merge single-method classes

## Standing Rules

- **Type safety is non-negotiable** — no `Object` returns, no raw types, proper generics always
- Inject services directly — never delegate to REST resources via `.getEntity()`
- `List<T>`, `Map<K,V>`, `Optional<T>` — always parameterised
- `Map<String,Object>` → create a typed record when the structure is known
