# Handover — Slot 194

## Active Issue
`casehubio/platform#380` — not started. Queue position 29/32.

## Context

The @McpDomain migration (platform#300 epic) is in its final stretch. This session closed two issues: work#405 (25 bare REST resources) and platform#379 (three generator bugs). Three issues remain, including a new evaluation issue (#382) added at the end.

## What Was Done

### work#405 — @McpDomain for 25 bare REST resources (closed)

Squash-merged branch `issue-405-mcpdomain-rest` to main as `db02951b`. Branch stamped, issue closed on GitHub. All 25 production REST resources in casehub-work now have @McpDomain coverage.

### platform#379 — Generator bugs (closed)

Landed as `2e61d3e5` on platform main. Three fixes across both Quarkus (`GraphQLResolverProcessor`) and Spring (`SpringDomainRestControllerWriter`) REST generators:

1. **`var page` shadowing** — renamed generated variable to `__pageResult` to avoid colliding with method parameters named `page`
2. **`@Valid` conditional** — `@jakarta.validation.Valid` now only propagated when present on the SPI parameter; previously added unconditionally for all body params
3. **`@DefaultValue` propagation** — new `@io.casehub.platform.api.mcp.DefaultValue` annotation; scanned by both Jandex and APT scanners; propagated to `@RequestParam(defaultValue=...)` (Spring) and `@jakarta.ws.rs.DefaultValue` (Quarkus)

### platform#382 — New evaluation issue created

Added end-of-epic evaluation: real-world LLM usability testing of @McpDomain hierarchy, discovery, indexing, and cross-repo automation. Evaluation-only scope — findings drive follow-up issues.

## Queue (platform#300 children)

1. ~~**platform#378**~~ done
2. ~~**ledger#210**~~ done
3. ~~**work#405**~~ done
4. ~~**platform#379**~~ done
5. **platform#380** — @HandWrittenEndpoint or delete old REST across 13 repos ← active
6. **platform#381** — Consolidation: shared ApiResult, merge single-method classes
7. **platform#382** — Eval: @McpDomain real-world LLM usability — hierarchy, discovery, indexing

## Notes for Next Session

- platform#380 is M scale, Low complexity — triage all remaining `@Path` classes across 13 repos: delete (covered by @McpDomain), annotate @HandWrittenEndpoint (webhook/SSE/auth), or flag as gap
- Some work modules (queues, ai) are commented out of the parent reactor (tracked by work#403) — deleting old REST resources may help re-enable them
- platform#381 is consolidation — shared ApiResult, merge single-method classes
- platform#382 is evaluation-only — hands-on testing with Claude against live endpoints, document findings, propose improvements

## Standing Rules

- **Type safety is non-negotiable** — no `Object` returns, no raw types, proper generics always
- Inject services directly — never delegate to REST resources via `.getEntity()`
- `List<T>`, `Map<K,V>`, `Optional<T>` — always parameterised
- `Map<String,Object>` → create a typed record when the structure is known
