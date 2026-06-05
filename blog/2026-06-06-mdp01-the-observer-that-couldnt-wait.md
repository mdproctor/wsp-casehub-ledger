---
layout: post
title: "The Observer That Couldn't Wait"
date: 2026-06-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [trust-scoring, cdi, transactions, refactoring, incremental]
excerpt: "Trust scores that update when attestations arrive, not when the clock ticks. The race condition in the obvious approach, the extraction that made it possible, and the CDI test gotcha that nearly derailed the integration test."
---

# CaseHub Ledger — The Observer That Couldn't Wait

## What I was trying to achieve: trust scores that update when it matters

The ledger's trust scoring pipeline runs on a schedule — default every 24 hours. `TrustScoreJob` loads every EVENT entry in the database, groups by actor, computes Bayesian Beta scores across four passes (capability, dimension, capability-dimension, global), and writes the results. It works. But between runs, new attestations are invisible. A routing decision made 30 seconds after an attestation reads yesterday's trust data.

The fix sounds obvious: recompute the affected actor's scores when an attestation arrives, not when the clock ticks. The issue had been open for a while because the obvious implementation has a race condition in it.

## The race nobody talks about

CDI gives you `Event.fire()` and `@ObservesAsync`. Fire the event from `saveAttestation()`, observe it on another thread, recompute. Clean and decoupled. Except the async observer starts before the attestation's transaction commits. The observer queries the database and doesn't see the new attestation — the one that triggered it. The recomputation runs against stale data, and the batch job silently corrects it hours later.

The alternative — `@Observes(during = TransactionPhase.AFTER_SUCCESS)` — guarantees the attestation is committed before the observer fires. Combined with `@Transactional(REQUIRES_NEW)`, the observer opens a fresh transaction that sees committed data. The overhead is about 5-10ms per attestation: one actor's events, their attestations, the in-memory Bayesian computation, and a handful of score upserts. Deployments where that matters keep incremental off and tune the batch schedule instead.

## Extracting what was already there

The interesting part wasn't the observer — it was what the observer needed to call. `TrustScoreJob.runComputation()` had all four scoring passes inline in a 130-line loop body. The batch job and the incremental path need the same computation, so the first step was extracting it.

`PerActorTrustComputer` is the result — a package-private CDI bean with two constructors (one for CDI, one for unit tests without Quarkus). It takes an actor's decisions and attestations, runs the four passes, upserts the scores, and returns them. `TrustScoreJob` now iterates over actors and calls `computeForActor()` for each. The incremental observer does the same for one actor.

The extraction was the riskiest part. Behaviour-preserving refactoring of computation code with this much state (grouped attestations, synthetic aggregations, capability score maps feeding the global pass) is exactly where subtle bugs hide. The existing integration tests — covering capability, dimension, capability-dimension, global, aggregation, and bootstrap paths — served as the regression gate.

## A new event, not a reused one

The incremental path fires `TrustScoreActorUpdatedEvent` — distinct from the batch job's `TrustScoreFullPayload`. Reusing the full payload would have worked (the cache's `index()` method is additive), but the semantic distinction matters. "One actor's score changed" and "full batch recomputation ran" are different signals. Engine's `TrustScoreCache` can adopt the new event type when it's ready — one observer method, calling the same `index()` it already has. Until then, the batch job keeps the cache current.

## What the tests taught me about CDI

The integration test for the observer hit a wall I should have predicted. The test method was `@Transactional`, as is standard for `@QuarkusTest`. But `AFTER_SUCCESS` observers are queued until the transaction commits — and the test's transaction doesn't commit until the method returns. Every assertion ran before the observer fired.

The fix is `QuarkusTransaction.requiringNew()` for programmatic transaction control. Each block commits independently, triggering the AFTER_SUCCESS observer before the next assertion. It's the kind of thing you discover once and never forget — or find in the garden six months from now.
