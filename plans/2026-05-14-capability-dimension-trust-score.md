# CAPABILITY_DIMENSION Composite Trust Score — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `CAPABILITY_DIMENSION` as a fourth trust score type, replacing the single `scope_key` column with typed `capability_key` + `dimension_key` columns, and adding per-capability quality scoring throughout the stack.

**Architecture:** Schema-first — V1005 drops `scope_key` and adds two nullable typed columns with a CHECK constraint enforcing score-type/key consistency. The API module `@MappedSuperclass` tracks this. All repository, service, federation, and test layers then update to match the clean two-column model. The new CAPABILITY_DIMENSION job pass and `TrustGateService` methods are added TDD-style after the foundation compiles.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM, H2 (test, PostgreSQL mode), JUnit 5, AssertJ, `@QuarkusTest`.

**Build commands:**
```bash
# API module only (pure JUnit 5, fast)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api

# Runtime only (QuarkusTest, needs api installed first if api changed)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && \
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime

# Full build
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

---

## File Map

| Action | File |
|--------|------|
| CREATE | `runtime/src/main/resources/db/migration/V1005__actor_trust_score_two_column_keys.sql` |
| MODIFY | `api/src/main/java/io/casehub/ledger/api/model/ActorTrustScore.java` |
| MODIFY | `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java` |
| MODIFY | `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java` |
| MODIFY | `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java` |
| MODIFY | `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java` |
| MODIFY | `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java` |
| MODIFY | `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/ActorExport.java` |
| CREATE | `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/CapabilityDimensionScoreExport.java` |
| MODIFY | `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustExportService.java` |
| MODIFY | `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/JpaTrustImportService.java` |
| MODIFY | `runtime/src/test/java/io/casehub/ledger/service/TrustGateServiceTest.java` |
| MODIFY | `runtime/src/test/java/io/casehub/ledger/service/ActorTrustScoreRepositoryIT.java` |
| MODIFY | `runtime/src/test/java/io/casehub/ledger/service/TrustScoreCapabilityIT.java` |
| MODIFY | `runtime/src/test/java/io/casehub/ledger/service/TrustScoreDimensionIT.java` |
| CREATE | `runtime/src/test/java/io/casehub/ledger/service/TrustScoreCapabilityDimensionIT.java` |

---

## Task 1: V1005 Flyway Migration

**Files:**
- Create: `runtime/src/main/resources/db/migration/V1005__actor_trust_score_two_column_keys.sql`

- [ ] **Step 1: Create the migration file**

```sql
-- CaseHub Ledger — actor_trust_score two-column key schema (V1005)
--
-- Replaces the single scope_key column with two typed nullable columns:
--   capability_key  — set for CAPABILITY and CAPABILITY_DIMENSION rows; null otherwise
--   dimension_key   — set for DIMENSION and CAPABILITY_DIMENSION rows; null otherwise
--
-- The CHECK constraint makes the schema self-enforcing.
-- The unique constraint no longer includes score_type (it is now deterministic from the keys).

ALTER TABLE actor_trust_score DROP CONSTRAINT uq_actor_trust_score_key;
ALTER TABLE actor_trust_score DROP COLUMN scope_key;

ALTER TABLE actor_trust_score ADD COLUMN capability_key VARCHAR(255);
ALTER TABLE actor_trust_score ADD COLUMN dimension_key  VARCHAR(255);

ALTER TABLE actor_trust_score
    ADD CONSTRAINT uq_actor_trust_score_key
        UNIQUE NULLS NOT DISTINCT (actor_id, capability_key, dimension_key);

ALTER TABLE actor_trust_score
    ADD CONSTRAINT chk_actor_trust_score_keys CHECK (
        (score_type = 'GLOBAL'               AND capability_key IS NULL      AND dimension_key IS NULL    ) OR
        (score_type = 'CAPABILITY'           AND capability_key IS NOT NULL   AND dimension_key IS NULL    ) OR
        (score_type = 'DIMENSION'            AND capability_key IS NULL       AND dimension_key IS NOT NULL) OR
        (score_type = 'CAPABILITY_DIMENSION' AND capability_key IS NOT NULL   AND dimension_key IS NOT NULL)
    );
```

---

## Task 2: API Module — `ScoreType` and `@MappedSuperclass` Fields

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/model/ActorTrustScore.java`

- [ ] **Step 1: Add `CAPABILITY_DIMENSION` to `ScoreType` enum and replace `scopeKey` with two fields**

Replace the entire file content with:

```java
package io.casehub.ledger.api.model;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.MappedSuperclass;

/**
 * Bayesian Beta trust score for a decision-making actor, scoped by score type.
 *
 * <p>
 * One row per {@code (actor_id, capability_key, dimension_key)} triple:
 * <ul>
 * <li>{@code GLOBAL} — capability_key null, dimension_key null. Classic score across all decisions.</li>
 * <li>{@code CAPABILITY} — capability_key set, dimension_key null. Scoped binary trust. See ADR 0008.</li>
 * <li>{@code DIMENSION} — capability_key null, dimension_key set. Cross-capability quality score. See #62.</li>
 * <li>{@code CAPABILITY_DIMENSION} — both keys set. Per-capability quality dimension score. See #76.</li>
 * </ul>
 *
 * <p>
 * Binary scores (GLOBAL, CAPABILITY, CAPABILITY_DIMENSION) use Bayesian Beta statistics.
 * Continuous scores (DIMENSION, CAPABILITY_DIMENSION) use decay-weighted average; alpha and beta
 * are stored as 0.0 for these rows. See #78 for the ADR documenting this distinction.
 */
@MappedSuperclass
public class ActorTrustScore {

    /** Score type discriminator — determines which key columns are non-null. */
    public enum ScoreType {
        /** Classic cross-decision score. capability_key and dimension_key are null. */
        GLOBAL,
        /** Capability-scoped binary trust. capability_key is set; dimension_key is null. See ADR 0008. */
        CAPABILITY,
        /** Cross-capability quality dimension. capability_key is null; dimension_key is set. See #62. */
        DIMENSION,
        /** Per-capability quality dimension. Both capability_key and dimension_key are set. See #76. */
        CAPABILITY_DIMENSION
    }

    @Id
    @Column(name = "id", nullable = false)
    public UUID id;

    @Column(name = "actor_id", nullable = false)
    public String actorId;

    @Enumerated(EnumType.STRING)
    @Column(name = "score_type", nullable = false)
    public ScoreType scoreType = ScoreType.GLOBAL;

    /** Capability tag for CAPABILITY and CAPABILITY_DIMENSION rows; null for GLOBAL and DIMENSION. */
    @Column(name = "capability_key")
    public String capabilityKey;

    /** Quality dimension name for DIMENSION and CAPABILITY_DIMENSION rows; null for GLOBAL and CAPABILITY. */
    @Column(name = "dimension_key")
    public String dimensionKey;

    @Enumerated(EnumType.STRING)
    @Column(name = "actor_type")
    public ActorType actorType;

    @Column(name = "trust_score")
    public double trustScore;

    /** Bayesian Beta α parameter. Stored as 0.0 for DIMENSION and CAPABILITY_DIMENSION rows. */
    @Column(name = "alpha_value")
    public double alpha;

    /** Bayesian Beta β parameter. Stored as 0.0 for DIMENSION and CAPABILITY_DIMENSION rows. */
    @Column(name = "beta_value")
    public double beta;

    @Column(name = "decision_count")
    public int decisionCount;

    @Column(name = "overturned_count")
    public int overturnedCount;

    @Column(name = "attestation_positive")
    public int attestationPositive;

    @Column(name = "attestation_negative")
    public int attestationNegative;

    @Column(name = "last_computed_at")
    public Instant lastComputedAt;

    /**
     * EigenTrust global trust share in [0.0, 1.0]; values sum to ≤ 1.0 across all actors.
     * Only meaningful on GLOBAL rows. Zero when EigenTrust is disabled or not yet computed.
     */
    @Column(name = "global_trust_score")
    public double globalTrustScore;
}
```

