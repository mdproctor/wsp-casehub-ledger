# capabilityTag on LedgerAttestation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `capabilityTag` to `LedgerAttestation` with a `"*"` global sentinel, three new SPI query methods (blocking + reactive parity), and full test coverage — enabling capability-scoped trust scores in B2 (#61).

**Architecture:** `CapabilityTag.GLOBAL = "*"` constant in the api module; `capabilityTag` field on the `@MappedSuperclass` (defaults to `"*"`); three `@NamedQuery` entries on the runtime entity; three new methods on both SPI interfaces; JPA implementation follows the existing `EntityManager` + named-query pattern. Schema is rewritten in V1000 (no installed instances).

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM (EntityManager + `@NamedQuery`), H2 (test), Jakarta Persistence, AssertJ, JUnit 5.

**Issues:** Closes #60 · Part of epic #50

---

## Pre-flight

Before every task, verify both IntelliJ MCPs are available:
- `mcp__intellij__get_project_modules` with `projectPath=/Users/mdproctor/claude/casehub/ledger`
- `mcp__intellij-index__ide_index_status` with `project_path=/Users/mdproctor/claude/casehub/ledger`

If either fails, stop and report. Do not use bash as a substitute for IntelliJ operations — always check if IntelliJ can perform the action first.

---

## File Map

| Action | File | What changes |
|---|---|---|
| Create | `api/src/main/java/io/casehub/ledger/api/model/CapabilityTag.java` | New constant class |
| Create | `api/src/test/java/io/casehub/ledger/api/model/CapabilityTagTest.java` | Unit tests for constant + field default |
| Modify | `api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java` | Add `capabilityTag` field |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java` | Add 3 `@NamedQuery` annotations |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java` | Add 3 SPI methods |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/ReactiveLedgerEntryRepository.java` | Add 3 reactive mirror methods |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java` | Implement 3 methods |
| Modify | `runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql` | Add `capability_tag` column + 2 indexes |
| Create | `runtime/src/test/java/io/casehub/ledger/service/LedgerAttestationCapabilityIT.java` | Integration tests |
| Modify | `runtime/src/test/java/io/casehub/ledger/service/LedgerTestFixtures.java` | Add capabilityTag overload |
| Modify | `docs/DESIGN.md` | Update Implementation Tracker |
| Modify | `docs/DESIGN-capabilities.md` | Add capabilityTag to trust scoring notes |
| Modify | `CLAUDE.md` | Add `CapabilityTag.java` to project structure |

---

### Task 1: CapabilityTag constant class

**Files:**
- Create: `api/src/main/java/io/casehub/ledger/api/model/CapabilityTag.java`

- [ ] **Step 1: Create the constant class**

```java
package io.casehub.ledger.api.model;

/**
 * Well-known capability tag sentinels for {@link LedgerAttestation#capabilityTag}.
 */
public final class CapabilityTag {

    /** Attestation applies to all capabilities — the default. */
    public static final String GLOBAL = "*";

    private CapabilityTag() {
    }
}
```

- [ ] **Step 2: Build api module to verify compilation**

Prefer IntelliJ: `mcp__intellij__build_project` with `projectPath=/Users/mdproctor/claude/casehub/ledger`, `filesToRebuild=["api/src/main/java/io/casehub/ledger/api/model/CapabilityTag.java"]`.

Fallback:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn compile -pl api -q
```
Expected: BUILD SUCCESS, no errors.

- [ ] **Step 3: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/model/CapabilityTag.java
git commit -m "feat: add CapabilityTag constant — GLOBAL = \"*\" sentinel

Refs #60 · Part of epic #50"
```

---

### Task 2: Unit tests for constant + field default (TDD — write before the field)

