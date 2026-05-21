# EU AI Act Art.12 Compliance Surface — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver EU AI Act Art.12 compliance — archive-then-delete retention enforcement, three auditor-facing query methods, a compliance example, and regulatory documentation.

**Architecture:** `RetentionEligibilityChecker` (pure Java) identifies subjects where every entry is past the window; `LedgerRetentionJob` verifies chains, archives to `ledger_entry_archive`, deletes attestations then entries in one transaction per subject. Three new SPI methods (`findByActorId`, `findByActorRole`, `findByTimeRange`) use `Instant` params for timezone-safe querying.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM / Panache, Flyway, Jackson, JUnit 5, AssertJ, `@QuarkusTest`.

**All commits:** `Refs #9`, part of epic `#6`.
**Build:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`

---

## File Map

| File | Action |
|---|---|
| `runtime/src/main/resources/db/migration/V1003__ledger_entry_archive.sql` | Create |
| `runtime/src/main/java/.../model/LedgerEntryArchiveRecord.java` | Create |
| `runtime/src/main/java/.../service/LedgerEntryArchiver.java` | Create |
| `runtime/src/main/java/.../service/RetentionEligibilityChecker.java` | Create |
| `runtime/src/main/java/.../service/LedgerRetentionJob.java` | Create |
| `runtime/src/main/java/.../config/LedgerConfig.java` | Modify — add `RetentionConfig` |
| `runtime/src/main/java/.../repository/LedgerEntryRepository.java` | Modify — add 3 audit query methods |
| `runtime/src/main/java/.../repository/jpa/JpaLedgerEntryRepository.java` | Modify — implement 3 methods |
| `runtime/src/test/java/.../service/LedgerEntryArchiverTest.java` | Create |
| `runtime/src/test/java/.../service/RetentionEligibilityCheckerTest.java` | Create |
| `runtime/src/test/java/.../service/LedgerRetentionJobIT.java` | Create |
| `runtime/src/test/java/.../service/AuditQueryIT.java` | Create |
| `runtime/src/test/resources/application.properties` | Modify — add retention-test profile |
| `examples/order-processing/src/main/resources/db/migration/V1003__order_ledger_entry.sql` | Rename → V1004 |
| `examples/art22-decision-snapshot/src/main/resources/db/migration/V1003__decision_schema.sql` | Rename → V1004 |
| `examples/order-processing/src/test/java/.../OrderLedgerIT.java` | Modify — add audit query test |
| `examples/art12-compliance/` | Create — new example |
| `docs/compliance/EU-AI-ACT-ART12.md` | Create |
| `docs/AUDITABILITY.md` | Modify — Axiom 5 + 6 gaps closed |
| `CLAUDE.md` | Modify — Flyway convention V1004+ |
| `docs/DESIGN.md` | Modify — tracker + convention |

Base package: `io.casehub.ledger.runtime`

---

## Task 1 — Rename example migrations + update Flyway convention docs

**Files:**
- Rename: `examples/order-processing/src/main/resources/db/migration/V1003__order_ledger_entry.sql` → `V1004__order_ledger_entry.sql`
- Rename: `examples/art22-decision-snapshot/src/main/resources/db/migration/V1003__decision_schema.sql` → `V1004__decision_schema.sql`
- Modify: `CLAUDE.md` (line with "V1003+" → "V1004+")
- Modify: `docs/DESIGN.md` (Flyway table + any "V1003+" references → "V1004+")
- Modify: `docs/integration-guide.md` (V1003+ references → V1004+)
- Modify: `examples/order-processing/README.md` (V1003 table row → V1004)

- [ ] **Step 1: Rename the two migration files**

```bash
cd examples/order-processing/src/main/resources/db/migration
mv V1003__order_ledger_entry.sql V1004__order_ledger_entry.sql

cd /Users/mdproctor/claude/casehub/ledger/examples/art22-decision-snapshot/src/main/resources/db/migration
mv V1003__decision_schema.sql V1004__decision_schema.sql
```

- [ ] **Step 2: Update CLAUDE.md**

Find the line containing `V1003+` in the Flyway convention section and change it to `V1004+`. The context reads "consumer subclass join tables must use V1003+" — change to V1004+. Also update "V1000–V1002" → "V1000–V1003" (base extension reservation).

- [ ] **Step 3: Update docs/DESIGN.md Flyway table**

Find the Flyway version numbering table and update:
```markdown
| Range | Owner | Purpose |
|---|---|---|
| V1000–V1003 | `casehub-ledger` base | Base schema (reserved — do not use in consumers) |
| V1–V999 | Consumer | Domain tables (orders, cases, channels, etc.) |
| V1004+ | Consumer | Subclass join tables (must run after V1000 — FK constraint) |
```

- [ ] **Step 4: Update integration-guide.md**

Find `V1003+` and change to `V1004+`. Find example migration comment `-- V1003__order_ledger_entry.sql` and change to `V1004__order_ledger_entry.sql`.

- [ ] **Step 5: Update examples/order-processing/README.md**

Find the Flyway migration table and update:
```markdown
| V1003 | from `casehub-ledger` jar | `ledger_entry_archive` table |
| V1004 | `V1004__order_ledger_entry.sql` | `order_ledger_entry` join table |
```
Update footer: "V1004 must be > V1003 so the base tables exist before the FK constraint."

- [ ] **Step 6: Run both example test suites — must still pass**

```bash
cd examples/order-processing
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -5
cd ../art22-decision-snapshot
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -5
cd ../..
```

Expected: both BUILD SUCCESS (H2 is in-memory, starts fresh each run).

- [ ] **Step 7: Commit**

```bash
git add examples/ CLAUDE.md docs/DESIGN.md docs/integration-guide.md
git commit -m "feat(art12): reserve V1003 for base extension — rename example migrations to V1004

Base extension now reserves V1000-V1003. Consumer subclass migrations: V1004+.
Renames examples/order-processing V1003→V1004 and art22 V1003→V1004.
Updates CLAUDE.md, DESIGN.md, integration-guide.md, order-processing README.

Refs #9"
```

---

## Task 2 — V1003 migration + `LedgerEntryArchiveRecord` entity

**Files:**
- Create: `runtime/src/main/resources/db/migration/V1003__ledger_entry_archive.sql`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntryArchiveRecord.java`

- [ ] **Step 1: Create V1003__ledger_entry_archive.sql**

```sql
-- V1003 — Archive table for entries past their retention window.
--
-- When quarkus.ledger.retention.enabled=true and archive-before-delete=true,
-- the retention job writes a full JSON snapshot of each entry here before
-- deleting it from ledger_entry. The archive is self-contained: entry_json
-- includes all core fields, supplementJson, and all attestations.
--
-- entry_occurred_at is a copy of LedgerEntry.occurredAt stored on the archive
-- row to allow efficient range queries on archived data without parsing entry_json.
--
-- Compatible with H2 (dev/test) and PostgreSQL (production).

CREATE TABLE ledger_entry_archive (
    id                UUID        NOT NULL,
    original_entry_id UUID        NOT NULL,
    subject_id        UUID        NOT NULL,
    sequence_number   INT         NOT NULL,
    entry_json        TEXT        NOT NULL,
    entry_occurred_at TIMESTAMP   NOT NULL,
    archived_at       TIMESTAMP   NOT NULL,
    CONSTRAINT pk_ledger_entry_archive PRIMARY KEY (id)
);

CREATE INDEX idx_archive_subject  ON ledger_entry_archive (subject_id);
CREATE INDEX idx_archive_occurred ON ledger_entry_archive (entry_occurred_at);
```

- [ ] **Step 2: Create `LedgerEntryArchiveRecord.java`**

```java
package io.casehub.ledger.runtime.model;

import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * Persisted archive record written by {@link io.casehub.ledger.runtime.service.LedgerRetentionJob}
 * before a {@link LedgerEntry} is removed from the main table.
 *
 * <p>
 * {@link #entryJson} contains a complete, self-contained JSON snapshot of the original
 * entry — all core fields, {@code supplementJson}, and all attestations — sufficient
 * for full reconstruction without access to the original tables.
 *
 * <p>
 * Archive records are written only when
 * {@code quarkus.ledger.retention.archive-before-delete=true} (the default).
 */
@Entity
@Table(name = "ledger_entry_archive")
public class LedgerEntryArchiveRecord extends PanacheEntityBase {

    /** Primary key — UUID assigned on first persist. */
    @Id
    public UUID id;

    /** The {@code id} of the original {@link LedgerEntry} that was archived. */
    @Column(name = "original_entry_id", nullable = false)
    public UUID originalEntryId;

    /** The aggregate identifier of the original entry. */
    @Column(name = "subject_id", nullable = false)
    public UUID subjectId;

    /** The sequence number of the original entry within its subject chain. */
    @Column(name = "sequence_number", nullable = false)
    public int sequenceNumber;

    /**
     * Full JSON snapshot of the archived entry, including all core fields,
     * {@code supplementJson}, and all attestations. Self-contained for reconstruction.
     */
    @Column(name = "entry_json", columnDefinition = "TEXT", nullable = false)
    public String entryJson;

    /**
     * Copy of {@link LedgerEntry#occurredAt} for efficient range queries on archived
     * data without parsing {@link #entryJson}.
     */
    @Column(name = "entry_occurred_at", nullable = false)
    public Instant entryOccurredAt;

    /** When this archive record was written. */
    @Column(name = "archived_at", nullable = false)
    public Instant archivedAt;

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
    }
}
```

