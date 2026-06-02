# Lightweight Outcome-Tracking Mode — Design Spec

**Issue:** casehubio/ledger#114
**Date:** 2026-06-02
**Branch:** issue-114-lightweight-mode
**Incremental pipeline deferred to:** casehubio/ledger#115

---

## 1. Context

QuarkMind (`mdproctor/quarkmind`) needs outcome-tracking to enable trust-weighted plugin
routing at game-loop granularity (~22Hz). Four plugins — strategy, economics, tactics,
scouting — make decisions each game tick. QuarkMind wants to record plugin outcomes and
feed them into trust scoring for adaptive routing via `TrustWeightedAgentStrategy`.

The compliance stack (Merkle hash chain, DID/VC identity verification, Ed25519 bilateral
signing) is not needed here. Game AI decisions are not compliance artifacts; signing
overhead per entry is unacceptable at game-loop frequency.

The core requirements from QuarkMind:
- Async, non-blocking writes that do not block the game loop
- No signing, no hash chain, no identity verification overhead
- In-memory backend for game sessions (no DB)
- Trust scores queryable per plugin capability
- Compatible with `TrustWeightedAgentStrategy` from `casehub-engine`

---

## 2. Routing Architecture — How Trust Reaches Routing

Understanding the actual read path is essential before designing the write path.

`TrustWeightedAgentStrategy` does **not** read `TrustGateService`. It reads `TrustScoreCache`
(`io.casehub.ledger.routing.TrustScoreCache` in `casehub-engine`), an in-memory
`ConcurrentHashMap` keyed by `"actorId:capabilityKey"`. `TrustScoreCache` populates via:

1. **Startup hydration**: `@PostConstruct hydrate()` reads all rows from
   `ActorTrustScoreRepository` directly.
2. **`TrustScoreFullPayload` events**: `@Observes onFull()` refreshes the entire cache.
   Fired by `TrustScoreRoutingPublisher` at the end of every `TrustScoreJob.runComputation()`.
3. **`TrustScoreDeltaPayload` events**: `onDelta()` is a **no-op** — delta payloads carry
   only GLOBAL scores; CAPABILITY scores update only from full payloads.

Consequence: trust score freshness for routing is governed by the `TrustScoreJob` schedule.
After each batch run, `TrustScoreFullPayload` refreshes the cache and routing sees new scores.

For QuarkMind (4 in-memory actors, games lasting minutes): setting
`casehub.ledger.trust-score.schedule=30s` means routing decisions use scores that are at
most 30 seconds stale. Per-game outcome attestations accumulate; the next batch run
incorporates them. This is correct and complete without any incremental infrastructure.

An incremental update pipeline (sub-batch trust freshness) is out of scope here because it
requires: (a) `TrustScoreCache` per-actor refresh in `casehub-engine`, (b) transaction
demarcation to avoid race conditions, and (c) `TrustScoreFullPayload` semantics resolution.
Tracked in casehubio/ledger#115.

**CAPABILITY scores are what routing reads.** All writes via `OutcomeRecorder` that are
intended to influence routing must include a `capabilityTag`. GLOBAL-tagged attestations
contribute to the global Beta score but not to `TrustScoreCache` and therefore not to
`TrustWeightedAgentStrategy`.

---

## 3. What Already Works — Configuration Only

These requirements are satisfied by existing feature flags. **No new code.**

| Requirement | Mechanism | Config key |
|---|---|---|
| Skip Merkle hash chain | `LedgerConfig.hashChain().enabled()` | `casehub.ledger.hash-chain.enabled=false` |
| Skip agent signing | No key configured → `AgentSignatureEnricher` is a no-op | *(omit `casehub.ledger.agent-signing.keys.*`)* |
| Skip DID/VC validation | No DID configured → identity enrichers are no-ops | *(omit `casehub.ledger.agent-identity.dids.*`)* |
| In-memory persistence | `casehub-ledger-memory` on classpath | Add `casehub-ledger-memory` dependency |
| Short trust score interval | `TrustScoreJob` schedule | `casehub.ledger.trust-score.schedule=30s` |
| EigenTrust disabled | Default | `casehub.ledger.trust-score.eigentrust.enabled=false` (default) |

