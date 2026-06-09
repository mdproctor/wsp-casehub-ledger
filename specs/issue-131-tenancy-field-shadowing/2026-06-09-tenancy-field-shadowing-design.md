# Fix: LedgerEntry.tenancyId Field Shadowing and @CrossTenant CDI Robustness

**Date:** 2026-06-09
**Issues:** casehubio/ledger#131, casehubio/ledger#132
**Branch:** issue-131-tenancy-field-shadowing

---

## Problem

Two bugs introduced by #127 (multi-tenancy):

**#131 — Field shadowing breaks JOINED subclass persistence.** `LedgerEntry` (base) declares `public String tenancyId` mapped to `ledger_entry.tenancy_id`. Downstream JOINED subclasses (`CaseLedgerEntry`, `WorkerDecisionEntry` in casehub-engine) redeclare their own `tenancyId` field mapped to join table columns (`case_ledger_entry.tenancy_id`, `worker_decision_entry.tenancy_id`). Java field shadowing means two separate fields exist on each subclass. Hibernate reads raw fields via reflection at `em.persist()` — the base field is null when only the subclass field was set, causing a `NOT NULL` constraint violation on `ledger_entry.tenancy_id`.

**#132 — @CrossTenant beans break downstream consumer @QuarkusTest contexts.** `CrossTenantProducer` injects four unqualified repository beans and re-produces them with `@CrossTenant` qualifier. This creates a fragile CDI chain — if any dependency is unsatisfied in a downstream consumer's test context, every `@CrossTenant` injection point fails with `UnsatisfiedResolutionException`.

---

## Design

### #131: Remove Shadowing Fields and Denormalized Columns

**Principle:** `tenancyId` belongs on `LedgerEntry` only. Subclasses inherit it through Java field inheritance. Queries filter on `ledger_entry.tenancy_id` through the JOINED inheritance join. The `tenancy_id` column on join tables is denormalized redundancy — it served a purpose before #127 (it was the only place tenancyId existed) but is now dead weight.

**Changes in casehub-ledger (this repo):**

1. Update `ledger-subclass-extension` protocol — add rule: subclass entities must NOT redeclare fields that exist on `LedgerEntry`.

**Changes in casehub-engine (downstream — engine#460 already filed, fix in this branch if engine repo is available):**

1. `CaseLedgerEntry.java` — delete the `tenancyId` field declaration and its `@Column` annotation. Remove `@Table(indexes = {@Index(... columnList = "tenancy_id")})`.
2. `WorkerDecisionEntry.java` — same removal.
3. `V2000__case_ledger_entry.sql` — remove `tenancy_id` column, its `DEFAULT`, and `idx_case_ledger_entry_tenancy_id` index. No production deployments — rewrite in place.
4. `V2001__worker_decision_entry.sql` — same removal.

After removal, `cle.tenancyId` resolves to the inherited `LedgerEntry.tenancyId`. The `save(entry, tenancyId)` method sets one field. Hibernate persists one column on the base table. JOINED inheritance join provides tenant filtering for all subclass queries.

### #132: Move @CrossTenant to Implementations, Delete Producer

**Principle:** The `@CrossTenant` qualifier belongs on the bean implementation, not on a producer wrapper. Each cross-tenant repository implementation carries `@CrossTenant` directly. CDI resolves qualified beans without a producer chain.

**Part A — Add @CrossTenant to implementations:**

Runtime JPA implementations (all `@ApplicationScoped`):
- `JpaCrossTenantLedgerEntryRepository` — add `@CrossTenant`
- `JpaActorTrustScoreRepository` — add `@CrossTenant`
- `JpaKeyRotationRepository` — add `@CrossTenant`
- `JpaActorIdentityBindingRepository` — add `@CrossTenant`

Persistence-memory implementations (all `@Alternative @Priority(1)`):
- `InMemoryCrossTenantLedgerEntryRepository` — add `@CrossTenant`
- `InMemoryActorTrustScoreRepository` — add `@CrossTenant`
- `InMemoryKeyRotationRepository` — add `@CrossTenant`
- `InMemoryActorIdentityBindingRepository` — add `@CrossTenant`

Reactive cross-tenant implementations (build-gated via `casehub.ledger.reactive.enabled=true`):
- `InMemoryCrossTenantReactiveLedgerEntryRepository` — add `@CrossTenant`

Injection sites (`@Inject @CrossTenant ...`) are unchanged — they already use the qualifier.

**Part B — Delete CrossTenantProducer:**

Delete `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/CrossTenantProducer.java`. The four-dependency producer chain is eliminated.

**Part C — Move startup assertion to @Startup bean:**

Create `CrossTenantBootValidator` — `@Startup @ApplicationScoped`. Injects `@LedgerSystem LedgerSystemCurrentPrincipal`. Validates `isCrossTenantAdmin() == true` at boot. Independent of repository wiring.

**Part D — Build-time scope validation in LedgerProcessor:**

Add a `@BuildStep` in the deployment module that scans all `@CrossTenant` injection points. If the declaring bean is `@RequestScoped`, produce a deployment validation error. This makes `@CrossTenant` a compile-time-enforced architectural constraint: system-level beans only, never request-scoped.

**Part E — Update @CrossTenant javadoc:**

```java
/**
 * CDI qualifier for cross-tenant data access — system-level operations that
 * span all tenants (trust computation, retention, key rotation, federation).
 *
 * <p>Applied to both implementations and injection sites. Unqualified injection
 * of a cross-tenant repository fails at startup — the qualifier is mandatory.
 *
 * <p>All cross-tenant beans run without request context: {@code @Scheduled} jobs,
 * CDI async observers, or infrastructure services. None operate on behalf of a
 * specific tenant.
 *
 * <p>Build-time enforcement: {@code @RequestScoped} beans injecting
 * {@code @CrossTenant} produce a deployment error via {@code LedgerProcessor}.
 */
```

---

## Downstream Impact

| Repo | Issue | What changes |
|------|-------|--------------|
| casehub-engine | #460 (field shadowing), #459 (SPI propagation) | Remove `tenancyId` field from `CaseLedgerEntry`, `WorkerDecisionEntry`. Rewrite V2000, V2001. |
| casehub-work | #260 (SPI propagation) | No field shadowing (work doesn't have a LedgerEntry subclass with tenancyId). @CrossTenant changes are ledger-internal. |
| casehub-qhorus | #263 (SPI propagation) | Same — no field shadowing. @CrossTenant changes are ledger-internal. |
| devtown | — | Remove `tenancyId` field from `MergeDecisionLedgerEntry` if it exists. |

---

## Protocol Updates

**`ledger-subclass-extension.md`** — add to checklist:
- [ ] Subclass entity does NOT redeclare any field that exists on `LedgerEntry` — Java field shadowing causes Hibernate persist failures with JOINED inheritance.

---

## Verification

- All 788 existing tests pass (field shadowing fix is in ledger base — engine entity changes are downstream)
- Build step validates no `@RequestScoped` bean injects `@CrossTenant`
- `@CrossTenant`-qualified injection resolves without a producer
- Unqualified injection of a cross-tenant repo fails at startup (CDI enforces qualifier)
