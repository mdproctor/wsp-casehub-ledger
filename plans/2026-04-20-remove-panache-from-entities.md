# Remove Panache from Internal Entities Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove `quarkus-hibernate-orm-panache` from `runtime/pom.xml` by converting all internal entities to plain `@Entity` POJOs with `@NamedQuery` for JPQL validation, and updating repositories to use `EntityManager`.

**Architecture:** Each entity gets `@NamedQuery` annotations (Hibernate validates at boot — typos fail startup, not at query time). Repositories and services use `EntityManager` injected via CDI. `LedgerEntry` was already converted in #16. No consumer impact — these entities are never subclassed externally.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate ORM, `jakarta.persistence.EntityManager`, JUnit 5, AssertJ. All tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`.

**Issues:** Refs #19, Refs #18 (entity + @NamedQuery) → Refs #20, Refs #18 (repositories) → Closes #21, Closes #18 (pom + install)

**Spec:** `docs/superpowers/specs/2026-04-20-remove-panache-from-entities.md`

---

## File Map

**Modified — entities (remove PanacheEntityBase, add @NamedQuery):**
- `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerMerkleFrontier.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntryArchiveRecord.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplement.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ComplianceSupplement.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ProvenanceSupplement.java`

**Modified — repositories/services (EntityManager queries):**
- `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerRetentionJob.java`

**Modified — build:**
- `runtime/pom.xml`

**Created — tests:**
- `runtime/src/test/java/io/casehub/ledger/service/PlainEntityTest.java`

---

## Task 1: TDD — Write PlainEntityTest (Structural + Correctness)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/PlainEntityTest.java`

- [ ] **Step 1: Write the failing structural test**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryArchiveRecord;
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.model.supplement.LedgerSupplement;
import io.casehub.ledger.runtime.model.supplement.ProvenanceSupplement;

/**
 * Structural tests ensuring all entities are plain @Entity POJOs.
 * Fails if any entity re-introduces PanacheEntityBase — prevents regression.
 */
class PlainEntityTest {

    private static final List<Class<?>> ALL_ENTITIES = List.of(
            LedgerEntry.class,           // already converted in #16
            LedgerMerkleFrontier.class,
            LedgerAttestation.class,
            ActorTrustScore.class,
            LedgerEntryArchiveRecord.class,
            LedgerSupplement.class,
            ComplianceSupplement.class,
            ProvenanceSupplement.class
    );

    @Test
    void allEntities_doNotExtendPanacheEntityBase() {
        for (Class<?> entity : ALL_ENTITIES) {
            boolean extendsPanache = false;
            Class<?> c = entity.getSuperclass();
            while (c != null && c != Object.class) {
                if (c.getName().contains("PanacheEntityBase")) {
                    extendsPanache = true;
                    break;
                }
                c = c.getSuperclass();
            }
            assertThat(extendsPanache)
                    .as(entity.getSimpleName() + " must not extend PanacheEntityBase — " +
                        "plain @Entity is required for reactive subclassing compatibility")
                    .isFalse();
        }
    }

    @Test
    void allEntities_haveEntityAnnotation() {
        for (Class<?> entity : ALL_ENTITIES) {
            assertThat(entity.isAnnotationPresent(jakarta.persistence.Entity.class)
                    || entity.isAnnotationPresent(jakarta.persistence.MappedSuperclass.class))
                    .as(entity.getSimpleName() + " must have @Entity or @MappedSuperclass")
                    .isTrue();
        }
    }
}
```

- [ ] **Step 2: Run to verify it FAILS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=PlainEntityTest 2>&1 | tail -8
```

Expected: FAIL — `LedgerMerkleFrontier must not extend PanacheEntityBase` (and others).

- [ ] **Step 3: Commit the failing test**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/PlainEntityTest.java
git commit -m "test(entities): add PlainEntityTest — structural guard against PanacheEntityBase regression

TDD: all assertions fail until entities are converted.

Refs #19, Refs #18"
```

---

## Task 2: Convert LedgerMerkleFrontier

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerMerkleFrontier.java`

- [ ] **Step 1: Rewrite the file**

