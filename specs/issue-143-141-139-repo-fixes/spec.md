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

Pure unit test (no container). Verifies all 9 methods: all reads return empty/empty-list, `upsert`
and `updateGlobalTrustScore` complete without exception.

---

## #141 — Dialect Detection for Named Datasource

### Problem

`LedgerSequenceAllocator` reads `quarkus.datasource.db-kind` (the default datasource key). When a
consumer sets `casehub.ledger.datasource=myds`, the actual db-kind is at
`quarkus.datasource."myds".db-kind`. The injected value remains `"h2"`, causing H2 MERGE SQL to run
against a PostgreSQL connection — immediate SQL syntax error. The existing inline comment
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

Replace `if ("postgresql".equals(dbKind))` with `if (isPostgresql())`.

Also update the inline comment: `// Tracked in casehubio/ledger#140` → `// casehubio/ledger#141`.

`volatile` provides cross-thread visibility. The redundant work on a first-call race is harmless —
both threads compute the same immutable value.

### Tests

No new test class. `JpaSequenceNumberIT` (H2) exercises `isPostgresql() == false`;
`JpaSequenceNumberPgIT` (PostgreSQL/Testcontainers) exercises `isPostgresql() == true`.

---

## #139 — Merkle Frontier and Sequence Tenancy

### Problem (full scope)

`ledger_merkle_frontier` has no `tenancy_id`. `ledger_subject_sequence` has no `tenancy_id`. Both are
keyed by `subjectId` alone. `KeyRotationEntry` and `ActorIdentityBindingEntry` both derive
`subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))`. In a multi-tenant deployment, two
tenants sharing the same `actorId` (e.g. `claude:reviewer@v1`) get the same `subjectId`.

**Frontier collision:** both tenants write to the same `ledger_merkle_frontier` rows. Each save
overwrites the other tenant's frontier.

**Sequence collision — Merkle proof correctness:** `LedgerVerificationService.inclusionProof()`
computes the Merkle leaf index as `k = entry.sequenceNumber - 1`. With a shared counter, tenant A's
entries get sequence numbers 1, 3, 5 — making `k = 0, 2, 4`. These are wrong leaf positions in A's
3-entry Merkle tree (leaves at 0, 1, 2). Inclusion proofs are **cryptographically incorrect** for
affected entries. Fixing the frontier without fixing the sequence counter leaves inclusion proofs
broken.

**Sequence collision — health monitoring:** `LedgerHealthJob.checkSequenceGaps()` groups by `subjectId`
only. After per-tenant counters, different tenants' entries for the same nameUUID `subjectId` overlap
in sequence number space, producing false-positive gap alerts.

Both the frontier and the sequence counter must be fixed together.

---

### Schema (rewrite V1000 in place — no production installs)

**`ledger_merkle_frontier`:**
- Add `tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce'`
- Change `UNIQUE (subject_id, level)` → `UNIQUE (subject_id, tenancy_id, level)`
- Update index: `(subject_id)` → `(subject_id, tenancy_id)`

**`ledger_subject_sequence`:**
- Add `tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce'`
- Change `PRIMARY KEY (subject_id)` → `PRIMARY KEY (subject_id, tenancy_id)`

**`ledger_entry`:**
- Change `UNIQUE INDEX idx_ledger_entry_subject_seq (subject_id, sequence_number)` →
  `UNIQUE INDEX idx_ledger_entry_subject_seq (subject_id, tenancy_id, sequence_number)`

The default `'278776f9-e1b0-46fb-9032-8bddebdcf9ce'` is `TenancyConstants.DEFAULT_TENANT_ID`.
Single-tenant deployments continue working with zero config change.

---

### API model `LedgerMerkleFrontier` (in `api/`)

Add:
```java
@Column(name = "tenancy_id", nullable = false)
public String tenancyId;
```

---

### Runtime model `LedgerMerkleFrontier` NamedQueries

```java
// findBySubjectId: add tenancyId filter
"SELECT f FROM LedgerMerkleFrontier f WHERE f.subjectId = :subjectId AND f.tenancyId = :tenancyId ORDER BY f.level ASC"

// deleteBySubjectAndLevel: add tenancyId filter
"DELETE FROM LedgerMerkleFrontier f WHERE f.subjectId = :subjectId AND f.level = :level AND f.tenancyId = :tenancyId"
```

---

### `JpaLedgerMerkleFrontierRepository`

`findBySubjectId`: pass `:tenancyId` parameter to named query.

`replace`:
- Bulk DELETE: add `AND f.tenancyId = :tenancyId`
- Per-node `deleteBySubjectAndLevel` call: pass `:tenancyId`
- Before `em.persist(node)`: set `node.tenancyId = tenancyId`

