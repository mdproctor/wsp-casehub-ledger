# @DefaultBean No-Op Repositories — Eliminate Consumer exclude-types

**Issue:** casehubio/ledger#138
**Branch:** issue-138-noop-defaultbean-ledger-repo
**Date:** 2026-06-14

---

## Problem

Two repository SPIs lack the `@DefaultBean` no-op that the PLATFORM.md mandate requires for
all Store SPIs. This forces consumers to add `quarkus.arc.exclude-types` workarounds.

### Gap 1 — LedgerEntryRepository

`JpaLedgerEntryRepository` is `@Alternative` — it does not activate unless explicitly
selected via `quarkus.arc.selected-alternatives`. No `@DefaultBean` no-op exists to fill
the default slot. CDI augmentation fails in consumers that depend on `casehub-ledger` without
selecting a JPA or in-memory alternative (e.g., engine test modules that run without a
datasource and without `casehub-ledger-memory` on the classpath).

### Gap 2 — ActorIdentityBindingRepository (root cause of identity service exclusions)

`JpaActorIdentityBindingRepository` is currently plain `@ApplicationScoped` — it is always
active and always wins the CDI resolution. At runtime, `ActorIdentityBindingObserver`
observes CDI events and calls `repository.save()`, which attempts DB writes. Consumers without
a datasource or with a DB the observer should not write to suppress this with a bulk exclusion:

- `casehub-work`: `quarkus.arc.exclude-types=io.casehub.ledger.runtime.service.identity.**`
- `casehub-engine`: individual exclusions of `IdentityCacheInvalidator`, `ActorDIDEnricher`,
  `ActorIdentityValidationEnricher`, `ActorIdentityBindingObserver`, `AgentIdentityVerificationService`

Three of those five (`ActorDIDEnricher`, `ActorIdentityValidationEnricher`,
`IdentityCacheInvalidator`) have all their dependencies satisfied via no-ops in
`casehub-platform-identity`. They are excluded only as collateral because the bulk exclusion
catches the whole package.

**The root cause is not CDI augmentation failure** — augmentation succeeds because the JPA
impl is always active. The problem is that making the JPA impl inactive-by-default (via
`@Alternative`) is not currently possible without losing the only implementation, since no
`@DefaultBean` no-op exists to take the default slot.

---

## Fix

### Part A — `NoOpLedgerEntryRepository @DefaultBean` (Gap 1)

Add to `runtime/src/main/java/io/casehub/ledger/runtime/repository/`:

```java
@DefaultBean
@ApplicationScoped
public class NoOpLedgerEntryRepository implements LedgerEntryRepository {
    // All read methods → empty (List.of() / Optional.empty())
    // save() → returns entry unchanged; no sequenceNumber, no enrichment, no side effects
    // saveAttestation() → returns attestation unchanged
}
```

### Part B — `NoOpActorIdentityBindingRepository @DefaultBean` + `JpaActorIdentityBindingRepository @Alternative` (Gap 2)

Two changes are required together — neither is effective without the other:

**1. Add `NoOpActorIdentityBindingRepository` to `runtime/src/main/java/io/casehub/ledger/runtime/repository/`:**

```java
@DefaultBean
@ApplicationScoped
public class NoOpActorIdentityBindingRepository implements ActorIdentityBindingRepository {
    // latestBindingFor() → Optional.empty()
    // bindingHistoryFor() → List.of()
    // save() → returns entry unchanged; no sequence, no hash, no persist
}
```

**2. Change `JpaActorIdentityBindingRepository` from `@ApplicationScoped` to `@Alternative @ApplicationScoped`.**

Without this change, `NoOpActorIdentityBindingRepository @DefaultBean` is dead code:
`@DefaultBean` has the lowest CDI priority and is shadowed by any plain `@ApplicationScoped`
bean for the same type. Making the JPA impl `@Alternative` removes it from the default slot
so the no-op can fill it.

**Consequence — `selected-alternatives` updates:** `JpaActorIdentityBindingRepository` must
now be added to every `selected-alternatives` list in
`runtime/src/test/resources/application.properties`. There are five such lists: the default
block (lines 4–6), plus four profile-specific blocks (`federation-import-test`,
`federation-bootstrap-test`, `trust-score-bootstrap-test`, `scim-did-provider-test`). This is
a mechanical update — all existing runtime tests run with a datasource and need the JPA impl
active.

Profiles without their own `selected-alternatives` override — including `identity-binding-test`
and `identity-enforce-test` — inherit the default block and pick up
`JpaActorIdentityBindingRepository` automatically. `ActorIdentityBindingEntryIT` and
`LedgerIdentityEnforcementIT` continue to work unchanged because they run under those inherited
profiles and the JPA impl is present via the default block.

