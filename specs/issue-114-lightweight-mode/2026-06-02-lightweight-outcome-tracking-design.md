# Lightweight Outcome-Tracking Mode — Design Spec

**Issue:** casehubio/ledger#114
**Date:** 2026-06-02
**Branch:** issue-114-lightweight-mode

---

## 1. Context

QuarkMind (`mdproctor/quarkmind`) needs outcome-tracking to enable trust-weighted plugin
routing at game-loop granularity (~22Hz). Four plugins — strategy, economics, tactics,
scouting — make decisions each game tick. QuarkMind wants to record plugin outcomes and
feed them into trust scoring for adaptive routing.

The compliance stack (Merkle hash chain, DID/VC identity verification, Ed25519 bilateral
signing) is not needed here. Game AI decisions are not compliance artifacts; signing
overhead per entry is unacceptable at game-loop frequency.

The core requirements from QuarkMind are:
- Async, non-blocking writes that do not block the game loop
- No signing, no hash chain, no identity verification overhead
- In-memory backend for game sessions (no DB)
- Trust scores queryable per plugin: `trustScore("quarkmind:strategy@v1")` is sufficient
- Compatible with `TrustWeightedAgentStrategy` from `casehub-engine`

---

## 2. What Already Works — Configuration Only

Several requirements are already satisfied by existing feature flags. **No code is needed
for these.**

| Requirement | Mechanism | Config key |
|---|---|---|
| Skip Merkle hash chain | `LedgerConfig.hashChain().enabled()` | `casehub.ledger.hash-chain.enabled=false` |
| Skip agent signing | No key configured → `AgentSignatureEnricher` is a no-op | *(omit `casehub.ledger.agent-signing.keys.*`)* |
| Skip DID/VC validation | No DID configured → enrichers are no-ops | *(omit `casehub.ledger.agent-identity.dids.*`)* |
| In-memory persistence | `casehub-ledger-memory` on classpath | Add `casehub-ledger-memory` dependency |
| Skip trust score | Default | `casehub.ledger.trust-score.enabled=false` (default) |
| EigenTrust disabled | Default | `casehub.ledger.trust-score.eigentrust.enabled=false` (default) |

The in-memory module (`casehub-ledger-memory`) provides `@Alternative @Priority(1)`
implementations for all persistence SPIs. With it on the classpath, no datasource is
required — CDI resolution activates the in-memory repos automatically.

### Why EigenTrust must stay disabled for QuarkMind

QuarkMind has 4 plugins and a single attestor (the game engine). This is a star attestation
graph — EigenTrust power iteration on a star graph is equivalent to direct trust from a
single source and adds no value. Smaller graphs with a pre-trusted fallback also risk
3-cycle non-convergence (GE-20260421-09d636). Use Bayesian Beta direct scores only.

See ADR 0016 for the platform-level applicability criteria.

---

## 3. What Is Genuinely Missing — New Additions

Five gaps remain after configuration:

1. **No combined write API.** Recording a decision + outcome currently requires two
   separate calls: `save(LedgerEntry)` then `saveAttestation(LedgerAttestation)`. The game
   loop needs one.

2. **Trust score recomputation is batch-only and full-table-scan.** `TrustScoreJob` loads
   all events for all actors on every run. With 22Hz writes accumulating, this degrades.
   Per-actor incremental recomputation is needed.

3. **No event fired when an attestation is saved.** Nothing can react to an attestation
   arriving to trigger incremental recomputation.

4. **`LedgerEntryRepository` lacks an unbounded per-actor event query.** The existing
   `findByActorId(actorId, from, to)` requires time bounds and returns all entry types.
   Per-actor trust recomputation needs all EVENT entries for one actor, unbounded.

5. **No pluggable trigger for when trust scores update.** There is no SPI for whether
   trust recomputation happens immediately on attestation or deferred to the batch job.

---

## 4. New Components

### 4.1 `LedgerAttestationRecordedEvent`

