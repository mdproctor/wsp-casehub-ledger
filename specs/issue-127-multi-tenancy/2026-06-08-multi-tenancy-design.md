# Multi-Tenancy for casehub-ledger

**Issue:** casehubio/ledger#127
**Date:** 2026-06-08
**Status:** Design — pending cross-repo coherence review with work and qhorus specs

---

## Problem

All `LedgerEntry` reads and writes are cross-tenant. Every consumer (`MessageLedgerEntry`, `CaseLedgerEntry`, `WorkItemLedgerEntry`) inherits this gap. In a multi-tenant deployment this means audit trail cross-contamination between tenants.

Zero `tenancyId` or `tenancy_id` references exist anywhere in ledger source or Flyway migrations.

## Governing Protocols

| Protocol | Rule |
|---|---|
| PP-20260520-439daf | Tenancy filtering is always unconditional. No `if (multiTenantEnabled)` guards. |
| PP-20260520-e6a5f0 | `tenancyId` binding belongs in data access classes only. Services and REST endpoints never call `currentPrincipal.tenancyId()` directly. |
| PP-20260607-69eba2 | `tenancyId` is server-side infrastructure — never exposed in client-facing APIs. |
| PP-20260511-ledger-spi | When a `LedgerEntryRepository` SPI method is added/changed, update all downstream implementations. |
| PP-20260519-3f2ea2 | Reactive repository SPIs ship no bundled JPA impl — provide a `@DefaultBean` blocking test shim. |

## Approach

Explicit `String tenancyId` parameter on every tenant-scoped SPI method. Cross-tenant methods split to separate `CrossTenant*Repository` interfaces produced via a `@CrossTenant` CDI qualifier. Full parity with casehub-engine's tenancy architecture (#299, #405, #406).

The explicit parameter approach was chosen over `CurrentPrincipal` injection in repositories because ledger has the same CDI scope problem as engine: `TrustScoreJob`, `LedgerHealthJob`, `IncrementalTrustUpdateObserver` (`@ObservesAsync`), and `LedgerRetentionJob` all run outside request scope. `CurrentPrincipal.tenancyId()` would throw `ContextNotActiveException` at runtime.

---

## 1. Entity Model

**`LedgerEntry`** gains a `tenancyId` field — `String`, non-null, default `TenancyConstants.DEFAULT_TENANT_ID`. JPA column `tenancy_id` on the base `ledger_entry` table. All subclasses (`KeyRotationEntry`, `ActorIdentityBindingEntry`, `PlainLedgerEntry`, and consumer-owned subclasses) inherit automatically via JOINED inheritance — no subclass migration needed.

**Tables that do NOT gain tenancyId:**

| Table | Reason |
|---|---|
| `actor_trust_score` | Trust scores are global — an actor's reputation is the same regardless of which tenant asks |
| `ledger_attestation` | FK to `ledger_entry` — inherits tenant scoping through the join |
| `ledger_merkle_frontier` | Scoped by `subjectId` which is already unique per tenant |
| `ledger_subject_sequence` | Scoped by `subjectId` |

## 2. SPI Interface Split

### Tenant-scoped: `LedgerEntryRepository` (all methods gain `String tenancyId`)

| Method | Signature after change |
|---|---|
| `save` | `save(LedgerEntry entry, String tenancyId)` |
| `findBySubjectId` | `findBySubjectId(UUID subjectId, String tenancyId)` |
| `findBySubjectIdAndTimeRange` | `findBySubjectIdAndTimeRange(UUID, Instant, Instant, String tenancyId)` |
| `findLatestBySubjectId` | `findLatestBySubjectId(UUID subjectId, String tenancyId)` |
| `findEntryById` | `findEntryById(UUID id, String tenancyId)` |
| `findAttestationsByEntryId` | `findAttestationsByEntryId(UUID, String tenancyId)` |
| `saveAttestation` | `saveAttestation(LedgerAttestation, String tenancyId)` |
| `findByActorId` | `findByActorId(String actorId, Instant, Instant, String tenancyId)` |
| `findByActorRole` | `findByActorRole(String actorRole, Instant, Instant, String tenancyId)` |
| `findCausedBy` | `findCausedBy(UUID entryId, String tenancyId)` |
| `findAttestationsByEntryIdAndCapabilityTag` | `...(UUID, String capTag, String tenancyId)` |
| `findAttestationsByEntryIdGlobal` | `...(UUID entryId, String tenancyId)` |
| `findAttestationsByAttestorIdAndCapabilityTag` | `...(String attestorId, String capTag, String tenancyId)` |

### Cross-tenant: `CrossTenantLedgerEntryRepository` (new interface, no tenancyId)