---

## Established Precedent

`NoOpLedgerMerkleFrontierRepository` already implements this exact pattern and lives in the
same target package (`runtime/src/main/java/io/casehub/ledger/runtime/repository/`). Its
Javadoc documents the activation priority ladder; the two new no-ops must follow the same
structure:

```java
/**
 * Activation priority (lowest to highest):
 * 1. This {@code @DefaultBean} — active when nothing else is present
 * 2. {@code JpaXxxRepository @Alternative} — activate via quarkus.arc.selected-alternatives
 * 3. {@code InMemoryXxxRepository @Alternative @Priority(1)} — active when
 *    casehub-ledger-memory is on the classpath
 */
```

`@DefaultBean` must be imported from `io.quarkus.arc`, not `jakarta.enterprise.inject`.

The frontier no-op's `replace()` is void — no return value to document. The new no-ops have
non-void `save()` methods, requiring explicit Javadoc on what is omitted:

```java
/**
 * No-op save — returns the entry unchanged. Does NOT:
 * assign sequenceNumber, set tenancyId, compute digest, run the enricher pipeline,
 * or call {@code EntityManager.persist()}. This bean exists solely to satisfy the
 * CDI injection point in deployments where the ActorIdentityBindingRepository SPI
 * should be inactive. Use a JPA or in-memory implementation for any context where
 * binding entries must actually be persisted.
 */
```

This convention is introduced here; the frontier no-op's `replace()` does not need it
because a void no-op's intent is self-evident.

---

## CDI Priority Ordering After Fix

For `LedgerEntryRepository` (Gap 1 — already has `@Alternative` JPA impl):

| Implementation | Annotation | When active |
|---|---|---|
| `NoOpLedgerEntryRepository` | `@DefaultBean` | Default — no datasource, no alternatives selected |
| `JpaLedgerEntryRepository` | `@Alternative` | Via `quarkus.arc.selected-alternatives` |
| `InMemoryLedgerEntryRepository` | `@Alternative @Priority(1)` | `casehub-ledger-memory` on classpath |

For `ActorIdentityBindingRepository` (Gap 2 — JPA impl changes from plain to `@Alternative`):

| Implementation | Annotation | When active |
|---|---|---|
| `NoOpActorIdentityBindingRepository` | `@DefaultBean` | Default — no datasource |
| `JpaActorIdentityBindingRepository` | `@Alternative` *(changed)* | Via `quarkus.arc.selected-alternatives` |
| `InMemoryActorIdentityBindingRepository` | `@Alternative @Priority(1)` | `casehub-ledger-memory` on classpath |

---

## Scope Boundary

Other repositories do not need no-ops in this branch. The rule: a no-op is needed when a
JPA implementation is `@Alternative` (leaving CDI unsatisfied without explicit selection) OR
when an always-active JPA impl causes runtime failures in contexts where DB writes must not
occur. Neither condition applies to the remaining repos:

- `LedgerMerkleFrontierRepository` — `NoOpLedgerMerkleFrontierRepository @DefaultBean`
  already exists in `runtime`. **Already fixed.**
