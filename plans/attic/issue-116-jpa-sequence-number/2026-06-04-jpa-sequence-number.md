# JPA Sequence Number Assignment — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix `JpaLedgerEntryRepository.save()` to assign `sequenceNumber` atomically per subject before persist, using a dedicated sequence table with native UPSERT.

**Architecture:** A `ledger_subject_sequence` table acts as the per-subject counter (analogous to in-memory `ConcurrentHashMap<UUID, AtomicInteger>`). A native SQL UPSERT atomically allocates the next sequence number within the existing `@Transactional`. Assignment happens before `leafHash` and `em.persist()` so enrichers and the Merkle chain see the correct value.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM 6, H2 2.4.240 (MODE=PostgreSQL), Flyway, JUnit 5 + AssertJ

**Spec:** `specs/issue-116-jpa-sequence-number/2026-06-04-jpa-sequence-number-design.md`

---

## File Map

| Action | File | Responsibility |
|--------|------|---------------|
| Modify | `runtime/src/main/resources/db/ledger/migration/V1000__ledger_base_schema.sql` | Add UNIQUE index + sequence table |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java` | Add `nextSequenceNumber()` + null guard in `save()` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java` | Update `save()` Javadoc |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/ReactiveLedgerEntryRepository.java` | Update `save()` Javadoc |
| Create | `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java` | All sequence number integration tests |

---

### Task 0: H2 Syntax Spike

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/H2UpsertSpikeIT.java`

This gates everything. If the native SQL doesn't work on H2, we switch to the MERGE fallback before proceeding.

- [ ] **Step 1: Write the spike test**

```java
package io.casehub.ledger.runtime.repository.jpa;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.quarkus.test.junit.QuarkusTest;

/**
 * Spike: verify INSERT ON CONFLICT DO UPDATE RETURNING executes on H2 MODE=PostgreSQL.
 * If this fails, switch to MERGE INTO fallback per spec.
 */
@QuarkusTest
class H2UpsertSpikeIT {

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @Test
    @Transactional
    void upsertReturningWorksOnH2() {
        UUID subjectId = UUID.randomUUID();

        // First call — INSERT path: should return 1
        Number first = (Number) em.createNativeQuery(
                "INSERT INTO ledger_subject_sequence (subject_id, next_seq) VALUES (?1, 2) " +
                "ON CONFLICT (subject_id) DO UPDATE SET next_seq = ledger_subject_sequence.next_seq + 1 " +
                "RETURNING next_seq - 1")
                .setParameter(1, subjectId)
                .getSingleResult();
        assertThat(first.intValue()).isEqualTo(1);

        // Second call — UPDATE path: should return 2
        Number second = (Number) em.createNativeQuery(
                "INSERT INTO ledger_subject_sequence (subject_id, next_seq) VALUES (?1, 2) " +
                "ON CONFLICT (subject_id) DO UPDATE SET next_seq = ledger_subject_sequence.next_seq + 1 " +
                "RETURNING next_seq - 1")
                .setParameter(1, subjectId)
                .getSingleResult();
        assertThat(second.intValue()).isEqualTo(2);

        // Third call — should return 3
        Number third = (Number) em.createNativeQuery(
                "INSERT INTO ledger_subject_sequence (subject_id, next_seq) VALUES (?1, 2) " +
                "ON CONFLICT (subject_id) DO UPDATE SET next_seq = ledger_subject_sequence.next_seq + 1 " +
                "RETURNING next_seq - 1")
                .setParameter(1, subjectId)
                .getSingleResult();
        assertThat(third.intValue()).isEqualTo(3);
    }
}
```

- [ ] **Step 2: Add the sequence table to V1000 (required for spike to run)**

Edit `runtime/src/main/resources/db/ledger/migration/V1000__ledger_base_schema.sql`.

Change line 41:
```sql
-- Before:
CREATE INDEX idx_ledger_entry_subject_seq ON ledger_entry (subject_id, sequence_number);
-- After:
CREATE UNIQUE INDEX idx_ledger_entry_subject_seq ON ledger_entry (subject_id, sequence_number);
```