### Why EigenTrust must stay disabled for QuarkMind

QuarkMind has 4 plugins and a single attestor (the game engine). This is a star attestation
graph — EigenTrust on a star graph is equivalent to direct trust from a single source and
adds no value. Smaller graphs with pre-trusted fallback risk 3-cycle non-convergence
(GE-20260421-09d636). See ADR 0016.

### Why `casehub.ledger.trust-score.schedule` is the freshness lever

`TrustScoreJob` is configurable via `casehub.ledger.trust-score.schedule` (default `24h`).
For QuarkMind: `30s`. A full batch run across 4 in-memory actors is microseconds. The job
fires `TrustScoreFullPayload`, `TrustScoreCache` refreshes, `TrustWeightedAgentStrategy`
sees new scores. No incremental infrastructure needed.

---

## 4. What Is Genuinely Missing — New Additions

Two gaps remain after configuration:

1. **No combined write API.** Recording a decision + outcome currently requires two separate
   calls: `save(LedgerEntry)` then `saveAttestation(LedgerAttestation)`. The game loop
   needs a single call.

2. **EigenTrust activation produces silent wrong results.** When `eigentrust.enabled=true`
   with an insufficient pre-trusted set, power iteration produces degenerate scores with no
   warning. A startup validation log entry is needed.

---

## 5. New Components

### 5.1 `OutcomeRecord` — Java record

A value type for one decision-with-outcome write. Lives in `api/`. Java record with compact
constructor validation, `of()` factory for required fields, and `with*` methods for optionals.

```java
// api/OutcomeRecord.java
public record OutcomeRecord(
    String actorId,           // required: the decision-making plugin identity
    UUID subjectId,            // required: aggregate this decision belongs to (e.g. game session UUID)
    AttestationVerdict verdict, // required: SOUND = positive outcome, FLAGGED = negative outcome
    double confidence,         // required: epistemic weight in (0.0, 1.0]
    String capabilityTag,      // default: CapabilityTag.GLOBAL — use a named capability for routing
    ActorType actorType,       // default: ActorType.AGENT
    String actorRole,          // nullable: functional role (e.g. "strategy")
    Instant occurredAt,        // nullable: defaults to Instant.now() at save time
    String attestorId,         // nullable: defaults to casehub.ledger.outcome.default-attestor-id
    ActorType attestorType     // nullable: defaults to casehub.ledger.outcome.default-attestor-type
) {
    // Compact constructor for validation
    public OutcomeRecord {
        Objects.requireNonNull(actorId, "actorId required");
        Objects.requireNonNull(subjectId, "subjectId required");
        Objects.requireNonNull(verdict, "verdict required");
        if (confidence <= 0.0 || confidence > 1.0) {
            throw new IllegalArgumentException(
                "confidence must be in (0.0, 1.0] — got " + confidence
                + ". Use 0.1 for tick-level, 0.7 for game-level, 1.0 for session-level.");
        }
        if (capabilityTag == null) capabilityTag = CapabilityTag.GLOBAL;
        if (actorType == null)     actorType = ActorType.AGENT;
    }

    /** Factory with required fields only; optionals get defaults. */
    public static OutcomeRecord of(String actorId, UUID subjectId,
                                    AttestationVerdict verdict, double confidence) {
        return new OutcomeRecord(actorId, subjectId, verdict, confidence,
                CapabilityTag.GLOBAL, ActorType.AGENT, null, null, null, null);
    }

    public OutcomeRecord withCapability(String cap)            { return new OutcomeRecord(actorId, subjectId, verdict, confidence, cap,         actorType, actorRole, occurredAt, attestorId, attestorType); }
    public OutcomeRecord withActorType(ActorType t)            { return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag, t,         actorRole, occurredAt, attestorId, attestorType); }
    public OutcomeRecord withActorRole(String role)            { return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag, actorType, role,      occurredAt, attestorId, attestorType); }
    public OutcomeRecord withOccurredAt(Instant ts)            { return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag, actorType, actorRole, ts,         attestorId, attestorType); }
    public OutcomeRecord withAttestor(String id, ActorType t)  { return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag, actorType, actorRole, occurredAt, id,         t           ); }
}
```

