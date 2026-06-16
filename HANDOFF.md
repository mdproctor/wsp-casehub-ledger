# Session Handoff — 2026-06-16

## Branch closed: issue-143-noop-trust-score-repo

Three repository fixes shipped and merged to `casehubio/ledger` main:

**#143 — NoOpActorTrustScoreRepository**
- `JpaActorTrustScoreRepository` is now `@Alternative`
- New `NoOpActorTrustScoreRepository @DefaultBean` satisfies CDI injection when no datasource or in-memory alternative is present
- Five test profiles updated; `%noop-test` deliberately omits it

**#141 — Dialect detection in LedgerSequenceAllocator**
- Replaced `@ConfigProperty(db-kind)` with lazy JDBC metadata detection via `em.unwrap(Session.class).doReturningWork(conn -> conn.getMetaData().getDatabaseProductName()...)`
- Fixes named-datasource consumers silently getting the H2 MERGE branch against PostgreSQL

**#139 — Merkle frontier + sequence tenancy**
- Root cause: `KeyRotationEntry` and `ActorIdentityBindingEntry` derive `subjectId = nameUUIDFromBytes(actorId)` — two tenants sharing the same `actorId` produced identical subjectIds, causing:
  - Frontier row collision (tenants overwriting each other's Merkle state)
  - Cryptographically incorrect inclusion proofs (`k = sequenceNumber - 1` used wrong leaf index)
  - False-positive gap alerts in `LedgerHealthJob`
- Added `tenancy_id` to `ledger_merkle_frontier` (PK+index) and `ledger_subject_sequence` (composite PK)
- Both JPA call sites updated; both InMemory implementations use composite key records
- `LedgerGapDetected` + `GapType` replaced by sealed `LedgerAnomalyDetected` hierarchy

**Pre-existing bug also fixed:**
- `consumer-compat-test` had `quarkus:build` goal which fails because test-scoped deps (casehub-platform, quarkus-jdbc-h2) are invisible at package phase; removed the build goal

## Current state

- `casehubio/ledger` main: 3 commits ahead of previous state (2 squashed feat+doc commits + 1 consumer-compat-test fix)
- All 818 H2 tests pass; 19 PostgreSQL tests pass (Podman/Testcontainers)
- Issues #143, #141, #139 closed on GitHub

## Open issues filed this session

- **#144**: `JpaActorIdentityBindingRepository.save()` never calls `frontierRepo.replace()` — Merkle chain is not maintained for `ActorIdentityBindingEntry` subjects; `LedgerVerificationService` proofs are broken for those subjects
- **#145**: `latestBindingFor(actorId)` and `bindingHistoryFor(actorId)` have no `tenancyId` filter — cross-tenant read leak for shared `actorId` scenarios

## Protocol added

- `PP-20260616-05dc6a`: Per-subject storage tables must include `tenancy_id` in their key — see `docs/protocols/casehub/per-subject-table-tenancy.md`

## Garden entries added

- `GE-20260616-aaf9b8`: H2 MERGE USING alias-as-type parsing gotcha (`? AS tid` → `Unknown data type: "TID"`)
- `GE-20260616-e04575`: JDBC dialect detection via `DatabaseMetaData` for named-datasource compatibility
- REVISE on `GE-20260520-c0e5b4`: added Podman machine-stopped discovery step (`docker context ls` + `podman machine start`)

## Blog entry

- `2026-06-16-mdp01-the-tenant-whose-key-was-always-the-same.md` — published to mdproctor.github.io