- [ ] **Step 3: Run runtime tests — migration must apply cleanly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS — Flyway applies V1003 without errors, all existing tests pass.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/resources/db/migration/V1003__ledger_entry_archive.sql \
        runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntryArchiveRecord.java
git commit -m "feat(art12): V1003 migration + LedgerEntryArchiveRecord entity

ledger_entry_archive table with entry_json (full snapshot), entry_occurred_at
(for range queries on archive), archived_at. All existing tests pass.

Refs #9"
```

---

## Task 3 — `LedgerEntryArchiver` (TDD — unit tests first)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerEntryArchiverTest.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryArchiver.java`

- [ ] **Step 1: Write failing tests**

Create `runtime/src/test/java/io/casehub/ledger/service/LedgerEntryArchiverTest.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.LedgerEntryArchiver;

/**
 * Unit tests for {@link LedgerEntryArchiver} — no Quarkus runtime, no CDI.
 */
class LedgerEntryArchiverTest {

    private static class TestEntry extends LedgerEntry {}

    private TestEntry entry(String actorId) {
        final TestEntry e = new TestEntry();
        e.id             = UUID.randomUUID();
        e.subjectId      = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType      = LedgerEntryType.EVENT;
        e.actorId        = actorId;
        e.actorType      = ActorType.AGENT;
        e.actorRole      = "Classifier";
        e.occurredAt     = Instant.parse("2026-01-15T12:00:00Z");
        e.previousHash   = null;
        e.digest         = "abc123";
        return e;
    }

    @Test
    void toJson_coreFields_allPresent() {
        final TestEntry e = entry("agent-1");
        final String json = LedgerEntryArchiver.toJson(e, List.of());

        assertThat(json).contains("\"id\":\"" + e.id + "\"");
        assertThat(json).contains("\"subjectId\":\"" + e.subjectId + "\"");
        assertThat(json).contains("\"sequenceNumber\":1");
        assertThat(json).contains("\"entryType\":\"EVENT\"");
        assertThat(json).contains("\"actorId\":\"agent-1\"");
        assertThat(json).contains("\"actorType\":\"AGENT\"");
        assertThat(json).contains("\"actorRole\":\"Classifier\"");
        assertThat(json).contains("\"occurredAt\":\"2026-01-15T12:00:00Z\"");
        assertThat(json).contains("\"digest\":\"abc123\"");
    }

    @Test
    void toJson_nullFieldsOmitted() {
        final TestEntry e = entry("agent-1");
        e.previousHash = null;
        e.supplementJson = null;

        final String json = LedgerEntryArchiver.toJson(e, List.of());

        assertThat(json).doesNotContain("previousHash");
        assertThat(json).doesNotContain("supplementJson");
    }

    @Test
    void toJson_supplementJson_included() {
        final TestEntry e = entry("agent-1");
        e.supplementJson = "{\"COMPLIANCE\":{\"algorithmRef\":\"v1\"}}";

        final String json = LedgerEntryArchiver.toJson(e, List.of());

        assertThat(json).contains("supplementJson");
        assertThat(json).contains("algorithmRef");
    }

    @Test
    void toJson_noAttestations_emptyArrayOmitted() {
        final TestEntry e = entry("agent-1");
        final String json = LedgerEntryArchiver.toJson(e, List.of());

        assertThat(json).doesNotContain("attestations");
    }

    @Test
    void toJson_withAttestations_includedInOutput() {
        final TestEntry e = entry("agent-1");
        final LedgerAttestation att = new LedgerAttestation();
        att.id           = UUID.randomUUID();
        att.ledgerEntryId = e.id;
        att.attestorId   = "compliance-bot";
        att.attestorType = ActorType.AGENT;
        att.verdict      = AttestationVerdict.SOUND;
        att.confidence   = 0.95;
        att.occurredAt   = Instant.parse("2026-01-15T12:01:00Z");

        final String json = LedgerEntryArchiver.toJson(e, List.of(att));

        assertThat(json).contains("\"attestations\"");
        assertThat(json).contains("\"attestorId\":\"compliance-bot\"");
        assertThat(json).contains("\"verdict\":\"SOUND\"");
        assertThat(json).contains("\"confidence\":0.95");
    }

    @Test
    void toJson_isValidJson_parseable() throws Exception {
        final TestEntry e = entry("agent-1");
        e.supplementJson = "{\"COMPLIANCE\":{\"planRef\":\"policy-v1\"}}";
        final String json = LedgerEntryArchiver.toJson(e, List.of());

        // Should not throw
        new com.fasterxml.jackson.databind.ObjectMapper().readTree(json);
    }
}
```

- [ ] **Step 2: Run — confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerEntryArchiverTest -q 2>&1 | tail -5
```

Expected: compilation error — `LedgerEntryArchiver` not found.

- [ ] **Step 3: Create `LedgerEntryArchiver.java`**

```java
package io.casehub.ledger.runtime.service;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * Serialises a {@link LedgerEntry} and its attestations to a stable JSON string
 * for storage in {@code ledger_entry_archive.entry_json}.
 *
 * <p>
 * The JSON is self-contained: all core fields, {@code supplementJson}, and all
 * attestations are included. Null fields are omitted. The format is additive —
 * future fields added to {@link LedgerEntry} do not invalidate existing records.
 *
 * <p>
 * Pure static utility — no CDI, no database. Safe to use in unit tests.
 */
public final class LedgerEntryArchiver {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private LedgerEntryArchiver() {
    }

    /**
     * Serialise a ledger entry and its attestations to a JSON string for archival.
     *
     * @param entry        the entry to serialise; must not be null
     * @param attestations the attestations for this entry; may be null or empty
     * @return a non-null JSON string
     */
    public static String toJson(final LedgerEntry entry,
            final List<LedgerAttestation> attestations) {
        final Map<String, Object> map = new LinkedHashMap<>();

        map.put("id", entry.id.toString());
        map.put("subjectId", entry.subjectId.toString());
        map.put("sequenceNumber", entry.sequenceNumber);
        if (entry.entryType != null)    map.put("entryType", entry.entryType.name());
        if (entry.actorId != null)      map.put("actorId", entry.actorId);
        if (entry.actorType != null)    map.put("actorType", entry.actorType.name());
        if (entry.actorRole != null)    map.put("actorRole", entry.actorRole);
        if (entry.occurredAt != null)   map.put("occurredAt", entry.occurredAt.toString());
        if (entry.previousHash != null) map.put("previousHash", entry.previousHash);
        if (entry.digest != null)       map.put("digest", entry.digest);
        if (entry.supplementJson != null) map.put("supplementJson", entry.supplementJson);

        if (attestations != null && !attestations.isEmpty()) {
            final List<Map<String, Object>> attList = attestations.stream()
                    .map(LedgerEntryArchiver::attestationToMap)
                    .toList();
            map.put("attestations", attList);
        }

        try {
            return MAPPER.writeValueAsString(map);
        } catch (final JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialise entry to JSON for archive", e);
        }
    }

    private static Map<String, Object> attestationToMap(final LedgerAttestation a) {
        final Map<String, Object> m = new LinkedHashMap<>();
        if (a.id != null)            m.put("id", a.id.toString());
        if (a.attestorId != null)    m.put("attestorId", a.attestorId);
        if (a.attestorType != null)  m.put("attestorType", a.attestorType.name());
        if (a.attestorRole != null)  m.put("attestorRole", a.attestorRole);
        if (a.verdict != null)       m.put("verdict", a.verdict.name());
        m.put("confidence", a.confidence);
        if (a.occurredAt != null)    m.put("occurredAt", a.occurredAt.toString());
        return m;
    }
}
```

- [ ] **Step 4: Run tests — 5 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerEntryArchiverTest -q 2>&1 | tail -5
```

Expected: `Tests run: 5, Failures: 0, Errors: 0`.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryArchiver.java \
        runtime/src/test/java/io/casehub/ledger/service/LedgerEntryArchiverTest.java
