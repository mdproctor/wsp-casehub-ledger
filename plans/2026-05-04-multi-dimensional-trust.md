# Multi-Dimensional Trust Infrastructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement dimension-labelled continuous trust scores (#62) — `trustDimension` / `dimensionScore` on attestations, a weighted-average dimension pass in `TrustScoreJob`, and query methods on `TrustGateService`.

**Architecture:** Add two nullable columns (`trust_dimension`, `dimension_score`) to `ledger_attestation` (V1000 in-place). `TrustScoreJob` gains a dimension pass after the capability pass: groups attestations by `(actorId, trustDimension)`, computes a decay-weighted average of `dimensionScore`, and upserts `DIMENSION` rows into `actor_trust_score` using `scope_key = dimensionName`. The `DIMENSION` ScoreType and discriminator infrastructure already exist. `TrustGateService` gets two new query methods.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, H2 (test), PostgreSQL (prod), JUnit 5, AssertJ. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`.

---

## File Map

| File | Action | Purpose |
|---|---|---|
| `runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql` | Modify in-place | Add `trust_dimension`, `dimension_score` columns + index to `ledger_attestation` |
| `api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java` | Modify | Add `trustDimension` and `dimensionScore` fields |
| `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java` | Modify | Add `computeDimensionScore(List, Instant)` → `OptionalDouble` |
| `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java` | Modify | Add `dimensionScores(actorId)` and `dimensionScore(actorId, dimension)` |
| `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java` | Modify | Add dimension pass after capability pass |
| `runtime/src/test/java/io/casehub/ledger/service/LedgerTestFixtures.java` | Modify | Add `seedDecisionWithDimension(...)` helper |
| `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java` | Modify | Extend with `computeDimensionScore` unit tests |
| `runtime/src/test/java/io/casehub/ledger/service/TrustGateServiceTest.java` | Modify | Extend with dimension query unit tests |
| `runtime/src/test/java/io/casehub/ledger/service/ActorTrustScoreRepositoryIT.java` | Modify | Extend with DIMENSION row integration tests |
| `runtime/src/test/java/io/casehub/ledger/service/LedgerAttestationDimensionIT.java` | Create | Integration tests for `trust_dimension`/`dimension_score` persistence |
| `runtime/src/test/java/io/casehub/ledger/service/TrustScoreDimensionIT.java` | Create | End-to-end integration tests for dimension scoring pipeline |
| `docs/DESIGN.md` | Modify | Add dimension-scoped trust scores section |
| `CLAUDE.md` | Modify | Update project structure table for new methods |

---

## Task 1: Schema — add dimension columns to `ledger_attestation` (V1000 in-place)

**Files:**
- Modify: `runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql`

- [ ] **Step 1: Edit V1000 in-place — add two nullable columns and an index after the `capability_tag` line**

In `V1000__ledger_base_schema.sql`, replace the `ledger_attestation` table definition:

```sql
CREATE TABLE ledger_attestation (
    id               UUID            NOT NULL,
    ledger_entry_id  UUID            NOT NULL,
    subject_id       UUID            NOT NULL,
    attestor_id      VARCHAR(255)    NOT NULL,
    attestor_type    VARCHAR(20)     NOT NULL,
    attestor_role    VARCHAR(100),
    verdict          VARCHAR(20)     NOT NULL,
    evidence         TEXT,
    confidence       DOUBLE PRECISION NOT NULL DEFAULT 1.0,
    capability_tag   VARCHAR(255)     NOT NULL DEFAULT '*',
    trust_dimension  VARCHAR(255),
    dimension_score  DOUBLE PRECISION,
    occurred_at      TIMESTAMP       NOT NULL,
    CONSTRAINT pk_ledger_attestation PRIMARY KEY (id),
    CONSTRAINT fk_attestation_entry FOREIGN KEY (ledger_entry_id) REFERENCES ledger_entry (id)
);
```

And add one more index after the existing attestation indexes:

```sql
CREATE INDEX idx_ledger_attestation_dimension ON ledger_attestation (attestor_id, trust_dimension);
```

- [ ] **Step 2: Verify the migration file compiles (build)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl runtime -Dtest=PlainEntityTest -q
```

Expected: BUILD SUCCESS. H2 re-creates the schema from scratch on each test run.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql
git commit -m "feat(#62): add trust_dimension and dimension_score columns to ledger_attestation

Closes part of #62. Refs #50"
```

---

## Task 2: API model — `trustDimension` and `dimensionScore` on `LedgerAttestation`

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java`

- [ ] **Step 1: Add two nullable fields after `capabilityTag`**

In `api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java`, after the `capabilityTag` field:

```java
    @Column(name = "trust_dimension")
    public String trustDimension;

    /**
     * Continuous quality score in [0.0, 1.0]. Only meaningful when {@code trustDimension} is
     * set. Null on ordinary (binary verdict) attestations.
     */
    @Column(name = "dimension_score")
    public Double dimensionScore;
```

Update the Javadoc on the class to mention the new fields:

```java
/**
 * A peer attestation stamped onto a {@link LedgerEntry}.
 *
 * <p>Carries either a binary verdict ({@code verdict}) for trust scoring, or a continuous
 * quality score ({@code dimensionScore} ∈ [0.0, 1.0]) for dimension-labelled scoring when
 * {@code trustDimension} is set. Both fields may be populated together.
 */
```

- [ ] **Step 2: Build to verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl api -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Run all tests to confirm nothing is broken**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime -q
```

Expected: BUILD SUCCESS, all existing tests pass.

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java
git commit -m "feat(#62): add trustDimension and dimensionScore fields to LedgerAttestation

Refs #62 #50"
```

---

## Task 3: TDD — `TrustScoreComputer.computeDimensionScore()`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java`

- [ ] **Step 1: Write the failing tests — add to `TrustScoreComputerTest.java`**

Add these tests to the existing `TrustScoreComputerTest` class (after the existing tests). The fixture `attestation(entryId, verdict, confidence)` helper is already in the file; add a new one for dimension attestations:

