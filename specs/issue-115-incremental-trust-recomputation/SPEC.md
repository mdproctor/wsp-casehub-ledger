# Incremental Per-Actor Trust Recomputation

**Issue:** casehubio/ledger#115
**Date:** 2026-06-05
**Status:** Design approved

## Problem

Trust scores are computed by a batch job (`TrustScoreJob`) on a configurable schedule (default 24h). Between runs, new attestations are invisible to trust-based routing — scores are stale by up to the full batch interval. For deployments with moderate attestation rates (1–100/minute), this staleness means routing decisions reference hours-old trust data.

## Solution

Add an incremental per-actor recomputation path that triggers on every attestation persist. When an attestation is saved, the affected actor's trust scores (CAPABILITY, DIMENSION, CAPABILITY_DIMENSION, GLOBAL) are recomputed immediately using the same Bayesian Beta algorithm as the batch job. A dedicated CDI event notifies downstream consumers of the updated scores.

The batch job remains as a consistency backstop and continues to run EigenTrust (which requires the full actor graph and cannot be decomposed per-actor).

## Design Decisions

**Config switch, not strategy SPI.** The issue proposed an `AttestationTrustUpdateStrategy` SPI with IMMEDIATE/SCHEDULED/NONE modes. These map directly to existing config: `trust-score.incremental.enabled` (IMMEDIATE), `trust-score.enabled` (SCHEDULED already exists), `trust-score.enabled=false` (NONE). An SPI adds indirection with no behavioral benefit.