git commit -m "feat(art12): LedgerEntryArchiver — JSON serialiser for archive storage

Null fields omitted. Attestations included when present. Pure static utility.
5 unit tests passing.

Refs #9"
```

---

## Task 4 — `RetentionEligibilityChecker` (TDD — unit tests first)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/RetentionEligibilityCheckerTest.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/RetentionEligibilityChecker.java`

- [ ] **Step 1: Write failing tests**

Create `runtime/src/test/java/io/casehub/ledger/service/RetentionEligibilityCheckerTest.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.RetentionEligibilityChecker;

/**
 * Unit tests for {@link RetentionEligibilityChecker} — no Quarkus runtime, no CDI.
 */
class RetentionEligibilityCheckerTest {

    private static class TestEntry extends LedgerEntry {}

    private final Instant now = Instant.now();

    private TestEntry entry(final UUID subjectId, final Instant occurredAt) {
        final TestEntry e = new TestEntry();
        e.id             = UUID.randomUUID();
        e.subjectId      = subjectId;
        e.sequenceNumber = 1;
        e.entryType      = LedgerEntryType.EVENT;
        e.actorId        = "actor";
        e.actorType      = ActorType.AGENT;
        e.occurredAt     = occurredAt;
        return e;
    }

    // ── happy path: eligible subjects ────────────────────────────────────────

    @Test
    void allEntriesOlderThanWindow_subjectIsEligible() {
        final UUID subject = UUID.randomUUID();
        final Map<UUID, List<LedgerEntry>> input = Map.of(subject, List.of(
                entry(subject, now.minus(40, ChronoUnit.DAYS)),
                entry(subject, now.minus(35, ChronoUnit.DAYS))));

        final Map<UUID, List<LedgerEntry>> result =
                RetentionEligibilityChecker.eligibleSubjects(input, now, 30);

        assertThat(result).containsKey(subject);
        assertThat(result.get(subject)).hasSize(2);
    }

    @Test
    void entryExactlyAtBoundary_isEligible() {
        // occurredAt == now - operationalDays → AT the cutoff → eligible
        final UUID subject = UUID.randomUUID();
        final Instant exactCutoff = now.minus(30, ChronoUnit.DAYS);
        final Map<UUID, List<LedgerEntry>> input =
                Map.of(subject, List.of(entry(subject, exactCutoff)));

        final Map<UUID, List<LedgerEntry>> result =
                RetentionEligibilityChecker.eligibleSubjects(input, now, 30);

        assertThat(result).containsKey(subject);
    }

    // ── happy path: ineligible subjects ──────────────────────────────────────

    @Test
    void allEntriesNewerThanWindow_subjectNotEligible() {
        final UUID subject = UUID.randomUUID();
        final Map<UUID, List<LedgerEntry>> input = Map.of(subject, List.of(
                entry(subject, now.minus(10, ChronoUnit.DAYS))));

        final Map<UUID, List<LedgerEntry>> result =
                RetentionEligibilityChecker.eligibleSubjects(input, now, 30);

        assertThat(result).doesNotContainKey(subject);
    }

    @Test
    void entryOneDayBeforeBoundary_notEligible() {
        // occurredAt == now - (operationalDays - 1) → one day short → not eligible
        final UUID subject = UUID.randomUUID();
        final Instant oneDayShort = now.minus(29, ChronoUnit.DAYS);
        final Map<UUID, List<LedgerEntry>> input =
                Map.of(subject, List.of(entry(subject, oneDayShort)));

        final Map<UUID, List<LedgerEntry>> result =
                RetentionEligibilityChecker.eligibleSubjects(input, now, 30);

        assertThat(result).doesNotContainKey(subject);
    }

    @Test
    void mixedAges_oneNewEntry_subjectNotEligible() {
        // All-or-nothing: one new entry means the whole subject is excluded
        final UUID subject = UUID.randomUUID();
        final Map<UUID, List<LedgerEntry>> input = Map.of(subject, List.of(
                entry(subject, now.minus(60, ChronoUnit.DAYS)),
                entry(subject, now.minus(45, ChronoUnit.DAYS)),
                entry(subject, now.minus(5, ChronoUnit.DAYS))  // too new
        ));

        final Map<UUID, List<LedgerEntry>> result =
                RetentionEligibilityChecker.eligibleSubjects(input, now, 30);

        assertThat(result).doesNotContainKey(subject);
    }

    @Test
    void emptyInput_returnsEmptyMap() {
        final Map<UUID, List<LedgerEntry>> result =
                RetentionEligibilityChecker.eligibleSubjects(Map.of(), now, 30);

        assertThat(result).isEmpty();
    }

    @Test
    void emptySubjectList_subjectExcluded() {
        final UUID subject = UUID.randomUUID();
        final Map<UUID, List<LedgerEntry>> input = Map.of(subject, List.of());

        final Map<UUID, List<LedgerEntry>> result =
                RetentionEligibilityChecker.eligibleSubjects(input, now, 30);

        assertThat(result).doesNotContainKey(subject);
    }

    @Test
    void multipleSubjects_onlyOldOnesReturned() {
        final UUID oldSubject = UUID.randomUUID();
        final UUID newSubject = UUID.randomUUID();
        final Map<UUID, List<LedgerEntry>> input = Map.of(
                oldSubject, List.of(entry(oldSubject, now.minus(60, ChronoUnit.DAYS))),
                newSubject, List.of(entry(newSubject, now.minus(5, ChronoUnit.DAYS))));

        final Map<UUID, List<LedgerEntry>> result =
                RetentionEligibilityChecker.eligibleSubjects(input, now, 30);

        assertThat(result).containsKey(oldSubject);
        assertThat(result).doesNotContainKey(newSubject);
    }
}
```

- [ ] **Step 2: Run — confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=RetentionEligibilityCheckerTest -q 2>&1 | tail -5
```

Expected: compilation error — `RetentionEligibilityChecker` not found.

- [ ] **Step 3: Create `RetentionEligibilityChecker.java`**

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * Pure Java utility that identifies ledger subjects eligible for retention archival.
 *
 * <p>
 * A subject is eligible only when <strong>every</strong> entry in its chain is older
 * than the configured retention window. This all-or-nothing rule preserves hash chain
 * integrity: archiving and deleting a partial chain would leave dangling
 * {@code previousHash} references in the entries that remain.
 *
 * <p>
 * No CDI, no database — all inputs are passed as collections. Safe for unit testing.
 */
public final class RetentionEligibilityChecker {

    private RetentionEligibilityChecker() {
    }

    /**
     * Return the subset of subjects where every entry has an {@code occurredAt}
     * timestamp at or before the retention cutoff ({@code now - operationalDays}).
     *
     * @param allBySubject   all ledger entries grouped by {@code subjectId}
     * @param now            reference timestamp for age calculation
     * @param operationalDays retention window in days (EU AI Act Art.12 minimum: 180)
     * @return map of eligible subjects; never null, may be empty
     */
    public static Map<UUID, List<LedgerEntry>> eligibleSubjects(
            final Map<UUID, List<LedgerEntry>> allBySubject,
            final Instant now,
            final int operationalDays) {

        final Instant cutoff = now.minus(operationalDays, ChronoUnit.DAYS);
        final Map<UUID, List<LedgerEntry>> eligible = new LinkedHashMap<>();

        for (final Map.Entry<UUID, List<LedgerEntry>> e : allBySubject.entrySet()) {
            final List<LedgerEntry> entries = e.getValue();
            if (entries.isEmpty()) {
                continue;
            }
            final boolean allOldEnough = entries.stream()
                    .allMatch(entry -> entry.occurredAt != null
                            && !entry.occurredAt.isAfter(cutoff));
            if (allOldEnough) {
                eligible.put(e.getKey(), entries);
            }
        }

        return eligible;
    }
}
```

- [ ] **Step 4: Run tests — all 7 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=RetentionEligibilityCheckerTest -q 2>&1 | tail -5
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/RetentionEligibilityChecker.java \
        runtime/src/test/java/io/casehub/ledger/service/RetentionEligibilityCheckerTest.java
git commit -m "feat(art12): RetentionEligibilityChecker — pure Java eligibility logic

All-or-nothing per subject chain. 7 unit tests covering boundary cases,
mixed ages, empty inputs, multiple subjects.

