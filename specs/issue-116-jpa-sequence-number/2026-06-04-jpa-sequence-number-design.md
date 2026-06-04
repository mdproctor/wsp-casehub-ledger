# JPA Sequence Number Assignment — Design Spec

**Issue:** casehubio/ledger#116
**Branch:** issue-116-jpa-sequence-number
**Date:** 2026-06-04
**Subsumes:** #100 (concurrent sequence race under @ObservesAsync writers)
**Deferred:** #122 (PostgreSQL DevServices for real-DB integration tests)

---

## Problem

`JpaLedgerEntryRepository.save()` does not assign `sequenceNumber` before persisting. The field is a primitive `int` (defaults to 0), so every JPA-persisted entry gets `sequenceNumber = 0`. This produces three failures:

1. **Ordering** — `findBySubjectId()` sorts by `sequenceNumber ASC`. All entries sort equally.
2. **Merkle integrity** — `LedgerMerkleTree.leafHash(entry)` computes the leaf hash from `canonicalBytes()`, which includes `sequenceNumber`. The hash covers `0` instead of the actual sequence position.
3. **Agent signature** — `AgentSignatureEnricher` signs `canonicalBytes()` at `@PrePersist`. The signature covers `sequenceNumber = 0`. Verification against the stored entry (which also has `0`) will succeed, but the signed data does not represent the entry's actual position in the subject's history — the signature is meaningless for ordering provenance.

`InMemoryLedgerEntryRepository.save()` handles this correctly via `ConcurrentHashMap<UUID, AtomicInteger>` keyed by `subjectId`.

**Downstream beneficiaries:** `KeyRotationEntry` is a `LedgerEntry` subclass persisted through the same `save()` path. Its `subjectId` is derived from `actorId` via `UUID.nameUUIDFromBytes()`. It has had the same `sequenceNumber = 0` bug.

## Ordering Constraint

The assignment must happen before two downstream operations:

1. **Before `leafHash()`** — called explicitly before `em.persist()` in the JPA path.
2. **Before `@PrePersist` enrichers** — `AgentSignatureEnricher` runs inside `em.persist()` via `LedgerTraceListener` and calls `canonicalBytes()`.

Both operations consume `sequenceNumber`. If it's assigned after either, the hash or signature covers the wrong value.

### Enricher ordering asymmetry (pre-existing, now load-bearing)

The in-memory and JPA paths run enrichers at different points relative to `leafHash`:

| Path | Order |
|------|-------|
| In-memory | sequenceNumber → enrichers → leafHash → store |
| JPA | sequenceNumber → leafHash → `em.persist()` → `@PrePersist` enrichers → DB write |

Enrichers run BEFORE `leafHash` in memory, AFTER `leafHash` in JPA. This is only safe because **no enricher modifies canonical fields** (`subjectId`, `sequenceNumber`, `entryType`, `actorId`, `actorRole`, `occurredAt`). This invariant was implicit before; with correct sequence assignment it becomes load-bearing — a future enricher that modifies a canonical field would silently break hash consistency in the JPA path but not the in-memory path. The `LedgerEntryEnricher` SPI Javadoc should state this constraint explicitly.

## Concurrency Constraint

Multiple concurrent `save()` calls for the same `subjectId` must get distinct sequence numbers. Under PostgreSQL's default `READ COMMITTED` isolation, a naive `MAX(sequenceNumber) + 1` pattern is racy: two concurrent transactions both read `MAX = 5`, both write `6`.

The fix must prevent the race through serialization. Detection-and-retry (UNIQUE constraint + `ConstraintViolationException`) is ruled out: after a `PersistenceException`, the JPA `EntityManager` is in an undefined state and the transaction is marked rollback-only. Retry requires transaction restructuring (`REQUIRES_NEW` inner service), which is disproportionate complexity for sequence allocation.

### Why not a JPA entity with PESSIMISTIC_WRITE?

A `LedgerSubjectSequence @Entity` with `em.find(id, PESSIMISTIC_WRITE)` would be pure JPA and fully portable. It works correctly for existing rows — `SELECT FOR UPDATE` serializes concurrent access. But the first-insert race defeats it: you cannot pessimistic-lock a row that does not exist. Two concurrent first entries for a new subject both find null, both try to insert, one fails with a PK constraint violation — the same EM-poisoning problem. Solving this requires a `REQUIRES_NEW` inner service just for the initial insert, adding a class and a transaction boundary for a single edge case. The native SQL UPSERT handles both existing and new rows atomically in one statement.

## Design

### Schema

**V1000 (edit in place)** — Two changes to the base migration:

1. Upgrade the existing index to UNIQUE:
```sql
-- Before:
CREATE INDEX idx_ledger_entry_subject_seq ON ledger_entry (subject_id, sequence_number);
-- After:
CREATE UNIQUE INDEX idx_ledger_entry_subject_seq ON ledger_entry (subject_id, sequence_number);
```

