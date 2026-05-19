---
layout: post
title: "The wrong fix for the right problem"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, cdi, reactive, architecture, design]
---

The bug report was straightforward. JDBC-only consumers — casehub-aml, casehub-clinical, casehub-devtown — failed to build after we shipped the reactive key rotation service. Quarkus CDI validation runs at augmentation, before any test or runtime config, and it found an unsatisfied injection point:

```
Unsatisfied dependency for type ReactiveLedgerEntryRepository
  - injection target: LedgerVerificationService#reactiveLedgerRepo
```

The obvious fix: make the injection optional with `Instance<T>`.

```java
@Inject @Any
Instance<ReactiveLedgerEntryRepository> reactiveLedgerRepo;
```

Guard with `isResolvable()`. JDBC-only consumers build cleanly. Done.

I didn't like it. `Instance<T>` moves the failure from build time to runtime, but it leaves the real problem intact: `LedgerVerificationService` is a bean that carries a dependency it only needs for one method. The reactive injection leaked in from `verifyAgentSignatureAsync`, which we added in a previous sprint. Everything else on the service — Merkle verification, inclusion proofs, the blocking signature check — has nothing to do with reactive. The bean became mixed-tier without anyone noticing.

The second rejected approach was a `@DefaultBean` no-op: ship a production fallback that throws `UnsupportedOperationException` for everything. `casehub-engine` does this for optional SPIs and it works well there. The problem here is that we already have `@DefaultBean` blocking shims in test sources. Two `@DefaultBean` beans for the same type is CDI ambiguity — the test suite breaks.

The right fix is structural: separate the tiers. Blocking-tier beans carry zero reactive imports. Reactive-tier beans live in separate classes, excluded from CDI when reactive isn't enabled. The gate mechanism is `ExcludedTypeBuildItem` in `LedgerProcessor` — the Quarkus extension deployment module — driven by a `@ConfigRoot(BUILD_TIME)` config property:

```java
@BuildStep
void excludeReactiveBeans(
        LedgerBuildTimeConfig config,
        BuildProducer<ExcludedTypeBuildItem> excluded) {
    if (!config.reactive().enabled()) {
        excluded.produce(new ExcludedTypeBuildItem(
            ReactiveKeyRotationService.class.getName()));
        excluded.produce(new ExcludedTypeBuildItem(
            ReactiveLedgerVerificationService.class.getName()));
    }
}
```

JDBC-only consumers leave `casehub.ledger.reactive.enabled` unset. The reactive beans never enter the CDI graph. Build succeeds.

Where things got interesting was the protocol work. The original "ledger services must ship blocking and reactive variants" protocol was narrowly scoped to ledger classes. Generalising it to any service library with heterogeneous deployment contexts turns it into a universal architectural principle. We wrote two new protocols: one universal (service beans must not carry dependencies optional in consuming deployments) and one Quarkus-specific (the `ExcludedTypeBuildItem` mechanism). We retired the original.

Then the question came back: is `@IfBuildProperty` — the simpler approach — actually reliable here? Claude flagged a risk in the review: `LedgerConfig.ReactiveConfig` in the runtime config root and `LedgerBuildTimeConfig` in the deployment module both declare `casehub.ledger.reactive.enabled`. A runtime caller who discovers `LedgerConfig.reactive().enabled()` and writes a guard against it will get a split-brain bug — the reactive beans may not be in the CDI graph regardless of what the runtime config says. We left the runtime declaration in place (removing it causes `SRCFG00050` at startup) but added explicit Javadoc: this field exists for config validation only; do not use it to gate CDI behaviour.

Four follow-on issues came out of the session: #93 (extract `AgentSignatureVerificationService` — Merkle and signature concerns still share `LedgerVerificationService`), #94 (enforce blocking/reactive parity at compile time), qhorus#172 (apply the same pattern there), and parent#32 (protocol for when to choose reactive vs blocking execution model). None of them blocked shipping.

`BlockingTierPurityTest` enforces the split via reflection — checking that blocking-tier beans have no `Uni<T>`-returning methods and no reactive SPI field injections. 450 tests passing.