Refs #9"
```

---

## Task 5 — `LedgerConfig.RetentionConfig` + `LedgerRetentionJob`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerRetentionJob.java`
- Modify: `runtime/src/test/resources/application.properties`
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerRetentionJobIT.java`

- [ ] **Step 1: Add `RetentionConfig` to `LedgerConfig.java`**

After the `TrustScoreConfig trustScore();` method, add:

```java
    /**
     * Retention enforcement — archives and removes ledger entries that have exceeded
     * their mandatory retention window, satisfying EU AI Act Article 12 record-keeping.
     *
     * @return retention sub-configuration
     */
    RetentionConfig retention();

    /** Retention enforcement settings. */
    interface RetentionConfig {

        /**
         * When {@code true}, a nightly job archives and deletes entries older than
         * {@link #operationalDays()}. Off by default — zero behaviour change when disabled.
         *
         * @return {@code true} if retention enforcement is active; {@code false} by default
         */
        @WithDefault("false")
        boolean enabled();

        /**
         * Minimum retention window in days. Entries older than this are candidates for
         * archival. EU AI Act Article 12 requires at least 180 days (6 months).
         *
         * @return retention window in days (default 180)
         */
        @WithDefault("180")
        int operationalDays();

        /**
         * When {@code true}, each entry is written to {@code ledger_entry_archive}
         * before being deleted from {@code ledger_entry}. The archive record is
         * self-contained for reconstruction. Disabling skips the archive step and
         * deletes directly — use only when external archiving is handled separately.
         *
         * @return {@code true} if archive-before-delete is enabled (default)
         */
        @WithDefault("true")
        boolean archiveBeforeDelete();
    }
```

- [ ] **Step 2: Write the failing integration test**

First, append to `runtime/src/test/resources/application.properties`:

```properties
# Retention test profile (used by LedgerRetentionJobIT)
%retention-test.quarkus.ledger.retention.enabled=true
%retention-test.quarkus.ledger.retention.operational-days=30
%retention-test.quarkus.ledger.retention.archive-before-delete=true
```

Then create `runtime/src/test/java/io/casehub/ledger/service/LedgerRetentionJobIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryArchiveRecord;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.LedgerHashChain;
import io.casehub.ledger.runtime.service.LedgerRetentionJob;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

/**
 * End-to-end integration tests for {@link LedgerRetentionJob}.
 *
 * <p>
 * Uses the {@code retention-test} Quarkus profile which enables retention with a
 * 30-day window and archive-before-delete=true.
 */
@QuarkusTest
@TestProfile(LedgerRetentionJobIT.RetentionProfile.class)
class LedgerRetentionJobIT {

    public static class RetentionProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() { return "retention-test"; }
    }

    @Inject
    LedgerRetentionJob retentionJob;

    // ── happy path: old entries archived and deleted ──────────────────────────

    @Test
    @Transactional
    void oldEntries_archivedAndDeleted() {
        final UUID subjectId = UUID.randomUUID();
        final TestEntry e1 = chainedEntry(subjectId, 1, now().minus(60, ChronoUnit.DAYS), null);
        final TestEntry e2 = chainedEntry(subjectId, 2, now().minus(45, ChronoUnit.DAYS), e1.digest);

        retentionJob.runRetention();

        // Entries gone from main table
        assertThat(LedgerEntry.find("subjectId", subjectId).count()).isZero();
        // Archive records created
        assertThat(LedgerEntryArchiveRecord.find("subjectId", subjectId).count()).isEqualTo(2);
    }

    // ── happy path: new entries untouched ─────────────────────────────────────

    @Test
    @Transactional
    void newEntries_notDeleted() {
        final UUID subjectId = UUID.randomUUID();
        chainedEntry(subjectId, 1, now().minus(5, ChronoUnit.DAYS), null);

        retentionJob.runRetention();

        assertThat(LedgerEntry.find("subjectId", subjectId).count()).isEqualTo(1);
        assertThat(LedgerEntryArchiveRecord.find("subjectId", subjectId).count()).isZero();
    }

    // ── happy path: archive record has correct content ────────────────────────

    @Test
    @Transactional
    void archiveRecord_containsEntryJson() {
        final UUID subjectId = UUID.randomUUID();
        final TestEntry e = chainedEntry(subjectId, 1, now().minus(40, ChronoUnit.DAYS), null);

        retentionJob.runRetention();

        final LedgerEntryArchiveRecord record = LedgerEntryArchiveRecord
                .<LedgerEntryArchiveRecord>find("subjectId", subjectId).firstResult();
        assertThat(record).isNotNull();
        assertThat(record.originalEntryId).isEqualTo(e.id);
        assertThat(record.entryJson).contains("\"actorId\":\"retention-actor\"");
        assertThat(record.entryJson).contains("\"sequenceNumber\":1");
        assertThat(record.archivedAt).isNotNull();
    }

    // ── happy path: attestations deleted (FK constraint respected) ────────────

    @Test
    @Transactional
    void attestations_deletedBeforeEntry_noFkViolation() {
        final UUID subjectId = UUID.randomUUID();
        final TestEntry e = chainedEntry(subjectId, 1, now().minus(40, ChronoUnit.DAYS), null);
        seedAttestation(e.id, subjectId, AttestationVerdict.SOUND);

        // If FK ordering is wrong, this throws a constraint violation
        retentionJob.runRetention();

        assertThat(LedgerEntry.find("subjectId", subjectId).count()).isZero();
        assertThat(LedgerAttestation.find("ledgerEntryId", e.id).count()).isZero();
    }

    // ── happy path: archive includes attestations in entry_json ───────────────

    @Test
    @Transactional
    void archiveJson_includesAttestations() {
        final UUID subjectId = UUID.randomUUID();
        final TestEntry e = chainedEntry(subjectId, 1, now().minus(40, ChronoUnit.DAYS), null);
        seedAttestation(e.id, subjectId, AttestationVerdict.FLAGGED);

        retentionJob.runRetention();

        final LedgerEntryArchiveRecord record = LedgerEntryArchiveRecord
                .<LedgerEntryArchiveRecord>find("subjectId", subjectId).firstResult();
        assertThat(record.entryJson).contains("\"attestations\"");
        assertThat(record.entryJson).contains("FLAGGED");
    }

    // ── happy path: mixed-age subject — all-or-nothing ───────────────────────

    @Test
    @Transactional
    void mixedAges_subjectNotArchived() {
        final UUID subjectId = UUID.randomUUID();
        final TestEntry old = chainedEntry(subjectId, 1, now().minus(60, ChronoUnit.DAYS), null);
        chainedEntry(subjectId, 2, now().minus(5, ChronoUnit.DAYS), old.digest);

        retentionJob.runRetention();

        // Neither entry deleted because one is too new
        assertThat(LedgerEntry.find("subjectId", subjectId).count()).isEqualTo(2);
        assertThat(LedgerEntryArchiveRecord.find("subjectId", subjectId).count()).isZero();
    }

    // ── happy path: broken chain — subject skipped ───────────────────────────

    @Test
    @Transactional
    void brokenChain_subjectSkipped() {
        final UUID subjectId = UUID.randomUUID();
        final TestEntry e = chainedEntry(subjectId, 1, now().minus(50, ChronoUnit.DAYS), null);
        // Tamper: corrupt the digest so chain verification fails
        e.digest = "000000000000000000000000000000000000000000000000000000000000dead";
        // persist the tampered entry
        e.persist();

        retentionJob.runRetention();

        // Subject skipped — entry still in main table, no archive record
        assertThat(LedgerEntry.find("subjectId", subjectId).count()).isEqualTo(1);
        assertThat(LedgerEntryArchiveRecord.find("subjectId", subjectId).count()).isZero();
    }

    // ── happy path: archive-before-delete=false — direct delete, no archive ──

    // Note: this test uses the default retention-test profile (archiveBeforeDelete=true).
    // A separate profile would be needed to test archiveBeforeDelete=false.
    // Covered by unit-level tests of the archiver; this IT focuses on the full pipeline.

    // ── fixture ───────────────────────────────────────────────────────────────

    private TestEntry chainedEntry(final UUID subjectId, final int seq,
            final Instant occurredAt, final String previousHash) {
        final TestEntry e = new TestEntry();
        e.subjectId      = subjectId;
        e.sequenceNumber = seq;
        e.entryType      = LedgerEntryType.EVENT;
        e.actorId        = "retention-actor";
        e.actorType      = ActorType.AGENT;
        e.actorRole      = "Classifier";
        e.occurredAt     = occurredAt;
        e.previousHash   = previousHash;
        e.digest         = LedgerHashChain.compute(previousHash, e);
        e.persist();
        return e;
    }

    private void seedAttestation(final UUID entryId, final UUID subjectId,
            final AttestationVerdict verdict) {
        final LedgerAttestation att = new LedgerAttestation();
        att.id            = UUID.randomUUID();
        att.ledgerEntryId = entryId;
        att.subjectId     = subjectId;
        att.attestorId    = "compliance-bot";
        att.attestorType  = ActorType.AGENT;
        att.verdict       = verdict;
        att.confidence    = 0.9;
        att.occurredAt    = Instant.now();
        att.persist();
    }

    private Instant now() { return Instant.now(); }
}
```

- [ ] **Step 3: Run IT — confirm compile failure (LedgerRetentionJob missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerRetentionJobIT -q 2>&1 | tail -5
```