2. Add the per-subject sequence counter table:
```sql
CREATE TABLE ledger_subject_sequence (
    subject_id UUID NOT NULL PRIMARY KEY,
    next_seq   INT  NOT NULL DEFAULT 1
);
```

The UNIQUE index enforces the invariant at the DB level — it is a safety net, not the mechanism. If it ever fires, it surfaces a bug in the sequence allocation logic. The sequence table is structurally part of the base schema — it is required for the ledger to function correctly.

### Sequence Allocation

A single native SQL statement using `INSERT ON CONFLICT DO UPDATE ... RETURNING`:

```java
private int nextSequenceNumber(UUID subjectId) {
    // next_seq stores the NEXT number to allocate. On first insert we set it to 2
    // (having just allocated 1). On conflict we increment and return the pre-increment
    // value via RETURNING next_seq - 1.
    Number nextSeq = (Number) em.createNativeQuery(
        "INSERT INTO ledger_subject_sequence (subject_id, next_seq) VALUES (?1, 2) " +
        "ON CONFLICT (subject_id) DO UPDATE SET next_seq = ledger_subject_sequence.next_seq + 1 " +
        "RETURNING next_seq - 1")
        .setParameter(1, subjectId)
        .getSingleResult();
    return nextSeq.intValue();
}
```

**H2 compatibility:** The test suite runs on H2 2.4.240 in `MODE=PostgreSQL`. This is the first native SQL in the codebase — there is no existing precedent confirming H2 handles this syntax. Test #0 (spike) must verify before any implementation proceeds. The spike cascades through three syntax levels:

1. **`INSERT ON CONFLICT DO UPDATE ... RETURNING`** (preferred — single statement)
2. **SQL-standard `MERGE INTO ... USING ... WHEN MATCHED/NOT MATCHED`** (fallback — two statements, MERGE + SELECT)
3. **H2-native `MERGE INTO ... KEY(col) VALUES(...)`** (last resort — H2 proprietary, two statements)

```sql
-- Fallback 1: SQL-standard MERGE
MERGE INTO ledger_subject_sequence AS t
USING (VALUES (?1)) AS s(subject_id)
ON t.subject_id = s.subject_id
WHEN MATCHED THEN UPDATE SET next_seq = t.next_seq + 1
WHEN NOT MATCHED THEN INSERT (subject_id, next_seq) VALUES (s.subject_id, 2)

-- Fallback 2: H2-native MERGE
MERGE INTO ledger_subject_sequence KEY(subject_id) VALUES (?1, 2)
```

All three have equivalent transactional semantics. Fallbacks 1 and 2 require a follow-up `SELECT next_seq - 1 FROM ledger_subject_sequence WHERE subject_id = ?1` to read the allocated value.

**JPA provider note:** `em.createNativeQuery("INSERT ... RETURNING ...").getSingleResult()` treats the statement as a query (returning a result set). Hibernate handles this for PostgreSQL, but H2's JDBC driver may treat `INSERT ... RETURNING` as DML, causing a "not a query" error from `getSingleResult()`. If this occurs, the graceful degradation is `executeUpdate()` for the UPSERT + a separate `SELECT` — effectively the two-statement approach. Test #0 covers this.

### Concurrency Behavior

| Scenario | Behavior |
|----------|----------|
| Two concurrent writers, row exists | Writer A's UPSERT acquires row lock, increments. Writer B blocks. A commits → B continues, increments, gets next value. |
| Two concurrent writers, new subject | Writer A's INSERT creates row. Writer B's INSERT conflicts → ON CONFLICT UPDATE increments. Both get distinct values. |
| Transaction rollback | UPSERT and entry INSERT share the same `@Transactional` boundary. Both roll back. Next successful transaction reuses the number — no gaps. |

### Updated save() Method

