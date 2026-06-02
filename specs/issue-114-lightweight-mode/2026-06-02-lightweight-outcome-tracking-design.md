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

Requirements:
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
   `ActorTrustScoreRepository`. At application start with a fresh in-memory store, this
   returns zero rows — the cache is empty and every plugin is in BOOTSTRAP phase until
   the first batch run completes. QuarkMind developers should expect availability routing
   (workload-based, Gastown parity) for the first game or two.

2. **`TrustScoreFullPayload` events**: `@Observes onFull()` refreshes the entire cache.
   Fired by `TrustScoreRoutingPublisher` at the end of every `TrustScoreJob.runComputation()`
   — **but only when `casehub.ledger.trust-score.routing-enabled=true`**. Without this
   flag, `publish()` returns early at its first line and `TrustScoreFullPayload` is never
   fired. `TrustScoreCache` stays at startup hydration state forever; all plugins remain
   in BOOTSTRAP permanently. This is the most critical config key in the QuarkMind setup.

3. **`TrustScoreDeltaPayload` events**: `onDelta()` is a **no-op** for capability scores —
   delta payloads carry only GLOBAL scores. CAPABILITY scores update only from full payloads.

Consequence: trust score freshness for routing is governed by the `TrustScoreJob` schedule.
After each batch run (when `routing-enabled=true`), `TrustScoreFullPayload` fires,
`TrustScoreCache` refreshes, and routing sees new scores.

For QuarkMind (4 in-memory actors, games lasting minutes): setting
`casehub.ledger.trust-score.schedule=30s` means routing decisions use scores at most 30
seconds stale. This is correct and complete without any incremental infrastructure. The
incremental pipeline (sub-batch freshness) is out of scope here — tracked in casehubio/ledger#115.

**CAPABILITY scores are what routing reads.** Writes via `OutcomeRecorder` intended to
influence routing must include a named `capabilityTag`. `CapabilityTag.GLOBAL`-tagged
attestations contribute to the global Beta score but not to `TrustScoreCache` and
therefore not to `TrustWeightedAgentStrategy`.

---

## 3. What Already Works — Configuration Only

These requirements are satisfied by existing feature flags. **No new code.**

| Requirement | Mechanism | Config key |
|---|---|---|
| Skip Merkle hash chain | `LedgerConfig.hashChain().enabled()` | `casehub.ledger.hash-chain.enabled=false` |
| Skip agent signing | No key configured → `AgentSignatureEnricher` is a no-op | *(omit `casehub.ledger.agent-signing.keys.*`)* |
| Skip DID/VC validation | No DID configured → identity enrichers are no-ops | *(omit `casehub.ledger.agent-identity.dids.*`)* |
| In-memory persistence | `casehub-ledger-memory` on classpath | Add `casehub-ledger-memory` dependency |
| Trust scoring enabled | `TrustScoreJob` gate | `casehub.ledger.trust-score.enabled=true` |
| Short trust score interval | `TrustScoreJob` schedule | `casehub.ledger.trust-score.schedule=30s` |
| Routing publisher active | `TrustScoreRoutingPublisher` gate | `casehub.ledger.trust-score.routing-enabled=true` |
| EigenTrust disabled | Default | `casehub.ledger.trust-score.eigentrust.enabled=false` (default) |

### Why EigenTrust must stay disabled for QuarkMind

QuarkMind has 4 plugins and a single attestor (the game engine). This is a star attestation
graph — EigenTrust on a star graph is equivalent to direct trust from a single source and
adds no value. Smaller graphs with pre-trusted fallback risk 3-cycle non-convergence
(GE-20260421-09d636). See ADR 0016.

---

## 4. What Is Genuinely Missing — New Additions

Two gaps remain after configuration:

1. **No combined write API.** Recording a decision + outcome currently requires two
   separate calls: `save(LedgerEntry)` then `saveAttestation(LedgerAttestation)`. The
   game loop needs a single call.

2. **EigenTrust activation produces silent wrong results.** When `eigentrust.enabled=true`
   with an insufficient pre-trusted set, power iteration produces degenerate scores with
   no warning. A startup validation is needed.

---

## 5. New Components