- [ ] **Step 2: Run API tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
```

Expected: BUILD SUCCESS (API module has no test that references `scopeKey`).

---

## Task 3: Runtime Entity — Update Named Queries

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java`

- [ ] **Step 1: Replace all `@NamedQuery` annotations**

Replace the entire file:

```java
package io.casehub.ledger.runtime.model;

import jakarta.persistence.Entity;
import jakarta.persistence.NamedQuery;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;

/**
 * Bayesian Beta trust score for a decision-making actor, scoped by score type.
 *
 * <p>
 * One row per {@code (actor_id, capability_key, dimension_key)} triple.
 * Plain {@code @Entity} — queries via {@code @NamedQuery} + EntityManager.
 */
@Entity
@Table(name = "actor_trust_score", uniqueConstraints = @UniqueConstraint(
        name = "uq_actor_trust_score_key",
        columnNames = {"actor_id", "capability_key", "dimension_key"}))
@NamedQuery(
        name = "ActorTrustScore.findAll",
        query = "SELECT s FROM ActorTrustScore s")
@NamedQuery(
        name = "ActorTrustScore.findGlobalByActorId",
        query = "SELECT s FROM ActorTrustScore s WHERE s.actorId = :actorId AND s.scoreType = :scoreType AND s.capabilityKey IS NULL AND s.dimensionKey IS NULL")
@NamedQuery(
        name = "ActorTrustScore.findByActorIdAndScoreType",
        query = "SELECT s FROM ActorTrustScore s WHERE s.actorId = :actorId AND s.scoreType = :scoreType")
@NamedQuery(
        name = "ActorTrustScore.findCapabilityByActorIdAndTag",
        query = "SELECT s FROM ActorTrustScore s WHERE s.actorId = :actorId AND s.scoreType = :scoreType AND s.capabilityKey = :capabilityKey AND s.dimensionKey IS NULL")
@NamedQuery(
        name = "ActorTrustScore.findDimensionByActorIdAndKey",
        query = "SELECT s FROM ActorTrustScore s WHERE s.actorId = :actorId AND s.scoreType = :scoreType AND s.capabilityKey IS NULL AND s.dimensionKey = :dimensionKey")
@NamedQuery(
        name = "ActorTrustScore.findCapabilityDimensionByKeys",
        query = "SELECT s FROM ActorTrustScore s WHERE s.actorId = :actorId AND s.scoreType = :scoreType AND s.capabilityKey = :capabilityKey AND s.dimensionKey = :dimensionKey")
@NamedQuery(
        name = "ActorTrustScore.findCapabilityDimensionsByCapability",
        query = "SELECT s FROM ActorTrustScore s WHERE s.actorId = :actorId AND s.scoreType = :scoreType AND s.capabilityKey = :capabilityKey")
@NamedQuery(
        name = "ActorTrustScore.findAllByLastComputedAtAfter",
        query = "SELECT s FROM ActorTrustScore s WHERE s.lastComputedAt > :since")
public class ActorTrustScore extends io.casehub.ledger.api.model.ActorTrustScore {

}
```

---

## Task 4: Repository SPI — Typed Methods

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java`

- [ ] **Step 1: Replace the SPI interface**

```java
package io.casehub.ledger.runtime.repository;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.runtime.model.ActorTrustScore;

/** SPI for persisting and querying {@link ActorTrustScore} records. */
public interface ActorTrustScoreRepository {

    /**
     * Find the GLOBAL trust score for an actor, or empty if none computed yet.
     */
    Optional<ActorTrustScore> findByActorId(String actorId);

    /**
     * Find the CAPABILITY score for an actor and capability tag, or empty if not yet computed.
     */
    Optional<ActorTrustScore> findCapabilityScore(String actorId, String capabilityTag);

    /**
     * Find the DIMENSION score for an actor and dimension name, or empty if not yet computed.
     */
    Optional<ActorTrustScore> findDimensionScore(String actorId, String dimension);

    /**
     * Find the CAPABILITY_DIMENSION score for an actor, capability tag, and dimension name.
     */
    Optional<ActorTrustScore> findCapabilityDimension(String actorId, String capabilityTag, String dimension);

    /**
     * Return all CAPABILITY_DIMENSION scores for an actor scoped to the given capability tag.
     */
    List<ActorTrustScore> findCapabilityDimensions(String actorId, String capabilityTag);

    /**
     * Return all trust scores for an actor of a given type.
     * For GLOBAL: returns 0 or 1 result. For CAPABILITY/DIMENSION/CAPABILITY_DIMENSION: returns all scoped rows.
     */
    List<ActorTrustScore> findByActorIdAndScoreType(String actorId, ScoreType scoreType);

    /**
     * Upsert (insert or update) a trust score for the given actor and scope.
     *
     * @param capabilityKey the capability tag, or null for GLOBAL and DIMENSION rows
     * @param dimensionKey  the dimension name, or null for GLOBAL and CAPABILITY rows
     */
    void upsert(String actorId, ScoreType scoreType,
            String capabilityKey, String dimensionKey,
            ActorType actorType, double trustScore,
            int decisionCount, int overturnedCount, double alpha, double beta,
            int attestationPositive, int attestationNegative,
            Instant lastComputedAt);

    /**
     * Update the EigenTrust global trust score for an actor's GLOBAL row.
     */
    void updateGlobalTrustScore(String actorId, double globalTrustScore);

    /**
     * Return all computed trust scores across all actors and score types.
     */
    List<ActorTrustScore> findAll();

    /**
     * Return all trust scores whose {@code lastComputedAt} timestamp is strictly after {@code since}.
     */
    List<ActorTrustScore> findAllByLastComputedAtAfter(Instant since);
}
```

---

## Task 5: JPA Implementation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java`

- [ ] **Step 1: Replace the entire implementation**

