# CaseHub Ledger — Session Handover
**Date:** 2026-05-14

## Current State

`casehub-ledger` v0.2-SNAPSHOT. Clean working tree. 390 tests, BUILD SUCCESS. Issue #76 closed and merged to main.

## What Landed This Session

**#76 — CAPABILITY_DIMENSION composite trust score:**

- **Schema:** V1001 rewritten in place — `scope_key` replaced with `capability_key` + `dimension_key` (nullable). Unique constraint `NULLS NOT DISTINCT (actor_id, capability_key, dimension_key)`. CHECK constraint enforces score_type/key nullity state machine. No V1005 (project convention: no incremental migrations).
- **API module:** `ScoreType` gains `CAPABILITY_DIMENSION`. `ActorTrustScore @MappedSuperclass` replaces `scopeKey` field with `capabilityKey` + `dimensionKey`.
- **Repository SPI:** `findByActorIdAndTypeAndKey` removed. Typed replacements: `findCapabilityScore`, `findDimensionScore`, `findCapabilityDimension`, `findCapabilityDimensions`. Upsert signature updated to two-key params.
- **`TrustScoreJob`:** Fourth pass (capability-dimension) between dimension and global. Groups raw `actorAttestations` by `(capabilityTag, trustDimension)`, calls `computeDimensionScore`.
- **`TrustGateService`:** `qualityScore`, `qualityScores`, `meetsQualityThreshold`.
- **Federation:** `CapabilityDimensionScoreExport` record, `ActorExport` gains fourth field, `TrustExportService` + `JpaTrustImportService` updated.
- **ADRs:** 0009 (decay-weighted average for continuous scores), 0010 (two-column key model, supersedes 0006).
- **Downstream check:** grepped all repos for `.scopeKey` — no references. Safe to publish snapshot.

## Key Decisions

- Two explicit columns beat composite string encoding (`"cap:dim"`) — see ADR 0010
- `score_type` column retained despite being deterministic from keys — enables indexed queries without multi-column IS NULL expressions
- CHECK constraint enforces state machine at DB level — no application-level enforcement needed
- Continuous scores (DIMENSION, CAPABILITY_DIMENSION) use decay-weighted average; binary scores (GLOBAL, CAPABILITY) use Bayesian Beta — see ADR 0009

## Open Issues

- #72 — cross-repo quality issues from sweep (still open)
- #50, #49, #48 — epics (still open)

## Immediate Next Steps

Check remaining open work: `gh issue list --repo casehubio/ledger`
No clear next issue identified — #72 (quality sweep) is the next candidate.

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-14-mdp01-quality-at-the-intersection.md` |
| Latest spec (project) | `docs/specs/2026-05-14-capability-dimension-trust-score-design.md` |
| Latest ADRs | `docs/adr/0009-continuous-scores-decay-weighted-average.md`, `docs/adr/0010-two-column-key-model-replaces-scope-key.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
