# Qhorus Query APIs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #202 — Streaming/cursor-based query API for offline verification
**Issue group:** #202, #201

**Goal:** Add streaming, cursor-based pagination, and aggregate query methods to
`LedgerEntryRepository` SPI for qhorus formal verification (E7) and compliance
evidence export (E5).

**Architecture:** Six new methods on the existing `LedgerEntryRepository` SPI + one
`AttestationSummary` record in `api/model/`. Default methods on the SPI return empty
results. Overrides in InMemoryLedgerEntryRepository and JpaLedgerEntryRepository
provide real behavior. All JPA queries use `@NamedQuery` per protocol PP-20260618-51c673.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, H2 (tests)

## Global Constraints

- All JPQL must be `@NamedQuery` on the entity class — never `em.createQuery()` (PP-20260618-51c673)
- All queries take `tenancyId` parameter — no exceptions (PP-20260616-05dc6a)
- `Stream<LedgerEntry>` callers must use try-with-resources within a transaction
- Time ranges `[from, to]` are inclusive on both bounds — consistent with existing methods
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime` to run tests

---

## Batch 1: SPI Contract + Model

### Task 1: AttestationSummary record + SPI default methods

**Files:**
- Create: `api/src/main/java/io/casehub/ledger/api/model/AttestationSummary.java`
- Modify: `api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryRepository.java`

**Interfaces:**
- Produces: `AttestationSummary` record, 6 new default methods on `LedgerEntryRepository`

- [ ] **Step 1: Create AttestationSummary record**

```java
package io.casehub.ledger.api.model;

import java.util.Map;

public record AttestationSummary(
    Map<AttestationVerdict, Long> verdictCounts,
    long totalAttestations,
    double meanConfidence,
    double minConfidence,
    double maxConfidence
) {
    public static final AttestationSummary EMPTY =
            new AttestationSummary(Map.of(), 0, 0.0, 0.0, 0.0);
}
```

- [ ] **Step 2: Add 6 default methods to LedgerEntryRepository**

Add after the existing `findPeerAttestationPairCounts` default method:

```java
// ── Streaming queries (#202) ─────────────────────────────────────────

default Stream<LedgerEntry> streamBySubjectId(UUID subjectId, String tenancyId) {
    return findBySubjectId(subjectId, tenancyId).stream();
}

default Stream<LedgerEntry> streamByActorId(String actorId, Instant from, Instant to, String tenancyId) {
    return findByActorId(actorId, from, to, tenancyId).stream();
}

// ── Cursor-based pagination (#202) ───────────────────────────────────

default List<LedgerEntry> findBySubjectIdPaged(UUID subjectId, int afterSequence, int limit, String tenancyId) {
    return findBySubjectId(subjectId, tenancyId).stream()
            .filter(e -> e.sequenceNumber > afterSequence)
            .limit(limit)
            .toList();
}

// ── Aggregate queries (#201) ─────────────────────────────────────────

default Map<AttestationVerdict, Long> countByActorAndVerdict(String actorId, Instant from, Instant to, String tenancyId) {
    return Map.of();
}

default Map<AttestationVerdict, Long> countBySubjectAndVerdict(UUID subjectId, Instant from, Instant to, String tenancyId) {
    return Map.of();
}

default AttestationSummary summariseAttestationsByActor(String actorId, Instant from, Instant to, String tenancyId) {
    return AttestationSummary.EMPTY;
}
```

Add required imports: `java.util.stream.Stream`, `io.casehub.ledger.api.model.AttestationSummary`, `io.casehub.ledger.api.model.AttestationVerdict`.

- [ ] **Step 3: Build api module to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 4: Build all modules to verify no downstream breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile`
Expected: BUILD SUCCESS — defaults satisfy all implementations

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/model/AttestationSummary.java
git add api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryRepository.java
git commit -m "feat(#202): SPI contract — streaming, cursor, aggregate default methods

Adds 6 default methods to LedgerEntryRepository and AttestationSummary
record. Defaults delegate to existing methods or return empty results.

