---
layout: post
title: "The write that commits before returning"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, cdi, transaction, trust-scoring, game-ai, outcome-recording]
---

QuarkMind needs trust-weighted routing for four game plugins — strategy, economics,
tactics, scouting — at roughly game-loop frequency. What it doesn't need is Merkle
chain verification, DID/VC identity binding, or Ed25519 bilateral signing. Game AI
decisions aren't compliance artifacts. Signing overhead per entry at 22Hz × 4 plugins
is unacceptable.

The initial design was a combined write API plus an incremental trust update pipeline:
every attestation would trigger a per-actor recomputation without waiting for the
nightly batch job. Three rounds of spec review later I was reasonably confident in
the design. Then Claude and I read `casehub-engine` more carefully.

`TrustWeightedAgentStrategy` doesn't read `TrustGateService`. It reads `TrustScoreCache`,
a `ConcurrentHashMap` keyed by `"actorId:capabilityKey"`. The cache has two hydration
paths: startup (reads `ActorTrustScoreRepository` directly) and `TrustScoreFullPayload`
events fired by `TrustScoreRoutingPublisher` after each `TrustScoreJob` run. There's
also a `TrustScoreDeltaPayload` path — but `onDelta()` is a no-op for CAPABILITY scores.
Delta payloads carry only GLOBAL scores.

That collapsed the design. The incremental pipeline I'd been refining wasn't wrong
exactly — it just didn't reach routing. There's no path from "update a score
incrementally" to "routing sees new scores" without `TrustScoreJob` running. The right
answer: set `casehub.ledger.trust-score.schedule=30s`. For in-memory, a batch run
across four actors takes microseconds. The cache refreshes. Done.

I descoped the entire incremental pipeline to issue #115, which needs `casehub-engine`
changes before it makes sense anyway.

## What we built

`OutcomeRecord` is a Java record. The compact constructor enforces: confidence in
(0.0, 1.0], attestor pair (both null or both set together), actorType defaulting to
AGENT when null, capabilityTag never null. The primary factory requires `capabilityTag`
explicitly:

```java
OutcomeRecord.of("quarkmind:strategy@v1", gameSessionId, "strategy", SOUND, 0.7)
```

GLOBAL-scoped attestations don't reach `TrustScoreCache`. I wanted the compiler to
enforce routing correctness, not leave it as a runtime footnote. `ofGlobal()` exists
for intentional global writes. `of()` requires you to name the capability.

`DefaultOutcomeRecorder` is `@DefaultBean @ApplicationScoped` and deliberately NOT
`@Transactional`. It delegates writes to `OutcomeRecordSaveService`, which is. The
separation matters for a race condition that doesn't exist yet but will when #115 ships:
if the outer `record()` method were itself transactional, async CDI observers would fire
before the transaction commits — reading uncommitted data. The separate `@Transactional`
delegate commits when `save()` returns. The outer method resumes with committed writes.
Post-commit hooks live in `record()` after `saveService.save()` — that's the design
seam for #115.

## The thing the spec got wrong

"No migrations." That was wrong. `runtime/model/LedgerEntry` is abstract — JOINED
inheritance requires it — and `OutcomeRecordSaveService.buildEntry()` has to instantiate
something concrete. Claude flagged this during planning: `new LedgerEntry()` won't
compile. We added `PlainLedgerEntry` — `@Entity @DiscriminatorValue("PLAIN")`, empty
body — and `V1009__plain_ledger_entry.sql`, a single-FK join table with no extra columns.
One migration, five lines of SQL.

`FlywayLocationContractTest` had a hardcoded migration count of 9. The full suite saw
10. One line fix. Everything else: 604 tests, 0 failures on the first full run.

## The config key that isn't obvious

`casehub.ledger.trust-score.routing-enabled=true` is the critical one.
`TrustScoreRoutingPublisher.publish()` checks that flag at its first line and returns
early if it's false. The cache never refreshes. The entire data flow doesn't execute
without it.

The capability tag mismatch is the other silent failure: write with GLOBAL, read with
`"strategy"`, every lookup returns empty, every plugin stays in BOOTSTRAP permanently.
`TrustScoreCache` has a comment about this. It won't stop anyone who isn't reading
the source.