```java
package io.casehub.ledger.runtime.model;

import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.NamedQuery;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

/**
 * One node in the Merkle Mountain Range frontier for a subject.
 *
 * <p>A subject with N entries has exactly {@code Integer.bitCount(N)} rows at any time —
 * one per set bit in N's binary representation. At 1 million entries: at most 20 rows.
 *
 * <p>Plain {@code @Entity} — no PanacheEntityBase. Queries are defined as {@code @NamedQuery}
 * on this class and executed via {@code EntityManager} in the repository layer.
 */
@Entity
@Table(name = "ledger_merkle_frontier")
@NamedQuery(
        name = "LedgerMerkleFrontier.findBySubjectId",
        query = "SELECT f FROM LedgerMerkleFrontier f WHERE f.subjectId = :subjectId ORDER BY f.level ASC")
@NamedQuery(
        name = "LedgerMerkleFrontier.deleteBySubjectAndLevel",
        query = "DELETE FROM LedgerMerkleFrontier f WHERE f.subjectId = :subjectId AND f.level = :level")
public class LedgerMerkleFrontier {

    @Id
    public UUID id;

    /** The aggregate this frontier node belongs to. */
    @Column(name = "subject_id", nullable = false)
    public UUID subjectId;

    /** Tree level — this node is the root of a perfect subtree of 2^level leaves. */
    @Column(nullable = false)
    public int level;

    /** SHA-256 root hash of this subtree — 64-char lowercase hex. */
    @Column(nullable = false, length = 64)
    public String hash;

    @PrePersist
    void prePersist() {
        if (id == null) id = UUID.randomUUID();
    }
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

Expected: BUILD SUCCESS (JpaLedgerEntryRepository will fail — fixed in Task 6).
If it fails on JpaLedgerEntryRepository referencing the old static methods, that's expected.

- [ ] **Step 3: Run PlainEntityTest for LedgerMerkleFrontier only**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=PlainEntityTest#allEntities_doNotExtendPanacheEntityBase 2>&1 | tail -5
```

The assertion for `LedgerMerkleFrontier` now passes; others still fail.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerMerkleFrontier.java
git commit -m "feat(entities): convert LedgerMerkleFrontier to plain @Entity + @NamedQuery

Refs #19, Refs #18"
```

---

## Task 3: Convert LedgerAttestation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java`

- [ ] **Step 1: Rewrite the file**

```java
package io.casehub.ledger.runtime.model;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.NamedQuery;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

/**
 * A peer attestation stamped onto a {@link LedgerEntry}.
 *
 * <p>Plain {@code @Entity} — queries defined as {@code @NamedQuery} and executed via
 * {@code EntityManager} in {@code JpaLedgerEntryRepository}.
 */
@Entity
@Table(name = "ledger_attestation")
@NamedQuery(
        name = "LedgerAttestation.findByEntryId",
        query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId = :entryId ORDER BY a.occurredAt ASC")
@NamedQuery(
        name = "LedgerAttestation.findBySubjectId",
        query = "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :subjectId ORDER BY a.occurredAt ASC")
@NamedQuery(
        name = "LedgerAttestation.findByEntryIds",
        query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId IN :entryIds")
public class LedgerAttestation {

    /** Primary key — UUID assigned on first persist. */
    @Id
    public UUID id;

    /** The ledger entry being attested. */
    @Column(name = "ledger_entry_id", nullable = false)
    public UUID ledgerEntryId;

    /** Denormalized aggregate identifier for efficient per-subject queries. */
    @Column(name = "subject_id", nullable = false)
    public UUID subjectId;

    /** Identity of the actor providing this attestation. */
    @Column(name = "attestor_id", nullable = false)
    public String attestorId;

    /** Whether the attestor is a human, autonomous agent, or the system itself. */
    @Enumerated(EnumType.STRING)
    @Column(name = "attestor_type", nullable = false)
    public ActorType attestorType;

    /** The functional role of the attestor — e.g. {@code "Auditor"}. Nullable. */
    @Column(name = "attestor_role")
    public String attestorRole;

    /** The attestor's formal verdict on the ledger entry. */
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public AttestationVerdict verdict;

    /** Supporting evidence provided by the attestor. Nullable. */
    @Column(columnDefinition = "TEXT")
    public String evidence;

    /** Confidence level for this attestation, in the range 0.0–1.0. */
    @Column(nullable = false)
    public double confidence;

    /** When this attestation was recorded — set automatically on first persist. */
    @Column(name = "occurred_at", nullable = false)
    public Instant occurredAt;

    @PrePersist
    void prePersist() {
        if (id == null) id = UUID.randomUUID();
        if (occurredAt == null) occurredAt = Instant.now();
    }
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java
git commit -m "feat(entities): convert LedgerAttestation to plain @Entity + @NamedQuery

Refs #19, Refs #18"
```

