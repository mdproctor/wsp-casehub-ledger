# EigenTrust Relevance Weighting — Design Spec

**Issue:** casehubio/ledger#238
**Date:** 2026-11-20
**Scale:** S | **Complexity:** Med

## Problem

EigenTrust builds its local-trust matrix with flat +/-1.0 edge weights per attestation. The Beta path already applies a four-factor weight chain (decay x confidence x credibility x relevance) with mass-preserving normalisation. EigenTrust gets none of these — an attestation from a different domain, years old, with low confidence counts the same as a recent, high-confidence, domain-relevant one.

## Stage 1: Three-Factor Edge Weighting (this issue)

### What changes

Apply three factors to EigenTrust edge weights: **decay x confidence x relevance**. Credibility is excluded because power iteration already handles attestor trust transitively — applying it as an edge weight would double-count.

### Where it changes

`EigenTrustComputer`'s matrix-building loop (currently ~lines 91-106). Instead of:

```
s[i][j] += verdict == SOUND || verdict == ENDORSED ? +1.0 : -1.0
```

Becomes:

```
double weight = decayWeight * confidenceWeight * relevanceWeight;
s[i][j] += verdict == SOUND || verdict == ENDORSED ? +weight : -weight;
```

### Method signature change

`EigenTrustComputer.compute()` currently receives:
- `List<LedgerAttestation> attestations`
- `Map<UUID, String> entryActorIndex`

Add `AttestationWeights weights` parameter. The caller (`TrustScoreCalculator` or batch job) already computes this for the Beta path — pass it through.

`EigenTrustComputer` reads `weights.relevanceByAttestationId()` and needs access to decay and confidence per attestation. Two options:

**(Recommended) Option A:** Compute a single `effectiveWeight` per attestation in the caller and pass as `Map<UUID, Double>`. EigenTrustComputer stays algorithm-agnostic — it just applies weights.

Option B: Pass individual factor maps and let EigenTrustComputer multiply them. More flexible but couples it to the factor model.

### Convergence

EigenTrust normalises rows to sum to 1: `c[i][j] = max(s[i][j], 0) / sum_k max(s[i][k], 0)`. This produces a row-stochastic matrix regardless of raw edge values. Weighted edges change the relative proportions (high-relevance attestations get more influence) but convergence properties are preserved.

### No consumer changes

All consumers read through `TrustScoreSource` which returns `OptionalDouble`. EigenTrust quality improves transparently.

## Evolution Roadmap

### Stage 2: Calibration Layer

Beta scores have natural probabilistic semantics (0.7 = 70% positive evidence). EigenTrust scores represent network position — different distribution, different semantics. Threshold gating (engine's BOOTSTRAP/QUALIFIED/BORDERLINE/EXCLUDED tiers) can't be calibrated consistently across algorithms when consumers see opaque scalars.

**Solution:** For actors with enough direct attestations, compute both Beta and EigenTrust. These actors form a calibration set. Fit a monotone mapping from EigenTrust's distribution to Beta's scale. Apply to all EigenTrust scores.

**Fallback:** Small networks without a sufficient calibration set use identity mapping.

### Stage 2.5: TrustScoreDetail Side-Channel SPI

Add an optional companion method to `TrustScoreSource`:

```java
public enum TrustAlgorithm { BETA, EIGENTRUST, HYBRID }

public record TrustScoreDetail(
    double score,
    TrustAlgorithm algorithm,
    int directAttestationCount,
    double dataSufficiency,      // 0.0 = no direct data, 1.0 = fully sufficient
    Double mixingWeight          // null if not HYBRID
) {}

// Default method on TrustScoreSource — existing impls don't break
default Optional<TrustScoreDetail> globalScoreDetail(String actorId) {
    return globalScore(actorId).map(s ->
        new TrustScoreDetail(s, TrustAlgorithm.BETA, -1, 1.0, null));
}
```

Consumers opt in by calling `globalScoreDetail()`. Primary API unchanged.

**Consumer value:**
- Engine: adjust threshold gating when `dataSufficiency` is low
- Neocortex: weight trust trajectory signal by data sufficiency
- Debugging: see exactly why a score is what it is

### Stage 3: Hybrid Fusion

Blend calibrated EigenTrust with Beta using a data-sufficiency mixing function:

```
score = (1 - lambda(n)) * beta + lambda(n) * calibratedEigenTrust
```

Where `lambda` is high when direct attestation count `n` is low (lean on network) and low when `n` is high (lean on direct evidence). Smooth transition — no algorithm-switching cliff.

**Design risk:** Fusion function shape requires empirical tuning. Start with a simple sigmoid and adjust based on observed score distributions.

## Testing Strategy

- Unit test: verify weighted edge values in EigenTrust matrix (mock attestations with known relevance/decay/confidence)
- Unit test: verify convergence produces same result for uniform weights as current flat +/-1.0
- Unit test: verify high-relevance attestations produce higher proportional influence after normalisation
- Integration test: verify TrustScoreCalculator passes weights through to EigenTrust path

## References

- `ledger-core/.../trust/EigenTrustComputer.java` — matrix building loop, normalisation, power iteration
- `ledger-core/.../trust/TrustScoreComputer.java` — Beta four-factor weight chain, mass-preserving normalisation
- `runtime/.../service/TrustScoreCalculator.java` — orchestration, batch relevance resolution
- `runtime/.../service/RelevanceComposer.java` — factor aggregation with floor
- `api/spi/AttestationRelevanceFactor.java` — relevance SPI
- `api/spi/TrustScoreSource.java` — consumer-facing read API
- casehubio/ledger#231 — attestation relevance SPI (parent)
- Engine `TrustWeightedAgentStrategy` — threshold gating consumer
- Eidos `SimpleAgentSelector` — threshold + ranking consumer
- Neocortex `TrustWeightedCbrRecordStore` — similarity weighting consumer