```java
package io.casehub.ledger.runtime.repository.jpa;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;

/**
 * JPA / EntityManager implementation of {@link ActorTrustScoreRepository}.
 *
 * <p>
 * Upsert is a find-then-update to remain compatible with H2 and PostgreSQL without
 * database-specific SQL. The unique constraint (actor_id, capability_key, dimension_key) with
 * NULLS NOT DISTINCT prevents duplicate rows at the database level.
 *
 * <p>
 * Upsert assumption: each {@code (actorId, scoreType, capabilityKey, dimensionKey)} combination
 * is upserted at most once per transaction. Calling {@code upsert()} twice for the same key in
 * a single transaction may produce a duplicate row if Hibernate does not flush before the second
 * find. Under the default {@code FlushModeType.AUTO}, named queries trigger a flush, so this is
 * safe in practice.
 */
@ApplicationScoped
public class JpaActorTrustScoreRepository implements ActorTrustScoreRepository {

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @Override
    public Optional<ActorTrustScore> findByActorId(final String actorId) {
        return em.createNamedQuery("ActorTrustScore.findGlobalByActorId", ActorTrustScore.class)
                .setParameter("actorId", actorId)
                .setParameter("scoreType", ScoreType.GLOBAL)
                .getResultStream()
                .findFirst();
    }

    @Override
    public Optional<ActorTrustScore> findCapabilityScore(final String actorId, final String capabilityTag) {
        return em.createNamedQuery("ActorTrustScore.findCapabilityByActorIdAndTag", ActorTrustScore.class)
                .setParameter("actorId", actorId)
                .setParameter("scoreType", ScoreType.CAPABILITY)
                .setParameter("capabilityKey", capabilityTag)
                .getResultStream()
                .findFirst();
    }

    @Override
    public Optional<ActorTrustScore> findDimensionScore(final String actorId, final String dimension) {
        return em.createNamedQuery("ActorTrustScore.findDimensionByActorIdAndKey", ActorTrustScore.class)
                .setParameter("actorId", actorId)
                .setParameter("scoreType", ScoreType.DIMENSION)
                .setParameter("dimensionKey", dimension)
                .getResultStream()
                .findFirst();
    }

    @Override
    public Optional<ActorTrustScore> findCapabilityDimension(final String actorId,
            final String capabilityTag, final String dimension) {
        return em.createNamedQuery("ActorTrustScore.findCapabilityDimensionByKeys", ActorTrustScore.class)
                .setParameter("actorId", actorId)
                .setParameter("scoreType", ScoreType.CAPABILITY_DIMENSION)
                .setParameter("capabilityKey", capabilityTag)
                .setParameter("dimensionKey", dimension)
                .getResultStream()
                .findFirst();
    }

    @Override
    public List<ActorTrustScore> findCapabilityDimensions(final String actorId, final String capabilityTag) {
        return em.createNamedQuery("ActorTrustScore.findCapabilityDimensionsByCapability", ActorTrustScore.class)
                .setParameter("actorId", actorId)
                .setParameter("scoreType", ScoreType.CAPABILITY_DIMENSION)
                .setParameter("capabilityKey", capabilityTag)
                .getResultList();
    }

    @Override
    public List<ActorTrustScore> findByActorIdAndScoreType(
            final String actorId, final ScoreType scoreType) {
        return em.createNamedQuery("ActorTrustScore.findByActorIdAndScoreType", ActorTrustScore.class)
                .setParameter("actorId", actorId)
                .setParameter("scoreType", scoreType)
                .getResultList();
    }

    @Override
    @Transactional
    public void upsert(final String actorId, final ScoreType scoreType,
            final String capabilityKey, final String dimensionKey,
            final ActorType actorType, final double trustScore,
            final int decisionCount, final int overturnedCount,
            final double alpha, final double beta,
            final int attestationPositive, final int attestationNegative,
            final Instant lastComputedAt) {

        ActorTrustScore score = findExisting(actorId, scoreType, capabilityKey, dimensionKey);
        if (score == null) {
            score = new ActorTrustScore();
            score.id = UUID.randomUUID();
            score.actorId = actorId;
            score.scoreType = scoreType;
            score.capabilityKey = capabilityKey;
            score.dimensionKey = dimensionKey;
        }
        score.actorType = actorType;
        score.trustScore = trustScore;
        score.alpha = alpha;
        score.beta = beta;
        score.decisionCount = decisionCount;
        score.overturnedCount = overturnedCount;
        score.attestationPositive = attestationPositive;
        score.attestationNegative = attestationNegative;
        score.lastComputedAt = lastComputedAt;
        em.merge(score);
    }

    private ActorTrustScore findExisting(final String actorId, final ScoreType scoreType,
            final String capabilityKey, final String dimensionKey) {
        return switch (scoreType) {
            case GLOBAL -> findByActorId(actorId).orElse(null);
            case CAPABILITY -> findCapabilityScore(actorId, capabilityKey).orElse(null);
            case DIMENSION -> findDimensionScore(actorId, dimensionKey).orElse(null);
            case CAPABILITY_DIMENSION -> findCapabilityDimension(actorId, capabilityKey, dimensionKey).orElse(null);
        };
    }

    @Override
    @Transactional
    public void updateGlobalTrustScore(final String actorId, final double globalTrustScore) {
        findByActorId(actorId).ifPresent(score -> {
            score.globalTrustScore = globalTrustScore;
            em.merge(score);
        });
    }

    @Override
    public List<ActorTrustScore> findAll() {
        return em.createNamedQuery("ActorTrustScore.findAll", ActorTrustScore.class)
                .getResultList();
    }

    @Override
    public List<ActorTrustScore> findAllByLastComputedAtAfter(final Instant since) {
        return em.createNamedQuery("ActorTrustScore.findAllByLastComputedAtAfter", ActorTrustScore.class)
                .setParameter("since", since)
                .getResultList();
    }
}
```

---

## Task 6: Create `CapabilityDimensionScoreExport` and Update `ActorExport`

> **Must run before Tasks 6c and 6d** — those tasks reference `CapabilityDimensionScoreExport`.

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/CapabilityDimensionScoreExport.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/ActorExport.java`

- [ ] **Step 1: Create `CapabilityDimensionScoreExport`**

```java
package io.casehub.ledger.runtime.service.federation;

import java.time.Instant;

/**
 * Per-capability quality dimension score for one actor.
 * Mirrors {@link DimensionScoreExport} but scoped to a specific capability tag.
 */
public record CapabilityDimensionScoreExport(
        String capabilityTag,
        String dimension,
        double score,
        int sampleCount,
        Instant lastComputedAt) {
}
```

- [ ] **Step 2: Update `ActorExport` to add the fourth field**

```java
package io.casehub.ledger.runtime.service.federation;

import java.util.List;

import io.casehub.ledger.api.model.ActorType;

/** All trust scores for a single actor, structured by score type. */
public record ActorExport(
        String actorId,
        ActorType actorType,
        GlobalScoreExport globalScore,
        List<CapabilityScoreExport> capabilityScores,
        List<DimensionScoreExport> dimensionScores,
        List<CapabilityDimensionScoreExport> capabilityDimensionScores) {
}
```

---

## Task 7: Fix Production Callers

### 7a: `TrustGateService`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java`

- [ ] **Step 1: Replace `TrustGateService` — update old calls, add new methods**