**Synchronous `AFTER_SUCCESS`, not async.** The observer uses `@Observes(during = TransactionPhase.AFTER_SUCCESS)` + `@Transactional(REQUIRES_NEW)`. This guarantees the attestation is committed before recomputation begins — no race condition. The ~5–10ms overhead (one actor's events + attestations, in-memory Bayesian computation, score UPSERTs) is acceptable for the use cases that need incremental freshness. Deployments with extreme attestation rates keep `incremental.enabled=false` and tune the batch schedule instead. CDI spec guarantees AFTER_SUCCESS observer exceptions are caught and logged, not propagated — the attestation is always safe.

**Dedicated event type, not reuse of `TrustScoreFullPayload`.** A new `TrustScoreActorUpdatedEvent` communicates "one actor's score changed" distinctly from "full batch recomputation ran." Consumers opt in to what they need. Engine's `TrustScoreCache` can adopt when ready — until then, the batch job's `TrustScoreFullPayload` keeps it current.

**Engine is downstream, not a blocker.** casehub-ledger is foundation; casehub-engine depends on it. Ledger ships the incremental path and the new event type. Engine adds an observer for `TrustScoreActorUpdatedEvent` at its own pace.

**Full actor history, not windowed.** The recomputation loads all EVENT entries for the actor, not just recent ones. Per-actor history is typically 10–1000 entries. Decay weighting already handles recency. A windowed variant (delta updates to running α/β) changes the computation model and is deferred.

**EigenTrust excluded.** EigenTrust power iteration requires the full actor×attestation graph. It cannot be decomposed per-actor. It remains batch-only.

## New Types

### `AttestationRecordedEvent` — `runtime/service/`

Internal CDI trigger fired from `saveAttestation()` in both JPA and in-memory repository implementations.

```java
public record AttestationRecordedEvent(
    String actorId,          // decision-maker whose score needs recomputation
    UUID ledgerEntryId,      // the entry that was attested
    UUID attestationId       // the attestation that was recorded
) {}
```

`actorId` is the decision-maker (the LedgerEntry's actorId), not the attestor. Resolved at fire time by looking up the entry from `ledgerEntryId` — avoids an extra query in the observer. If the entry lookup returns null (invalid `ledgerEntryId`), the event is not fired and a warning is logged.

### `TrustScoreActorUpdatedEvent` — `runtime/service/routing/`

Downstream notification fired after per-actor recomputation completes.

```java
public record TrustScoreActorUpdatedEvent(
    String actorId,
    List<ActorTrustScore> scores,    // all score types for this actor
    Instant computedAt
) {}
```

Relationship to existing routing events:
- `TrustScoreFullPayload` — batch only, all actors. Unchanged.
- `TrustScoreDeltaPayload` — batch only, changed actors' GLOBAL scores. Unchanged.
- `TrustScoreComputedAt` — batch only, lightweight signal. Unchanged.
- `TrustScoreActorUpdatedEvent` — incremental only, one actor. **New.**

The incremental path fires `TrustScoreActorUpdatedEvent` directly — it does not go through `TrustScoreRoutingPublisher`. The publisher remains batch-only.

### `PerActorTrustComputer` — `runtime/service/`

Package-private CDI bean extracted from `TrustScoreJob.runComputation()`. Encapsulates the four-pass per-actor computation:

1. **Capability pass** — group attestations by capabilityTag, Bayesian Beta via `TrustScoreComputer.compute()`, upsert CAPABILITY rows
2. **Dimension pass** — group by trustDimension, decay-weighted average via `computeDimensionScore()`, upsert DIMENSION rows
3. **Capability×Dimension pass** — group by (capabilityTag, trustDimension), upsert CAPABILITY_DIMENSION rows
4. **Global pass** — `GlobalScoreStrategy.selectAttestations()` + `derive()`, upsert GLOBAL row

`actorType` is resolved internally from the decisions list (first non-null `actorType`, defaulting to `ActorType.HUMAN`) — same logic as `TrustScoreJob` lines 139–143.

**Injections:**
- `DecayFunction` — to construct `TrustScoreComputer`
- `ActorTrustScoreRepository` — for score upserts
- `GlobalScoreStrategy` — for global pass (selectAttestations + derive)
- `AttestationAggregator` — for `buildEffectiveAttestations` (moves here from `TrustScoreJob`)
- `LedgerConfig` — for `aggregationStrategy()`

Signature:

```java
List<ActorTrustScore> computeForActor(
    String actorId,
    List<LedgerEntry> decisions,
    Map<UUID, List<LedgerAttestation>> attestationsByEntry,
    Instant now)
```

Returns the computed scores for the caller to use in routing events. Both `TrustScoreJob` and `IncrementalTrustUpdateObserver` call this — same code path, no duplication.

### `IncrementalTrustUpdateObserver` — `runtime/service/`

CDI bean that observes `AttestationRecordedEvent` and triggers per-actor recomputation.

```java
@ApplicationScoped
public class IncrementalTrustUpdateObserver {

    // Injections: LedgerConfig, LedgerEntryRepository, PerActorTrustComputer,
    //             Event<TrustScoreActorUpdatedEvent>

    @Transactional(TxType.REQUIRES_NEW)
    void onAttestationRecorded(
            @Observes(during = TransactionPhase.AFTER_SUCCESS)
            AttestationRecordedEvent event) {
        if (!config.trustScore().enabled()
                || !config.trustScore().incremental().enabled()) {
            return;
        }
        List<LedgerEntry> decisions = ledgerRepo.findEventsByActorId(event.actorId());
        // ... build attestation map, call PerActorTrustComputer, fire event
    }
}
```

## Modified Types

### `LedgerEntryRepository`

Add:

```java
List<LedgerEntry> findEventsByActorId(String actorId);
```

Returns all EVENT-type entries for one actor. Used by the incremental observer and available for general per-actor queries.

### `JpaLedgerEntryRepository`

- Implement `findEventsByActorId` via named query on `LedgerEntry`. The implementation calls `actorIdentityProvider.tokeniseForQuery(actorId)` before querying — consistent with every existing actor-scoped query method (e.g. `findByActorId` line 228). The incremental observer passes an already-tokenised actorId (read from a persisted entry), but tokenising in the repository ensures correctness for any future caller.
- Inject `Event<AttestationRecordedEvent>`
- Fire event from `saveAttestation()` after persist. Resolve `actorId` via `em.find(LedgerEntry.class, attestation.ledgerEntryId)`. If the entry is null, skip event firing and log a warning.

### `InMemoryLedgerEntryRepository`

- Implement `findEventsByActorId` by filtering `allEntries()`. Apply `actorIdentityProvider.tokeniseForQuery(actorId)` for API consistency with the JPA implementation.
- Inject `Event<AttestationRecordedEvent>`
- Fire event from `saveAttestation()` after adding to the collection. Resolve `actorId` from the entry via `findEntryById()`. If the entry is null, skip event firing and log a warning.

### `TrustScoreJob`

Refactor `runComputation()` to delegate per-actor logic to `PerActorTrustComputer`. The job iterates over actors and calls `computeForActor()` for each. Attestation aggregation (`buildEffectiveAttestations`) moves into `PerActorTrustComputer`. The EigenTrust pass and routing publisher call remain in the job.

### `LedgerConfig.TrustScoreConfig`

Add:

```java
IncrementalConfig incremental();

interface IncrementalConfig {
    @WithDefault("false")
    boolean enabled();
}
```

## Configuration

```properties
# Activate incremental per-actor recomputation (default: false)
casehub.ledger.trust-score.incremental.enabled=false
```

No build-time gating — the observer is always in the CDI graph and short-circuits when disabled. Runtime-toggleable via config refresh.

Gating checks both master switch and incremental switch:

```java
if (!config.trustScore().enabled() || !config.trustScore().incremental().enabled()) {
    return;
}
```

## Engine Adoption Path

`TrustScoreCache` in casehub-engine adds one observer:

```java
public void onActorUpdated(@Observes TrustScoreActorUpdatedEvent event) {
    event.scores().forEach(this::index);
}
```

Reuses the existing `index()` method. No architectural changes. Engine adopts when ready — until then, the batch job's `TrustScoreFullPayload` keeps the cache current. This is a separate issue in casehub-engine, not part of this implementation.

## In-Memory Parity

`InMemoryLedgerEntryRepository` fires `AttestationRecordedEvent` from `saveAttestation()`. Since there is no JTA transaction in in-memory mode, `AFTER_SUCCESS` fires immediately as `IN_PROGRESS` per CDI spec. This is correct — the in-memory save is synchronous and the attestation is available immediately.

`InMemoryActorTrustScoreRepository` already supports `upsert()`. No changes needed.

## What This Does Not Change

- **Batch job** — still runs on schedule. Remains the EigenTrust path and consistency backstop.
- **`TrustScoreRoutingPublisher`** — batch-only. Not used by incremental path.
- **Schema** — no database migrations. Trust scores use existing `actor_trust_score` table.
- **`TrustScoreComputer`** — pure computation, unchanged. Called by `PerActorTrustComputer`.
- **`AttestationAggregator`** — unchanged. Called by `PerActorTrustComputer`.
- **Bootstrap** — `TrustBootstrapService` remains batch-only. The incremental observer creates scores from attestation data for actors without existing scores.

## Deferred

- **Windowed recomputation** — load only events since last computation, delta-update running α/β. Changes the computation model. Optimize if per-actor history grows large.
- **Debouncing** — skip recomputation if `lastComputedAt` is within N seconds. Optimize if concurrent attestations for the same actor cause wasted work.
- **Async mode** — two-stage `AFTER_SUCCESS → fireAsync → @ObservesAsync` for zero calling-thread overhead. Add behind `casehub.ledger.trust-score.incremental.async=true` if profiling shows the 5–10ms synchronous overhead matters.
