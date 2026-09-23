# Reporting SPI — EU AI Act Art.12 Compliance Report + Audit Trail Export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #211 — feat: Reporting SPI — ReportRenderer + EU AI Act Art.12 compliance report
**Issue group:** #211, #212

**Goal:** Add tenancy-level compliance reporting, audit trail export, and opt-in PDF/HTML rendering to casehub-ledger.

**Architecture:** Data services (tenancy queries, compliance reports, audit trail export) live in `runtime`. Presentation (Qute templates, PDF via platform `PdfGenerator`, content negotiation) lives in a new opt-in `reporting` module. Report models live in `ledger-core`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Qute, casehub-platform-api (`PdfGenerator`), H2 (tests)

## Global Constraints

- JPQL queries MUST use `@NamedQuery` on the entity class — never inline `em.createQuery()` (protocol PP-20260618-51c673)
- JPQL MUST use `FROM LedgerEntry` (the `@Entity(name)` value), not `FROM JpaLedgerEntry`
- All tenant-scoped methods MUST place `tenancyId` as the last parameter
- New SPI methods on `LedgerEntryRepository` MUST be `default` methods returning `List.of()`
- `ReportFormat` enum in `ledger-core` is UNCHANGED — no `PDF` value. The reporting module defines its own `OutputFormat`
- All tests use H2 in-memory — no Docker required

---

## Batch 1: Repository + Report Models

### Task 1: Add tenancy-level query methods to LedgerEntryRepository SPI

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/jpa/JpaLedgerEntry.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java`
- Test: `runtime/src/test/java/io/casehub/ledger/repository/FindByTimeRangeIT.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository` existing SPI, `JpaLedgerEntry` existing `@NamedQuery` declarations
- Produces: `LedgerEntryRepository.findByTimeRange(Instant from, Instant to, String tenancyId)` → `List<LedgerEntry>`, `LedgerEntryRepository.findDistinctSubjectIds(String tenancyId)` → `List<UUID>`

- [ ] **Step 1: Write failing test for `findByTimeRange`**

Create `runtime/src/test/java/io/casehub/ledger/repository/FindByTimeRangeIT.java`:

```java
package io.casehub.ledger.repository;