```java
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

Changes from current:
- `subjectId` null guard before `nextSequenceNumber()` — produces a clear error instead of an opaque native SQL stack trace.
- `entry.sequenceNumber = nextSequenceNumber(entry.subjectId)` after sanitisation, before `leafHash` and `em.persist()`.

### SPI Contract Update

`LedgerEntryRepository.save()` Javadoc:

> Persists a ledger entry with automatic sequence number assignment. The repository
> assigns `sequenceNumber` based on the entry's `subjectId` — any value set by the
> caller is overwritten. Sequence numbers are monotonically increasing and contiguous
> on insert within committed transactions. Retention deletion may remove entries from
> the start of the sequence without breaking the contiguity invariant for gap detection.

Same contract documented on `ReactiveLedgerEntryRepository.save()`.

The contiguity guarantee is intentional: the implementation produces 1, 2, 3, ... with no gaps (the UPSERT and entry INSERT share a transaction — rollback reverts both). `LedgerHealthJob.checkSequenceGaps()` relies on contiguity via `HAVING COUNT(e) != MAX(e.sequenceNumber) - MIN(e.sequenceNumber) + 1`. The contract must match the implementation and the health job's assumption.

### Enricher SPI Contract Update

`LedgerEntryEnricher` Javadoc should add:

> Enrichers must not modify canonical fields (`subjectId`, `sequenceNumber`, `entryType`,
> `actorId`, `actorRole`, `occurredAt`). The JPA and in-memory persistence paths run
> enrichers at different points relative to leaf hash computation. Modifying a canonical
> field in an enricher would break hash consistency in the JPA path.

### Relationship to #100

Issue #100 describes the same root cause from the consumer side: casehub-engine's `CaseLedgerEventCapture` computes sequence numbers before calling `save()`. With this fix:

- The repository always overwrites `sequenceNumber` → consumer-side computation is unnecessary.
- The UNIQUE constraint prevents duplicates at the DB level → the race in #100 is impossible.
- Consumer-side cleanup (removing redundant sequence computation in casehub-engine) is a follow-up, not blocking — the overwrite makes their computation harmless.

### Known Gaps

**Reactive JPA path:** No `JpaReactiveLedgerEntryRepository` exists today. `InMemoryReactiveLedgerEntryRepository` delegates to the blocking impl — no change needed. If a reactive JPA implementation is added in future, it will need to replicate the UPSERT pattern using `Mutiny.Session` and reactive native queries. The `ReactiveLedgerEntryRepository.save()` SPI Javadoc documents the contract; implementors will need to satisfy it.

### What's NOT Changed

- **In-memory path** — `InMemoryLedgerEntryRepository` already handles sequence assignment correctly. No changes.
- **Enricher pipeline** — No code changes. The enricher ordering invariant is documented, not enforced programmatically.

## Tests

0. **H2 syntax spike** — Cascading syntax test against H2 `MODE=PostgreSQL`. Try in order: (a) `INSERT ON CONFLICT DO UPDATE RETURNING` via `getSingleResult()`, (b) same UPSERT via `executeUpdate()` + separate SELECT, (c) SQL-standard `MERGE INTO ... USING ... WHEN MATCHED`, (d) H2-native `MERGE INTO ... KEY(...)`. Document whichever works and use it for the implementation. This gates all other tests.

1. **JPA sequence assignment** — Persist multiple entries for the same subject via JPA. Assert `sequenceNumber` values are 1, 2, 3, ... Assert ordering via `findBySubjectId()` matches insertion order.

2. **Multi-subject isolation** — Persist entries for two different subjects. Assert each subject has its own independent sequence (both start at 1).

3. **Idempotent re-save** — Create an entry with `sequenceNumber` pre-set by the caller. Call `save()`. Assert the repository overwrites the caller's value with the correct next sequence number.

4. **UNIQUE constraint enforcement** — Attempt to insert two entries with the same `(subject_id, sequence_number)` via native SQL (bypassing the repository). Assert constraint violation. Confirms the DB-level invariant.

5. **LeafHash correctness** — Persist an entry via JPA with hash-chain enabled. Recompute `leafHash()` from the persisted entry's fields. Assert the stored `digest` matches — confirming `sequenceNumber` was correct at hash time.

6. **Agent signature correctness** — Persist an entry via JPA with agent signing enabled. Verify the stored signature against `canonicalBytes()` of the persisted entry. Assert verification succeeds — confirming `sequenceNumber` was correct at signing time.

7. **KeyRotationEntry subclass** — Save via `KeyRotationService.recordRotation()`. Assert the persisted `KeyRotationEntry` has correct `sequenceNumber` (not 0). Exercises the subclass path.

8. **LedgerHealthJob integration** — Save N entries via JPA for a single subject. Run `LedgerHealthJob` gap detection. Assert no gap detected — confirms JPA-assigned sequences satisfy the health job's contiguity assumption.

9. **subjectId null guard** — Call `save()` with `subjectId = null`. Assert `IllegalArgumentException` with clear message.

10. **Concurrency (deferred to #122)** — Spin up N threads, each calling `save()` for the same `subjectId`. Assert all N entries get distinct, contiguous sequence numbers with zero constraint violations. This is the spec's central behavioral guarantee but cannot be meaningfully tested on H2 — H2's locking semantics under concurrent JDBC connections do not match PostgreSQL's row-level locking. Deferred to #122 (PostgreSQL DevServices) where real row locking can be validated.

## Protocol Coherence

- **Transaction demarcation (PP-20260602-a44c4e):** `save()` remains `@Transactional`. The sequence query runs within the same transaction. No new transaction boundaries introduced.
- **Test profile datasource (PP-20260529-6047d2):** New IT classes that use JPA need the standard test profile with `quarkus.arc.selected-alternatives` activating `JpaLedgerEntryRepository`.
- **DB vendor coverage (#122):** Native SQL is tested on H2 in PostgreSQL mode. Real PostgreSQL testing is deferred to #122.