```java
    // ── Dimension score fixture ───────────────────────────────────────────────

    private LedgerAttestation dimensionAttestation(final UUID entryId, final String dimension,
            final double dimensionScore) {
        return dimensionAttestation(entryId, dimension, dimensionScore, now);
    }

    private LedgerAttestation dimensionAttestation(final UUID entryId, final String dimension,
            final double dimensionScore, final Instant occurredAt) {
        final LedgerAttestation a = new LedgerAttestation();
        a.id = UUID.randomUUID();
        a.ledgerEntryId = entryId;
        a.attestorId = "peer";
        a.attestorType = ActorType.HUMAN;
        a.verdict = AttestationVerdict.SOUND;
        a.confidence = 1.0;
        a.trustDimension = dimension;
        a.dimensionScore = dimensionScore;
        a.occurredAt = occurredAt;
        return a;
    }

    // ── computeDimensionScore — happy path ────────────────────────────────────

    @Test
    void computeDimensionScore_singleAttestation_returnsDimensionScore() {
        final TestLedgerEntry d = decision("actor", now.minus(1, ChronoUnit.DAYS));
        final LedgerAttestation a = dimensionAttestation(d.id, "thoroughness", 0.8);

        final OptionalDouble result = computer.computeDimensionScore(List.of(a), now);

        assertThat(result).isPresent();
        // single attestation — score equals dimensionScore (decay weight applied uniformly)
        assertThat(result.getAsDouble()).isCloseTo(0.8, within(0.001));
    }

    @Test
    void computeDimensionScore_twoAttestations_returnsDecayWeightedAverage() {
        // Both at age 0 → equal weight → simple mean = (0.6 + 0.9) / 2 = 0.75
        final TestLedgerEntry d = decision("actor", now.minus(1, ChronoUnit.DAYS));
        final LedgerAttestation a1 = dimensionAttestation(d.id, "thoroughness", 0.6);
        final LedgerAttestation a2 = dimensionAttestation(d.id, "thoroughness", 0.9);

        final OptionalDouble result = computer.computeDimensionScore(List.of(a1, a2), now);

        assertThat(result).isPresent();
        assertThat(result.getAsDouble()).isCloseTo(0.75, within(0.01));
    }

    // ── computeDimensionScore — correctness ───────────────────────────────────

    @Test
    void computeDimensionScore_olderAttestationHasLowerWeight() {
        final TestLedgerEntry d = decision("actor", now.minus(10, ChronoUnit.DAYS));
        // recent: high score → should pull mean up
        final LedgerAttestation recent = dimensionAttestation(d.id, "thoroughness", 1.0, now);
        // old: low score → decays out, pulls mean down less
        final LedgerAttestation old = dimensionAttestation(d.id, "thoroughness", 0.0,
                now.minus(180, ChronoUnit.DAYS));

        final OptionalDouble result = computer.computeDimensionScore(List.of(recent, old), now);

        assertThat(result).isPresent();
        // recent (score=1.0) carries more weight than old (score=0.0), so result > 0.5
        assertThat(result.getAsDouble()).isGreaterThan(0.5);
    }

    @Test
    void computeDimensionScore_nullDimensionScore_excluded() {
        final TestLedgerEntry d = decision("actor", now);
        final LedgerAttestation withScore = dimensionAttestation(d.id, "thoroughness", 0.7);
        // null dimensionScore — should be excluded from average
        final LedgerAttestation noScore = new LedgerAttestation();
        noScore.id = UUID.randomUUID();
        noScore.ledgerEntryId = d.id;
        noScore.attestorId = "peer";
        noScore.attestorType = ActorType.HUMAN;
        noScore.verdict = AttestationVerdict.SOUND;
        noScore.confidence = 1.0;
        noScore.trustDimension = "thoroughness";
        noScore.dimensionScore = null;
        noScore.occurredAt = now;

        final OptionalDouble result = computer.computeDimensionScore(List.of(withScore, noScore), now);

        assertThat(result).isPresent();
        assertThat(result.getAsDouble()).isCloseTo(0.7, within(0.001));
    }

    // ── computeDimensionScore — robustness ────────────────────────────────────

    @Test
    void computeDimensionScore_emptyList_returnsEmpty() {
        final OptionalDouble result = computer.computeDimensionScore(List.of(), now);

        assertThat(result).isEmpty();
    }

    @Test
    void computeDimensionScore_allNullDimensionScores_returnsEmpty() {
        final TestLedgerEntry d = decision("actor", now);
        final LedgerAttestation a = new LedgerAttestation();
        a.id = UUID.randomUUID();
        a.ledgerEntryId = d.id;
        a.attestorId = "peer";
        a.attestorType = ActorType.HUMAN;
        a.verdict = AttestationVerdict.SOUND;
        a.confidence = 1.0;
        a.trustDimension = "thoroughness";
        a.dimensionScore = null;
        a.occurredAt = now;

        final OptionalDouble result = computer.computeDimensionScore(List.of(a), now);

        assertThat(result).isEmpty();
    }

    @Test
    void computeDimensionScore_clampedToUnitInterval() {
        // Pathological case: dimensionScore above 1.0 (guard against bad input)
        final TestLedgerEntry d = decision("actor", now);
        final LedgerAttestation a = dimensionAttestation(d.id, "thoroughness", 2.0);

        final OptionalDouble result = computer.computeDimensionScore(List.of(a), now);

        assertThat(result).isPresent();
        assertThat(result.getAsDouble()).isLessThanOrEqualTo(1.0);
    }
```

Also add the import at the top of the file:
```java
import java.util.OptionalDouble;
```

- [ ] **Step 2: Run tests — confirm they fail with "cannot find symbol: computeDimensionScore"**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreComputerTest -q 2>&1 | tail -5
```

Expected: COMPILATION ERROR — `computeDimensionScore` not found.

- [ ] **Step 3: Implement `computeDimensionScore` in `TrustScoreComputer.java`**

Add after the existing `compute()` method:

```java
    /**
     * Computes a decay-weighted average of continuous quality dimension scores.
     *
     * <p>
     * Unlike the Bayesian Beta model used by {@link #compute}, dimension scores are continuous
     * in [0.0, 1.0]. This method computes {@code Σ(weight_i × dimensionScore_i) / Σ(weight_i)}
     * where weight decays purely with age (using {@link AttestationVerdict#SOUND} to suppress
     * the FLAGGED/CHALLENGED valence asymmetry — continuous scores have no verdict polarity).
     *
     * <p>
     * Attestations with {@code null} {@code dimensionScore} are excluded.
     *
     * @param dimensionAttestations attestations for one (actor, trustDimension) pair
     * @param now reference timestamp for age calculation
     * @return weighted average in [0.0, 1.0], or empty if no valid attestations exist
     */
    public OptionalDouble computeDimensionScore(
            final List<LedgerAttestation> dimensionAttestations,
            final Instant now) {
        double weightedSum = 0.0;
        double totalWeight = 0.0;

        for (final LedgerAttestation a : dimensionAttestations) {
            if (a.dimensionScore == null) {
                continue;
            }
            final Instant attestedAt = a.occurredAt != null ? a.occurredAt : now;
            final long ageInDays = Math.max(0, java.time.Duration.between(attestedAt, now).toDays());
            final double weight = decayFunction.weight(ageInDays, AttestationVerdict.SOUND);
            weightedSum += weight * a.dimensionScore;
            totalWeight += weight;
        }

        if (totalWeight == 0.0) {
            return OptionalDouble.empty();
        }
        return OptionalDouble.of(Math.max(0.0, Math.min(1.0, weightedSum / totalWeight)));
    }