### 5.1 `OutcomeRecord` — Java record

A value type for one decision-with-outcome write. Lives in `api/`. Java record with compact
constructor validation, two factory methods (routing path and global path), and `with*`
methods for optional fields.

```java
// api/OutcomeRecord.java
public record OutcomeRecord(
    String actorId,            // required
    UUID subjectId,             // required
    AttestationVerdict verdict, // required
    double confidence,          // required: in (0.0, 1.0]
    String capabilityTag,       // required (use CapabilityTag.GLOBAL if intentional)
    ActorType actorType,        // defaults to ActorType.AGENT
    String actorRole,           // nullable
    Instant occurredAt,         // nullable: defaults to Instant.now() at save time
    String attestorId,          // nullable: defaults to casehub.ledger.outcome.default-attestor-id
    ActorType attestorType      // nullable: must be non-null iff attestorId is non-null
) {
    public OutcomeRecord {
        Objects.requireNonNull(actorId,      "actorId required");
        Objects.requireNonNull(subjectId,    "subjectId required");
        Objects.requireNonNull(verdict,      "verdict required");
        Objects.requireNonNull(capabilityTag,"capabilityTag required — use CapabilityTag.GLOBAL if intentional");
        if (confidence <= 0.0 || confidence > 1.0) {
            throw new IllegalArgumentException(
                "confidence must be in (0.0, 1.0] — got " + confidence
                + ". Recommended: 0.1 (tick), 0.7 (game), 1.0 (session).");
        }
        if ((attestorId == null) != (attestorType == null)) {
            throw new IllegalArgumentException(
                "attestorId and attestorType must be provided together or both left null.");
        }
        if (actorType == null) actorType = ActorType.AGENT;
    }

    /**
     * Primary factory for routing-aware outcome recording.
     * capabilityTag is required because GLOBAL-scoped attestations do not reach
     * TrustScoreCache and therefore do not influence TrustWeightedAgentStrategy.
     *
     * Example: OutcomeRecord.of("quarkmind:strategy@v1", gameId, "strategy", SOUND, 0.7)
     */
    public static OutcomeRecord of(String actorId, UUID subjectId, String capabilityTag,
                                    AttestationVerdict verdict, double confidence) {
        return new OutcomeRecord(actorId, subjectId, verdict, confidence,
                capabilityTag, ActorType.AGENT, null, null, null, null);
    }

    /**
     * Factory for outcomes that intentionally target the global Beta score only.
     * These do NOT reach TrustScoreCache or TrustWeightedAgentStrategy.
     * Use only when capability-differentiated routing is not the goal.
     */
    public static OutcomeRecord ofGlobal(String actorId, UUID subjectId,
                                          AttestationVerdict verdict, double confidence) {
        return new OutcomeRecord(actorId, subjectId, verdict, confidence,
                CapabilityTag.GLOBAL, ActorType.AGENT, null, null, null, null);
    }

    /** @throws NullPointerException if role is null */
    public OutcomeRecord withActorRole(String role) {
        Objects.requireNonNull(role, "role");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, role, occurredAt, attestorId, attestorType);
    }

    /**
     * @throws NullPointerException if t is null.
     *   Pass ActorType.AGENT explicitly rather than null to "clear" the type —
     *   null is rejected to prevent confusion with the AGENT default.
     */
    public OutcomeRecord withActorType(ActorType t) {
        Objects.requireNonNull(t, "actorType — use ActorType.AGENT to set the default explicitly");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                t, actorRole, occurredAt, attestorId, attestorType);
    }

    /** @throws NullPointerException if ts is null */
    public OutcomeRecord withOccurredAt(Instant ts) {
        Objects.requireNonNull(ts, "occurredAt");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, actorRole, ts, attestorId, attestorType);
    }

    /**
     * Override the attestor for this record. Both id and type must be non-null;
     * they are always set together to maintain the invariant enforced by the canonical constructor.
     */
    public OutcomeRecord withAttestor(String id, ActorType t) {
        Objects.requireNonNull(id, "attestorId");
        Objects.requireNonNull(t,  "attestorType");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, actorRole, occurredAt, id, t);
    }
}
```

**Placement:** `api/`