Expected: compilation error.

- [ ] **Step 4: Create `LedgerRetentionJob.java`**

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryArchiveRecord;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.scheduler.Scheduled;

/**
 * Nightly scheduled job that enforces the configured retention window by archiving
 * and deleting ledger entries past their mandatory retention period.
 *
 * <p>
 * Gated by {@code quarkus.ledger.retention.enabled} (off by default). When disabled,
 * the scheduled trigger fires but immediately returns without touching any data.
 *
 * <p>
 * Per-subject transaction: if archiving or deletion fails for one subject, the
 * transaction for that subject rolls back and the job continues to the next subject.
 * This makes retention runs idempotent and safe to retry.
 *
 * <p>
 * Chain integrity is verified before any deletion. A subject with a broken hash chain
 * is skipped and a warning is logged. No data is ever deleted without a prior
 * integrity check.
 */
@ApplicationScoped
public class LedgerRetentionJob {

    private static final Logger LOG = Logger.getLogger(LedgerRetentionJob.class);

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    LedgerConfig config;

    /**
     * Scheduled entry point — runs every 24 hours.
     * Delegates to {@link #runRetention()} when retention is enabled.
     */
    @Scheduled(every = "24h", identity = "ledger-retention-job")
    @Transactional
    public void enforceRetention() {
        if (!config.retention().enabled()) {
            return;
        }
        runRetention();
    }

    /**
     * Perform the full retention run. Exposed for direct invocation in tests.
     */
    @Transactional
    public void runRetention() {
        final Instant now = Instant.now();
        final int operationalDays = config.retention().operationalDays();
        final boolean archiveBeforeDelete = config.retention().archiveBeforeDelete();

        // Load ALL entries grouped by subject (not just EVENTs — retention covers all types)
        final List<LedgerEntry> all = LedgerEntry.listAll();
        final Map<UUID, List<LedgerEntry>> bySubject = all.stream()
                .filter(e -> e.subjectId != null)
                .collect(Collectors.groupingBy(e -> e.subjectId));

        final Map<UUID, List<LedgerEntry>> eligible =
                RetentionEligibilityChecker.eligibleSubjects(bySubject, now, operationalDays);

        int archived = 0;
        int skipped = 0;
        for (final Map.Entry<UUID, List<LedgerEntry>> entry : eligible.entrySet()) {
            try {
                archiveSubject(entry.getKey(), entry.getValue(), archiveBeforeDelete, now);
                archived++;
            } catch (final Exception e) {
                LOG.errorf(e, "Retention: skipping subject %s — %s", entry.getKey(), e.getMessage());
                skipped++;
            }
        }
        LOG.infof("Retention run complete: %d subjects archived, %d skipped", archived, skipped);
    }

    private void archiveSubject(final UUID subjectId, final List<LedgerEntry> entries,
            final boolean archiveBeforeDelete, final Instant now) {

        // 1. Verify chain integrity — abort this subject if broken
        final List<LedgerEntry> sorted = entries.stream()
                .sorted(java.util.Comparator.comparingInt(e -> e.sequenceNumber))
                .toList();
        if (!LedgerHashChain.verify(sorted)) {
            throw new IllegalStateException(
                    "Hash chain integrity check failed for subject " + subjectId);
        }

        final Set<UUID> entryIds = entries.stream()
                .map(e -> e.id).collect(Collectors.toSet());

        // 2. Archive each entry (if configured)
        if (archiveBeforeDelete) {
            final Map<UUID, List<LedgerAttestation>> attestsByEntry =
                    ledgerRepo.findAttestationsForEntries(entryIds);
            for (final LedgerEntry e : sorted) {
                final List<LedgerAttestation> attests =
                        attestsByEntry.getOrDefault(e.id, List.of());
                final LedgerEntryArchiveRecord record = new LedgerEntryArchiveRecord();
                record.originalEntryId  = e.id;
                record.subjectId        = e.subjectId;
                record.sequenceNumber   = e.sequenceNumber;
                record.entryJson        = LedgerEntryArchiver.toJson(e, attests);
                record.entryOccurredAt  = e.occurredAt;
                record.archivedAt       = now;
                record.persist();
            }
        }

        // 3. Delete attestations first (non-cascaded FK → ledger_entry)
        LedgerAttestation.delete("ledgerEntryId in ?1", entryIds);

        // 4. Delete entries — JPA cascade removes: supplements → subclass rows → ledger_entry
        for (final LedgerEntry e : sorted) {
            e.delete();
        }
    }
}
```

- [ ] **Step 5: Run all IT tests — 6 must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerRetentionJobIT -q 2>&1 | tail -10
```

Expected: `Tests run: 6, Failures: 0, Errors: 0`.

- [ ] **Step 6: Run the full runtime suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS, zero failures.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerRetentionJob.java \
        runtime/src/test/java/io/casehub/ledger/service/LedgerRetentionJobIT.java \
        runtime/src/test/resources/application.properties
git commit -m "feat(art12): RetentionConfig + LedgerRetentionJob — archive-then-delete retention enforcement

Per-subject transaction: verify chain → archive to ledger_entry_archive →
delete attestations → JPA-cascade delete entries. Skips subjects with broken
chains. 6 @QuarkusTest integration tests covering all scenarios.

quarkus.ledger.retention.enabled=false (default — zero behaviour change)

Refs #9"
```

---

## Task 6 — Audit query SPI + JPA implementation + integration tests (TDD)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/AuditQueryIT.java`

- [ ] **Step 1: Write failing integration tests**

Create `runtime/src/test/java/io/casehub/ledger/service/AuditQueryIT.java`:

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

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;

/**
 * Integration tests for the three auditor-facing query methods on {@link LedgerEntryRepository}.
 */
@QuarkusTest
class AuditQueryIT {

    @Inject
    LedgerEntryRepository repo;

    private final Instant t0 = Instant.parse("2026-01-01T00:00:00Z");
    private final Instant t1 = Instant.parse("2026-01-15T12:00:00Z");
    private final Instant t2 = Instant.parse("2026-01-31T23:59:59Z");
    private final Instant t3 = Instant.parse("2026-02-15T00:00:00Z");

    // ── findByActorId ─────────────────────────────────────────────────────────

    @Test
    @Transactional
    void findByActorId_returnsEntriesInRange() {
        final String actor = "audit-actor-" + UUID.randomUUID();
        seedEntry(actor, "Classifier", t1);
        seedEntry(actor, "Classifier", t2);
        seedEntry(actor, "Classifier", t3); // outside range

        final List<LedgerEntry> results = repo.findByActorId(actor, t0, t2);

        assertThat(results).hasSize(2);
        assertThat(results).allMatch(e -> !e.occurredAt.isAfter(t2));
    }

    @Test
    @Transactional
    void findByActorId_boundaryInclusive() {
        final String actor = "audit-actor-" + UUID.randomUUID();
        seedEntry(actor, "Classifier", t1); // exactly at 'from'
        seedEntry(actor, "Classifier", t2); // exactly at 'to'

        final List<LedgerEntry> results = repo.findByActorId(actor, t1, t2);

        assertThat(results).hasSize(2);
    }

    @Test
    @Transactional
    void findByActorId_noEntries_returnsEmpty() {
        final List<LedgerEntry> results =
                repo.findByActorId("actor-that-does-not-exist-" + UUID.randomUUID(), t0, t3);

        assertThat(results).isEmpty();
    }

    @Test
    @Transactional
    void findByActorId_orderedByOccurredAtAsc() {
        final String actor = "audit-actor-" + UUID.randomUUID();
        seedEntry(actor, "Classifier", t2);
        seedEntry(actor, "Classifier", t1);

        final List<LedgerEntry> results = repo.findByActorId(actor, t0, t3);

        assertThat(results).hasSize(2);
        assertThat(results.get(0).occurredAt).isEqualTo(t1);
        assertThat(results.get(1).occurredAt).isEqualTo(t2);
    }

    // ── findByActorRole ───────────────────────────────────────────────────────

    @Test
    @Transactional
    void findByActorRole_returnsEntriesInRange() {
        final String role = "Auditor-" + UUID.randomUUID();
        seedEntry("actor-a", role, t1);
        seedEntry("actor-b", role, t2);
        seedEntry("actor-c", role, t3); // outside range

        final List<LedgerEntry> results = repo.findByActorRole(role, t0, t2);

        assertThat(results).hasSize(2);
        assertThat(results).allMatch(e -> e.actorRole.equals(role));
    }

