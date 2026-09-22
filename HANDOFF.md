# Handover — Slot 194

## Active Issue
`casehubio/platform#381` — not started. Queue position 30/36.

## Context

The @McpDomain migration (platform#300 epic) is in its final stretch. This session completed platform#380 (REST resource triage across 17 repos), #387 (clean generated names — done by user), #388 (inner record extraction), and #389 (SPI injection anti-pattern). Six issues remain in the queue.

## What Was Done

### platform#380 — @HandWrittenEndpoint or delete old REST (closed)

Triaged all remaining hand-written `@Path` REST resources across 17 repos + qhorus:

- **~55 old resources deleted** — each verified to have full method coverage by its @McpDomain-generated counterpart
- **17 gap SPIs created** — resources that had no @McpDomain SPI were converted (fsitrading 11, qhorus 3, iot 2, clinical 1, devtown 1)
- **33 genuine @HandWrittenEndpoint** — webhooks, SSE, auth, A2A, connectors, game simulation
- **SPI injection anti-pattern fixed** — 7 repos had SPIs injecting old REST resources directly; refactored to inject services
- **Inner records extracted** — chat-app, ops, fsitrading had inner record types blocking deletion; extracted to standalone classes
- **Object return types fixed** — multiple SPIs had `Object` returns; replaced with typed records

### platform#387 — Drop Generated prefix (landed by user)

Generator now produces clean class names (`ChatMessagesResource` not `GeneratedChatMessagesResource`) in project-local packages (`.rest` not `.platform.rest.generated`). Package derivation: replace `.api` segment with `.rest`/`.graphql`.

### platform#388, #389 — Inner records + SPI injection (closed)

Both completed as part of #380 work. All inner records extracted, all SPI injection anti-patterns resolved.

### New issues filed

- **platform#400** — `@PlatformSse` generator support for SSE streaming (M / Med)
- **platform#401** — `@PlatformWebhook` generator support for webhook endpoints (L / High)
- **platform#403** — migrate 4 SSE resources after #400 lands (S / Low, blocked by #400)
- **platform#404** — migrate 8 webhook resources after #401 lands (S / Low, blocked by #401)

### Rebase status

17 of 18 repos rebased against canonical main. Conflicts resolved for iot and claudony. AML skipped (concurrent work from slot 181).

## Queue (platform#300 children)

1. ~~**platform#380**~~ done
2. **platform#381** — Consolidation: shared ApiResult, merge single-method classes ← active
3. **platform#382** — Eval: @McpDomain real-world LLM usability
4. **platform#400** — @PlatformSse generator support
5. **platform#403** — migrate SSE resources (blocked by #400)
6. **platform#401** — @PlatformWebhook generator support
7. **platform#404** — migrate webhook resources (blocked by #401)

## Notes for Next Session

- platform#381 is M scale, Med complexity — shared ApiResult type, merge single-method @McpDomain classes
- platform#382 is evaluation-only — hands-on LLM testing, document findings
- #400→#403 and #401→#404 are paired: generator feature first, then consumer migration
- AML slot clone has rebase conflicts (7 CBR files) — leave for slot 181
- The 33 remaining @HandWrittenEndpoint resources trend toward zero as #400/#401 land (12 of 33 become generatable)

## Standing Rules

- **Type safety is non-negotiable** — no `Object` returns, no raw types, proper generics always
- Inject services directly — never delegate to REST resources via `.getEntity()`
- `List<T>`, `Map<K,V>`, `Optional<T>` — always parameterised
- `Map<String,Object>` → create a typed record when the structure is known
- **Delete only what the generator fully replaces** — method-by-method verification before deletion
