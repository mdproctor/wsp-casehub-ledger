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

## Ordering Constraint

The assignment must happen before two downstream operations:

1. **Before `leafHash()`** — called explicitly before `em.persist()` in the JPA path.
2. **Before `@PrePersist` enrichers** — `AgentSignatureEnricher` runs inside `em.persist()` via `LedgerTraceListener` and calls `canonicalBytes()`.

Both operations consume `sequenceNumber`. If it's assigned after either, the hash or signature covers the wrong value.

## Concurrency Constraint

Multiple concurrent `save()` calls for the same `subjectId` must get distinct sequence numbers. Under PostgreSQL's default `READ COMMITTED` isolation, a naive `MAX(sequenceNumber) + 1` pattern is racy: two concurrent transactions both read `MAX = 5`, both write `6`.

The fix must either prevent the race (serialization) or detect it (constraint + retry). The EM-poisoning problem rules out retry: after a `ConstraintViolationException`, the JPA `EntityManager` is in an undefined state and the transaction is marked rollback-only. Retry requires transaction restructuring (`REQUIRES_NEW` inner service), which is disproportionate complexity for sequence allocation.

## Design

### Schema

**V1000 (edit in place)** — Upgrade the existing index to UNIQUE:

```sql
-- Before:
CREATE INDEX idx_ledger_entry_subject_seq ON ledger_entry (subject_id, sequence_number);

-- After:
CREATE UNIQUE INDEX idx_ledger_entry_subject_seq ON ledger_entry (subject_id, sequence_number);
```

The UNIQUE index enforces the invariant at the DB level and serves as the query index. It is a safety net — the sequence allocation mechanism prevents duplicates, so this constraint should never fire. If it does, it surfaces a bug.

**V1009 (new migration)** — Per-subject sequence counter:

```sql
CREATE TABLE ledger_subject_sequence (
    subject_id UUID NOT NULL PRIMARY KEY,
    next_seq   INT  NOT NULL DEFAULT 1
);
```

One row per subject. Analogous to the in-memory `ConcurrentHashMap<UUID, AtomicInteger>`.

### Sequence Allocation

A private method in `JpaLedgerEntryRepository` using two native SQL statements within the existing `@Transactional`:

```java
private int nextSequenceNumber(UUID subjectId) {
    em.createNativeQuery(
        "INSERT INTO ledger_subject_sequence (subject_id, next_seq) VALUES (?1, 2) " +
        "ON CONFLICT (subject_id) DO UPDATE SET next_seq = ledger_subject_sequence.next_seq + 1")
        .setParameter(1, subjectId)
        .executeUpdate();

    Number nextSeq = (Number) em.createNativeQuery(
        "SELECT next_seq FROM ledger_subject_sequence WHERE subject_id = ?1")
        .setParameter(1, subjectId)
        .getSingleResult();

    return nextSeq.intValue() - 1;
}
```

**Statement 1 (UPSERT):** Atomically inserts a new row (first entry for subject, `next_seq = 2`) or increments the existing row's `next_seq`. Acquires a row-level lock in PostgreSQL.

**Statement 2 (SELECT):** Reads the post-increment value within the same transaction. The row lock from statement 1 is held until commit — concurrent writers block on statement 1, so this SELECT always returns the exclusively allocated value.

**Return:** `next_seq - 1` is the allocated sequence number. First entry returns 1, subsequent entries return the previous `next_seq` value.

**Why two statements instead of RETURNING:** `INSERT ON CONFLICT DO UPDATE ... RETURNING` has inconsistent support across DB engines. Splitting avoids dialect risk while maintaining transactional atomicity. The row lock from statement 1 guarantees the SELECT in statement 2 returns the correct value.

### Concurrency Behavior

| Scenario | Behavior |
|----------|----------|
| Two concurrent writers, row exists | Writer A's UPSERT acquires row lock, increments. Writer B blocks. A commits → B continues, increments, gets next value. |
| Two concurrent writers, new subject | Writer A's INSERT creates row. Writer B's INSERT conflicts → ON CONFLICT UPDATE increments. Both get distinct values. |
| Transaction rollback | UPSERT and entry INSERT are in the same transaction. Both roll back. Next successful transaction reuses the number — no gaps from rollbacks. |

### Updated save() Method

```java
@Override
@Transactional
public LedgerEntry save(final LedgerEntry entry) {
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

The only change is the addition of `entry.sequenceNumber = nextSequenceNumber(entry.subjectId)` after sanitisation and before `leafHash`.

### SPI Contract Update

`LedgerEntryRepository.save()` Javadoc:

> Persists a ledger entry with automatic sequence number assignment. The repository
> assigns `sequenceNumber` based on the entry's `subjectId` — any value set by the
> caller is overwritten. Sequence numbers are monotonically increasing per subject
> with no guarantee of contiguity.

Same contract documented on `ReactiveLedgerEntryRepository.save()`.

### Relationship to #100

Issue #100 describes the same root cause from the consumer side: casehub-engine's `CaseLedgerEventCapture` computes sequence numbers before calling `save()`. With this fix:

- The repository always overwrites `sequenceNumber` → consumer-side computation is unnecessary.
- The UNIQUE constraint prevents duplicates at the DB level → the race in #100 is impossible.
- Consumer-side cleanup (removing redundant sequence computation in casehub-engine) is a follow-up, not blocking — the overwrite makes their computation harmless.

### What's NOT Changed

- **In-memory path** — `InMemoryLedgerEntryRepository` already handles sequence assignment correctly. No changes.
- **Reactive path** — `InMemoryReactiveLedgerEntryRepository` delegates to the blocking impl. No separate fix needed.
- **Enricher pipeline** — No changes. The ordering fix ensures enrichers see the correct `sequenceNumber`.
- **No new JPA entity** — The sequence table is accessed only via native query. No `LedgerSubjectSequence` class.

## Tests

1. **JPA sequence assignment IT** — Persist multiple entries for the same subject via JPA. Assert `sequenceNumber` values are 1, 2, 3, ... Assert ordering via `findBySubjectId()` matches insertion order.

2. **Multi-subject isolation IT** — Persist entries for two different subjects. Assert each subject has its own independent sequence (both start at 1).

3. **UNIQUE constraint IT** — Attempt to insert two entries with the same `(subject_id, sequence_number)` via native SQL (bypassing the repository). Assert constraint violation. Confirms the DB-level invariant.

4. **LeafHash correctness** — Persist an entry via JPA with hash-chain enabled. Recompute `leafHash()` from the persisted entry's fields. Assert the stored `digest` matches — confirming `sequenceNumber` was correct at hash time.

5. **Agent signature correctness** — Persist an entry via JPA with agent signing enabled. Verify the stored signature against `canonicalBytes()` of the persisted entry. Assert verification succeeds — confirming `sequenceNumber` was correct at signing time.

## Protocol Coherence

- **Transaction demarcation (PP-20260602-a44c4e):** `save()` remains `@Transactional`. The sequence query runs within the same transaction. No new transaction boundaries introduced.
- **Test profile datasource (PP-20260529-6047d2):** New IT classes that use JPA need the standard test profile with `quarkus.arc.selected-alternatives` activating `JpaLedgerEntryRepository`.
- **DB vendor coverage (#122):** Native SQL is tested on H2 in PostgreSQL mode. Real PostgreSQL testing is deferred to #122.