**Placement:** `api/` — consumers depend on `casehub-ledger-api` only.

**Multi-granularity via `confidence`:** The Bayesian Beta computation in `TrustScoreComputer`
already applies `weight = decayFunction.weight(ageInDays, verdict) × confidence`. A
`confidence=0.7` game attestation contributes 7× more than a `confidence=0.1` tick
attestation of the same age. No additional mechanism needed.

Recommended values for QuarkMind:
- Per-tick outcome: `0.1` — high frequency, noisy; use sparingly
- Per-game outcome (recommended): `0.7` — meaningful signal for routing
- Session rollup: `1.0` — long-term performance synthesis

**Recommendation: record at per-game granularity.** Per-tick attestations at 22Hz × 4
plugins = 88/second accumulates rapidly in the in-memory store and produces noisy trust
signals. Trust routing decisions happen at game start, not tick frequency.

---

### 5.2 `OutcomeRecorder` — blocking interface

```java
// api/OutcomeRecorder.java
public interface OutcomeRecorder {
    /**
     * Record a plugin decision and its outcome as a single atomic operation.
     * Writes a LedgerEntry (EVENT) followed by a LedgerAttestation.
     * Both writes commit together; neither is visible until the operation completes.
     */
    void record(OutcomeRecord record);
}
```

**Placement:** `api/`

---

### 5.3 `ReactiveOutcomeRecorder` — reactive interface

```java
// api/ReactiveOutcomeRecorder.java
public interface ReactiveOutcomeRecorder {
    /** Non-blocking variant — returns a Uni that completes after both writes commit. */
    Uni<Void> record(OutcomeRecord record);
}
```

**Placement:** `api/`

---

### 5.4 `DefaultOutcomeRecorder` — blocking implementation

`@DefaultBean @ApplicationScoped` in `runtime/service/`. Allows consumers to provide a
custom `OutcomeRecorder` via `@ApplicationScoped` if they need custom write logic.

**Transaction demarcation.** `DefaultOutcomeRecorder.record()` is **not** `@Transactional`.
It delegates writes to a package-private `@Transactional` `OutcomeRecordSaveService`:

```java
// Non-transactional outer method
public void record(OutcomeRecord record) {
    saveService.save(record, resolveAttestor(record));
    // transaction has committed by this point for JPA consumers
    // future: trust update strategy hook lives here (#115)
}

// @Transactional inner service — writes LedgerEntry + LedgerAttestation and commits
@Transactional
void OutcomeRecordSaveService.save(OutcomeRecord record, AttestorDefaults attestor) {
    LedgerEntry entry = buildEntry(record);
    ledgerRepo.save(entry);
    LedgerAttestation attestation = buildAttestation(record, entry, attestor);
    ledgerRepo.saveAttestation(attestation);
}
```

This ensures that for JPA consumers, both writes are visible in the database before any
subsequent operation. For in-memory consumers, `@Transactional` is a no-op and the
distinction is immaterial.

**Attestor defaults.** `OutcomeRecorder` reads `casehub.ledger.outcome.default-attestor-id`
and `casehub.ledger.outcome.default-attestor-type` when `OutcomeRecord.attestorId` is null.
`attestorId` is required on `LedgerAttestation`; providing a deployment-level default avoids
per-call boilerplate. For QuarkMind: `attestorId="quarkmind:game-engine@v1"`, `attestorType=SYSTEM`.

`LedgerEntry.entryType` is set to `LedgerEntryType.EVENT`. `LedgerEntry.actorId` is the
plugin identity. `LedgerAttestation.attestorId` is the game engine identity.

