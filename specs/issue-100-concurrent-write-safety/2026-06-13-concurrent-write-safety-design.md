# Concurrent Write Safety — Sequence Allocation and Merkle Frontier

**Issue:** casehubio/ledger#100
**Branch:** issue-100-concurrent-write-safety
**Date:** 2026-06-13

---

## Problem

`LedgerEntryRepository.save()` for the same `subjectId` from concurrent `@ObservesAsync` handlers
can corrupt the Merkle hash chain. In the casehub-engine scenario, `CaseLifecycleEvent` and
`WorkerDecisionEvent` fire in the same Mutiny chain for every worker completion — both target the
same `subjectId` and both call `save()` concurrently.

---

## What Is Already Correct

The following was addressed in fix(#116) and does **not** need to change:

- **`LedgerSequenceAllocator`** uses a SQL MERGE on `ledger_subject_sequence` to atomically allocate
  sequence numbers. For existing subjects (row already in `ledger_subject_sequence`), the MERGE takes a
  write lock on that row, serialising concurrent saves end-to-end: the second transaction blocks on MERGE
  until the first commits — including its Merkle frontier update. This incidentally makes the Merkle chain
  correct for existing subjects at no extra cost.
- **`InMemoryLedgerEntryRepository`** uses `ConcurrentHashMap<UUID, AtomicInteger>` with
  `incrementAndGet()` — sequence numbers are already race-safe.
- **`UNIQUE INDEX idx_ledger_entry_subject_seq`** on `(subject_id, sequence_number)` is in V1000 as a
  last-resort safety net.
- Serial sequence correctness tests exist in `JpaSequenceNumberIT` / `JpaSequenceNumberPgIT`.

---

## Remaining Gaps

### Gap 1 — First-entry MERGE race

When two concurrent transactions both call `nextSequenceNumber()` for a **brand-new** subject (no row
yet in `ledger_subject_sequence`), both hit MERGE's `WHEN NOT MATCHED` branch. Because there is no row
to lock, both attempt INSERT simultaneously. One succeeds; the other gets a primary key constraint
violation. No retry logic exists. The transaction aborts and the save is lost.

### Gap 2 — InMemory Merkle frontier race

`InMemoryLedgerEntryRepository.save()` does:
```
sequenceNumber = AtomicInteger.incrementAndGet()   // atomic
// ... prepareKey, enrich, hash, sign ...
currentFrontier = frontierRepo.findBySubjectId()   // read
newFrontier = LedgerMerkleTree.append(digest, current)  // compute
frontierRepo.replace(subjectId, newFrontier)        // write
```

If seq=2 races ahead of seq=1 into the frontier read, it reads an empty frontier, computes a frontier
rooted at its own digest, and overwrites seq=1's contribution. Worse: the `AtomicInteger` allocates
seq=1 and seq=2 atomically, but the Merkle update runs outside any lock. seq=2 could reach the frontier
read first, producing a chain with the wrong ordering at the base.

### Gap 3 — No concurrent integration test

All existing `JpaSequenceNumberIT` tests are `@Transactional` (single thread, single transaction).
No test exercises actual concurrent saves and proves that sequences are unique, contiguous, and the
Merkle chain is valid under concurrency.

---

## Design

### Fix 1 — `INSERT ON CONFLICT` in `LedgerSequenceAllocator`

Replace the MERGE with `INSERT ... ON CONFLICT DO UPDATE`. PostgreSQL's upsert is proven race-safe for
concurrent first-inserts: when two transactions race for a new subject, the second blocks on the implicit
row lock until the first commits, then applies the `DO UPDATE` branch rather than retrying a duplicate
INSERT. H2 2.x supports the same syntax.

```java
em.createNativeQuery(
        "INSERT INTO ledger_subject_sequence (subject_id, next_seq) " +
        "VALUES (CAST(?1 AS UUID), 2) " +
        "ON CONFLICT (subject_id) DO UPDATE " +
        "SET next_seq = ledger_subject_sequence.next_seq + 1")
        .setParameter(1, subjectId)
        .executeUpdate();
em.flush();
final Number seq = (Number) em.createNativeQuery(
        "SELECT next_seq - 1 FROM ledger_subject_sequence WHERE subject_id = ?1")
        .setParameter(1, subjectId)
        .getSingleResult();
return seq.intValue();
```

**`CAST(?1 AS UUID)` in the VALUES clause is required.** H2 cannot coerce a raw `?1` parameter to
UUID when binding against a UUID-typed column — the same reason the existing MERGE used
`USING (SELECT CAST(?1 AS UUID) AS sid)`. Omitting it causes a type-mismatch error in H2 tests.

**`ledger_subject_sequence.next_seq` in `DO UPDATE SET`** references the existing row value — correct
PostgreSQL semantics. H2's PostgreSQL compatibility mode applies the same semantics. The existing
H2 serial tests (`JpaSequenceNumberIT`) exercise `nextSequenceNumber()` on every `repo.save()` call
and will catch any H2 divergence in this clause.

`em.flush()` between the upsert and the SELECT is unchanged — required to ensure the upsert is flushed
to the database before the SELECT reads the result within the same transaction. Two-step structure
retained because H2's `ON CONFLICT` does not support `RETURNING`.

No schema changes. No new migration.

### Fix 2 — Per-subject lock in `InMemoryLedgerEntryRepository`

Add a `ConcurrentHashMap<UUID, Object>` of per-subject mutex objects. Move sequence allocation and
everything that follows (enrich, hash, sign, store, Merkle update) inside a `synchronized` block on
the per-subject lock.

`agentEntrySigner.prepareKey(entry)` moves to the **pre-lock** section. It is per-entry stateless:
reads from a `ConcurrentHashMap<String, KeyPair>` keyed by actorId (`InMemoryAgentSigner`), writes
only to `entry.agentPublicKey` and `entry.agentKeyRef` — fields on the entry instance, not shared
between concurrent threads. It has no dependency on `sequenceNumber` and touches no per-subject shared
state. Moving it pre-lock reduces lock hold time and correctly matches the treatment of other
per-entry stateless work (tokenise, sanitise). The ordering invariant (`prepareKey` before
`enricherPipeline.enrich`) is preserved: pre-lock work completes before in-lock work for any given
entry.

```
// Pre-work (concurrent across all subjects):
entry.tenancyId = tenancyId
entry.id = UUID.randomUUID() if null
entry.occurredAt = Instant.now() if null
entry.actorId = actorIdentityProvider.tokenise(...)
entry.compliance() sanitise
agentEntrySigner.prepareKey(entry)   // ← pre-lock: per-entry stateless

// Per-subject serialised section:
lock = subjectLocks.computeIfAbsent(subjectId, k -> new Object())
synchronized (lock):
    entry.sequenceNumber = sequenceCounters.computeIfAbsent(sid, k -> new AtomicInteger(0))
                                           .incrementAndGet()
    enricherPipeline.enrich(entry)
    if hashChain.enabled: entry.digest = LedgerMerkleTree.leafHash(entry)
    agentEntrySigner.sign(entry)
    entries.put(entry.id, entry)
    if hashChain.enabled:
        current  = frontierRepo.findBySubjectId(subjectId, tenancyId)
        newFront = LedgerMerkleTree.append(entry.digest, current, subjectId)
        frontierRepo.replace(subjectId, newFront, tenancyId)
        merklePublisher.publish(subjectId, entry.sequenceNumber, treeRoot(newFront))
```

From the perspective of **concurrent saves to the same subject**, the per-subject lock guarantees that
entry and frontier are both written before the lock is released — a second concurrent save cannot read
a partial state. This does NOT extend to concurrent reads: `findBySubjectId()` and all query methods
access `entries` via `ConcurrentHashMap.values()` without holding the per-subject lock.
`ConcurrentHashMap.put()` makes the entry immediately visible to any concurrent reader via the map's
internal volatile write, regardless of the `synchronized` block. A concurrent read can therefore
observe an entry while the Merkle frontier update is still in progress — the same window that exists
in the JPA path under READ COMMITTED isolation.

`AtomicInteger` is retained (its atomicity is now redundant under the lock but harmless). Different
subjects still run fully concurrently — the lock is per-subject, not global.

`clear()` must also reset `subjectLocks`. The complete updated method:

```java
public void clear() {
    entries.clear();
    attestations.clear();
    sequenceCounters.clear();
    subjectLocks.clear();   // new
    if (frontierRepo instanceof InMemoryLedgerMerkleFrontierRepository m) {
        m.clear();
    }
}
```

### Fix 3 — Concurrent integration tests

**`JpaSequenceNumberConcurrentPgIT`** (runtime module, PostgreSQL via Testcontainers):

Test methods are **not** `@Transactional`. Each `repo.save()` call from a pool thread starts its own
transaction, exercising the actual concurrent behaviour. Fresh subject UUIDs per test — no `@BeforeEach`
cleanup needed.

Start-gate pattern: `CountDownLatch(8)` ready + `CountDownLatch(1)` start ensures all 8 threads are
alive and blocked before any starts saving (maximises contention).

Three tests:

| Test | Assertions |
|------|-----------|
| `concurrentSavesHaveUniqueContiguousSequences` | (1) All 8 futures completed without exception. (2) `findBySubjectId` returns exactly 8 entries. (3) Sorted sequences == [1..8], no gaps, no duplicates. |
| `concurrentSavesProduceValidMerkleChain` | (1) All 8 futures completed without exception. (2) `findBySubjectId` returns exactly 8 entries. (3) Replay `append()` in sequence order → `treeRoot(expected) == treeRoot(actual)`. |
| `concurrentSavesForDifferentSubjectsAreIsolated` | 2 subjects × 4 threads each. Per subject: (1) all 4 futures completed. (2) exactly 4 entries returned. (3) sequences == [1..4]. |

**Asserting all N saves completed** is essential: if a thread throws and is silently discarded, the
Merkle replay runs over N-1 entries and may still pass — both frontier and replay use the same N-1
digests. Each thread's `Future<LedgerEntry>` must be checked with `future.get()` (which re-throws),
and `findBySubjectId` must return exactly N entries.

Merkle verification uses `LedgerMerkleTree` static methods directly — replay `append()` over entries
sorted by `sequenceNumber`, then compare `treeRoot`. No `LedgerVerificationService` needed.

In the JPA path, `entry.digest` is read directly (field is accessible in the runtime module — proven
by the existing `JpaSequenceNumberIT.leafHashCoversCorrectSequenceNumber` test). Using the stored
digest is sufficient here: Fix 1's row lock ensures that `sequenceNumber` is assigned before
`leafHash()` is called within the same transaction, so a wrong-digest-before-sequenceNumber bug
cannot occur. If it could, assertion (1) — "all futures completed without exception" — would catch
the constraint violation before the Merkle check is reached.

In the InMemory path (see below), `LedgerMerkleTree.leafHash(e)` recomputation is used instead
because `entry.digest` is inaccessible. These two approaches are intentionally different for
module-constraint reasons, not because the InMemory test is stricter — both are correct for their
respective modules.

**`InMemoryLedgerEntryConcurrentTest`** (persistence-memory module, `@QuarkusTest`):

Same three tests, same thread count (8), same assertions, against the InMemory backend.
`@BeforeEach repo.clear()` since there is no rollback.

The persistence-memory module has a concrete implementation constraint: Quarkus bytecode enhancement
changes `@Entity` public fields to `protected` in the augmented classloader. Accessing `LedgerEntry`
fields directly from a test class (not a subclass) causes `IllegalAccessError`. This is documented in
`InMemoryLedgerEntryRepositoryTest` and handled by the existing `MemoryTestEntry` pattern. The
InMemory concurrent test must follow the same pattern:

1. **Entries created as `MemoryTestEntry.of(subjectId, LedgerEntryType.EVENT)` instances** — not raw
   `TestEntry` or any other runtime-module test helper.

2. **Sequence assertions via accessor** — `((MemoryTestEntry) e).getSequenceNumber()` for every
   sequence read. Direct `e.sequenceNumber` access throws `IllegalAccessError`.

3. **Merkle verification via `LedgerMerkleTree.leafHash(e)` recomputation, not `entry.digest` reads.**
   `MemoryTestEntry` has no `getDigest()` accessor; `entry.digest` access throws `IllegalAccessError`.
   The correct approach: after collecting all saved entries sorted by `getSequenceNumber()`, build the
   expected frontier by calling `LedgerMerkleTree.leafHash(e)` per entry and feeding each into
   `LedgerMerkleTree.append()`. This is a *stronger* assertion than reading the stored digest — it
   catches any bug where `entry.digest` was computed before `sequenceNumber` was assigned (since
   `canonicalBytes()`, which `leafHash()` calls, includes `sequenceNumber`).

4. **Frontier read via `@Inject InMemoryLedgerMerkleFrontierRepository frontierRepo`** — the concrete
   class, not the `LedgerMerkleFrontierRepository` interface. The reason is not `clear()` —
   `@BeforeEach` calls `repo.clear()`, which already clears the frontier via its `instanceof` check.
   The reason is the Merkle assertion: the test needs
   `frontierRepo.findBySubjectId(subjectId, DEFAULT_TENANT_ID)` to obtain the actual frontier for
   `treeRoot(actual)`. Injecting the concrete class makes the dependency explicit; the interface would
   work equally well for this call, but the concrete class is consistent with how
   `InMemoryLedgerEntryRepositoryTest` injects its repositories.

---

## Merkle Serialization Invariant (JPA path)

The following three facts must simultaneously hold for the JPA path to guarantee Merkle chain
correctness under concurrent writes. **This is not documented anywhere else.** If any of these
three facts changes, the Merkle chain silently corrupts without any test or constraint catching it.

1. **`save()` is `@Transactional`** — the entire save pipeline runs in one transaction.
2. **`nextSequenceNumber()` is the first per-subject serializing DB write in `save()`** — the
   `INSERT ON CONFLICT DO UPDATE` acquires a write lock on the `ledger_subject_sequence` row for this
   subject. This lock is held for the duration of the transaction, which covers the Merkle frontier
   update. A second concurrent save for the same subject blocks on this lock until the first commits,
   guaranteeing it reads the committed frontier. Other writes may precede this call (e.g.,
   `actorIdentityProvider.tokenise()` for `ActorType.HUMAN` writes to `actor_identity`), but those
   are per-actor, not per-subject, and do not affect the subject-level serialization guarantee.
3. **`JpaLedgerMerkleFrontierRepository.replace()` participates in the same transaction** — via
   `@Transactional(REQUIRED)` propagation. If this becomes `NOT_SUPPORTED` or starts a new
   transaction, the frontier update is no longer covered by the lock.

**If any of these changes:** `save()` split into two transactions, sequence allocation moved to
`REQUIRES_NEW`, `replace()` made `NOT_SUPPORTED` — the serialization guarantee breaks. Two concurrent
saves would both read an old frontier, and the last writer would silently overwrite the other's
contribution.

A protocol entry should be created in `casehubio/garden` to formalize this invariant.

---

## Platform Coherence

- Fix is internal to ledger — no cross-repo coordination required.
- No SPI changes, no new CDI events, no API surface changes.
- No schema changes — UNIQUE index and `ledger_subject_sequence` table are already in V1000.
- PLATFORM.md capability ownership table does not need updating.

---

## Out of Scope

- Retry logic in `save()` for `ConstraintViolationException` — not needed; `INSERT ON CONFLICT`
  eliminates the race that would cause it.
- `SELECT FOR UPDATE` on Merkle frontier reads in the JPA path — not needed; the `INSERT ON CONFLICT`
  row lock (held for the full `@Transactional save()` duration) already serialises the entire pipeline
  per subject, including the frontier update. See Merkle Serialization Invariant above.
- InMemory concurrency guarantee when Merkle is **disabled** (`hashChain.enabled=false`) — sequences
  are still atomic via `AtomicInteger`, and there is no frontier to corrupt.
- **Merkle frontier tenant-scope gap (#139)** — `LedgerMerkleFrontier` has no `tenancyId` column;
  `JpaLedgerMerkleFrontierRepository.findBySubjectId()` ignores `tenancyId`. For `KeyRotationEntry`
  and `ActorIdentityBindingEntry` — both of which derive `subjectId` as
  `UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))` — two tenants sharing the same `actorId` produce
  the same `subjectId` UUID. Their key-rotation and identity-binding Merkle frontiers would collide:
  each save overwrites the other's frontier rows, making verification fail for both chains. Fix
  requires adding `tenancyId` to `ledger_merkle_frontier` and keying the frontier by
  `(subject_id, tenancy_id)`. Tracked in casehubio/ledger#139.