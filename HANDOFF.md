# Session Handover — casehub-ledger

**Date:** 2026-11-20
**Branch:** main (work landed)

## What was done

- **#243** — Stale .m2 cache after LedgerEntryEnricher refactor. Not a code bug. Updated CLAUDE.md build guidance.
- **#244** — ADR 0005 updated for enricher relocation to api module.
- **#245** — Contributor guide enricher pipeline section updated.
- **#238** — EigenTrust relevance weighting: three-factor edge weights (decay × confidence × relevance, not credibility). Two commits landed. Blog written.
- **CLAUDE.md** — Stripped stale workspace metadata (slot 194 paths, dual-repo workflow). Matched platform pattern.

## Issues filed

| # | Title | Scale | Blocked by |
|---|-------|-------|------------|
| 246 | fix: EigenTrust relevance snapshot deserialization gap | S | — |
| 247 | refactor: extract AttestationRelevanceContext construction | XS | #246 |
| 248 | test: integration test for EigenTrust weighted pipeline | S | — |
| 249 | feat: TrustScoreDetail side-channel SPI (Stage 2.5) | M | — |
| 250 | feat: EigenTrust score calibration layer (Stage 2) | M | — |
| 251 | feat: hybrid trust fusion (Stage 3) | M | #250 |

## Queue

#239 (independence-aware relevance factor — anti-collusion cap, M/High) is active. Then #246, #247, #248.

## Context for next session

- Design spec at `specs/issue-238-eigentrust-relevance/2026-11-20-eigentrust-relevance-weighting-design.md` covers the evolution roadmap (Stages 1-3 + TrustScoreDetail)
- #239 depends on eidos#248 (org-proximity factor) which has landed per the user
- Blog at `blog/2026-11-20-mdp01-the-network-that-couldnt-tell-time.md` — promoted and published
- Runtime test bootstrap issue (#243) is resolved — was stale .m2 cache, not a code bug