`LedgerMerkleTree.append()` remains unchanged — pure algorithm, sets `subjectId` on returned nodes.
`tenancyId` is set by `replace()` immediately before persistence.

---

### `LedgerSequenceAllocator`

Signature: `int nextSequenceNumber(UUID subjectId, String tenancyId)`.

All SQL operations key on `(subject_id, tenancy_id)`:

**PostgreSQL INSERT:** seed row on `(subject_id, tenancy_id)`, `ON CONFLICT (subject_id, tenancy_id) DO NOTHING`.  
**PostgreSQL UPDATE:** `WHERE subject_id = CAST(?1 AS UUID) AND tenancy_id = ?2`.  
**SELECT:** `WHERE subject_id = ?1 AND tenancy_id = ?2`.  
**H2 MERGE:** `ON t.subject_id = s.sid AND t.tenancy_id = s.tid`; INSERT includes `tenancy_id` column.

---

### `JpaLedgerEntryRepository` (call site 1)

```java
// Before:
entry.sequenceNumber = sequenceAllocator.nextSequenceNumber(entry.subjectId);
// After (tenancyId already in scope from save() parameter):
entry.sequenceNumber = sequenceAllocator.nextSequenceNumber(entry.subjectId, tenancyId);
```

### `JpaActorIdentityBindingRepository` (call site 2 — line 56)

```java
// Before:
entry.sequenceNumber = sequenceAllocator.nextSequenceNumber(entry.subjectId);
// After (entry.tenancyId set by ActorIdentityBindingObserver.persistBinding() before save()):
entry.sequenceNumber = sequenceAllocator.nextSequenceNumber(entry.subjectId, entry.tenancyId);
```

These are the only two call sites of `nextSequenceNumber()`, confirmed via reference search.

---

### `InMemoryLedgerEntryRepository`

Add private record:
```java
private record SubjectKey(UUID subjectId, String tenancyId) {}
```

Change:
- `ConcurrentHashMap<UUID, AtomicInteger> sequenceCounters` → `ConcurrentHashMap<SubjectKey, AtomicInteger>`
- `ConcurrentHashMap<UUID, Object> subjectLocks` → `ConcurrentHashMap<SubjectKey, Object>`

Both `computeIfAbsent` calls use `new SubjectKey(entry.subjectId, entry.tenancyId)`.

Side-effect: tenants sharing a nameUUID `subjectId` no longer contend on the same lock —
more concurrent and architecturally correct.

---

### `InMemoryLedgerMerkleFrontierRepository`

Add private record:
```java
private record FrontierKey(UUID subjectId, String tenancyId) {}
```

Change `ConcurrentHashMap<UUID, List<LedgerMerkleFrontier>>` → `ConcurrentHashMap<FrontierKey, ...>`.
Update `findBySubjectId`, `replace`, `clear`.

---

### `LedgerHealthJob` + anomaly event redesign

`LedgerGapDetected` already conflates two incompatible shapes: `subjectId` is a UUID-string for
`SEQUENCE_GAP` and an entity-type name for `RECONCILIATION_MISMATCH`. Every observer must check
`type()` to know which fields are meaningful. Adding nullable `tenancyId` deepens the smell. Fix the
design with a sealed interface.

#### Delete

- `LedgerGapDetected` record
- `GapType` enum

#### New: `LedgerAnomalyDetected` sealed hierarchy

```java
public sealed interface LedgerAnomalyDetected
        permits LedgerSequenceGapDetected, LedgerReconciliationMismatchDetected {}

public record LedgerSequenceGapDetected(
        UUID subjectId,
        String tenancyId,
        long expectedCount,
        long actualCount)
        implements LedgerAnomalyDetected {}

public record LedgerReconciliationMismatchDetected(
        String entityType,
        long domainCount,
        long ledgerCount)
        implements LedgerAnomalyDetected {}
```

Observers pattern-match on the sealed type. `subjectId` in `LedgerSequenceGapDetected` is typed
`UUID` (not `String`) — always a genuine aggregate UUID at this call site.

#### `LedgerHealthJob` changes

Inject `Event<LedgerAnomalyDetected>` (replaces `Event<LedgerGapDetected>`).

**`checkSequenceGaps()` — JPQL and Java tuple-reading both change:**

```java
// JPQL: add e.tenancyId at position 1
"SELECT e.subjectId, e.tenancyId, COUNT(e), MIN(e.sequenceNumber), MAX(e.sequenceNumber) " +
"FROM LedgerEntry e " +
"GROUP BY e.subjectId, e.tenancyId " +
"HAVING COUNT(e) != MAX(e.sequenceNumber) - MIN(e.sequenceNumber) + 1"

// Java tuple-reading: positions shift — must match JPQL projection exactly
UUID   subjectId   = (UUID)   row[0];   // unchanged position, now UUID not toString()
String tenancyId   = (String) row[1];   // NEW
long   actualCount = ((Number) row[2]).longValue();  // was row[1]
long   min         = ((Number) row[3]).longValue();  // was row[2]
long   max         = ((Number) row[4]).longValue();  // was row[3]
long   expected    = max - min + 1;

gapEvent.fire(new LedgerSequenceGapDetected(subjectId, tenancyId, expected, actualCount));
```