```java
package io.casehub.ledger.runtime.service;

import java.util.Map;
import java.util.Optional;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;

/**
 * CDI bean for trust threshold enforcement.
 *
 * <p>
 * Single query point for trust decisions. Consumers call these methods rather than querying
 * {@link ActorTrustScoreRepository} directly — threshold and fallback logic stays in one place.
 */
@ApplicationScoped
public class TrustGateService {

    private final ActorTrustScoreRepository repository;

    @Inject
    public TrustGateService(final ActorTrustScoreRepository repository) {
        this.repository = repository;
    }

    /** Returns true if the actor's global trust score meets or exceeds {@code minTrust}. */
    public boolean meetsThreshold(final String actorId, final double minTrust) {
        return repository.findByActorId(actorId)
                .map(s -> s.trustScore >= minTrust)
                .orElse(false);
    }

    /**
     * Returns true if the actor's CAPABILITY score for {@code capabilityTag} meets {@code minTrust}.
     * Falls back to the global score when no capability-specific score has been computed.
     */
    public boolean meetsThreshold(final String actorId, final String capabilityTag,
            final double minTrust) {
        return repository.findCapabilityScore(actorId, capabilityTag)
                .map(s -> s.trustScore >= minTrust)
                .orElseGet(() -> meetsThreshold(actorId, minTrust));
    }

    /**
     * Returns true if the actor's CAPABILITY_DIMENSION quality score for the given
     * capability+dimension meets {@code minScore}. Returns false if no score exists.
     */
    public boolean meetsQualityThreshold(final String actorId, final String capabilityTag,
            final String dimension, final double minScore) {
        return repository.findCapabilityDimension(actorId, capabilityTag, dimension)
                .map(s -> s.trustScore >= minScore)
                .orElse(false);
    }

    /** Returns the actor's global trust score, or empty if not yet computed. */
    public Optional<Double> currentScore(final String actorId) {
        return repository.findByActorId(actorId).map(s -> s.trustScore);
    }

    /** Returns the actor's CAPABILITY score for the given tag, or empty if not yet computed. */
    public Optional<Double> currentScore(final String actorId, final String capabilityTag) {
        return repository.findCapabilityScore(actorId, capabilityTag).map(s -> s.trustScore);
    }

    /** Returns the actor's CAPABILITY_DIMENSION quality score, or empty if not yet computed. */
    public Optional<Double> qualityScore(final String actorId, final String capabilityTag,
            final String dimension) {
        return repository.findCapabilityDimension(actorId, capabilityTag, dimension)
                .map(s -> s.trustScore);
    }

    /**
     * Returns all DIMENSION scores for the actor, keyed by dimension name.
     * Empty map if no dimension scores have been computed.
     */
    public Map<String, Double> dimensionScores(final String actorId) {
        return repository.findByActorIdAndScoreType(actorId, ScoreType.DIMENSION).stream()
                .collect(Collectors.toMap(s -> s.dimensionKey, s -> s.trustScore));
    }

    /** Returns the DIMENSION score for a specific dimension, or empty if not yet computed. */
    public Optional<Double> dimensionScore(final String actorId, final String dimension) {
        return repository.findDimensionScore(actorId, dimension).map(s -> s.trustScore);
    }

    /**
     * Returns all CAPABILITY_DIMENSION quality scores for the actor scoped to
     * {@code capabilityTag}, keyed by dimension name. Empty map if none computed.
     */
    public Map<String, Double> qualityScores(final String actorId, final String capabilityTag) {
        return repository.findCapabilityDimensions(actorId, capabilityTag).stream()
                .collect(Collectors.toMap(s -> s.dimensionKey, s -> s.trustScore));
    }

    /**
     * Returns the full {@link ActorTrustScore} entity for the actor's GLOBAL row, or empty.
     * Use when the caller needs metrics beyond the scalar score (alpha, beta, decisionCount, etc.).
     */
    public Optional<ActorTrustScore> findScore(final String actorId) {
        return repository.findByActorId(actorId);
    }
}
```

### 7b: `TrustScoreJob` — update upsert calls

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`

- [ ] **Step 1: Update all `trustRepo.upsert(...)` calls to the new two-key signature**

In `runComputation()`, find and update each `trustRepo.upsert(...)` call. The new signature takes `capabilityKey` and `dimensionKey` instead of `scopeKey`.

CAPABILITY upsert (inside the capability pass loop):
```java
trustRepo.upsert(actorId, ActorTrustScore.ScoreType.CAPABILITY, capabilityTag, null,
        actorType, capScore.trustScore(),
        capScore.decisionCount(), capScore.overturnedCount(),
        capScore.alpha(), capScore.beta(),
        capScore.attestationPositive(), capScore.attestationNegative(), now);
```

DIMENSION upsert (inside the dimension pass loop):
```java
trustRepo.upsert(actorId, ActorTrustScore.ScoreType.DIMENSION, null, dimension,
        actorType, dimScore,
        dimDecisionCount, 0,
        0.0, 0.0,
        dimPositive, dimNegative, now);
```

GLOBAL upsert:
```java
trustRepo.upsert(actorId, ActorTrustScore.ScoreType.GLOBAL, null, null,
        actorType, finalScore.trustScore(),
        finalScore.decisionCount(), finalScore.overturnedCount(),
        finalScore.alpha(), finalScore.beta(),
        finalScore.attestationPositive(), finalScore.attestationNegative(), now);
```

### 7c: `TrustExportService` — update field references

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustExportService.java`

- [ ] **Step 1: Update `exportActor` to include `CAPABILITY_DIMENSION` rows**

In `exportActor`, add:
```java
scores.addAll(trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY_DIMENSION));
```

- [ ] **Step 2: Update `toActorExport` — replace `s.scopeKey` with `s.capabilityKey`/`s.dimensionKey`, add composite projection**

Replace `toActorExport`:
```java
private ActorExport toActorExport(final List<ActorTrustScore> scores) {
    final String actorId = scores.get(0).actorId;
    final ActorType actorType = scores.stream()
            .map(s -> s.actorType)
            .filter(t -> t != null)
            .findFirst()
            .orElse(ActorType.HUMAN);

    final GlobalScoreExport global = scores.stream()
            .filter(s -> s.scoreType == ScoreType.GLOBAL)
            .findFirst()
            .map(s -> new GlobalScoreExport(s.alpha, s.beta, s.trustScore,
                    s.decisionCount, s.attestationPositive, s.attestationNegative,
                    s.lastComputedAt))
            .orElse(null);

    final List<CapabilityScoreExport> capabilities = scores.stream()
            .filter(s -> s.scoreType == ScoreType.CAPABILITY)
            .map(s -> new CapabilityScoreExport(s.capabilityKey, s.alpha, s.beta, s.trustScore,
                    s.decisionCount, s.attestationPositive, s.attestationNegative,
                    s.lastComputedAt))
            .collect(Collectors.toList());

    final List<DimensionScoreExport> dimensions = scores.stream()
            .filter(s -> s.scoreType == ScoreType.DIMENSION)
            .map(s -> new DimensionScoreExport(s.dimensionKey, s.trustScore,
                    s.attestationPositive + s.attestationNegative, s.lastComputedAt))
            .collect(Collectors.toList());

    final List<CapabilityDimensionScoreExport> capabilityDimensions = scores.stream()
            .filter(s -> s.scoreType == ScoreType.CAPABILITY_DIMENSION)
            .map(s -> new CapabilityDimensionScoreExport(s.capabilityKey, s.dimensionKey,
                    s.trustScore, s.attestationPositive + s.attestationNegative, s.lastComputedAt))
            .collect(Collectors.toList());

    return new ActorExport(actorId, actorType, global, capabilities, dimensions, capabilityDimensions);
}
```

