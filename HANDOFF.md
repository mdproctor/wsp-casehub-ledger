# Handover — Slot 194

## Active Issue
`casehubio/platform#403` — migrate 4 SSE resources to @PlatformStream. Queue position 33/36.

## Context

The @McpDomain migration (platform#300 epic) is nearing completion. This session completed three issues: consolidation (#381), LLM usability evaluation (#382), and @PlatformStream runtime discovery (#400). Three issues remain in the queue.

## What Was Done

### platform#381 — Consolidation: shared ApiResult, merge single-method classes (closed)

- Created `ApiResult(boolean ok, String id, String detail)` in platform-api — shared result record
- Merged `ClaudonyActionApi` (1 method) into `ClaudonyCaseApi` at `/actions` sub-path
- Merged `FsiAuditApi` (1 method) into `FsiComplianceApi` at `/audit/orders/{orderId}`
- Extracted `PostMessageResult` from chat-app `ChatMessageApi` to standalone file
- Replaced `MoveChannelResult` in chat-app with `ApiResult`
- Replaced `WorkitemResult` and `CommitmentResult` in openclaw with `ApiResult`
- Net: 2 @McpDomain classes eliminated, 2 per-repo result records consolidated

### platform#382 — Eval: @McpDomain real-world LLM usability (closed)

Structural evaluation of the full CaseHub MCP surface: 142 domains, 607 operations across 16 repos.

Key findings:
- Hierarchy works for single-app, breaks at org scale (142 domains in flat list)
- Discovery is browse-only — keyword search is highest-impact improvement
- Return type erasure (List not List<LedgerEntry>) hampers LLM understanding

Filed 4 improvement issues:
- platform#406 — `casehub_search` tool (keyword search across operations)
- platform#407 — app-level grouping in `casehub_model`
- platform#408 — build-time summary validation
- platform#409 — return type generics in discovery output

Evaluation spec: `specs/eval-mcpdomain-llm-usability.md`

### platform#400 — @PlatformStream generator support (closed)

Found that `@PlatformStream` annotation and both APT generators (Quarkus + Spring) already existed. The gap was runtime MCP discovery:
- Added `STREAM` to `OperationDescriptor.OperationType`
- `GraphQLModelScanner` now scans `@PlatformStream` methods
- `DomainContentFormatter` shows streams in `casehub_model` output
- Stream operations excluded from `casehub_action`/`casehub_activate` (Multi<T> can't serialize)

## Uncommitted Changes Across Repos

All repos are clean. Commits on feature branches:
- **platform** (`issue-381-consolidation`): ApiResult + stream discovery (2 commits)
- **claudony** (`issue-204-mcpdomain-spi`): merged ActionApi into CaseApi
- **fsitrading** (`issue-47-mcpdomain-spi`): merged AuditApi into ComplianceApi
- **chat-app** (`issue-42-ux-overhaul`): extracted PostMessageResult, replaced MoveChannelResult
- **openclaw** (`issue-77-mcpdomain-spi`): replaced WorkitemResult/CommitmentResult with ApiResult

## Queue (platform#300 children)

1. ~~**platform#381**~~ done
2. ~~**platform#382**~~ done
3. ~~**platform#400**~~ done
4. **platform#403** — migrate 4 SSE resources to @PlatformStream ← active
5. **platform#401** — @PlatformWebhook generator support
6. **platform#404** — migrate webhook resources to @PlatformWebhook

## Notes for Next Session

- platform#403 (S / Low, blocked by #400 which is now done): migrate iot DeviceSseResource, life LifeEventSseResource, ops ReconciliationResource (SSE portion), openclaw ScenarioSseResource. **Caveat:** iot's DeviceSseResource has complex business logic (snapshot merging, tenancy filtering, CDI event observation) — may not fit a simple @PlatformStream pass-through. Evaluate each resource individually.
- #401→#404 are paired: webhook generator feature first, then consumer migration
- All repos are rebased against origin/main as of 2026-09-23
- Platform branch is `issue-381-consolidation` (covers both #381 and #400 work)
- New improvement issues filed: #406, #407, #408, #409 — not in the current queue, future work

## Standing Rules

- **Type safety is non-negotiable** — no `Object` returns, no raw types, proper generics always
- Inject services directly — never delegate to REST resources via `.getEntity()`
- `List<T>`, `Map<K,V>`, `Optional<T>` — always parameterised
- `Map<String,Object>` → create a typed record when the structure is known
- **Delete only what the generator fully replaces** — method-by-method verification before deletion
