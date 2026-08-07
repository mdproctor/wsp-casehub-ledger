---
layout: post
title: "Trust Score Snapshots: Where to Hook the History"
date: 2026-08-07
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [trust-scoring, entity-design, quarkus]
---

The issue was straightforward: devtown needs sparklines showing how an agent's trust score evolves over time. The current `ActorTrustScore` entity is a live value — upserted in place on every recomputation. No history survives.

The interesting decision was where to capture the snapshot. The original issue specified `IncrementalTrustUpdateObserver` — the CDI observer that fires per-attestation. That would mean only the incremental path produces history. The nightly batch job (`TrustScoreJob`) would silently overwrite scores without recording what changed.

Both paths converge in `PerActorTrustComputer`. It's package-private, injected into both the observer and the batch job, and it's the exact point where old score meets new score. Reading the previous value from the repository before each upsert, then persisting a `TrustScoreSnapshot` with both values, gives complete coverage with zero duplication.

The entity is minimal: `(id, actorId, capabilityTag, score, previousScore, occurredAt)`. `capabilityTag` is nullable — null for global score snapshots. Two `@NamedQuery` annotations handle the split: one for global history, one filtered by capability. The index on `(actor_id, capability_tag, occurred_at DESC)` keeps the most-recent-first retrieval fast.

One thing that tripped us up: the runtime test suite activates JPA `@Alternative` repositories via `quarkus.arc.selected-alternatives` in `application.properties`, not the `@Priority(1)` in-memory implementations from `persistence-memory`. The new `JpaTrustScoreSnapshotRepository` needed adding to every test profile that activates JPA trust scoring — six separate blocks in the properties file. The `@DefaultBean` NoOp silently swallowed everything until we noticed.

This opens up the consumer side — devtown can now query score trends per actor and capability, which is what the trust visibility UI needs for its sparkline rendering.
