# Fix: LedgerEntry.tenancyId Field Shadowing and @CrossTenant CDI Robustness

**Date:** 2026-06-09
**Issues:** casehubio/ledger#131, casehubio/ledger#132
**Branch:** issue-131-tenancy-field-shadowing

---

## Problem

Two bugs introduced by #127 (multi-tenancy):

**#131 — Field shadowing breaks JOINED subclass persistence.**

`LedgerEntry` (base) declares `public String tenancyId` mapped to `ledger_entry.tenancy_id`. Downstream JOINED subclasses (`CaseLedgerEntry`, `WorkerDecisionEntry` in casehub-engine) redeclare their own `tenancyId` field mapped to join table columns (`case_ledger_entry.tenancy_id`, `worker_decision_entry.tenancy_id`). Java field shadowing means two separate JVM fields exist on each subclass instance.

**Failure mechanism (Mode B — enhancement-redirected write):** Hibernate bytecode enhancement instruments `LedgerEntry` with intercepted field accessors (`$$_hibernate_write_tenancyId()`). When `JpaLedgerEntryRepository.save()` does `entry.tenancyId = tenancyId` (compile-time type `LedgerEntry`), enhancement rewrites the PUTFIELD bytecode to call the instrumented writer. At runtime, virtual dispatch calls `CaseLedgerEntry.$$_hibernate_write_tenancyId()` which writes the subclass field. The raw base-class field is never written by any Java code path. At `em.persist()` time, Hibernate reads raw fields via reflection (bypassing enhancement) — `LedgerEntry.tenancyId` is null — `NOT NULL` constraint violation on `ledger_entry.tenancy_id`.

The same redirection applies to reads: `((LedgerEntry) cle).tenancyId` returns the subclass value through enhancement, masking the null base field from normal Java code. Only reflection reveals the discrepancy.

**Root cause — multi-repo coordination gap:** Engine#299 (2026-05-31) added `tenancyId` to `CaseLedgerEntry` and `WorkerDecisionEntry` because the base class didn't have it. Ledger#127 (2026-06-09) added `tenancyId` to `LedgerEntry` — the correct platform-level location. Neither change was wrong in isolation. The failure is that #127 didn't coordinate removal of the consumer-level fields, and no automated guard prevented the shadowing.

**Timeline:** The bug is latent. Engine's capture code already calls the two-arg `save(entry, tenancyId)` API. The NOT NULL violation fires when engine picks up the ledger#127 SNAPSHOT (which adds `tenancy_id NOT NULL` to `ledger_entry` via V1000). The `InMemoryLedgerEntryRepository` has the same enhancement-redirected write at line 86 — no NOT NULL violation (no `em.persist()`), but the raw base field is null, which could affect code reading `LedgerEntry.tenancyId` via reflection (serializers, archive records).

**#132 — @CrossTenant beans break downstream consumer @QuarkusTest contexts.** `CrossTenantProducer` injects four unqualified repository beans and re-produces them with `@CrossTenant` qualifier. This creates a fragile CDI chain — if any dependency is unsatisfied in a downstream consumer's test context, every `@CrossTenant` injection point fails with `UnsatisfiedResolutionException`.

---

## Design

### #131: Remove Shadowing Fields and Denormalized Columns

**Principle:** `tenancyId` belongs on `LedgerEntry` only. Subclasses inherit it through Java field inheritance. Queries filter on `ledger_entry.tenancy_id` through the JOINED inheritance join. The `tenancy_id` column on join tables is denormalized redundancy that creates write-skew risk. Removal is the only path — denormalization adds no query benefit (JOINED inheritance always joins the base table) and is the source of the shadowing bug.

**Changes in casehub-ledger (this repo):**

1. Update `ledger-subclass-extension` protocol — add rule: subclass entities must NOT redeclare fields that exist on `LedgerEntry`.
2. Add build-time field-shadowing guard in `LedgerProcessor` (see Part F below).

**Internal subclasses verified clean:** `PlainLedgerEntry`, `KeyRotationEntry`, `ActorIdentityBindingEntry` — none declare `tenancyId`. No shadowing issue within ledger itself.