**Files:**
- Create: `api/src/test/java/io/casehub/ledger/api/model/CapabilityTagTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.ledger.api.model;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

/**
 * Unit tests for {@link CapabilityTag} constant and {@link LedgerAttestation#capabilityTag} default.
 * Pure JUnit 5 — no Quarkus runtime needed.
 */
class CapabilityTagTest {

    // ── Constant correctness ──────────────────────────────────────────────────

    @Test
    void global_constant_value_is_star() {
        assertThat(CapabilityTag.GLOBAL).isEqualTo("*");
    }

    // ── Field default ─────────────────────────────────────────────────────────

    @Test
    void ledgerAttestation_capabilityTag_defaultsToGlobal() {
        // LedgerAttestation is a @MappedSuperclass — instantiable as plain Java for field tests.
        final LedgerAttestation att = new LedgerAttestation();
        assertThat(att.capabilityTag)
                .as("capabilityTag must default to GLOBAL, never null")
                .isEqualTo(CapabilityTag.GLOBAL);
    }

    @Test
    void ledgerAttestation_capabilityTag_isNotNullByDefault() {
        assertThat(new LedgerAttestation().capabilityTag).isNotNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Prefer IntelliJ: open `CapabilityTagTest`, click the run button (or use `mcp__intellij__execute_run_configuration` if a run config exists).

Fallback:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl api -Dtest=CapabilityTagTest -q 2>&1 | tail -20
```
Expected: FAIL — `capabilityTag` field does not exist yet.

- [ ] **Step 3: Add `capabilityTag` field to `api/model/LedgerAttestation.java`**

Add after the `occurredAt` field (line 48):

```java
@Column(name = "capability_tag", nullable = false)
public String capabilityTag = CapabilityTag.GLOBAL;
```

Full updated file:
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
 * A peer attestation stamped onto a {@link LedgerEntry}.
 */
@MappedSuperclass
public class LedgerAttestation {

    @Id
    public UUID id;

    @Column(name = "ledger_entry_id", nullable = false)
    public UUID ledgerEntryId;

    @Column(name = "subject_id", nullable = false)
    public UUID subjectId;

    @Column(name = "attestor_id", nullable = false)
    public String attestorId;

    @Enumerated(EnumType.STRING)
    @Column(name = "attestor_type", nullable = false)
    public ActorType attestorType;

    @Column(name = "attestor_role")
    public String attestorRole;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public AttestationVerdict verdict;

    @Column(columnDefinition = "TEXT")
    public String evidence;

    @Column(nullable = false)
    public double confidence;

    @Column(name = "occurred_at", nullable = false)
    public Instant occurredAt;

    @Column(name = "capability_tag", nullable = false)
    public String capabilityTag = CapabilityTag.GLOBAL;
}
```

- [ ] **Step 4: Run test to verify it passes**

Same command as Step 2.
Expected: BUILD SUCCESS, 3 tests PASSED.

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java \
        api/src/test/java/io/casehub/ledger/api/model/CapabilityTagTest.java
git commit -m "feat: add capabilityTag field to LedgerAttestation — defaults to \"*\" (global)

Refs #60 · Part of epic #50"
```

---

### Task 3: V1000 schema — add capability_tag column and indexes

**Files:**
- Modify: `runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql`

No new migration file — rewrite in place. No installed instances exist.

- [ ] **Step 1: Add column and indexes to `ledger_attestation` in V1000**

In the `CREATE TABLE ledger_attestation` block, add `capability_tag` after `confidence`:

```sql
    confidence       DOUBLE PRECISION NOT NULL DEFAULT 1.0,
    capability_tag   VARCHAR(255)     NOT NULL DEFAULT '*',
    occurred_at      TIMESTAMP       NOT NULL,
```

After the existing two attestation indexes, add:

```sql
CREATE INDEX idx_ledger_attestation_capability ON ledger_attestation (ledger_entry_id, capability_tag);
CREATE INDEX idx_ledger_attestation_actor_cap  ON ledger_attestation (attestor_id, capability_tag);
```

Full updated `ledger_attestation` block (replace the existing block):

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
    occurred_at      TIMESTAMP       NOT NULL,
    CONSTRAINT pk_ledger_attestation PRIMARY KEY (id),
    CONSTRAINT fk_attestation_entry FOREIGN KEY (ledger_entry_id) REFERENCES ledger_entry (id)
);