A CDI event record fired by `saveAttestation()` in both the JPA and in-memory
implementations of `LedgerEntryRepository`, immediately after the attestation is stored.

```java
// runtime/service/LedgerAttestationRecordedEvent.java
public record LedgerAttestationRecordedEvent(
    String actorId,
    String capabilityTag,
    UUID ledgerEntryId,
    UUID attestationId
) {}
```

**Placement:** `runtime/service/` — consistent with `AgentKeyRotatedEvent`, `AgentSignatureSuspectEvent`.

**Where it fires:** At the end of `saveAttestation()` in both `InMemoryLedgerEntryRepository`
and `JpaLedgerEntryRepository`. Uses synchronous CDI `Event.fire()` — the CDI container
delivers to `@ObservesAsync` observers asynchronously on a worker thread.

**Not fired by:** `OutcomeRecorder.record()` directly — it fires through `saveAttestation()`.
This ensures the event fires regardless of how an attestation reaches the repository.

---

### 4.2 `OutcomeRecord` value type and `OutcomeRecorder` SPI

#### 4.2.1 `OutcomeRecord`

A builder-based value type representing one decision-with-outcome write. Lives in `api/`
so consumers depend only on `casehub-ledger-api`.

Required fields (must be set via the factory method):
- `actorId` — the decision-making plugin identity (e.g. `"quarkmind:strategy@v1"`)
- `subjectId` — the aggregate this decision belongs to (e.g. game session UUID)
- `verdict` — `AttestationVerdict.SOUND` (win) or `FLAGGED` (loss/error)
- `confidence` — epistemic weight of this outcome observation, in (0.0, 1.0]

Optional fields (have defaults):
- `capabilityTag` — defaults to `CapabilityTag.GLOBAL`
- `actorType` — defaults to `ActorType.AGENT`
- `actorRole` — nullable; the functional role (e.g. `"strategy"`)
- `occurredAt` — defaults to `Instant.now()` at write time
- `attestorId` — defaults to `casehub.ledger.outcome.default-attestor-id` from config
- `attestorType` — defaults to `casehub.ledger.outcome.default-attestor-type` from config

```java
// api/OutcomeRecord.java
public final class OutcomeRecord {
    public final String actorId;
    public final UUID subjectId;
    public final AttestationVerdict verdict;
    public final double confidence;
    public final String capabilityTag;
    public final ActorType actorType;
    public final String actorRole;
    public final Instant occurredAt;
    public final String attestorId;    // null → use configured default
    public final ActorType attestorType; // null → use configured default

    public static Builder of(String actorId, UUID subjectId,
                              AttestationVerdict verdict, double confidence) { ... }

    public static final class Builder {
        public Builder withCapability(String capabilityTag) { ... }
        public Builder withActorType(ActorType actorType) { ... }
        public Builder withActorRole(String actorRole) { ... }
        public Builder withOccurredAt(Instant occurredAt) { ... }
        public Builder withAttestor(String attestorId, ActorType attestorType) { ... }
        public OutcomeRecord build() { ... }
    }
}
```

#### 4.2.2 `OutcomeRecorder`

Blocking interface in `api/`. Consumers that are not reactive inject this.

```java
// api/OutcomeRecorder.java
public interface OutcomeRecorder {
    /**
     * Record a plugin decision and its outcome as a single atomic operation.
     * Writes a LedgerEntry (EVENT) followed by a LedgerAttestation.
     */
    void record(OutcomeRecord record);
}
```

#### 4.2.3 `ReactiveOutcomeRecorder`

Reactive interface in `api/`. QuarkMind's game loop uses this to avoid blocking.

```java
// api/ReactiveOutcomeRecorder.java
public interface ReactiveOutcomeRecorder {
    Uni<Void> record(OutcomeRecord record);
}
```

#### 4.2.4 `DefaultOutcomeRecorder`