| Method | Why cross-tenant |
|---|---|
| `listAll()` | Retention sweep, bulk export |
| `findAllEvents()` | Trust score computation — all actors, all tenants |
| `findEventsByActorId(String)` | Trust recomputation — actor's full history |
| `findByTimeRange(Instant, Instant)` | Compliance bulk export, retention |
| `findAttestationsForEntries(Set<UUID>)` | Trust computation — bulk attestation fetch |

### Reactive parity

`ReactiveLedgerEntryRepository` — same split. Tenant-scoped methods gain `String tenancyId`. Cross-tenant methods move to `CrossTenantReactiveLedgerEntryRepository`.

### Repositories that are entirely cross-tenant (no split needed)

| Repository | Reason |
|---|---|
| `ActorTrustScoreRepository` | Trust scores are global — always cross-tenant. Produced via `@CrossTenant`. |
| `KeyRotationRepository` / `ReactiveKeyRotationRepository` | Rotations are per-actor, not per-tenant. Global. |
| `ActorIdentityBindingRepository` | Identity bindings are per-actor. Global. |

### `LedgerMerkleFrontierRepository`

Add `String tenancyId` to both methods (`findBySubjectId`, `replace`) for defence in depth. The subjectId is already unique per tenant, but the tenancyId parameter ensures a guessed UUID from another tenant is rejected.

## 3. CDI Infrastructure

**`@CrossTenant`** — CDI qualifier in `io.casehub.ledger.runtime.qualifier`. Convention-based marker, same pattern as engine's `io.casehub.engine.common.qualifier.CrossTenant`.

**`@LedgerSystem`** — CDI qualifier for the system-level `CurrentPrincipal`. Analogous to engine's `@EngineSystem`.

**`LedgerSystemCurrentPrincipal`** — `@ApplicationScoped @LedgerSystem`, implements `CurrentPrincipal`. Returns `actorId() = "system"`, `tenancyId() = DEFAULT_TENANT_ID`, `isCrossTenantAdmin() = true`. Not `@DefaultBean`. Interim — delete when platform ships a platform-level system principal.

**`CrossTenantProducer`** — `@ApplicationScoped`. Injects `@LedgerSystem LedgerSystemCurrentPrincipal`. Produces `@CrossTenant`-qualified beans:

| Produced bean | Always/conditional |
|---|---|
| `CrossTenantLedgerEntryRepository` | Always |
| `CrossTenantReactiveLedgerEntryRepository` | When `casehub.ledger.reactive.enabled=true` |
| `ActorTrustScoreRepository` | Always |
| `KeyRotationRepository` | Always |
| `ReactiveKeyRotationRepository` | When reactive enabled |
| `ActorIdentityBindingRepository` | Always |

Each `@Produces` method asserts `systemPrincipal.isCrossTenantAdmin()`.

**Injection sites that switch to `@CrossTenant`:**

| Class | What it injects |
|---|---|
| `TrustScoreJob` | `@CrossTenant CrossTenantLedgerEntryRepository`, `@CrossTenant ActorTrustScoreRepository` |
| `PerActorTrustComputer` | Same as TrustScoreJob |
| `IncrementalTrustUpdateObserver` | `@CrossTenant CrossTenantLedgerEntryRepository`, `@CrossTenant ActorTrustScoreRepository` |
| `LedgerRetentionJob` | `@CrossTenant CrossTenantLedgerEntryRepository` |
| `TrustExportService` | `@CrossTenant ActorTrustScoreRepository` |
| `TrustBootstrapService` | `@CrossTenant ActorTrustScoreRepository` |
| `LedgerHealthJob` | Uses raw JPQL via EntityManager — no repo injection change, but cross-tenant by nature |

## 4. Write Path

`LedgerEntryRepository.save(LedgerEntry entry, String tenancyId)` — the JPA implementation stamps `entry.tenancyId = tenancyId` before persist. tenancyId comes from the caller, which got it from `CurrentPrincipal.tenancyId()` at the HTTP boundary.

The `LedgerEntryEnricher` pipeline (`TraceIdEnricher`, `AgentSignatureEnricher`, etc.) runs at `@PrePersist` via `LedgerTraceListener`. Enrichers do not need tenancyId — they read OTel span, signing keys, DID context. tenancyId is already stamped on the entity before the pipeline fires.

`saveAttestation(LedgerAttestation, String tenancyId)` — the attestation FKs to a ledger entry. The JPA implementation validates that the referenced `ledgerEntryId` belongs to the given tenancyId before persisting — rejects the save if the entry is in a different tenant. Defence in depth.

`LedgerSequenceAllocator.nextSequenceNumber(UUID subjectId)` — no tenancyId needed. Sequences are per-subject, subjects are unique per tenant.

