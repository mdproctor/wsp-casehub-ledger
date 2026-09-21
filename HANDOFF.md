# Handover — Slot 194 (COMPLETE — ready to archive)

## Queue: 24/24 done

The @McpDomain SPI migration across the CaseHub ecosystem is complete.
Every user-facing operation now has tri-channel REST + GraphQL + MCP parity.
Build enforcement prevents drift.

## Deliverables

- **~140 @McpDomain classes** across 18 repos
- **~260+ operations** with tri-channel parity
- **Build enforcement** (platform#374) — `@HandWrittenEndpoint` + `McpDomainEnforcementProcessor`
- **All @Tool annotations stripped** (except platform infra)
- **CDI fixes** across engine, eidos, neocortex
- **Qhorus** — 113 ops migrated separately, verified green

## Follow-up Epic: platform#377

Post-migration hardening — 14 items covering:
- Type safety: 104 `Object` returns → typed (ops, claudony, fsitrading, life, chat-app)
- Config: ledger APT generator, domain filter gaps (work, life, devtown)
- Coverage: work has 25 bare REST resources
- Generator bugs: @PaginatedResponse shadowing, @Valid dep, @DefaultValue propagation
- Consolidation: shared ApiResult, merge single-method classes, extract inner records

## Standing Rule

**Type safety is non-negotiable.** No `Object` returns, no raw types, proper generics always. Inject services directly — never delegate to REST resources via `.getEntity()`. See feedback memory `type-safety-generics`.