```

Add the import at the top of `TrustScoreComputer.java`:
```java
import java.util.OptionalDouble;
```

- [ ] **Step 4: Run tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreComputerTest -q
```

Expected: BUILD SUCCESS, all `TrustScoreComputerTest` tests pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreComputer.java \
        runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerTest.java
git commit -m "feat(#62): add TrustScoreComputer.computeDimensionScore() — decay-weighted average

Refs #62 #50"
```

---

## Task 4: TDD — `TrustGateService` dimension query methods

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustGateServiceTest.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java`

- [ ] **Step 1: Extend `TrustGateServiceTest` — add dimension tests**

Add a factory method and tests to `TrustGateServiceTest`. The `StubRepository` already returns `List.of()` from `findByActorIdAndScoreType` — extend it to support DIMENSION rows:

```java
    // ── Dimension query fixtures ──────────────────────────────────────────────

    private static ActorTrustScoreRepository repoWithDimensions(
            final String actorId,
            final Map<String, Double> dimensionScores) {
        return new StubRepository(null) {
            @Override
            public List<ActorTrustScore> findByActorIdAndScoreType(
                    final String id, final ScoreType type) {
                if (!actorId.equals(id) || type != ScoreType.DIMENSION) {
                    return List.of();
                }
                return dimensionScores.entrySet().stream().map(e -> {
                    final ActorTrustScore s = new ActorTrustScore();
                    s.id = UUID.randomUUID();
                    s.actorId = actorId;
                    s.scoreType = ScoreType.DIMENSION;
                    s.scopeKey = e.getKey();
                    s.actorType = ActorType.AGENT;
                    s.trustScore = e.getValue();
                    s.lastComputedAt = Instant.now();
                    return s;
                }).collect(java.util.stream.Collectors.toList());
            }

            @Override
            public Optional<ActorTrustScore> findByActorIdAndTypeAndKey(
                    final String id, final ScoreType type, final String scopeKey) {
                if (!actorId.equals(id) || type != ScoreType.DIMENSION) {
                    return Optional.empty();
                }
                final Double score = dimensionScores.get(scopeKey);
                if (score == null) {
                    return Optional.empty();
                }
                final ActorTrustScore s = new ActorTrustScore();
                s.id = UUID.randomUUID();
                s.actorId = actorId;
                s.scoreType = ScoreType.DIMENSION;
                s.scopeKey = scopeKey;
                s.actorType = ActorType.AGENT;
                s.trustScore = score;
                s.lastComputedAt = Instant.now();
                return Optional.of(s);
            }
        };
    }

    // ── dimensionScores ───────────────────────────────────────────────────────

    @Test
    void dimensionScores_returnsAllDimensionScores() {
        final TrustGateService gate = new TrustGateService(
                repoWithDimensions("actor-d",
                        Map.of("thoroughness", 0.8, "false-positive-rate", 0.2)));

        final Map<String, Double> scores = gate.dimensionScores("actor-d");

        assertThat(scores).hasSize(2);
        assertThat(scores.get("thoroughness")).isEqualTo(0.8);
        assertThat(scores.get("false-positive-rate")).isEqualTo(0.2);
    }

    @Test
    void dimensionScores_returnsEmptyMap_whenNoDimensionRows() {
        final TrustGateService gate = new TrustGateService(emptyRepo());

        assertThat(gate.dimensionScores("ghost")).isEmpty();
    }

    // ── dimensionScore ────────────────────────────────────────────────────────

    @Test
    void dimensionScore_returnsScore_whenDimensionExists() {
        final TrustGateService gate = new TrustGateService(
                repoWithDimensions("actor-e", Map.of("thoroughness", 0.75)));

        assertThat(gate.dimensionScore("actor-e", "thoroughness")).isPresent();
        assertThat(gate.dimensionScore("actor-e", "thoroughness").get()).isEqualTo(0.75);
    }

    @Test
    void dimensionScore_returnsEmpty_whenDimensionNotFound() {
        final TrustGateService gate = new TrustGateService(
                repoWithDimensions("actor-f", Map.of("thoroughness", 0.75)));

        assertThat(gate.dimensionScore("actor-f", "false-positive-rate")).isEmpty();
    }

    @Test
    void dimensionScore_returnsEmpty_whenActorUnknown() {
        final TrustGateService gate = new TrustGateService(emptyRepo());

        assertThat(gate.dimensionScore("ghost", "thoroughness")).isEmpty();
    }
```

Add the import at the top of `TrustGateServiceTest.java`:
```java
import java.util.Map;
import java.util.stream.Collectors;
```

- [ ] **Step 2: Run tests — confirm they fail with "cannot find symbol: dimensionScores"**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustGateServiceTest -q 2>&1 | tail -5
```

Expected: COMPILATION ERROR — `dimensionScores` and `dimensionScore` not found.

- [ ] **Step 3: Implement in `TrustGateService.java`**

Add two methods after `currentScore(actorId, capabilityTag)`:

```java
    /**
     * Returns all dimension trust scores for the given actor, keyed by dimension name.
     * Returns an empty map if no dimension scores have been computed yet.
     *
     * @param actorId the actor identity string
     * @return dimension name → score in [0.0, 1.0] for each DIMENSION row
     */
    public Map<String, Double> dimensionScores(final String actorId) {
        return repository.findByActorIdAndScoreType(actorId, ScoreType.DIMENSION).stream()
                .collect(java.util.stream.Collectors.toMap(s -> s.scopeKey, s -> s.trustScore));
    }

    /**
     * Returns the trust score for a specific dimension, or empty if not yet computed.
     *
     * @param actorId the actor identity string
     * @param dimension the dimension name (e.g. {@code "review-thoroughness"})
     * @return the dimension score in [0.0, 1.0], or empty
     */
    public Optional<Double> dimensionScore(final String actorId, final String dimension) {
        return repository.findByActorIdAndTypeAndKey(actorId, ScoreType.DIMENSION, dimension)
                .map(s -> s.trustScore);
    }
