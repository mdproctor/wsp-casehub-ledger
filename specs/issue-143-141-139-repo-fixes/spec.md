# Spec: Repository Fixes — NoOp Trust Score, Dialect Detection, Merkle + Sequence Tenancy

**Branch:** `issue-143-noop-trust-score-repo`  
**Covers:** #143, #141, #139  
**Date:** 2026-06-15

---

## #143 — NoOpActorTrustScoreRepository + JpaActorTrustScoreRepository @Alternative

### Problem

`JpaActorTrustScoreRepository` is `@ApplicationScoped` (always active). No `@DefaultBean` no-op exists.
Consumers without a datasource and without `casehub-ledger-memory` must exclude it via
`quarkus.arc.exclude-types` — the fragile pattern eliminated for `LedgerEntryRepository` and
`ActorIdentityBindingRepository` by #138.

### CDI Tier Table

| Class | Annotation | When active |
|---|---|---|
| `NoOpActorTrustScoreRepository` | `@DefaultBean @ApplicationScoped` | Nothing else on classpath |
| `JpaActorTrustScoreRepository` | `@Alternative @ApplicationScoped` | Activated via `selected-alternatives` |
| `InMemoryActorTrustScoreRepository` | `@Alternative @Priority(1)` | `casehub-ledger-memory` on classpath — **no change** |

Pattern C from `alternative-extension-patterns.md`. `InMemoryActorTrustScoreRepository` already
carries `@Alternative @Priority(1)` and requires no change.

### New: `NoOpActorTrustScoreRepository`

`@DefaultBean @ApplicationScoped`, in `runtime/.../repository/`.

All 9 methods of `ActorTrustScoreRepository`:

| Method | Return |
|---|---|
| `findByActorId(actorId)` | `Optional.empty()` |
| `findCapabilityScore(actorId, tag)` | `Optional.empty()` |
| `findDimensionScore(actorId, dimension)` | `Optional.empty()` |
| `findCapabilityDimension(actorId, tag, dimension)` | `Optional.empty()` |
| `findCapabilityDimensions(actorId, tag)` | `List.of()` |
| `findByActorIdAndScoreType(actorId, type)` | `List.of()` |
| `upsert(...)` | no-op (13 parameters) |
| `updateGlobalTrustScore(actorId, score)` | no-op |
| `findAll()` | `List.of()` |
| `findAllByLastComputedAtAfter(since)` | `List.of()` |

### Changed: `JpaActorTrustScoreRepository`

Add `@Alternative` alongside existing `@ApplicationScoped`.

### Changed: `application.properties`

Add `io.casehub.ledger.runtime.repository.jpa.JpaActorTrustScoreRepository` to:

| Profile block | Action |
|---|---|
| Default `quarkus.arc.selected-alternatives` | Add |
| `%federation-import-test` | Add |
| `%federation-bootstrap-test` | Add |
| `%trust-score-bootstrap-test` | Add |
| `%scim-did-provider-test` | Add |
| `%noop-test` | **Omit** — deliberately tests the no-op path |

### Test: `NoOpActorTrustScoreRepositoryTest`

Pure unit test (no container). Verifies all 9 methods: all reads return empty/empty-list, `upsert` and
`updateGlobalTrustScore` complete without exception.

---

## #141 — Dialect Detection for Named Datasource

### Problem

`LedgerSequenceAllocator` reads `quarkus.datasource.db-kind` (the default datasource key). When a
consumer sets `casehub.ledger.datasource=myds`, the actual db-kind is at
`quarkus.datasource."myds".db-kind`. The injected value remains `"h2"`, causing H2 MERGE SQL to run
against a PostgreSQL connection — immediate SQL syntax error. The existing comment in the source
incorrectly references `casehubio/ledger#140`; the tracking issue is `#141`.

### Design

Replace `@ConfigProperty` with lazy dialect detection via the live JDBC connection.
`nextSequenceNumber()` is always called within `@Transactional`, so `em.unwrap(Session.class)` is
always valid at detection time.

```java
// Remove:
@ConfigProperty(name = "quarkus.datasource.db-kind", defaultValue = "h2")
String dbKind;

// Add:
private volatile Boolean postgresql = null;

private boolean isPostgresql() {
    Boolean result = postgresql;
    if (result == null) {
        result = em.unwrap(Session.class)
                .doReturningWork(conn ->
                    conn.getMetaData().getDatabaseProductName()
                            .toLowerCase(Locale.ROOT).contains("postgresql"));
        postgresql = result;
    }
    return result;
}
```

`volatile` provides cross-thread visibility. The redundant work on a first-call race is harmless —
both threads compute the same immutable value.