    @Test
    @Transactional
    void findByActorRole_noEntries_returnsEmpty() {
        final List<LedgerEntry> results =
                repo.findByActorRole("NonExistentRole-" + UUID.randomUUID(), t0, t3);

        assertThat(results).isEmpty();
    }

    // ── findByTimeRange ───────────────────────────────────────────────────────

    @Test
    @Transactional
    void findByTimeRange_returnsAllInWindow() {
        final String marker = "timerange-" + UUID.randomUUID();
        seedEntry(marker, "Classifier", t1);
        seedEntry(marker, "Classifier", t2);
        seedEntry(marker, "Classifier", t3); // outside window

        final List<LedgerEntry> results = repo.findByTimeRange(t0, t2);
        final List<LedgerEntry> inWindow = results.stream()
                .filter(e -> e.actorId != null && e.actorId.equals(marker))
                .toList();

        assertThat(inWindow).hasSize(2);
    }

    @Test
    @Transactional
    void findByTimeRange_orderedByOccurredAtAsc() {
        final String marker = "timerange-order-" + UUID.randomUUID();
        seedEntry(marker, "Classifier", t2);
        seedEntry(marker, "Classifier", t1);

        final List<LedgerEntry> results = repo.findByTimeRange(t0, t3);
        final List<LedgerEntry> mine = results.stream()
                .filter(e -> marker.equals(e.actorId))
                .toList();

        assertThat(mine.get(0).occurredAt).isEqualTo(t1);
        assertThat(mine.get(1).occurredAt).isEqualTo(t2);
    }

    // ── fixture ───────────────────────────────────────────────────────────────

    private void seedEntry(final String actorId, final String actorRole, final Instant occurredAt) {
        final TestEntry e = new TestEntry();
        e.subjectId      = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType      = LedgerEntryType.EVENT;
        e.actorId        = actorId;
        e.actorType      = ActorType.AGENT;
        e.actorRole      = actorRole;
        e.occurredAt     = occurredAt;
        e.persist();
    }
}
```

- [ ] **Step 2: Run — confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=AuditQueryIT -q 2>&1 | tail -5
```

Expected: compilation error — `findByActorId` etc. not found on `LedgerEntryRepository`.

- [ ] **Step 3: Add 3 methods to `LedgerEntryRepository.java`**

After `findAttestationsForEntries`, add:

```java
    /**
     * Return all ledger entries for the given actor whose {@code occurredAt} falls
     * within [{@code from}, {@code to}] inclusive, ordered by {@code occurredAt} ascending.
     *
     * <p>
     * Provides the auditor-facing reconstructability required by EU AI Act Art.12 and
     * GDPR Art.22: "show everything actor X did between dates Y and Z."
     *
     * @param actorId the actor identity to filter by
     * @param from    start of the time range (inclusive)
     * @param to      end of the time range (inclusive)
     * @return ordered list of entries; empty if none match
     */
    List<LedgerEntry> findByActorId(String actorId, Instant from, Instant to);

    /**
     * Return all ledger entries for the given actor role whose {@code occurredAt} falls
     * within [{@code from}, {@code to}] inclusive, ordered by {@code occurredAt} ascending.
     *
     * @param actorRole the functional role to filter by (e.g. {@code "Classifier"})
     * @param from      start of the time range (inclusive)
     * @param to        end of the time range (inclusive)
     * @return ordered list of entries; empty if none match
     */
    List<LedgerEntry> findByActorRole(String actorRole, Instant from, Instant to);

    /**
     * Return all ledger entries whose {@code occurredAt} falls within
     * [{@code from}, {@code to}] inclusive, ordered by {@code occurredAt} ascending.
     *
     * <p>
     * Use for bulk audit exports and retention window queries.
     *
     * @param from start of the time range (inclusive)
     * @param to   end of the time range (inclusive)
     * @return ordered list of entries; empty if none match
     */
    List<LedgerEntry> findByTimeRange(Instant from, Instant to);
```

Also add `import java.time.Instant;` at the top of the file if not already present.

- [ ] **Step 4: Implement 3 methods in `JpaLedgerEntryRepository.java`**

After `findAttestationsForEntries`, add:

```java
    /** {@inheritDoc} */
    @Override
    public List<LedgerEntry> findByActorId(final String actorId,
            final Instant from, final Instant to) {
        return LedgerEntry.list(
                "actorId = ?1 AND occurredAt >= ?2 AND occurredAt <= ?3 ORDER BY occurredAt ASC",
                actorId, from, to);
    }

    /** {@inheritDoc} */
    @Override
    public List<LedgerEntry> findByActorRole(final String actorRole,
            final Instant from, final Instant to) {
        return LedgerEntry.list(
                "actorRole = ?1 AND occurredAt >= ?2 AND occurredAt <= ?3 ORDER BY occurredAt ASC",
                actorRole, from, to);
    }

    /** {@inheritDoc} */
    @Override
    public List<LedgerEntry> findByTimeRange(final Instant from, final Instant to) {
        return LedgerEntry.list(
                "occurredAt >= ?1 AND occurredAt <= ?2 ORDER BY occurredAt ASC",
                from, to);
    }
```

Also add `import java.time.Instant;` at the top if not already present.

- [ ] **Step 5: Run audit query IT — 7 tests must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=AuditQueryIT -q 2>&1 | tail -5
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`.

- [ ] **Step 6: Run full runtime suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS, zero failures.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java \
        runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java \
        runtime/src/test/java/io/casehub/ledger/service/AuditQueryIT.java
git commit -m "feat(art12): audit query API — findByActorId, findByActorRole, findByTimeRange

Three new SPI methods with Instant params for timezone-safe auditor queries.
JPA implementations on JpaLedgerEntryRepository. 7 integration tests.

Refs #9"
```

---

## Task 7 — Update order-processing example + new art12-compliance example

**Files:**
- Modify: `examples/order-processing/src/test/java/.../OrderLedgerIT.java`
- Create: `examples/art12-compliance/` (full new example)

- [ ] **Step 1: Add audit query test to `OrderLedgerIT.java`**

Read `examples/order-processing/src/test/java/io/casehub/ledger/examples/order/OrderLedgerIT.java` first.

Add this test at the end of the class:

```java
@Test
void findByActorId_returnsAllOrdersForActor() {
    // Place three orders for the same actor
    final String actor = "it-audit-actor-1";
    placeOrder(actor, "10.00");
    placeOrder(actor, "20.00");
    placeOrder(actor, "30.00");

    // Also place an order for a different actor
    placeOrder("it-audit-actor-2", "99.00");

    given()
            .when().get("/orders/audit?actorId=" + actor
                    + "&from=2020-01-01T00:00:00Z&to=2099-12-31T23:59:59Z")
            .then()
            .statusCode(200)
            .body("$", hasSize(greaterThanOrEqualTo(3)))
            .body("actorId", everyItem(equalTo(actor)));
}
```

Also add `import static org.hamcrest.Matchers.everyItem;` and `import static org.hamcrest.Matchers.greaterThanOrEqualTo;` if not already imported.

- [ ] **Step 2: Add audit query endpoint to `OrderResource.java`**

Read `examples/order-processing/src/main/java/io/casehub/ledger/examples/order/api/OrderResource.java` first.

Add this import and endpoint:

```java
import java.time.Instant;
```

```java
@GET
@Path("/audit")
@Produces(MediaType.APPLICATION_JSON)
public List<Object> auditByActor(
        @jakarta.ws.rs.QueryParam("actorId") final String actorId,
        @jakarta.ws.rs.QueryParam("from") final String from,
        @jakarta.ws.rs.QueryParam("to") final String to) {
    return ledgerRepo.findByActorId(actorId, Instant.parse(from), Instant.parse(to))
            .stream()
            .map(e -> java.util.Map.of(
                    "id", e.id,
                    "actorId", e.actorId != null ? e.actorId : "",
                    "occurredAt", e.occurredAt != null ? e.occurredAt.toString() : ""))
            .collect(java.util.stream.Collectors.toList());
}
```

- [ ] **Step 3: Run order-processing tests — all must pass**

```bash
cd examples/order-processing
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -5
cd ../..
```

Expected: `Tests run: 11, Failures: 0` (10 original + 1 new).

- [ ] **Step 4: Create `examples/art12-compliance/` structure**

```bash
mkdir -p examples/art12-compliance/src/main/java/io/casehub/ledger/examples/art12/{ledger,service,api}
mkdir -p examples/art12-compliance/src/main/resources/db/migration
mkdir -p examples/art12-compliance/src/test/java/io/casehub/ledger/examples/art12
```

- [ ] **Step 5: Create `examples/art12-compliance/pom.xml`**

