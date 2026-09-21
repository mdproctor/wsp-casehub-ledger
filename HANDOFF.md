# Handover — Slot 194

## Active Issue
`casehubio/platform#378` — fix 104 Object returns with typed records
Queue position 1/6 (hardening phase).

## Context

The @McpDomain migration (24/24 original queue items) is complete. This slot is now in **hardening phase** — fixing type safety, config gaps, and generator bugs identified in the final audit.

## Queue (platform#377 children)

1. **platform#378** ← active — Replace 104 `Object`-returning @McpDomain methods with typed records across ops(29), claudony(27), fsitrading(26), life(15), chat-app(5)
2. **ledger#210** — Wire APT generator for 5 existing @McpDomain classes
3. **work#405** — @McpDomain for 25 bare REST resources (queues, federation, bulk ops)
4. **platform#379** — Generator bugs: @PaginatedResponse shadowing, @Valid dep, @DefaultValue
5. **platform#380** — @HandWrittenEndpoint or delete old REST across 13 repos
6. **platform#381** — Consolidation: shared ApiResult, merge single-method classes

## Standing Rules

- **Type safety is non-negotiable** — no `Object` returns, no raw types, proper generics always
- Inject services directly — never delegate to REST resources via `.getEntity()`
- `List<T>`, `Map<K,V>`, `Optional<T>` — always parameterised
- `Map<String,Object>` → create a typed record when the structure is known

## Ecosystem State

- 18 repos with @McpDomain tri-channel parity
- Build enforcement active (platform#374)
- All @Tool annotations stripped (except platform infra)
- Qhorus migrated separately (113 ops, verified green)
