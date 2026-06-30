# Design: ActorIdentityBindingRepository — Merkle frontier and tenancy fixes

**Branch:** `issue-144-identity-binding-merkle-and-tenancy`  
**Issues:** #144 (Merkle frontier gap), #145 (cross-tenant reads)  
**Date:** 2026-06-16

---

## Problem

`ActorIdentityBindingRepository` diverged from `KeyRotationRepository` by having its own
`save()` method. That method duplicated parts of the `LedgerEntryRepository.save()` pipeline
incompletely, producing two bugs:

- **#144:** `JpaActorIdentityBindingRepository.save()` computes a leaf hash but never calls
  `frontierRepo.replace()`. `LedgerVerificationService` sees an empty frontier for any subject
  whose chain consists entirely of `ActorIdentityBindingEntry` rows. `InMemoryActorIdentityBindingRepository.save()`
  has the same gap and additionally never assigns `sequenceNumber` or computes `digest` — all
  in-memory binding entries have `sequenceNumber=0` and `digest=null`.

- **#145:** `latestBindingFor(actorId)` and `bindingHistoryFor(actorId)` have no `tenancyId`
  parameter. Two tenants sharing the same `actorId` (e.g. `claude:reviewer@v1`) cross-read
  each other's binding history. Violates PP-20260616-05dc6a.

---

## Design

### Principle

`KeyRotationRepository` has no `save()` — its Javadoc is explicit: *"persisted via
`LedgerEntryRepository#save()`."* `KeyRotationService.recordRotation()` calls
`ledgerRepo.save(entry, tenancyId)`. `InMemoryKeyRotationRepository` delegates reads to
`InMemoryLedgerEntryRepository.allEntries()` filtered by type. `ActorIdentityBindingRepository`
is structurally identical. The dedicated `save()` is removed. All implementations converge on
the single save pipeline.

---

## The event loop — verified, bounded, guarded

Routing `ActorIdentityBindingEntry` through `LedgerEntryRepository.save()` runs the enricher
pipeline. Two relevant enrichers:

**`ActorDIDEnricher` (Priority 40):** skips if `entry.actorDid != null`; binding entries have
`actorDid = null` at construction, so the enricher runs and resolves the actor's current DID
from the provider.

**`ActorIdentityValidationEnricher` (Priority 50):** triggers only on cache miss. On miss,
fires `AgentIdentityValidatedEvent` (async). On hit, sets `pendingIdentityStatus` but does not
fire an event.

Without guards, the chain is:

1. Normal entry saved → `ActorDIDEnricher` sets `actorDid` → `ActorIdentityValidationEnricher`
   cache miss → fires `AgentIdentityValidatedEvent` (async, separate thread, `REQUIRES_NEW`).
2. Observer creates binding entry → routes through `LedgerEntryRepository.save()` → enrichers
   run → `ActorDIDEnricher` sets `actorDid` on binding entry → `ActorIdentityValidationEnricher`
   runs → cache hit from step 1 → no event fired.

The loop is **cache-bounded**: iteration 2 terminates because the cache populated in step 1
prevents the event from firing again. However, the loop is **not bounded under cache
invalidation**: key rotation invalidates the cache. If a binding entry is saved while the cache
is empty (between invalidation and the next normal-entry save repopulating it), a new cache miss
fires a new event, which saves another binding entry, which could fire again. The loop is also
**async-across-transactions**: each iteration is a separate thread with a new `REQUIRES_NEW`
transaction, not recursive within one transaction.

**Two guards in `ActorDIDEnricher`:**

```java
if (entry.actorId == null || entry.actorDid != null) return;  // existing
if (entry instanceof ActorIdentityBindingEntry) return;        // new
```