Also update the inline comment: `// Tracked in casehubio/ledger#140` → `// casehubio/ledger#141`.

### Tests

No new test class. `JpaSequenceNumberIT` (H2) exercises `isPostgresql() == false`;
`JpaSequenceNumberPgIT` (PostgreSQL/Testcontainers) exercises `isPostgresql() == true`.

---

## #139 — Merkle Frontier and Sequence Tenancy

### Problem (full scope)

`ledger_merkle_frontier` has no `tenancy_id`. `ledger_subject_sequence` has no `tenancy_id`.
Both are keyed by `subjectId` alone. `KeyRotationEntry` and `ActorIdentityBindingEntry` both
derive `subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))`. In a multi-tenant deployment,
two tenants sharing the same `actorId` (e.g. `claude:reviewer@v1`) get the same `subjectId`.

**Frontier collision (original issue):** both tenants write to the same `ledger_merkle_frontier`
rows. Each save overwrites the other tenant's frontier.

**Sequence collision (adjacent, must fix here):**

1. `LedgerVerificationService.inclusionProof()` computes the Merkle leaf index as
   `k = entry.sequenceNumber - 1`. With a shared counter, tenant A's entries get sequence numbers
   1, 3, 5 — making `k = 0, 2, 4`. These are wrong leaf positions in A's Merkle tree (which has
   leaves at 0, 1, 2). Inclusion proofs are **incorrect** for affected entries.

2. Separate from Merkle: `LedgerHealthJob.checkSequenceGaps()` groups by `subjectId` only. After
   per-tenant counters are applied, it would fire false-positive gap alerts for any `subjectId`
   shared across tenants.

3. `InMemoryLedgerEntryRepository` keys both `sequenceCounters` and `subjectLocks` by `UUID`
   (subjectId) alone — same collision.

Fixing the frontier without fixing the sequence counter leaves `inclusionProof()` broken. Both
must be fixed together.

### Schema (rewrite V1000 in place — no production installs)

**`ledger_merkle_frontier`:**
- Add `tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce'`
- Change `UNIQUE (subject_id, level)` → `UNIQUE (subject_id, tenancy_id, level)`
- Update index: `(subject_id)` → `(subject_id, tenancy_id)`

**`ledger_subject_sequence`:**
- Change `PRIMARY KEY (subject_id)` → `PRIMARY KEY (subject_id, tenancy_id)`
- Add `tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce'`

**`ledger_entry`:**
- Change `UNIQUE INDEX idx_ledger_entry_subject_seq (subject_id, sequence_number)` →
  `UNIQUE INDEX idx_ledger_entry_subject_seq (subject_id, tenancy_id, sequence_number)`

The default value `'278776f9-e1b0-46fb-9032-8bddebdcf9ce'` is the single-tenant sentinel
(`TenancyConstants.DEFAULT_TENANT_ID`). Single-tenant deployments continue working with zero
config change.

### API model `LedgerMerkleFrontier` (in `api/`)

Add:
```java
@Column(name = "tenancy_id", nullable = false)
public String tenancyId;
```

### Runtime model `LedgerMerkleFrontier` NamedQueries

```java
// findBySubjectId: add tenancyId filter
"SELECT f FROM LedgerMerkleFrontier f WHERE f.subjectId = :subjectId AND f.tenancyId = :tenancyId ORDER BY f.level ASC"

// deleteBySubjectAndLevel: add tenancyId filter
"DELETE FROM LedgerMerkleFrontier f WHERE f.subjectId = :subjectId AND f.level = :level AND f.tenancyId = :tenancyId"
```

### `JpaLedgerMerkleFrontierRepository`

`findBySubjectId`: pass `:tenancyId` to named query.

`replace`:
- Bulk DELETE: add `AND f.tenancyId = :tenancyId`
- Per-node `deleteBySubjectAndLevel` call: pass `:tenancyId`
- Before `em.persist(node)`: set `node.tenancyId = tenancyId`

`LedgerMerkleTree.append()` remains unchanged — pure algorithm, sets `subjectId` on returned nodes.
`tenancyId` is set by `replace()` immediately before persistence. Separation of concerns preserved.

### `LedgerSequenceAllocator`

Signature change: `nextSequenceNumber(UUID subjectId, String tenancyId)`.

All three SQL operations updated to key on `(subject_id, tenancy_id)`:

**PostgreSQL INSERT:** seed on `(subject_id, tenancy_id)`, ON CONFLICT on that pair.  
**PostgreSQL UPDATE:** WHERE clause includes `tenancy_id = ?`.  
**SELECT:** WHERE clause includes `tenancy_id = ?`.  
**H2 MERGE:** ON condition includes `tenancy_id`; INSERT includes `tenancy_id` column.