Refs #202, Refs #201"
```

---

## Batch 2: In-Memory Implementation + Tests

### Task 2: Streaming + cursor in-memory implementation and tests

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java`
- Create: `runtime/src/test/java/io/casehub/ledger/repository/StreamingQueryIT.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository.streamBySubjectId(UUID, String)`, `streamByActorId(String, Instant, Instant, String)`, `findBySubjectIdPaged(UUID, int, int, String)`
- Produces: Working in-memory implementations, test coverage

- [ ] **Step 1: Write failing streaming tests**

```java
package io.casehub.ledger.repository;

import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.stream.Stream;

import static io.casehub.ledger.api.model.AttestationVerdict.SOUND;
import static io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(StreamingQueryIT.Profile.class)
class StreamingQueryIT {

    public static class Profile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "outcome-recorder-test";
        }
    }

    @Inject LedgerEntryRepository repo;
    @Inject OutcomeRecorder recorder;

    @Test
    void streamBySubjectId_returnsEntriesInSequenceOrder() {
        final UUID subjectId = UUID.randomUUID();
        final String actor = "agent-" + UUID.randomUUID();

        recorder.record(OutcomeRecord.of(actor, subjectId, "routing", SOUND, 0.8));
        recorder.record(OutcomeRecord.of(actor, subjectId, "routing", SOUND, 0.9));
        recorder.record(OutcomeRecord.of(actor, subjectId, "routing", SOUND, 0.7));

        try (Stream<LedgerEntry> stream = repo.streamBySubjectId(subjectId, DEFAULT_TENANT_ID)) {
            final List<LedgerEntry> entries = stream.toList();
            assertThat(entries).hasSize(3);
            assertThat(entries.get(0).sequenceNumber).isLessThan(entries.get(1).sequenceNumber);
            assertThat(entries.get(1).sequenceNumber).isLessThan(entries.get(2).sequenceNumber);
        }
    }

    @Test
    void streamByActorId_filtersOnTimeRange() {
        final UUID subjectId = UUID.randomUUID();
        final String actor = "agent-" + UUID.randomUUID();
        final Instant now = Instant.now();

        recorder.record(OutcomeRecord.of(actor, subjectId, "classify", SOUND, 0.8)
                .withOccurredAt(now.minusSeconds(3600)));
        recorder.record(OutcomeRecord.of(actor, subjectId, "classify", SOUND, 0.9)
                .withOccurredAt(now.minusSeconds(1800)));
        recorder.record(OutcomeRecord.of(actor, subjectId, "classify", SOUND, 0.7)
                .withOccurredAt(now.plusSeconds(3600)));

        try (Stream<LedgerEntry> stream = repo.streamByActorId(
                actor, now.minusSeconds(7200), now, DEFAULT_TENANT_ID)) {
            final List<LedgerEntry> entries = stream.toList();
            assertThat(entries).hasSize(2);
        }
    }

    @Test
    void findBySubjectIdPaged_returnsPagesCorrectly() {
        final UUID subjectId = UUID.randomUUID();
        final String actor = "agent-" + UUID.randomUUID();

        for (int i = 0; i < 5; i++) {
            recorder.record(OutcomeRecord.of(actor, subjectId, "routing", SOUND, 0.8));
        }

        final List<LedgerEntry> page1 = repo.findBySubjectIdPaged(
                subjectId, 0, 2, DEFAULT_TENANT_ID);
        assertThat(page1).hasSize(2);

        final List<LedgerEntry> page2 = repo.findBySubjectIdPaged(
                subjectId, page1.get(1).sequenceNumber, 2, DEFAULT_TENANT_ID);
        assertThat(page2).hasSize(2);

        final List<LedgerEntry> page3 = repo.findBySubjectIdPaged(
                subjectId, page2.get(1).sequenceNumber, 2, DEFAULT_TENANT_ID);
        assertThat(page3).hasSize(1);
    }

    @Test
    void streamBySubjectId_emptyForUnknownSubject() {
        try (Stream<LedgerEntry> stream = repo.streamBySubjectId(
                UUID.randomUUID(), DEFAULT_TENANT_ID)) {
            assertThat(stream.count()).isZero();
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they pass via default methods**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=StreamingQueryIT`
Expected: PASS — defaults delegate to existing `findBySubjectId`/`findByActorId`

Note: The defaults already provide working (non-streaming) behavior by wrapping
`List.stream()`. The in-memory override adds explicit filtering and ordering for
correctness with the `allEntries()` backing store rather than depending on the
default's delegation chain.