import static org.assertj.core.api.Assertions.assertThat;
import static io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class FindByTimeRangeIT {

    @Inject LedgerEntryRepository repo;

    @Test
    @Transactional
    void findByTimeRange_returnsEntriesInRange() {
        final String tenancyId = DEFAULT_TENANT_ID;
        final UUID subject1 = UUID.randomUUID();
        final UUID subject2 = UUID.randomUUID();
        final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
        final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

        repo.save(entry(subject1, "actor-a"), tenancyId);
        repo.save(entry(subject2, "actor-b"), tenancyId);

        final List<LedgerEntry> result = repo.findByTimeRange(from, to, tenancyId);

        assertThat(result).hasSizeGreaterThanOrEqualTo(2);
    }

    @Test
    @Transactional
    void findByTimeRange_excludesOutsideRange() {
        final String tenancyId = DEFAULT_TENANT_ID;
        final UUID subject = UUID.randomUUID();
        final Instant pastFrom = Instant.now().minus(2, ChronoUnit.DAYS);
        final Instant pastTo = Instant.now().minus(1, ChronoUnit.DAYS);

        repo.save(entry(subject, "actor-past"), tenancyId);

        final List<LedgerEntry> result = repo.findByTimeRange(pastFrom, pastTo, tenancyId);

        assertThat(result).noneMatch(e -> "actor-past".equals(e.actorId));
    }

    @Test
    @Transactional
    void findByTimeRange_isolatesByTenancy() {
        final UUID subject = UUID.randomUUID();
        final String tenantA = "tenant-a-" + UUID.randomUUID();
        final String tenantB = "tenant-b-" + UUID.randomUUID();
        final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
        final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

        repo.save(entry(subject, "actor-a"), tenantA);
        repo.save(entry(subject, "actor-b"), tenantB);

        final List<LedgerEntry> resultA = repo.findByTimeRange(from, to, tenantA);

        assertThat(resultA).allMatch(e -> "actor-a".equals(e.actorId));
    }

    @Test
    @Transactional
    void findDistinctSubjectIds_returnsUniqueSubjects() {
        final String tenancyId = DEFAULT_TENANT_ID;
        final UUID subject1 = UUID.randomUUID();
        final UUID subject2 = UUID.randomUUID();

        repo.save(entry(subject1, "actor-1"), tenancyId);
        repo.save(entry(subject1, "actor-2"), tenancyId);
        repo.save(entry(subject2, "actor-3"), tenancyId);

        final List<UUID> subjects = repo.findDistinctSubjectIds(tenancyId);

        assertThat(subjects).contains(subject1, subject2);
    }

    private static TestEntry entry(final UUID subjectId, final String actorId) {
        final TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.actorRole = "Reporter";
        e.occurredAt = Instant.now();
        return e;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=FindByTimeRangeIT -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — `findByTimeRange` method does not exist on `LedgerEntryRepository`

- [ ] **Step 3: Add default methods to `LedgerEntryRepository`**

Use `ide_insert_member` to add to `api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryRepository.java`:

```java
default List<LedgerEntry> findByTimeRange(Instant from, Instant to, String tenancyId) {
    return List.of();
}

default List<UUID> findDistinctSubjectIds(String tenancyId) {
    return List.of();
}
```

- [ ] **Step 4: Add `@NamedQuery` declarations to `JpaLedgerEntry`**

Use `ide_edit_member` on `runtime/src/main/java/io/casehub/ledger/runtime/model/jpa/JpaLedgerEntry.java` to add to the existing `@NamedQuery` block:

```java
@NamedQuery(name = "LedgerEntry.findByTimeRange",
    query = "SELECT e FROM LedgerEntry e WHERE e.tenancyId = :tenancyId " +
            "AND e.occurredAt >= :from AND e.occurredAt <= :to ORDER BY e.occurredAt ASC")
@NamedQuery(name = "LedgerEntry.findDistinctSubjectIds",
    query = "SELECT DISTINCT e.subjectId FROM LedgerEntry e WHERE e.tenancyId = :tenancyId")
```

- [ ] **Step 5: Implement in `JpaLedgerEntryRepository`**

Use `ide_insert_member` on `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`:

```java
@Override
public List<LedgerEntry> findByTimeRange(final Instant from, final Instant to, final String tenancyId) {
    return em.createNamedQuery("LedgerEntry.findByTimeRange", LedgerEntry.class)
            .setParameter("tenancyId", tenancyId)
            .setParameter("from", from)
            .setParameter("to", to)
            .getResultList();
}

@Override
public List<UUID> findDistinctSubjectIds(final String tenancyId) {
    return em.createNamedQuery("LedgerEntry.findDistinctSubjectIds", UUID.class)
            .setParameter("tenancyId", tenancyId)
            .getResultList();
}
```

- [ ] **Step 6: Implement in `InMemoryLedgerEntryRepository`**

Use `ide_insert_member` on `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java`:

```java
@Override
public List<LedgerEntry> findByTimeRange(final Instant from, final Instant to, final String tenancyId) {
    return allEntries().stream()
            .filter(e -> tenancyId.equals(e.tenancyId))
            .filter(e -> e.occurredAt != null && !e.occurredAt.isBefore(from) && !e.occurredAt.isAfter(to))
            .sorted(java.util.Comparator.comparing(e -> e.occurredAt))
            .toList();
}

@Override
public List<UUID> findDistinctSubjectIds(final String tenancyId) {
    return allEntries().stream()
            .filter(e -> tenancyId.equals(e.tenancyId))
            .map(e -> e.subjectId)
            .distinct()
            .toList();
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=FindByTimeRangeIT`
Expected: All 4 tests PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryRepository.java \
  runtime/src/main/java/io/casehub/ledger/runtime/model/jpa/JpaLedgerEntry.java \
  runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java \
  runtime/src/test/java/io/casehub/ledger/repository/FindByTimeRangeIT.java
git commit -m "feat(#211): add findByTimeRange and findDistinctSubjectIds to LedgerEntryRepository

Refs #211"
```

### Task 2: Evolve report models in ledger-core

**Files:**
- Modify: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/ComplianceReport.java`
- Modify: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/DecisionRecord.java`
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/ComplianceSummary.java`
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/AuditTrailExport.java`
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/SubjectAuditTrail.java`
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/AuditEntry.java`
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/VerificationSummary.java`
- Test: `ledger-core/src/test/java/io/casehub/ledger/core/compliance/ComplianceSummaryTest.java`

**Interfaces:**
- Consumes: existing `ComplianceReport`, `DecisionRecord`, `ReportFormat`
- Produces: `ComplianceSummary(int totalDecisions, int aiAssistedDecisions, int humanOverrideCount, Map<String,Integer> decisionsByType)`, `AuditTrailExport(...)`, `SubjectAuditTrail(...)`, `AuditEntry(...)`, `VerificationSummary(...)`

- [ ] **Step 1: Write failing test for `ComplianceSummary`**

Create `ledger-core/src/test/java/io/casehub/ledger/core/compliance/ComplianceSummaryTest.java`:

```java
package io.casehub.ledger.core.compliance;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

class ComplianceSummaryTest {

    @Test
    void fromDecisions_countsCorrectly() {
        final List<DecisionRecord> decisions = List.of(
                decision("PlainLedgerEntry", "alg-v1", true),
                decision("PlainLedgerEntry", null, false),
                decision("WorkItemLedgerEntry", "alg-v2", true));

        final ComplianceSummary summary = ComplianceSummary.fromDecisions(decisions);

        assertThat(summary.totalDecisions()).isEqualTo(3);
        assertThat(summary.aiAssistedDecisions()).isEqualTo(2);
        assertThat(summary.humanOverrideCount()).isEqualTo(2);
        assertThat(summary.decisionsByType()).containsEntry("PlainLedgerEntry", 2);
        assertThat(summary.decisionsByType()).containsEntry("WorkItemLedgerEntry", 1);
    }

    @Test
    void fromDecisions_empty_allZero() {
        final ComplianceSummary summary = ComplianceSummary.fromDecisions(List.of());

        assertThat(summary.totalDecisions()).isEqualTo(0);
        assertThat(summary.aiAssistedDecisions()).isEqualTo(0);
        assertThat(summary.humanOverrideCount()).isEqualTo(0);
        assertThat(summary.decisionsByType()).isEmpty();
    }

    private static DecisionRecord decision(final String entryType,
            final String algorithmRef, final boolean humanOverride) {
        return new DecisionRecord(UUID.randomUUID(), entryType, Instant.now(),
                "actor-1", algorithmRef, algorithmRef != null ? 0.9 : null,
                null, null, humanOverride, null, null);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=ComplianceSummaryTest`
Expected: Compilation error — `ComplianceSummary` does not exist, `DecisionRecord` missing fields

- [ ] **Step 3: Create `ComplianceSummary` record**

Create `ledger-core/src/main/java/io/casehub/ledger/core/compliance/ComplianceSummary.java`:

```java
package io.casehub.ledger.core.compliance;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public record ComplianceSummary(
        int totalDecisions,
        int aiAssistedDecisions,
        int humanOverrideCount,
        Map<String, Integer> decisionsByType) {

    public static ComplianceSummary fromDecisions(final List<DecisionRecord> decisions) {
        int aiAssisted = 0;
        int humanOverride = 0;
        final Map<String, Integer> byType = new LinkedHashMap<>();

        for (final DecisionRecord d : decisions) {
            if (d.algorithmRef() != null) {
                aiAssisted++;
            }
            if (Boolean.TRUE.equals(d.humanOverrideAvailable())) {
                humanOverride++;
            }
            if (d.entryType() != null) {
                byType.merge(d.entryType(), 1, Integer::sum);
            }
        }
        return new ComplianceSummary(decisions.size(), aiAssisted, humanOverride, Map.copyOf(byType));
    }
}
```

- [ ] **Step 4: Update `DecisionRecord` — add `entryType`, `actorId`, `planRef`**

Use `ide_replace_member` on `ledger-core/src/main/java/io/casehub/ledger/core/compliance/DecisionRecord.java` to replace the record definition:

```java
package io.casehub.ledger.core.compliance;

import java.time.Instant;
import java.util.UUID;

public record DecisionRecord(
        UUID entryId,
        String entryType,
        Instant occurredAt,
        String actorId,
        String algorithmRef,
        Double confidenceScore,
        String planRef,
        String contestationUri,
        Boolean humanOverrideAvailable,
        String sourceEntityType,
        String sourceEntityId) {
}
```

- [ ] **Step 5: Update `ComplianceReport` — add `tenancyId`, `summary`**

Use `ide_replace_member` on `ledger-core/src/main/java/io/casehub/ledger/core/compliance/ComplianceReport.java` to update the record. Add `tenancyId` and `summary` parameters. Update `format()` to include `tenancyId` and summary stats in JSON/CSV output. Add `UnsupportedOperationException` for any future format value that can't produce `String`.

- [ ] **Step 6: Update `LedgerComplianceReportService` call sites**

The `toDecisionRecord()` in `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerComplianceReportService.java` constructs `DecisionRecord` — update it to pass the new fields (`entryType`, `actorId`, `planRef`). Update `reportForActor()` and `reportForSubject()` to pass `tenancyId` and a `ComplianceSummary` built via `ComplianceSummary.fromDecisions(decisions)`.

- [ ] **Step 7: Create audit trail model records**

Create four new files in `ledger-core/src/main/java/io/casehub/ledger/core/compliance/`:

`AuditTrailExport.java`:
```java
package io.casehub.ledger.core.compliance;

import java.time.Instant;
import java.util.List;

public record AuditTrailExport(
        String tenancyId, Instant from, Instant to, Instant generatedAt,
        List<SubjectAuditTrail> subjects, VerificationSummary verification) {
}
```

`SubjectAuditTrail.java`:
```java
package io.casehub.ledger.core.compliance;

import java.util.List;
import java.util.UUID;

public record SubjectAuditTrail(
        UUID subjectId, List<AuditEntry> entries, String merkleRoot,
        boolean chainValid, String provJsonLd) {
}
```

`AuditEntry.java`:
```java
package io.casehub.ledger.core.compliance;

import java.time.Instant;
import java.util.UUID;

public record AuditEntry(
        UUID entryId, String entryType, Instant occurredAt,
        String actorId, String digest, int sequenceNumber) {
}
```

`VerificationSummary.java`:
```java
package io.casehub.ledger.core.compliance;

import java.util.List;

public record VerificationSummary(
        int totalSubjects, int validChains, int invalidChains, int totalEntries) {

    public static VerificationSummary fromTrails(final List<SubjectAuditTrail> trails) {
        int valid = 0, invalid = 0, entries = 0;
        for (final SubjectAuditTrail t : trails) {
            if (t.chainValid()) valid++; else invalid++;
            entries += t.entries().size();
        }
        return new VerificationSummary(trails.size(), valid, invalid, entries);
    }
}
```

- [ ] **Step 8: Run all tests to verify nothing is broken**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,ledger-core -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=ComplianceSummaryTest && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerComplianceReportServiceIT`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git add ledger-core/src/main/java/io/casehub/ledger/core/compliance/ \
  runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerComplianceReportService.java \
  ledger-core/src/test/java/io/casehub/ledger/core/compliance/ComplianceSummaryTest.java
git commit -m "feat(#211): evolve report models — ComplianceSummary, AuditTrailExport, enriched DecisionRecord

Refs #211"
```

---

## Batch 2: Data Services

### Task 3: Add `reportForTenancy()` to `LedgerComplianceReportService`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerComplianceReportService.java`
- Test: `runtime/src/test/java/io/casehub/ledger/service/LedgerComplianceReportServiceIT.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository.findByTimeRange(from, to, tenancyId)`, `ComplianceSummary.fromDecisions(decisions)`, `LedgerVerificationService.treeRoot(subjectId, tenancyId)`
- Produces: `LedgerComplianceReportService.reportForTenancy(String tenancyId, Instant from, Instant to)` → `ComplianceReport`

- [ ] **Step 1: Write failing test for `reportForTenancy()`**

Add to existing `runtime/src/test/java/io/casehub/ledger/service/LedgerComplianceReportServiceIT.java`:

```java
@Test
@Transactional
void tenancyReport_aggregatesAcrossSubjects() {
    final String tenancyId = DEFAULT_TENANT_ID;
    final UUID subject1 = UUID.randomUUID();
    final UUID subject2 = UUID.randomUUID();
    final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
    final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

    entryWithComplianceForSubject(subject1, "actor-t1", "alg-1", 0.9);
    entryWithComplianceForSubject(subject2, "actor-t2", "alg-2", 0.8);
    bareEntry("actor-t3"); // no compliance supplement — excluded

    final ComplianceReport report = reportService.reportForTenancy(tenancyId, from, to);

    assertThat(report.tenancyId()).isEqualTo(tenancyId);
    assertThat(report.actorId()).isNull();
    assertThat(report.subjectId()).isNull();
    assertThat(report.totalDecisions()).isEqualTo(2);
    assertThat(report.summary()).isNotNull();
    assertThat(report.summary().aiAssistedDecisions()).isEqualTo(2);
}

@Test
@Transactional
void tenancyReport_emptyTenancy_zeroCounts() {
    final String tenancyId = "empty-tenant-" + UUID.randomUUID();
    final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
    final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

    final ComplianceReport report = reportService.reportForTenancy(tenancyId, from, to);

    assertThat(report.totalDecisions()).isEqualTo(0);
    assertThat(report.decisions()).isEmpty();
    assertThat(report.summary().totalDecisions()).isEqualTo(0);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerComplianceReportServiceIT#tenancyReport_aggregatesAcrossSubjects`
Expected: Compilation error — `reportForTenancy` does not exist

- [ ] **Step 3: Implement `reportForTenancy()` and `buildSummary()`**

Use `ide_insert_member` on `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerComplianceReportService.java`:

```java
@Transactional
public ComplianceReport reportForTenancy(
        final String tenancyId, final Instant from, final Instant to) {
    final List<LedgerEntry> entries = repo.findByTimeRange(from, to, tenancyId);
    final List<DecisionRecord> decisions = entries.stream()
            .filter(e -> e.compliance().isPresent())
            .map(this::toDecisionRecord)
            .toList();
    final ComplianceSummary summary = ComplianceSummary.fromDecisions(decisions);
    final String merkleRoot = buildTenancyMerkleRoot(entries, tenancyId);
    return new ComplianceReport(null, null, tenancyId, from, to,
            decisions.size(), decisions, summary, merkleRoot);
}

private String buildTenancyMerkleRoot(final List<LedgerEntry> entries, final String tenancyId) {
    return buildActorMerkleRoot(entries, tenancyId);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerComplianceReportServiceIT`
Expected: All tests PASS (existing + new)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerComplianceReportService.java \
  runtime/src/test/java/io/casehub/ledger/service/LedgerComplianceReportServiceIT.java
git commit -m "feat(#211): add reportForTenancy() to LedgerComplianceReportService

Refs #211"
```

### Task 4: Create `AuditTrailExportService`

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AuditTrailExportService.java`
- Test: `runtime/src/test/java/io/casehub/ledger/service/AuditTrailExportServiceIT.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository.findDistinctSubjectIds(tenancyId)`, `LedgerEntryRepository.findBySubjectIdAndTimeRange(subjectId, from, to, tenancyId)`, `LedgerVerificationService.verify(subjectId, tenancyId)`, `LedgerVerificationService.treeRoot(subjectId, tenancyId)`, `LedgerProvExportService.exportSubject(subjectId, tenancyId)`, `VerificationSummary.fromTrails(trails)`
- Produces: `AuditTrailExportService.generateForTenancy(String tenancyId, Instant from, Instant to)` → `AuditTrailExport`

- [ ] **Step 1: Write failing test**

Create `runtime/src/test/java/io/casehub/ledger/service/AuditTrailExportServiceIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.core.compliance.AuditTrailExport;
import io.casehub.ledger.runtime.service.AuditTrailExportService;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class AuditTrailExportServiceIT {

    @Inject AuditTrailExportService exportService;
    @Inject LedgerEntryRepository repo;

    @Test
    @Transactional
    void generateForTenancy_twoSubjects_bothIncluded() {
        final UUID subject1 = UUID.randomUUID();
        final UUID subject2 = UUID.randomUUID();
        final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
        final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

        repo.save(entry(subject1, "actor-1"), DEFAULT_TENANT_ID);
        repo.save(entry(subject2, "actor-2"), DEFAULT_TENANT_ID);

        final AuditTrailExport export = exportService.generateForTenancy(DEFAULT_TENANT_ID, from, to);

        assertThat(export.tenancyId()).isEqualTo(DEFAULT_TENANT_ID);
        assertThat(export.subjects()).hasSizeGreaterThanOrEqualTo(2);
        assertThat(export.verification().totalSubjects()).isGreaterThanOrEqualTo(2);
    }

    @Test
    @Transactional
    void generateForTenancy_emptyTenancy_emptyResult() {
        final String tenancyId = "empty-audit-" + UUID.randomUUID();
        final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
        final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

        final AuditTrailExport export = exportService.generateForTenancy(tenancyId, from, to);

        assertThat(export.subjects()).isEmpty();
        assertThat(export.verification().totalSubjects()).isEqualTo(0);
    }

    @Test
    @Transactional
    void generateForTenancy_verificationPerSubject() {
        final UUID subject = UUID.randomUUID();
        final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
        final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

        repo.save(entry(subject, "actor-v"), DEFAULT_TENANT_ID);

        final AuditTrailExport export = exportService.generateForTenancy(DEFAULT_TENANT_ID, from, to);

        assertThat(export.subjects()).anyMatch(s ->
                s.subjectId().equals(subject) && s.merkleRoot() != null);
    }

    private static TestEntry entry(final UUID subjectId, final String actorId) {
        final TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.actorRole = "AuditExporter";
        e.occurredAt = Instant.now();
        return e;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AuditTrailExportServiceIT -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — `AuditTrailExportService` does not exist

- [ ] **Step 3: Create `AuditTrailExportService`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/AuditTrailExportService.java` with the implementation from the spec (§3b). Use `LedgerEntryRepository.findDistinctSubjectIds()`, `findBySubjectIdAndTimeRange()`, `LedgerVerificationService.verify()` and `treeRoot()`, `LedgerProvExportService.exportSubject()`, and `VerificationSummary.fromTrails()`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AuditTrailExportServiceIT`
Expected: All 3 tests PASS

- [ ] **Step 5: Run full runtime test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/AuditTrailExportService.java \
  runtime/src/test/java/io/casehub/ledger/service/AuditTrailExportServiceIT.java
git commit -m "feat(#211): add AuditTrailExportService — tenancy-level audit trail with verification

Refs #211"
```

---

## Batch 3: Reporting Module

### Task 5: Create `casehub-ledger-reporting` module with Qute rendering

**Files:**
- Modify: `pom.xml` (root — add `<module>reporting</module>`)
- Create: `reporting/pom.xml`
- Create: `reporting/src/main/java/io/casehub/ledger/reporting/OutputFormat.java`
- Create: `reporting/src/main/java/io/casehub/ledger/reporting/ReportMediaType.java`
- Create: `reporting/src/main/java/io/casehub/ledger/reporting/LedgerReportingService.java`
- Create: `reporting/src/main/resources/templates/reports/compliance-art12.html`
- Create: `reporting/src/main/resources/templates/reports/audit-trail.html`
- Test: `reporting/src/test/java/io/casehub/ledger/reporting/ReportMediaTypeTest.java`
- Test: `reporting/src/test/java/io/casehub/ledger/reporting/LedgerReportingServiceIT.java`

**Interfaces:**
- Consumes: `ComplianceReport`, `AuditTrailExport`, `ReportFormat`, `PdfGenerator.generateFromHtml(html, PdfOptions)`, Qute `Template`
- Produces: `OutputFormat` enum, `ReportMediaType.fromAcceptHeader(String)` → `OutputFormat`, `LedgerReportingService.renderComplianceReport(ComplianceReport, OutputFormat)` → `byte[]`, `LedgerReportingService.renderAuditTrail(AuditTrailExport, OutputFormat)` → `byte[]`

- [ ] **Step 1: Add `reporting` module to root pom.xml**

Use `ide_edit_member` on `pom.xml` to add `<module>reporting</module>` after `<module>graphql</module>`.

- [ ] **Step 2: Create `reporting/pom.xml`**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ledger-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-ledger-reporting</artifactId>
  <name>CaseHub Ledger - Reporting</name>
  <description>Opt-in reporting: Qute HTML templates, PDF rendering via platform PdfGenerator, content negotiation</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-qute</artifactId>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jackson</artifactId>
      <scope>provided</scope>
    </dependency>

    <dependency>
      <groupId>jakarta.enterprise</groupId>
      <artifactId>jakarta.enterprise.cdi-api</artifactId>
      <scope>provided</scope>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger-persistence-memory</artifactId>
      <version>${project.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger-deployment</artifactId>
      <version>${project.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <executions>
          <execution>
            <id>make-index</id>
            <goals>
              <goal>jandex</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 3: Create `OutputFormat` enum**

Create `reporting/src/main/java/io/casehub/ledger/reporting/OutputFormat.java`:

```java
package io.casehub.ledger.reporting;

public enum OutputFormat {
    JSON, JSON_LD, CSV, HTML, PDF
}
```

- [ ] **Step 4: Write failing test for `ReportMediaType`**

Create `reporting/src/test/java/io/casehub/ledger/reporting/ReportMediaTypeTest.java`:

```java
package io.casehub.ledger.reporting;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class ReportMediaTypeTest {

    @Test void pdf() { assertThat(ReportMediaType.fromAcceptHeader("application/pdf")).isEqualTo(OutputFormat.PDF); }
    @Test void html() { assertThat(ReportMediaType.fromAcceptHeader("text/html")).isEqualTo(OutputFormat.HTML); }
    @Test void csv() { assertThat(ReportMediaType.fromAcceptHeader("text/csv")).isEqualTo(OutputFormat.CSV); }
    @Test void jsonLd() { assertThat(ReportMediaType.fromAcceptHeader("application/ld+json")).isEqualTo(OutputFormat.JSON_LD); }
    @Test void json() { assertThat(ReportMediaType.fromAcceptHeader("application/json")).isEqualTo(OutputFormat.JSON); }
    @Test void nullHeader() { assertThat(ReportMediaType.fromAcceptHeader(null)).isEqualTo(OutputFormat.JSON); }
}
```

- [ ] **Step 5: Create `ReportMediaType`**

Create `reporting/src/main/java/io/casehub/ledger/reporting/ReportMediaType.java`:

```java
package io.casehub.ledger.reporting;

public final class ReportMediaType {

    private ReportMediaType() {}

    public static OutputFormat fromAcceptHeader(final String accept) {
        if (accept == null) return OutputFormat.JSON;
        if (accept.contains("application/pdf")) return OutputFormat.PDF;
        if (accept.contains("text/html")) return OutputFormat.HTML;
        if (accept.contains("text/csv")) return OutputFormat.CSV;
        if (accept.contains("application/ld+json")) return OutputFormat.JSON_LD;
        return OutputFormat.JSON;
    }
}
```

- [ ] **Step 6: Create Qute templates**

Create `reporting/src/main/resources/templates/reports/compliance-art12.html` — structured HTML with report header, summary stats table, decisions table, and Merkle root anchor. Clean CSS, no JavaScript, suitable for `openhtmltopdf`.

Create `reporting/src/main/resources/templates/reports/audit-trail.html` — per-subject sections with entry tables, verification status per subject, and verification summary footer.

- [ ] **Step 7: Create `LedgerReportingService`**

Create `reporting/src/main/java/io/casehub/ledger/reporting/LedgerReportingService.java` with the implementation from the spec (§4b). Maps `OutputFormat` values to appropriate renderers — text formats delegate to `ComplianceReport.format()`, HTML uses Qute templates, PDF chains Qute HTML → `PdfGenerator.generateFromHtml()`.

- [ ] **Step 8: Write integration test for rendering**

Create `reporting/src/test/java/io/casehub/ledger/reporting/LedgerReportingServiceIT.java`:

```java
package io.casehub.ledger.reporting;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.core.compliance.ComplianceReport;
import io.casehub.ledger.core.compliance.ComplianceSummary;
import io.casehub.ledger.core.compliance.DecisionRecord;
import io.quarkus.test.junit.QuarkusTest;

import java.util.UUID;

@QuarkusTest
class LedgerReportingServiceIT {

    @Inject LedgerReportingService service;

    @Test
    void renderComplianceReport_json_validOutput() {
        final byte[] result = service.renderComplianceReport(sampleReport(), OutputFormat.JSON);
        assertThat(new String(result)).contains("\"totalDecisions\"");
    }

    @Test
    void renderComplianceReport_html_containsHtmlTags() {
        final byte[] result = service.renderComplianceReport(sampleReport(), OutputFormat.HTML);
        assertThat(new String(result)).contains("<html");
    }

    @Test
    void renderComplianceReport_pdf_throwsWithNoOpGenerator() {
        assertThatThrownBy(() -> service.renderComplianceReport(sampleReport(), OutputFormat.PDF))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("PDF generation unavailable");
    }

    private ComplianceReport sampleReport() {
        final DecisionRecord d = new DecisionRecord(
                UUID.randomUUID(), "PlainLedgerEntry", Instant.now(),
                "actor-1", "alg-v1", 0.9, null, null, true, null, null);
        final ComplianceSummary summary = ComplianceSummary.fromDecisions(List.of(d));
        return new ComplianceReport("actor-1", null, "default", Instant.now().minusSeconds(3600),
                Instant.now(), 1, List.of(d), summary, null);
    }
}
```

- [ ] **Step 9: Add test `application.properties`**

Create `reporting/src/test/resources/application.properties`:

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:reporting-test;MODE=PostgreSQL;DATABASE_TO_LOWER=TRUE
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.flyway.migrate-at-start=false
quarkus.arc.selected-alternatives=io.casehub.ledger.memory.InMemoryLedgerEntryRepository,io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,io.casehub.ledger.memory.InMemoryTrustScoreSnapshotRepository
```

- [ ] **Step 10: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,ledger-core,runtime,deployment,persistence-memory -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl reporting`
Expected: All tests PASS

- [ ] **Step 11: Run full build to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 12: Update `CLAUDE.md` — add reporting module to project structure**

Add reporting module coordinates and structure to the CLAUDE.md `## Maven Coordinates` and `## Project Structure` sections.

- [ ] **Step 13: Commit**

```bash
git add reporting/ pom.xml CLAUDE.md
git commit -m "feat(#211): add casehub-ledger-reporting module — Qute templates, PDF via platform, content negotiation

Refs #211"
```

---

## References

- `specs/issue-211-reporting-spi-compliance-report/2026-09-23-reporting-spi-compliance-report-design.md` — design spec
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerComplianceReportService.java` — existing compliance service
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java` — Merkle verification
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvExportService.java` — PROV-O export
- `api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryRepository.java` — tenant-scoped SPI
- `ledger-core/src/main/java/io/casehub/ledger/core/compliance/ComplianceReport.java` — existing model
- `platform-api/src/main/java/io/casehub/platform/api/pdf/PdfGenerator.java` — platform PDF SPI
- `rest/pom.xml` — module pattern reference
- `runtime/src/test/java/io/casehub/ledger/service/LedgerComplianceReportServiceIT.java` — test pattern
- `docs/protocols/casehub/ledger-entry-named-query.md` — PP-20260618-51c673
- `docs/protocols/casehub/per-subject-table-tenancy.md` — PP-20260616-05dc6a
- GitHub #211, #212
