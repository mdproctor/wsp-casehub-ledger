# Reactive Key Rotation Design

**Issue:** casehubio/ledger#86
**Epic:** epic-reactive-key-service
**Date:** 2026-05-19
**Protocol:** PP-20260517-15bf75 — ledger-sync-async-parity

---

## Problem

`KeyRotationService` provides only blocking methods. Protocol PP-20260517-15bf75 requires both
blocking and `Uni<T>` reactive variants for all ledger service methods.

`verifyAgentSignatureAsync` in `LedgerVerificationService` contains a known blocking bridge:

```java
// Blocking bridge — see #86 for reactive KeyRotationService
final Optional<Instant> effectiveSince =
        compromisedEffectiveSince(entry.actorId, entry.agentKeyRef, entry.occurredAt);
```

This bridge calls `keyRotationService.compromisedWindows()` (blocking) inside a reactive `Uni`
pipeline — a threading hazard on the Vert.x event loop.

---

## Approach

Option A — parallel SPI, matching the established `LedgerEntryRepository` /
`ReactiveLedgerEntryRepository` platform pattern. Both SPIs are persistence-agnostic contracts;
implementations are provider-specific.

---

## Architecture

### New components

| Component | Location | Purpose |
|-----------|----------|---------|
| `ReactiveKeyRotationRepository` | `runtime/repository/` | Reactive SPI — query-only, `Uni<List<>>` returns |
| `BlockingReactiveKeyRotationRepository` | test sources | Test shim wrapping blocking JPA impl |
| `ReactiveKeyRotationRepositoryIT` | test sources | Structural SPI compliance verification |

### Modified components

| Component | Change |
|-----------|--------|
| `KeyRotationService` | Gains `compromisedWindowsAsync`, `rotationHistoryAsync`, `recordRotationAsync`; two new injections |
| `LedgerVerificationService` | `verifyAgentSignatureAsync` blocking bridge replaced with reactive chain; new `compromisedEffectiveSinceAsync` private helper |

### Unchanged

- `KeyRotationRepository` (blocking SPI) — untouched
- `JpaKeyRotationRepository` — untouched
- `verifyAgentSignature` sync path — untouched
- Schema — no migrations

---

## `ReactiveKeyRotationRepository` SPI

Query-only, mirroring `KeyRotationRepository`. Saves route through
`ReactiveLedgerEntryRepository.save()` for Merkle chain inclusion — same discipline as the
blocking path routing through `LedgerEntryRepository.save()`.

```java
public interface ReactiveKeyRotationRepository {

    /** All rotation events for an actor, ordered by {@code occurredAt} ascending. */
    Uni<List<KeyRotationEntry>> findByActorId(String actorId);

    /** All COMPROMISED rotation events for a specific actor and keyRef, ordered by effectiveSince ascending. */
    Uni<List<KeyRotationEntry>> findCompromisedByActorIdAndKeyRef(String actorId, String keyRef);
}
```

No production JPA implementation is bundled in `casehub-ledger` — identical to how
`ReactiveLedgerEntryRepository` works. Consumers provide their own implementation:

| Provider | Blocking | Reactive |
|----------|----------|---------|
| In-memory / H2 (tests) | `JpaKeyRotationRepository` | `BlockingReactiveKeyRotationRepository` (test shim) |
| PostgreSQL (production) | `JpaKeyRotationRepository` | Consumer-provided Hibernate Reactive impl |
| MongoDB | Consumer `@Alternative` impl | Consumer-provided reactive Mongo impl |

The SPI is persistence-agnostic. `@NamedQuery` annotations on `KeyRotationEntry` are a JPA
implementation detail invisible to the SPI; MongoDB providers ignore them entirely.

---

## `KeyRotationService` — reactive additions

Two new CDI injections:

```java
@Inject ReactiveKeyRotationRepository reactiveRepository;
@Inject ReactiveLedgerEntryRepository reactiveLedgerRepo;
```

Three new methods:

**`compromisedWindowsAsync`** — mirrors `compromisedWindows` exactly:
```java
public Uni<List<CompromisedWindow>> compromisedWindowsAsync(String actorId, String keyRef) {
    return reactiveRepository.findCompromisedByActorIdAndKeyRef(actorId, keyRef)
            .map(entries -> entries.stream()
                    .map(e -> new CompromisedWindow(e.previousKeyRef, e.effectiveSince))
                    .toList());
}
```

**`rotationHistoryAsync`** — thin delegate:
```java
public Uni<List<KeyRotationEntry>> rotationHistoryAsync(String actorId) {
    return reactiveRepository.findByActorId(actorId);
}
```

**`recordRotationAsync`** — builds the same `KeyRotationEntry` as `recordRotation`, persists
through `reactiveLedgerRepo.save()`, uses `@ReactiveTransactional`:
```java
@ReactiveTransactional
public Uni<KeyRotationEntry> recordRotationAsync(
        String actorId, String previousKeyRef, String newKeyRef,
        KeyRotationReason reason, Instant effectiveSince) {
    final KeyRotationEntry entry = new KeyRotationEntry();
    // identical field assignments to recordRotation
    return reactiveLedgerRepo.save(entry).map(e -> (KeyRotationEntry) e);
}
```

---

## `LedgerVerificationService` — blocking bridge removal

A new private helper mirrors `compromisedEffectiveSince` for the reactive path:

```java
private Uni<Optional<Instant>> compromisedEffectiveSinceAsync(
        String actorId, String keyRef, Instant occurredAt) {
    return keyRotationService.compromisedWindowsAsync(actorId, keyRef)
            .map(windows -> windows.stream()
                    .filter(w -> occurredAt != null && !occurredAt.isBefore(w.effectiveSince()))
                    .map(CompromisedWindow::effectiveSince)
                    .findFirst());
}
```

`verifyAgentSignatureAsync` replaces the blocking bridge with a `.chain()` call:

```java
// Before — blocking bridge inside reactive pipeline:
final Optional<Instant> effectiveSince =
        compromisedEffectiveSince(entry.actorId, entry.agentKeyRef, entry.occurredAt);

// After — fully reactive:
return compromisedEffectiveSinceAsync(entry.actorId, entry.agentKeyRef, entry.occurredAt)
        .chain(effectiveSince -> {
            if (effectiveSince.isPresent()) {
                return Uni.createFrom().completionStage(
                        () -> suspectEvent.fireAsync(...))
                        .replaceWith(VerificationResult.SUSPECT);
            }
            return Uni.createFrom().item(VerificationResult.VALID);
        });
```

The `// Blocking bridge — see #86` comment is removed. The sync path
(`verifyAgentSignature` + `compromisedEffectiveSince`) is completely unchanged.

---

## Testing

### Structural — `ReactiveKeyRotationRepositoryIT` (no `@QuarkusTest`)

- All methods on `ReactiveKeyRotationRepository` return `Uni<T>`
- `ReactiveKeyRotationRepository` covers all methods on `KeyRotationRepository` by name

### Test shim — `BlockingReactiveKeyRotationRepository` (test sources only)

`@DefaultBean @ApplicationScoped`. Wraps `JpaKeyRotationRepository` with `Uni.createFrom().item()`
for both methods. Allows `@QuarkusTest` to resolve `ReactiveKeyRotationRepository` injections
without a Vert.x reactive datasource. Mirrors `BlockingReactiveLedgerEntryRepository`.

### `KeyRotationServiceIT` additions (`@QuarkusTest`)

Four new tests mirroring the existing blocking equivalents, using `.await().atMost(Duration.ofSeconds(5))`:

- `recordRotationAsync_persistsEntry`
- `rotationHistoryAsync_returnsAllEventsOrdered`
- `compromisedWindowsAsync_onlyReturnsCompromisedReason`
- `compromisedWindowsAsync_emptyWhenNoCompromiseRecord`

### `LedgerVerificationServiceIT` addition (`@QuarkusTest`)

- `verifyAgentSignatureAsync_suspectEntry_firesEventViaReactivePath` — seeds a signed entry,
  records a COMPROMISED rotation, calls `verifyAgentSignatureAsync`, asserts SUSPECT result
  and `AgentSuspectEventCapture` received the event.