**`capabilityTag` is required in the primary factory** because omitting it silently breaks
routing — GLOBAL attestations do not reach `TrustScoreCache`. Callers who want GLOBAL must
use `ofGlobal()`, making the choice explicit. The compiler enforces routing correctness.

**Multi-granularity via `confidence`:** `TrustScoreComputer` computes
`weight = decayFunction.weight(ageInDays, verdict) × confidence`. A `confidence=0.7` game
attestation contributes 7× more than a `confidence=0.1` tick attestation of the same age.

Recommended values for QuarkMind:
- Per-tick outcome: `0.1` — noisy; use sparingly or not at all
- Per-game outcome (recommended): `0.7`
- Session rollup: `1.0`

**Recommendation: record at per-game granularity** — per-tick at 22Hz × 4 plugins = 88/second
accumulates rapidly and produces noisy trust signals.

---

### 5.2 `OutcomeRecorder` — blocking interface

```java
// api/OutcomeRecorder.java
public interface OutcomeRecorder {
    /**
     * Record a plugin decision and its outcome as a single atomic operation.
     * Writes a LedgerEntry (EVENT) followed by a LedgerAttestation, committed together.
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

### 5.4 `DefaultOutcomeRecorder` and `OutcomeRecordSaveService`

#### Transaction demarcation

`DefaultOutcomeRecorder.record()` is **not** `@Transactional`. It delegates writes to a
package-private `@ApplicationScoped OutcomeRecordSaveService`, which is `@Transactional`.
After `saveService.save()` returns, the transaction has committed (for JPA consumers) or
the writes are immediately visible (for in-memory consumers). Any future incremental trust
update trigger (#115) would be invoked here, after the commit.

This prevents the race condition where an `@ObservesAsync` observer fires before the
originating transaction commits and reads uncommitted data.

#### `DefaultOutcomeRecorder`

`@DefaultBean @ApplicationScoped`. Allows consumers to substitute a custom `OutcomeRecorder`
via `@ApplicationScoped`.

```java
// runtime/service/DefaultOutcomeRecorder.java
@DefaultBean
@ApplicationScoped
public class DefaultOutcomeRecorder implements OutcomeRecorder {

    @Inject OutcomeRecordSaveService saveService;
    @Inject LedgerConfig config;

    @Override
    public void record(OutcomeRecord record) {
        AttestorDefaults attestor = resolveAttestor(record);
        saveService.save(record, attestor);
    }

    private AttestorDefaults resolveAttestor(OutcomeRecord record) {
        if (record.attestorId() != null) {
            // withAttestor() enforces both non-null; compact constructor validates the pair
            return new AttestorDefaults(record.attestorId(), record.attestorType());
        }
        String id = config.outcome().defaultAttestorId().orElseThrow(() ->
            new IllegalStateException(
                "OutcomeRecord.attestorId is null and casehub.ledger.outcome.default-attestor-id"
                + " is not configured. Set one of them before calling record()."));
        ActorType type = config.outcome().defaultAttestorType();
        return new AttestorDefaults(id, type);
    }
}
```

#### `AttestorDefaults`

Package-private record in `runtime/service/`:

```java
// runtime/service/AttestorDefaults.java (package-private)
record AttestorDefaults(String attestorId, ActorType attestorType) {}
```

#### `OutcomeRecordSaveService`

`@ApplicationScoped` (required for Quarkus ArC to discover the bean and apply the
`@Transactional` interceptor). Package-private visibility; the generated ArC subclass
is placed in the same package.

```java
// runtime/service/OutcomeRecordSaveService.java (package-private)
@ApplicationScoped
class OutcomeRecordSaveService {

    @Inject LedgerEntryRepository ledgerRepo;

    @Transactional
    void save(OutcomeRecord record, AttestorDefaults attestor) {
        LedgerEntry entry = buildEntry(record);
        ledgerRepo.save(entry);                          // repo assigns sequenceNumber, enriches

        LedgerAttestation attestation = buildAttestation(record, entry, attestor);
        ledgerRepo.saveAttestation(attestation);
    }

