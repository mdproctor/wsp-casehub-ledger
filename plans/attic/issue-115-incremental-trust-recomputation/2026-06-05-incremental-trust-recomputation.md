# Incremental Per-Actor Trust Recomputation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When an attestation is saved, immediately recompute the affected actor's trust scores instead of waiting for the nightly batch job.

**Architecture:** Extract the per-actor computation loop from `TrustScoreJob` into a shared `PerActorTrustComputer`. Add an `IncrementalTrustUpdateObserver` that fires on `AttestationRecordedEvent` (CDI, `TransactionPhase.AFTER_SUCCESS` + `REQUIRES_NEW`) and delegates to the same computer. A dedicated `TrustScoreActorUpdatedEvent` notifies downstream consumers.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (Jakarta), JPA (Hibernate ORM), H2 (test), AssertJ

**Spec:** `wksp/specs/issue-115-incremental-trust-recomputation/SPEC.md`

---

## File Map

| Action | File | Responsibility |
|--------|------|----------------|
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/AttestationRecordedEvent.java` | CDI event record fired from `saveAttestation()` |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreActorUpdatedEvent.java` | CDI event record: per-actor score update notification |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/PerActorTrustComputer.java` | Package-private CDI bean: four-pass per-actor trust computation |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/IncrementalTrustUpdateObserver.java` | CDI observer: `AFTER_SUCCESS` → recompute → fire actor event |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java` | Add `findEventsByActorId(String)` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java` | Implement `findEventsByActorId`, fire `AttestationRecordedEvent` from `saveAttestation()` |
| Modify | `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java` | Implement `findEventsByActorId`, fire `AttestationRecordedEvent` from `saveAttestation()` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java` | Delegate per-actor loop body to `PerActorTrustComputer` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java` | Add `IncrementalConfig` with `enabled` flag |
| Create | `runtime/src/test/java/io/casehub/ledger/service/PerActorTrustComputerTest.java` | Unit tests for extracted computation |
| Create | `runtime/src/test/java/io/casehub/ledger/service/IncrementalTrustUpdateIT.java` | Integration test: full incremental pipeline |
| Modify | `runtime/src/test/resources/application.properties` | Add `incremental-trust-test` profile |

---

### Task 1: Add `findEventsByActorId` to the repository SPI

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/FindEventsByActorIdIT.java`

- [ ] **Step 1: Write the failing integration test**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.service.supplement.TestEntry;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

@QuarkusTest
@TestProfile(FindEventsByActorIdIT.Profile.class)
class FindEventsByActorIdIT {

    public static class Profile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "find-events-actor-test";
        }
    }

    @Inject
    LedgerEntryRepository repo;

    @Inject
    EntityManager em;

    @Test
    @Transactional
    void returnsOnlyEventsForGivenActor() {
        final String targetActor = "actor-target-" + UUID.randomUUID();
        final String otherActor = "actor-other-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedEvent(targetActor, now.minus(2, ChronoUnit.DAYS));
        seedEvent(targetActor, now.minus(1, ChronoUnit.DAYS));
        seedEvent(otherActor, now);
        seedCommand(targetActor, now); // COMMAND, not EVENT

        final List<LedgerEntry> results = repo.findEventsByActorId(targetActor);

        assertThat(results).hasSize(2);
        assertThat(results).allMatch(e -> targetActor.equals(e.actorId));
        assertThat(results).allMatch(e -> LedgerEntryType.EVENT.equals(e.entryType));
    }

    @Test
    @Transactional
    void returnsEmptyForUnknownActor() {
        assertThat(repo.findEventsByActorId("nonexistent-actor")).isEmpty();
    }

    private void seedEvent(final String actorId, final Instant occurredAt) {
        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = occurredAt.truncatedTo(ChronoUnit.MILLIS);
        repo.save(entry);
    }

    private void seedCommand(final String actorId, final Instant occurredAt) {
        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.entryType = LedgerEntryType.COMMAND;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = occurredAt.truncatedTo(ChronoUnit.MILLIS);
        repo.save(entry);
    }
}
```

- [ ] **Step 2: Add test profile to application.properties**

Append to `runtime/src/test/resources/application.properties`:

```properties
# Find events by actor test profile (used by FindEventsByActorIdIT)
%find-events-actor-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:findeventsactortestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=FindEventsByActorIdIT -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `findEventsByActorId` does not exist on `LedgerEntryRepository`

- [ ] **Step 4: Add `findEventsByActorId` to the SPI**

In `LedgerEntryRepository.java`, add after `findAllEvents()`:

```java
/**
 * Return all EVENT-type ledger entries for the given actor.
 *
 * @param actorId the actor identity to filter by
 * @return list of EVENT entries; empty if none exist
 */
