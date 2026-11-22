---
layout: post
title: "The Snapshot That Never Unpacked"
date: 2026-11-22
entry_type: note
subtype: diary
projects: [casehubio/ledger]
tags: [trust, eigentrust, relevance, deserialization, testing, refactoring]
---

# The Snapshot That Never Unpacked

The enricher pipeline populates `relevanceSnapshot` at save time — a JSON string on every attestation, carrying whatever data each enricher contributes. Org-distance, capability overlap, collaboration recency. The snapshot is the extensibility mechanism: add an enricher bean, it writes a key; add a relevance factor bean, it reads the key. No schema changes in ledger.

The Bayesian Beta path deserializes this snapshot when it builds `AttestationRelevanceContext`. Jackson reads the JSON string into a `Map<String, Object>`, the context record carries the map, relevance factors read their keys, and the weight chain works. This was wired correctly from the start.

The EigenTrust path passed `Map.of()`.

Every relevance factor that relied on snapshot data — org-distance, anything that reads enricher-populated keys — always returned NEUTRAL for EigenTrust. The factors checked the snapshot, found nothing, and fell back to their default. The edge weight computation ran, the decay and confidence multipliers applied, but relevance was flat across every attestation.

The bug survived because the pipeline is correct at every other level. `RelevanceComposer.assess()` received well-formed `AttestationRelevanceContext` records. The factors ran without error. The scores produced were valid numbers. Nothing blew up. The only symptom: relevance factors had no effect on EigenTrust scores. Unless you compared Beta and EigenTrust behaviour side by side with snapshot-dependent factors active, you wouldn't notice.

## One Line, Two Callers

The fix in each caller was replacing `Map.of()` with a `parseSnapshot(att)` call that deserializes `att.relevanceSnapshot` using Jackson — the same pattern already working in `TrustScoreCalculator`. Both `TrustScoreComputationService` (framework-neutral, in ledger-core) and `TrustScoreJob` (Quarkus-scheduled, in runtime) had the identical gap.

The fact that there were two callers with the same bug pointed to the second problem: the `AttestationRelevanceContext` construction was duplicated. Same field mapping, same parameter order, same snapshot logic — copy-pasted when the EigenTrust pipeline was wired up. We extracted `EigenTrustEdgeWeights.buildRelevanceContexts()` as a shared static method that takes the attestation list and the entry-actor index, handles the deserialization internally, and returns the context list. Both callers reduced to a single-line delegation.

The Beta path keeps its own inline construction because it passes different parameters — `tenancyId` from the entry (tenant-scoped), vs `null` for EigenTrust (cross-tenant). The divergence is structural, not accidental. Consolidating further would force a parameter that's sometimes null, adding complexity for no gain.

## Testing the Weight Chain

The integration tests needed a specific setup to verify that edge weights actually propagate through EigenTrust. The naive approach — give bob one attestation and charlie one attestation with different weights — produces identical scores. EigenTrust normalizes per row in the trust matrix: if an actor has only one positive attestation, the row normalizes to 1.0 regardless of the absolute weight.

The fix: give a single attestor multiple attestations with different weights. Charlie attests both alice's decision and bob's decision. If charlie's attestation on alice is recent and high-confidence, while his attestation on bob is old, the row normalizes to something like `[0.9, 0.1]` instead of `[0.5, 0.5]`. Now the weight factors are visible in the converged trust scores.

Four tests cover the pipeline end-to-end through `TrustScoreComputationService`:

- **Basic pass:** attestations in, global scores out, all positive
- **Decay:** recent attestation gives the attested actor a higher score than an old one
- **Confidence:** high-confidence attestation gives more trust than low-confidence
- **Relevance factor:** a custom factor that penalises attestations on one actor shifts scores

Each test captures `updateGlobalTrustScore()` calls via Mockito argument captors and asserts on the relative ordering. The assertions are directional (greater-than), not magnitude-specific — the exact scores depend on alpha, decay half-life, and convergence behaviour that's not the point of the test.

## What This Opens

The snapshot deserialization gap meant that any relevance factor contributed through the enricher-snapshot mechanism was silently inactive for EigenTrust. With the fix, the `IndependenceAwareRelevanceFactor` (landed in eidos last session) will actually fire for EigenTrust's transitive trust computation — not just for Beta's per-actor scoring. That matters: independence-based attestation weighting is specifically designed to detect collusion patterns in the network graph, and the network graph is exactly what EigenTrust operates on.

The extraction also means that adding a new field to the EigenTrust context construction is a single-point change. When the snapshot schema evolves — and it will, as more enrichers contribute keys — the mapping lives in one place.