**`OutcomeRecordSaveService` is package-private** — not a public API. It exists solely for
transaction demarcation.

---

### 5.5 `BlockingToReactiveOutcomeRecorder` — bridge

`@DefaultBean @ApplicationScoped` per the `reactive-spi-bridge-default-bean` protocol.
**No `@IfBuildProperty` gate** — the bridge has no Hibernate Reactive dependency and must
be active under all profiles.

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

**Placement:** `runtime/service/`

---

### 5.6 EigenTrust Startup Validation

At application startup (runtime `@PostConstruct` or `@Observes StartupEvent`): if
`casehub.ledger.trust-score.eigentrust.enabled=true` and
`casehub.ledger.trust-score.eigentrust.pre-trusted-actors` has fewer than 3 entries or is
empty, log a WARNING:

```
casehub-ledger: EigenTrust is enabled but pre-trusted-actors has fewer than 3 entries.
EigenTrust is inappropriate for small agent graphs or single-attestor deployments —
results may be degenerate or non-convergent. Disable with:
casehub.ledger.trust-score.eigentrust.enabled=false (the default). See ADR 0016.
```

A `@BuildStep` in the deployment module would give earlier feedback (augmentation-time),
but build-time config does not have access to the runtime `pre-trusted-actors` list.
Runtime startup warning is the practical option.

---

### 5.7 `LedgerConfig` additions

New config group in `LedgerConfig` (in `runtime/`) for `OutcomeRecorder` defaults:

```java
// runtime/config/LedgerConfig.java
OutcomeConfig outcome();

interface OutcomeConfig {
    /**
     * Default attestor ID used when OutcomeRecord.attestorId is null.
     * For QuarkMind: "quarkmind:game-engine@v1"
     */
    java.util.Optional<String> defaultAttestorId();

    /**
     * Default attestor type used when OutcomeRecord.attestorType is null.
     * Defaults to SYSTEM.
     */
    @WithDefault("SYSTEM")
    io.casehub.platform.api.identity.ActorType defaultAttestorType();
}
```

`OutcomeConfig` lives in `runtime/config/LedgerConfig.java`, not in `api/`. `@ConfigMapping`
and `@ConfigRoot` are SmallRye Config annotations that do not exist in `api/`.

---

## 6. Data Flow

### 6.1 Game loop write path (QuarkMind)

```
game outcome known (e.g. game won/lost)

→ ReactiveOutcomeRecorder.record(
      OutcomeRecord.of("quarkmind:strategy@v1", gameSessionId, AttestationVerdict.SOUND, 0.7)
                   .withCapability("strategy")
                   .build()
  )
→ BlockingToReactiveOutcomeRecorder wraps → worker thread
→ DefaultOutcomeRecorder.record()  [NOT @Transactional]
    → OutcomeRecordSaveService.save()  [@Transactional — commits both writes]
        → ledgerRepo.save(LedgerEntry{
              actorId="quarkmind:strategy@v1",
              subjectId=gameSessionId,
              entryType=EVENT,
              occurredAt=now
          })
        → ledgerRepo.saveAttestation(LedgerAttestation{
              attestorId="quarkmind:game-engine@v1",
              attestorType=SYSTEM,
              verdict=SOUND,
              confidence=0.7,
              capabilityTag="strategy"
          })
    [transaction committed]
```

### 6.2 Trust score update path

```
TrustScoreJob fires every 30s (configurable)
→ runComputation()  [@Transactional — all actors in one transaction]
    → findAllEvents() — loads all EVENT entries
    → groups by actorId
    → capability pass, dimension pass, global pass per actor
    → trustRepo.upsert(actorId, CAPABILITY, "strategy", ...)
    → trustRepo.upsert(actorId, GLOBAL, ...)
    → [all upserts committed atomically]
→ TrustScoreRoutingPublisher.publish(currentScores, ...)
    → fires TrustScoreFullPayload(all current CAPABILITY scores)

TrustScoreCache.onFull() [in casehub-engine]
    → refreshes ConcurrentHashMap with all CAPABILITY scores
```