Services that call `save()` (`KeyRotationService`, `LedgerProvExportService`) pass tenancyId from their own callers. The service layer never touches `CurrentPrincipal` directly.

## 5. Read Path

Every tenant-scoped JPQL query adds `AND e.tenancyId = :tenancyId`. Unconditional — no conditional checks per PP-20260520-439daf.

Cross-tenant queries (in `JpaCrossTenantLedgerEntryRepository`) have no tenancyId filter — same WHERE clauses as today.

**Service-level tenancyId threading:**

| Service | Change |
|---|---|
| `LedgerVerificationService` | Add `String tenancyId` to `treeRoot()`, `inclusionProof()`, `verify()` — pass through to frontier and entry repo calls |
| `AgentSignatureVerificationService` | No change — pure computation on an already-loaded entry |
| `TrustGateService` | No change — reads cross-tenant trust scores |
| `TrustExportService` | No change — exports are cross-tenant |
| `LedgerComplianceReportService` | Add `String tenancyId` to `reportForActor()`, `reportForSubject()` — tenant-scoped |

## 6. InMemory Implementations (persistence-memory)

`InMemoryLedgerEntryRepository` — `save()` stamps `entry.tenancyId = tenancyId`. All tenant-scoped find methods filter by `tenancyId.equals(entry.tenancyId)`. `allEntries()` (internal accessor) stays unfiltered for cross-tenant delegates.

`InMemoryCrossTenantLedgerEntryRepository` — new class, `@Alternative @Priority(1)`. Delegates to `InMemoryLedgerEntryRepository.allEntries()` with no filter. Implements `CrossTenantLedgerEntryRepository`.

`InMemoryActorTrustScoreRepository`, `InMemoryKeyRotationRepository`, `InMemoryActorIdentityBindingRepository` — no changes. Already cross-tenant.

`InMemoryLedgerMerkleFrontierRepository` — add `String tenancyId` to method signatures for SPI parity. In-memory impl can ignore the parameter (frontier is keyed by subjectId).

Reactive in-memory variants — delegate to blocking counterparts. Signature changes propagate mechanically.

## 7. Flyway Migration

Modify V1000 in place (no existing installations per CLAUDE.md schema convention).

Add to `ledger_entry` table definition:
```sql
tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce',
```

Add index:
```sql
CREATE INDEX idx_ledger_entry_tenancy ON ledger_entry (tenancy_id);
```

No other migration files change. Consumer JOINED subclass tables inherit `tenancy_id` from the base table — no consumer migration needed.

## 8. Test Strategy

**`TenancyIsolationIT`** — core contract test:
- Save entries for two different tenancyIds, verify each tenant's `findBySubjectId` returns only its own
- Verify `CrossTenantLedgerEntryRepository.listAll()` returns entries from both tenants
- Verify `findEntryById` with valid UUID but wrong tenancyId returns empty
- Same pattern for attestations

**Existing test updates:**
- `TrustScoreJobIT` — operates through `@CrossTenant` injection, continues to see all entries
- `LedgerHealthJobIT` / `LedgerHealthJobPgIT` — cross-tenant aggregation unchanged
- `LedgerVerificationServiceIT` — gains tenancyId on method calls
- `InMemoryLedgerEntryRepositoryTest` — updated for new signatures
- `JpaSequenceNumberPgIT` — no change (sequence allocation is per-subject)

**New tests:**
- `InMemoryCrossTenantLedgerEntryRepositoryTest`
- `CrossTenantProducerTest` — verify `isCrossTenantAdmin()` guard

## 9. Downstream Propagation

Breaking SPI change — tracked as separate issues per PP-20260511-ledger-spi:

| Repo | Issue | Scope |
|---|---|---|
| casehub-work | work#260 | Update `JpaWorkItemLedgerEntryRepository` + produce own tenancy spec |
| casehub-qhorus | qhorus#263 | Update `MessageLedgerEntryRepository` + reactive variant + produce own tenancy spec |
| casehub-engine | engine#459 | Update 4 NoOp test classes + `WorkerDecisionEventCapture` |

Implementation blocked on cross-repo coherence review: ledger, work, and qhorus specs must be reviewed together before any implementation begins.

## 10. Out of Scope

- **RLS (Row Level Security)** — application-level WHERE clause filtering is sufficient. RLS is a defence-in-depth layer for a future issue.
- **Downstream consumer updates** — separate issues filed above.
- **REST endpoint tenancy** — ledger has no REST endpoints.
- **`TenantAwareRepository` base class** — engine uses Panache reactive with `withTenantTransaction()`. Ledger uses raw EntityManager — different pattern, no shared base class.
- **`LedgerErasureService` tenancy** — GDPR erasure is cross-tenant by nature (data-subject right). Stays as-is.
