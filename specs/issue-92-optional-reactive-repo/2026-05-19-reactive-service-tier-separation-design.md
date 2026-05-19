# Reactive Service Tier Separation Design

**Issue:** casehubio/ledger#92
**Branch:** issue-92-optional-reactive-repo
**Date:** 2026-05-19
**Protocols:** PP-20260519-f2e160 (reactive-blocking-tier-separation), PP-20260519-39a9a5 (reactive-service-build-gating)

---

## Problem

`LedgerVerificationService` and `KeyRotationService` mix blocking and reactive methods in
a single `@ApplicationScoped` bean. Both inject `ReactiveLedgerEntryRepository` and/or
`ReactiveKeyRotationRepository`, which are absent in JDBC-only consumers (casehub-aml,
casehub-clinical, casehub-devtown). Quarkus CDI augmentation validates all injection points
at build time — before tests or runtime config — causing build failures in those consumers.

The root cause is a design smell: mixing execution tiers in a single bean violates
PP-20260519-f2e160. The fix is structural separation, not workarounds.

**Rejected approaches:**
- `Instance<T>` — defers the problem to runtime, obscures intent, no structural improvement
- `@DefaultBean` no-op — conflicts with existing `@DefaultBean` blocking test shims

---

## Design

### Principle

Two tiers. Each tier is a separate `@ApplicationScoped` bean. The blocking tier has zero
reactive imports. The reactive tier injects reactive SPIs directly — no `Instance<T>`,
no `isResolvable()` guards. The reactive tier is excluded by the deployment module when
reactive capability is absent.

---

## New classes

**`ReactiveLedgerVerificationService`** — no CDI annotations beyond `@ApplicationScoped`

Injects: `ReactiveLedgerEntryRepository`, `ReactiveKeyRotationService`,
`Event<AgentSignatureSuspectEvent>`

Methods: `verifyAgentSignatureAsync(UUID)`, `compromisedEffectiveSinceAsync(...)` (private)

**`ReactiveKeyRotationService`** — no CDI annotations beyond `@ApplicationScoped`

Injects: `ReactiveKeyRotationRepository`, `ReactiveLedgerEntryRepository`

Methods: `compromisedWindowsAsync(String, String)`, `rotationHistoryAsync(String)`,
`recordRotationAsync(...)`

---

## What moves

| From | To |
|---|---|
| `LedgerVerificationService.verifyAgentSignatureAsync` | `ReactiveLedgerVerificationService` |
| `LedgerVerificationService.compromisedEffectiveSinceAsync` | `ReactiveLedgerVerificationService` |
| `KeyRotationService.compromisedWindowsAsync` | `ReactiveKeyRotationService` |
| `KeyRotationService.rotationHistoryAsync` | `ReactiveKeyRotationService` |
| `KeyRotationService.recordRotationAsync` | `ReactiveKeyRotationService` |

## What stays

`LedgerVerificationService` retains: `treeRoot`, `inclusionProof`, `verify`,
`verifyAgentSignature`, `compromisedEffectiveSince` (private), `verifyCryptographic` (private).
Zero reactive imports.

`KeyRotationService` retains: `recordRotation`, `rotationHistory`, `compromisedWindows`.
Zero reactive imports.

---

## Build-time gating — deployment module (not @IfBuildProperty)

`@IfBuildProperty` on runtime beans was evaluated and rejected for Quarkus extension use:
properties must be declared as `BUILD_TIME` phase config or Quarkus warns about reading
undeclared runtime properties at augmentation. The canonical extension pattern uses
`ExcludedTypeBuildItem` via `@BuildStep` in the deployment module.

### `LedgerBuildTimeConfig` (new — deployment module)

```java
@ConfigRoot(name = "casehub.ledger", phase = ConfigPhase.BUILD_TIME)
public interface LedgerBuildTimeConfig {
    /** Whether to activate the reactive service tier. Requires a reactive datasource. */
    @ConfigItem(name = "reactive.enabled", defaultValue = "false")
    boolean reactiveEnabled();
}
```

Property: `casehub.ledger.reactive.enabled=true`

The ledger uses `@LedgerPersistenceUnit` — its own named persistence unit — so a consumer
could have a reactive datasource for their own entities while keeping the ledger datasource
as JDBC. Using `quarkus.datasource.reactive` as the gate would conflate these. The
ledger-specific property is correct here.

### `LedgerProcessor` extension (deployment module)

```java
@BuildStep
void excludeReactiveBeans(
        LedgerBuildTimeConfig config,
        BuildProducer<ExcludedTypeBuildItem> excluded) {
    if (!config.reactiveEnabled()) {
        excluded.produce(new ExcludedTypeBuildItem(
            ReactiveLedgerVerificationService.class.getName()));
        excluded.produce(new ExcludedTypeBuildItem(
            ReactiveKeyRotationService.class.getName()));
    }
}
```

The reactive beans carry no gating annotation — exclusion is entirely the deployment
module's concern. This is the canonical Quarkus extension pattern.

### Three deployment outcomes

| Context | `casehub.ledger.reactive.enabled` | Reactive beans present? |
|---|---|---|
| JDBC-only consumer (aml, clinical, devtown) | not set (default false) | No — excluded by build step |
| Reactive consumer (qhorus, claudony) | `true` | Yes — full reactive tier |
| casehub-ledger test suite | `true` (test application.properties) | Yes — satisfied by @DefaultBean shims |

---

## Desynchronization guard

A consumer can set `casehub.ledger.reactive.enabled=true` without providing a
`ReactiveLedgerEntryRepository` implementation. This produces a runtime CDI failure rather
than a build failure. Add a startup check in `LedgerProcessor`:

```java
@BuildStep
void validateReactiveConfig(
        LedgerBuildTimeConfig config,
        Capabilities capabilities,
        BuildProducer<ValidationPhaseBuildItem.ValidationErrorBuildItem> errors) {
    if (config.reactiveEnabled()
            && !capabilities.isPresent(Capability.HIBERNATE_REACTIVE)
            && !capabilities.isPresent("io.quarkus.reactive.datasource")) {
        errors.produce(new ValidationPhaseBuildItem.ValidationErrorBuildItem(
            new ConfigurationException(
                "casehub.ledger.reactive.enabled=true requires a reactive datasource " +
                "extension (e.g. quarkus-hibernate-reactive) on the classpath")));
    }
}
```

This converts a deferred runtime failure into a build-time error with an actionable message.

---

## Testing

### Test application.properties

```properties
casehub.ledger.reactive.enabled=true
```

Existing `BlockingReactiveLedgerEntryRepository` and `BlockingReactiveKeyRotationRepository`
`@DefaultBean` shims satisfy reactive SPI injections in the test suite.

### JDBC-only test profile (new — regression test for the original bug)

```java
public class LedgerJdbcOnlyTestProfile implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of("casehub.ledger.reactive.enabled", "false");
    }
}
```

A `@QuarkusTest @TestProfile(LedgerJdbcOnlyTestProfile.class)` verifies:
- `LedgerVerificationService` starts cleanly with no CDI unsatisfied dependency
- `KeyRotationService` starts cleanly
- All blocking methods (`treeRoot`, `verifyAgentSignature`, `compromisedWindows`, etc.) work

This is the direct regression test for casehubio/ledger#92.

### Test class moves

| Currently in | Moves to |
|---|---|
| `KeyRotationServiceIT` — all `*Async` tests | `ReactiveKeyRotationServiceIT` (new) — injects `ReactiveKeyRotationService` |
| `LedgerVerificationServiceIT` — `verifyAgentSignatureAsync_*` | `ReactiveLedgerVerificationServiceIT` (new) — injects `ReactiveLedgerVerificationService` |

### New structural test

A pure reflection test (no `@QuarkusTest`) verifies that `LedgerVerificationService` and
`KeyRotationService` contain no `Uni<T>`-returning methods — enforcing that the blocking
tier remains clean.

---

## Transaction boundary — `recordRotationAsync`

`recordRotationAsync` in `ReactiveKeyRotationService` carries no `@ReactiveTransactional`
annotation. Reactive transaction management is the caller's responsibility — this is
intentional and consistent with the reactive SPI contract established by
`ReactiveLedgerEntryRepository`. Callers must ensure a reactive transaction context exists
when calling `recordRotationAsync`. This is documented in the Javadoc on the method.

---

## Parity enforcement — known gap

The protocol requires that every method in the blocking tier has a reactive counterpart,
and vice versa. There is currently no automated enforcement of this requirement. A developer
adding `KeyRotationService.revokeKey()` without `ReactiveKeyRotationService.revokeKeyAsync()`
violates the protocol silently.

Tracked at casehubio/ledger#94. Until resolved, parity is enforced by code review and
the protocol documentation only.

---

## Service granularity — known gap

`LedgerVerificationService` currently mixes Merkle tree operations and agent signature
operations. These are separate concerns. A sharper decomposition — `AgentSignatureVerificationService`
(blocking) and `ReactiveAgentSignatureVerificationService` (reactive) alongside a
Merkle-only `LedgerVerificationService` — is tracked at casehubio/ledger#93.

This refactor is deferred to avoid scope creep in #92.

---

## DESIGN.md impact (journal merge at branch close)

`§Architecture` — update reactive/blocking tier split section: gating is via
`ExcludedTypeBuildItem` in `LedgerProcessor` + `LedgerBuildTimeConfig`, not `@IfBuildProperty`.

`§Key Design Decisions` — update "AgentSignatureSuspectEvent" entry: `verifyAgentSignatureAsync`
now lives in `ReactiveLedgerVerificationService`. Add rationale for `ExcludedTypeBuildItem`
over `@IfBuildProperty` for extension runtime beans.

`§Implementation Tracker` — new row for #92.

---

## Protocol alignment

| Protocol | Status |
|---|---|
| PP-20260519-f2e160 (reactive-blocking-tier-separation) | Implemented by this design |
| PP-20260519-39a9a5 (reactive-service-build-gating) | Implemented — update to note ExcludedTypeBuildItem as canonical extension mechanism |
| PP-20260517-15bf75 (ledger-sync-async-parity) | Retired — superseded by above |
| ledger-reactive-spi-shim (PP-20260519-3f2ea2) | Unchanged |
| ledger-spi-propagation | Not applicable — no SPI method changes |

### Protocol update required

`reactive-service-build-gating.md` (PP-20260519-39a9a5) currently references `@IfBuildProperty`
as the gating mechanism. Update to note that for Quarkus **extensions** (with `runtime/`
and `deployment/` modules), `ExcludedTypeBuildItem` via `@BuildStep` is the canonical
mechanism. `@IfBuildProperty` is valid for **application** beans but requires `BUILD_TIME`
phase config declaration to work reliably in extensions.

---

## Open follow-ons

| Issue | What |
|---|---|
| casehubio/ledger#93 | Extract `AgentSignatureVerificationService` — separate Merkle and signature concerns |
| casehubio/ledger#94 | Enforce blocking/reactive tier parity at compile time |
| casehubio/qhorus#172 | Align qhorus with this pattern |
| casehubio/parent#32 | Protocol: when to choose reactive vs blocking execution model |
