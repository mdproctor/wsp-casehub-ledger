# Causality & Observability to Core + findCausedBy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move `correlationId` and `causedByEntryId` from `ObservabilitySupplement` to `LedgerEntry` core, delete `ObservabilitySupplement` entirely, and add `findCausedBy(UUID)` to the SPI for cross-system causal chain traversal.

**Architecture:** No new migrations — edit existing SQL source files in place (pre-release, no deployed data). Add two fields to `LedgerEntry`, delete `ObservabilitySupplement`, update serialiser/archiver/tests, add one SPI method with JPA implementation, and a `@QuarkusTest` integration test covering the full Claudony→Tarkus→Qhorus orchestration scenario.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM / Panache, JUnit 5, AssertJ.

**All commits:** `Refs #10`.
**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl runtime -q`

**NOTE on `mvn clean`:** Editing V1000/V1002 SQL files changes their Flyway checksum. Always use `mvn clean test` (not `mvn test`) to get a fresh H2 database and avoid checksum mismatch errors.

---

## File Map

| File | Change |
|---|---|
| `runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql` | Add `idx_ledger_entry_caused_by` index; update comment |
| `runtime/src/main/resources/db/migration/V1002__ledger_supplement.sql` | Remove DROP COLUMN for correlation_id and caused_by_entry_id; remove CREATE TABLE ledger_supplement_observability |
| `runtime/src/main/java/.../model/LedgerEntry.java` | Add `correlationId` and `causedByEntryId` fields; remove `observability()` |
| `runtime/src/main/java/.../model/supplement/ObservabilitySupplement.java` | **Delete** |
| `runtime/src/main/java/.../model/supplement/LedgerSupplementSerializer.java` | Remove OBSERVABILITY branch |
| `runtime/src/main/java/.../service/LedgerEntryArchiver.java` | Add correlationId + causedByEntryId to core field block |
| `runtime/src/test/java/.../service/supplement/LedgerSupplementSerializerTest.java` | Remove OBSERVABILITY tests; update multi-supplement test |
| `runtime/src/test/java/.../service/supplement/LedgerSupplementIT.java` | Remove observability IT; update multi-supplement IT |
| `runtime/src/test/java/.../service/LedgerEntryArchiverTest.java` | Add correlationId + causedByEntryId tests |
| `runtime/src/main/java/.../repository/LedgerEntryRepository.java` | Add `findCausedBy(UUID)` |
| `runtime/src/main/java/.../repository/jpa/JpaLedgerEntryRepository.java` | Implement `findCausedBy(UUID)` |
| `examples/order-processing/src/main/java/.../ledger/OrderLedgerEntryRepository.java` | Add `findCausedBy(UUID)` |
| `runtime/src/test/java/.../service/CausalityQueryIT.java` | **Create** — 6 @QuarkusTest cases |
| `examples/order-processing/src/test/java/.../OrderLedgerIT.java` | Add correlationId happy-path test |
| `docs/AUDITABILITY.md` | Axiom 3 → ✅ Addressed (#10) |
| `docs/DESIGN.md` | Update tracker |

---

## Task 1 — Entity refactor: LedgerEntry + ObservabilitySupplement removal + schema source edits

**Files:** See map above (all except the IT test files and docs)

This task has no TDD gate — it's pure refactoring. The gate is `mvn clean test` passing at the end with existing tests.

- [ ] **Step 1: Edit V1000 — add caused_by index and update comment**

In `runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql`:

After the line `CREATE INDEX idx_ledger_entry_correlation ON ledger_entry (correlation_id);`, add:
```sql
CREATE INDEX idx_ledger_entry_caused_by   ON ledger_entry (caused_by_entry_id);
```

Update the comment block (lines 10-11) to:
```sql
-- Note: Optional supplement fields (plan_ref, rationale, decision_context, etc.) are present here
-- for schema compatibility. V1002 migrates these to supplement tables and drops them.
-- correlation_id and caused_by_entry_id are KEPT on this table — they are core fields.
```

- [ ] **Step 2: Edit V1002 — remove DROP statements and observability table**

In `runtime/src/main/resources/db/migration/V1002__ledger_supplement.sql`:

Remove these two lines:
```sql
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS correlation_id;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS caused_by_entry_id;
```

Remove the entire `-- ── ObservabilitySupplement ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────` section — that is, the `CREATE TABLE ledger_supplement_observability` block and its constraints.

- [ ] **Step 3: Add `correlationId` and `causedByEntryId` to `LedgerEntry.java`**

Read `LedgerEntry.java` first. After the `digest` field and before `// ── Supplements ───`, add:

