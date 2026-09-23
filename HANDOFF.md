# Handover — Slot 194

## Active Issue
`casehubio/platform#401` — @PlatformWebhook generator support. Queue position 34/36.

## Context

The @McpDomain migration (platform#300 epic) is nearing completion. This session completed #403 (SSE migration). Two issues remain: #401 (webhook generator feature) and #404 (webhook consumer migration).

## What Was Done

### platform#403 — migrate SSE resources to @PlatformStream (closed)

Migrated 4 hand-written SSE resources across 4 repos to `@PlatformStream` methods on `@McpDomain` classes. The generator produces `@GET` + `@Produces(SERVER_SENT_EVENTS)` + `@RestStreamElementType(APPLICATION_JSON)` endpoints that delegate to the domain class methods.

| Repo | Old Resource | New Location | Complexity |
|---|---|---|---|
| openclaw | `ScenarioSseResource` | `OpenClawScenarioApi.watchEvents()` | Simple — listener pattern |
| ops | `ReconciliationResource` | `OpsReconciliationApi.watchEvents()` | Simple — old endpoint was a stub |
| life | `LifeEventSseResource` | `LifeEventStreamApi` (new @McpDomain class, 3 endpoints) | Moderate — 3 filtered streams |
| iot | `DeviceSseResource` | `DefaultIoTDeviceApi.streamDevices()` | Complex — snapshot merge, CDI observer, tenancy filtering |

Key decisions:
- **iot typed response:** Replaced raw `Multi<String>` JSON with typed `DeviceStreamEvent(operation, data)` record
- **iot tenancy:** Changed from injected `CurrentPrincipal` to `@ContextParam("tenancyId")` method parameter
- **life heartbeat dropped:** Removed 30s keepalive heartbeat — RESTEasy Reactive handles SSE keepalive via `quarkus.rest.sse.keepalive-interval`
- **life new domain:** Created `@McpDomain("life/events")` rather than adding to an existing domain — event streams are cross-cutting
- **ops stub upgraded:** Old `/events` endpoint returned JSON metadata, not actual SSE. New method streams real reconciliation events from `ApplicationEventBroadcaster`

All 4 repos build cleanly (verified via IDE build). Docs and ARC42STORIES updated in each repo.

## Uncommitted Changes Across Repos

All repos clean. Commits on branches:
- **openclaw** (`issue-77-mcpdomain-spi`): 1 commit — SSE migration
- **ops** (`issue-90-mcpdomain-spi`): 1 commit — SSE migration
- **life** (`issue-118-mcpdomain-spi`): 1 commit — SSE migration
- **iot** (`main`): 1 commit — SSE migration

## Queue (platform#300 children)

1. ~~**platform#403**~~ done
2. **platform#401** — @PlatformWebhook generator support ← active
3. **platform#404** — migrate webhook resources to @PlatformWebhook

## Notes for Next Session

- #401 is a platform generator feature (new annotation + APT code), not a consumer migration — needs brainstorming and TDD
- #401 and #404 are paired: generator feature first, then consumer migration
- The `@PlatformStream` implementation in `GraphQLResolverProcessor` (lines ~1020-1080) is the template for `@PlatformWebhook` — check how STREAM is handled and follow the same pattern for WEBHOOK
- Platform branch is `issue-381-consolidation` (accumulated work from #381, #382, #400)
- All repos have the platform SNAPSHOT in local .m2 cache
- New improvement issues filed previously (#406-#409) are not in the current queue

## Standing Rules

- **Type safety is non-negotiable** — no `Object` returns, no raw types, proper generics always
- Inject services directly — never delegate to REST resources via `.getEntity()`
- `List<T>`, `Map<K,V>`, `Optional<T>` — always parameterised
- `Map<String,Object>` → create a typed record when the structure is known
- **Delete only what the generator fully replaces** — method-by-method verification before deletion
