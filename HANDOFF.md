# Session Handover — casehub-ledger

**Date:** 2026-11-18
**Branch:** main (landed from issue-231-org-scoped-attestation-weighting)
**Issue:** casehubio/ledger#231 — closed

## What Happened

Designed and implemented the attestation relevance SPI for org-scoped trust weighting. The design went through two review cycles (decision review + spec review) which caught and fixed: dependency inversion (eidos ships the implementation, not ledger), write-time snapshot for auditability (not compute-time lookup), per-pass mass-preserving normalisation, and the factor pipeline model. Implementation: 8 tasks across 4 batches, all 868 runtime tests green.

## Decisions

- Pair-scoped relevance (per attestation) is separate from attestor-scoped credibility — different SPI
- Write-time JSON snapshot (`relevance_snapshot`) instead of compute-time org tree lookup — deterministic, auditable
- Mass-preserving normalisation inside TrustScoreComputer — per pass, not per actor
- Factor pipeline via `Instance<AttestationRelevanceFactor>` — extensible without ledger changes
- Self-attestation excluded from Beta path (attestorId == entry.actorId)
- Ledger has zero eidos dependency; eidos ships `OrgProximityRelevanceFactor`

## Cross-Module

- **eidos#248** — companion issue: `OrgProximityRelevanceFactor` + `OrgDistanceAttestationEnricher` implementation

## Queue

| # | Issue | Status |
|---|-------|--------|
| #240 | ADR for AttestationRelevanceFactor SPI | queued |
| #241 | ARC42 updates for attestation relevance | queued |
| #242 | Move LedgerEntryEnricher to api | queued |

## References

- Spec: `docs/specs/issue-231-org-scoped-attestation-weighting/2026-11-18-attestation-relevance-design.md`
- Blog: `docs/blog/2026-11-18-mdp01-the-weight-that-knows-where-you-sit.md`
- Plan: `plans/2026-11-18-attestation-relevance.md`