Add at the end of the file (after the `ledger_merkle_frontier` section):
```sql
CREATE TABLE ledger_subject_sequence (
    subject_id UUID NOT NULL PRIMARY KEY,
    next_seq   INT  NOT NULL DEFAULT 1
);
```

- [ ] **Step 3: Run the spike**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="H2UpsertSpikeIT" -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — all three assertions succeed.

**If it fails** with "not a query" or syntax error: switch to the MERGE fallback. Replace the UPSERT SQL with:
```sql
MERGE INTO ledger_subject_sequence AS t
USING (VALUES (?1)) AS s(subject_id)
ON t.subject_id = s.subject_id
WHEN MATCHED THEN UPDATE SET next_seq = t.next_seq + 1
WHEN NOT MATCHED THEN INSERT (subject_id, next_seq) VALUES (s.subject_id, 2)
```
and add a separate SELECT:
```sql
SELECT next_seq - 1 FROM ledger_subject_sequence WHERE subject_id = ?1
```

If MERGE USING also fails, try H2-native:
```sql
MERGE INTO ledger_subject_sequence KEY(subject_id) VALUES (?1, 2)
```
with SELECT. Document whichever works in a code comment.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/H2UpsertSpikeIT.java \
       runtime/src/main/resources/db/ledger/migration/V1000__ledger_base_schema.sql
git commit -m "test(#116): H2 UPSERT RETURNING spike + V1000 schema (UNIQUE index, sequence table)"
```

---

### Task 1: Implement nextSequenceNumber in JpaLedgerEntryRepository

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java:88-121`

**Depends on:** Task 0 (need to know which SQL syntax works)

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java`:

```java
package io.casehub.ledger.runtime.repository.jpa;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

@QuarkusTest
@TestProfile(JpaSequenceNumberIT.Profile.class)
class JpaSequenceNumberIT {

    public static class Profile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "sequence-number-test";
        }
    }

    @Inject
    LedgerEntryRepository repo;

    @Inject
    EntityManager em;

    private TestEntry newEntry(UUID subjectId) {
        TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "test-actor";
        e.actorType = ActorType.SYSTEM;
        e.actorRole = "tester";
        return e;
    }

    @Test
    @Transactional
    void assignsContiguousSequenceNumbers() {
        UUID subjectId = UUID.randomUUID();

        LedgerEntry e1 = repo.save(newEntry(subjectId));
        LedgerEntry e2 = repo.save(newEntry(subjectId));
        LedgerEntry e3 = repo.save(newEntry(subjectId));

        assertThat(e1.sequenceNumber).isEqualTo(1);
        assertThat(e2.sequenceNumber).isEqualTo(2);
        assertThat(e3.sequenceNumber).isEqualTo(3);
    }

    @Test
    @Transactional
    void findBySubjectIdReturnsInSequenceOrder() {
        UUID subjectId = UUID.randomUUID();

        repo.save(newEntry(subjectId));
        repo.save(newEntry(subjectId));
        repo.save(newEntry(subjectId));

        List<LedgerEntry> entries = repo.findBySubjectId(subjectId);
        assertThat(entries).extracting(e -> e.sequenceNumber)
                .containsExactly(1, 2, 3);
    }

    @Test
    @Transactional
    void multiSubjectIsolation() {
        UUID subject1 = UUID.randomUUID();
        UUID subject2 = UUID.randomUUID();

        LedgerEntry s1e1 = repo.save(newEntry(subject1));
        LedgerEntry s2e1 = repo.save(newEntry(subject2));
        LedgerEntry s1e2 = repo.save(newEntry(subject1));
        LedgerEntry s2e2 = repo.save(newEntry(subject2));

        assertThat(s1e1.sequenceNumber).isEqualTo(1);
        assertThat(s1e2.sequenceNumber).isEqualTo(2);
        assertThat(s2e1.sequenceNumber).isEqualTo(1);
        assertThat(s2e2.sequenceNumber).isEqualTo(2);
    }

    @Test
    @Transactional
    void overwritesCallerSetSequenceNumber() {
        UUID subjectId = UUID.randomUUID();
        TestEntry entry = newEntry(subjectId);
        entry.sequenceNumber = 999;

        LedgerEntry saved = repo.save(entry);

        assertThat(saved.sequenceNumber).isEqualTo(1);
    }

    @Test
    void rejectsNullSubjectId() {
        TestEntry entry = newEntry(null);
        entry.subjectId = null;

        assertThatThrownBy(() -> repo.save(entry))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("subjectId");
    }
}
```

- [ ] **Step 2: Add the test profile to application.properties**

Append to `runtime/src/test/resources/application.properties`:
```properties
# Sequence number test profile (used by JpaSequenceNumberIT, H2UpsertSpikeIT)
%sequence-number-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:sequencenumbertestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaSequenceNumberIT" -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — `assignsContiguousSequenceNumbers` fails because `sequenceNumber` is 0 for all entries.

