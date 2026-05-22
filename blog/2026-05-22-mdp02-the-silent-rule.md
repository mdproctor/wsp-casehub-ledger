---
layout: post
title: "The Silent Rule"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [ledger]
tags: [java, archunit, testing, structural-testing]
---

The parity protocol for casehub-ledger's blocking and reactive service tiers was enforced by the document that stated it. Add a method to `KeyRotationService`, add the reactive equivalent to `ReactiveKeyRotationService`, and vice versa. The rule existed. The build didn't know about it.

I wanted to fix that. Three options were on the table: a plain reflection test that hardcodes the class pairs, a marker annotation with a Quarkus `@BuildStep` validating at augmentation time, or something better. The reflection test would work for two pairs but not generalise — updating a manual registry every time a new pair is added is the kind of thing that gets forgotten. The `@BuildStep` approach fires at consumer deploy time, not during the extension's own build, so the people developing casehub-ledger wouldn't see violations until a consumer hit a broken build.

I asked Claude to go look for the industry answer. It came back pointing at ArchUnit — a library that scans bytecode, runs as plain JUnit 5, and has a proper domain model for expressing cross-class structural constraints.

The dependency was one line. The test discovers all `Reactive*Service` classes by naming convention — no annotation required on the production classes. `ClassFileImporter` reads compiled bytecode from `target/classes/`, before Quarkus augmentation, so the methods visible to the test are exactly the methods the developer wrote. For each reactive service the condition looks up its blocking counterpart, then checks parity in both directions: every public method `foo()` in the blocking class must have `fooAsync()` in the reactive class with the same raw parameter types, and vice versa.

Then I asked Claude to review the test independently. It caught something I'd missed.

ArchUnit rules pass silently when the `that()` predicate matches zero classes. No failure, no warning, zero assertions executed — the test reports green. If the package moves, or `ClassFileImporter` is pointed at the wrong location after a rename, you have a structural enforcement test that enforces nothing. The fix is an explicit count assertion before the rule runs:

```java
assertThat(reactiveServiceCount())
        .as("expected at least 2 Reactive*Service classes in scope — "
                + "if the package moved, update the ClassFileImporter")
        .isGreaterThanOrEqualTo(2);
```

This is the right shape for any structural test that discovers subjects by naming convention. The assertion doesn't test the architecture — it confirms the test has something to say.

We also hit one minor API gap during implementation: `JavaMethod.isSynthetic()` doesn't exist in ArchUnit 1.4.1, unlike Java reflection's `Method.isSynthetic()`. The filter turned out to be unnecessary anyway — ArchUnit reads pre-augmentation bytecode and Quarkus doesn't add public synthetic methods at compile time — but the compile error was the first signal.

One test class, one new dependency. The parity protocol is no longer enforced by the document that states it.