### 7d: `JpaTrustImportService` — update upsert calls

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/JpaTrustImportService.java`

- [ ] **Step 1: Update `seedActor` to use two-key upsert and handle `CAPABILITY_DIMENSION`**

Replace `seedActor`:
```java
private void seedActor(final ActorExport actor, final Instant now) {
    if (actor.globalScore() != null) {
        final GlobalScoreExport g = actor.globalScore();
        trustRepo.upsert(actor.actorId(), ScoreType.GLOBAL, null, null,
                actor.actorType(), g.trustScore(),
                g.decisionCount(), 0, g.alpha(), g.beta(),
                g.attestationPositive(), g.attestationNegative(), now);
    }
    for (final CapabilityScoreExport c : actor.capabilityScores()) {
        trustRepo.upsert(actor.actorId(), ScoreType.CAPABILITY, c.capabilityTag(), null,
                actor.actorType(), c.trustScore(),
                c.decisionCount(), 0, c.alpha(), c.beta(),
                c.attestationPositive(), c.attestationNegative(), now);
    }
    for (final DimensionScoreExport d : actor.dimensionScores()) {
        trustRepo.upsert(actor.actorId(), ScoreType.DIMENSION, null, d.dimension(),
                actor.actorType(), d.score(),
                d.sampleCount(), 0, 0.0, 0.0,
                d.sampleCount(), 0, now);
    }
    for (final CapabilityDimensionScoreExport cd : actor.capabilityDimensionScores()) {
        trustRepo.upsert(actor.actorId(), ScoreType.CAPABILITY_DIMENSION,
                cd.capabilityTag(), cd.dimension(),
                actor.actorType(), cd.score(),
                cd.sampleCount(), 0, 0.0, 0.0,
                cd.sampleCount(), 0, now);
    }
}
```

---

## Task 8: Fix Existing Tests

### 8a: `TrustGateServiceTest`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustGateServiceTest.java`

- [ ] **Step 1: Replace `StubRepository` — remove `findByActorIdAndTypeAndKey`, add new typed stubs**

The `StubRepository` inner class currently implements `findByActorIdAndTypeAndKey`. Replace with the new typed methods:

```java
private static class StubRepository implements ActorTrustScoreRepository {
    private final ActorTrustScore score;

    StubRepository(final ActorTrustScore score) {
        this.score = score;
    }

    @Override
    public Optional<ActorTrustScore> findByActorId(final String actorId) {
        return score != null && score.actorId.equals(actorId)
                ? Optional.of(score)
                : Optional.empty();
    }

    @Override
    public Optional<ActorTrustScore> findCapabilityScore(final String actorId, final String capabilityTag) {
        return Optional.empty();
    }

    @Override
    public Optional<ActorTrustScore> findDimensionScore(final String actorId, final String dimension) {
        return Optional.empty();
    }

    @Override
    public Optional<ActorTrustScore> findCapabilityDimension(final String actorId,
            final String capabilityTag, final String dimension) {
        return Optional.empty();
    }

    @Override
    public List<ActorTrustScore> findCapabilityDimensions(final String actorId, final String capabilityTag) {
        return List.of();
    }

    @Override
    public List<ActorTrustScore> findByActorIdAndScoreType(
            final String actorId, final ScoreType scoreType) {
        return List.of();
    }

    @Override
    public void upsert(final String actorId, final ScoreType scoreType,
            final String capabilityKey, final String dimensionKey,
            final ActorType actorType, final double trustScore,
            final int decisionCount, final int overturnedCount,
            final double alpha, final double beta,
            final int attestationPositive, final int attestationNegative,
            final Instant lastComputedAt) {
    }

    @Override
    public void updateGlobalTrustScore(final String actorId, final double globalTrustScore) {
    }

    @Override
    public List<ActorTrustScore> findAll() {
        return score != null ? List.of(score) : List.of();
    }

    @Override
    public List<ActorTrustScore> findAllByLastComputedAtAfter(final Instant since) {
        return List.of();
    }
}
```

- [ ] **Step 2: Update `repoWith(actorId, globalScore, capabilityTag, capabilityScore)` factory**

Replace `capability.scopeKey = capabilityTag` with `capability.capabilityKey = capabilityTag`, and override `findCapabilityScore` instead of `findByActorIdAndTypeAndKey`:

```java
private static ActorTrustScoreRepository repoWith(
        final String actorId, final double globalScore,
        final String capabilityTag, final double capabilityScore) {
    final ActorTrustScore global = new ActorTrustScore();
    global.id = UUID.randomUUID();
    global.actorId = actorId;
    global.scoreType = ScoreType.GLOBAL;
    global.actorType = ActorType.AGENT;
    global.trustScore = globalScore;
    global.lastComputedAt = Instant.now();

    final ActorTrustScore capability = new ActorTrustScore();
    capability.id = UUID.randomUUID();
    capability.actorId = actorId;
    capability.scoreType = ScoreType.CAPABILITY;
    capability.capabilityKey = capabilityTag;
    capability.actorType = ActorType.AGENT;
    capability.trustScore = capabilityScore;
    capability.lastComputedAt = Instant.now();

    return new StubRepository(global) {
        @Override
        public Optional<ActorTrustScore> findCapabilityScore(
                final String id, final String tag) {
            if (actorId.equals(id) && capabilityTag.equals(tag)) {
                return Optional.of(capability);
            }
            return Optional.empty();
        }
    };
}
```

- [ ] **Step 3: Update `repoWithDimensions` factory**

Replace `s.scopeKey = e.getKey()` with `s.dimensionKey = e.getKey()`, and replace `findByActorIdAndTypeAndKey` override with `findDimensionScore` and `findByActorIdAndScoreType`:

```java
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
                s.dimensionKey = e.getKey();
                s.actorType = ActorType.AGENT;
                s.trustScore = e.getValue();
                s.lastComputedAt = Instant.now();
                return s;
            }).collect(Collectors.toList());
        }

        @Override
        public Optional<ActorTrustScore> findDimensionScore(
                final String id, final String dimension) {
            if (!actorId.equals(id)) {
                return Optional.empty();
            }
            final Double score = dimensionScores.get(dimension);
            if (score == null) {
                return Optional.empty();
            }
            final ActorTrustScore s = new ActorTrustScore();
            s.id = UUID.randomUUID();
            s.actorId = actorId;
            s.scoreType = ScoreType.DIMENSION;
            s.dimensionKey = dimension;
            s.actorType = ActorType.AGENT;
            s.trustScore = score;
            s.lastComputedAt = Instant.now();
            return Optional.of(s);
        }
    };
}
```

### 8b: `ActorTrustScoreRepositoryIT`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/ActorTrustScoreRepositoryIT.java`

- [ ] **Step 1: Update upsert calls to two-key signature**

All `repo.upsert(actorId, ScoreType.X, "someKey", ...)` calls must be updated to pass two key params.

GLOBAL calls: `repo.upsert(actorId, ScoreType.GLOBAL, null, null, ...)`
CAPABILITY calls: `repo.upsert(actorId, ScoreType.CAPABILITY, "security-review", null, ...)`
DIMENSION calls: `repo.upsert(actorId, ScoreType.DIMENSION, null, "thoroughness", ...)`

- [ ] **Step 2: Replace `findByActorIdAndTypeAndKey` with typed methods**

Replace all `repo.findByActorIdAndTypeAndKey(actorId, ScoreType.CAPABILITY, "security-review")` with `repo.findCapabilityScore(actorId, "security-review")`.

Replace all `repo.findByActorIdAndTypeAndKey(actorId, ScoreType.DIMENSION, "thoroughness")` with `repo.findDimensionScore(actorId, "thoroughness")`.