```java
    // ── Observability & causality ─────────────────────────────────────────────

    /**
     * OpenTelemetry trace ID linking this entry to a distributed trace.
     * Use the W3C trace context format (32-char hex string).
     * Set from the active OTel span context at capture time.
     */
    @Column(name = "correlation_id", length = 255)
    public String correlationId;

    /**
     * FK to the ledger entry that causally produced this entry.
     * Null for entries with no known causal predecessor.
     *
     * <p>
     * Enables cross-system causal chain traversal via
     * {@link io.casehub.ledger.runtime.repository.LedgerEntryRepository#findCausedBy(UUID)}.
     * When Claudony orchestrates Tarkus → Qhorus, each downstream entry's
     * {@code causedByEntryId} points to its upstream cause.
     */
    @Column(name = "caused_by_entry_id")
    public UUID causedByEntryId;
```

Also remove the `observability()` method and its `ObservabilitySupplement` import.
Update the class Javadoc supplements `<ul>` to remove the ObservabilitySupplement bullet.

- [ ] **Step 4: Delete `ObservabilitySupplement.java`**

```bash
rm runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ObservabilitySupplement.java
```

- [ ] **Step 5: Remove OBSERVABILITY from `LedgerSupplementSerializer.java`**

Read the file. Remove:
- The `if (supplement instanceof ObservabilitySupplement)` check in `typeKey()`
- The `} else if (supplement instanceof final ObservabilitySupplement o) {` block in `toFieldMap()`
- The `ObservabilitySupplement` import

- [ ] **Step 6: Add core fields to `LedgerEntryArchiver.java`**

Read the file. After `if (entry.digest != null) map.put("digest", entry.digest);`, add:

```java
        if (entry.correlationId != null)
            map.put("correlationId", entry.correlationId);
        if (entry.causedByEntryId != null)
            map.put("causedByEntryId", entry.causedByEntryId.toString());
```

- [ ] **Step 7: Update `LedgerSupplementSerializerTest.java`**

Read the file. Remove `toJson_observabilitySupplement_serialisedCorrectly` entirely.

Replace `toJson_multipleSupplements_allPresent` with:

```java
@Test
void toJson_multipleSupplements_allPresent() {
    final ComplianceSupplement cs = new ComplianceSupplement();
    cs.algorithmRef = "v1";
    final ProvenanceSupplement ps = new ProvenanceSupplement();
    ps.sourceEntitySystem = "quarkus-flow";

    final String json = LedgerSupplementSerializer.toJson(List.of(cs, ps));

    assertThat(json).contains("\"COMPLIANCE\"");
    assertThat(json).contains("\"PROVENANCE\"");
    assertThat(json).contains("algorithmRef");
    assertThat(json).contains("quarkus-flow");
}
```

Remove `ObservabilitySupplement` import.

- [ ] **Step 8: Update `LedgerSupplementIT.java`**

Read the file. Remove `observabilitySupplement_persistsAndLoads` test entirely.

Replace `supplementJson_containsAllAttachedSupplements` with:

```java
@Test
@Transactional
void supplementJson_containsTwoSupplementTypes() {
    final TestEntry entry = bareEntry();

    final ComplianceSupplement cs = new ComplianceSupplement();
    cs.algorithmRef = "v1";
    entry.attach(cs);

    final ProvenanceSupplement ps = new ProvenanceSupplement();
    ps.sourceEntitySystem = "quarkus-flow";
    entry.attach(ps);
    entry.persist();

    final TestEntry found = TestEntry.findById(entry.id);
    assertThat(found.supplementJson).contains("\"COMPLIANCE\"");
    assertThat(found.supplementJson).contains("\"PROVENANCE\"");
    assertThat(found.supplementJson).contains("quarkus-flow");
}
```

Remove `ObservabilitySupplement` import.

- [ ] **Step 9: Add tests to `LedgerEntryArchiverTest.java`**

Read the file. Add before the closing `}`:

```java
@Test
void toJson_correlationId_included() {
    final TestEntry e = entry("agent-1");
    e.correlationId = "trace-abc123";

    final String json = LedgerEntryArchiver.toJson(e, List.of());

    assertThat(json).contains("\"correlationId\":\"trace-abc123\"");
}

@Test
void toJson_causedByEntryId_included() {
    final TestEntry e = entry("agent-1");
    final UUID parentId = UUID.randomUUID();
    e.causedByEntryId = parentId;

    final String json = LedgerEntryArchiver.toJson(e, List.of());

    assertThat(json).contains("\"causedByEntryId\":\"" + parentId + "\"");
}

@Test
void toJson_nullObservabilityFields_omitted() {
    final TestEntry e = entry("agent-1");
    e.correlationId   = null;
    e.causedByEntryId = null;

    final String json = LedgerEntryArchiver.toJson(e, List.of());

    assertThat(json).doesNotContain("correlationId");
    assertThat(json).doesNotContain("causedByEntryId");
}
```