### 6.3 Routing read path

```
game loop: select which plugin implementation to use

→ TrustWeightedAgentStrategy.select(context, candidates)
→ TrustCandidateClassifier.classify(candidates, "strategy", policy, cache)
→ TrustScoreCache.getCapabilityScore("quarkmind:strategy@v1", "strategy")
    → returns OptionalDouble from ConcurrentHashMap
→ phase classification (BOOTSTRAP if decisionCount < minimumObservations, else QUALIFIED)
→ blended score = trust × blendFactor + workload × (1 - blendFactor)
```

**Key point:** writes are CAPABILITY-scoped (`capabilityTag="strategy"`). Routing reads
CAPABILITY scores from `TrustScoreCache`. The read path matches the write path. GLOBAL-tagged
attestations feed the global Beta score but do NOT reach `TrustWeightedAgentStrategy`.

### 6.4 Batch atomicity

`TrustScoreJob.runComputation()` remains `@Transactional`. All actor upserts commit
together or not at all. This is unchanged — the batch atomicity guarantee is preserved.

`TrustScoreJob` does NOT call any new per-actor service. The incremental path is tracked
separately in casehubio/ledger#115.

---

## 7. QuarkMind Configuration Reference

Complete `application.properties` for a QuarkMind game session:

```properties
# Disable compliance overhead
casehub.ledger.hash-chain.enabled=false

# Enable trust scoring — short schedule for game-loop freshness
casehub.ledger.trust-score.enabled=true
casehub.ledger.trust-score.schedule=30s
casehub.ledger.trust-score.eigentrust.enabled=false

# Reactive writes for non-blocking game loop
casehub.ledger.reactive.enabled=true

# OutcomeRecorder default attestor
casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1
casehub.ledger.outcome.default-attestor-type=SYSTEM
```

Maven dependencies:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger</artifactId>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-memory</artifactId>
  <scope>runtime</scope>  <!-- application module scope -->