**Why `ActorDIDEnricher`, not just `ActorIdentityValidationEnricher`:** If `ActorDIDEnricher`
runs on a binding entry, it sets `actorDid` from the provider's cache at async commit time. Due
to async timing, the actor may have rotated their DID between the validation event firing and the
binding entry being committed. The binding entry would then have `boundDid` (the DID that was
actually validated) differing from `actorDid` (the provider's current DID). Both fields are on
the same audit record with conflicting values. `actorDid` is not in `canonicalBytes()`, so there
is no hash impact — but the audit integrity concern stands. The binding entry is self-describing:
`boundDid` is the authoritative DID for this record. Guarding `ActorDIDEnricher` prevents this
inconsistency.

Consequence: with `actorDid = null` on the binding entry, `ActorIdentityValidationEnricher`'s
existing null guard (`if (entry.actorDid == null) return;`) also short-circuits. Both guards
are consistent: binding entries carry only what they validated, nothing more. The
`ActorDIDEnricher` guard makes the loop unconditionally impossible, not merely cache-bounded —
binding entries are short-circuited before `ActorIdentityValidationEnricher` is reached
regardless of cache state. Future cache invalidation paths (e.g. DID-method-change global
invalidation) cannot reopen the loop.

**`LedgerIdentityEnforcementListener`:** checks `if (entry.pendingIdentityStatus == null) return`.
Since the guards leave `pendingIdentityStatus = null` on binding entries, enforcement exits
immediately. No blocking of binding writes in ENFORCE mode.

**`LedgerTraceListener`:** checks `if (hashChain.enabled() && entry.digest == null) throw`.
Previously this guard was satisfied by `JpaActorIdentityBindingRepository.save()`, which computed
digest before calling `em.persist()`. After our change, the guard is satisfied by the full
`LedgerEntryRepository.save()` pipeline. The invariant is now enforced more broadly: any direct
`em.persist(bindingEntry)` without going through `LedgerEntryRepository.save()` (and without a
precomputed digest) will throw.

---

## Changes

### SPI: `ActorIdentityBindingRepository`

Remove `save()`. Add `tenancyId` to both read methods:

```java
public interface ActorIdentityBindingRepository {
    Optional<ActorIdentityBindingEntry> latestBindingFor(String actorId, String tenancyId);
    List<ActorIdentityBindingEntry>    bindingHistoryFor(String actorId, String tenancyId);
}
```

### Enricher: `ActorDIDEnricher`

Add `instanceof ActorIdentityBindingEntry` guard as the second check after the existing null
check. Prevents DID resolution for binding entries, which in turn prevents
`ActorIdentityValidationEnricher` from seeing a non-null `actorDid` and maintains audit
consistency between `boundDid` and the (now absent) `actorDid`.

### Observer: `ActorIdentityBindingObserver`

- Add injection: `LedgerEntryRepository ledgerRepo`
- Remove injection: `ActorIdentityBindingRepository repository`
- In `persistBinding()`: replace `repository.save(entry)` with `ledgerRepo.save(entry, tenancyId)`
- Remove `entry.tenancyId = tenancyId` — `LedgerEntryRepository.save()` sets it
- `@Transactional(REQUIRES_NEW)` unchanged — the `@Transactional` (REQUIRED) on `ledgerRepo.save()` joins the existing transaction

### Enricher: `ActorIdentityValidationEnricher`

No change needed. The existing `if (entry.actorDid == null) return;` guard short-circuits for
binding entries because `ActorDIDEnricher` no longer sets `actorDid` on them. The guard in
`ActorDIDEnricher` is the correct and complete fix; the validation enricher guard is implicit.

### Named queries: `ActorIdentityBindingEntry`

Switch ordering from `occurredAt` to `sequenceNumber`. After routing through the full save
pipeline, every binding entry has a monotonically increasing, unique-per-(subjectId, tenancyId)
sequence number. Using `occurredAt` is unreliable: two binding entries saved within the same
millisecond (realistic in test suites that flush the cache between back-to-back saves) have the
same `occurredAt` and non-deterministic ordering. `sequenceNumber` is the canonical ordering
key used by every other `findBySubjectId` query in the codebase.

```java
@NamedQuery(name = "ActorIdentityBindingEntry.findLatestByActorId",
    query = "SELECT e FROM ActorIdentityBindingEntry e "
          + "WHERE e.actorId = :actorId AND e.tenancyId = :tenancyId "
          + "ORDER BY e.sequenceNumber DESC")
@NamedQuery(name = "ActorIdentityBindingEntry.findHistoryByActorId",
    query = "SELECT e FROM ActorIdentityBindingEntry e "
          + "WHERE e.actorId = :actorId AND e.tenancyId = :tenancyId "
          + "ORDER BY e.sequenceNumber ASC")
```

### JPA: `JpaActorIdentityBindingRepository`

Remove `save()` and its supporting injections (`LedgerSequenceAllocator`; `LedgerConfig` and
`LedgerMerkleTree` imports used only by save). Update both read methods to pass `tenancyId`.

### In-memory: `InMemoryActorIdentityBindingRepository`

Remove `save()` and the `CopyOnWriteArrayList<> store` field. Inject `InMemoryLedgerEntryRepository blocking`
(package-internal). Delegate reads to `blocking.allEntries()` filtered by `instanceof ActorIdentityBindingEntry`
and `tenancyId`. Order by `sequenceNumber` (not `occurredAt`).

```java
// latestBindingFor
return blocking.allEntries().stream()
    .filter(e -> e instanceof ActorIdentityBindingEntry)
    .map(e -> (ActorIdentityBindingEntry) e)
    .filter(e -> actorId.equals(e.actorId) && tenancyId.equals(e.tenancyId))
    .max(Comparator.comparingInt(e -> e.sequenceNumber));

// bindingHistoryFor
... .sorted(Comparator.comparingInt(e -> e.sequenceNumber)).toList();
```

**Concurrency note:** The previous `CopyOnWriteArrayList.add()` in `save()` had no sequence
protection — all binding entries had `sequenceNumber=0`. After our change, binding entries go
through `InMemoryLedgerEntryRepository.save()`, which holds a per-subject lock during sequence
allocation, hash computation, and frontier update. Concurrent async binding saves for the same
actor now produce correct, monotonically increasing sequence numbers and a valid Merkle chain.
This is a meaningful correctness improvement.

### No-op: `NoOpActorIdentityBindingRepository`

Remove `save()`. Update read method signatures to include `tenancyId`. Both still return empty.

---

## Tests

### `NoOpActorIdentityBindingRepositoryIT` — repurpose, do not delete

The noop-test profile (`JpaLedgerEntryRepository` + `JpaLedgerMerkleFrontierRepository`, without
`JpaActorIdentityBindingRepository`) is still valid — but its assertion inverts.

**Before our change:** observer calls `NoOpActorIdentityBindingRepository.save()` → write
suppressed → DB count = 0.

**After our change:** observer calls `LedgerEntryRepository.save()` (which IS
`JpaLedgerEntryRepository` in this profile) → binding entry written to `actor_identity_binding`
→ DB count > 0.

The repurposed test validates the key architectural consequence of this design: writes do NOT
require `JpaActorIdentityBindingRepository` in `selected-alternatives`. The write path goes
through `LedgerEntryRepository` exclusively. The read path — `bindingRepo.latestBindingFor(actorId, DEFAULT_TENANT_ID)`
— still returns empty because `NoOpActorIdentityBindingRepository` is active.

Two assertions:
1. DB count > 0 after the async observer fires (binding entry persisted via `JpaLedgerEntryRepository`)
2. `bindingRepo.latestBindingFor(actorId, DEFAULT_TENANT_ID)` returns empty (no-op read is active)

This also validates a consumer consequence: to read binding entries via the SPI, consumers must
add `JpaActorIdentityBindingRepository` to `selected-alternatives`. Write correctness does not
depend on it.

### Existing: `ActorIdentityBindingEntryIT`

Update `readLatestBinding(actorId)` and `readBindingHistory(actorId)` to pass `DEFAULT_TENANT_ID`.

### New: cross-tenant isolation test

Verify that `latestBindingFor` and `bindingHistoryFor` with tenant A's ID do not return binding
entries written under tenant B for the same `actorId`. Uses the in-memory path (no datasource
required).

### New: Merkle proof coverage test

Save one or more binding entries via the observer pipeline (triggered by a normal ledger entry
save with DID configured). The binding entry's `subjectId` is derived deterministically as
`UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))` — distinct from the triggering entry's
`subjectId` (which is `UUID.randomUUID()`). All assertions must use the binding entry's
`subjectId`, not the triggering entry's. Read it back from the persisted binding entry or
compute it directly.

Assert:
- `entry.sequenceNumber > 0`
- `entry.digest != null`
- `verificationService.treeRoot(bindingSubjectId, tenancyId)` does not throw (throws
  `IllegalStateException` when the frontier is empty — i.e. when no binding entries were
  persisted for that subject)
- `verificationService.verify(bindingSubjectId, tenancyId)` returns `true`

`verify(UUID subjectId, String tenancyId)` takes two parameters. There is no `sequenceNumber`
overload. It recomputes all leaf hashes for the subject and compares the recomputed root against
the stored frontier — exactly the right assertion for #144.

This is the #144 regression guard: it fails against the current code and passes after the fix.

---

## What is not changed

- No Flyway migrations — `tenancyId` is already on `ledger_entry` (V1000); named-query changes
  are JPQL-only
- `KeyRotationRepository` — same structural tenancy gap (#146 filed, needs separate decision on
  cross-tenant SUSPECT detection semantics before fixing)
- `NoOpLedgerEntryRepository` — binding entries saved in no-op ledger contexts remain no-ops
  (observer calls `ledgerRepo.save()` which is the no-op; no binding entries written, no pipeline
  runs)

---

## Deferred: #146

`KeyRotationRepository.findByActorId()` and `findCompromisedByActorIdAndKeyRef()` have no
`tenancyId` parameter. The fix requires a design decision: SUSPECT detection may be intentionally
actor-scoped (a compromised key should be flagged across all tenants). Filed as #146 before
leaving brainstorming.