```

Add the import at the top of `TrustGateService.java`:
```java
import java.util.Map;
```

- [ ] **Step 4: Run tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustGateServiceTest -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java \
        runtime/src/test/java/io/casehub/ledger/service/TrustGateServiceTest.java
git commit -m "feat(#62): add TrustGateService.dimensionScores() and dimensionScore() query methods

Refs #62 #50"
```

---

## Task 5: Test fixture — `LedgerTestFixtures.seedDecisionWithDimension()`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerTestFixtures.java`

- [ ] **Step 1: Add overload to `LedgerTestFixtures`**

Add after the existing capability overload of `seedDecision`:

```java
    /**
     * Persist a {@link TestEntry} EVENT with a dimension-tagged attestation.
     *
     * <p>Persists the attestation directly via {@link jakarta.persistence.EntityManager}.
     * {@code verdict} is set to {@link io.casehub.ledger.api.model.AttestationVerdict#SOUND}
     * for all dimension attestations — the quality measurement is carried by
     * {@code dimensionScore}, not the verdict.
     *
     * @param actorId        the actor who made the decision
     * @param decisionTime   when the decision occurred
     * @param trustDimension the dimension label (e.g. "review-thoroughness")
     * @param dimensionScore continuous quality score in [0.0, 1.0]
     * @param capabilityTag  the capability tag (or {@link io.casehub.ledger.api.model.CapabilityTag#GLOBAL})
     * @param repo           entry repository for persisting the EVENT entry
     * @param em             entity manager for direct attestation persist
     */
    public static TestEntry seedDecisionWithDimension(final String actorId, final Instant decisionTime,
            final String trustDimension, final double dimensionScore,
            final String capabilityTag,
            final LedgerEntryRepository repo, final EntityManager em) {

        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Reviewer";
        entry.occurredAt = decisionTime.truncatedTo(java.time.temporal.ChronoUnit.MILLIS);
        repo.save(entry);

        final LedgerAttestation att = new LedgerAttestation();
        att.id = UUID.randomUUID();
        att.ledgerEntryId = entry.id;
        att.subjectId = entry.subjectId;
        att.attestorId = "dimension-assessor";
        att.attestorType = ActorType.AGENT;
        att.verdict = io.casehub.ledger.api.model.AttestationVerdict.SOUND;
        att.confidence = 1.0;
        att.capabilityTag = capabilityTag;
        att.trustDimension = trustDimension;
        att.dimensionScore = dimensionScore;
        att.occurredAt = decisionTime.plusSeconds(60).truncatedTo(java.time.temporal.ChronoUnit.MILLIS);
        em.persist(att);

        return entry;
    }
```

- [ ] **Step 2: Build — confirm compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/LedgerTestFixtures.java
git commit -m "test(#62): add LedgerTestFixtures.seedDecisionWithDimension() helper

Refs #62 #50"
```

---

## Task 6: Integration test — `LedgerAttestationDimensionIT`

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerAttestationDimensionIT.java`

- [ ] **Step 1: Write the integration test**

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

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;

/**
 * Integration tests for dimension-tagged attestation persistence (#62).
 *
 * <p>Covers: happy path, correctness/isolation, and backward compatibility.
 */
@QuarkusTest
class LedgerAttestationDimensionIT {

    @Inject
    LedgerEntryRepository repo;

    @Inject
    EntityManager em;

    // ── Happy path: trustDimension and dimensionScore stored and retrieved ────

    @Test
    @Transactional
    void dimensionAttestation_storedAndRetrieved() {
        final TestEntry entry = savedEntry();
        final LedgerAttestation att = dimensionAttestation(entry.id, entry.subjectId,
                "review-thoroughness", 0.75);
        em.persist(att);
        em.flush();
        em.clear();

        final List<LedgerAttestation> results = repo.findAttestationsByEntryId(entry.id);

        assertThat(results).hasSize(1);
        assertThat(results.get(0).trustDimension).isEqualTo("review-thoroughness");
        assertThat(results.get(0).dimensionScore).isEqualTo(0.75);
    }

    // ── Happy path: dimensionScore = 0.0 is stored (not treated as null) ─────

    @Test
    @Transactional
    void dimensionAttestation_zeroScore_stored() {
        final TestEntry entry = savedEntry();
        final LedgerAttestation att = dimensionAttestation(entry.id, entry.subjectId,
                "false-positive-rate", 0.0);
        em.persist(att);
        em.flush();
        em.clear();

        final List<LedgerAttestation> results = repo.findAttestationsByEntryId(entry.id);

        assertThat(results.get(0).dimensionScore).isEqualTo(0.0);
    }

    // ── Correctness: ordinary attestation has null trustDimension/dimensionScore

    @Test
    @Transactional
    void ordinaryAttestation_nullDimensionFields() {
        final TestEntry entry = savedEntry();
        final LedgerAttestation att = new LedgerAttestation();
        att.id = UUID.randomUUID();
        att.ledgerEntryId = entry.id;
        att.subjectId = entry.subjectId;
        att.attestorId = "peer";
        att.attestorType = ActorType.AGENT;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 1.0;
        att.capabilityTag = CapabilityTag.GLOBAL;
        att.occurredAt = Instant.now();
        em.persist(att);
        em.flush();
        em.clear();

        final List<LedgerAttestation> results = repo.findAttestationsByEntryId(entry.id);

        assertThat(results).hasSize(1);
        assertThat(results.get(0).trustDimension).isNull();
        assertThat(results.get(0).dimensionScore).isNull();
    }

    // ── Correctness: dimension and ordinary attestations coexist on same entry

    @Test
    @Transactional
    void dimensionAndOrdinaryAttestations_coexistOnSameEntry() {
        final TestEntry entry = savedEntry();

        final LedgerAttestation ordinary = new LedgerAttestation();
        ordinary.id = UUID.randomUUID();
        ordinary.ledgerEntryId = entry.id;
        ordinary.subjectId = entry.subjectId;
        ordinary.attestorId = "peer";
        ordinary.attestorType = ActorType.AGENT;
        ordinary.verdict = AttestationVerdict.SOUND;
        ordinary.confidence = 1.0;
        ordinary.capabilityTag = CapabilityTag.GLOBAL;
        ordinary.occurredAt = Instant.now();
        em.persist(ordinary);

        em.persist(dimensionAttestation(entry.id, entry.subjectId, "thoroughness", 0.8));
        em.flush();
        em.clear();

        final List<LedgerAttestation> results = repo.findAttestationsByEntryId(entry.id);

        assertThat(results).hasSize(2);
        final long withDimension = results.stream().filter(a -> a.trustDimension != null).count();
        final long withoutDimension = results.stream().filter(a -> a.trustDimension == null).count();
        assertThat(withDimension).isEqualTo(1);
        assertThat(withoutDimension).isEqualTo(1);
    }