### `JpaLedgerEntryRepository`

```java
// Before:
entry.sequenceNumber = sequenceAllocator.nextSequenceNumber(entry.subjectId);
// After:
entry.sequenceNumber = sequenceAllocator.nextSequenceNumber(entry.subjectId, tenancyId);
```

(`tenancyId` is already in scope from the `save()` parameter.)

### `InMemoryLedgerEntryRepository`

Add private record:
```java
private record SubjectKey(UUID subjectId, String tenancyId) {}
```

Change:
- `ConcurrentHashMap<UUID, AtomicInteger> sequenceCounters` → `ConcurrentHashMap<SubjectKey, AtomicInteger>`
- `ConcurrentHashMap<UUID, Object> subjectLocks` → `ConcurrentHashMap<SubjectKey, Object>`

Both `computeIfAbsent` calls use `new SubjectKey(entry.subjectId, entry.tenancyId)`.

Side effect: tenants sharing a nameUUID `subjectId` no longer block each other on the per-subject
lock — correct and more concurrent.

### `InMemoryLedgerMerkleFrontierRepository`

Add private record:
```java
private record FrontierKey(UUID subjectId, String tenancyId) {}
```

Change `ConcurrentHashMap<UUID, List<LedgerMerkleFrontier>>` → `ConcurrentHashMap<FrontierKey, ...>`.
Update `findBySubjectId`, `replace`, `clear`.

### `LedgerHealthJob.checkSequenceGaps()`

JPQL updated to scope gaps per `(subjectId, tenancyId)`:

```java
"SELECT e.subjectId, e.tenancyId, COUNT(e), MIN(e.sequenceNumber), MAX(e.sequenceNumber) " +
"FROM LedgerEntry e " +
"GROUP BY e.subjectId, e.tenancyId " +
"HAVING COUNT(e) != MAX(e.sequenceNumber) - MIN(e.sequenceNumber) + 1"
```

### `LedgerGapDetected` record

Add `tenancyId` field (nullable `String` — `null` for `RECONCILIATION_MISMATCH` which operates at
entity-type granularity, not per-tenant):

```java
public record LedgerGapDetected(
        String subjectId,
        String tenancyId,       // new — null for RECONCILIATION_MISMATCH
        long expectedCount,
        long actualCount,
        GapType type) {}
```

Existing observers that ignore `tenancyId` continue compiling but must be updated to pass the field
when constructing the event.

### No-change confirmations

**`NoOpLedgerMerkleFrontierRepository`:** already accepts `tenancyId` in both SPI signatures and
correctly ignores it (no-op has no state to isolate). No change needed.

**`LedgerVerificationService`:** no code change needed. `treeRoot`, `inclusionProof`, and `verify`
already pass `tenancyId` through to `frontierRepo.findBySubjectId`. After the sequence fix,
`inclusionProof`'s `k = entry.sequenceNumber - 1` is correct because per-tenant counters are
contiguous starting from 1 — the bug is eliminated by fixing the counter, not by changing
`LedgerVerificationService`.

### Design note: `tenancyId` excluded from canonical hash

`tenancyId` is storage metadata, not entry content. It controls *where* the frontier is stored,
not what the entry proves. The canonical hash covers the tamper-evident content of the entry
itself (structural metadata, supplementJson, subclass domain fields). A tenant cannot substitute
another tenant's frontier for verification because the frontier is now keyed by `tenancyId` — so
the storage isolation provides the isolation guarantee without making `tenancyId` part of the hash.
Including it in the hash would add a dependency on storage topology to a content-addressing scheme,
which is incorrect design.

### Tests

**`InMemoryLedgerMerkleFrontierRepositoryTest`:** add test — two tenants with identical `subjectId`
produce independent frontiers.

**New `MerkleFrontierTenancyIT`:** `@QuarkusTest` with `@TestProfile(MerkleFrontierTenancyProfile.class)`.

Profile: `%merkle-tenancy-test` block in `application.properties` — isolated H2 URL, hash chain
enabled, `JpaLedgerMerkleFrontierRepository` active via default selected-alternatives block.

Test asserts: two `KeyRotationEntry` saves for the same `actorId` but different `tenancyId` values
produce independent, correctly-ordered frontiers retrievable per tenant.

**`JpaSequenceNumberIT` and `JpaSequenceNumberPgIT`:** update `nextSequenceNumber` calls to include `tenancyId`.

**`LedgerHealthJobIT` and `LedgerHealthJobPgIT`:** update gap detection assertions to include
`tenancyId` in the detected event.