---

## Task 4: Convert ActorTrustScore

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java`

- [ ] **Step 1: Rewrite the file**

```java
package io.casehub.ledger.runtime.model;

import java.time.Instant;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.NamedQuery;
import jakarta.persistence.Table;

/**
 * Stores the computed EigenTrust-inspired trust score for a decision-making actor.
 *
 * <p>Plain {@code @Entity} — queries defined as {@code @NamedQuery} and executed via
 * {@code EntityManager} in {@code JpaActorTrustScoreRepository}.
 */
@Entity
@Table(name = "actor_trust_score")
@NamedQuery(
        name = "ActorTrustScore.findAll",
        query = "SELECT s FROM ActorTrustScore s")
public class ActorTrustScore {

    /** Primary key — the actor's identity string. */
    @Id
    @Column(name = "actor_id")
    public String actorId;

    /** Whether this actor is a human, autonomous agent, or the system itself. */
    @Enumerated(EnumType.STRING)
    @Column(name = "actor_type")
    public ActorType actorType;

    /** Computed trust score in the range [0.0, 1.0]. Neutral prior is 0.5. */
    @Column(name = "trust_score")
    public double trustScore;

    /** Total number of EVENT-type ledger entries attributed to this actor. */
    @Column(name = "decision_count")
    public int decisionCount;

    /** Number of decisions that received at least one negative attestation. */
    @Column(name = "overturned_count")
    public int overturnedCount;

    /** Number of appeal events attributed to this actor (reserved for future use). */
    @Column(name = "appeal_count")
    public int appealCount;

    /** Total count of positive attestations (SOUND or ENDORSED) across all decisions. */
    @Column(name = "attestation_positive")
    public int attestationPositive;

    /** Total count of negative attestations (FLAGGED or CHALLENGED) across all decisions. */
    @Column(name = "attestation_negative")
    public int attestationNegative;

