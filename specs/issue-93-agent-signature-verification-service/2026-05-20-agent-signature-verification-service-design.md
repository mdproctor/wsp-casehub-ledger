# AgentSignatureVerificationService Extraction — Design Spec
**Issue:** casehubio/ledger#93  
**Date:** 2026-05-20  
**Status:** Approved

---

## Problem

`LedgerVerificationService` currently holds two unrelated concerns:

- **Merkle tree operations** — `treeRoot`, `inclusionProof`, `verify`
- **Agent signature operations** — `verifyAgentSignature`, plus private helpers `verifyCryptographic` and `compromisedEffectiveSince`

These concerns share a class by accident of history, not design. The Merkle methods are consumed by `LedgerComplianceReportService` and `LedgerRetentionJob`. The signature methods are consumed only by tests and the reactive counterpart. Mixing them forces any caller that needs only Merkle operations to also pull in `KeyRotationService` and `Event<AgentSignatureSuspectEvent>` as transitive CDI dependencies.

In addition, `ReactiveLedgerVerificationService` duplicates `verifyCryptographic` verbatim — flagged in its own Javadoc as "pending decomposition at casehubio/ledger#93".

---

## Design

### New: `AgentCryptographicVerifier` (package-private utility)

A static-only utility class in `io.casehub.ledger.runtime.service`, holding a single method:

```java
static VerificationResult verifyCryptographic(LedgerEntry entry)
```

Pure Java — no IO, no CDI, no reactive imports. Performs Ed25519 signature verification against the stored public key bytes using `LedgerMerkleTree.canonicalBytes(entry)`. Mirrors the `LedgerMerkleTree` pattern: a focused, testable utility that both blocking and reactive tiers can call without depending on each other.

### New: `AgentSignatureVerificationService` (`@ApplicationScoped`, blocking tier)

Extracted from `LedgerVerificationService`:

| Method | Visibility | Note |
|---|---|---|
| `verifyAgentSignature(UUID entryId)` | `public @Transactional` | Full signature verification pipeline |
| `compromisedEffectiveSince(actorId, keyRef, occurredAt)` | `private` | Key compromise window check |

Injects: `LedgerEntryRepository`, `KeyRotationService`, `Event<AgentSignatureSuspectEvent>`.  
Calls `AgentCryptographicVerifier.verifyCryptographic()` — no cryptographic logic duplicated here.

No `@Transactional` annotation on the class; only `verifyAgentSignature` is transactional (same as current `LedgerVerificationService`).

### Modified: `LedgerVerificationService` (Merkle only)

Drops `KeyRotationService`, `Event<AgentSignatureSuspectEvent>`, and all three signature-related methods. Retains `treeRoot`, `inclusionProof`, `verify` unchanged.

No changes to `LedgerComplianceReportService` or `LedgerRetentionJob` — both callers only ever called Merkle methods.

### Renamed: `ReactiveLedgerVerificationService` → `ReactiveAgentSignatureVerificationService`

All reactive logic stays identical. The sole change: `verifyCryptographic` body is replaced with a call to `AgentCryptographicVerifier.verifyCryptographic(entry)`, eliminating the duplication.

The rename makes the blocking/reactive pairing explicit:

| Blocking | Reactive |
|---|---|
| `LedgerVerificationService` | *(no reactive Merkle — Merkle is always blocking)* |
| `AgentSignatureVerificationService` | `ReactiveAgentSignatureVerificationService` |
| `KeyRotationService` | `ReactiveKeyRotationService` |

### Modified: `LedgerProcessor`

Replaces `ReactiveLedgerVerificationService.class.getName()` with `ReactiveAgentSignatureVerificationService.class.getName()` in the `ExcludedTypeBuildItem`. No other logic change.

---

## Test Changes

### `BlockingTierPurityTest`

Add two new structural checks for the new blocking bean:

- `agentSignatureVerificationService_hasNoUniMethods()`
- `agentSignatureVerificationService_doesNotInjectReactiveSpi()`

Existing `LedgerVerificationService` purity checks remain — they enforce that the Merkle bean never acquires signature baggage in the future.

### `LedgerVerificationServiceIT`

Merkle tests (`treeRoot_*`, `inclusionProof_*`, `verify_*`) stay in this file unchanged.  
Signature tests (`verifyAgentSignature_*`) move to new `AgentSignatureVerificationServiceIT`, injecting `AgentSignatureVerificationService`.

### `AgentSigningIT`

- Injects `AgentSignatureVerificationService` for the three `verifyAgentSignature` call sites.
- Retains `LedgerVerificationService` for the `verify(sub)` call in `signedEntry_merkleChainStillValid`.

### `ReactiveLedgerVerificationServiceIT` → `ReactiveAgentSignatureVerificationServiceIT`

Rename only; injects `ReactiveAgentSignatureVerificationService`.

### `SuspectEventIT`

Updates both injections to the new names.

### `KeyRotationIT`

Updates `LedgerVerificationService` injection to `AgentSignatureVerificationService`.

---

## Javadoc Updates

| File | Change |
|---|---|
| `AgentSignatureSuspectEvent` | `@link` references updated to `AgentSignatureVerificationService` and `ReactiveAgentSignatureVerificationService` |
| `KeyRotationService` | `@link` in `compromisedWindows` Javadoc updated |
| `ReactiveKeyRotationService` | `@link` in `compromisedWindowsAsync` Javadoc updated |
| `LedgerVerificationService` | No change needed — class Javadoc already accurate |

---

## Invariants Preserved

- PP-20260519-f2e160: blocking tier carries no reactive dependencies.
- PP-20260519-39a9a5: `Reactive*Service` naming, `ExcludedTypeBuildItem` gating, direct reactive SPI injection.
- `BlockingTierPurityTest` structural coverage extended to cover the new blocking bean.
- No Flyway migrations required — this is a service-layer refactor only.
- All 450 existing tests must pass; test logic is redistributed, not changed.

---

## Out of Scope

Nothing deferred — scope is fully contained within this issue.
