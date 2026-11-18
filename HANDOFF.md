# Session Handover — casehub-ledger

**Date:** 2026-11-18
**Branch:** main (landed from issue-240-adr-attestation-relevance-factor)
**Issues closed:** #240, #241, #242

## What Happened

Completed all documentation and cleanup follow-ups from the #231 attestation relevance implementation:

1. **ADR 0018** — formal decision record for the `AttestationRelevanceFactor` SPI. Covers relevance-vs-credibility separation, factor pipeline composition (product with floor, mass-preserving normalisation), write-time snapshot for auditability, accuracy-vs-independence tension, and JSON snapshot extensibility.

2. **ARC42 updates** — L4 trust section: four-factor weight chain, relevance pipeline, write-time snapshot. Active Risks: in-unit amplification trade-off. Glossary: 5 new entries. ADR count bumped to 18.

3. **LedgerEntryEnricher moved from ledger-core to api** — both enricher SPIs now live in `api/spi/` for consumer visibility. Added `String name()` method to the interface and all 11 implementations. Consumer-guide module column corrected. CLAUDE.md project structure updated.

Code review found 3 warnings (all fixed): consumer-guide stale module column, `TraceIdEnricher.name()` hardcoding instead of delegating, formatting inconsistency.

Blog written: "The Interface That Lived Next Door" — the enricher module placement story.

## Decisions

- `LedgerEntryEnricher` belongs in `api` not `ledger-core` — consumers implement it, so they must be able to see it without depending on internal modules
- `name()` added as abstract (not default) for consistency with `LedgerAttestationEnricher` — no production deployments, so source-breaking is acceptable

## Issues Filed

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #243 | fix: runtime test bootstrap — AuditedInterceptor CDI binding | S | Med |
| #244 | docs: update ADR 0005 — enricher moved to api | XS | Low |
| #245 | docs: contributor-guide enricher pipeline section | XS | Low |

## Queue (remaining)

| # | Issue | Notes |
|---|-------|-------|
| #243 | fix: runtime test bootstrap — AuditedInterceptor CDI binding | Pre-existing; blocks all runtime module tests |
| #244 | docs: update ADR 0005 — enricher moved to api | Cross-reference the relocation |
| #245 | docs: contributor-guide enricher pipeline section | Verify prose references to old package |

## Known Issue

Runtime module tests fail on main with `AuditedInterceptor has no bindings` CDI error (#243). Pre-existing — not caused by this session's work. API and ledger-core tests pass.

## References

- ADR 0018: `docs/adr/0018-attestation-relevance-factor-spi.md`
- Blog: `blog/2026-11-18-mdp02-the-interface-that-lived-next-door.md`
- Previous session blog: `blog/2026-11-18-mdp01-the-weight-that-knows-where-you-sit.md`
- Spec (from #231): `docs/specs/issue-231-org-scoped-attestation-weighting/2026-11-18-attestation-relevance-design.md`
