# Design: LedgerHealthJob HAVING Clause Fix (#153)

## Problem

`LedgerHealthJob.checkSequenceGaps()` uses an inline JPQL query with aggregate arithmetic
in a HAVING clause:

```jpql
SELECT e.subjectId, e.tenancyId, COUNT(e), MIN(e.sequenceNumber), MAX(e.sequenceNumber)
FROM LedgerEntry e
GROUP BY e.subjectId, e.tenancyId
HAVING COUNT(e) != MAX(e.sequenceNumber) - MIN(e.sequenceNumber) + 1
```

Hibernate 6 rejects this with:

```
SemanticException: Operand of - is of type 'java.lang.Object' which is not a numeric type
```

The `-` operands (`MAX(e.sequenceNumber)`, `MIN(e.sequenceNumber)`) are classified as `Object`
by Hibernate 6's semantic checker when used in HAVING arithmetic. This is a Hibernate 6
regression: the type resolution for aggregate results in arithmetic expressions fails regardless
of the underlying field type (`sequenceNumber` is `int`).

**Dialect scope:** The `SemanticException` is thrown at `em.createQuery()` parse time —
before any SQL is generated or any database connection is made. All three supported dialects
(H2 standard, H2+MODE=PostgreSQL, PostgreSQL) produce the same exception. All 14 test
executions across `LedgerHealthJobIT` (7 tests) and `LedgerHealthJobPgIT` (7 inherited tests)
currently fail.

**Root cause — two violations compounding:**

1. **HAVING arithmetic:** Hibernate 6 rejects arithmetic on aggregate results in HAVING.
   Fix: remove HAVING, filter in Java.

2. **Inline `createQuery()` instead of `@NamedQuery`:** The convention
   (`docs/specs/2026-04-20-remove-panache-from-entities.md`, CLAUDE.md) is that all JPQL
   lives in `@NamedQuery` on entity classes — Hibernate validates them at startup, not at
   first query execution. Had this query been a `@NamedQuery`, the SemanticException would
   have been thrown at boot and caught in development, not after a 1h production interval.

Both violations must be fixed together.

---

## Fix

### 1. `SubjectSequenceStats` record — new type in `service/model/`

```java
public record SubjectSequenceStats(
    UUID subjectId,
    String tenancyId,
    long count,   // COUNT() returns Long — standard unboxing
    int min,      // MIN(int field) returns Integer — exact type match, no widening
    int max       // MAX(int field) returns Integer — exact type match, no widening
) {}
```

`sequenceNumber` is declared as `int` on `LedgerEntry`. `MIN`/`MAX` on an `int` field
produce `Integer` in Hibernate's JPQL type system. Declaring `min`/`max` as `int` gives
an exact constructor match — no implicit widening through reflection's invocation path.
`long` would work via `Integer → long` unboxing+widening, but that relies on Hibernate's
constructor-invocation implementation details (standard reflection vs. method handles),
which is fragile across Hibernate minor versions.

Replaces untyped `Object[]`. Consistent with the typed-result pattern on
`CrossTenantLedgerEntryRepository` (which already returns `List<LedgerEntry>`,
`Map<UUID, List<LedgerAttestation>>`, etc.).

### 2. `@NamedQuery` on `LedgerEntry` (runtime entity)

```java
@NamedQuery(
    name = "LedgerEntry.findSequenceStats",
    query = """
        SELECT NEW io.casehub.ledger.runtime.service.model.SubjectSequenceStats(
            e.subjectId, e.tenancyId, COUNT(e),
            MIN(e.sequenceNumber), MAX(e.sequenceNumber)
        )
        FROM LedgerEntry e
        GROUP BY e.subjectId, e.tenancyId
        """
)
```

No arithmetic in HAVING — the aggregate values are passed directly as constructor arguments.
Hibernate validates this at startup. The `NEW` constructor expression returns typed
`SubjectSequenceStats` directly — no `Object[]` mapping needed in the JPA implementation.

`LedgerEntry` is abstract; `@NamedQuery` on an abstract entity is valid in JPA — the query
runs polymorphically across all concrete subclasses, which is the correct behaviour here.

### 3. New SPI method on `CrossTenantLedgerEntryRepository`

```java
/**
 * Return sequence aggregates for all (subject, tenant) pairs, across all tenants.
 * Used by the health job for gap detection.
 */
List<SubjectSequenceStats> findSequenceStats();
```