**Changes in casehub-engine (downstream — engine#460 already filed, fix in this branch since engine repo is available):**

1. `CaseLedgerEntry.java` — delete the `tenancyId` field declaration and its `@Column` annotation. Remove the `@Index(name = "idx_case_ledger_entry_tenancy_id", columnList = "tenancy_id")` from the `@Table` annotation.
2. `WorkerDecisionEntry.java` — same: delete field, `@Column`, and `@Index(name = "idx_worker_decision_entry_tenancy_id", columnList = "tenancy_id")` from the `@Table` annotation.
3. `V2000__case_ledger_entry.sql` — remove `tenancy_id` column, its `DEFAULT '__system__'`, and `CREATE INDEX idx_case_ledger_entry_tenancy_id`. No production deployments — rewrite in place.
4. `V2001__worker_decision_entry.sql` — same: remove `tenancy_id` column, `DEFAULT`, and `CREATE INDEX idx_worker_decision_entry_tenancy_id`.
5. `CaseLedgerEventCapture.java` — remove the redundant `entry.tenancyId = event.tenancyId()` assignment (line 72). The repository's `save(entry, tenancyId)` is the sole writer of `tenancyId` per the documented contract and `tenancy-repository-pattern` protocol. The capture code assignment is both redundant (save() overwrites it) and — before this fix — harmful (with shadowing, it sets the subclass field while save() also writes only to the subclass field via enhancement, leaving the base field null).
6. `WorkerDecisionEventCapture.java` — same: remove `entry.tenancyId = event.tenancyId()` (line 81).

After removal, `cle.tenancyId` resolves to the inherited `LedgerEntry.tenancyId`. The `save(entry, tenancyId)` method sets one field. Hibernate persists one column on the base table. JOINED inheritance join provides tenant filtering for all subclass queries.

**Follow-up (not in scope):** `CaseLedgerEntryRepository.findByCaseId()`, `findLatestByCaseId()`, and `findWorkerDecisionsByCaseId()` don't filter by `tenancyId`. This is a pre-existing design gap — engine#459 (SPI propagation) already tracks it.

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

**Part D — Build-time @CrossTenant scope validation in LedgerProcessor:**

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

**Part F — Build-time field-shadowing guard in LedgerProcessor:**

Add a `@BuildStep` in the deployment module that scans all registered `LedgerEntry` subclasses (via Jandex `CombinedIndexBuildItem`) for fields that shadow base class fields. For each subclass, compare `getDeclaredFields()` names against `LedgerEntry.getDeclaredFields()` names. Any collision produces a deployment validation error:

> "CaseLedgerEntry.tenancyId shadows LedgerEntry.tenancyId — remove the subclass field."

Zero runtime cost. Catches the problem permanently, across all consumers, regardless of which repo adds the field first. This is the automated guard that prevents the #131 root cause from recurring.

---

## Downstream Impact

| Repo | Issue | What changes |
|------|-------|--------------|
| casehub-engine | #460 (field shadowing), #459 (SPI propagation) | Remove `tenancyId` field from `CaseLedgerEntry`, `WorkerDecisionEntry`. Remove redundant `entry.tenancyId` assignment from capture code. Rewrite V2000, V2001. |
| casehub-work | #260 (SPI propagation) | No field shadowing (work doesn't have a LedgerEntry subclass with tenancyId). @CrossTenant changes are ledger-internal. |
| casehub-qhorus | #263 (SPI propagation) | Same — no field shadowing. @CrossTenant changes are ledger-internal. |
| devtown | — | Remove `tenancyId` field from `MergeDecisionLedgerEntry` if it exists. |

---

## Protocol Updates

**`ledger-subclass-extension.md`** — add to checklist:
- [ ] Subclass entity does NOT redeclare any field that exists on `LedgerEntry` — Java field shadowing causes Hibernate persist failures with JOINED inheritance. Enforced at build time by `LedgerProcessor`.

---

## Verification

- All 788 existing ledger tests pass
- In-memory tests verify tenancyId is correctly set on entries persisted through `InMemoryLedgerEntryRepository` (base field no longer null after shadowing removal)
- Build step detects field shadowing in LedgerEntry subclasses — fails the build with a clear message
- Build step validates no `@RequestScoped` bean injects `@CrossTenant`
- `@CrossTenant`-qualified injection resolves without a producer
- Unqualified injection of a cross-tenant repo fails at startup (CDI enforces qualifier)
- Engine capture code no longer redundantly sets `entry.tenancyId` — repository is the sole writer