Implementation in `runtime/`. `@DefaultBean @ApplicationScoped` — allows consumers to
provide a custom `OutcomeRecorder` via `@ApplicationScoped` if they need custom write
logic, following the platform-standard CDI priority pattern.

Writes `LedgerEntry` then `LedgerAttestation` within a `@Transactional` boundary. Uses
`casehub.ledger.outcome.*` config for attestor defaults when `OutcomeRecord.attestorId`
is null.

**New config group** (`LedgerConfig.OutcomeConfig`):
```
casehub.ledger.outcome.default-attestor-id=   # e.g. "quarkmind:game-engine@v1"
casehub.ledger.outcome.default-attestor-type=SYSTEM
```

`LedgerEntry.entryType` is set to `LedgerEntryType.EVENT`. The `actorId` on the entry is
the plugin identity; the `actorId` on the attestation is the attestor identity.

#### 4.2.5 `BlockingToReactiveOutcomeRecorder`

Bridge implementation in `runtime/`. `@DefaultBean @ApplicationScoped` per the
`reactive-spi-bridge-default-bean` protocol. **No `@IfBuildProperty` gate** — the bridge
has no Hibernate Reactive dependency.

```java
@DefaultBean
@ApplicationScoped
public class BlockingToReactiveOutcomeRecorder implements ReactiveOutcomeRecorder {
    @Inject OutcomeRecorder blocking;

    @Override
    public Uni<Void> record(OutcomeRecord record) {
        return Uni.createFrom()
                  .item(() -> { blocking.record(record); return null; })
                  .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

**Placement:** `api/` holds `OutcomeRecord`, `OutcomeRecorder`, `ReactiveOutcomeRecorder`.
`runtime/service/` holds `DefaultOutcomeRecorder` and `BlockingToReactiveOutcomeRecorder`.

---

### 4.3 `LedgerEntryRepository.findEventsByActorId(String actorId)`

New method on the existing SPI. Returns all `LedgerEntryType.EVENT` entries for the given
actor, ordered by sequence number ascending, with no time bounds.

```java
// Added to LedgerEntryRepository SPI
List<LedgerEntry> findEventsByActorId(String actorId);
```

Implementations:
- **In-memory:** stream filter on `entries` by `LedgerEntryType.EVENT` and actor ID.
  Handles `actorIdentityProvider.tokeniseForQuery()` for pseudonymised IDs.
- **JPA:** new `@NamedQuery` on `LedgerEntry`: `SELECT e FROM LedgerEntry e WHERE e.actorId = :actorId AND e.entryType = 'EVENT' ORDER BY e.sequenceNumber ASC`

This is a breaking addition to the SPI. All existing implementations must add the method.
The JPA implementation and the in-memory implementation are both in this repo; there are no
known external implementors.

---

### 4.4 `TrustScoreRecomputeService`

A new `@ApplicationScoped` CDI bean that extracts per-actor trust recomputation from
`TrustScoreJob`. `TrustScoreJob` will delegate its per-actor loop to this service.

```java
// runtime/service/TrustScoreRecomputeService.java
@ApplicationScoped
public class TrustScoreRecomputeService {