**`checkReconciliation()` — fire site changes:**

```java
// Before:
gapEvent.fire(new LedgerGapDetected(source.subjectType(), domainCount, ledgerCount, GapType.RECONCILIATION_MISMATCH));
// After:
gapEvent.fire(new LedgerReconciliationMismatchDetected(source.subjectType(), domainCount, ledgerCount));
```

Update `LedgerReconciliationSource.subjectType()` Javadoc: replace reference to `LedgerGapDetected`
with `LedgerReconciliationMismatchDetected.entityType`.

---

### No-change confirmations

**`NoOpLedgerMerkleFrontierRepository`:** correctly accepts `tenancyId` in both SPI signatures and
ignores it (no state to isolate). No change needed.

**`LedgerVerificationService`:** no code change needed. `treeRoot`, `inclusionProof`, and `verify`
already pass `tenancyId` to `frontierRepo.findBySubjectId`. After the sequence fix, per-tenant
counters are contiguous from 1, so `k = entry.sequenceNumber - 1` yields correct leaf positions —
the correctness bug is eliminated by fixing the counter, not by changing `LedgerVerificationService`.

---

### Design note: `tenancyId` excluded from canonical hash

`tenancyId` is storage metadata, not entry content. Tenant isolation after this fix is provided by
the keying of frontier rows and sequence rows on `(subject_id, tenancy_id)` — a tenant cannot
substitute another tenant's frontier during verification. Including `tenancyId` in `canonicalBytes()`
would introduce a dependency on storage topology into a content-addressing scheme, which is incorrect.
The canonical hash covers what the entry proves; where it is stored is orthogonal.

---

### Pre-existing gaps (not in scope, filed for follow-up)

**#144 — `JpaActorIdentityBindingRepository.save()` never calls `frontierRepo.replace()`.**
`LedgerVerificationService.treeRoot()`, `inclusionProof()`, and `verify()` are broken for any subject
whose chain consists entirely of `ActorIdentityBindingEntry` rows — they always see an empty frontier.
Not introduced by this PR. `KeyRotationEntry` is unaffected because it saves via
`LedgerEntryRepository.save()` → `JpaLedgerEntryRepository.updateMerkleFrontier()`.

**#145 — `latestBindingFor(actorId)` and `bindingHistoryFor(actorId)` have no `tenancyId` filter.**
In a multi-tenant deployment with shared `actorId`, both return binding entries from any tenant.
Read-side counterpart of the write-side collision fixed here.

---

### Tests

**`InMemoryLedgerMerkleFrontierRepositoryTest`:** add test — two tenants with identical `subjectId`
produce independent frontiers; save by tenant A does not overwrite tenant B's frontier.

**New `MerkleFrontierTenancyIT`:** `@QuarkusTest` with `@TestProfile(MerkleFrontierTenancyProfile.class)`.
Add `%merkle-tenancy-test` block to `application.properties` with isolated H2 URL and hash chain
enabled. `JpaLedgerMerkleFrontierRepository` is active via the default `selected-alternatives` block.

Use `KeyRotationService.recordRotation()` (not `JpaActorIdentityBindingRepository`): `KeyRotationEntry`
saves via `LedgerEntryRepository.save()` → `JpaLedgerEntryRepository.updateMerkleFrontier()` →
`frontierRepo.replace()`, so the frontier is written. `JpaActorIdentityBindingRepository.save()` never
calls `frontierRepo.replace()` (#144), so using it here would produce vacuously-passing assertions —
two empty frontiers are trivially independent.

Test: call `KeyRotationService.recordRotation()` for the same `actorId` under two different
`tenancyId` values. Both produce `subjectId = UUID.nameUUIDFromBytes(actorId)`. Assert that each
tenant's frontier is independently retrievable, non-empty, and reflects only that tenant's entry count.

**`JpaSequenceNumberIT` and `JpaSequenceNumberPgIT`:** update `nextSequenceNumber` calls to pass
`tenancyId`.

**`LedgerHealthJobIT` and `LedgerHealthJobPgIT`:** update `GapEventCapture.onGap()` to observe
`LedgerAnomalyDetected` (or split to two typed observers). Update assertions that access `LedgerGapDetected`
fields to use the new record accessors (`subjectId()`, `tenancyId()` on `LedgerSequenceGapDetected`;
`entityType()`, `domainCount()`, `ledgerCount()` on `LedgerReconciliationMismatchDetected`).
