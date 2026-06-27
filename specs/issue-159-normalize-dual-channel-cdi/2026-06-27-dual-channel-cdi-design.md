# Normalize CDI Event Producers to Dual-Channel Firing

**Issue:** casehubio/ledger#159
**Date:** 2026-06-27
**Status:** Design

## Problem

CDI 4.x has two disjoint event delivery channels: `fire()` → `@Observes` and
`fireAsync()` → `@ObservesAsync`. Neither delivers to the other. A producer that
fires one channel silently drops observers on the other — no error, no warning.

Ledger has 11 distinct event types across 10 producer classes (11 producer
methods, 14 distinct fire operations). Only one event type
(`TrustScoreComputedAt`) currently fires both channels. The rest are
single-channel, leaving latent bugs:

- `ReactiveKeyRotationService.fireAsync()` silently skips two sync
  cache-invalidation observers (`IdentityCacheInvalidator`,
  `ActorIdentityValidationEnricher`). This is a real bug — caches are not
  invalidated when keys are rotated via the reactive path.
  (`AbstractCachingAgentSigner.onKeyRotated()` has no `@Observes` — intentionally
  omitted to avoid Arc ambiguity; any future consumer subclass following the
  documented delegation pattern would also be affected.)
- Any future `@ObservesAsync` observer of `AttestationRecordedEvent`,
  `TrustScoreActorUpdatedEvent`, `LedgerSequenceGapDetected`,
  `LedgerReconciliationMismatchDetected`, `TrustScoreFullPayload`,
  or `TrustScoreDeltaPayload` would be silently dropped.
- Any future `@Observes` observer of `AgentIdentityValidatedEvent` or
  `AgentIdentityViolationEvent` would be silently dropped.

## Platform alignment

PLATFORM.md documents four distinct CDI event dispatch strategies:

| Pattern | Sync | Async | Use case |
|---|---|---|---|
| Routing decisions | `fire()` | — | Control flow |
| Ledger capture | — | `fireAsync()` | Audit trail |
| CloudEvent envelope | — | `fireAsync()` | Cross-cutting event bus |
| WorkItem lifecycle | `fire()` | `fireAsync()` | Dual-channel |

