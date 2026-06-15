# Spec: Repository Fixes — NoOp Trust Score, Dialect Detection, Merkle Tenancy

**Branch:** `issue-143-noop-trust-score-repo`  
**Covers:** #143, #141, #139  
**Date:** 2026-06-15

---

## #143 — NoOpActorTrustScoreRepository + JpaActorTrustScoreRepository @Alternative

### Problem

`JpaActorTrustScoreRepository` is `@ApplicationScoped` (always active). `InMemoryActorTrustScoreRepository`
is `@Alternative @Priority(1)` (wins when `casehub-ledger-memory` is on the classpath). No `@DefaultBean`
no-op exists. Consumers without a datasource and without `casehub-ledger-memory` must add
`io.casehub.ledger.runtime.repository.jpa.JpaActorTrustScoreRepository` to `quarkus.arc.exclude-types`
— the fragile pattern eliminated for `LedgerEntryRepository` and `ActorIdentityBindingRepository` by #138.

### Design

Applies Pattern C from `alternative-extension-patterns.md`:

```
NoOpActorTrustScoreRepository     @DefaultBean            (active when nothing else present)
JpaActorTrustScoreRepository      @Alternative            (activate via selected-alternatives)
InMemoryActorTrustScoreRepository @Alternative @Priority(1) (active when ledger-memory on classpath)
```

#### New: `NoOpActorTrustScoreRepository`

`@DefaultBean @ApplicationScoped` in `runtime/.../repository/`.

- `findByActorId` → `Optional.empty()`
- `findCapabilityScore` → `Optional.empty()`
- `findDimensionScore` → `Optional.empty()`
- `findCapabilityDimension` → `Optional.empty()`
- `findCapabilityDimensions` → `List.of()`
- `findByActorIdAndScoreType` → `List.of()`
- `upsert(...)` → no-op

#### Changed: `JpaActorTrustScoreRepository`

Add `@Alternative` alongside existing `@ApplicationScoped`.

#### Changed: `application.properties`

Add `io.casehub.ledger.runtime.repository.jpa.JpaActorTrustScoreRepository` to:

| Block | Action |
|---|---|
| Default block (top-level `quarkus.arc.selected-alternatives`) | Add |
| `%federation-import-test` | Add |
| `%federation-bootstrap-test` | Add |
| `%trust-score-bootstrap-test` | Add |
| `%scim-did-provider-test` | Add |
| `%noop-test` | **Do NOT add** — deliberately tests the no-op path |

#### Test: `NoOpActorTrustScoreRepositoryTest`

Pure unit test (no container). Verifies all read methods return empty, `upsert` completes without exception.

---

## #141 — Dialect Detection for Named Datasource

### Problem

`LedgerSequenceAllocator` reads `quarkus.datasource.db-kind` to choose between PostgreSQL (`INSERT ON
CONFLICT`) and H2 (`MERGE`) SQL. This key is the *default* datasource config. When a consumer configures
`casehub.ledger.datasource=myds`, the db-kind lives at `quarkus.datasource."myds".db-kind` and the
injected value remains `"h2"`, causing the H2 MERGE branch to run against a PostgreSQL connection —
failing immediately with a SQL syntax error. (Not silent; tracked in #141, not #140 as the old comment
incorrectly stated.)

### Design

Replace the `@ConfigProperty` field with lazy dialect detection from the live JDBC connection.
`nextSequenceNumber()` is always called within a `@Transactional` method; `em.unwrap(Session.class)` is
therefore always valid when dialect detection runs.

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

`volatile` on the field provides visibility without synchronization; the database product name is
immutable at runtime, so the redundant work on first-call races is harmless.

#### Tests

No new test class. Existing `JpaSequenceNumberIT` (H2) exercises `isPostgresql() == false`;
`JpaSequenceNumberPgIT` (PostgreSQL/Testcontainers) exercises `isPostgresql() == true`. Both paths
already have full coverage.

---

## #139 — Merkle Frontier Tenancy

### Problem

`ledger_merkle_frontier` has no `tenancy_id` column. `JpaLedgerMerkleFrontierRepository` accepts
`tenancyId` in SPI method signatures but ignores it in the named queries and DML. `InMemoryLedgerMerkleFrontierRepository` keys the frontier map by `UUID subjectId` only.

In multi-tenant deployments, `KeyRotationEntry` and `ActorIdentityBindingEntry` derive `subjectId` via
`UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))`. Two tenants sharing the same `actorId` (e.g.
`claude:reviewer@v1`) produce identical `subjectId` UUIDs. Their `ledger_entry` rows are correctly
tenant-scoped (filtered by `tenancyId`). Their Merkle frontiers are not — both tenants write to the same
`ledger_merkle_frontier` rows, each save overwriting the other's frontier. Verification fails for both chains.

### Design

#### Schema (rewrite V1000 in place — no production installs)

`ledger_merkle_frontier` changes:

```sql
-- Add column:
tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce'

-- Change UNIQUE constraint:
-- Old: UNIQUE (subject_id, level)
-- New: UNIQUE (subject_id, tenancy_id, level)

-- Update index:
-- Old: CREATE INDEX idx_merkle_frontier_subject ON ledger_merkle_frontier (subject_id)
-- New: CREATE INDEX idx_merkle_frontier_subject ON ledger_merkle_frontier (subject_id, tenancy_id)
```

Default value `'278776f9-e1b0-46fb-9032-8bddebdcf9ce'` is `TenancyConstants.DEFAULT_TENANT_ID` — the
single-tenant sentinel. Single-tenant deployments work unchanged.

#### API model `LedgerMerkleFrontier` (in `api/`)

Add:
```java
@Column(name = "tenancy_id", nullable = false)
public String tenancyId;
```

#### Runtime model `LedgerMerkleFrontier` NamedQueries

```java
// findBySubjectId: add tenancyId filter
"SELECT f FROM LedgerMerkleFrontier f WHERE f.subjectId = :subjectId AND f.tenancyId = :tenancyId ORDER BY f.level ASC"

// deleteBySubjectAndLevel: add tenancyId filter
"DELETE FROM LedgerMerkleFrontier f WHERE f.subjectId = :subjectId AND f.level = :level AND f.tenancyId = :tenancyId"
```

#### `JpaLedgerMerkleFrontierRepository`

`findBySubjectId`: pass `:tenancyId` to named query.

`replace`:
- Bulk DELETE gets `AND f.tenancyId = :tenancyId`
- Per-node named DELETE gets `:tenancyId` parameter
- Set `node.tenancyId = tenancyId` on each node before `em.persist(node)`

`LedgerMerkleTree.append()` remains unchanged — it is a pure algorithm that sets `subjectId` on returned
nodes. `tenancyId` is set by `replace()` before persistence. Separation of concerns preserved.

#### `InMemoryLedgerMerkleFrontierRepository`

Add private composite key record:
```java
private record FrontierKey(UUID subjectId, String tenancyId) {}
```

Change `ConcurrentHashMap<UUID, ...>` to `ConcurrentHashMap<FrontierKey, ...>`. Update
`findBySubjectId`, `replace`, and `clear` accordingly.

#### Tests

**`InMemoryLedgerMerkleFrontierRepositoryTest`:** add test — two tenants with identical `subjectId`
produce independent frontiers; save by tenant A does not overwrite tenant B's frontier.

**New `MerkleFrontierTenancyIT`:** `@QuarkusTest` (H2) — two `LedgerEntry` saves for the same nameUUID-derived
`subjectId` but different `tenancyId` values; assert each frontier is independently retrievable and correct.