    private LedgerEntry buildEntry(OutcomeRecord record) {
        LedgerEntry entry = new LedgerEntry();
        entry.actorId     = record.actorId();
        entry.actorRole   = record.actorRole();
        entry.actorType   = record.actorType();
        entry.subjectId   = record.subjectId();
        entry.entryType   = LedgerEntryType.EVENT;
        entry.occurredAt  = record.occurredAt();  // null → repo fills at persist time
        return entry;
    }

    private LedgerAttestation buildAttestation(OutcomeRecord record, LedgerEntry saved,
                                                AttestorDefaults attestor) {
        LedgerAttestation a = new LedgerAttestation();
        a.ledgerEntryId = saved.id;
        a.subjectId     = saved.subjectId;
        a.attestorId    = attestor.attestorId();
        a.attestorType  = attestor.attestorType();
        a.verdict       = record.verdict();
        a.confidence    = record.confidence();
        a.capabilityTag = record.capabilityTag();
        a.occurredAt    = record.occurredAt();  // null → LedgerAttestation @PrePersist fills it
        return a;
    }
}
```

**`sequenceNumber` responsibility:** `sequenceNumber` is assigned by
`LedgerEntryRepository.save()`, not by `buildEntry()`. `InMemoryLedgerEntryRepository.save()`
handles it via `ConcurrentHashMap<UUID, AtomicInteger>` keyed by `subjectId`.
`JpaLedgerEntryRepository.save()` does not currently assign `sequenceNumber` — this is a
pre-existing gap in the JPA implementation that exists regardless of `OutcomeRecorder`.
JPA consumers must address `sequenceNumber` assignment in their `LedgerEntryRepository`
before using `DefaultOutcomeRecorder` in production. The in-memory path (QuarkMind's use
case) is not affected.

---

### 5.5 `BlockingToReactiveOutcomeRecorder` — bridge

`@DefaultBean @ApplicationScoped`. **No `@IfBuildProperty` gate** — the bridge has no
Hibernate Reactive dependency; it wraps the blocking `OutcomeRecorder` on the worker pool.

```java
// runtime/service/BlockingToReactiveOutcomeRecorder.java
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

### 5.6 `EigenTrustStartupValidator`

A new `@ApplicationScoped` bean. At application startup (`@Observes StartupEvent`): if
`casehub.ledger.trust-score.eigentrust.enabled=true` and
`casehub.ledger.trust-score.eigentrust.pre-trusted-actors` has fewer than 3 entries or is
empty, log a WARNING:

```
casehub-ledger: EigenTrust is enabled but pre-trusted-actors has fewer than 3 entries.
EigenTrust is inappropriate for small agent graphs or single-attestor deployments —
results may be degenerate or non-convergent. Disable with:
casehub.ledger.trust-score.eigentrust.enabled=false (the default). See ADR 0016.
```

`StartupEvent` fires at runtime startup, so the runtime-phase config value
(`pre-trusted-actors`) is available. A `@BuildStep` in the deployment module would provide
earlier feedback, but build-time config does not include runtime-phase lists — startup
observer is the practical option.

**Placement:** `runtime/service/EigenTrustStartupValidator.java`

---

### 5.7 `LedgerConfig` additions

New `OutcomeConfig` inner interface in `runtime/config/LedgerConfig.java`.
`OutcomeConfig` is in `runtime/`, not `api/`. `@ConfigMapping` and `@ConfigRoot` are
SmallRye Config annotations; they do not exist in `api/`.

```java
// In runtime/config/LedgerConfig.java
OutcomeConfig outcome();

interface OutcomeConfig {
    /**
     * Default attestor ID used when OutcomeRecord.attestorId() is null.
     * For QuarkMind: "quarkmind:game-engine@v1".
     * If empty and OutcomeRecord.attestorId() is also null, DefaultOutcomeRecorder.record()
     * throws IllegalStateException at call time.
     */
    java.util.Optional<String> defaultAttestorId();

    /**
     * Default attestor type used when OutcomeRecord.attestorType() is null.
     * Ignored when defaultAttestorId is also empty.
     *
     * @return SYSTEM by default
     */
    @WithDefault("SYSTEM")
    io.casehub.platform.api.identity.ActorType defaultAttestorType();
}
```

---