- [ ] **Step 10: Run full clean test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl runtime -q 2>&1 | tail -8
```

Expected: BUILD SUCCESS. (`clean` is mandatory — Flyway checksum changed on V1000/V1002.)

- [ ] **Step 11: Commit**

```bash
git add runtime/src/main/resources/db/migration/ \
        runtime/src/main/java/ \
        runtime/src/test/java/
git commit -m "feat(causality): correlationId + causedByEntryId to LedgerEntry core; delete ObservabilitySupplement

Both fields are structural (universal OTel correlation, fundamental causal link)
not optional enrichment. ObservabilitySupplement deleted — its fields now live
directly on LedgerEntry.

V1000: add caused_by index (column already existed, just missing index).
V1002: stop dropping correlation_id + caused_by_entry_id; remove observability table.
LedgerEntryArchiver: both fields included in archive JSON.
LedgerSupplementSerializer: OBSERVABILITY branch removed.
Tests updated: 3 new archiver tests, supplement tests use ProvenanceSupplement.

Refs #10"
```

---

## Task 2 — `findCausedBy` SPI + JPA + IT tests (TDD — tests first)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/CausalityQueryIT.java`
- Modify: `runtime/src/main/java/.../repository/LedgerEntryRepository.java`
- Modify: `runtime/src/main/java/.../repository/jpa/JpaLedgerEntryRepository.java`
- Modify: `examples/order-processing/src/main/java/.../ledger/OrderLedgerEntryRepository.java`

- [ ] **Step 1: Write the failing tests**