    @Transactional(REQUIRES_NEW)
    public void recomputeForActor(String actorId) {
        List<LedgerEntry> decisions = ledgerRepo.findEventsByActorId(actorId);
        if (decisions.isEmpty()) return;

        Set<UUID> entryIds = decisions.stream().map(e -> e.id).collect(toSet());
        Map<UUID, List<LedgerAttestation>> attestationsByEntry =
            ledgerRepo.findAttestationsForEntries(entryIds);

        ActorType actorType = decisions.stream()
            .map(e -> e.actorType).filter(Objects::nonNull).findFirst()
            .orElse(ActorType.AGENT);

        // capability pass, dimension pass, global pass — same logic as TrustScoreJob
        // ...
        // upsert via trustRepo
    }
}
```

`@Transactional(REQUIRES_NEW)` ensures that when called from an `@ObservesAsync` context
(which runs after the originating transaction commits), the recomputation reads committed
attestation data. For in-memory consumers, `@Transactional` is a no-op.

`TrustScoreJob.runComputation()` is refactored to call `recomputeForActor(actorId)` for
each actor in its loop, removing the duplicated per-actor logic.

---

### 4.5 `AttestationTrustUpdateStrategy` SPI and `IncrementalTrustUpdateListener`

#### 4.5.1 `AttestationTrustUpdateStrategy`

Pure interface — no CDI observer annotations. In `runtime/service/` because its method
parameters use `String` and `UUID` only, but implementations will typically inject CDI
beans.

```java
// runtime/service/AttestationTrustUpdateStrategy.java
public interface AttestationTrustUpdateStrategy {
    /**
     * Called when a new attestation has been persisted for an actor.
     * Implementations decide whether and how to trigger trust score recomputation.
     *
     * @param actorId      the attested actor
     * @param capabilityTag the capability tag on the attestation
     */
    void onAttestationRecorded(String actorId, String capabilityTag);
}
```

#### 4.5.2 `ConfigDrivenAttestationTrustUpdateStrategy`

`@DefaultBean @ApplicationScoped`. Reads `casehub.ledger.trust-score.update-trigger`
from `LedgerConfig`:
- `IMMEDIATE` — calls `trustScoreRecomputeService.recomputeForActor(actorId)` on the
  calling thread. Because `IncrementalTrustUpdateListener` delivers this call via
  `@ObservesAsync`, it already runs on a worker thread, not the event loop.
- `SCHEDULED` — no-op; defers entirely to the `TrustScoreJob` batch schedule.
- `NONE` — no-op; no trust score updates at all (for consumers that manage scores externally).

New config key: `casehub.ledger.trust-score.update-trigger` with enum values
`IMMEDIATE | SCHEDULED | NONE`. Default: `IMMEDIATE`.

#### 4.5.3 `IncrementalTrustUpdateListener`

`@ApplicationScoped` CDI bean. Observes `LedgerAttestationRecordedEvent` asynchronously
and delegates to the strategy. Always present in the CDI graph; the strategy impl controls
whether any work is done.

```java
// runtime/service/IncrementalTrustUpdateListener.java
@ApplicationScoped
public class IncrementalTrustUpdateListener {

    @Inject AttestationTrustUpdateStrategy strategy;

    public void onAttestationRecorded(
            @ObservesAsync LedgerAttestationRecordedEvent event) {
        strategy.onAttestationRecorded(event.actorId(), event.capabilityTag());
    }
}
```

---

### 4.6 EigenTrust Startup Validation

At Quarkus startup, if `casehub.ledger.trust-score.eigentrust.enabled=true` and
`casehub.ledger.trust-score.eigentrust.pre-trusted-actors` has fewer than 3 entries (or
is empty), log a WARNING:

```
casehub-ledger: EigenTrust is enabled but pre-trusted-actors has fewer than 3 entries.
EigenTrust is inappropriate for small agent graphs or single-attestor deployments —
results may be degenerate or non-convergent. See ADR 0016.
```

This runs at runtime startup (a `@PostConstruct` or startup observer), not build time.

---

## 5. Data Flow

### 5.1 Game loop write path (QuarkMind)

```
game tick completes → outcome known
  → ReactiveOutcomeRecorder.record(OutcomeRecord.of(pluginId, gameSessionId, verdict, confidence).withCapability("strategy").build())
  → BlockingToReactiveOutcomeRecorder wraps → Worker thread
  → DefaultOutcomeRecorder.record() [@Transactional]
      → ledgerRepo.save(LedgerEntry{actorId=pluginId, subjectId=gameSessionId, entryType=EVENT, ...})
      → ledgerRepo.saveAttestation(LedgerAttestation{attestorId=gameEngineId, verdict=verdict, confidence=confidence, ...})
          → fires LedgerAttestationRecordedEvent(pluginId, capabilityTag, entryId, attestationId) [sync CDI fire]
  → CDI container delivers LedgerAttestationRecordedEvent to @ObservesAsync → Worker thread
  → IncrementalTrustUpdateListener.onAttestationRecorded()
  → ConfigDrivenAttestationTrustUpdateStrategy.onAttestationRecorded()  [IMMEDIATE mode]
  → TrustScoreRecomputeService.recomputeForActor(pluginId) [@Transactional(REQUIRES_NEW)]
      → ledgerRepo.findEventsByActorId(pluginId)
      → ledgerRepo.findAttestationsForEntries(entryIds)
      → TrustScoreComputer.compute(decisions, attestationsByEntry, now)
      → trustRepo.upsert(pluginId, GLOBAL, ...)
      → trustRepo.upsert(pluginId, CAPABILITY, "strategy", ...)