- [ ] **Step 3: Replace `s.scopeKey` references with `s.capabilityKey` or `s.dimensionKey`**

In assertions like `assertThat(result.get().scopeKey).isEqualTo("security-review")`:
- For CAPABILITY rows: `assertThat(result.get().capabilityKey).isEqualTo("security-review")`
- For DIMENSION rows: `assertThat(result.get().dimensionKey).isEqualTo("thoroughness")`

In `findByActorIdAndScoreType` result assertions that extract `s.scopeKey`:
- CAPABILITY: `results).extracting(s -> s.capabilityKey)`
- DIMENSION: `results).extracting(s -> s.dimensionKey)`

Also update the `findByActorId_returnsGlobalScore` test: `assertThat(result.get().scopeKey).isNull()` → verify both are null:
```java
assertThat(result.get().capabilityKey).isNull();
assertThat(result.get().dimensionKey).isNull();
```

### 8c: `TrustScoreCapabilityIT`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreCapabilityIT.java`

- [ ] **Step 1: Replace `s.scopeKey` with `s.capabilityKey` and `findByActorIdAndTypeAndKey` with `findCapabilityScore`**

Lines with `s.scopeKey` (lines 70, 74, 95, 97): replace `.scopeKey` with `.capabilityKey`.

Lines using `findByActorIdAndTypeAndKey` (lines 119, 121):
```java
// Before:
trustRepo.findByActorIdAndTypeAndKey(actorId, ScoreType.CAPABILITY, "security-review")
// After:
trustRepo.findCapabilityScore(actorId, "security-review")
```

### 8d: `TrustScoreDimensionIT`

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreDimensionIT.java`

- [ ] **Step 1: Replace `s.scopeKey` with `s.dimensionKey` and `findByActorIdAndTypeAndKey` with typed methods**

Lines with `s.scopeKey` (lines 71, 76): replace `.scopeKey` with `.dimensionKey`.

Lines using `findByActorIdAndTypeAndKey` for DIMENSION (lines 181, 185):
```java
// Before:
trustRepo.findByActorIdAndTypeAndKey(actorId, ScoreType.DIMENSION, "thoroughness")
// After:
trustRepo.findDimensionScore(actorId, "thoroughness")
```

Also remove the `import io.casehub.ledger.api.model.ActorTrustScore.ScoreType` if it becomes unused after these changes (check if `ScoreType.DIMENSION` is still referenced in the file).

---

## Task 9: Build Verification — Foundation Pass

- [ ] **Step 1: Install api, then run runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && \
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: BUILD SUCCESS. All existing tests pass. If any fail, fix before proceeding.

- [ ] **Step 2: Commit the foundation**

```bash
cd ~/claude/casehub/ledger
git add -p
git commit -m "$(cat <<'EOF'
feat(#76): two-column key schema and typed repository SPI

V1005 migration: drop scope_key, add capability_key + dimension_key with
CHECK constraint. ScoreType gains CAPABILITY_DIMENSION. Repository SPI
replaces generic findByActorIdAndTypeAndKey with typed methods. All callers
and existing tests updated to match.

Refs #76
EOF
)"
```

---

## Task 10: TDD — Unit Tests for New `TrustGateService` Methods

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/TrustGateServiceTest.java`

- [ ] **Step 1: Add `repoWithCapabilityDimension` factory and new tests**

Add before the closing `}` of `TrustGateServiceTest`:

```java
private static ActorTrustScoreRepository repoWithCapabilityDimension(
        final String actorId,
        final String capabilityTag,
        final Map<String, Double> dimensionScores) {
    return new StubRepository(null) {
        @Override
        public Optional<ActorTrustScore> findCapabilityDimension(
                final String id, final String cap, final String dim) {
            if (!actorId.equals(id) || !capabilityTag.equals(cap)) {
                return Optional.empty();
            }
            final Double score = dimensionScores.get(dim);
            if (score == null) {
                return Optional.empty();
            }
            final ActorTrustScore s = new ActorTrustScore();
            s.id = UUID.randomUUID();
            s.actorId = actorId;
            s.scoreType = ScoreType.CAPABILITY_DIMENSION;
            s.capabilityKey = cap;
            s.dimensionKey = dim;
            s.actorType = ActorType.AGENT;
            s.trustScore = score;
            s.lastComputedAt = Instant.now();
            return Optional.of(s);
        }

        @Override
        public List<ActorTrustScore> findCapabilityDimensions(
                final String id, final String cap) {
            if (!actorId.equals(id) || !capabilityTag.equals(cap)) {
                return List.of();
            }
            return dimensionScores.entrySet().stream().map(e -> {
                final ActorTrustScore s = new ActorTrustScore();
                s.id = UUID.randomUUID();
                s.actorId = actorId;
                s.scoreType = ScoreType.CAPABILITY_DIMENSION;
                s.capabilityKey = capabilityTag;
                s.dimensionKey = e.getKey();
                s.actorType = ActorType.AGENT;
                s.trustScore = e.getValue();
                s.lastComputedAt = Instant.now();
                return s;
            }).collect(Collectors.toList());
        }
    };
}

// ── qualityScore ──────────────────────────────────────────────────────────

@Test
void qualityScore_returnsScore_whenCompositeExists() {
    final TrustGateService gate = new TrustGateService(
            repoWithCapabilityDimension("actor-cd", "security-review",
                    Map.of("thoroughness", 0.92)));

    assertThat(gate.qualityScore("actor-cd", "security-review", "thoroughness")).isPresent();
    assertThat(gate.qualityScore("actor-cd", "security-review", "thoroughness").get())
            .isEqualTo(0.92);
}

@Test
void qualityScore_returnsEmpty_whenNotComputed() {
    final TrustGateService gate = new TrustGateService(emptyRepo());

    assertThat(gate.qualityScore("ghost", "security-review", "thoroughness")).isEmpty();
}

@Test
void qualityScore_returnsEmpty_whenCapabilityMismatch() {
    final TrustGateService gate = new TrustGateService(
            repoWithCapabilityDimension("actor-cd2", "security-review",
                    Map.of("thoroughness", 0.92)));

    assertThat(gate.qualityScore("actor-cd2", "architecture-review", "thoroughness")).isEmpty();
}

// ── qualityScores ─────────────────────────────────────────────────────────

@Test
void qualityScores_returnsAllDimensionsForCapability() {
    final TrustGateService gate = new TrustGateService(
            repoWithCapabilityDimension("actor-qs", "security-review",
                    Map.of("thoroughness", 0.9, "false-positive-rate", 0.1)));

    final Map<String, Double> scores = gate.qualityScores("actor-qs", "security-review");
    assertThat(scores).hasSize(2);
    assertThat(scores.get("thoroughness")).isEqualTo(0.9);
    assertThat(scores.get("false-positive-rate")).isEqualTo(0.1);
}

@Test
void qualityScores_returnsEmptyMap_whenNoneComputed() {
    final TrustGateService gate = new TrustGateService(emptyRepo());

    assertThat(gate.qualityScores("ghost", "security-review")).isEmpty();
}

// ── meetsQualityThreshold ─────────────────────────────────────────────────

@Test
void meetsQualityThreshold_true_whenScoreMeetsMin() {
    final TrustGateService gate = new TrustGateService(
            repoWithCapabilityDimension("actor-mqt", "security-review",
                    Map.of("thoroughness", 0.8)));

    assertThat(gate.meetsQualityThreshold("actor-mqt", "security-review", "thoroughness", 0.75))
            .isTrue();
}

@Test
void meetsQualityThreshold_false_whenScoreBelowMin() {
    final TrustGateService gate = new TrustGateService(
            repoWithCapabilityDimension("actor-mqt2", "security-review",
                    Map.of("thoroughness", 0.6)));

    assertThat(gate.meetsQualityThreshold("actor-mqt2", "security-review", "thoroughness", 0.75))
            .isFalse();
}

@Test
void meetsQualityThreshold_false_whenNoScoreExists() {
    final TrustGateService gate = new TrustGateService(emptyRepo());

    assertThat(gate.meetsQualityThreshold("ghost", "security-review", "thoroughness", 0.0))
            .isFalse();
}
```