Create `runtime/src/test/java/io/casehub/ledger/service/CausalityQueryIT.java`:

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
 * Integration tests for {@link LedgerEntryRepository#findCausedBy(UUID)}.
 *
 * <p>
 * Covers cross-system causal chain traversal: Claudony orchestrates Tarkus
 * which triggers Qhorus — each entry's {@code causedByEntryId} points upstream.
 */
@QuarkusTest
class CausalityQueryIT {

    @Inject
    LedgerEntryRepository repo;

    // ── happy path: direct effects ────────────────────────────────────────────

    @Test
    @Transactional
    void findCausedBy_rootEntry_returnsDirectEffects() {
        final UUID rootId  = UUID.randomUUID();
        final TestEntry ea = seedEntry("tarkus-worker", now().minus(2, ChronoUnit.MINUTES), rootId);
        final TestEntry eb = seedEntry("qhorus-agent",  now().minus(1, ChronoUnit.MINUTES), rootId);
        seedEntry("unrelated", now(), null); // no causal link — must not appear

        final List<LedgerEntry> results = repo.findCausedBy(rootId);

        assertThat(results).hasSize(2);
        assertThat(results.stream().map(e -> e.id).toList())
                .containsExactlyInAnyOrder(ea.id, eb.id);
    }

    @Test
    @Transactional
    void findCausedBy_midChain_returnsOneHop() {
        final UUID rootId   = UUID.randomUUID();
        final TestEntry tarkus = seedEntry("tarkus", now().minus(2, ChronoUnit.MINUTES), rootId);
        final TestEntry qhorus = seedEntry("qhorus", now().minus(1, ChronoUnit.MINUTES), tarkus.id);

        final List<LedgerEntry> results = repo.findCausedBy(tarkus.id);

        assertThat(results).hasSize(1);
        assertThat(results.get(0).id).isEqualTo(qhorus.id);
    }

    @Test
    @Transactional
    void findCausedBy_leaf_returnsEmpty() {
        final UUID rootId   = UUID.randomUUID();
        final TestEntry leaf = seedEntry("leaf", now(), rootId);

        assertThat(repo.findCausedBy(leaf.id)).isEmpty();
    }

    @Test
    @Transactional
    void findCausedBy_unknownId_returnsEmpty() {
        assertThat(repo.findCausedBy(UUID.randomUUID())).isEmpty();
    }

    @Test
    @Transactional
    void findCausedBy_orderedByOccurredAtAsc() {
        final UUID rootId   = UUID.randomUUID();
        final TestEntry late  = seedEntry("agent-b", now().minus(1, ChronoUnit.MINUTES), rootId);
        final TestEntry early = seedEntry("agent-a", now().minus(5, ChronoUnit.MINUTES), rootId);

        final List<LedgerEntry> results = repo.findCausedBy(rootId);

        assertThat(results).hasSize(2);
        assertThat(results.get(0).id).isEqualTo(early.id);
        assertThat(results.get(1).id).isEqualTo(late.id);
    }

    // ── happy path: full two-hop orchestration scenario ───────────────────────

    @Test
    @Transactional
    void fullOrchestrationChain_claudonyTarkusQhorus() {
        // Claudony orchestration step (external — just its UUID, not a ledger entry here)
        final UUID claudonyId = UUID.randomUUID();

        // Tarkus work item triggered by Claudony
        final TestEntry tarkus = seedEntry("tarkus-worker",
                now().minus(3, ChronoUnit.MINUTES), claudonyId);

        // Qhorus agent message triggered by Tarkus
        final TestEntry qhorus = seedEntry("qhorus-agent",
                now().minus(1, ChronoUnit.MINUTES), tarkus.id);

        // Hop 1: claudony → tarkus
        final List<LedgerEntry> hop1 = repo.findCausedBy(claudonyId);
        assertThat(hop1).hasSize(1);
        assertThat(hop1.get(0).id).isEqualTo(tarkus.id);
        assertThat(hop1.get(0).actorId).isEqualTo("tarkus-worker");

        // Hop 2: tarkus → qhorus
        final List<LedgerEntry> hop2 = repo.findCausedBy(tarkus.id);
        assertThat(hop2).hasSize(1);
        assertThat(hop2.get(0).id).isEqualTo(qhorus.id);
        assertThat(hop2.get(0).actorId).isEqualTo("qhorus-agent");

        // Leaf: qhorus causes nothing
        assertThat(repo.findCausedBy(qhorus.id)).isEmpty();
    }

    // ── fixture ───────────────────────────────────────────────────────────────

    private TestEntry seedEntry(final String actorId, final Instant occurredAt,
            final UUID causedByEntryId) {
        final TestEntry e = new TestEntry();
        e.subjectId       = UUID.randomUUID();
        e.sequenceNumber  = 1;
        e.entryType       = LedgerEntryType.EVENT;
        e.actorId         = actorId;
        e.actorType       = ActorType.AGENT;
        e.actorRole       = "Processor";
        e.occurredAt      = occurredAt;
        e.causedByEntryId = causedByEntryId;
        e.persist();
        return e;
    }

    private Instant now() { return Instant.now(); }
}
```

- [ ] **Step 2: Run — confirm compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl runtime \
  -Dtest=CausalityQueryIT -q 2>&1 | tail -5
```

Expected: compilation error — `findCausedBy` not found.

- [ ] **Step 3: Add `findCausedBy` to `LedgerEntryRepository.java`**

Read the file. After `findByTimeRange`, add:

```java
    /**
     * Return all ledger entries causally triggered by the given entry,
     * ordered by {@code occurredAt} ascending. One hop only — recursive
     * traversal is the caller's responsibility.
     *
     * @param entryId the entry whose direct effects to retrieve
     * @return ordered list; empty if none
     */
    List<LedgerEntry> findCausedBy(UUID entryId);
```

- [ ] **Step 4: Implement in `JpaLedgerEntryRepository.java`**

Read the file. After `findByTimeRange` implementation, add:

```java
    /** {@inheritDoc} */
    @Override
    public List<LedgerEntry> findCausedBy(final UUID entryId) {
        return LedgerEntry.list(
                "causedByEntryId = ?1 ORDER BY occurredAt ASC", entryId);
    }
```

- [ ] **Step 5: Add `findCausedBy` to `OrderLedgerEntryRepository.java`**

Read `examples/order-processing/src/main/java/io/casehub/ledger/examples/order/ledger/OrderLedgerEntryRepository.java`. Add after the last override:

```java
    @Override
    public java.util.List<io.casehub.ledger.runtime.model.LedgerEntry> findCausedBy(
            final java.util.UUID entryId) {
        return io.casehub.ledger.runtime.model.LedgerEntry.list(
                "causedByEntryId = ?1 ORDER BY occurredAt ASC", entryId);
    }
```

- [ ] **Step 6: Run — 6 IT tests must pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl runtime \
  -Dtest=CausalityQueryIT -q 2>&1 | tail -5
```

Expected: `Tests run: 6, Failures: 0, Errors: 0`.

- [ ] **Step 7: Run full clean suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/ \
        runtime/src/test/java/io/casehub/ledger/service/CausalityQueryIT.java \
        examples/order-processing/
git commit -m "feat(causality): findCausedBy — SPI + JPA + 6 @QuarkusTest IT tests

One-hop causal traversal. Happy path: full Claudony→Tarkus→Qhorus two-hop
scenario verified. Ordered by occurredAt ASC.

Refs #10"
```

---

## Task 3 — End-to-end happy path + documentation

**Files:**
- Modify: `examples/order-processing/src/test/java/.../OrderLedgerIT.java`
- Modify: `docs/AUDITABILITY.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Add `correlationId` end-to-end test to `OrderLedgerIT.java`**

Read `examples/order-processing/src/main/java/io/casehub/ledger/examples/order/api/OrderResource.java`. Find the `LedgerEntryView` record. Add `String correlationId` to it and map `e.correlationId` in the constructor. If there is no view record and entries are returned directly, `correlationId` will already appear in the JSON — skip this sub-step.

Read `examples/order-processing/src/test/java/io/casehub/ledger/examples/order/OrderLedgerIT.java`. Add:

```java
@Test
void placeOrder_correlationId_fieldExistsInResponse() {
    // correlationId is now a core field on LedgerEntry — the column exists and persists.
    // OrderService does not currently populate it (no OTel in the example),
    // but the field must be present in the JSON response (null is valid).
    final String orderId = placeOrder("it-otel-1", "75.00");

    given()
            .when().get(BASE + "/" + orderId + "/ledger")
            .then()
            .statusCode(200)
            .body("[0]", org.hamcrest.Matchers.hasKey("correlationId"));
}
```

- [ ] **Step 2: Run order-processing tests**

```bash
cd /Users/mdproctor/claude/casehub/ledger/examples/order-processing
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -q 2>&1 | tail -5
cd ../..
```

Expected: BUILD SUCCESS.

If `correlationId` is not in the response view, read `OrderResource.java` and add it to the view record.

- [ ] **Step 3: Update `docs/AUDITABILITY.md`**

Read the file. In the Axiom Summary table update row 3:
```
| 3. Temporal Coherence | ✅ Addressed | `causedByEntryId` core field + `findCausedBy()` (#10) |
```

In the Axiom 3 body section, change `**Status:**` from `⚠️ Partial` to `✅ Addressed (#10)`. After the Gap section add:

```markdown
**Addressed by (#10):**
`causedByEntryId` is a core nullable field on `LedgerEntry` — consumers set it
directly when an entry is causally triggered by another. `correlationId` is also
core, linking entries to OTel distributed traces. `findCausedBy(UUID)` enables
one-hop traversal; recursive chain reconstruction is application-level.
```

- [ ] **Step 4: Update `docs/DESIGN.md` tracker**

Add row after the forgiveness row:
```markdown
| **Causality & Observability to core** | ✅ Done | `correlationId` + `causedByEntryId` on `LedgerEntry`; `ObservabilitySupplement` deleted; `findCausedBy()` SPI |
```

- [ ] **Step 5: Reinstall runtime and verify all examples**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl runtime -DskipTests -q
cd examples/art22-decision-snapshot && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -q 2>&1 | tail -3 && cd ../..
cd examples/art12-compliance && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -q 2>&1 | tail -3 && cd ../..
```

Expected: all BUILD SUCCESS. These examples extend `JpaLedgerEntryRepository` which already has `findCausedBy`.

- [ ] **Step 6: Commit and push**

```bash
git add examples/order-processing/ docs/AUDITABILITY.md docs/DESIGN.md
git commit -m "feat(causality): e2e correlationId test + docs (Axiom 3 ✅)

OrderLedgerIT: correlationId field present in API response (core field, schema verified).
AUDITABILITY.md: Axiom 3 (Temporal Coherence) ✅ Addressed (#10).
DESIGN.md: tracker row added.

Closes #10"
git push origin main
```

---

## Self-Review

**Spec coverage:**

| Requirement | Task |
|---|---|
| V1000: add caused_by index | Task 1 |
| V1002: keep correlation_id + caused_by on ledger_entry | Task 1 |
| V1002: remove observability table | Task 1 |
| `LedgerEntry.correlationId` + `causedByEntryId` | Task 1 |
| `ObservabilitySupplement` deleted | Task 1 |
| OBSERVABILITY removed from serialiser | Task 1 |
| Both fields in archiver JSON | Task 1 |
| Existing tests updated | Task 1 |
| 3 new archiver unit tests | Task 1 |
| `findCausedBy(UUID)` SPI + JPA | Task 2 |
| OrderLedgerEntryRepository updated | Task 2 |
| 6 `@QuarkusTest` IT cases incl. full e2e orchestration | Task 2 |
| correlationId in order-processing response | Task 3 |
| AUDITABILITY.md Axiom 3 ✅ | Task 3 |
| DESIGN.md tracker | Task 3 |

**No placeholders.** All code complete.