</dependency>
```

Actor ID convention for QuarkMind plugins:
`"quarkmind:strategy@v1"`, `"quarkmind:economics@v1"`, `"quarkmind:tactics@v1"`, `"quarkmind:scouting@v1"`

---

## 8. Testing Approach

All tests use in-memory repos. No datasource required.

### Unit tests (pure Java, no Quarkus)

- `OutcomeRecord` — compact constructor validation: null actorId throws, null subjectId
  throws, `confidence=0.0` throws, `confidence=1.1` throws, `confidence=0.7` accepted,
  null capabilityTag defaults to `CapabilityTag.GLOBAL`, null actorType defaults to AGENT.
- `OutcomeRecord.with*` — each `with*` produces a new record with the correct field updated
  and all others preserved.

### Integration tests (`@QuarkusTest` with in-memory profile)

**`OutcomeRecorderIT`**
- Call `OutcomeRecorder.record()` with a SOUND outcome, `capabilityTag="strategy"`
- Assert `LedgerEntryRepository` contains one EVENT entry with the plugin actorId
- Assert `LedgerEntryRepository.findAttestationsByEntryId()` returns one SOUND attestation
  with `confidence=0.7` and `capabilityTag="strategy"`
- Manually trigger `TrustScoreJob.runComputation()`
- Assert `TrustGateService.currentScore(actorId, "strategy")` is present and > 0.5
- Assert `ActorTrustScoreRepository.findCapabilityScore(actorId, "strategy")` is present

**`ReactiveOutcomeRecorderIT`**
- Call `ReactiveOutcomeRecorder.record(record).await().indefinitely()`
- Assert same post-conditions as above

**`MultiCapabilityIT`**
- Record outcomes for all 4 plugins (strategy, economics, tactics, scouting)
- One SOUND, three FLAGGED per game for economics (consistently bad)
- Trigger `TrustScoreJob.runComputation()`
- Assert strategy CAPABILITY score > economics CAPABILITY score
- Assert all four plugins have distinct CAPABILITY scores for their respective tags

**`HighConfidenceOutweighsLowIT`**
- Record `confidence=0.1` SOUND tick and `confidence=0.7` SOUND game for same actor
- Trigger recomputation
- Assert alpha contribution from game attestation is 7× the tick attestation contribution
  (verify via `ActorTrustScoreRepository.findCapabilityScore().alpha`)

**`EigenTrustStartupValidationIT`**
- Configure `eigentrust.enabled=true`, empty `pre-trusted-actors`
- Assert WARN log contains "EigenTrust is enabled but pre-trusted-actors has fewer than 3 entries"

**`OutcomeRecorderDefaultAttestorIT`**
- Configure `casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1`
- Record an `OutcomeRecord` with null `attestorId`
- Assert saved `LedgerAttestation.attestorId == "quarkmind:game-engine@v1"`

---

## 9. Out of Scope

**Incremental per-actor trust recomputation (casehubio/ledger#115):** Allows trust scores
to update within seconds of an attestation rather than waiting for the batch schedule.
Descoped because: (a) requires `TrustScoreCache` per-actor refresh in casehub-engine,
(b) has a transaction race condition requiring careful demarcation, (c) the batch schedule
is sufficient for QuarkMind's use case.

**On-read trust computation:** Compute trust score from raw attestation history at query
time, bypassing the materialized store. Eliminates staleness entirely. Requires
`TrustWeightedAgentStrategy` to accept a pluggable trust source SPI. Tracked as a
casehub-engine architectural improvement.

**`DecayFunction` signature extension (add `confidence` param):** The existing
`weight × confidence` in `TrustScoreComputer` handles multi-granularity weighting.
Decay-rate differentiation by confidence level is a future enhancement for compliance
deployments, not needed for game AI.

**CRDT incremental accumulator:** `alpha += weight_at_record_time`. Incorrect for
deployments running longer than hours — decay weight is computed from `Instant.now()` at
recomputation time, not at record time. CRDT bakes in the weight at `ageInDays=0` and
never re-weights. The batch job's history scan is correctness, not inefficiency.

**`findEventsByActorId` on `LedgerEntryRepository`:** Was needed by `TrustScoreRecomputeService`,
which is now descoped. The existing `findByActorId(actorId, from, to)` covers other per-actor
queries. Deferred to casehubio/ledger#115 if the incremental path is built.

**JPA performance for JOINED inheritance queries:** `findAllEvents()` in `TrustScoreJob`
triggers a Hibernate JOIN across all registered subclass tables. For installations with many
subclasses (e.g., `work_item_ledger_entry`, `message_ledger_entry`), each batch run incurs
this cost. Acceptable for QuarkMind (in-memory, no JOIN). For large JPA deployments,
`findEventsByActorId` with a bounded window query (from #115) would reduce the JOIN scope.

---

## 10. Module Impact Summary

| Module | Changes |
|---|---|
| `api` | + `OutcomeRecord` (Java record), `OutcomeRecorder`, `ReactiveOutcomeRecorder` |
| `runtime` | + `DefaultOutcomeRecorder` (@DefaultBean), `OutcomeRecordSaveService` (package-private @Transactional), `BlockingToReactiveOutcomeRecorder` (@DefaultBean bridge); ~ `LedgerConfig` + `outcome.*` config group; + EigenTrust startup validation |
| `deployment` | No changes |
| `persistence-memory` | No changes |
| Schema | No migrations |

Legend: `+` = new, `~` = modified

### Pre-existing methods referenced (not new)

- `LedgerEntryRepository.saveAttestation()` — already exists; `DefaultOutcomeRecorder` calls it
- `LedgerEntryRepository.findAttestationsForEntries()` — already exists; `TrustScoreJob` calls it
- `TrustScoreJob.runComputation()` — unchanged; batch schedule is the trust freshness lever
