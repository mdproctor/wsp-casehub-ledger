---
layout: post
title: "The Issue That Was Already Done"
date: 2026-08-23
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [query-api, trust-scoring, design-verification]
---

## The Issue That Was Already Done

Three issues came in from qhorus — streaming queries for formal verification,
aggregate queries for compliance reporting, and materialised trust score
snapshots for per-message routing. The third one asked for a snapshot entity,
periodic computation, on-write triggers, staleness awareness, and a low-latency
read path.

The ledger already had all of it. `ActorTrustScore` with `lastComputedAt`.
`TrustScoreSnapshot` captured on every score upsert. `MaterializedTrustScoreSource`
for O(1) reads from the database. `CachedTrustScoreSource` for in-memory reads
with event-driven invalidation. `IncrementalTrustUpdateObserver` for per-attestation
recomputation. The issue was filed against a description of the codebase that
predated four months of trust infrastructure work.

The right response was a comment documenting what already exists — a table mapping
each requested capability to the class that provides it — and closing the issue.
Qhorus needs to inject `TrustScoreSource` and call `capabilityScore()`. That's it.

## The Outcome That Wasn't a Field

The remaining two issues needed aggregate queries: "what's the fulfillment rate
for this actor?" and "how does quality vary across channels?" The natural
instinct was to add an `outcome` field to `LedgerEntry` — make the aggregation
trivial with a single `GROUP BY`.

But the ledger separates decisions from assessments. A `LedgerEntry` records what
happened. A `LedgerAttestation` records how it was judged — with a verdict, a
confidence score, and a capability scope. The same entry can receive different
verdicts from different attestors. An `outcome` field on the entry would collapse
that into a single value, breaking multi-attestor scoring.

The aggregate queries JOIN through the attestation table instead. Fulfillment
rate is `endorsed / (endorsed + challenged)` from `countByActorAndVerdict`.
Per-channel quality is the same query scoped by subject. Confidence distribution
comes from `summariseAttestationsByActor` — mean, min, max across all attestations
in a time window.

We wrote the tests to prove the derivation works — not just that the queries
return correct data, but that a qhorus consumer can compute fulfillment rate,
routing success, and per-channel quality from the entry+attestation model without
any schema changes. The model was already right. It just needed the query surface
to expose it.