    // ── Robustness: multiple dimension attestations on same entry ─────────────

    @Test
    @Transactional
    void multipleDimensionAttestations_sameEntry_storedSeparately() {
        final TestEntry entry = savedEntry();
        em.persist(dimensionAttestation(entry.id, entry.subjectId, "thoroughness", 0.9));
        em.persist(dimensionAttestation(entry.id, entry.subjectId, "false-positive-rate", 0.1));
        em.flush();
        em.clear();

        final List<LedgerAttestation> results = repo.findAttestationsByEntryId(entry.id);

        assertThat(results).hasSize(2);
        assertThat(results).extracting(a -> a.trustDimension)
                .containsExactlyInAnyOrder("thoroughness", "false-positive-rate");
    }

    // ── Fixtures ─────────────────────────────────────────────────────────────

    private TestEntry savedEntry() {
        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = "reviewer-" + UUID.randomUUID();
        entry.actorType = ActorType.AGENT;
        entry.occurredAt = Instant.now().minus(1, ChronoUnit.HOURS);
        repo.save(entry);
        return entry;
    }

    private LedgerAttestation dimensionAttestation(final UUID entryId, final UUID subjectId,
            final String dimension, final double score) {
        final LedgerAttestation att = new LedgerAttestation();
        att.id = UUID.randomUUID();
        att.ledgerEntryId = entryId;
        att.subjectId = subjectId;
        att.attestorId = "dimension-peer";
        att.attestorType = ActorType.AGENT;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 1.0;
        att.capabilityTag = CapabilityTag.GLOBAL;
        att.trustDimension = dimension;
        att.dimensionScore = score;
        att.occurredAt = Instant.now();
        return att;
    }
}
```

- [ ] **Step 2: Run the test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerAttestationDimensionIT -q
```

Expected: BUILD SUCCESS, all 5 tests pass.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/LedgerAttestationDimensionIT.java
git commit -m "test(#62): LedgerAttestationDimensionIT — dimension attestation persistence

Refs #62 #50"
```

---

## Task 7: Repository IT — extend `ActorTrustScoreRepositoryIT` with DIMENSION tests

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/ActorTrustScoreRepositoryIT.java`

- [ ] **Step 1: Add DIMENSION tests at the end of `ActorTrustScoreRepositoryIT`**

```java
    // ── DIMENSION rows ────────────────────────────────────────────────────────

    @Test
    @Transactional
    void upsert_dimension_storesDimensionRow() {
        final String actorId = "actor-dim-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.DIMENSION, "thoroughness", ActorType.AGENT,
                0.78, 5, 0, 0.0, 0.0, 4, 1, Instant.now());

        final var result = repo.findByActorIdAndTypeAndKey(actorId, ScoreType.DIMENSION, "thoroughness");

        assertThat(result).isPresent();
        assertThat(result.get().trustScore).isEqualTo(0.78);
        assertThat(result.get().scoreType).isEqualTo(ScoreType.DIMENSION);
        assertThat(result.get().scopeKey).isEqualTo("thoroughness");
    }

    @Test
    @Transactional
    void upsert_dimension_isIdempotent() {
        final String actorId = "actor-dim-idem-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.DIMENSION, "thoroughness", ActorType.AGENT,
                0.5, 2, 0, 0.0, 0.0, 2, 0, Instant.now());
        repo.upsert(actorId, ScoreType.DIMENSION, "thoroughness", ActorType.AGENT,
                0.9, 10, 0, 0.0, 0.0, 10, 0, Instant.now());

        final var results = repo.findByActorIdAndScoreType(actorId, ScoreType.DIMENSION);
        assertThat(results).hasSize(1);
        assertThat(results.get(0).trustScore).isEqualTo(0.9);
    }

    @Test
    @Transactional
    void findByActorIdAndScoreType_dimension_returnsAllDimensionRows() {
        final String actorId = "actor-dim-multi-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.DIMENSION, "thoroughness", ActorType.AGENT,
                0.8, 5, 0, 0.0, 0.0, 5, 0, Instant.now());
        repo.upsert(actorId, ScoreType.DIMENSION, "false-positive-rate", ActorType.AGENT,
                0.1, 3, 2, 0.0, 0.0, 1, 2, Instant.now());

        final var results = repo.findByActorIdAndScoreType(actorId, ScoreType.DIMENSION);

        assertThat(results).hasSize(2);
        assertThat(results).extracting(s -> s.scopeKey)
                .containsExactlyInAnyOrder("thoroughness", "false-positive-rate");
    }

    @Test
    @Transactional
    void findByActorId_global_notAffectedByDimensionRows() {
        final String actorId = "actor-dim-isolation-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.GLOBAL, null, ActorType.AGENT,
                0.7, 5, 0, 2.5, 1.0, 5, 0, Instant.now());
        repo.upsert(actorId, ScoreType.DIMENSION, "thoroughness", ActorType.AGENT,
                0.9, 3, 0, 0.0, 0.0, 3, 0, Instant.now());

        // findByActorId must return only the GLOBAL row
        final var global = repo.findByActorId(actorId);
        assertThat(global).isPresent();
        assertThat(global.get().scoreType).isEqualTo(ScoreType.GLOBAL);
        assertThat(global.get().trustScore).isEqualTo(0.7);
    }

    @Test
    @Transactional
    void findByActorIdAndTypeAndKey_dimension_wrongKey_returnsEmpty() {
        final String actorId = "actor-dim-wrongkey-" + System.nanoTime();
        repo.upsert(actorId, ScoreType.DIMENSION, "thoroughness", ActorType.AGENT,
                0.8, 5, 0, 0.0, 0.0, 5, 0, Instant.now());

        assertThat(repo.findByActorIdAndTypeAndKey(actorId, ScoreType.DIMENSION, "false-positive-rate"))
                .isEmpty();
    }
```