    /** When this score was last computed. */
    @Column(name = "last_computed_at")
    public Instant lastComputedAt;
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java
git commit -m "feat(entities): convert ActorTrustScore to plain @Entity + @NamedQuery

Refs #19, Refs #18"
```

---

## Task 5: Convert LedgerEntryArchiveRecord + Supplement Stack

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntryArchiveRecord.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplement.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ComplianceSupplement.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ProvenanceSupplement.java`

These entities have no explicit query methods — only `@PrePersist` and cascade persistence. Change is minimal: remove `extends PanacheEntityBase` and the import.

- [ ] **Step 1: Edit LedgerEntryArchiveRecord**

Read the file. Remove:
```java
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
```
Change:
```java
public class LedgerEntryArchiveRecord extends PanacheEntityBase {
```
To:
```java
public class LedgerEntryArchiveRecord {
```

- [ ] **Step 2: Edit LedgerSupplement**

Read the file. Remove the `PanacheEntityBase` import and `extends PanacheEntityBase`. Keep all fields, `@Entity`, `@Inheritance`, `@DiscriminatorColumn`, `@ManyToOne`, `@PrePersist`.

- [ ] **Step 3: Edit ComplianceSupplement**

Read the file. Remove `PanacheEntityBase` import. If `ComplianceSupplement extends LedgerSupplement`, no change to extends clause — just remove the Panache import if it has one independently.

- [ ] **Step 4: Edit ProvenanceSupplement**

Same as ComplianceSupplement — remove any Panache import.

- [ ] **Step 5: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

Expected: BUILD SUCCESS. (JpaLedgerEntryRepository still has old calls — that's fine, compile-time is checking model only.)

- [ ] **Step 6: Run PlainEntityTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=PlainEntityTest 2>&1 | tail -8
```

Expected: Both PlainEntityTest tests PASS — all 8 entity classes are now plain @Entity.

- [ ] **Step 7: Commit**

```bash
git add \
  runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntryArchiveRecord.java \
  runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplement.java \
  runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ComplianceSupplement.java \
  runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ProvenanceSupplement.java
git commit -m "feat(entities): convert LedgerEntryArchiveRecord and supplement stack to plain @Entity

No query changes needed — cascade handles persistence.
PlainEntityTest now fully green.

Closes #19, Refs #18"
```

---

## Task 6: Update JpaLedgerEntryRepository — Attestation + Frontier via EntityManager

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`

- [ ] **Step 1: Read the current file**

Identify all remaining Panache active-record calls:
- `LedgerAttestation.list("ledgerEntryId = ?1 ORDER BY occurredAt ASC", ...)` → named query
- `LedgerAttestation.list("ledgerEntryId IN ?1", entryIds)` → named query
- `attestation.persist()` in `saveAttestation()` → `em.persist(attestation)`
- `LedgerMerkleFrontier.findBySubjectId(entry.subjectId)` → named query
- `LedgerMerkleFrontier.deleteBySubjectAndLevel(...)` → named query (DELETE)
- `node.persist()` → `em.persist(node)`

- [ ] **Step 2: Update saveAttestation()**

Replace:
```java
@Override
public LedgerAttestation saveAttestation(final LedgerAttestation attestation) {
    attestation.persist();
    return attestation;
}
```

With:
```java
@Override
@Transactional
public LedgerAttestation saveAttestation(final LedgerAttestation attestation) {
    em.persist(attestation);
    return attestation;
}
```

- [ ] **Step 3: Update findAttestationsByEntryId()**

Replace:
```java
return LedgerAttestation.list("ledgerEntryId = ?1 ORDER BY occurredAt ASC", ledgerEntryId);
```

With:
```java
return em.createNamedQuery("LedgerAttestation.findByEntryId", LedgerAttestation.class)
        .setParameter("entryId", ledgerEntryId)
        .getResultList();
```

- [ ] **Step 4: Update findAttestationsForEntries()**

Replace:
```java
final List<LedgerAttestation> all = LedgerAttestation.list("ledgerEntryId IN ?1", entryIds);
```

With:
```java
final List<LedgerAttestation> all = em.createNamedQuery("LedgerAttestation.findByEntryIds", LedgerAttestation.class)
        .setParameter("entryIds", entryIds)
        .getResultList();
```

- [ ] **Step 5: Update frontier operations in save()**

Replace:
```java
final List<LedgerMerkleFrontier> currentFrontier =
        LedgerMerkleFrontier.findBySubjectId(entry.subjectId);
```
With:
```java
final List<LedgerMerkleFrontier> currentFrontier =
        em.createNamedQuery("LedgerMerkleFrontier.findBySubjectId", LedgerMerkleFrontier.class)
                .setParameter("subjectId", entry.subjectId)
                .getResultList();
```

Replace each:
```java
LedgerMerkleFrontier.deleteBySubjectAndLevel(entry.subjectId, old.level);
```
With:
```java
em.createNamedQuery("LedgerMerkleFrontier.deleteBySubjectAndLevel")
        .setParameter("subjectId", entry.subjectId)
        .setParameter("level", old.level)
        .executeUpdate();
```

Replace:
```java
node.persist();
```
With:
```java
em.persist(node);
```

- [ ] **Step 6: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Run attestation + verification ITs**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="AuditQueryIT,LedgerVerificationServiceIT,LedgerProvExportServiceIT" 2>&1 | tail -8
```

Expected: all pass (attestation queries + frontier operations correct via named queries).

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java
git commit -m "feat(repo): migrate JpaLedgerEntryRepository attestation+frontier to EntityManager @NamedQuery

Refs #20, Refs #18"
```

---

## Task 7: Update JpaActorTrustScoreRepository

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java`

- [ ] **Step 1: Write the updated class**

```java
package io.casehub.ledger.runtime.repository.jpa;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;

/**
 * JPA/EntityManager implementation of {@link ActorTrustScoreRepository}.
 *
 * <p>Upsert is implemented as find-then-persist/update: if the actor's score row exists,
 * the managed entity is updated in-place (dirty-checked by JPA); otherwise a new row is
 * inserted. No database-specific SQL required.
 */
@ApplicationScoped
public class JpaActorTrustScoreRepository implements ActorTrustScoreRepository {

    @Inject
    EntityManager em;

    /** {@inheritDoc} */
    @Override
    public Optional<ActorTrustScore> findByActorId(final String actorId) {
        return Optional.ofNullable(em.find(ActorTrustScore.class, actorId));
    }

    /** {@inheritDoc} */
    @Override
    @Transactional
    public void upsert(final String actorId, final ActorType actorType, final double trustScore,
            final int decisionCount, final int overturnedCount, final int appealCount,
            final int attestationPositive, final int attestationNegative,
            final Instant lastComputedAt) {

        ActorTrustScore score = em.find(ActorTrustScore.class, actorId);
        final boolean isNew = (score == null);
        if (isNew) {
            score = new ActorTrustScore();
            score.actorId = actorId;
        }
        score.actorType = actorType;
        score.trustScore = trustScore;
        score.decisionCount = decisionCount;
        score.overturnedCount = overturnedCount;
        score.appealCount = appealCount;
        score.attestationPositive = attestationPositive;
        score.attestationNegative = attestationNegative;
        score.lastComputedAt = lastComputedAt;
        if (isNew) {
            em.persist(score);
        }
        // existing entity is managed — JPA dirty-checks and flushes on commit
    }

    /** {@inheritDoc} */
    @Override
    public List<ActorTrustScore> findAll() {
        return em.createNamedQuery("ActorTrustScore.findAll", ActorTrustScore.class)
                .getResultList();
    }
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

- [ ] **Step 3: Run trust score ITs**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="TrustScoreComputerTest,TrustScoreForgivenessIT" 2>&1 | tail -8
```

Expected: all pass.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorTrustScoreRepository.java
git commit -m "feat(repo): migrate JpaActorTrustScoreRepository to EntityManager + @NamedQuery

Refs #20, Refs #18"
```

---

## Task 8: Update LedgerRetentionJob — record.persist() → em.persist()

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerRetentionJob.java`

- [ ] **Step 1: Find and replace record.persist()**

Read the file. Find the line:
```java
record.persist();
```
Replace with:
```java
em.persist(record);
```

`LedgerRetentionJob` already has `@Inject EntityManager entityManager` from the #16 refactor. Use that same field — the variable may be named `entityManager` not `em`. Check and use the correct field name.

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

- [ ] **Step 3: Run retention ITs**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerRetentionJobIT 2>&1 | tail -8
```

Expected: 6/6 pass.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerRetentionJob.java
git commit -m "feat(service): migrate LedgerRetentionJob archive record to em.persist()

Closes #20, Refs #18"
```

---

## Task 9: Full Suite — Verify All Tests Pass

- [ ] **Step 1: Run the complete runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -20
```

Expected: BUILD SUCCESS, all 130+ tests pass (129 existing + 2 new PlainEntityTest).

- [ ] **Step 2: If any test fails, debug before proceeding**

Common issues:
- `@NamedQuery` typo → Quarkus fails to start with `QuerySyntaxException` — fix the query string in the entity
- `em.find()` returns null unexpectedly → check `@Id` type matches (String for ActorTrustScore, UUID for others)
- `executeUpdate()` on DELETE named query requires `@Transactional` — ensure `save()` in `JpaLedgerEntryRepository` is still `@Transactional`

---

## Task 10: Remove quarkus-hibernate-orm-panache from pom.xml

**Files:**
- Modify: `runtime/pom.xml`

- [ ] **Step 1: Check what quarkus-hibernate-orm-panache provides transitively**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn dependency:tree -pl runtime | grep "hibernate-orm"
```

Verify `quarkus-hibernate-orm` is already present (directly or as transitive dep of `quarkus-flyway` or similar). If NOT present, add it explicitly before removing panache.

- [ ] **Step 2: Remove the panache dependency**

Read `runtime/pom.xml`. Remove the block:
```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>
```

If `quarkus-hibernate-orm` is not already present, add:
```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-hibernate-orm</artifactId>
</dependency>
```

- [ ] **Step 3: Compile to verify no missing classes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

If compilation fails with missing classes from Panache, you missed a usage. Search:
```bash
grep -rn "PanacheEntityBase\|PanacheRepository\|panache" runtime/src/main --include="*.java"
```

Fix any remaining references.

- [ ] **Step 4: Full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Full install (all modules + examples)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS. JAR installed to `~/.m2/repository/io/casehub/ledger/`.

- [ ] **Step 6: Verify dep is gone from effective pom**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn dependency:tree -pl runtime | grep panache
```

Expected: no output (panache no longer in tree).

- [ ] **Step 7: Commit and close issues**

```bash
git add runtime/pom.xml
git commit -m "feat(build): remove quarkus-hibernate-orm-panache from runtime pom

All entities are plain @Entity. Repositories use EntityManager.
No Panache in the model layer — consumers choose their own ORM strategy.

Closes #21, Closes #18"
```

```bash
gh issue close 18 --repo casehubio/ledger \
  --comment "All done: plain @Entity throughout, panache dep removed, 130+ tests passing."
```