```

### 5.2 Trust score read path (TrustWeightedAgentStrategy)

```
TrustWeightedAgentStrategy (casehub-engine)
  → TrustGateService.currentScore("quarkmind:strategy@v1")
  → ActorTrustScoreRepository.findByActorId("quarkmind:strategy@v1")
  → InMemoryActorTrustScoreRepository.store.get(key)
  → returns Optional<Double>
```

No changes needed to `TrustGateService`, `TrustWeightedAgentStrategy`, or
`ActorTrustScoreRepository` — these already work correctly with in-memory repos.

### 5.3 Multi-granularity confidence weighting

`confidence` is already used in `TrustScoreComputer`:
```
weight = decayFunction.weight(ageInDays, verdict) × confidence
alpha += weight  (SOUND/ENDORSED)
beta  += weight  (FLAGGED/CHALLENGED)
```

So `confidence=0.7` per-game attestations contribute 7× more than `confidence=0.1`
per-tick attestations of the same age. The ratio is constant over time — both decay at the
same rate, only the contribution magnitude differs.

**Recommended confidence values for QuarkMind:**
- Per-tick outcome: `0.1` — high frequency, noisy, use sparingly
- Per-game outcome: `0.7` — recommended default for routing decisions
- Session rollup: `1.0` — long-term performance synthesis

**Recommendation:** record at per-game granularity only. Per-tick writes at 22Hz × 4
plugins = 88 attestations/second — this will grow the in-memory store rapidly and produce
noisy trust signals. Trust routing decisions are made at game start (not tick start), so
per-game outcomes are the correct granularity.

---

## 6. QuarkMind Configuration Reference

Complete `application.properties` for a QuarkMind game session:

```properties
# Disable compliance stack
casehub.ledger.hash-chain.enabled=false
casehub.ledger.agent-identity.validation-mode=WARN

# Enable trust scoring with immediate incremental updates
casehub.ledger.trust-score.enabled=true
casehub.ledger.trust-score.update-trigger=IMMEDIATE
casehub.ledger.trust-score.eigentrust.enabled=false

# Reactive writes (required for non-blocking game loop)
casehub.ledger.reactive.enabled=true

# OutcomeRecorder default attestor (the game engine itself)
casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1
casehub.ledger.outcome.default-attestor-type=SYSTEM

# Actor ID convention for plugins (use in OutcomeRecord.of())
# "quarkmind:strategy@v1", "quarkmind:economics@v1", "quarkmind:tactics@v1", "quarkmind:scouting@v1"
```

Maven dependencies to add:
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-memory</artifactId>
  <scope>runtime</scope>  <!-- application module -->
</dependency>
```

---

## 7. Testing Approach

All tests use in-memory repos (`casehub-ledger-memory`). No datasource required.

### Unit tests (pure Java, no Quarkus)

- `TrustScoreRecomputeService` — inject mock repos, verify per-actor Beta computation
- `ConfigDrivenAttestationTrustUpdateStrategy` — verify IMMEDIATE calls recompute, SCHEDULED/NONE are no-ops
- `OutcomeRecord` builder — validate required fields, defaults