## 6. Data Flow

### 6.1 Game loop write path (QuarkMind)

```
game outcome known

→ ReactiveOutcomeRecorder.record(
      OutcomeRecord.of("quarkmind:strategy@v1", gameSessionId, "strategy", SOUND, 0.7)
  )
→ BlockingToReactiveOutcomeRecorder wraps → worker thread
→ DefaultOutcomeRecorder.record()  [NOT @Transactional]
    → resolveAttestor() — reads config defaults; throws IllegalStateException if unconfigured
    → OutcomeRecordSaveService.save()  [@Transactional — commits both writes]
        → ledgerRepo.save(LedgerEntry{
              actorId="quarkmind:strategy@v1", subjectId=gameSessionId, entryType=EVENT
          })
          // InMemoryLedgerEntryRepository.save() assigns sequenceNumber
        → ledgerRepo.saveAttestation(LedgerAttestation{
              attestorId="quarkmind:game-engine@v1", attestorType=SYSTEM,
              verdict=SOUND, confidence=0.7, capabilityTag="strategy"
          })
    [transaction committed]
```

### 6.2 Trust score update path

```
TrustScoreJob fires every 30s
→ computeTrustScores() — gates on casehub.ledger.trust-score.enabled=true
→ runComputation()  [@Transactional — all actors, one transaction]
    → findAllEvents() → groups by actorId
    → capability pass, dimension pass, global pass per actor
    → trustRepo.upsert(actorId, CAPABILITY, "strategy", ...)
    → trustRepo.upsert(actorId, GLOBAL, ...)
    [all upserts committed atomically]
→ TrustScoreRoutingPublisher.publish()
    — gates on casehub.ledger.trust-score.routing-enabled=true
    → fires TrustScoreFullPayload(all current CAPABILITY scores)

TrustScoreCache.onFull() [in casehub-engine]
    → refreshes ConcurrentHashMap: "quarkmind:strategy@v1:strategy" → {score, decisionCount}
```

### 6.3 Routing read path

```
TrustWeightedAgentStrategy.select(context, candidates)
→ TrustCandidateClassifier.classify(candidates, "strategy", policy, cache)
→ TrustScoreCache.getCapabilityScore("quarkmind:strategy@v1", "strategy")
    → reads "quarkmind:strategy@v1:strategy" from ConcurrentHashMap
→ phase classification (BOOTSTRAP if decisionCount < minimumObservations, else QUALIFIED)
→ blended score = trust × blendFactor + workload × (1 - blendFactor)
```

**Key point:** writes use `capabilityTag="strategy"`. Routing reads CAPABILITY scores from
`TrustScoreCache`. The write tag must match the routing context capability name
(`context.capabilityName()`). Verify at integration time.

### 6.4 Batch atomicity

`TrustScoreJob.runComputation()` remains `@Transactional`. All actor upserts commit
together or not at all. This is unchanged.

---

## 7. QuarkMind Configuration Reference

```properties
# Disable compliance overhead
casehub.ledger.hash-chain.enabled=false

# Enable trust scoring
casehub.ledger.trust-score.enabled=true
casehub.ledger.trust-score.schedule=30s
casehub.ledger.trust-score.routing-enabled=true   # CRITICAL: without this, TrustScoreCache never refreshes
casehub.ledger.trust-score.eigentrust.enabled=false

# Reactive writes
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
  <scope>runtime</scope>
</dependency>
```

Actor ID convention for QuarkMind plugins:
`"quarkmind:strategy@v1"`, `"quarkmind:economics@v1"`, `"quarkmind:tactics@v1"`, `"quarkmind:scouting@v1"`

The capability tag in `OutcomeRecord.of()` must match the capability name declared in the
QuarkMind case definition YAML and used by `TrustRoutingPolicyProvider.forCapability()`.
Verify that these namespaces align — `TrustScoreCache` has a comment warning that a
mismatch causes every lookup to return empty, silently keeping all plugins in BOOTSTRAP.

---

## 8. Testing Approach