- [ ] **Step 2: Run the repository IT**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=ActorTrustScoreRepositoryIT -q
```

Expected: BUILD SUCCESS, all tests pass (existing + new DIMENSION tests).

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/ActorTrustScoreRepositoryIT.java
git commit -m "test(#62): extend ActorTrustScoreRepositoryIT with DIMENSION row tests

Refs #62 #50"
```

---

## Task 8: TDD — `TrustScoreJob` dimension pass + end-to-end `TrustScoreDimensionIT`

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreDimensionIT.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`

- [ ] **Step 1: Write `TrustScoreDimensionIT` — the failing end-to-end test**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * End-to-end integration tests for multi-dimensional trust scoring (#62).
 *
 * <p>Covers: happy path, correctness/isolation, robustness, and backward compatibility.
 */
@QuarkusTest
@TestProfile(TrustScoreIT.TrustScoreTestProfile.class)
class TrustScoreDimensionIT {

    @Inject
    TrustScoreJob trustScoreJob;

    @Inject
    TrustGateService trustGateService;

    @Inject
    LedgerEntryRepository repo;

    @Inject
    ActorTrustScoreRepository trustRepo;

    @Inject
    EntityManager em;

    // ── Happy path: dimension attestations produce DIMENSION rows ─────────────

    @Test
    @Transactional
    void dimensionAttestations_producesDimensionRows() {
        final String actorId = "agent-dim-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "review-thoroughness", 0.9);
        seedDimension(actorId, now.minus(2, ChronoUnit.DAYS), "review-thoroughness", 0.7);
        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "false-positive-rate", 0.1);

        trustScoreJob.runComputation();

        final List<ActorTrustScore> dimScores = trustRepo.findByActorIdAndScoreType(actorId, ScoreType.DIMENSION);
        assertThat(dimScores).hasSize(2);

        final ActorTrustScore thoroughness = dimScores.stream()
                .filter(s -> "review-thoroughness".equals(s.scopeKey)).findFirst().orElseThrow();
        assertThat(thoroughness.trustScore).isGreaterThan(0.7);
        assertThat(thoroughness.trustScore).isLessThan(1.0);

        final ActorTrustScore fpr = dimScores.stream()
                .filter(s -> "false-positive-rate".equals(s.scopeKey)).findFirst().orElseThrow();
        assertThat(fpr.trustScore).isCloseTo(0.1, within(0.05));
    }

    // ── Happy path: TrustGateService.dimensionScores() returns all dimensions ─

    @Test
    @Transactional
    void trustGateService_dimensionScores_returnsAll() {
        final String actorId = "agent-dimmap-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "thoroughness", 0.8);
        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "accuracy", 0.6);

        trustScoreJob.runComputation();

        final Map<String, Double> scores = trustGateService.dimensionScores(actorId);
        assertThat(scores).containsKey("thoroughness");
        assertThat(scores).containsKey("accuracy");
        assertThat(scores.get("thoroughness")).isCloseTo(0.8, within(0.05));
        assertThat(scores.get("accuracy")).isCloseTo(0.6, within(0.05));
    }

    // ── Happy path: TrustGateService.dimensionScore() returns specific dimension

    @Test
    @Transactional
    void trustGateService_dimensionScore_returnsSpecificDimension() {
        final String actorId = "agent-dimone-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "thoroughness", 0.85);

        trustScoreJob.runComputation();

        assertThat(trustGateService.dimensionScore(actorId, "thoroughness")).isPresent();
        assertThat(trustGateService.dimensionScore(actorId, "thoroughness").get())
                .isCloseTo(0.85, within(0.05));
    }

    // ── Correctness: decay — older attestation has lower weight ───────────────

    @Test
    @Transactional
    void dimensionScore_olderAttestationDecays() {
        final String actorId = "agent-decay-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // recent high score + very old low score → result should be > 0.5 (recent wins)
        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "thoroughness", 1.0);
        seedDimension(actorId, now.minus(365, ChronoUnit.DAYS), "thoroughness", 0.0);

        trustScoreJob.runComputation();

        final var score = trustGateService.dimensionScore(actorId, "thoroughness");
        assertThat(score).isPresent();
        assertThat(score.get()).isGreaterThan(0.5);
    }

    // ── Correctness: separate dimensions are isolated ─────────────────────────

    @Test
    @Transactional
    void dimensionScores_isolatedFromEachOther() {
        final String actorId = "agent-iso-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "thoroughness", 0.9);
        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "false-positive-rate", 0.05);

        trustScoreJob.runComputation();

        final var thoroughness = trustGateService.dimensionScore(actorId, "thoroughness").orElseThrow();
        final var fpr = trustGateService.dimensionScore(actorId, "false-positive-rate").orElseThrow();

        assertThat(thoroughness).isGreaterThan(0.8);
        assertThat(fpr).isLessThan(0.2);
        // isolation: thoroughness computation did not contaminate fpr
        assertThat(thoroughness).isNotCloseTo(fpr, within(0.5));
    }

    // ── Correctness: GLOBAL and CAPABILITY rows unaffected ────────────────────

    @Test
    @Transactional
    void dimensionPass_doesNotAffectGlobalOrCapabilityRows() {
        final String actorId = "agent-compat-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // ordinary GLOBAL attestation
        LedgerTestFixtures.seedDecision(actorId, now.minus(1, ChronoUnit.DAYS),
                AttestationVerdict.SOUND, now.minus(1, ChronoUnit.DAYS).plusSeconds(60),
                CapabilityTag.GLOBAL, repo, em);

        // capability attestation
        LedgerTestFixtures.seedDecision(actorId, now.minus(2, ChronoUnit.DAYS),
                AttestationVerdict.SOUND, now.minus(2, ChronoUnit.DAYS).plusSeconds(60),
                "security-review", repo, em);

        // dimension attestation
        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "thoroughness", 0.7);

        trustScoreJob.runComputation();

        final var global = trustRepo.findByActorId(actorId);
        assertThat(global).isPresent();
        assertThat(global.get().scoreType).isEqualTo(ScoreType.GLOBAL);

        final var capability = trustRepo.findByActorIdAndTypeAndKey(actorId, ScoreType.CAPABILITY, "security-review");
        assertThat(capability).isPresent();

        final var dimension = trustRepo.findByActorIdAndTypeAndKey(actorId, ScoreType.DIMENSION, "thoroughness");
        assertThat(dimension).isPresent();
        assertThat(dimension.get().trustScore).isCloseTo(0.7, within(0.05));
    }

    // ── Robustness: actor with no dimension attestations gets no DIMENSION rows

    @Test
    @Transactional
    void nonemptyActor_noDimensionAttestations_noDimensionRows() {
        final String actorId = "agent-nodim-" + UUID.randomUUID();
        final Instant now = Instant.now();

        LedgerTestFixtures.seedDecision(actorId, now.minus(1, ChronoUnit.DAYS),
                AttestationVerdict.SOUND, repo, em);

        trustScoreJob.runComputation();

        assertThat(trustRepo.findByActorIdAndScoreType(actorId, ScoreType.DIMENSION)).isEmpty();
        assertThat(trustRepo.findByActorId(actorId)).isPresent(); // GLOBAL row still created
    }

    // ── Robustness: attestation with trustDimension but null dimensionScore excluded

    @Test
    @Transactional
    void dimensionAttestation_nullDimensionScore_excludedFromComputation() {
        final String actorId = "agent-nullscore-" + UUID.randomUUID();
        final Instant now = Instant.now();

        // Persist a dimension-tagged attestation with null dimensionScore
        LedgerTestFixtures.seedDecision(actorId, now.minus(1, ChronoUnit.DAYS),
                AttestationVerdict.SOUND, now.minus(1, ChronoUnit.DAYS).plusSeconds(60),
                CapabilityTag.GLOBAL, repo, em);

        // Manually persist a dimension attestation with null dimensionScore via em
        final io.casehub.ledger.runtime.model.LedgerAttestation badAtt =
                new io.casehub.ledger.runtime.model.LedgerAttestation();
        badAtt.id = UUID.randomUUID();
        // find the entry we just seeded
        final var entries = repo.findAllEvents();
        final var myEntry = entries.stream()
                .filter(e -> actorId.equals(e.actorId)).findFirst().orElseThrow();
        badAtt.ledgerEntryId = myEntry.id;
        badAtt.subjectId = myEntry.subjectId;
        badAtt.attestorId = "bad-peer";
        badAtt.attestorType = io.casehub.ledger.api.model.ActorType.AGENT;
        badAtt.verdict = AttestationVerdict.SOUND;
        badAtt.confidence = 1.0;
        badAtt.capabilityTag = CapabilityTag.GLOBAL;
        badAtt.trustDimension = "thoroughness";
        badAtt.dimensionScore = null; // explicitly null
        badAtt.occurredAt = now;
        em.persist(badAtt);

        trustScoreJob.runComputation();

        // null dimensionScore → excluded → no DIMENSION row produced
        assertThat(trustRepo.findByActorIdAndScoreType(actorId, ScoreType.DIMENSION)).isEmpty();
    }

    // ── Robustness: unknown dimension queried → empty ─────────────────────────

    @Test
    @Transactional
    void trustGateService_unknownDimension_returnsEmpty() {
        final String actorId = "agent-unknowndim-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedDimension(actorId, now.minus(1, ChronoUnit.DAYS), "thoroughness", 0.8);
        trustScoreJob.runComputation();

        assertThat(trustGateService.dimensionScore(actorId, "nonexistent-dimension")).isEmpty();
    }

    // ── Fixtures ─────────────────────────────────────────────────────────────

    private void seedDimension(final String actorId, final Instant decisionTime,
            final String dimension, final double dimensionScore) {
        LedgerTestFixtures.seedDecisionWithDimension(actorId, decisionTime,
                dimension, dimensionScore, CapabilityTag.GLOBAL, repo, em);
    }
}
```