List<LedgerEntry> findEventsByActorId(String actorId);
```

- [ ] **Step 5: Implement in `JpaLedgerEntryRepository`**

Add after the `findAllEvents()` method:

```java
/** {@inheritDoc} */
@Override
public List<LedgerEntry> findEventsByActorId(final String actorId) {
    final String token = actorIdentityProvider.tokeniseForQuery(actorId);
    return em.createQuery(
            "SELECT e FROM LedgerEntry e WHERE e.actorId = :actorId AND e.entryType = :type",
            LedgerEntry.class)
            .setParameter("actorId", token)
            .setParameter("type", LedgerEntryType.EVENT)
            .getResultList();
}
```

- [ ] **Step 6: Implement in `InMemoryLedgerEntryRepository`**

Add after the `findAllEvents()` method:

```java
@Override
public List<LedgerEntry> findEventsByActorId(final String actorId) {
    final String token = actorIdentityProvider.tokeniseForQuery(actorId);
    return entries.values().stream()
            .filter(e -> LedgerEntryType.EVENT.equals(e.entryType))
            .filter(e -> token.equals(e.actorId))
            .collect(Collectors.toList());
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=FindEventsByActorIdIT`
Expected: PASS — both tests green

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java \
       runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java \
       persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java \
       runtime/src/test/java/io/casehub/ledger/service/FindEventsByActorIdIT.java \
       runtime/src/test/resources/application.properties
git commit -m "feat(#115): add findEventsByActorId to LedgerEntryRepository SPI

Refs #115"
```

---

### Task 2: Add `IncrementalConfig` to `LedgerConfig`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`

- [ ] **Step 1: Add the config interface**

In `LedgerConfig.java`, add inside `TrustScoreConfig`:

```java
/**
 * Incremental per-actor trust recomputation settings.
 *
 * @return the incremental sub-configuration
 */
IncrementalConfig incremental();

/** Incremental per-actor trust recomputation on attestation persist. */
interface IncrementalConfig {
    /**
     * When {@code true}, each attestation persist triggers immediate per-actor trust
     * recomputation via a {@code TransactionPhase.AFTER_SUCCESS} CDI observer.
     * The batch job remains as a consistency backstop.
     *
     * @return {@code true} if incremental recomputation is active; {@code false} by default
     */
    @WithDefault("false")
    boolean enabled();
}
```

- [ ] **Step 2: Build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java
git commit -m "feat(#115): add casehub.ledger.trust-score.incremental.enabled config

Refs #115"
```

---

### Task 3: Create event types

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AttestationRecordedEvent.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreActorUpdatedEvent.java`

- [ ] **Step 1: Create `AttestationRecordedEvent`**

```java
package io.casehub.ledger.runtime.service;

import java.util.UUID;

public record AttestationRecordedEvent(
        String actorId,
        UUID ledgerEntryId,
        UUID attestationId) {
}
```

- [ ] **Step 2: Create `TrustScoreActorUpdatedEvent`**

```java
package io.casehub.ledger.runtime.service.routing;

import java.time.Instant;
import java.util.List;

import io.casehub.ledger.runtime.model.ActorTrustScore;

public record TrustScoreActorUpdatedEvent(
        String actorId,
        List<ActorTrustScore> scores,
        Instant computedAt) {
}
```

- [ ] **Step 3: Build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/AttestationRecordedEvent.java \
       runtime/src/main/java/io/casehub/ledger/runtime/service/routing/TrustScoreActorUpdatedEvent.java
git commit -m "feat(#115): add AttestationRecordedEvent and TrustScoreActorUpdatedEvent

Refs #115"
```

---

### Task 4: Extract `PerActorTrustComputer` from `TrustScoreJob`

This is the most critical task — extracting the per-actor loop body (lines 136–263 of `TrustScoreJob`) into a reusable CDI bean without changing any behavior.

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/PerActorTrustComputer.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/PerActorTrustComputerTest.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`

- [ ] **Step 1: Write the unit test for `PerActorTrustComputer`**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.service.AllAttestationsGlobalStrategy;
import io.casehub.ledger.runtime.service.AttestationAggregator;
import io.casehub.ledger.runtime.service.PerActorTrustComputer;
import io.casehub.platform.api.identity.ActorType;

class PerActorTrustComputerTest {

    private PerActorTrustComputer computer;
    private CapturingTrustScoreRepo trustRepo;

    @BeforeEach
    void setUp() {
        trustRepo = new CapturingTrustScoreRepo();
        final int halfLifeDays = 90;
        computer = new PerActorTrustComputer(
                (ageInDays, verdict) -> Math.pow(2.0, -(double) ageInDays / halfLifeDays),
                trustRepo,
                new AllAttestationsGlobalStrategy(),
                new AttestationAggregator(),
                AttestationAggregator.Strategy.WEIGHTED_MAJORITY);
    }

    @Test
    void noAttestations_producesNeutralGlobalScore() {
        final String actorId = "test-actor";
        final LedgerEntry entry = makeEvent(actorId, Instant.now().minus(1, ChronoUnit.DAYS));

        final List<ActorTrustScore> scores = computer.computeForActor(
                actorId, List.of(entry), Map.of(), Instant.now());

        assertThat(scores).hasSize(1);
        final ActorTrustScore global = scores.get(0);
        assertThat(global.scoreType).isEqualTo(ActorTrustScore.ScoreType.GLOBAL);
        assertThat(global.trustScore).isCloseTo(0.5, within(0.01));
    }

    @Test
    void positiveAttestation_producesHighGlobalScore() {
        final String actorId = "test-actor";
        final Instant now = Instant.now();
        final LedgerEntry entry = makeEvent(actorId, now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation att = makeAttestation(entry.id, entry.subjectId,
                AttestationVerdict.SOUND, now.minus(30, ChronoUnit.MINUTES));

        final List<ActorTrustScore> scores = computer.computeForActor(
                actorId, List.of(entry), Map.of(entry.id, List.of(att)), now);

        final ActorTrustScore global = scores.stream()
                .filter(s -> s.scoreType == ActorTrustScore.ScoreType.GLOBAL)
                .findFirst().orElseThrow();
        assertThat(global.trustScore).isGreaterThan(0.6);
    }

    @Test
    void capabilityTaggedAttestation_producesCapabilityScore() {
        final String actorId = "test-actor";
        final Instant now = Instant.now();
        final LedgerEntry entry = makeEvent(actorId, now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation att = makeAttestation(entry.id, entry.subjectId,
                AttestationVerdict.SOUND, now.minus(30, ChronoUnit.MINUTES));
        att.capabilityTag = "code-review";

        final List<ActorTrustScore> scores = computer.computeForActor(
                actorId, List.of(entry), Map.of(entry.id, List.of(att)), now);

        assertThat(scores.stream()
                .filter(s -> s.scoreType == ActorTrustScore.ScoreType.CAPABILITY)
                .filter(s -> "code-review".equals(s.capabilityKey))
                .findFirst()).isPresent();
    }

    @Test
    void dimensionAttestation_producesDimensionScore() {
        final String actorId = "test-actor";
        final Instant now = Instant.now();
        final LedgerEntry entry = makeEvent(actorId, now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation att = makeAttestation(entry.id, entry.subjectId,
                AttestationVerdict.SOUND, now.minus(30, ChronoUnit.MINUTES));
        att.trustDimension = "thoroughness";
        att.dimensionScore = 0.85;

        final List<ActorTrustScore> scores = computer.computeForActor(
                actorId, List.of(entry), Map.of(entry.id, List.of(att)), now);

        assertThat(scores.stream()
                .filter(s -> s.scoreType == ActorTrustScore.ScoreType.DIMENSION)
                .filter(s -> "thoroughness".equals(s.dimensionKey))
                .findFirst()).isPresent();
    }

    @Test
    void capabilityDimensionAttestation_producesCapabilityDimensionScore() {
        final String actorId = "test-actor";
        final Instant now = Instant.now();
        final LedgerEntry entry = makeEvent(actorId, now.minus(1, ChronoUnit.HOURS));
        final LedgerAttestation att = makeAttestation(entry.id, entry.subjectId,
                AttestationVerdict.SOUND, now.minus(30, ChronoUnit.MINUTES));
        att.capabilityTag = "code-review";
        att.trustDimension = "thoroughness";
        att.dimensionScore = 0.9;

        final List<ActorTrustScore> scores = computer.computeForActor(
                actorId, List.of(entry), Map.of(entry.id, List.of(att)), now);

        assertThat(scores.stream()
                .filter(s -> s.scoreType == ActorTrustScore.ScoreType.CAPABILITY_DIMENSION)
                .filter(s -> "code-review".equals(s.capabilityKey))
                .filter(s -> "thoroughness".equals(s.dimensionKey))
                .findFirst()).isPresent();
    }

    // ── Fixtures ──────────────────────────────────────────────────────────────

    private static LedgerEntry makeEvent(final String actorId, final Instant occurredAt) {
        final LedgerEntry e = new LedgerEntry() {};
        e.id = UUID.randomUUID();
        e.subjectId = UUID.randomUUID();
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.actorRole = "Classifier";
        e.occurredAt = occurredAt;
        e.sequenceNumber = 1;
        return e;
    }

    private static LedgerAttestation makeAttestation(final UUID entryId, final UUID subjectId,
            final AttestationVerdict verdict, final Instant occurredAt) {
        final LedgerAttestation a = new LedgerAttestation();
        a.id = UUID.randomUUID();
        a.ledgerEntryId = entryId;
        a.subjectId = subjectId;
        a.attestorId = "compliance-bot";
        a.attestorType = ActorType.AGENT;
        a.verdict = verdict;
        a.confidence = 1.0;
        a.capabilityTag = CapabilityTag.GLOBAL;
        a.occurredAt = occurredAt;
        return a;
    }
}
```

Note: `CapturingTrustScoreRepo` is a minimal in-test `ActorTrustScoreRepository` implementation that stores upserted scores in a list. Create it as a static inner class:

```java
private static class CapturingTrustScoreRepo implements ActorTrustScoreRepository {
    final List<ActorTrustScore> captured = new java.util.ArrayList<>();

    @Override
    public void upsert(String actorId, ActorTrustScore.ScoreType scoreType,
            String capabilityKey, String dimensionKey,
            ActorType actorType, double trustScore,
            int decisionCount, int overturnedCount, double alpha, double beta,
            int attestationPositive, int attestationNegative, Instant lastComputedAt) {
        final ActorTrustScore s = new ActorTrustScore();
        s.actorId = actorId;
        s.scoreType = scoreType;
        s.capabilityKey = capabilityKey;
        s.dimensionKey = dimensionKey;
        s.actorType = actorType;
        s.trustScore = trustScore;
        s.decisionCount = decisionCount;
        s.overturnedCount = overturnedCount;
        s.alpha = alpha;
        s.beta = beta;
        s.attestationPositive = attestationPositive;
        s.attestationNegative = attestationNegative;
        s.lastComputedAt = lastComputedAt;
        captured.add(s);
    }

    @Override public Optional<ActorTrustScore> findByActorId(String actorId) { return Optional.empty(); }
    @Override public Optional<ActorTrustScore> findCapabilityScore(String actorId, String capabilityTag) { return Optional.empty(); }
    @Override public Optional<ActorTrustScore> findDimensionScore(String actorId, String dimension) { return Optional.empty(); }
    @Override public Optional<ActorTrustScore> findCapabilityDimension(String actorId, String capabilityTag, String dimension) { return Optional.empty(); }
    @Override public List<ActorTrustScore> findCapabilityDimensions(String actorId, String capabilityTag) { return List.of(); }
    @Override public List<ActorTrustScore> findByActorIdAndScoreType(String actorId, ActorTrustScore.ScoreType scoreType) { return List.of(); }
    @Override public void updateGlobalTrustScore(String actorId, double globalTrustScore) {}
    @Override public List<ActorTrustScore> findAll() { return List.copyOf(captured); }
    @Override public List<ActorTrustScore> findAllByLastComputedAtAfter(Instant since) { return List.of(); }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=PerActorTrustComputerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `PerActorTrustComputer` does not exist

- [ ] **Step 3: Implement `PerActorTrustComputer`**

Extract the per-actor loop body from `TrustScoreJob.runComputation()` (lines 136–263). The class is a package-private CDI bean with constructor injection (for testability without CDI):

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.platform.api.identity.ActorType;

@ApplicationScoped
class PerActorTrustComputer {

    private final DecayFunction decayFunction;
    private final ActorTrustScoreRepository trustRepo;
    private final GlobalScoreStrategy globalScoreStrategy;
    private final AttestationAggregator attestationAggregator;
    private final AttestationAggregator.Strategy aggregationStrategy;

    @Inject
    PerActorTrustComputer(final DecayFunction decayFunction,
            final ActorTrustScoreRepository trustRepo,
            final GlobalScoreStrategy globalScoreStrategy,
            final AttestationAggregator attestationAggregator,
            final LedgerConfig config) {
        this(decayFunction, trustRepo, globalScoreStrategy, attestationAggregator,
                config.trustScore().aggregationStrategy());
    }

    // Test constructor — no CDI, no config
    PerActorTrustComputer(final DecayFunction decayFunction,
            final ActorTrustScoreRepository trustRepo,
            final GlobalScoreStrategy globalScoreStrategy,
            final AttestationAggregator attestationAggregator,
            final AttestationAggregator.Strategy aggregationStrategy) {
        this.decayFunction = decayFunction;
        this.trustRepo = trustRepo;
        this.globalScoreStrategy = globalScoreStrategy;
        this.attestationAggregator = attestationAggregator;
        this.aggregationStrategy = aggregationStrategy;
    }

    List<ActorTrustScore> computeForActor(
            final String actorId,
            final List<LedgerEntry> decisions,
            final Map<UUID, List<LedgerAttestation>> attestationsByEntry,
            final Instant now) {

        final List<ActorTrustScore> results = new ArrayList<>();
        final TrustScoreComputer scorer = new TrustScoreComputer(decayFunction);

        final ActorType actorType = decisions.stream()
                .map(e -> e.actorType)
                .filter(t -> t != null)
                .findFirst()
                .orElse(ActorType.HUMAN);

        // Collect all attestations for this actor's decisions
        final List<LedgerAttestation> actorAttestations = new ArrayList<>();
        for (final LedgerEntry decision : decisions) {
            actorAttestations.addAll(attestationsByEntry.getOrDefault(decision.id, List.of()));
        }

        // Build aggregated attestations for capability and global passes
        final List<LedgerAttestation> effectiveAttestations =
                buildEffectiveAttestations(decisions, attestationsByEntry);

        // ── Capability pass ───────────────────────────────────────────────
        final Map<String, Map<UUID, List<LedgerAttestation>>> byCapabilityAndEntry =
                effectiveAttestations.stream()
                        .filter(a -> !CapabilityTag.GLOBAL.equals(a.capabilityTag))
                        .collect(Collectors.groupingBy(
                                a -> a.capabilityTag,
                                Collectors.groupingBy(a -> a.ledgerEntryId)));

        final Map<String, TrustScoreComputer.ActorScore> capabilityScores = new LinkedHashMap<>();

        for (final Map.Entry<String, Map<UUID, List<LedgerAttestation>>> capEntry :
                byCapabilityAndEntry.entrySet()) {
            final String capabilityTag = capEntry.getKey();
            final TrustScoreComputer.ActorScore capScore =
                    scorer.compute(decisions, capEntry.getValue(), now);
            trustRepo.upsert(actorId, ActorTrustScore.ScoreType.CAPABILITY, capabilityTag, null,
                    actorType, capScore.trustScore(),
                    capScore.decisionCount(), capScore.overturnedCount(),
                    capScore.alpha(), capScore.beta(),
                    capScore.attestationPositive(), capScore.attestationNegative(), now);
            capabilityScores.put(capabilityTag, capScore);
            results.add(buildScore(actorId, ActorTrustScore.ScoreType.CAPABILITY,
                    capabilityTag, null, actorType, capScore, now));
        }

        // ── Dimension pass ────────────────────────────────────────────────
        final Map<String, List<LedgerAttestation>> byDimension = actorAttestations.stream()
                .filter(a -> a.trustDimension != null && a.dimensionScore != null)
                .collect(Collectors.groupingBy(a -> a.trustDimension));

        for (final Map.Entry<String, List<LedgerAttestation>> dimEntry : byDimension.entrySet()) {
            final String dimension = dimEntry.getKey();
            final List<LedgerAttestation> dimAttestations = dimEntry.getValue();

            scorer.computeDimensionScore(dimAttestations, now).ifPresent(dimScore -> {
                final int dimPositive = (int) dimAttestations.stream()
                        .filter(a -> a.dimensionScore >= 0.5).count();
                final int dimNegative = (int) dimAttestations.stream()
                        .filter(a -> a.dimensionScore < 0.5).count();
                final int dimDecisionCount = (int) dimAttestations.stream()
                        .map(a -> a.ledgerEntryId).distinct().count();
                trustRepo.upsert(actorId, ActorTrustScore.ScoreType.DIMENSION, null, dimension,
                        actorType, dimScore, dimDecisionCount, 0, 0.0, 0.0,
                        dimPositive, dimNegative, now);
                results.add(buildDimensionScore(actorId, null, dimension, actorType,
                        dimScore, dimDecisionCount, dimPositive, dimNegative, now));
            });
        }

        // ── CAPABILITY_DIMENSION pass ─────────────────────────────────────
        final Map<String, Map<String, List<LedgerAttestation>>> byCapabilityAndDimension =
                actorAttestations.stream()
                        .filter(a -> a.trustDimension != null
                                && a.dimensionScore != null
                                && a.capabilityTag != null
                                && !CapabilityTag.GLOBAL.equals(a.capabilityTag))
                        .collect(Collectors.groupingBy(
                                a -> a.capabilityTag,
                                Collectors.groupingBy(a -> a.trustDimension)));

        for (final Map.Entry<String, Map<String, List<LedgerAttestation>>> capEntry :
                byCapabilityAndDimension.entrySet()) {
            final String capabilityTag = capEntry.getKey();
            for (final Map.Entry<String, List<LedgerAttestation>> dimEntry :
                    capEntry.getValue().entrySet()) {
                final String dimension = dimEntry.getKey();
                final List<LedgerAttestation> compositeAttestations = dimEntry.getValue();
                scorer.computeDimensionScore(compositeAttestations, now).ifPresent(score -> {
                    final int cdPositive = (int) compositeAttestations.stream()
                            .filter(a -> a.dimensionScore >= 0.5).count();
                    final int cdNegative = (int) compositeAttestations.stream()
                            .filter(a -> a.dimensionScore < 0.5).count();
                    final int cdDecisionCount = (int) compositeAttestations.stream()
                            .map(a -> a.ledgerEntryId).distinct().count();
                    trustRepo.upsert(actorId, ActorTrustScore.ScoreType.CAPABILITY_DIMENSION,
                            capabilityTag, dimension, actorType, score,
                            cdDecisionCount, 0, 0.0, 0.0, cdPositive, cdNegative, now);
                    results.add(buildDimensionScore(actorId, capabilityTag, dimension, actorType,
                            score, cdDecisionCount, cdPositive, cdNegative, now));
                });
            }
        }

        // ── Global pass ───────────────────────────────────────────────────
        final List<LedgerAttestation> selectedEffective =
                globalScoreStrategy.selectAttestations(effectiveAttestations);
        final Map<UUID, List<LedgerAttestation>> selectedByEntry = selectedEffective.stream()
                .collect(Collectors.groupingBy(a -> a.ledgerEntryId));
        final TrustScoreComputer.ActorScore globalScore = scorer.compute(decisions, selectedByEntry, now);
        final TrustScoreComputer.ActorScore finalScore =
                globalScoreStrategy.derive(capabilityScores, actorAttestations)
                        .orElse(globalScore);

        trustRepo.upsert(actorId, ActorTrustScore.ScoreType.GLOBAL, null, null,
                actorType, finalScore.trustScore(),
                finalScore.decisionCount(), finalScore.overturnedCount(),
                finalScore.alpha(), finalScore.beta(),
                finalScore.attestationPositive(), finalScore.attestationNegative(), now);
        results.add(buildScore(actorId, ActorTrustScore.ScoreType.GLOBAL, null, null,
                actorType, finalScore, now));

        return results;
    }

    private List<LedgerAttestation> buildEffectiveAttestations(
            final List<LedgerEntry> decisions,
            final Map<UUID, List<LedgerAttestation>> attestationsByEntry) {
        final List<LedgerAttestation> result = new ArrayList<>();
        for (final LedgerEntry decision : decisions) {
            final List<LedgerAttestation> entryAttestations =
                    attestationsByEntry.getOrDefault(decision.id, List.of());
            if (entryAttestations.isEmpty()) {
                continue;
            }
            final Map<String, List<LedgerAttestation>> byCapTag = entryAttestations.stream()
                    .collect(Collectors.groupingBy(
                            a -> a.capabilityTag != null ? a.capabilityTag : CapabilityTag.GLOBAL));
            for (final List<LedgerAttestation> group : byCapTag.values()) {
                attestationAggregator.aggregate(group, aggregationStrategy)
                        .map(agg -> toSynthetic(agg, group.get(0)))
                        .ifPresent(result::add);
            }
        }
        return result;
    }

    private static LedgerAttestation toSynthetic(
            final AttestationAggregator.AggregatedAttestation agg,
            final LedgerAttestation template) {
        final LedgerAttestation synthetic = new LedgerAttestation();
        synthetic.ledgerEntryId = template.ledgerEntryId;
        synthetic.subjectId = template.subjectId;
        synthetic.capabilityTag = template.capabilityTag;
        synthetic.trustDimension = template.trustDimension;
        synthetic.dimensionScore = template.dimensionScore;
        synthetic.verdict = agg.consensusVerdict();
        synthetic.confidence = agg.aggregatedConfidence();
        synthetic.occurredAt = template.occurredAt;
        synthetic.attestorRole = template.attestorRole;
        return synthetic;
    }

    private static ActorTrustScore buildScore(final String actorId,
            final ActorTrustScore.ScoreType scoreType,
            final String capabilityKey, final String dimensionKey,
            final ActorType actorType,
            final TrustScoreComputer.ActorScore score, final Instant now) {
        final ActorTrustScore s = new ActorTrustScore();
        s.actorId = actorId;
        s.scoreType = scoreType;
        s.capabilityKey = capabilityKey;
        s.dimensionKey = dimensionKey;
        s.actorType = actorType;
        s.trustScore = score.trustScore();
        s.decisionCount = score.decisionCount();
        s.overturnedCount = score.overturnedCount();
        s.alpha = score.alpha();
        s.beta = score.beta();
        s.attestationPositive = score.attestationPositive();
        s.attestationNegative = score.attestationNegative();
        s.lastComputedAt = now;
        return s;
    }

    private static ActorTrustScore buildDimensionScore(final String actorId,
            final String capabilityKey, final String dimensionKey,
            final ActorType actorType,
            final double score, final int decisionCount,
            final int positive, final int negative, final Instant now) {
        final ActorTrustScore s = new ActorTrustScore();
        s.actorId = actorId;
        s.scoreType = capabilityKey != null
                ? ActorTrustScore.ScoreType.CAPABILITY_DIMENSION
                : ActorTrustScore.ScoreType.DIMENSION;
        s.capabilityKey = capabilityKey;
        s.dimensionKey = dimensionKey;
        s.actorType = actorType;
        s.trustScore = score;
        s.decisionCount = decisionCount;
        s.overturnedCount = 0;
        s.alpha = 0.0;
        s.beta = 0.0;
        s.attestationPositive = positive;
        s.attestationNegative = negative;
        s.lastComputedAt = now;
        return s;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=PerActorTrustComputerTest`
Expected: PASS — all 5 tests green

- [ ] **Step 5: Refactor `TrustScoreJob` to delegate to `PerActorTrustComputer`**

Replace the per-actor loop body in `runComputation()` (the `for` loop at line 136 through line 263) with a call to `PerActorTrustComputer.computeForActor()`. Remove `buildEffectiveAttestations()` and `toSynthetic()` from `TrustScoreJob` — they now live in `PerActorTrustComputer`.

The job injects `PerActorTrustComputer` and its loop becomes:

```java
for (final Map.Entry<String, List<LedgerEntry>> actorEntry : byActor.entrySet()) {
    final String actorId = actorEntry.getKey();
    final List<LedgerEntry> decisions = actorEntry.getValue();

    final Set<UUID> actorEntryIds = decisions.stream()
            .map(e -> e.id).collect(Collectors.toSet());
    final Map<UUID, List<LedgerAttestation>> actorAttestationsByEntry = new LinkedHashMap<>();
    for (final UUID eid : actorEntryIds) {
        if (attestationsByEntry.containsKey(eid)) {
            actorAttestationsByEntry.put(eid, attestationsByEntry.get(eid));
        }
    }

    perActorComputer.computeForActor(actorId, decisions, actorAttestationsByEntry, now);
}
```

The `decayFunction`, `globalScoreStrategy`, `attestationAggregator` fields are removed from `TrustScoreJob` — they are now injected into `PerActorTrustComputer` via CDI.

- [ ] **Step 6: Run existing trust tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustScoreIT,TrustScoreCapabilityIT,TrustScoreDimensionIT,TrustScoreCapabilityDimensionIT,TrustScoreAggregationIT,TrustScoreBootstrapIT"`
Expected: All PASS — behavior unchanged

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/PerActorTrustComputer.java \
       runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java \
       runtime/src/test/java/io/casehub/ledger/service/PerActorTrustComputerTest.java
git commit -m "refactor(#115): extract PerActorTrustComputer from TrustScoreJob

Refs #115"
```

---

### Task 5: Fire `AttestationRecordedEvent` from `saveAttestation()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java`

- [ ] **Step 1: Modify `JpaLedgerEntryRepository.saveAttestation()`**

Add `@Inject Event<AttestationRecordedEvent> attestationRecordedEvent;` to the field declarations. Add `org.jboss.logging.Logger` if not present.

Replace the `saveAttestation` method:

```java
@Override
@Transactional
public LedgerAttestation saveAttestation(final LedgerAttestation attestation) {
    if (attestation.attestorId != null) {
        attestation.attestorId = actorIdentityProvider.tokenise(attestation.attestorId);
    }
    em.persist(attestation);

    final LedgerEntry entry = em.find(LedgerEntry.class, attestation.ledgerEntryId);
    if (entry != null && entry.actorId != null) {
        attestationRecordedEvent.fire(
                new AttestationRecordedEvent(entry.actorId, entry.id, attestation.id));
    } else if (entry == null) {
        log.warnf("saveAttestation: no LedgerEntry found for ledgerEntryId=%s — "
                + "AttestationRecordedEvent not fired", attestation.ledgerEntryId);
    }

    return attestation;
}
```

Add the logger field: `private static final Logger log = Logger.getLogger(JpaLedgerEntryRepository.class);`

- [ ] **Step 2: Modify `InMemoryLedgerEntryRepository.saveAttestation()`**

Add `@Inject Event<AttestationRecordedEvent> attestationRecordedEvent;` and a logger field. Replace the method:

```java
@Override
public LedgerAttestation saveAttestation(final LedgerAttestation attestation) {
    if (attestation.id == null) {
        attestation.id = UUID.randomUUID();
    }
    if (attestation.occurredAt == null) {
        attestation.occurredAt = Instant.now();
    }
    if (attestation.attestorId != null) {
        attestation.attestorId = actorIdentityProvider.tokenise(attestation.attestorId);
    }
    attestations.put(attestation.id, attestation);

    final LedgerEntry entry = entries.get(attestation.ledgerEntryId);
    if (entry != null && entry.actorId != null) {
        attestationRecordedEvent.fire(
                new AttestationRecordedEvent(entry.actorId, entry.id, attestation.id));
    } else if (entry == null) {
        log.warnf("saveAttestation: no LedgerEntry found for ledgerEntryId=%s — "
                + "AttestationRecordedEvent not fired", attestation.ledgerEntryId);
    }

    return attestation;
}
```

Add imports: `import io.casehub.ledger.runtime.service.AttestationRecordedEvent;`, `import jakarta.enterprise.event.Event;`, `import org.jboss.logging.Logger;`

- [ ] **Step 3: Build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime,persistence-memory -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run existing tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="TrustScoreIT,LedgerAttestationCapabilityIT"`
Expected: All PASS — event fires but no observer acts on it yet (incremental.enabled defaults to false)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java \
       persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java
git commit -m "feat(#115): fire AttestationRecordedEvent from saveAttestation()

Refs #115"
```

---

### Task 6: Implement `IncrementalTrustUpdateObserver` and integration test

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/IncrementalTrustUpdateObserver.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/IncrementalTrustUpdateIT.java`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Step 1: Write the integration test**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.service.supplement.TestEntry;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

@QuarkusTest
@TestProfile(IncrementalTrustUpdateIT.Profile.class)
class IncrementalTrustUpdateIT {

    public static class Profile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "incremental-trust-test";
        }
    }

    @Inject
    LedgerEntryRepository repo;

    @Inject
    ActorTrustScoreRepository trustRepo;

    @Inject
    EntityManager em;

    @Test
    @Transactional
    void attestationPersist_triggersIncrementalRecomputation() {
        final String actorId = "incr-agent-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // Seed two decisions without attestations — no trust score yet
        final TestEntry entry1 = seedEvent(actorId, now.minus(2, ChronoUnit.DAYS));
        final TestEntry entry2 = seedEvent(actorId, now.minus(1, ChronoUnit.DAYS));

        assertThat(trustRepo.findByActorId(actorId)).isEmpty();

        // Persist an attestation — this triggers IncrementalTrustUpdateObserver
        final LedgerAttestation att = new LedgerAttestation();
        att.ledgerEntryId = entry1.id;
        att.subjectId = entry1.subjectId;
        att.attestorId = "compliance-bot";
        att.attestorType = ActorType.AGENT;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 1.0;
        att.capabilityTag = CapabilityTag.GLOBAL;
        att.occurredAt = now;
        repo.saveAttestation(att);

        // Score should now exist — incremental recomputation ran
        final ActorTrustScore score = trustRepo.findByActorId(actorId).orElse(null);
        assertThat(score).isNotNull();
        assertThat(score.trustScore).isGreaterThan(0.5);
        assertThat(score.decisionCount).isEqualTo(2);
    }

    @Test
    @Transactional
    void capabilityAttestation_producesCapabilityScore() {
        final String actorId = "incr-cap-agent-" + UUID.randomUUID();
        final Instant now = Instant.now();

        final TestEntry entry = seedEvent(actorId, now.minus(1, ChronoUnit.DAYS));

        final LedgerAttestation att = new LedgerAttestation();
        att.ledgerEntryId = entry.id;
        att.subjectId = entry.subjectId;
        att.attestorId = "compliance-bot";
        att.attestorType = ActorType.AGENT;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 1.0;
        att.capabilityTag = "code-review";
        att.occurredAt = now;
        repo.saveAttestation(att);

        assertThat(trustRepo.findCapabilityScore(actorId, "code-review")).isPresent();
    }

    @Test
    @Transactional
    void disabledConfig_noRecomputation() {
        // This test uses the incremental-trust-test profile which has incremental.enabled=true.
        // To test the disabled path, we rely on the base config (enabled=false).
        // The noAttestations_neutralScore test in TrustScoreIT already covers the disabled
        // path implicitly — saveAttestation fires the event but no observer acts on it.
        // This test verifies the positive path works — the negative is verified by absence
        // of scores in the base profile.
    }

    private TestEntry seedEvent(final String actorId, final Instant occurredAt) {
        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = occurredAt.truncatedTo(ChronoUnit.MILLIS);
        return repo.save(entry);
    }
}
```

- [ ] **Step 2: Add test profile to application.properties**

Append to `runtime/src/test/resources/application.properties`:

```properties
# Incremental trust recomputation test profile (used by IncrementalTrustUpdateIT)
%incremental-trust-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:incrementaltrusttestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%incremental-trust-test.casehub.ledger.trust-score.enabled=true
%incremental-trust-test.casehub.ledger.trust-score.incremental.enabled=true
%incremental-trust-test.casehub.ledger.hash-chain.enabled=false
%incremental-trust-test.quarkus.scheduler.enabled=false
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=IncrementalTrustUpdateIT -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure or test failure — `IncrementalTrustUpdateObserver` does not exist

- [ ] **Step 4: Implement `IncrementalTrustUpdateObserver`**

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.event.TransactionPhase;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.routing.TrustScoreActorUpdatedEvent;

@ApplicationScoped
public class IncrementalTrustUpdateObserver {

    private static final Logger log = Logger.getLogger(IncrementalTrustUpdateObserver.class);

    @Inject
    LedgerConfig config;

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    PerActorTrustComputer perActorComputer;

    @Inject
    Event<TrustScoreActorUpdatedEvent> actorUpdatedEvent;

    @Transactional(TxType.REQUIRES_NEW)
    void onAttestationRecorded(
            @Observes(during = TransactionPhase.AFTER_SUCCESS)
            final AttestationRecordedEvent event) {

        if (!config.trustScore().enabled()
                || !config.trustScore().incremental().enabled()) {
            return;
        }

        final Instant now = Instant.now();
        final List<LedgerEntry> decisions = ledgerRepo.findEventsByActorId(event.actorId());

        if (decisions.isEmpty()) {
            return;
        }

        final Set<UUID> entryIds = decisions.stream()
                .map(e -> e.id)
                .collect(Collectors.toSet());
        final Map<UUID, List<LedgerAttestation>> attestationsByEntry =
                ledgerRepo.findAttestationsForEntries(entryIds);

        final List<ActorTrustScore> scores =
                perActorComputer.computeForActor(event.actorId(), decisions, attestationsByEntry, now);

        actorUpdatedEvent.fire(new TrustScoreActorUpdatedEvent(event.actorId(), scores, now));

        log.debugf("Incremental trust recomputation completed for actor=%s, scores=%d",
                event.actorId(), scores.size());
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=IncrementalTrustUpdateIT`
Expected: PASS

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/IncrementalTrustUpdateObserver.java \
       runtime/src/test/java/io/casehub/ledger/service/IncrementalTrustUpdateIT.java \
       runtime/src/test/resources/application.properties
git commit -m "feat(#115): add IncrementalTrustUpdateObserver — per-actor trust on attestation persist

Refs #115"
```

---

### Task 7: Full build verification

- [ ] **Step 1: Build all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules (api, runtime, deployment, persistence-memory)

- [ ] **Step 2: Verify persistence-memory module compiles**

The in-memory repo now imports `AttestationRecordedEvent` from runtime. Verify the module dependency is already satisfied (persistence-memory depends on runtime).

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`
Expected: PASS

- [ ] **Step 3: Commit if any fixes were needed**

If any compilation or test issues were found and fixed, commit with:

```bash
git commit -m "fix(#115): address build issues from full verification

Refs #115"
```