- [ ] **Step 3: Override streaming + cursor methods in InMemoryLedgerEntryRepository**

Add to `InMemoryLedgerEntryRepository`:

```java
@Override
public Stream<LedgerEntry> streamBySubjectId(final UUID subjectId, final String tenancyId) {
    return entries.values().stream()
            .filter(e -> subjectId.equals(e.subjectId) && tenancyId.equals(e.tenancyId))
            .sorted(Comparator.comparingInt(e -> e.sequenceNumber));
}

@Override
public Stream<LedgerEntry> streamByActorId(final String actorId, final Instant from,
                                            final Instant to, final String tenancyId) {
    return entries.values().stream()
            .filter(e -> actorId.equals(e.actorId)
                    && tenancyId.equals(e.tenancyId)
                    && !e.occurredAt.isBefore(from)
                    && !e.occurredAt.isAfter(to))
            .sorted(Comparator.comparing(e -> e.occurredAt));
}

@Override
public List<LedgerEntry> findBySubjectIdPaged(final UUID subjectId, final int afterSequence,
                                               final int limit, final String tenancyId) {
    return entries.values().stream()
            .filter(e -> subjectId.equals(e.subjectId)
                    && tenancyId.equals(e.tenancyId)
                    && e.sequenceNumber > afterSequence)
            .sorted(Comparator.comparingInt(e -> e.sequenceNumber))
            .limit(limit)
            .toList();
}
```

Add import: `java.util.Comparator`.

- [ ] **Step 4: Run tests to verify overrides**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=StreamingQueryIT`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java
git add runtime/src/test/java/io/casehub/ledger/repository/StreamingQueryIT.java
git commit -m "feat(#202): streaming + cursor queries — in-memory impl + tests

Overrides streamBySubjectId, streamByActorId, findBySubjectIdPaged in
InMemoryLedgerEntryRepository. Tests verify ordering, time-range
filtering, and cursor pagination correctness.

Refs #202"
```

### Task 3: Aggregate in-memory implementation + qhorus derivation tests

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java`
- Create: `runtime/src/test/java/io/casehub/ledger/repository/AggregateQueryIT.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository.countByActorAndVerdict(String, Instant, Instant, String)`, `countBySubjectAndVerdict(UUID, Instant, Instant, String)`, `summariseAttestationsByActor(String, Instant, Instant, String)`, `AttestationSummary`
- Produces: Working aggregate implementations, qhorus derivation test coverage

- [ ] **Step 1: Write failing aggregate + qhorus derivation tests**

```java
package io.casehub.ledger.repository;

import io.casehub.ledger.api.model.AttestationSummary;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

import static io.casehub.ledger.api.model.AttestationVerdict.*;
import static io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(AggregateQueryIT.Profile.class)
class AggregateQueryIT {

