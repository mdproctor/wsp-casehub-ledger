# CaseHub Ledger — Session Handover
**Date:** 2026-05-13

## Current State

`casehub-ledger` v0.2-SNAPSHOT. Clean working tree (ignore untracked squash plans and `wksp` symlink). 366 tests, BUILD SUCCESS. Groups C (#63) and D (#65) shipped and pushed.

## What Landed This Session

**Groups C and D — trust federation and bootstrap:**
- `TrustExportService` — CDI read-model: `exportAll(minScore)` / `exportActor(actorId)` / `exportDelta(since)` → structured `TrustExportPayload` (GLOBAL / CAPABILITY / DIMENSION per actor)
- `TrustImportService` SPI — `importTrust(payload)`; `NoOpTrustImportService` @DefaultBean; `JpaTrustImportService` @Alternative (seed-if-absent)
- `TrustBootstrapSource` SPI — `fetchPriorTrust(actorId)`; `NoOpTrustBootstrapSource` @DefaultBean
- `TrustBootstrapService` — calls source per actor, delegates to import; wired into `TrustScoreJob` as batch pre-pass (gated by `casehub.ledger.trust-score.bootstrap.enabled=false`)
- All in `runtime/service/federation/` package

**Config additions:** `LedgerConfig.TrustScoreConfig.ExportConfig` (`deploymentId: Optional<String>`) and `BootstrapConfig` (`enabled: boolean`)

**Repo method added:** `ActorTrustScoreRepository.findAllByLastComputedAtAfter(Instant)`

**Doc/routing fix:** Specs and ADRs now route to project repo (`docs/specs/`, `docs/adr/`) at epic close. CLAUDE.md routing updated. Session spec promoted.

**#75 closed:** ActorTypeResolver A2A mappings (pre-existing commit, issue wasn't linked).

## Key Decisions

- `HttpTrustBootstrapSource` and REST export endpoint cut — no concrete multi-deployment topology yet; SPI exists for when needed
- `TrustImportService` implementation IS the strategy (no `MergeStrategy` enum)
- `@WithDefault("") String` causes SmallRye Config boot failure → `Optional<String>` (garden GE-20260513-a2f5b7)
- `@Alternative` inner classes in `@QuarkusTest` require explicit `selected-alternatives` — not auto-discovered (garden GE-20260512-c246b0 corrected)

## Open Issues

- #76 — `CAPABILITY_DIMENSION` composite trust score (next candidate)
- #72 — cross-repo quality issues from sweep (still open)
- #65, #64, #63 — closed ✅
- Epics #51, #52 — closed ✅

## Immediate Next Steps

Check remaining open work: `gh issue list --repo casehubio/ledger`
Most likely next: `#76` — CAPABILITY_DIMENSION composite trust score

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-13-mdp01-trust-without-ceremony.md` |
| Latest spec (project) | `docs/specs/2026-05-12-trust-federation-bootstrap-design.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