- [ ] **Step 2: Run the test — confirm it fails because no DIMENSION rows are produced**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreDimensionIT -q 2>&1 | grep -E "FAIL|PASS|ERROR|Tests run" | head -10
```

Expected: Tests run with failures — `dimensionAttestations_producesDimensionRows` fails because `trustRepo.findByActorIdAndScoreType(actorId, DIMENSION)` returns empty list.

- [ ] **Step 3: Implement the dimension pass in `TrustScoreJob.java`**

In `TrustScoreJob.runComputation()`, add the dimension pass between the capability pass and the global pass. After the `capabilityScores` loop (the `for (Map.Entry<String, Map<UUID,...>> capEntry : ...)` block) and before the `// ── Global pass ──` comment:

```java
            // ── Dimension pass ─────────────────────────────────────────────────────────
            // Group actor's dimension-tagged attestations by dimension in one pass.
            // Excludes attestations with null dimensionScore — they carry no quality signal.
            final Map<String, List<LedgerAttestation>> byDimension = actorAttestations.stream()
                    .filter(a -> a.trustDimension != null && a.dimensionScore != null)
                    .collect(Collectors.groupingBy(a -> a.trustDimension));

            for (final Map.Entry<String, List<LedgerAttestation>> dimEntry : byDimension.entrySet()) {
                final String dimension = dimEntry.getKey();
                final List<LedgerAttestation> dimAttestations = dimEntry.getValue();

                computer.computeDimensionScore(dimAttestations, now).ifPresent(dimScore -> {
                    final int dimPositive = (int) dimAttestations.stream()
                            .filter(a -> a.dimensionScore != null && a.dimensionScore > 0.5).count();
                    final int dimNegative = (int) dimAttestations.stream()
                            .filter(a -> a.dimensionScore != null && a.dimensionScore <= 0.5).count();
                    final int dimDecisionCount = (int) dimAttestations.stream()
                            .map(a -> a.ledgerEntryId).distinct().count();

                    trustRepo.upsert(actorId, ActorTrustScore.ScoreType.DIMENSION, dimension,
                            actorType, dimScore,
                            dimDecisionCount, 0,
                            0.0, 0.0,
                            dimPositive, dimNegative, now);
                });
            }
```

- [ ] **Step 4: Run `TrustScoreDimensionIT` — confirm all tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=TrustScoreDimensionIT -q
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 5: Run the full runtime test suite — confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```

Expected: BUILD SUCCESS. 280+ tests pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java \
        runtime/src/test/java/io/casehub/ledger/service/TrustScoreDimensionIT.java
git commit -m "feat(#62): add TrustScoreJob dimension pass — DIMENSION rows from dimension-tagged attestations

Refs #62 #50"
```

---

## Task 9: Documentation

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update `docs/DESIGN.md` — add dimension-scoped trust section**

