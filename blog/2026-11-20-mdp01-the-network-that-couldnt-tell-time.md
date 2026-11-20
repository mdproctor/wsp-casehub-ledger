---
layout: post
title: "The Network That Couldn't Tell Time"
date: 2026-11-20
entry_type: article
subtype: explanation
projects: [casehubio/ledger]
tags: [trust, eigentrust, relevance, decay, convergence, design]
---

# The Network That Couldn't Tell Time

EigenTrust scores every actor in the mesh by propagating trust transitively. If Alice trusts Bob and Bob trusts Carol, Alice gets a derived signal about Carol — even without a direct attestation. The power iteration converges to an eigenvector where each actor's trust share reflects the entire network's opinion, not just their immediate attestors.

The problem: until now, every attestation contributed ±1.0 to the trust matrix. A SOUND verdict from two years ago counted exactly the same as one from yesterday. A verdict given with 60% confidence counted the same as 99%. An attestation from a completely different domain counted the same as one from a close peer.

The Bayesian Beta path already applied four weight factors — decay, confidence, credibility, and relevance. EigenTrust had none.

## Three Factors, Not Four

The fix applies three of those factors to EigenTrust edge weights: decay × confidence × relevance. The obvious question: why not credibility?

It comes down to what EigenTrust already does. Power iteration *is* a credibility mechanism. An attestor with low trust in the converged network — because other actors flagged their work — sees their influence naturally reduced through matrix propagation. If we also scaled their edge weights down by their credibility score, we'd penalise them twice: once explicitly through the weight, once implicitly through convergence.

The Beta path needs credibility because it's purely local. It sums up attestation evidence for a single actor without considering the network at all. If nobody tells it that attestor X is unreliable, it has no way to know. EigenTrust knows — that's the whole point of the transitive propagation.

So the factor chain for each path reflects what the algorithm *can't* derive on its own. Beta needs all four. EigenTrust needs three.

## What the Consumers Actually See

This distinction is invisible downstream. Engine routes agents via `TrustGateService` — threshold gating with maturity tiers (BOOTSTRAP, QUALIFIED, BORDERLINE, EXCLUDED). Eidos selects agents via `TrustScoreSource.capabilityScore()` with a fallback to `globalScore()`. Neocortex multiplies CBR case similarity by the source agent's trust authority. All three read opaque `OptionalDouble` scalars through `TrustScoreSource`.

None of them know or care whether the score came from Beta or EigenTrust. The implementation behind the SPI determines that — `MaterializedTrustScoreSource` reads persisted scores, `ComputedTrustScoreSource` computes on demand, and the nightly job writes EigenTrust globals alongside Beta.

This opacity is mostly a feature. But it creates a problem that matters once both algorithms coexist.

## The Calibration Problem

Beta's 0.7 means "70% of weighted attestation evidence is positive." It has a natural probabilistic interpretation. EigenTrust's 0.7 means "this actor holds a 0.7 share in the converged trust eigenvector" — a network position, not a probability. Different distributions, different semantics.

When engine sets a threshold at 0.6, it means different things depending on which algorithm produced the score. For ranking (sort by trust) and weighting (multiply similarity) the absolute value matters less — relative ordering is preserved. For threshold gating, it matters a lot.

Today this is fine because deployments run one global score strategy at a time. But it becomes a real constraint if the system evolves toward hybrid scoring — Beta for actors with enough direct evidence, EigenTrust for newcomers with sparse attestation history.

## The Evolution Path

We sketched a three-stage evolution that never breaks the opaque scalar interface:

**Calibration layer.** For actors with enough direct attestations, compute both Beta and EigenTrust. These actors form a calibration set. Fit a monotone mapping from EigenTrust's distribution to Beta's scale. Apply the mapping to all EigenTrust scores. Now both algorithms produce numbers on the same scale, and thresholds work consistently regardless of source.

**Side-channel metadata.** Add a `TrustScoreDetail` companion to the existing `globalScore()`:

```java
public enum TrustAlgorithm { BETA, EIGENTRUST, HYBRID }

public record TrustScoreDetail(
    double score,
    TrustAlgorithm algorithm,
    int directAttestationCount,
    double dataSufficiency,
    Double mixingWeight
) {}
```

Consumers that want the number keep calling `globalScore()`. Consumers that want to reason about derivation call `globalScoreDetail()`. Engine could adjust threshold gating when `dataSufficiency` is low. Neocortex could weight trust trajectory signals by data sufficiency. The default method wraps the existing score, so no implementation breaks.

**Hybrid fusion.** Blend calibrated EigenTrust with Beta using a data-sufficiency mixing function. Actors with many direct attestations get Beta-dominant scores. Newcomers get EigenTrust-dominant scores. The transition is smooth — no cliff where you switch algorithms.

## The Implementation

The edge weight computation is a pure static utility:

```java
public static Map<UUID, Double> compute(
        List<LedgerAttestation> attestations,
        DecayFunction decayFunction,
        Map<UUID, Double> relevanceByAttestationId,
        Instant now) {
    // ...
    double weight = decay * confidence * relevance;
    // ...
}
```

`EigenTrustComputer.compute()` gained a fourth parameter — `Map<UUID, Double> edgeWeights` — with the existing three-argument method delegating with an empty map. The matrix-building loop applies `edgeWeights.getOrDefault(att.id, 1.0)` instead of flat ±1.0. Row normalisation already ensures the matrix stays row-stochastic; weighted edges change the relative proportions without affecting convergence properties.

Both callers — the framework-neutral `TrustScoreComputationService` in ledger-core and the Quarkus-scheduled `TrustScoreJob` in runtime — now compute relevance via `RelevanceComposer.assess()`, combine it with decay and confidence, and pass the resulting map through. The callers mirror each other because the weight computation is factored into the utility; only the injection mechanism differs.

One edge case worth noting: if `occurredAt` is in the future (clock skew between services), the age calculation can go negative. `ExponentialDecayFunction` treats negative age as exponential *growth* — a weight above 1.0, which is wrong. The utility clamps age to zero. It's the kind of thing that wouldn't surface until a multi-region deployment with unsynchronised clocks.
