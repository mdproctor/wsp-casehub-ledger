# Independence-Aware Relevance Factor — Design Spec

**Issue:** casehubio/ledger#239
**Date:** 2026-11-22
**Scale:** S | **Complexity:** Med (reduced from M/High — implementation already landed)

## Problem

The org-proximity factor (#231) is accuracy-biased: same-unit attestors get higher weight
because they share operational context. This widens the anti-gaming gap in ARC42 §Active
Risks: correlated positive attestations from a unit produce higher trust scores than the
same performance attested cross-unit. An independence cap limits the maximum influence ratio.

## Current State

The core implementation for the independence-aware relevance factor has already landed
across two repos:

| Component | Repo | Commit/Branch | Status |
|-----------|------|---------------|--------|
| `IndependenceAwareRelevanceFactor` | eidos (org-runtime) | eidos#257 | Implemented + tested |
| `IndependenceAwareRelevanceFactorTest` | eidos (org-runtime) | eidos#257 | 15 test cases |
| `RelevanceComposer` assessBatch refactor | ledger (ledger-core) | `a13ec9e8` | Implemented + tested |
| `RelevanceComposerTest` batch tests | ledger (ledger-core) | `082ea0e7` | 2 test cases |

### How the Factor Works

`IndependenceAwareRelevanceFactor` overrides `assessBatch()` to reason about group
composition. For each batch:

1. Classify attestations by `org-distance`: same-unit (== 1), cross-unit (> 1), unknown (≤ 0 or missing)
2. Compute share: `sameCount / (sameCount + crossCount)` — unknowns excluded from denominator
3. If share ≤ cap (default 0.6): all weights NEUTRAL
4. If share > cap: same-unit weight = `cap × crossCount / ((1 − cap) × sameCount)`, cross-unit stays 1.0
5. Guard: if `crossCount == 0` or `sameCount == 0`: all NEUTRAL (degenerate distributions)

Config: `casehub.eidos.trust.independence.max-same-unit-share=0.6`

## Remaining Work (this issue)

### 1. Integration Tests

**Gap:** No test in ledger exercises `TrustScoreCalculator.computeAll()` with a
`RelevanceComposer` wired with actual factors producing non-uniform weights. All existing
calculator/computer tests use `RelevanceComposer(List.of(), 0.5)` (empty factor list).

**Needed:**

- **TrustScoreCalculator integration test** — Wire a `RelevanceComposer` with a mock
  factor that produces non-uniform weights via `assessBatch()`. Verify that:
  - Capability-pass scores reflect the weighted attestation influence
  - Dimension-pass scores apply relevance weights correctly
  - Global-pass scores are affected by relevance
  - The weight chain (decay × confidence × credibility × relevance) composes correctly

- **Snapshot round-trip test** — Create attestations with `relevanceSnapshot` JSON containing
  `{"org-distance": N}`, build `AttestationRelevanceContext` from them (as `TrustScoreCalculator`
  does in production), and verify the composer produces correct weights. This tests the
  serialisation path that connects write-time enrichment to read-time factor evaluation.

### 2. Documentation Updates

**ARC42STORIES.MD:**
- Update Active Risks table (line ~1313): change "independence-biased factors can be contributed
  via same SPI" to reflect that the factor is shipped and active when eidos is on the classpath
- Add the independence factor to the relevance weight chain description

**Contributor guide (docs/guides/contributor-guide.md):**
- Update relevance pipeline section to document `assessBatch()` composition
- Document the independence factor as a shipped example of a group-aware factor

**ADR 0018:**
- Update consequences to reflect the `assessBatch()` switch in `RelevanceComposer`
- Note the independence factor as the first group-aware factor implementation

**CLAUDE.md:**
- Update `RelevanceComposer` description to reflect batch-oriented composition

## Design Decisions

Decisions for the overall independence factor design were captured during brainstorming
and validated by a 3-round standard decision review. See `decisions.md` in this directory.

Key decisions:
- **D1:** Unit grouping by org-distance == 1 (hardcoded, not configurable)
- **D2:** Factor lives in eidos org-runtime (ADR 0018: implementations in consumers)
- **D3:** assessBatch() override for group-level reasoning (RelevanceComposer calls assessBatch)
- **D4:** Proportional down-weight (formula: `cap × crossCount / ((1 − cap) × sameCount)`)
- **D5:** Independent composition (no ordering dependency on org-proximity factor)
- **D6:** Share computed by attestation count (not evidence mass)
- **D7:** Unknown attestations excluded from share denominator

## References

- `eidos/org-runtime/.../IndependenceAwareRelevanceFactor.java` — factor implementation
- `eidos/org-runtime/.../IndependenceAwareRelevanceFactorTest.java` — 15 test cases
- `ledger-core/.../RelevanceComposer.java` — batch-oriented composition (commit `a13ec9e8`)
- `ledger-core/.../RelevanceComposerTest.java` — assessBatch composition tests
- `ledger-core/.../TrustScoreCalculator.java:73-95` — context building from attestation snapshots
- `ledger-core/.../TrustScoreComputer.java:98-207` — four-factor weight chain with normalisation
- `api/spi/AttestationRelevanceFactor.java:36-42` — assessBatch default method
- ARC42STORIES.MD line 1313 — in-unit amplification risk
- casehubio/ledger#231 — attestation relevance SPI (parent)
- casehubio/eidos#257 — companion eidos issue (factor implementation)
- ADR 0018 — AttestationRelevanceFactor SPI design
- `specs/issue-239-independence-relevance-factor/decisions.md` — 7 validated decisions