### Integration tests (`@QuarkusTest` with in-memory profile)

1. **`OutcomeRecorderIT`** — call `OutcomeRecorder.record()` with a SOUND outcome; assert
   `LedgerEntryRepository` contains the entry and attestation; assert `TrustGateService.currentScore(actorId)` is present and > 0.5.

2. **`ReactiveOutcomeRecorderIT`** — call `ReactiveOutcomeRecorder.record().await()`;
   assert same post-conditions as above.

3. **`IncrementalTrustUpdateIT`** — record two outcomes (SOUND, then FLAGGED); assert
   trust score decreases after FLAGGED; assert score is updated within the same test
   without manually triggering `TrustScoreJob`.

4. **`EigenTrustStartupValidationIT`** — set `eigentrust.enabled=true` with empty
   `pre-trusted-actors`; assert WARN log entry at startup.

5. **`MultiGranularityIT`** — record a low-confidence tick outcome and a high-confidence
   game outcome for the same actor; assert game-level outcome has dominant influence on
   computed trust score.

### What is NOT tested here

- `TrustWeightedAgentStrategy` integration — tested in `casehub-engine`
- Batch `TrustScoreJob` schedule behaviour — existing tests cover this

---

## 8. Out of Scope

The following were considered and explicitly deferred:

**`DecayFunction` signature extension (add `confidence` param)** — the existing
`weight × confidence` multiplication already handles multi-granularity weighting. Decay-rate
differentiation by confidence (session attestations decay slower than tick attestations of the
same age) is a meaningful improvement for compliance deployments but not needed for game AI
where all attestations come from one source at uniform confidence. Tracked for future consideration.

**DEBOUNCED trust update trigger** — batches burst attestations (e.g. 4 plugin outcomes
at game end) into one recomputation after a quiescence window. `IMMEDIATE` with per-game
recording produces at most 4 recomputations per game (one per plugin), each cheap with
in-memory. Debouncing adds timer infrastructure with marginal benefit at QuarkMind's scale.
Add if per-tick recording becomes necessary.

**New module (`casehub-ledger-outcome`)** — the existing module structure and feature
flags achieve the same separation. A new module adds Maven coordinate overhead without
architectural benefit.

**JPA implementation of `findEventsByActorId` is in scope.** Because `findEventsByActorId`
is a new SPI method, the JPA implementation must ship in this issue — the SPI cannot be
added without it. It is a single `@NamedQuery` addition on `LedgerEntry` and a one-method
addition on `JpaLedgerEntryRepository`. Not deferred.

**`casehub-engine` changes** — `TrustWeightedAgentStrategy` reads from `TrustGateService`
which reads from `ActorTrustScoreRepository`. No changes needed in `casehub-engine` —
the in-memory repo satisfies the injection point.

---

## 9. Module Impact Summary

| Module | Changes |
|---|---|
| `api` | + `OutcomeRecord`, `OutcomeRecorder`, `ReactiveOutcomeRecorder`; + `LedgerConfig.OutcomeConfig` (new config group) |
| `runtime` | + `LedgerAttestationRecordedEvent`, `DefaultOutcomeRecorder`, `BlockingToReactiveOutcomeRecorder`, `AttestationTrustUpdateStrategy`, `ConfigDrivenAttestationTrustUpdateStrategy`, `IncrementalTrustUpdateListener`, `TrustScoreRecomputeService`; ~ `TrustScoreJob` refactored; ~ `LedgerEntryRepository` SPI + `findEventsByActorId`; ~ `LedgerConfig` + `update-trigger` key and `outcome.*` group; + EigenTrust startup validation |
| `deployment` | No changes |
| `persistence-memory` | ~ `InMemoryLedgerEntryRepository`: fire `LedgerAttestationRecordedEvent` from `saveAttestation()`; + `findEventsByActorId()` impl |
| Schema | No migrations |

Legend: `+` = new, `~` = modified