- `CrossTenantLedgerEntryRepository` — `JpaCrossTenantLedgerEntryRepository` is plain
  `@ApplicationScoped` (Javadoc: "not `@Alternative` because cross-tenant operations are a
  system-level concern"). Always satisfies CDI. Its read-only queries are safe to run without
  side effects in any context.
- `KeyRotationRepository` — `JpaKeyRotationRepository` is `@DefaultBean @ApplicationScoped`.
  Is itself the default (read-only queries). Displaced by `InMemoryKeyRotationRepository
  @Alternative @Priority(1)` when `persistence-memory` is present. No no-op needed.
- `ActorTrustScoreRepository` — `JpaActorTrustScoreRepository` is plain `@ApplicationScoped`.
  Always satisfies CDI. Not in any consumer exclude-types list.
- Reactive counterparts — gated by `casehub.ledger.reactive.enabled=false` and excluded via
  `ExcludedTypeBuildItem` in `LedgerProcessor` when reactive is disabled. No no-op needed.

---

## Tests

**A new test profile is required** — it cannot be an extension of `sequence-number-test`.
`sequence-number-test` has no profile-specific `selected-alternatives` and inherits the
default block. After the fix the default block includes `JpaActorIdentityBindingRepository`,
so any test running under that profile would get the JPA impl, not the no-op.

The new `noop-test` profile must set its own `selected-alternatives` that deliberately omits
`JpaActorIdentityBindingRepository`:

```properties
%noop-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:nooptestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%noop-test.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository
```

`JpaActorIdentityBindingRepository` is deliberately absent. Result:
- `LedgerEntryRepository` → JPA (selected) — real saves drive the enricher and observer
- `LedgerMerkleFrontierRepository` → JPA (selected)
- `ActorIdentityBindingRepository` → `NoOpActorIdentityBindingRepository @DefaultBean`
  (JPA not selected; `persistence-memory` not on the runtime test classpath)

`quarkus.arc.selected-alternatives` replaces the default — it does not merge. This is the
mechanism by which the no-op test achieves its stated condition.

Under those conditions, verify:

1. `ActorIdentityBindingRepository` resolves to `NoOpActorIdentityBindingRepository` (injected
   into `ActorIdentityBindingObserver`).
2. **End-to-end observer no-op** (mirrors `ActorIdentityBindingEntryIT` but asserts the
   inverse): save a `TestEntry` with a non-null `actorDid` (any `did:` scheme string, e.g.
   `"did:web:noop-test.example.com"`) via `QuarkusTransaction.requiringNew()`.

   No DID document registration is required. `InjectableTestDIDResolver @Alternative
   @Priority(1)` auto-activates in all `@QuarkusTest` runs and ships with an empty registry
   — `resolve()` returns `Optional.empty()` for any unregistered DID. The enricher returns
   `DID_UNRESOLVABLE`, fires `AgentIdentityViolationEvent`, and
   `ActorIdentityBindingObserver.onViolation()` calls `repository.save()`. Because the
   repository resolves to the no-op, `em.persist()` is never called.

   Assert via a direct native query against the live datasource — **not** via
   `bindingRepo.latestBindingFor()`. The no-op's `latestBindingFor()` always returns
   `Optional.empty()` synchronously without reading the database, so an Awaitility assertion
   on it is trivially true regardless of whether the observer ever fired.

   The correct assertion injects `@LedgerPersistenceUnit EntityManager em` and uses:

   ```java
   await().during(Duration.ofMillis(500)).atMost(Duration.ofSeconds(2))
       .untilAsserted(() -> {
           Long count = (Long) em.createNativeQuery(
               "SELECT COUNT(*) FROM actor_identity_binding WHERE actor_id = :id")
               .setParameter("id", actorId)
               .getSingleResult();
           assertThat(count).isZero();
       });
   ```

   The `during(500ms)` window gives the async observer time to have fired before the
   assertion is evaluated. If the observer ran and the JPA impl were somehow active, a row
   would appear and the assertion would fail. The datasource is live (H2), the table exists
   (Flyway ran), but the no-op prevented any `em.persist()` call — the count stays 0. This
   is the distinction between "observer ran, no-op prevented the write" and "observer never
   ran, trivially empty table": only the direct database count can make it.

   The test class does not need `@Inject InjectableTestDIDResolver` or any `register()` call.
3. No `quarkus.arc.exclude-types` entry for `io.casehub.ledger.runtime.service.identity.**`
   is present in the `noop-test` profile — augmentation and runtime both succeed.

**`NoOpLedgerEntryRepository` unit test (separate from the noop-test profile):**

`NoOpLedgerEntryRepository` has no CDI dependencies and does not need a Quarkus container.
A standalone JUnit 5 unit test instantiates it directly, calls `save()` and asserts the
entry is returned unchanged, and calls each read method and asserts the result is empty.
This keeps the no-op's behaviour verification fast and container-free.

Note: `LedgerEntryRepository` resolves to `JpaLedgerEntryRepository` in the `noop-test`
profile because it is explicitly selected. Testing `NoOpLedgerEntryRepository` in that
profile would contradict the profile's configuration — the no-op is not active there by
design.

---

## Platform Coherence

Implements the PLATFORM.md Store SPI mandate: "Store SPIs always get a no-op `@DefaultBean`
in the mock module." For `casehub-ledger`, `runtime` is the correct module for no-ops that
must be available on the classpath without additional test dependencies.

No schema changes, no Flyway migrations, no new API surface. `JpaActorIdentityBindingRepository`
becoming `@Alternative` is a breaking change within the ledger repo only — all callers in
`runtime/src/test/resources/application.properties` must add it to `selected-alternatives`,
which is a mechanical update captured above.

Consumer repos (`casehub-work`, `casehub-engine`) can remove their `exclude-types` entries
after this ships — that is a follow-up in each consumer repo, not in scope here.