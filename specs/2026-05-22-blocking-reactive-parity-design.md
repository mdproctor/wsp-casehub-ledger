# Blocking/Reactive Service Parity Enforcement — Design

**Issue:** casehubio/ledger#94
**Branch:** issue-94-parity-enforcement
**Date:** 2026-05-22
**Deferred:** #96 (code-generation approach), #97 (generic-depth Uni<T> wrapping check)

---

## Problem

The `reactive-service-build-gating` protocol (PP-20260519-39a9a5) requires that every
public method in a blocking service has a `nameAsync` counterpart in its reactive pair,
and vice versa. Currently this is convention-only: no automated check exists.

A developer adding `KeyRotationService.revokeKey()` and forgetting
`ReactiveKeyRotationService.revokeKeyAsync()` violates the protocol silently. No test
catches it. The gap surfaces only at consumer compile time when the reactive path is called.

---

## Rejected approaches

**Plain reflection test (hardcoded pairs):** Reinvents ArchUnit badly. One test per pair;
updating the test registry is manual work that is easy to forget when a new pair is added.
Parameter type comparison via raw reflection is fragile and lacks a proper Java class model.

**Custom annotation + file-system scan:** Unnecessary complexity. ArchUnit discovers by
naming convention from bytecode — no annotations on production classes, no file-system
tricks, no classpath scanner library dependency.

**Quarkus `@BuildStep` in `LedgerProcessor`:** Fires at consumer deploy time, not during
the extension's own `mvn test`. Internal extension discipline requires the check to run
during the extension build. `@BuildStep` is the right place for consumer-facing contract
validation — wrong place for developer discipline within the extension.

**Research finding:** ArchUnit (TNG, v1.4.1) is the established industry tool for this
class of structural constraint. Vert.x solves the problem at the root via code generation
(`vertx-codegen` + `@VertxGen`) — deferred to #96 as a long-term consideration.

---

## Design

### Dependency

Add `archunit-junit5:1.4.1` in `test` scope to `runtime/pom.xml`. This is the first
ArchUnit adoption in the casehub platform; no BOM entry exists yet.

### New test class

`runtime/src/test/java/io/casehub/ledger/BlockingReactiveParityTest.java`

Follows the `FlywayLocationContractTest` pattern: plain JUnit 5, no `@QuarkusTest`, no
CDI container, no datasource. Runs in under 200ms.

### Discovery

```java
private static final JavaClasses SERVICE_CLASSES = new ClassFileImporter()
        .importPackages("io.casehub.ledger.runtime.service");
```

Reads compiled `.class` files from `target/classes/` — works from exploded Maven output
during `mvn test`. Scans the package and all subpackages. Filters via ArchUnit predicate:

```
haveSimpleNameStartingWith("Reactive").and().haveSimpleNameEndingWith("Service")
```

This auto-enrolls any `Reactive*Service` class added anywhere under the service package,
including future subpackages. No test maintenance required when new pairs are added.

**Current pairs discovered:**
- `ReactiveKeyRotationService` ↔ `KeyRotationService`
- `ReactiveAgentSignatureVerificationService` ↔ `AgentSignatureVerificationService`

### Test 1 — bidirectional method parity

`reactiveServices_mustMirrorBlockingCounterpart()`

A custom `ArchCondition<JavaClass>` that captures `SERVICE_CLASSES` in closure and for
each discovered reactive class:

1. **Blocking class exists** — derives name by stripping the `Reactive` prefix; asserts a
   class with that simple name exists in the imported set. Reports the reactive class as
   the violation site if not found.

2. **Blocking → Reactive** — for each `public`, non-synthetic method `foo(P1, P2, ...)` in
   the blocking class: asserts `fooAsync(P1, P2, ...)` exists in the reactive class with
   identical raw parameter types (count + fully qualified type names, in order). Both
   missing method and parameter mismatch are reported as separate `ConditionEvent`s.

3. **Reactive → Blocking** — for each `public`, non-synthetic method in the reactive class:
   asserts it ends in `Async` (naming convention), then asserts the blocking class has a
   method named without the `Async` suffix with matching parameters.

All violations are collected before the assertion fails — multi-violation failures show
the complete picture, not first-fail.

**Excluded from both directions:**
- Private methods — `compromisedEffectiveSince` (private in both current services) excluded correctly
- Synthetic methods — CDI proxy methods injected by Quarkus augmentation
- Constructors — `JavaClass.getMethods()` excludes them by definition in ArchUnit

### Test 2 — return type enforcement

`reactiveServices_allPublicMethodsMustReturnUni()`

For each `public`, non-synthetic method in each discovered reactive class: asserts raw
return type name equals `io.smallrye.mutiny.Uni`. Uses the correct `isAssignableFrom`
direction (GE-20260519-3ffbc0 gotcha — inverted form is always false):

```java
Uni.class.isAssignableFrom(m.getRawReturnType().reflect())
```

Reports the specific method name and actual return type in the violation message.

Note: this checks erasure only — `Uni<KeyRotationEntry>` and `Uni<Void>` both pass.
Full generic-depth checking (verifying the `T` in `Uni<T>` matches the blocking return
type) deferred to #97.

### Failure messages

If `KeyRotationService.revokeKey(String)` is added without a reactive counterpart:
```
Architecture Violation
  Blocking method 'revokeKey()' in KeyRotationService has no reactive counterpart
  'revokeKeyAsync()' in ReactiveKeyRotationService
```

If a reactive method has wrong parameter types:
```
Architecture Violation
  Parameter mismatch: revokeKey([java.lang.String])
  vs revokeKeyAsync([java.lang.String, boolean])
```

If a reactive method returns the wrong type:
```
Architecture Violation
  Reactive service method 'revokeKeyAsync()' must return Uni<T>,
  but returns java.util.List
```

If a new `ReactiveAuditService` is added without a blocking `AuditService`:
```
Architecture Violation
  No blocking counterpart class named 'AuditService' found
  for io.casehub.ledger.runtime.service.ReactiveAuditService
```

---

## Files

| File | Change |
|------|--------|
| `runtime/pom.xml` | Add `archunit-junit5:1.4.1` test dependency |
| `runtime/src/test/java/io/casehub/ledger/BlockingReactiveParityTest.java` | New — 2 `@Test` methods |

No production code changes. No annotation on reactive services. No changes to
`LedgerProcessor`, `LedgerBuildTimeConfig`, or any existing service class.

---

## Protocol alignment

| Protocol | Status |
|---|---|
| `PP-20260519-39a9a5` (reactive-service-build-gating) | Fulfilled — parity now machine-verified, not convention-only |
| `PP-20260519-f2e160` (reactive-blocking-tier-separation) | No change — already implemented |

The protocol line *"Adding a method to the blocking tier requires adding the Uni<T>
equivalent to the reactive tier, and vice versa — parity is structural, not co-located"*
goes from convention-only to build-enforced.

---

## Platform scope

ArchUnit is not currently used in any casehub repo. This is the first adoption. The
pattern — `ClassFileImporter` + custom `ArchCondition` for cross-class structural
constraints — is transferable to `casehub-qhorus` (#172) and others.

---

## Deferred

| Issue | What |
|-------|------|
| #96 | Code-generation approach (à la Vert.x codegen) — eliminates drift by construction |
| #97 | Strengthen return type check to verify `Uni<T>` wraps the correct `T` |