- [ ] **Step 4: Implement nextSequenceNumber and update save()**

Edit `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`.

Add the `nextSequenceNumber` method (after the `updateMerkleFrontier` method):

```java
/**
 * Atomically allocates the next sequence number for the given subject.
 * Uses INSERT ON CONFLICT DO UPDATE on the ledger_subject_sequence table.
 * The UPSERT and the entry INSERT share the same @Transactional — rollback
 * reverts both, preserving contiguity.
 */
private int nextSequenceNumber(final UUID subjectId) {
    // next_seq stores the NEXT value to allocate. First insert sets next_seq=2
    // (allocating 1). Subsequent calls increment and return the pre-increment value.
    final Number nextSeq = (Number) em.createNativeQuery(
            "INSERT INTO ledger_subject_sequence (subject_id, next_seq) VALUES (?1, 2) " +
            "ON CONFLICT (subject_id) DO UPDATE SET next_seq = ledger_subject_sequence.next_seq + 1 " +
            "RETURNING next_seq - 1")
            .setParameter(1, subjectId)
            .getSingleResult();
    return nextSeq.intValue();
}
```

Update `save()` — add null guard and sequence assignment. Replace lines 88-121 with:

```java
/** {@inheritDoc} */
@Override
@Transactional
public LedgerEntry save(final LedgerEntry entry) {
    if (entry.subjectId == null) {
        throw new IllegalArgumentException("LedgerEntry.subjectId must not be null");
    }
    if (entry.occurredAt == null) {
        entry.occurredAt = Instant.now();
    }

    if (entry.actorId != null) {
        entry.actorId = actorIdentityProvider.tokenise(entry.actorId);
    }

    entry.compliance().ifPresent(cs -> {
        if (cs.decisionContext != null) {
            cs.decisionContext = decisionContextSanitiser.sanitise(cs.decisionContext);
            entry.refreshSupplementJson();
        }
    });

    entry.sequenceNumber = nextSequenceNumber(entry.subjectId);

    if (ledgerConfig.hashChain().enabled()) {
        entry.digest = LedgerMerkleTree.leafHash(entry);
    }
    em.persist(entry);

    if (ledgerConfig.hashChain().enabled()) {
        updateMerkleFrontier(entry);
    }

    return entry;
}
```

- [ ] **Step 5: Run the sequence number tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaSequenceNumberIT" -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — all 5 tests pass.

- [ ] **Step 6: Run full test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: All existing tests pass. The V1000 schema change (UNIQUE index) and new sequence table should not break any existing test — the tests use isolated H2 databases per profile.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java \
       runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java \
       runtime/src/test/resources/application.properties
git commit -m "fix(#116): JpaLedgerEntryRepository assigns sequenceNumber via atomic UPSERT

Adds nextSequenceNumber() using INSERT ON CONFLICT DO UPDATE RETURNING
on ledger_subject_sequence table. Assignment happens before leafHash and
em.persist() so enrichers and Merkle chain see the correct value.

