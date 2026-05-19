# Reactive Service Tier Separation Design

**Issue:** casehubio/ledger#92
**Branch:** issue-92-optional-reactive-repo
**Date:** 2026-05-19
**Protocols:** PP-20260519-f2e160 (reactive-blocking-tier-separation), PP-20260519-39a9a5 (reactive-service-build-gating)

---

## Problem

`LedgerVerificationService` and `KeyRotationService` mix blocking and reactive methods in
a single `@ApplicationScoped` bean. The reactive methods require `ReactiveLedgerEntryRepository`
and `ReactiveKeyRotationRepository`, which are not present in JDBC-only consumers (casehub-aml,
casehub-clinical, casehub-devtown). Quarkus CDI augmentation validates all injection points at
build time — before tests or runtime config — causing build failures in those consumers.

The root cause is a code smell, not a CDI workaround problem. Mixing execution tiers in a
single bean violates PP-20260519-f2e160: service beans must not carry dependencies on
capabilities that are optional in consuming deployments.

`Instance<T>` and `@DefaultBean` no-op approaches were evaluated and rejected:
- `Instance<T>` defers the problem to runtime without fixing the design
- `@DefaultBean` no-ops conflict with existing `@DefaultBean` blocking test shims

---

## Design

### Principle

Two tiers. Each tier is a separate `@ApplicationScoped` bean. The blocking tier has zero
reactive imports. The reactive tier injects reactive SPIs directly — no `Instance<T>`, no
`isResolvable()` guards. The reactive tier is build-time gated; if the capability is
absent, the bean does not exist.

### New classes

**`ReactiveLedgerVerificationService`**
```
@ApplicationScoped
@IfBuildProperty(name = "casehub.ledger.reactive.enabled", stringValue = "true")
```
Injects: `ReactiveLedgerEntryRepository`, `ReactiveKeyRotationService`, `Event<AgentSignatureSuspectEvent>`
Methods: `verifyAgentSignatureAsync(UUID)`, `compromisedEffectiveSinceAsync(...)` (private)

**`ReactiveKeyRotationService`**
```
@ApplicationScoped
@IfBuildProperty(name = "casehub.ledger.reactive.enabled", stringValue = "true")
```
Injects: `ReactiveKeyRotationRepository`, `ReactiveLedgerEntryRepository`
Methods: `compromisedWindowsAsync(String, String)`, `rotationHistoryAsync(String)`, `recordRotationAsync(...)`

### What moves

| From | To |
|---|---|
| `LedgerVerificationService.verifyAgentSignatureAsync` | `ReactiveLedgerVerificationService` |
| `LedgerVerificationService.compromisedEffectiveSinceAsync` | `ReactiveLedgerVerificationService` |
| `KeyRotationService.compromisedWindowsAsync` | `ReactiveKeyRotationService` |
| `KeyRotationService.rotationHistoryAsync` | `ReactiveKeyRotationService` |
| `KeyRotationService.recordRotationAsync` | `ReactiveKeyRotationService` |

### What stays

`LedgerVerificationService` retains: `treeRoot`, `inclusionProof`, `verify`,
`verifyAgentSignature`, `compromisedEffectiveSince` (private), `verifyCryptographic` (private).
Zero reactive imports. Zero optional wiring.

`KeyRotationService` retains: `recordRotation`, `rotationHistory`, `compromisedWindows`.
Zero reactive imports. Zero optional wiring.

Both blocking services lose their reactive `@Inject` fields entirely.

---

## Build-time gating

```properties
# In reactive-capable consumers and test suite:
casehub.ledger.reactive.enabled=true

# JDBC-only consumers: property absent — reactive beans not indexed by augmentation
```

Three deployment outcomes:

| Context | Property set? | Reactive beans present? |
|---|---|---|
| JDBC-only consumer (aml, clinical, devtown) | No | No — clean build |
| Reactive consumer (qhorus, claudony) | Yes | Yes — full reactive tier |
| casehub-ledger test suite | Yes (test application.properties) | Yes — satisfied by @DefaultBean blocking shims |

`@IfBuildProperty` is from `io.quarkus.arc.properties` — already on the classpath.

---

## Reactive tier dependency chain

When `casehub.ledger.reactive.enabled=true`, both reactive services are present:

```
ReactiveLedgerVerificationService
  └── ReactiveLedgerEntryRepository (SPI, satisfied by consumer impl or @DefaultBean shim)
  └── ReactiveKeyRotationService
        └── ReactiveKeyRotationRepository (SPI, satisfied by consumer impl or @DefaultBean shim)
        └── ReactiveLedgerEntryRepository
```

Both services are gated on the same property — they appear and disappear as a coherent unit.

---

## Testing

**`src/test/resources/application.properties`** — add:
```properties
casehub.ledger.reactive.enabled=true
```

Existing `BlockingReactiveLedgerEntryRepository` and `BlockingReactiveKeyRotationRepository`
`@DefaultBean` shims satisfy the reactive SPI injections in the test suite. No change to
the shim pattern.

**Test class moves:**

| Currently in | Moves to |
|---|---|
| `KeyRotationServiceIT` — all `*Async` test methods | `ReactiveKeyRotationServiceIT` (new) — injects `ReactiveKeyRotationService` |
| `LedgerVerificationServiceIT` — `verifyAgentSignatureAsync_*` tests | `ReactiveLedgerVerificationServiceIT` (new) — injects `ReactiveLedgerVerificationService` |

Blocking tests in `KeyRotationServiceIT` and `LedgerVerificationServiceIT` are unchanged.

**New structural test:** A pure reflection test (no `@QuarkusTest`) verifies that
`LedgerVerificationService` and `KeyRotationService` contain no `Uni<T>`-returning methods —
enforcing that the blocking tier remains clean.

---

## DESIGN.md impact (journal merge at branch close)

`§Architecture` — add subsection on reactive/blocking tier split and `@IfBuildProperty` gating.

`§Key Design Decisions` — update "AgentSignatureSuspectEvent" entry: `verifyAgentSignatureAsync`
now lives in `ReactiveLedgerVerificationService`. Add rationale for tier separation over
`Instance<T>` and `@DefaultBean` no-op approaches.

`§Implementation Tracker` — new row for #92.

---

## Protocol alignment

| Protocol | Status |
|---|---|
| PP-20260519-f2e160 (reactive-blocking-tier-separation) | Implemented by this design |
| PP-20260519-39a9a5 (reactive-service-build-gating) | Implemented by this design |
| PP-20260517-15bf75 (ledger-sync-async-parity) | Retired — superseded by above |
| ledger-reactive-spi-shim (PP-20260519-3f2ea2) | Unchanged — shim pattern still valid |
| ledger-spi-propagation | Not applicable — no SPI method changes |