casehubio/parent#302 evaluated whether dual-channel should be universal;
conclusion: repo-specific, not an adoption mandate. The criterion:
adopt when `TransactionPhase.AFTER_SUCCESS` observers or transactional sync
observers exist. Ledger qualifies — `IncrementalTrustUpdateObserver` and
`ComputedTrustScoreSource` both use `@Observes(during = AFTER_SUCCESS)`. This
is the same rationale casehub-work used for its normalisation (work#273).

## Design

### Pattern

Every producer fires `fire()` first, then `fireAsync()`:

```java
event.fire(payload);
event.fireAsync(payload)
        .exceptionally(ex -> { log.debugf(ex, "async observer failed"); return null; });
```

**Ordering:** `fire()` first so that if a sync observer throws, `fireAsync()` is
skipped — don't dispatch async for a failed sync. In a `@Transactional` context
this also prevents async dispatch for events whose transaction will roll back.
The same ordering applies outside `@Transactional` contexts (in-memory
repositories, reactive pipelines) for general causal correctness.

**Error handling:**
- `fire()`: follows the producer's existing convention. Repositories let it
  propagate; `TrustScoreRoutingPublisher` wraps in try/catch.
- `fireAsync()`: always fire-and-forget with `.exceptionally()` at debug level.

**No abstraction.** The call sites vary in error handling, transaction context,
and reactive vs blocking context. A generic wrapper would either ignore these
variations or become overparameterized. The 2–3 line inline pattern is explicit
and easy to review. (See Approach evaluation in the brainstorm.)

### Changes

#### Group 1 — Add `fireAsync()` alongside existing `fire()`

| Producer | File | Event type |
|---|---|---|
| `JpaLedgerEntryRepository.saveAttestation()` | `runtime/.../repository/jpa/JpaLedgerEntryRepository.java` | `AttestationRecordedEvent` |
| `InMemoryLedgerEntryRepository.saveAttestation()` | `persistence-memory/.../memory/InMemoryLedgerEntryRepository.java` | `AttestationRecordedEvent` |
| `KeyRotationService.recordRotation()` | `runtime/.../service/KeyRotationService.java` | `AgentKeyRotatedEvent` |
| `AgentSignatureVerificationService.verifyAgentSignature()` | `runtime/.../service/AgentSignatureVerificationService.java` | `AgentSignatureSuspectEvent` |
| `IncrementalTrustUpdateObserver.onAttestationRecorded()` | `runtime/.../service/IncrementalTrustUpdateObserver.java` | `TrustScoreActorUpdatedEvent` |
| `LedgerHealthJob.checkSequenceGaps()` | `runtime/.../service/LedgerHealthJob.java` | `LedgerAnomalyDetected` |
| `LedgerHealthJob.checkReconciliation()` | `runtime/.../service/LedgerHealthJob.java` | `LedgerAnomalyDetected` |

#### Group 2 — Add `fire()` alongside existing `fireAsync()`

| Producer | File | Event type |
|---|---|---|
| `ReactiveKeyRotationService.recordRotationAsync()` | `runtime/.../service/ReactiveKeyRotationService.java` | `AgentKeyRotatedEvent` |
| `ReactiveAgentSignatureVerificationService.verifyAgentSignatureAsync()` | `runtime/.../service/ReactiveAgentSignatureVerificationService.java` | `AgentSignatureSuspectEvent` |
| `ActorIdentityValidationEnricher.fireEvent()` | `runtime/.../service/identity/ActorIdentityValidationEnricher.java` | `AgentIdentityValidatedEvent`, `AgentIdentityViolationEvent` |

**Reactive path safety (verified):** All sync observers of events fired from
reactive pipelines are non-blocking ConcurrentHashMap operations:
- `IdentityCacheInvalidator.onKeyRotated()` → `ConcurrentHashMap.remove()`
- `ActorIdentityValidationEnricher.onKeyRotated()` → `ConcurrentHashMap.remove()`

(`AbstractCachingAgentSigner.onKeyRotated()` has no `@Observes` — it is a manual
delegation point for subclasses. No production subclass exists; `ConfiguredAgentSigner`
implements `AgentSigner` directly. Not a CDI observer.)

**Identity enricher safety (verified):**
- Enricher runs from `save()`, not `@PrePersist` (LedgerTraceListener is only a guard)
- Recursion naturally broken: `ActorDIDEnricher` (priority 40) skips
  `ActorIdentityBindingEntry` via instanceof guard → `actorDid` stays null →
  `ActorIdentityValidationEnricher.enrich()` returns immediately at line 78
- Double try/catch isolation: enricher's own catch + pipeline's per-enricher catch

**Identity events contract:** `AgentIdentityValidatedEvent` and
`AgentIdentityViolationEvent` fire from within the enricher pipeline, which
executes inside `LedgerEntryRepository.save()`. Sync observers of these events
run inside the save call of another `LedgerEntry`. Observers must be
non-blocking and must not call `LedgerEntryRepository.save()` synchronously.
For persistence, use `@ObservesAsync` with `@Transactional(REQUIRES_NEW)`, as
`ActorIdentityBindingObserver` does.

#### Group 3 — Complete dual-channel in `TrustScoreRoutingPublisher`

Add `fireAsync()` for `TrustScoreFullPayload` and `TrustScoreDeltaPayload` inside
their existing `if (hasXxxObservers)` guards, matching the `TrustScoreComputedAt`
pattern already in the same class.

Observer detection stays — it optimises payload construction (delta computation),
not just dispatch. `BeanManager.resolveObserverMethods()` returns both sync and
async observers, so the guards correctly trigger when either kind exists.

### Test strategy

**Existing captures to verify newly-added channel:**
- `AgentKeyRotatedEventCapture` — already has `@Observes` + `@ObservesAsync`.
  Assert the newly-added channel fires.
- `AgentSuspectEventCapture` — same.

**New captures needed:**
- `AttestationRecordedEventCapture` — `@Observes` + `@ObservesAsync` with
  `CompletableFuture` for async synchronisation.
- `TrustScoreActorUpdatedEventCapture` — same pattern.
- `LedgerAnomalyDetectedCapture` — same pattern.
- `IdentityEventCapture` — covers sync channel for `AgentIdentityValidatedEvent`
  and `AgentIdentityViolationEvent`.

**`TrustScoreRoutingPublisher`:** existing unit test uses mocked `Event<T>`. Add
`verify(fullEvent).fireAsync(any())` and `verify(deltaEvent).fireAsync(any())`.

### In-memory / JPA dispatch asymmetry

`InMemoryLedgerEntryRepository` has no JTA transaction. After this change:

- `@ObservesAsync AttestationRecordedEvent` observers fire on **both** JPA and
  in-memory paths (new behaviour from `fireAsync()`).
- `@Observes(during = AFTER_SUCCESS) AttestationRecordedEvent` observers fire
  **only** on the JPA path. On the in-memory path, `fire()` dispatches but
  `AFTER_SUCCESS` observers silently do not fire when no JTA transaction is active
  (GE-20260501-884024).

This is acceptable: the in-memory path is for tests and lightweight deployments
where trust score computation is not meaningful. But the asymmetry should be
visible to future developers adding `AttestationRecordedEvent` observers.

### Events NOT changed

`TrustScoreComputedAt` in `TrustScoreRoutingPublisher` — already fires both
channels. No change needed.

## Coherence review

- **PLATFORM.md:** parent#302's adoption criterion (AFTER_SUCCESS observers) is
  met. Ledger uses the same rationale as casehub-work's normalisation
  (work#273). ✅
- **Protocols:** No protocol applies to CDI event dispatch. Checked
  `ledger-entry-named-query`, `ledger-subclass-repo-readonly`,
  `per-subject-table-tenancy`. ✅
- **Garden:** GE-20260423-daef97 (fire() doesn't reach @ObservesAsync) confirms
  the root cause. GE-20260512-0fe012 (fireAsync() dispatches before commit)
  confirms the ordering rationale. GE-20260501-884024 (AFTER_SUCCESS silent
  no-fire without JTA) is documented in the asymmetry section above. ✅
- **No deferred concerns.** The scope is exhaustive — every single-channel
  producer in the repo is covered.