CREATE INDEX idx_ledger_attestation_entry     ON ledger_attestation (ledger_entry_id);
CREATE INDEX idx_ledger_attestation_subject   ON ledger_attestation (subject_id);
CREATE INDEX idx_ledger_attestation_capability ON ledger_attestation (ledger_entry_id, capability_tag);
CREATE INDEX idx_ledger_attestation_actor_cap  ON ledger_attestation (attestor_id, capability_tag);
```

- [ ] **Step 2: Verify schema file looks correct**

Use IntelliJ: open `V1000__ledger_base_schema.sql` in editor via `mcp__intellij__open_file_in_editor`.

Verify the `ledger_attestation` table has `capability_tag VARCHAR(255) NOT NULL DEFAULT '*'` and all four indexes.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql
git commit -m "feat: add capability_tag column and indexes to ledger_attestation (V1000 rewrite)

Refs #60 · Part of epic #50"
```

---

### Task 4: Named queries + SPI interfaces

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/ReactiveLedgerEntryRepository.java`

- [ ] **Step 1: Add three `@NamedQuery` annotations to `runtime/model/LedgerAttestation.java`**

Add after the existing three `@NamedQuery` annotations (before the class body):

```java
@NamedQuery(
        name = "LedgerAttestation.findByEntryIdAndCapabilityTag",
        query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId = :entryId AND a.capabilityTag = :capabilityTag ORDER BY a.occurredAt ASC")
@NamedQuery(
        name = "LedgerAttestation.findGlobalByEntryId",
        query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId = :entryId AND a.capabilityTag = '*' ORDER BY a.occurredAt ASC")
@NamedQuery(
        name = "LedgerAttestation.findByAttestorIdAndCapabilityTag",
        query = "SELECT a FROM LedgerAttestation a WHERE a.attestorId = :attestorId AND a.capabilityTag = :capabilityTag ORDER BY a.occurredAt ASC")
```

Full updated file:

```java
package io.casehub.ledger.runtime.model;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Entity;
import jakarta.persistence.NamedQuery;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

/**
 * A peer attestation stamped onto a {@link LedgerEntry}.
 * Plain {@code @Entity} — queries via {@code @NamedQuery} + EntityManager.
 */
@Entity
@Table(name = "ledger_attestation")
@NamedQuery(name = "LedgerAttestation.findByEntryId",
        query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId = :entryId ORDER BY a.occurredAt ASC")
@NamedQuery(name = "LedgerAttestation.findBySubjectId",
        query = "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :subjectId ORDER BY a.occurredAt ASC")
@NamedQuery(name = "LedgerAttestation.findByEntryIds",
        query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId IN :entryIds")
@NamedQuery(
        name = "LedgerAttestation.findByEntryIdAndCapabilityTag",
        query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId = :entryId AND a.capabilityTag = :capabilityTag ORDER BY a.occurredAt ASC")
@NamedQuery(
        name = "LedgerAttestation.findGlobalByEntryId",
        query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId = :entryId AND a.capabilityTag = '*' ORDER BY a.occurredAt ASC")
@NamedQuery(
        name = "LedgerAttestation.findByAttestorIdAndCapabilityTag",
        query = "SELECT a FROM LedgerAttestation a WHERE a.attestorId = :attestorId AND a.capabilityTag = :capabilityTag ORDER BY a.occurredAt ASC")
public class LedgerAttestation extends io.casehub.ledger.api.model.LedgerAttestation {

    @PrePersist
    void prePersist() {
        if (id == null)
            id = UUID.randomUUID();
        if (occurredAt == null)
            occurredAt = Instant.now();
    }
}
```

- [ ] **Step 2: Add three methods to `LedgerEntryRepository` (blocking SPI)**

Add after `findAttestationsByEntryId` (around line 62):

```java
    /**
     * Return all attestations for the given ledger entry scoped to the given capability tag,
     * ordered by occurrence time ascending. Use {@link io.casehub.ledger.api.model.CapabilityTag#GLOBAL}
     * to query global attestations explicitly.
     *
     * @param entryId the ledger entry UUID
     * @param capabilityTag the capability tag to filter by; never {@code null}
     * @return ordered list; empty if none match
     */
    List<LedgerAttestation> findAttestationsByEntryIdAndCapabilityTag(UUID entryId, String capabilityTag);

    /**
     * Return all global attestations for the given ledger entry (where
     * {@code capabilityTag = "*"}), ordered by occurrence time ascending.
     *
     * @param entryId the ledger entry UUID
     * @return ordered list; empty if none exist
     */
    List<LedgerAttestation> findGlobalAttestationsByEntryId(UUID entryId);

    /**
     * Return all attestations written by the given attestor for the given capability tag,
     * across all ledger entries, ordered by occurrence time ascending.
     *
     * <p>Used by B2 trust scoring to aggregate per-actor, per-capability attestation history.
     *
     * @param attestorId the attestor identity
     * @param capabilityTag the capability tag to filter by; never {@code null}
     * @return ordered list; empty if none match
     */
    List<LedgerAttestation> findAttestationsByAttestorIdAndCapabilityTag(String attestorId, String capabilityTag);
```

- [ ] **Step 3: Add three mirrored methods to `ReactiveLedgerEntryRepository`**

Add after `findAttestationsByEntryId` (around line 51):

```java
    Uni<List<LedgerAttestation>> findAttestationsByEntryIdAndCapabilityTag(UUID entryId, String capabilityTag);

    Uni<List<LedgerAttestation>> findGlobalAttestationsByEntryId(UUID entryId);

    Uni<List<LedgerAttestation>> findAttestationsByAttestorIdAndCapabilityTag(String attestorId, String capabilityTag);
```

- [ ] **Step 4: Build runtime module to verify named queries are validated by Hibernate at startup**

Prefer IntelliJ: `mcp__intellij__build_project` with `projectPath=/Users/mdproctor/claude/casehub/ledger`.

Fallback:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn compile -pl runtime -q
```
Expected: BUILD SUCCESS. Hibernate validates `@NamedQuery` JPQL at startup — a typo would fail here.

- [ ] **Step 5: Run `ReactiveRepositoryIT` to verify parity test still passes**

Prefer IntelliJ: run `ReactiveRepositoryIT` via the test runner.

Fallback:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=ReactiveRepositoryIT -q
```
Expected: BUILD SUCCESS, 2 tests PASSED (`reactiveSpi_usesUniReturnTypes`, `reactiveSpi_coversAllBlockingSpiMethods`).

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java \
        runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java \
        runtime/src/main/java/io/casehub/ledger/runtime/repository/ReactiveLedgerEntryRepository.java
git commit -m "feat: add named queries and SPI methods for capability-scoped attestation queries

Refs #60 · Part of epic #50"
```

---

### Task 5: Write failing integration tests (TDD)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerAttestationCapabilityIT.java`

- [ ] **Step 1: Create the integration test class**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;
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
 * Integration tests for capability-scoped attestation queries.
 *
 * <p>Covers: happy path, correctness/isolation, robustness, and backward compatibility.
 *
 * @see io.casehub.ledger.runtime.repository.LedgerEntryRepository
 */
@QuarkusTest
class LedgerAttestationCapabilityIT {

    @Inject
    LedgerEntryRepository repo;

    // ── Happy path: specific capability tag ───────────────────────────────────

    @Test
    @Transactional
    void save_specificCapabilityTag_storedAndRetrieved() {
        final TestEntry entry = savedEntry();
        repo.saveAttestation(attestation(entry.id, entry.subjectId, "security-review"));

        final List<LedgerAttestation> results =
                repo.findAttestationsByEntryIdAndCapabilityTag(entry.id, "security-review");

        assertThat(results).hasSize(1);
        assertThat(results.get(0).capabilityTag).isEqualTo("security-review");
    }

    // ── Happy path: global attestation ────────────────────────────────────────

    @Test
    @Transactional
    void save_globalCapabilityTag_retrievedByGlobalQuery() {
        final TestEntry entry = savedEntry();
        repo.saveAttestation(attestation(entry.id, entry.subjectId, CapabilityTag.GLOBAL));

        final List<LedgerAttestation> results =
                repo.findGlobalAttestationsByEntryId(entry.id);

        assertThat(results).hasSize(1);
        assertThat(results.get(0).capabilityTag).isEqualTo(CapabilityTag.GLOBAL);
    }

    // ── Correctness: capability query excludes global attestations ────────────

    @Test
    @Transactional
    void findByEntryIdAndCapabilityTag_doesNotReturnGlobal() {
        final TestEntry entry = savedEntry();
        repo.saveAttestation(attestation(entry.id, entry.subjectId, CapabilityTag.GLOBAL));
        repo.saveAttestation(attestation(entry.id, entry.subjectId, "security-review"));

        final List<LedgerAttestation> results =
                repo.findAttestationsByEntryIdAndCapabilityTag(entry.id, "security-review");

        assertThat(results).hasSize(1);
        assertThat(results.get(0).capabilityTag).isEqualTo("security-review");
    }

    // ── Correctness: global query excludes capability-specific attestations ───

    @Test
    @Transactional
    void findGlobalByEntryId_doesNotReturnCapabilitySpecific() {
        final TestEntry entry = savedEntry();
        repo.saveAttestation(attestation(entry.id, entry.subjectId, CapabilityTag.GLOBAL));
        repo.saveAttestation(attestation(entry.id, entry.subjectId, "security-review"));

        final List<LedgerAttestation> results =
                repo.findGlobalAttestationsByEntryId(entry.id);

        assertThat(results).hasSize(1);
        assertThat(results.get(0).capabilityTag).isEqualTo(CapabilityTag.GLOBAL);
    }

    // ── Correctness: capability query isolates across multiple tags ───────────

    @Test
    @Transactional
    void findByEntryIdAndCapabilityTag_isolatesFromOtherTags() {
        final TestEntry entry = savedEntry();
        repo.saveAttestation(attestation(entry.id, entry.subjectId, "security-review"));
        repo.saveAttestation(attestation(entry.id, entry.subjectId, "architecture-review"));

        final List<LedgerAttestation> results =
                repo.findAttestationsByEntryIdAndCapabilityTag(entry.id, "security-review");

        assertThat(results).hasSize(1);
        assertThat(results.get(0).capabilityTag).isEqualTo("security-review");
    }

    // ── Correctness: actor+capability query spans multiple entries ────────────

    @Test
    @Transactional
    void findByAttestorIdAndCapabilityTag_spansMultipleEntries() {
        final String attestorId = "peer-" + UUID.randomUUID();
        final TestEntry e1 = savedEntry();
        final TestEntry e2 = savedEntry();
        final TestEntry e3 = savedEntry();

        repo.saveAttestation(attestationBy(e1.id, e1.subjectId, attestorId, "security-review"));
        repo.saveAttestation(attestationBy(e2.id, e2.subjectId, attestorId, "security-review"));
        repo.saveAttestation(attestationBy(e3.id, e3.subjectId, attestorId, "architecture-review"));

        final List<LedgerAttestation> results =
                repo.findAttestationsByAttestorIdAndCapabilityTag(attestorId, "security-review");

        assertThat(results).hasSize(2);
        assertThat(results).allMatch(a -> "security-review".equals(a.capabilityTag));
    }

    // ── Robustness: no matching tag returns empty ─────────────────────────────

    @Test
    @Transactional
    void findByEntryIdAndCapabilityTag_noMatch_returnsEmpty() {
        final TestEntry entry = savedEntry();
        repo.saveAttestation(attestation(entry.id, entry.subjectId, "security-review"));

        final List<LedgerAttestation> results =
                repo.findAttestationsByEntryIdAndCapabilityTag(entry.id, "nonexistent-capability");

        assertThat(results).isEmpty();
    }

    @Test
    @Transactional
    void findGlobalByEntryId_noGlobalAttestations_returnsEmpty() {
        final TestEntry entry = savedEntry();
        repo.saveAttestation(attestation(entry.id, entry.subjectId, "security-review"));

        final List<LedgerAttestation> results =
                repo.findGlobalAttestationsByEntryId(entry.id);

        assertThat(results).isEmpty();
    }

    // ── Backward compatibility: unset capabilityTag defaults to global ────────

    @Test
    @Transactional
    void newAttestation_capabilityTagNotSet_defaultsToGlobalSentinel() {
        final TestEntry entry = savedEntry();

        final LedgerAttestation att = new LedgerAttestation();
        att.ledgerEntryId = entry.id;
        att.subjectId = entry.subjectId;
        att.attestorId = "auto-reviewer-" + UUID.randomUUID();
        att.attestorType = ActorType.AGENT;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 1.0;
        att.occurredAt = Instant.now().truncatedTo(ChronoUnit.MILLIS);
        // capabilityTag intentionally NOT set — field default must kick in
        repo.saveAttestation(att);

        final List<LedgerAttestation> globalResults =
                repo.findGlobalAttestationsByEntryId(entry.id);

        assertThat(globalResults).hasSize(1);
        assertThat(globalResults.get(0).capabilityTag).isEqualTo(CapabilityTag.GLOBAL);
    }

    // ── Fixtures ─────────────────────────────────────────────────────────────

    private TestEntry savedEntry() {
        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = "actor-" + UUID.randomUUID();
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = Instant.now().truncatedTo(ChronoUnit.MILLIS);
        return repo.save(entry);
    }

    private LedgerAttestation attestation(final UUID entryId, final UUID subjectId,
            final String capabilityTag) {
        return attestationBy(entryId, subjectId, "peer-reviewer-" + UUID.randomUUID(), capabilityTag);
    }

    private LedgerAttestation attestationBy(final UUID entryId, final UUID subjectId,
            final String attestorId, final String capabilityTag) {
        final LedgerAttestation att = new LedgerAttestation();
        att.ledgerEntryId = entryId;
        att.subjectId = subjectId;
        att.attestorId = attestorId;
        att.attestorType = ActorType.AGENT;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 1.0;
        att.capabilityTag = capabilityTag;
        att.occurredAt = Instant.now().truncatedTo(ChronoUnit.MILLIS);
        return att;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Prefer IntelliJ: run `LedgerAttestationCapabilityIT` via the test runner.

Fallback:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=LedgerAttestationCapabilityIT -q 2>&1 | tail -30
```
Expected: FAIL — `findAttestationsByEntryIdAndCapabilityTag`, `findGlobalAttestationsByEntryId`, `findAttestationsByAttestorIdAndCapabilityTag` not yet implemented in `JpaLedgerEntryRepository`.

- [ ] **Step 3: Commit the failing tests**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/LedgerAttestationCapabilityIT.java
git commit -m "test: add failing integration tests for capability-scoped attestation queries

TDD — tests fail until JpaLedgerEntryRepository implements the three new SPI methods.
Refs #60 · Part of epic #50"
```

---

### Task 6: JPA implementation (make tests pass)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`

- [ ] **Step 1: Add three methods to `JpaLedgerEntryRepository`**

Add after `findAttestationsByEntryId` (after line ~160):

```java
    /** {@inheritDoc} */
    @Override
    public List<LedgerAttestation> findAttestationsByEntryIdAndCapabilityTag(
            final UUID entryId, final String capabilityTag) {
        return em.createNamedQuery("LedgerAttestation.findByEntryIdAndCapabilityTag", LedgerAttestation.class)
                .setParameter("entryId", entryId)
                .setParameter("capabilityTag", capabilityTag)
                .getResultList();
    }

    /** {@inheritDoc} */
    @Override
    public List<LedgerAttestation> findGlobalAttestationsByEntryId(final UUID entryId) {
        return em.createNamedQuery("LedgerAttestation.findGlobalByEntryId", LedgerAttestation.class)
                .setParameter("entryId", entryId)
                .getResultList();
    }

    /** {@inheritDoc} */
    @Override
    public List<LedgerAttestation> findAttestationsByAttestorIdAndCapabilityTag(
            final String attestorId, final String capabilityTag) {
        final String token = actorIdentityProvider.tokeniseForQuery(attestorId);
        return em.createNamedQuery("LedgerAttestation.findByAttestorIdAndCapabilityTag", LedgerAttestation.class)
                .setParameter("attestorId", token)
                .setParameter("capabilityTag", capabilityTag)
                .getResultList();
    }
```

- [ ] **Step 2: Run integration tests to verify they pass**

Prefer IntelliJ: run `LedgerAttestationCapabilityIT` via the test runner.

Fallback:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -Dtest=LedgerAttestationCapabilityIT -q
```
Expected: BUILD SUCCESS, 9 tests PASSED.

- [ ] **Step 3: Run the full runtime test suite to check for regressions**

Prefer IntelliJ: `mcp__intellij__build_project` then run all tests.

Fallback:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -q
```
Expected: BUILD SUCCESS, all tests pass (no regressions).

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java
git commit -m "feat: implement capability-scoped attestation queries in JpaLedgerEntryRepository

Refs #60 · Part of epic #50"
```

---

### Task 7: LedgerTestFixtures — add capabilityTag overload

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerTestFixtures.java`

- [ ] **Step 1: Add capabilityTag overload**

Add a third `seedDecision` overload after the existing two:

```java
    /**
     * Persist a {@link TestEntry} EVENT with an attestation at an explicit timestamp and
     * capability tag. Use {@link io.casehub.ledger.api.model.CapabilityTag#GLOBAL} for
     * cross-capability attestations.
     */
    public static TestEntry seedDecision(final String actorId, final Instant decisionTime,
            final AttestationVerdict verdictOrNull, final Instant attestationTime,
            final String capabilityTag,
            final LedgerEntryRepository repo, final EntityManager em) {

        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = decisionTime.truncatedTo(ChronoUnit.MILLIS);
        repo.save(entry);

        if (verdictOrNull != null) {
            final LedgerAttestation att = new LedgerAttestation();
            att.id = UUID.randomUUID();
            att.ledgerEntryId = entry.id;
            att.subjectId = entry.subjectId;
            att.attestorId = "compliance-bot";
            att.attestorType = ActorType.AGENT;
            att.verdict = verdictOrNull;
            att.confidence = 1.0;
            att.capabilityTag = capabilityTag;
            att.occurredAt = attestationTime.truncatedTo(ChronoUnit.MILLIS);
            em.persist(att);
        }

        return entry;
    }
```

Also add the import for `CapabilityTag` at the top of the file:
```java
import io.casehub.ledger.api.model.CapabilityTag;
```

- [ ] **Step 2: Run full test suite to verify no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -q
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/LedgerTestFixtures.java
git commit -m "test: add capabilityTag overload to LedgerTestFixtures for B2 test support

Refs #60 · Part of epic #50"
```

---

### Task 8: Full build verification

- [ ] **Step 1: Build all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn clean install -q
```
Expected: BUILD SUCCESS across all modules (api, runtime, deployment).

- [ ] **Step 2: Verify test count increased**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) /opt/homebrew/bin/mvn test -pl runtime -q 2>&1 | grep "Tests run"
```
Expected: total tests increased by at least 12 (3 unit + 9 IT).

---

### Task 9: Documentation update

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `docs/DESIGN-capabilities.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update DESIGN.md Implementation Tracker**

Add a new row to the Implementation Tracker table (after the `OTel trace ID auto-wiring` row):

```markdown
| **capabilityTag on LedgerAttestation** | ✅ Done | `"*"` sentinel (no NULL); `CapabilityTag.GLOBAL` constant in api module; 3 new SPI query methods (blocking + reactive parity); V1000 schema updated; 9 IT tests + 3 unit tests. Closes #60. |
```

Also update the Roadmap entry for `capabilityTag` (near line 238-243) to mark it as done:

Find:
```markdown
`ActorTrustScore` uses a discriminator model keyed by `(actor_id, score_type, scope_key)`.
`score_type` is `GLOBAL` (classic cross-decision Beta score), `CAPABILITY` (scoped to a
capability tag — wired by #61), or `DIMENSION` (scoped to a trust dimension — wired by #62).
`scope_key` is null for GLOBAL rows; the unique constraint uses `NULLS NOT DISTINCT` to
enforce one GLOBAL row per actor. `TrustScoreJob` writes GLOBAL rows; capability and dimension
computation is added by #61 and #62 respectively.
```

Add after that paragraph:

```markdown
`LedgerAttestation.capabilityTag` (✅ #60) — nullable-free `"*"` sentinel (`CapabilityTag.GLOBAL`) marks cross-capability attestations. Capability-specific attestations carry an explicit tag (e.g. `"security-review"`). Three new SPI query methods allow `TrustScoreJob` (#61) to retrieve per-actor, per-capability attestation history.
```

- [ ] **Step 2: Update DESIGN-capabilities.md — add capabilityTag note under Agent Identity section**

The closest section to trust scoring in `docs/DESIGN-capabilities.md` is not yet present. Add a new `## Trust Scoring — Capability Tags` section before `## Agent Identity Model`:

```markdown
## Trust Scoring — Capability Tags

`LedgerAttestation.capabilityTag` scopes each verdict to a capability domain. The sentinel
`CapabilityTag.GLOBAL = "*"` (never null) means the verdict applies to all capabilities.
A specific tag (e.g. `"security-review"`) restricts it.

Three SPI query methods feed capability-aware trust computation:

| Method | Purpose |
|---|---|
| `findAttestationsByEntryIdAndCapabilityTag(entryId, tag)` | All capability-specific verdicts on one entry |
| `findGlobalAttestationsByEntryId(entryId)` | Global (`"*"`) verdicts on one entry |
| `findAttestationsByAttestorIdAndCapabilityTag(attestorId, tag)` | All verdicts by an actor for one capability (feeds B2 `TrustScoreJob`) |

See `docs/DESIGN.md` (Roadmap → Trust scoring) for the `ActorTrustScore` discriminator model
that consumes these signals in B2 (#61) and B3 (#62).

---
```

- [ ] **Step 3: Update CLAUDE.md project structure — add CapabilityTag.java**

In the `model/` section of the project structure, add `CapabilityTag.java` after `AttestationVerdict.java`:

```
│   ├── CapabilityTag.java           — sentinel constants: GLOBAL = "*" for cross-capability attestations (api module)
```

- [ ] **Step 4: Check for any cross-reference drift using IntelliJ search**

Use `mcp__intellij-index__ide_search_text` with query `capabilityTag` to find any references in docs or comments that may be stale or missing.

Also search for `capability_tag` to check SQL references are consistent.

Fix any stale references found.

- [ ] **Step 5: Commit documentation**

```bash
git add docs/DESIGN.md docs/DESIGN-capabilities.md CLAUDE.md
git commit -m "docs: document capabilityTag on LedgerAttestation across DESIGN.md, DESIGN-capabilities.md, CLAUDE.md

Closes #60 · Part of epic #50"
```

---

### Task 10: Close issue

- [ ] **Step 1: Verify the issue is closed by the final commit**

```bash
gh issue view 60 --repo casehubio/ledger
```

If not automatically closed (the `Closes #60` in the commit message only auto-closes on merge to default branch), close it manually:

```bash
gh issue close 60 --repo casehubio/ledger --comment "Implemented in commits on main. capabilityTag added to LedgerAttestation with CapabilityTag.GLOBAL = \"*\" sentinel, three new SPI query methods (blocking + reactive parity), V1000 schema updated, 9 IT tests + 3 unit tests."
```

- [ ] **Step 2: Verify epic #50 still shows #60 as resolved**

```bash
gh issue view 50 --repo casehubio/ledger
```