- [ ] **Step 2: Run unit tests and verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && \
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TrustGateServiceTest
```

Expected: BUILD SUCCESS (all new tests pass — `TrustGateService` methods are already implemented in Task 6a).

---

## Task 11: TDD — Integration Tests for `CAPABILITY_DIMENSION` Pass

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreCapabilityDimensionIT.java`

- [ ] **Step 1: Create the integration test file with failing tests**

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
 * End-to-end integration tests for CAPABILITY_DIMENSION composite trust scoring (#76).
 */
@QuarkusTest
@TestProfile(TrustScoreIT.TrustScoreTestProfile.class)
class TrustScoreCapabilityDimensionIT {

    @Inject TrustScoreJob trustScoreJob;
    @Inject TrustGateService trustGateService;
    @Inject LedgerEntryRepository repo;
    @Inject ActorTrustScoreRepository trustRepo;
    @Inject EntityManager em;

    // ── Happy path: composite attestation produces CAPABILITY_DIMENSION row ──────

    @Test
    @Transactional
    void compositeAttestation_producesCapabilityDimensionRow() {
        final String actorId = "agent-cd-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "security-review", "thoroughness", 0.92);

        trustScoreJob.runComputation();

        final var row = trustRepo.findCapabilityDimension(actorId, "security-review", "thoroughness");
        assertThat(row).isPresent();
        assertThat(row.get().scoreType).isEqualTo(ScoreType.CAPABILITY_DIMENSION);
        assertThat(row.get().capabilityKey).isEqualTo("security-review");
        assertThat(row.get().dimensionKey).isEqualTo("thoroughness");
        assertThat(row.get().trustScore).isCloseTo(0.92, within(0.05));
    }

    // ── Happy path: multiple capability+dimension combinations ────────────────

    @Test
    @Transactional
    void multipleComposites_produceSeparateRows() {
        final String actorId = "agent-cd-multi-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "security-review", "thoroughness", 0.9);
        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "security-review", "false-positive-rate", 0.1);
        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "architecture-review", "thoroughness", 0.6);

        trustScoreJob.runComputation();

        final List<ActorTrustScore> rows =
                trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY_DIMENSION);
        assertThat(rows).hasSize(3);
    }

    // ── Isolation: CAPABILITY and DIMENSION rows are unaffected ──────────────

    @Test
    @Transactional
    void compositePass_doesNotAffectCapabilityOrDimensionRows() {
        final String actorId = "agent-cd-iso-" + UUID.randomUUID();
        final Instant now = Instant.now();

        LedgerTestFixtures.seedDecision(actorId, now.minus(1, ChronoUnit.DAYS),
                AttestationVerdict.SOUND, now.minus(1, ChronoUnit.DAYS).plusSeconds(60),
                "security-review", repo, em);

        LedgerTestFixtures.seedDecisionWithDimension(actorId, now.minus(1, ChronoUnit.DAYS),
                "thoroughness", 0.8, CapabilityTag.GLOBAL, repo, em);

        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "security-review", "thoroughness", 0.9);

        trustScoreJob.runComputation();

        assertThat(trustRepo.findCapabilityScore(actorId, "security-review")).isPresent();
        assertThat(trustRepo.findDimensionScore(actorId, "thoroughness")).isPresent();
        assertThat(trustRepo.findCapabilityDimension(actorId, "security-review", "thoroughness")).isPresent();
    }

    // ── Isolation: only-capability attestation produces no CAPABILITY_DIMENSION row

    @Test
    @Transactional
    void capabilityOnlyAttestation_producesNoCapabilityDimensionRow() {
        final String actorId = "agent-cd-caponly-" + UUID.randomUUID();
        final Instant now = Instant.now();

        LedgerTestFixtures.seedDecision(actorId, now.minus(1, ChronoUnit.DAYS),
                AttestationVerdict.SOUND, now.minus(1, ChronoUnit.DAYS).plusSeconds(60),
                "security-review", repo, em);

        trustScoreJob.runComputation();

        assertThat(trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY_DIMENSION))
                .isEmpty();
    }

    // ── Isolation: GLOBAL capability tag does not produce CAPABILITY_DIMENSION row

    @Test
    @Transactional
    void globalCapabilityTag_producesNoDimensionRow() {
        final String actorId = "agent-cd-global-" + UUID.randomUUID();
        final Instant now = Instant.now();

        LedgerTestFixtures.seedDecisionWithDimension(actorId, now.minus(1, ChronoUnit.DAYS),
                "thoroughness", 0.8, CapabilityTag.GLOBAL, repo, em);

        trustScoreJob.runComputation();

        assertThat(trustRepo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY_DIMENSION))
                .isEmpty();
        assertThat(trustRepo.findDimensionScore(actorId, "thoroughness")).isPresent();
    }

    // ── Decay: older attestation has lower weight ─────────────────────────────

    @Test
    @Transactional
    void compositeScore_decays_olderAttestationHasLessWeight() {
        final String actorId = "agent-cd-decay-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "security-review", "thoroughness", 1.0);
        seedComposite(actorId, now.minus(365, ChronoUnit.DAYS),
                "security-review", "thoroughness", 0.0);

        trustScoreJob.runComputation();

        final var score = trustGateService.qualityScore(actorId, "security-review", "thoroughness");
        assertThat(score).isPresent();
        assertThat(score.get()).isGreaterThan(0.5);
    }

    // ── TrustGateService integration ─────────────────────────────────────────

    @Test
    @Transactional
    void trustGateService_qualityScore_returnsComputedScore() {
        final String actorId = "agent-cd-gate-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "security-review", "thoroughness", 0.88);

        trustScoreJob.runComputation();

        assertThat(trustGateService.qualityScore(actorId, "security-review", "thoroughness"))
                .isPresent();
        assertThat(trustGateService.qualityScore(actorId, "security-review", "thoroughness").get())
                .isCloseTo(0.88, within(0.05));
    }

    @Test
    @Transactional
    void trustGateService_qualityScores_returnsAllDimensionsForCapability() {
        final String actorId = "agent-cd-bulk-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "security-review", "thoroughness", 0.9);
        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "security-review", "false-positive-rate", 0.1);
        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "architecture-review", "thoroughness", 0.5);

        trustScoreJob.runComputation();

        final Map<String, Double> secScores =
                trustGateService.qualityScores(actorId, "security-review");
        assertThat(secScores).containsOnlyKeys("thoroughness", "false-positive-rate");
        assertThat(secScores.get("thoroughness")).isCloseTo(0.9, within(0.05));

        final Map<String, Double> archScores =
                trustGateService.qualityScores(actorId, "architecture-review");
        assertThat(archScores).containsOnlyKeys("thoroughness");
    }

    @Test
    @Transactional
    void trustGateService_meetsQualityThreshold_trueWhenScoreSufficient() {
        final String actorId = "agent-cd-thresh-" + UUID.randomUUID();
        final Instant now = Instant.now();

        seedComposite(actorId, now.minus(1, ChronoUnit.DAYS),
                "security-review", "thoroughness", 0.9);

        trustScoreJob.runComputation();

        assertThat(trustGateService.meetsQualityThreshold(
                actorId, "security-review", "thoroughness", 0.75)).isTrue();
        assertThat(trustGateService.meetsQualityThreshold(
                actorId, "security-review", "thoroughness", 0.95)).isFalse();
    }

    // ── Fixture ───────────────────────────────────────────────────────────────

    private void seedComposite(final String actorId, final Instant decisionTime,
            final String capabilityTag, final String dimension, final double score) {
        LedgerTestFixtures.seedDecisionWithDimension(
                actorId, decisionTime, dimension, score, capabilityTag, repo, em);
    }
}
```

- [ ] **Step 2: Run the new tests — verify they fail (CAPABILITY_DIMENSION pass not yet implemented)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && \
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
    -Dtest=TrustScoreCapabilityDimensionIT
```