New integration tests live in the `runtime` module under
`src/test/java/io/casehub/ledger/runtime/service/` — the same package as
`TrustScoreJob`. This is required because `TrustScoreJob.runComputation()` has
package-accessible visibility (`void runComputation()`, not `public`) as documented in the
Javadoc: "exposed with package-accessible visibility for direct invocation in integration
tests where the scheduler is disabled via a test profile."

### Unit tests (pure Java, no Quarkus)

**`OutcomeRecordTest`** — compact constructor:
- `actorId=null` → NullPointerException
- `subjectId=null` → NullPointerException
- `capabilityTag=null` → NullPointerException
- `confidence=0.0` → IllegalArgumentException
- `confidence=1.1` → IllegalArgumentException
- `confidence=0.7` → accepted
- `actorType=null` → defaults to `ActorType.AGENT`
- `attestorId="id", attestorType=null` → IllegalArgumentException (mixed state)
- `attestorId=null, attestorType=SYSTEM` → IllegalArgumentException (mixed state)
- `attestorId=null, attestorType=null` → accepted (use config defaults)
- `of(actorId, subjectId, "strategy", SOUND, 0.7)` → capabilityTag="strategy"
- `ofGlobal(actorId, subjectId, SOUND, 0.7)` → capabilityTag=CapabilityTag.GLOBAL
- `withActorType(null)` → NullPointerException (null rejected, not silently AGENT)
- `withActorType(ActorType.HUMAN)` → produces record with actorType=HUMAN, all others unchanged
- `with*` methods return new record instances (record immutability)

**`TrustScoreComputerConfidenceTest`** (pure Java, no Quarkus) — verifying multi-granularity weighting:
- Actor A: one SOUND attestation, `confidence=0.7`, `ageInDays=0` → alpha contribution ≈ 0.7
- Actor B: one SOUND attestation, `confidence=0.1`, `ageInDays=0` → alpha contribution ≈ 0.1
- `TrustScoreComputer` initialises α = β = 1.0 (Jeffreys prior); stored alpha includes the prior.
  Assert: `(A.alpha − 1.0) / (B.alpha − 1.0) == 7.0` (i.e., `0.7 / 0.1`)
- Uses `TrustScoreComputer` directly; verifiable because each actor has exactly one attestation

### Integration tests (`@QuarkusTest`, in-memory profile)

**`OutcomeRecorderIT`** (package: `io.casehub.ledger.runtime.service`)
- Call `OutcomeRecorder.record(OutcomeRecord.of(pluginId, gameId, "strategy", SOUND, 0.7))`
- Assert `ledgerRepo.findEntryById()` finds the entry with `actorId=pluginId`, `entryType=EVENT`
- Assert `ledgerRepo.findAttestationsByEntryId()` returns one SOUND attestation with
  `confidence=0.7`, `capabilityTag="strategy"`, `attestorId` from config default
- Call `TrustScoreJob.runComputation()` directly
- Assert `ActorTrustScoreRepository.findCapabilityScore(pluginId, "strategy")` is present and `trustScore > 0.5`

**`ReactiveOutcomeRecorderIT`**
- Call `ReactiveOutcomeRecorder.record(record).await().indefinitely()`
- Assert same post-conditions as `OutcomeRecorderIT`

**`MultiCapabilityIT`**
- Record outcomes for all 4 plugins with their respective `capabilityTag` values
- Economics plugin: 3 SOUND, 7 FLAGGED (consistently bad actor)
- Strategy plugin: 9 SOUND, 1 FLAGGED (consistently good actor)
- Call `TrustScoreJob.runComputation()`
- Assert `ActorTrustScoreRepository.findCapabilityScore("quarkmind:strategy@v1", "strategy").trustScore`
  > `ActorTrustScoreRepository.findCapabilityScore("quarkmind:economics@v1", "economics").trustScore`
- Assert both have distinct CAPABILITY rows in `ActorTrustScoreRepository`

**`OutcomeRecorderDefaultAttestorIT`**
- Configure `casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1`
- Record an `OutcomeRecord` created with `of()` (no explicit `withAttestor()`)
- Assert saved `LedgerAttestation.attestorId == "quarkmind:game-engine@v1"`