Refs #116"
```

---

### Task 2: SPI Javadoc Updates

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java:24-31`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/ReactiveLedgerEntryRepository.java` (save() Javadoc)

- [ ] **Step 1: Update LedgerEntryRepository.save() Javadoc**

Edit `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java`. Replace lines 24-30 with:

```java
    /**
     * Persist a new ledger entry with automatic sequence number assignment.
     *
     * <p>The repository assigns {@code sequenceNumber} based on the entry's
     * {@code subjectId} — any value set by the caller is overwritten. Sequence
     * numbers are monotonically increasing and contiguous on insert within
     * committed transactions. Retention deletion may remove entries from the
     * start of the sequence without affecting the contiguity invariant.
     *
     * @param entry the entry to persist; {@code subjectId} must not be {@code null}
     * @return the persisted entry with {@code sequenceNumber} assigned
     */
```

- [ ] **Step 2: Update ReactiveLedgerEntryRepository.save() Javadoc**

Find `ReactiveLedgerEntryRepository.java` and update the `save()` Javadoc to match:

```java
    /**
     * Persist a new ledger entry with automatic sequence number assignment.
     *
     * <p>The repository assigns {@code sequenceNumber} based on the entry's
     * {@code subjectId} — any value set by the caller is overwritten. Sequence
     * numbers are monotonically increasing and contiguous on insert within
     * committed transactions.
     *
     * @param entry the entry to persist; {@code subjectId} must not be {@code null}
     * @return the persisted entry with {@code sequenceNumber} assigned
     */
```

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java \
       runtime/src/main/java/io/casehub/ledger/runtime/repository/ReactiveLedgerEntryRepository.java
git commit -m "docs(#116): update save() SPI contract — sequenceNumber is repo-assigned, contiguous on insert

Refs #116"
```

---

### Task 3: UNIQUE Constraint and Health Job Integration Tests

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java` (add tests)

- [ ] **Step 1: Add UNIQUE constraint enforcement test**

Add to `JpaSequenceNumberIT.java`:

```java
@Test
@Transactional
void uniqueConstraintPreventsduplicateSequenceNumber() {
    UUID subjectId = UUID.randomUUID();
    repo.save(newEntry(subjectId)); // gets seqNum=1

    // Bypass the repository and try to insert a second entry with seqNum=1
    assertThatThrownBy(() -> {
        em.createNativeQuery(
                "INSERT INTO ledger_entry (id, dtype, subject_id, sequence_number, entry_type, occurred_at) " +
                "VALUES (?1, 'TEST', ?2, 1, 'EVENT', CURRENT_TIMESTAMP)")
                .setParameter(1, UUID.randomUUID())
                .setParameter(2, subjectId)
                .executeUpdate();
        em.flush();
    }).hasMessageContaining("IDX_LEDGER_ENTRY_SUBJECT_SEQ");
}
```

- [ ] **Step 2: Add health job integration test**

Add to `JpaSequenceNumberIT.java` (requires injecting `LedgerHealthJob` or reproducing its gap query):

```java
@Test
@Transactional
void healthJobDetectsNoGapsInJpaAssignedSequences() {
    UUID subjectId = UUID.randomUUID();
    for (int i = 0; i < 5; i++) {
        repo.save(newEntry(subjectId));
    }

    // Reproduce the health job's gap detection query
    List<?> gapResults = em.createQuery(
            "SELECT e.subjectId FROM LedgerEntry e " +
            "WHERE e.subjectId = :sid " +
            "GROUP BY e.subjectId " +
            "HAVING COUNT(e) != MAX(e.sequenceNumber) - MIN(e.sequenceNumber) + 1")
            .setParameter("sid", subjectId)
            .getResultList();

    assertThat(gapResults).isEmpty();
}
```

**Note on spec Test 6 (agent signature):** The agent signature enricher calls `canonicalBytes()` — the same input as `leafHash()`. The leafHash test below proves `sequenceNumber` was correct at `canonicalBytes()` computation time, which indirectly validates signature correctness. A direct signature test requires a signing-enabled profile with key configuration — orthogonal to #116 and already tested by existing signing ITs once sequenceNumber is correct.

- [ ] **Step 3: Add leafHash correctness test**

Add to `JpaSequenceNumberIT.java`:

```java
@Test
@Transactional
void leafHashCoversCorrectSequenceNumber() {
    UUID subjectId = UUID.randomUUID();
    LedgerEntry saved = repo.save(newEntry(subjectId));

    // Recompute leafHash from the persisted entry's fields
    String recomputed = LedgerMerkleTree.leafHash(saved);

    assertThat(saved.digest).isEqualTo(recomputed);
    assertThat(saved.sequenceNumber).isEqualTo(1);
}
```

Add import: `import io.casehub.ledger.runtime.service.LedgerMerkleTree;`

- [ ] **Step 4: Run all sequence tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaSequenceNumberIT" -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — all 8 tests pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java
git commit -m "test(#116): UNIQUE constraint, health job gap detection, leafHash correctness

Refs #116"
```