Find the existing "Capability-Scoped Trust Scores" section (or the `actor_trust_score` / `TrustScoreJob` description) in `DESIGN.md`. After the capability-scoped trust discussion (which references #61), add:

```markdown
### Dimension-Scoped Trust Scores (#62)

Dimension scores answer "along which quality axes does this actor excel?" — orthogonal to
capability scores ("what tasks can this actor handle?").

A dimension attestation carries a continuous `dimensionScore` ∈ [0.0, 1.0] alongside the
binary `verdict`. `trustDimension` names the quality axis (e.g. `"review-thoroughness"`,
`"false-positive-rate"`). Both fields are nullable — ordinary attestations omit them.

**Computation model:** For each `(actorId, trustDimension)` pair, `TrustScoreJob` computes a
decay-weighted average:

```
score = Σ(weight_i × dimensionScore_i) / Σ(weight_i)
weight_i = 2^(-ageInDays_i / halfLifeDays)
```

Pure time-based decay (no valence asymmetry) — `TrustScoreComputer.computeDimensionScore()`.
Stored as `DIMENSION` rows in `actor_trust_score` (`scope_key = dimensionName`).
The `alpha_value` and `beta_value` columns are not meaningful for DIMENSION rows (stored as 0.0).

**Query surface:** `TrustGateService.dimensionScores(actorId)` → `Map<String, Double>` (all
dimensions). `TrustGateService.dimensionScore(actorId, dimension)` → `Optional<Double>` (one).

**Application responsibility:** dimension names (e.g. `"review-thoroughness"`) are defined
and stamped by consuming extensions — this library provides the storage and computation
infrastructure only.
```

- [ ] **Step 2: Update `CLAUDE.md` — project structure table**

In the `TrustGateService.java` row in the project structure, update the description to include the dimension methods:

Replace the `TrustGateService.java` line:
```
├── TrustGateService.java        — CDI bean: trust threshold enforcement (meetsThreshold, currentScore)
```
With:
```
├── TrustGateService.java        — CDI bean: trust threshold enforcement (meetsThreshold, currentScore, dimensionScores, dimensionScore)
```

Also update the `TrustScoreJob.java` line to mention the dimension pass:
Replace:
```
├── TrustScoreJob.java           — @Scheduled nightly recomputation
```
With:
```
├── TrustScoreJob.java           — @Scheduled nightly recomputation (capability pass → dimension pass → global pass)
```

And update the `TrustScoreComputer.java` line:
Replace:
```
├── TrustScoreComputer.java      — Bayesian Beta trust scoring; delegates decay to DecayFunction (pure Java)
```
With:
```
├── TrustScoreComputer.java      — Bayesian Beta trust scoring (compute) + decay-weighted dimension average (computeDimensionScore); delegates decay to DecayFunction (pure Java)
```

- [ ] **Step 3: Check for any other documentation referencing #62 as future work, and mark it done**

```bash
grep -rn "#62\|dimension\|trustDimension\|trust_dimension" \
  /Users/mdproctor/claude/casehub/ledger/docs/ \
  /Users/mdproctor/claude/casehub/ledger/CLAUDE.md \
  --include="*.md" | grep -v "target/" | grep -v ".git/"
```

For each occurrence: verify it is either updated to reflect the completed implementation, or correctly describes the current state. Common places to check: `DESIGN.md`, `DESIGN-capabilities.md`, `RESEARCH.md`, `CAPABILITIES.md`, `CLAUDE.md`.

- [ ] **Step 4: Check cross-references are correct**

```bash
grep -rn "Requires #62\|wired by #62\|added by #62" \
  /Users/mdproctor/claude/casehub/ledger/ \
  --include="*.java" --include="*.md" | grep -v "target/"
```

Update any comment saying "requires #62" to remove the future-tense reference (it is now implemented).

- [ ] **Step 5: Build to confirm docs-only changes don't break compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit documentation**

```bash
git add docs/DESIGN.md CLAUDE.md
git commit -m "docs(#62): document dimension-scoped trust scores, update CLAUDE.md structure

Closes #62. Refs #50"
```

---

## Task 10: Close the issue and run final verification

- [ ] **Step 1: Run the full test suite across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -q
```

Expected: BUILD SUCCESS. Note the final test count and confirm it is larger than the 272 from the session start.

- [ ] **Step 2: Verify test count increase**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | grep "Tests run" | tail -3
```

Expected: runtime module shows a higher test count than before (new tests added in Tasks 3, 4, 6, 7, 8).

- [ ] **Step 3: Close GitHub issue #62**

```bash
gh issue close 62 --repo casehubio/ledger \
  --comment "Implemented in this session. All acceptance criteria met:
- \`trustDimension\` and \`dimensionScore\` fields on \`LedgerAttestation\` ✅
- \`ActorTrustScore\` DIMENSION rows computed per actor per dimension ✅  
- \`TrustScoreJob\` dimension pass (decay-weighted average, not Bayesian Beta) ✅
- \`TrustGateService.dimensionScores()\` and \`dimensionScore()\` query methods ✅
- Design decision (score_type discriminator) already in schema; documented in DESIGN.md ✅
- All existing tests pass (backward-compatible) ✅"
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ `trustDimension` and `dimensionScore` on `LedgerAttestation` — Tasks 1, 2
- ✅ `ActorTrustScore` DIMENSION rows with scope_key = dimensionName — Tasks 7, 8
- ✅ `TrustScoreJob` dimension pass — Task 8
- ✅ `getDimensionScores` / `getDimensionScore` query methods — Task 4
- ✅ Design decision (score_type discriminator) documented — Task 9
- ✅ All existing tests pass — verified in Tasks 2, 8 (full suite run)
- ✅ Unit tests (TrustScoreComputerTest, TrustGateServiceTest) — Tasks 3, 4
- ✅ Integration tests (ActorTrustScoreRepositoryIT, LedgerAttestationDimensionIT) — Tasks 6, 7
- ✅ End-to-end tests (TrustScoreDimensionIT) — Task 8
- ✅ Happy path tests — all IT files
- ✅ Robustness tests (null dimensionScore, no dimension attestations, unknown dimension) — Tasks 6, 8
- ✅ Correctness tests (isolation, decay, backward compat) — Tasks 6, 7, 8
- ✅ Documentation — Task 9
- ✅ All commits linked to #62 and #50 — every commit message

**Placeholder scan:** None — all steps contain exact code.

**Type consistency:** `OptionalDouble` used throughout (not `Optional<Double>`) for `computeDimensionScore` return type. `TrustGateService.dimensionScore()` returns `Optional<Double>` (consistent with `currentScore(actorId)`). `TrustGateService.dimensionScores()` returns `Map<String, Double>`. All consistent across tasks.