**`OutcomeRecorderUnconfiguredAttestorIT`**
- Do NOT configure `casehub.ledger.outcome.default-attestor-id`
- Call `OutcomeRecorder.record()` with a record that has null `attestorId`
- Assert `IllegalStateException` thrown with message containing "default-attestor-id is not configured"

**`EigenTrustStartupValidationIT`**
- Configure `eigentrust.enabled=true`, `pre-trusted-actors` empty
- Assert WARN log contains "EigenTrust is enabled but pre-trusted-actors has fewer than 3 entries"

---

## 9. Out of Scope

**Incremental per-actor trust recomputation (casehubio/ledger#115):** Sub-batch trust
freshness. Descoped because it requires `TrustScoreCache` per-actor refresh in
casehub-engine, transaction demarcation to avoid race conditions, and `TrustScoreFullPayload`
semantics resolution.

**On-read trust computation:** Compute trust from raw history at query time, bypassing the
materialized store. Requires `TrustWeightedAgentStrategy` to accept a pluggable trust source
SPI (a casehub-engine change). File as a casehub-engine follow-up.

**`DecayFunction` signature extension:** The existing `weight × confidence` in
`TrustScoreComputer` handles multi-granularity. Decay-rate differentiation by confidence
is a future enhancement.

**CRDT incremental accumulator:** `alpha += weight_at_record_time`. Incorrect for
deployments running more than hours — `TrustScoreComputer` computes `ageInDays` from
`Instant.now()` at recomputation time, so historical attestations are correctly
re-weighted on each batch run. CRDT bakes in `ageInDays=0` at record time and never
corrects it.

**`JpaLedgerEntryRepository.sequenceNumber` gap (casehubio/ledger#116):** `InMemoryLedgerEntryRepository.save()`
assigns `sequenceNumber` via a `ConcurrentHashMap<UUID, AtomicInteger>`. `JpaLedgerEntryRepository.save()`
does not currently assign `sequenceNumber` — it calls `em.persist(entry)` directly. This
is a pre-existing gap in the JPA implementation, not introduced by `OutcomeRecorder`. JPA
consumers must address `sequenceNumber` assignment before using `DefaultOutcomeRecorder`
in production. Not in scope for this issue.

**In-memory store memory growth for long-running processes (casehubio/ledger#117):**
`InMemoryLedgerEntryRepository` accumulates entries indefinitely with no eviction or session-boundary
reset. For QuarkMind's recommended per-game granularity (4 writes per game), growth is slow in
practice. A `clear()` / reset mechanism or bounded window query (from #115) addresses this day-2.

**On-read trust score computation (casehubio/ledger#118):** A `TrustSourceProvider` SPI in
`casehub-engine` would let `TrustWeightedAgentStrategy` compute scores on demand from raw
attestation history, eliminating staleness entirely. Requires a `casehub-engine` change.

**`findEventsByActorId` on `LedgerEntryRepository`:** Was needed by the incremental
pipeline, which is now descoped. The existing `findByActorId(actorId, from, to)` is
sufficient for other per-actor queries. Deferred to casehubio/ledger#115.

---

## 10. Module Impact Summary

| Module | Changes |
|---|---|
| `api` | + `OutcomeRecord` (Java record), `OutcomeRecorder`, `ReactiveOutcomeRecorder` |
| `runtime` | + `DefaultOutcomeRecorder` (@DefaultBean), `OutcomeRecordSaveService` (package-private @ApplicationScoped @Transactional), `BlockingToReactiveOutcomeRecorder` (@DefaultBean bridge), `AttestorDefaults` (package-private record), `EigenTrustStartupValidator` (@ApplicationScoped); ~ `LedgerConfig` + `outcome.*` config group |
| `deployment` | No changes |
| `persistence-memory` | No changes |
| Schema | No migrations |

Legend: `+` = new, `~` = modified

### Pre-existing methods referenced (not new)

- `LedgerEntryRepository.save()` — already exists; `OutcomeRecordSaveService` calls it
- `LedgerEntryRepository.saveAttestation()` — already exists; `OutcomeRecordSaveService` calls it
- `LedgerEntryRepository.findAttestationsForEntries()` — already exists; `TrustScoreJob` calls it
- `TrustScoreJob.runComputation()` — unchanged; batch schedule is the trust freshness lever
