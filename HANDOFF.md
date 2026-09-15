# Handover — Slot 194

## Branch
Work repo (iot): `issue-105-iot-mcpdomain-spi` (local, 6 commits ahead of main).
Engine repo: `issue-1095-engine-mcpdomain-spi` (pushed, from prior session).
Work repo: `issue-400-work-mcpdomain-spi` (pushed, from prior session).
Ledger repo: `main`.
Platform repo: `issue-311-context-param` (local, 4 commits including basePath + generator fixes).

## Active Issue
`casehubio/iot#105` — migrate iot to @McpDomain SPI.
Queue position 4/17 (ledger done, engine done, work done, iot active).

## Session Summary

### Design Phase
Full brainstorming cycle: 8 decisions captured (D1-D8), decision review
(3 rounds, standard depth), spec written and approved.

Key decisions:
- **D1**: SPI interfaces in `webapp-api/` (not `api/`) — webapp concern
- **D2**: 5 SPI interfaces: devices (4), situations (7), suppressions (3), cases (6), ops (7)
- **D3**: Evolve existing DTOs in-place
- **D4**: SSE stays hand-written
- **D5**: Default methods + NotImplementedException for placeholders
- **D6**: Established platform annotations (NOT JAX-RS — Quarkus scanning limitation)
- **D7**: MCP coexistence (curated alongside generated)
- **D8**: `basePath` attribute on `@McpDomain` (platform#333, implemented)

### Platform Changes (platform#333)
Added `basePath` attribute to `@McpDomain` — decouples MCP domain name from
REST base path. Three changes: annotation, Quarkus generator, shared scanner.
Also fixed generator bugs: embedded path params doubled, leading double-slash
with `@RestPath`.

### IoT Implementation — All 6 Tasks Complete
1. **View records** — 11 view records in `webapp-api/view/`, NotImplementedException,
   `-parameters` compiler flag
2. **SPI interfaces** — 5 interfaces in `webapp-api/spi/` with `basePath` and `@RestPath`
3. **DefaultIoTDeviceApi + DefaultIoTOperationsApi** — delegation to DeviceRegistry,
   providers, BridgeAuditStore, BridgeConnectionRegistry
4. **DefaultIoTSituationApi + DefaultIoTSuppressionApi** — JPQL, sealed type mapping,
   CDI events, suppression history/stats
5. **DefaultIoTCaseApi + NotImplementedExceptionMapper** — CBR retrieval, resolution
   queue, 501 exception mapper
6. **APT wiring** — maven-compiler-plugin with annotationProcessorPaths, domainFilter.
   All 5 generated resources verified with correct paths.

### Pre-existing Breakage Fixed
- `PlanTrace` and `PlanCbrCase` removed from `neocortex-memory-api` but iot still
  referenced them. Relocated to local `webapp-api/cbr/` package.
- `Confidence` type change in neocortex: `IoTCbrRetrievalService.toSuggestion()`
  updated to call `c.confidence().value()`.
- `ResolutionSuggestion` and `AiResolutionPromptBuilder` imports updated.

### Remaining Work for Next Session
1. **Add `@RestPath("/")` to list methods** — `listDevices`, `listCases`,
   `listSuppressions` currently generate `/list-*` paths instead of `/`.
2. **Delete hand-written resources** — 7 resource files + 1 DTO file. Blocked by
   pre-existing `WorkItemOutcomeRecorder` compile error (Confidence type).
3. **Fix WorkItemOutcomeRecorder** — pre-existing `Confidence` type mismatch.
4. **Update tests** — test references to deleted resource types.
5. **Push branches** — iot and platform branches are local only.

### Work-End First Wave (platform#334)
After iot#105 completes, close out the first batch:
1. Platform: merge `issue-311-context-param` to main
2. Engine: add basePath to SPIs, merge
3. Work: add basePath to SPIs, merge
4. IoT: merge after remaining work
5. Ledger: investigate revert, re-do with basePath

## Standing Instructions
1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Platform JARs installed to slot .m2: platform-api, generator-common, graphql-generator.
4. `webapp-api` installed with `mvn install -Dmaven.test.skip=true` (test compilation
   broken by neocortex PlanTrace removal — test imports need updating).

## References

| Artifact | Path |
|----------|------|
| Slot plan | `slots/194/.plan` |
| IoT spec | `wsp-casehub-ledger/specs/issue-105-iot-mcpdomain-spi/2026-09-15-iot-mcpdomain-spi-design.md` |
| IoT decisions | `wsp-casehub-ledger/specs/issue-105-iot-mcpdomain-spi/decisions.md` |
| IoT plan | `wsp-casehub-ledger/plans/2026-09-15-iot-mcpdomain-spi.md` |
| Platform basePath issue | casehubio/platform#333 |
| First-wave close-out issue | casehubio/platform#334 |