---

### Task 4: KeyRotationEntry Subclass Test

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java` (add test)

- [ ] **Step 1: Add KeyRotationEntry sequence test**

Add to `JpaSequenceNumberIT.java`:

```java
@Test
@Transactional
void keyRotationEntryGetsCorrectSequenceNumber() {
    String actorId = "claude:reviewer@v1";
    UUID subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(java.nio.charset.StandardCharsets.UTF_8));

    // Persist a KeyRotationEntry through the repo
    KeyRotationEntry rotation = new KeyRotationEntry();
    rotation.subjectId = subjectId;
    rotation.entryType = LedgerEntryType.EVENT;
    rotation.actorId = actorId;
    rotation.actorType = ActorType.AGENT;
    rotation.actorRole = "reviewer";
    rotation.previousKeyRef = "old-key-ref";
    rotation.newKeyRef = "new-key-ref";
    rotation.reason = KeyRotationReason.SCHEDULED;
    rotation.effectiveSince = Instant.now();

    LedgerEntry saved = repo.save(rotation);

    assertThat(saved.sequenceNumber).isEqualTo(1);
    assertThat(saved).isInstanceOf(KeyRotationEntry.class);
}
```

Add imports:
```java
import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.ledger.api.model.KeyRotationReason;
import java.time.Instant;
import java.nio.charset.StandardCharsets;
```

- [ ] **Step 2: Run the test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaSequenceNumberIT#keyRotationEntryGetsCorrectSequenceNumber" -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: All tests pass. No regressions.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java
git commit -m "test(#116): KeyRotationEntry subclass gets correct sequenceNumber

Refs #116"
```

---

### Task 5: Clean Up Spike Test

**Files:**
- Delete: `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/H2UpsertSpikeIT.java`

The spike test validated H2 compatibility. Its assertions are now covered by `JpaSequenceNumberIT` which exercises the same SQL through the production code path. Remove the spike to avoid redundant test maintenance.

- [ ] **Step 1: Delete the spike test**

```bash
rm runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/H2UpsertSpikeIT.java
```

- [ ] **Step 2: Run full test suite to confirm no dependency on it**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: All tests pass.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/H2UpsertSpikeIT.java
git commit -m "chore(#116): remove H2 UPSERT spike — covered by JpaSequenceNumberIT

Refs #116"
```

---

### Task 6: Final Verification

- [ ] **Step 1: Full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

Expected: BUILD SUCCESS across all modules (api, runtime, deployment, persistence-memory).

- [ ] **Step 2: Verify no regressions in persistence-memory module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`

Expected: All in-memory tests pass. The V1000 schema change doesn't affect the in-memory module (it doesn't use Flyway).

- [ ] **Step 3: Verify test count**

```bash
grep -r "Tests run:" runtime/target/surefire-reports/*.txt | tail -5
```

Confirm the new IT tests appear in the report.