Expected: FAIL — `trustRepo.findCapabilityDimension(...)` returns empty because no rows are written yet.

---

## Task 12: Implement `CAPABILITY_DIMENSION` Pass in `TrustScoreJob`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`

- [ ] **Step 1: Add the CAPABILITY_DIMENSION pass in `runComputation()`, between the dimension pass and the global pass**

Insert after the dimension pass loop and before the global pass section:

```java
// ── CAPABILITY_DIMENSION pass ─────────────────────────────────────────────
// Attestations tagged with both a non-GLOBAL capabilityTag and a trustDimension.
// Uses raw actorAttestations (not aggregated synthetics) — same as dimension pass.
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

        computer.computeDimensionScore(compositeAttestations, now).ifPresent(score -> {
            final int cdPositive = (int) compositeAttestations.stream()
                    .filter(a -> a.dimensionScore >= 0.5).count();
            final int cdNegative = (int) compositeAttestations.stream()
                    .filter(a -> a.dimensionScore < 0.5).count();
            final int cdDecisionCount = (int) compositeAttestations.stream()
                    .map(a -> a.ledgerEntryId).distinct().count();

            trustRepo.upsert(actorId, ActorTrustScore.ScoreType.CAPABILITY_DIMENSION,
                    capabilityTag, dimension, actorType, score,
                    cdDecisionCount, 0, 0.0, 0.0, cdPositive, cdNegative, now);
        });
    }
}
```

- [ ] **Step 2: Run the new integration tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && \
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
    -Dtest=TrustScoreCapabilityDimensionIT
```

Expected: BUILD SUCCESS — all 9 tests pass.

- [ ] **Step 3: Run the full runtime test suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && \
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
cd ~/claude/casehub/ledger
git add -p
git commit -m "$(cat <<'EOF'
feat(#76): CAPABILITY_DIMENSION pass in TrustScoreJob and TrustGateService

New fourth pass computes decay-weighted quality scores per (actor, capability,
dimension) from attestations tagged with both capabilityTag and trustDimension.
TrustGateService gains qualityScore, qualityScores, and meetsQualityThreshold.

Refs #76
EOF
)"
```

---

## Task 13: TDD — Federation Tests

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/federation/TrustExportServiceIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/federation/TrustImportServiceIT.java`

- [ ] **Step 1: Add export test for `capabilityDimensionScores`**

In `TrustExportServiceIT`, add after the existing tests:

```java
@Test
@Transactional
void exportAll_includesCapabilityDimensionScores() {
    final String actorId = "agent-export-cd-" + UUID.randomUUID();
    final Instant now = Instant.now();

    // Seed a GLOBAL row (required for exportAll threshold check)
    trustRepo.upsert(actorId, ScoreType.GLOBAL, null, null,
            ActorType.AGENT, 0.8, 5, 0, 3.0, 1.0, 5, 0, now);
    // Seed a CAPABILITY_DIMENSION row
    trustRepo.upsert(actorId, ScoreType.CAPABILITY_DIMENSION,
            "security-review", "thoroughness",
            ActorType.AGENT, 0.9, 3, 0, 0.0, 0.0, 3, 0, now);

    final TrustExportPayload payload = exportService.exportAll(0.0);

    final ActorExport actor = payload.actors().stream()
            .filter(a -> actorId.equals(a.actorId()))
            .findFirst().orElseThrow();
    assertThat(actor.capabilityDimensionScores()).hasSize(1);
    assertThat(actor.capabilityDimensionScores().get(0).capabilityTag())
            .isEqualTo("security-review");
    assertThat(actor.capabilityDimensionScores().get(0).dimension())
            .isEqualTo("thoroughness");
    assertThat(actor.capabilityDimensionScores().get(0).score())
            .isCloseTo(0.9, within(0.001));
}
```

- [ ] **Step 2: Add import test for `capabilityDimensionScores`**

In `TrustImportServiceIT`, add after existing tests:

```java
@Test
@Transactional
void importTrust_seedsCapabilityDimensionScores() {
    final String actorId = "agent-import-cd-" + UUID.randomUUID();
    final Instant now = Instant.now();

    final CapabilityDimensionScoreExport cd = new CapabilityDimensionScoreExport(
            "security-review", "thoroughness", 0.88, 5, now);
    final ActorExport actor = new ActorExport(
            actorId, ActorType.AGENT,
            new GlobalScoreExport(2.0, 1.0, 0.67, 3, 3, 0, now),
            List.of(),
            List.of(),
            List.of(cd));
    final TrustExportPayload payload = new TrustExportPayload(now, "remote", List.of(actor));

    importService.importTrust(payload);

    final var row = trustRepo.findCapabilityDimension(actorId, "security-review", "thoroughness");
    assertThat(row).isPresent();
    assertThat(row.get().trustScore).isCloseTo(0.88, within(0.001));
}
```

- [ ] **Step 3: Run federation tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && \
  JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
    -Dtest="TrustExportServiceIT,TrustImportServiceIT"
```

Expected: BUILD SUCCESS.

---

## Task 14: Final Verification and Commit

- [ ] **Step 1: Run the complete test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS. All tests pass.

- [ ] **Step 2: Check for any remaining `scopeKey` references in production code**

```bash
grep -r "scopeKey" ~/claude/casehub/ledger/runtime/src/main --include="*.java"
grep -r "findByActorIdAndTypeAndKey" ~/claude/casehub/ledger/runtime/src/main --include="*.java"
```

Expected: no output.

- [ ] **Step 3: Commit**

```bash
cd ~/claude/casehub/ledger
git add -p
git commit -m "$(cat <<'EOF'
feat(#76): federation export/import for CAPABILITY_DIMENSION scores

CapabilityDimensionScoreExport record added. ActorExport gains the fourth
field. TrustExportService projects CAPABILITY_DIMENSION rows. JpaTrustImportService
seeds them on import. Full round-trip tested.

Closes #76
EOF
)"
```