`LedgerHealthJob` is a system-level bean operating across all tenants. The query belongs in
`CrossTenantLedgerEntryRepository` alongside the other cross-tenant reads (`listAll`,
`findAllEvents`, retention queries). `LedgerHealthJob` injecting `EntityManager` directly
was the architectural gap that allowed the violation.

### 4. JPA implementation — `JpaCrossTenantLedgerEntryRepository`

```java
@Override
public List<SubjectSequenceStats> findSequenceStats() {
    return em.createNamedQuery("LedgerEntry.findSequenceStats", SubjectSequenceStats.class)
             .getResultList();
}
```

Uses the named query — type-safe, no inline JPQL.

### 5. In-memory implementation — `InMemoryCrossTenantLedgerEntryRepository`

```java
@Override
public List<SubjectSequenceStats> findSequenceStats() {
    record Key(UUID subjectId, String tenancyId) {}
    return blocking.allEntries().stream()
            .collect(Collectors.groupingBy(e -> new Key(e.subjectId, e.tenancyId)))
            .entrySet().stream()
            .map(entry -> {
                final Key k = entry.getKey();
                final List<LedgerEntry> entries = entry.getValue();
                final long count = entries.size();
                final int min = entries.stream().mapToInt(e -> e.sequenceNumber).min().getAsInt();
                final int max = entries.stream().mapToInt(e -> e.sequenceNumber).max().getAsInt();
                return new SubjectSequenceStats(k.subjectId(), k.tenancyId(), count, min, max);
            })
            .toList();
}
```

Mirrors the JPA semantics — no HAVING, no arithmetic issues.

### 6. `LedgerHealthJob` — remove `EntityManager`, inject repository

Before:
```java
@Inject
@LedgerPersistenceUnit
EntityManager em;
```

After: removed. Replace the inline `createQuery()` in `checkSequenceGaps()` with:

```java
@Inject
@CrossTenant
CrossTenantLedgerEntryRepository crossTenantRepo;

private void checkSequenceGaps() {
    for (final SubjectSequenceStats stats : crossTenantRepo.findSequenceStats()) {
        final long expected = (long) stats.max() - stats.min() + 1;
        if (stats.count() == expected) continue;

        LOG.warnf("Sequence gap for subject %s / tenant %s: expected %d entries (seq %d–%d), found %d",
                stats.subjectId(), stats.tenancyId(), expected, stats.min(), stats.max(), stats.count());
        anomalyEvent.fire(new LedgerSequenceGapDetected(
                stats.subjectId(), stats.tenancyId(), expected, stats.count()));
    }
}
```

The explicit `(long)` cast on `stats.max()` prevents `int` overflow when sequence
numbers are large (`max - min + 1` computed entirely in `int` would overflow at
`Integer.MAX_VALUE`). `min` and `max` are `int` to match the exact JPQL type;
the arithmetic needs `long` range, hence the explicit widening cast.

---

## Scope

| File | Change |
|------|--------|
| `runtime/model/LedgerEntry.java` | Add `@NamedQuery` for `findSequenceStats` |
| `runtime/service/model/SubjectSequenceStats.java` | New record |
| `runtime/repository/CrossTenantLedgerEntryRepository.java` | Add `findSequenceStats()` |
| `runtime/repository/jpa/JpaCrossTenantLedgerEntryRepository.java` | Implement via named query |
| `runtime/service/LedgerHealthJob.java` | Remove EntityManager; inject CrossTenantRepo; refactor `checkSequenceGaps()` |
| `persistence-memory/.../InMemoryCrossTenantLedgerEntryRepository.java` | Implement via Java grouping |

**Tests:** 7 tests in `LedgerHealthJobIT` (14 total including `LedgerHealthJobPgIT` inheritance).
No test changes required — the behaviour is identical. All 14 currently fail; all 14 pass
after the fix.

**No migrations, no API changes, no SPI propagation to consumers.**

---

## Out of Scope

`JpaCrossTenantLedgerEntryRepository` has four other inline `createQuery()` calls (`listAll`,
`findAllEvents`, `findEventsByActorId`, `findByTimeRange`) that also violate the `@NamedQuery`
convention. Fixing them is correct but separate from this bug fix — file a follow-up issue.