    public static class Profile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "outcome-recorder-test";
        }
    }

    @Inject LedgerEntryRepository repo;
    @Inject OutcomeRecorder recorder;

    // ── Qhorus derivation: fulfillment rate ──────────────────────────────

    @Test
    void fulfillmentRate_derivedFromVerdictCounts() {
        final String actor = "agent-" + UUID.randomUUID();
        final UUID subjectId = UUID.randomUUID();
        final Instant now = Instant.now();

        recorder.record(OutcomeRecord.of(actor, subjectId, "routing", ENDORSED, 0.9)
                .withOccurredAt(now.minusSeconds(100)));
        recorder.record(OutcomeRecord.of(actor, subjectId, "routing", ENDORSED, 0.8)
                .withOccurredAt(now.minusSeconds(80)));
        recorder.record(OutcomeRecord.of(actor, subjectId, "routing", CHALLENGED, 0.7)
                .withOccurredAt(now.minusSeconds(60)));

        final Map<AttestationVerdict, Long> counts = repo.countByActorAndVerdict(
                actor, now.minusSeconds(200), now, DEFAULT_TENANT_ID);

        assertThat(counts.getOrDefault(ENDORSED, 0L)).isEqualTo(2);
        assertThat(counts.getOrDefault(CHALLENGED, 0L)).isEqualTo(1);

        final double fulfillmentRate = (double) counts.getOrDefault(ENDORSED, 0L)
                / (counts.getOrDefault(ENDORSED, 0L) + counts.getOrDefault(CHALLENGED, 0L));
        assertThat(fulfillmentRate).isCloseTo(0.6667, org.assertj.core.data.Offset.offset(0.01));
    }

    // ── Qhorus derivation: per-channel quality ───────────────────────────

    @Test
    void perChannelQuality_derivedFromSubjectVerdictCounts() {
        final String actor = "agent-" + UUID.randomUUID();
        final UUID channel1 = UUID.randomUUID();
        final UUID channel2 = UUID.randomUUID();
        final Instant now = Instant.now();

        recorder.record(OutcomeRecord.of(actor, channel1, "classify", ENDORSED, 0.9)
                .withOccurredAt(now.minusSeconds(100)));
        recorder.record(OutcomeRecord.of(actor, channel1, "classify", ENDORSED, 0.8)
                .withOccurredAt(now.minusSeconds(80)));
        recorder.record(OutcomeRecord.of(actor, channel2, "classify", CHALLENGED, 0.6)
                .withOccurredAt(now.minusSeconds(60)));

        final Map<AttestationVerdict, Long> ch1 = repo.countBySubjectAndVerdict(
                channel1, now.minusSeconds(200), now, DEFAULT_TENANT_ID);
        final Map<AttestationVerdict, Long> ch2 = repo.countBySubjectAndVerdict(
                channel2, now.minusSeconds(200), now, DEFAULT_TENANT_ID);

        assertThat(ch1.getOrDefault(ENDORSED, 0L)).isEqualTo(2);
        assertThat(ch1.getOrDefault(CHALLENGED, 0L)).isZero();
        assertThat(ch2.getOrDefault(CHALLENGED, 0L)).isEqualTo(1);
    }

    // ── Qhorus derivation: confidence distribution ───────────────────────

    @Test
    void confidenceDistribution_derivedFromSummary() {
        final String actor = "agent-" + UUID.randomUUID();
        final UUID subjectId = UUID.randomUUID();
        final Instant now = Instant.now();

        recorder.record(OutcomeRecord.of(actor, subjectId, "analyse", ENDORSED, 0.6)
                .withOccurredAt(now.minusSeconds(100)));
        recorder.record(OutcomeRecord.of(actor, subjectId, "analyse", SOUND, 0.9)
                .withOccurredAt(now.minusSeconds(80)));
        recorder.record(OutcomeRecord.of(actor, subjectId, "analyse", CHALLENGED, 0.3)
                .withOccurredAt(now.minusSeconds(60)));

        final AttestationSummary summary = repo.summariseAttestationsByActor(
                actor, now.minusSeconds(200), now, DEFAULT_TENANT_ID);

        assertThat(summary.totalAttestations()).isEqualTo(3);
        assertThat(summary.meanConfidence()).isCloseTo(0.6, org.assertj.core.data.Offset.offset(0.01));
        assertThat(summary.minConfidence()).isEqualTo(0.3);
        assertThat(summary.maxConfidence()).isEqualTo(0.9);
        assertThat(summary.verdictCounts()).containsEntry(ENDORSED, 1L);
        assertThat(summary.verdictCounts()).containsEntry(SOUND, 1L);
        assertThat(summary.verdictCounts()).containsEntry(CHALLENGED, 1L);
    }

    // ── Empty results ────────────────────────────────────────────────────

    @Test
    void countByActorAndVerdict_emptyWhenNoAttestations() {
        final Map<AttestationVerdict, Long> counts = repo.countByActorAndVerdict(
                "nonexistent-" + UUID.randomUUID(),
                Instant.now().minusSeconds(3600), Instant.now(), DEFAULT_TENANT_ID);
        assertThat(counts).isEmpty();
    }

    @Test
    void summariseAttestationsByActor_emptyWhenNoAttestations() {
        final AttestationSummary summary = repo.summariseAttestationsByActor(
                "nonexistent-" + UUID.randomUUID(),
                Instant.now().minusSeconds(3600), Instant.now(), DEFAULT_TENANT_ID);
        assertThat(summary.totalAttestations()).isZero();
        assertThat(summary.meanConfidence()).isEqualTo(0.0);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail (defaults return empty maps)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AggregateQueryIT`
Expected: FAIL — `fulfillmentRate` and `confidenceDistribution` tests fail because
defaults return empty maps/EMPTY summary

- [ ] **Step 3: Implement aggregate methods in InMemoryLedgerEntryRepository**

Add to `InMemoryLedgerEntryRepository`:

```java
@Override
public Map<AttestationVerdict, Long> countByActorAndVerdict(final String actorId,
        final Instant from, final Instant to, final String tenancyId) {
    final Set<UUID> entryIds = entries.values().stream()
            .filter(e -> actorId.equals(e.actorId)
                    && tenancyId.equals(e.tenancyId)
                    && !e.occurredAt.isBefore(from)
                    && !e.occurredAt.isAfter(to))
            .map(e -> e.id)
            .collect(Collectors.toSet());
    return attestations.values().stream()
            .filter(a -> entryIds.contains(a.ledgerEntryId))
            .collect(Collectors.groupingBy(a -> a.verdict, Collectors.counting()));
}

@Override
public Map<AttestationVerdict, Long> countBySubjectAndVerdict(final UUID subjectId,
        final Instant from, final Instant to, final String tenancyId) {
    final Set<UUID> entryIds = entries.values().stream()
            .filter(e -> subjectId.equals(e.subjectId)
                    && tenancyId.equals(e.tenancyId)
                    && !e.occurredAt.isBefore(from)
                    && !e.occurredAt.isAfter(to))
            .map(e -> e.id)
            .collect(Collectors.toSet());
    return attestations.values().stream()
            .filter(a -> entryIds.contains(a.ledgerEntryId))
            .collect(Collectors.groupingBy(a -> a.verdict, Collectors.counting()));
}

@Override
public AttestationSummary summariseAttestationsByActor(final String actorId,
        final Instant from, final Instant to, final String tenancyId) {
    final Set<UUID> entryIds = entries.values().stream()
            .filter(e -> actorId.equals(e.actorId)
                    && tenancyId.equals(e.tenancyId)
                    && !e.occurredAt.isBefore(from)
                    && !e.occurredAt.isAfter(to))
            .map(e -> e.id)
            .collect(Collectors.toSet());
    final List<LedgerAttestation> matched = attestations.values().stream()
            .filter(a -> entryIds.contains(a.ledgerEntryId))
            .toList();
    if (matched.isEmpty()) {
        return AttestationSummary.EMPTY;
    }
    final Map<AttestationVerdict, Long> verdictCounts = matched.stream()
            .collect(Collectors.groupingBy(a -> a.verdict, Collectors.counting()));
    final double mean = matched.stream().mapToDouble(a -> a.confidence).average().orElse(0.0);
    final double min = matched.stream().mapToDouble(a -> a.confidence).min().orElse(0.0);
    final double max = matched.stream().mapToDouble(a -> a.confidence).max().orElse(0.0);
    return new AttestationSummary(verdictCounts, matched.size(), mean, min, max);
}
```

Add imports: `java.util.stream.Collectors`, `java.util.Set`, `io.casehub.ledger.api.model.AttestationSummary`, `io.casehub.ledger.api.model.AttestationVerdict`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AggregateQueryIT`
Expected: PASS

- [ ] **Step 5: Run full runtime test suite for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS — no regressions

- [ ] **Step 6: Commit**

```bash
git add persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java
git add runtime/src/test/java/io/casehub/ledger/repository/AggregateQueryIT.java
git commit -m "feat(#201): aggregate queries — in-memory impl + qhorus derivation tests

Implements countByActorAndVerdict, countBySubjectAndVerdict,
summariseAttestationsByActor in InMemoryLedgerEntryRepository. Tests
demonstrate qhorus outcome derivation: fulfillment rate, per-channel
quality, and confidence distribution from the entry+attestation model.

Refs #201"
```

---

## Batch 3: JPA Implementation

### Task 4: @NamedQuery declarations + JPA implementations

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/jpa/JpaLedgerEntry.java` (add @NamedQuery)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java` (add @NamedQuery for aggregates)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository` SPI methods from Task 1
- Produces: JPA implementations backed by `@NamedQuery` for all 6 methods

- [ ] **Step 1: Add streaming + cursor @NamedQuery to JpaLedgerEntry**

Add to the `@NamedQuery` block on `JpaLedgerEntry` (before `@Entity`):

```java
@NamedQuery(
        name = "LedgerEntry.streamBySubjectId",
        query = "SELECT e FROM LedgerEntry e WHERE e.subjectId = :subjectId AND e.tenancyId = :tenancyId ORDER BY e.sequenceNumber ASC")
@NamedQuery(
        name = "LedgerEntry.streamByActorIdAndTimeRange",
        query = "SELECT e FROM LedgerEntry e WHERE e.actorId = :actorId AND e.occurredAt >= :from AND e.occurredAt <= :to AND e.tenancyId = :tenancyId ORDER BY e.occurredAt ASC")
@NamedQuery(
        name = "LedgerEntry.findBySubjectIdPaged",
        query = "SELECT e FROM LedgerEntry e WHERE e.subjectId = :subjectId AND e.sequenceNumber > :afterSequence AND e.tenancyId = :tenancyId ORDER BY e.sequenceNumber ASC")
```

- [ ] **Step 2: Add aggregate @NamedQuery to LedgerAttestation**

Add to the `@NamedQuery` block on `LedgerAttestation`:

```java
@NamedQuery(
        name = "LedgerAttestation.countByActorAndVerdict",
        query = "SELECT a.verdict, COUNT(a) FROM LedgerAttestation a JOIN LedgerEntry e ON a.ledgerEntryId = e.id WHERE e.actorId = :actorId AND e.occurredAt >= :from AND e.occurredAt <= :to AND e.tenancyId = :tenancyId GROUP BY a.verdict")
@NamedQuery(
        name = "LedgerAttestation.countBySubjectAndVerdict",
        query = "SELECT a.verdict, COUNT(a) FROM LedgerAttestation a JOIN LedgerEntry e ON a.ledgerEntryId = e.id WHERE e.subjectId = :subjectId AND e.occurredAt >= :from AND e.occurredAt <= :to AND e.tenancyId = :tenancyId GROUP BY a.verdict")
@NamedQuery(
        name = "LedgerAttestation.summariseByActor",
        query = "SELECT a.verdict, COUNT(a), AVG(a.confidence), MIN(a.confidence), MAX(a.confidence) FROM LedgerAttestation a JOIN LedgerEntry e ON a.ledgerEntryId = e.id WHERE e.actorId = :actorId AND e.occurredAt >= :from AND e.occurredAt <= :to AND e.tenancyId = :tenancyId GROUP BY a.verdict")
```

- [ ] **Step 3: Implement JPA streaming + cursor methods in JpaLedgerEntryRepository**

Add to `JpaLedgerEntryRepository`:

```java
@Override
public Stream<LedgerEntry> streamBySubjectId(final UUID subjectId, final String tenancyId) {
    return em.createNamedQuery("LedgerEntry.streamBySubjectId", LedgerEntry.class)
             .setParameter("subjectId", subjectId)
             .setParameter("tenancyId", tenancyId)
             .getResultStream();
}

@Override
public Stream<LedgerEntry> streamByActorId(final String actorId, final Instant from,
                                            final Instant to, final String tenancyId) {
    return em.createNamedQuery("LedgerEntry.streamByActorIdAndTimeRange", LedgerEntry.class)
             .setParameter("actorId", actorId)
             .setParameter("from", from)
             .setParameter("to", to)
             .setParameter("tenancyId", tenancyId)
             .getResultStream();
}

@Override
public List<LedgerEntry> findBySubjectIdPaged(final UUID subjectId, final int afterSequence,
                                               final int limit, final String tenancyId) {
    final List<LedgerEntry> results = em.createNamedQuery("LedgerEntry.findBySubjectIdPaged", LedgerEntry.class)
             .setParameter("subjectId", subjectId)
             .setParameter("afterSequence", afterSequence)
             .setParameter("tenancyId", tenancyId)
             .setMaxResults(limit)
             .getResultList();
    loadSupplements(results);
    return results;
}
```

Add import: `java.util.stream.Stream`.

- [ ] **Step 4: Implement JPA aggregate methods in JpaLedgerEntryRepository**

```java
@Override
public Map<AttestationVerdict, Long> countByActorAndVerdict(final String actorId,
        final Instant from, final Instant to, final String tenancyId) {
    final List<Object[]> rows = em.createNamedQuery("LedgerAttestation.countByActorAndVerdict", Object[].class)
            .setParameter("actorId", actorId)
            .setParameter("from", from)
            .setParameter("to", to)
            .setParameter("tenancyId", tenancyId)
            .getResultList();
    return rows.stream().collect(Collectors.toMap(
            r -> (AttestationVerdict) r[0],
            r -> (Long) r[1]));
}

@Override
public Map<AttestationVerdict, Long> countBySubjectAndVerdict(final UUID subjectId,
        final Instant from, final Instant to, final String tenancyId) {
    final List<Object[]> rows = em.createNamedQuery("LedgerAttestation.countBySubjectAndVerdict", Object[].class)
            .setParameter("subjectId", subjectId)
            .setParameter("from", from)
            .setParameter("to", to)
            .setParameter("tenancyId", tenancyId)
            .getResultList();
    return rows.stream().collect(Collectors.toMap(
            r -> (AttestationVerdict) r[0],
            r -> (Long) r[1]));
}

@Override
public AttestationSummary summariseAttestationsByActor(final String actorId,
        final Instant from, final Instant to, final String tenancyId) {
    final List<Object[]> rows = em.createNamedQuery("LedgerAttestation.summariseByActor", Object[].class)
            .setParameter("actorId", actorId)
            .setParameter("from", from)
            .setParameter("to", to)
            .setParameter("tenancyId", tenancyId)
            .getResultList();
    if (rows.isEmpty()) {
        return AttestationSummary.EMPTY;
    }
    final Map<AttestationVerdict, Long> verdictCounts = rows.stream()
            .collect(Collectors.toMap(r -> (AttestationVerdict) r[0], r -> (Long) r[1]));
    long total = 0;
    double weightedSum = 0.0;
    double min = Double.MAX_VALUE;
    double max = Double.MIN_VALUE;
    for (final Object[] row : rows) {
        final long count = (Long) row[1];
        final double avg = (Double) row[2];
        final double rowMin = (Double) row[3];
        final double rowMax = (Double) row[4];
        total += count;
        weightedSum += avg * count;
        min = Math.min(min, rowMin);
        max = Math.max(max, rowMax);
    }
    return new AttestationSummary(verdictCounts, total, weightedSum / total, min, max);
}
```

Add imports: `java.util.stream.Collectors`, `io.casehub.ledger.api.model.AttestationSummary`, `io.casehub.ledger.api.model.AttestationVerdict`.

- [ ] **Step 5: Build to verify @NamedQuery validation at boot**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=StreamingQueryIT,AggregateQueryIT`
Expected: PASS — Hibernate validates @NamedQuery JPQL at startup

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: PASS — all modules clean

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/jpa/JpaLedgerEntry.java
git add runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java
git commit -m "feat(#202): JPA implementation — @NamedQuery streaming, cursor, aggregates

Declares @NamedQuery on JpaLedgerEntry (streaming, cursor) and
LedgerAttestation (aggregate verdicts). JPA implementations use
getResultStream() for streaming, setMaxResults for cursor, and
JPQL GROUP BY for aggregates.

Closes #202, Closes #201"
```

---

## References

- [2026-08-23-qhorus-query-apis-design.md] — design spec this plan implements
- `api/spi/LedgerEntryRepository.java` — existing SPI (15 methods)
- `runtime/model/jpa/JpaLedgerEntry.java:50-91` — existing @NamedQuery declarations
- `runtime/model/LedgerAttestation.java` — attestation entity with existing @NamedQuery
- `runtime/repository/jpa/JpaLedgerEntryRepository.java:179-186` — JPA implementation pattern
- `persistence-memory/InMemoryLedgerEntryRepository.java` — in-memory backing store pattern
- `runtime/test/service/OutcomeRecorderIT.java` — test profile and setup pattern
- `docs/protocols/casehub/ledger-entry-named-query.md` — PP-20260618-51c673
- `docs/protocols/casehub/per-subject-table-tenancy.md` — PP-20260616-05dc6a
- GitHub #202 — streaming/cursor queries
- GitHub #201 — aggregate queries