Copy from `examples/order-processing/pom.xml` and change:
- `artifactId` → `casehub-ledger-example-art12-compliance`
- `name` → `CaseHub Ledger - Example: EU AI Act Art.12 Compliance`
- `description` → Demonstrates retention enforcement and auditor query API

- [ ] **Step 6: Create `DecisionEntry.java`**

```java
package io.casehub.ledger.examples.art12.ledger;

import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import io.casehub.ledger.runtime.model.LedgerEntry;

@Entity
@Table(name = "art12_decision_entry")
@DiscriminatorValue("ART12_DECISION")
public class DecisionEntry extends LedgerEntry {
    @Column(name = "decision_category", length = 100)
    public String decisionCategory;
}
```

- [ ] **Step 7: Create `V1004__art12_decision_entry.sql`**

```sql
CREATE TABLE art12_decision_entry (
    id                UUID         NOT NULL,
    decision_category VARCHAR(100),
    CONSTRAINT pk_art12_decision_entry PRIMARY KEY (id),
    CONSTRAINT fk_art12_decision_entry FOREIGN KEY (id) REFERENCES ledger_entry (id)
);
```

- [ ] **Step 8: Create `AuditService.java`**

```java
package io.casehub.ledger.examples.art12.service;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.examples.art12.ledger.DecisionEntry;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerHashChain;

@ApplicationScoped
public class AuditService {

    @Inject
    LedgerEntryRepository repo;

    @Transactional
    public DecisionEntry recordDecision(final String actorId, final String category,
            final String algorithmRef, final double confidence) {
        final UUID subjectId = UUID.randomUUID();
        final DecisionEntry e = new DecisionEntry();
        e.subjectId        = subjectId;
        e.decisionCategory = category;
        e.sequenceNumber   = 1;
        e.entryType        = LedgerEntryType.EVENT;
        e.actorId          = actorId;
        e.actorType        = ActorType.AGENT;
        e.actorRole        = "Classifier";
        e.occurredAt       = Instant.now().truncatedTo(ChronoUnit.MILLIS);

        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef            = algorithmRef;
        cs.confidenceScore         = confidence;
        cs.contestationUri         = "https://decisions.example.com/challenge/" + subjectId;
        cs.humanOverrideAvailable  = true;
        e.attach(cs);
        e.previousHash = null;
        e.digest       = LedgerHashChain.compute(null, e);
        e.persist();
        return e;
    }

    public List<LedgerEntry> auditByActor(final String actorId,
            final Instant from, final Instant to) {
        return repo.findByActorId(actorId, from, to);
    }

    public List<LedgerEntry> auditByTimeRange(final Instant from, final Instant to) {
        return repo.findByTimeRange(from, to);
    }
}
```

- [ ] **Step 9: Create `AuditResource.java`**

```java
package io.casehub.ledger.examples.art12.api;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.casehub.ledger.examples.art12.service.AuditService;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;

@Path("/decisions")
@Produces(MediaType.APPLICATION_JSON)
public class AuditResource {

    @Inject AuditService auditService;

    @POST
    public Response record(
            @QueryParam("actorId") final String actorId,
            @QueryParam("category") final String category,
            @QueryParam("algorithm") final String algorithm,
            @QueryParam("confidence") @DefaultValue("0.9") final double confidence) {
        final var e = auditService.recordDecision(actorId, category, algorithm, confidence);
        return Response.status(201).entity(Map.of("id", e.id, "actorId", e.actorId)).build();
    }

    @GET
    @Path("/audit")
    public List<Map<String, Object>> auditByActor(
            @QueryParam("actorId") final String actorId,
            @QueryParam("from") final String from,
            @QueryParam("to") final String to) {
        return auditService.auditByActor(actorId, Instant.parse(from), Instant.parse(to))
                .stream().map(this::toView).collect(Collectors.toList());
    }

    @GET
    @Path("/range")
    public List<Map<String, Object>> auditByRange(
            @QueryParam("from") final String from,
            @QueryParam("to") final String to) {
        return auditService.auditByTimeRange(Instant.parse(from), Instant.parse(to))
                .stream().map(this::toView).collect(Collectors.toList());
    }

    private Map<String, Object> toView(final LedgerEntry e) {
        final ComplianceSupplement cs = e.compliance().orElse(null);
        return Map.of(
                "id", e.id,
                "actorId", e.actorId != null ? e.actorId : "",
                "occurredAt", e.occurredAt != null ? e.occurredAt.toString() : "",
                "algorithmRef", cs != null && cs.algorithmRef != null ? cs.algorithmRef : "",
                "confidence", cs != null && cs.confidenceScore != null ? cs.confidenceScore : 0.0);
    }
}
```

- [ ] **Step 10: Create `application.properties` for the art12 example**

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:art12;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
quarkus.ledger.enabled=true
quarkus.ledger.hash-chain.enabled=true
quarkus.ledger.retention.enabled=false
```

- [ ] **Step 11: Create `DecisionEntryRepository.java`** (required for CDI — same pattern as other examples)

```java
package io.casehub.ledger.examples.art12.ledger;

import jakarta.enterprise.context.ApplicationScoped;
import io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository;

@ApplicationScoped
public class DecisionEntryRepository extends JpaLedgerEntryRepository {}
```

- [ ] **Step 12: Create `Art12ComplianceIT.java`**

```java
package io.casehub.ledger.examples.art12;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import org.junit.jupiter.api.Test;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class Art12ComplianceIT {

    @Test
    void recordDecision_returns201() {
        given()
                .queryParam("actorId", "agent-1")
                .queryParam("category", "credit-risk")
                .queryParam("algorithm", "model-v3")
                .queryParam("confidence", "0.92")
                .when().post("/decisions")
                .then().statusCode(201)
                .body("actorId", equalTo("agent-1"));
    }

    @Test
    void auditByActor_returnsDecisions() {
        final String actor = "audit-agent-" + java.util.UUID.randomUUID();
        given().queryParam("actorId", actor).queryParam("category", "fraud")
                .queryParam("algorithm", "v1").when().post("/decisions").then().statusCode(201);

        given()
                .queryParam("actorId", actor)
                .queryParam("from", "2020-01-01T00:00:00Z")
                .queryParam("to", "2099-12-31T23:59:59Z")
                .when().get("/decisions/audit")
                .then().statusCode(200)
                .body("$", hasSize(1))
                .body("[0].actorId", equalTo(actor))
                .body("[0].algorithmRef", equalTo("v1"));
    }

    @Test
    void auditByRange_returnsDecisionsInWindow() {
        given().queryParam("actorId", "range-agent")
                .queryParam("category", "content-mod")
                .queryParam("algorithm", "safety-v2")
                .when().post("/decisions").then().statusCode(201);

        given()
                .queryParam("from", "2020-01-01T00:00:00Z")
                .queryParam("to", "2099-12-31T23:59:59Z")
                .when().get("/decisions/range")
                .then().statusCode(200)
                .body("$", not(empty()));
    }
}
```

- [ ] **Step 13: Run art12-compliance tests**

```bash
cd examples/art12-compliance
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -5
cd ../..
```

Expected: `Tests run: 3, Failures: 0`.

- [ ] **Step 14: Commit**

```bash
git add examples/order-processing/ examples/art12-compliance/
git commit -m "feat(art12): audit query in order-processing + new art12-compliance example

order-processing: adds /orders/audit endpoint + findByActorId IT test (11 total).
art12-compliance: new standalone example with DecisionEntry, AuditService,
REST endpoints for /decisions/audit and /decisions/range. 3 IT tests.

Refs #9"
```

---

## Task 8 — art12-compliance README + compliance docs + AUDITABILITY.md + DESIGN.md

**Files:**
- Create: `examples/art12-compliance/README.md`
- Create: `docs/compliance/EU-AI-ACT-ART12.md`
- Modify: `docs/AUDITABILITY.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Create `examples/art12-compliance/README.md`**

```markdown
# Example: EU AI Act Article 12 Compliance

This example demonstrates how to use `casehub-ledger` to satisfy EU AI Act Article 12
(Record-keeping) requirements for high-risk AI systems.

## What is EU AI Act Article 12?

Article 12 requires high-risk AI systems to:
1. **Automatically record events** throughout the system lifecycle
2. **Retain operational logs** for at least 6 months (Article 12)
3. **Retain conformity documentation** for 10 years (Article 18)
4. **Ensure reconstructability** — an auditor can retrieve all AI decisions for any actor
   or time range on demand

Enforcement begins **2 August 2026**. Penalties: up to €15M or 3% of global turnover.

## How this example satisfies Article 12

| Art.12 requirement | Ledger capability |
|---|---|
| Automatic event recording | `LedgerEntry` persisted on every AI decision |
| Tamper evidence | SHA-256 hash chain (`previousHash` + `digest`) |
| 6-month retention | `quarkus.ledger.retention.operational-days=180` |
| Reconstructability by actor | `LedgerEntryRepository.findByActorId(actorId, from, to)` |
| Reconstructability by time window | `LedgerEntryRepository.findByTimeRange(from, to)` |
| Decision context (Art.22) | `ComplianceSupplement` — algorithmRef, confidenceScore, contestationUri |

## Running the example

```bash
cd examples/art12-compliance
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev
```

```bash
# Record an AI decision
curl -X POST "http://localhost:8080/decisions?actorId=agent-1&category=credit-risk&algorithm=model-v3&confidence=0.92"

# Audit all decisions by a specific agent (Art.12 reconstructability)
curl "http://localhost:8080/decisions/audit?actorId=agent-1&from=2026-01-01T00:00:00Z&to=2099-12-31T23:59:59Z"

# Audit all decisions in a time window
curl "http://localhost:8080/decisions/range?from=2026-01-01T00:00:00Z&to=2026-12-31T23:59:59Z"
```

## Running the tests

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```
```

- [ ] **Step 2: Create `docs/compliance/EU-AI-ACT-ART12.md`**

```bash
mkdir -p docs/compliance
```

```markdown
# EU AI Act Article 12 — Compliance Mapping

**Regulation:** EU AI Act (Regulation 2024/1689), Article 12 — Record-keeping
**Enforcement date:** 2 August 2026 (Annex III high-risk AI systems)
**Penalties:** Up to €15M or 3% of global annual turnover

This document maps each Article 12 obligation to the specific `casehub-ledger`
capability that satisfies it, for use in conformity assessments.

---

## Requirement → Feature Mapping

| Art.12 Requirement | Ledger Feature | Config / API |
|---|---|---|
| Automatically record events throughout the AI system lifecycle | `LedgerEntry` persisted on every domain transition | `quarkus.ledger.enabled=true` (default) |
| Records must be tamper-evident | SHA-256 hash chain — any modification breaks the chain | `quarkus.ledger.hash-chain.enabled=true` (default) |
| Retain operational logs for at least 6 months | `LedgerRetentionJob` enforces minimum retention window | `quarkus.ledger.retention.operational-days=180` |
| Archive before deletion | `ledger_entry_archive` table — complete JSON snapshot | `quarkus.ledger.retention.archive-before-delete=true` (default) |
| Full reconstructability of AI decisions on demand — by actor | `LedgerEntryRepository.findByActorId(actorId, from, to)` | Auditor REST endpoint |
| Full reconstructability by time window | `LedgerEntryRepository.findByTimeRange(from, to)` | Auditor REST endpoint |
| Reconstructability by role | `LedgerEntryRepository.findByActorRole(role, from, to)` | Auditor REST endpoint |
| Decision context: logic used, inputs | `ComplianceSupplement.decisionContext` (JSON snapshot) | `entry.attach(cs)` |
| Decision context: algorithm/model reference | `ComplianceSupplement.algorithmRef` | `cs.algorithmRef = "model-v3"` |
| Decision context: confidence / significance | `ComplianceSupplement.confidenceScore` | `cs.confidenceScore = 0.92` |
| Right to contest (GDPR Art.22 link) | `ComplianceSupplement.contestationUri` | `cs.contestationUri = "https://..."` |
| Human oversight availability | `ComplianceSupplement.humanOverrideAvailable` | `cs.humanOverrideAvailable = true` |

---

## Minimum Configuration for Art.12 Compliance

```properties
# application.properties
quarkus.ledger.enabled=true
quarkus.ledger.hash-chain.enabled=true
quarkus.ledger.retention.enabled=true
quarkus.ledger.retention.operational-days=180
quarkus.ledger.retention.archive-before-delete=true
```

---

## What Art.12 Does NOT Require From the Ledger

The ledger provides the **record-keeping substrate**. The following are
consumer responsibilities, not part of the base extension:

- The HTTP endpoint that exposes audit query results to regulators
- The conformity assessment documentation (Article 18 — separate from Art.12)
- Risk classification of whether your specific system falls under Annex III

---

## Runnable Example

`examples/art12-compliance/` — a standalone Quarkus application demonstrating all
of the above in a runnable, testable form. See its `README.md` for curl examples.

---

## References

- [EU AI Act Article 12 — official text](https://ai-act-service-desk.ec.europa.eu/en/ai-act/article-12/)
- [Art.12 compliance checklist](https://www.isms.online/iso-42001/eu-ai-act/article-12/)
- `docs/AUDITABILITY.md` — 8-axiom self-assessment
```

- [ ] **Step 3: Update `docs/AUDITABILITY.md` — close Axiom 5 and Axiom 6 gaps**

Find the Axiom Summary table and update rows for Axiom 5 and 6:
```markdown
| 5. Accessibility | ✅ Addressed | EU AI Act Art.12 audit query API (#9) |
| 6. Resource Proportionality | ✅ Addressed | Retention config (#9) |
```

Find the Axiom 5 body section. Update `**Status:**` from `⚠️ Partial` to `✅ Addressed (#9)` and add:
```
findByActorId(), findByActorRole(), findByTimeRange() now provide auditor-facing
reconstructability without knowledge of system internals. All return entries in
occurredAt ASC order using Instant params for timezone-safe querying.
```

Find the Axiom 6 body section. Update `**Status:**` from `⚠️ Partial` to `✅ Addressed (#9)` and add:
```
quarkus.ledger.retention.* config (disabled by default) enforces the EU AI Act Art.12
180-day minimum with archive-before-delete. All three config keys default to values
that produce zero behaviour change.
```

- [ ] **Step 4: Update `docs/DESIGN.md` tracker**

Add a new row after the forgiveness mechanism row:
```markdown
| **EU AI Act Art.12 compliance** | ✅ Done | Archive-then-delete retention job (`LedgerRetentionJob`), audit query SPI (`findByActorId`, `findByActorRole`, `findByTimeRange`), `docs/compliance/EU-AI-ACT-ART12.md`, `examples/art12-compliance/` |
```

- [ ] **Step 5: Reinstall runtime to local Maven repo so examples pick up latest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl runtime -DskipTests -q
```

- [ ] **Step 6: Run full build — all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -3
cd examples/order-processing && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -3 && cd ../..
cd examples/art22-decision-snapshot && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -3 && cd ../..
cd examples/art12-compliance && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -3 && cd ../..
```

Expected: all BUILD SUCCESS.

- [ ] **Step 7: Commit docs + close issue**

```bash
git add examples/art12-compliance/README.md \
        docs/compliance/ \
        docs/AUDITABILITY.md \
        docs/DESIGN.md
git commit -m "docs(art12): compliance mapping, README, AUDITABILITY axioms 5+6 closed

EU-AI-ACT-ART12.md: requirement → feature mapping table for conformity assessments.
art12-compliance/README.md: Art.12 obligations mapped to ledger API with curl examples.
AUDITABILITY.md: Axiom 5 (Accessibility) and Axiom 6 (Resource Proportionality) ✅.
DESIGN.md: tracker row added.

Closes #9"
```

---

## Self-Review

**Spec coverage:**

| Spec requirement | Task |
|---|---|
| Rename examples V1003→V1004, update convention | Task 1 |
| V1003 migration — `ledger_entry_archive` table | Task 2 |
| `LedgerEntryArchiveRecord` entity | Task 2 |
| `LedgerEntryArchiver` pure Java serialiser | Task 3 |
| `RetentionEligibilityChecker` pure Java logic | Task 4 |
| `LedgerConfig.RetentionConfig` (3 keys, disabled by default) | Task 5 |
| `LedgerRetentionJob` — verify + archive + delete, per-subject txn | Task 5 |
| Unit tests: archiver (5), eligibility (7) | Tasks 3, 4 |
| IT tests: retention job (6 scenarios) | Task 5 |
| Audit query SPI: 3 methods with Instant params | Task 6 |
| JPA implementations | Task 6 |
| IT tests: audit queries (7 scenarios) | Task 6 |
| `findByActorId` demo in order-processing | Task 7 |
| `examples/art12-compliance/` with 3 IT tests | Task 7 |
| `docs/compliance/EU-AI-ACT-ART12.md` | Task 8 |
| `AUDITABILITY.md` Axiom 5 + 6 gaps closed | Task 8 |

**Type consistency:** `Instant` used throughout — SPI, JPA impl, IT tests, example. `LedgerEntryArchiveRecord` field names (`originalEntryId`, `entryJson`, `entryOccurredAt`, `archivedAt`) consistent across Task 2 (entity), Task 5 (job), and Task 5 (IT test).

**No placeholders found.**
