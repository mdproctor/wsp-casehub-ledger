---
layout: post
title: "The Persistence Layer That Didn't Know It Was One"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: diary
projects: [ledger]
---

`LedgerVerificationService` had a secret. Inside its `treeRoot()` method, injected alongside the proper `LedgerEntryRepository`, was a bare `EntityManager` — used to query `LedgerMerkleFrontier` directly. No SPI. Just JPA, baked in.

I knew this before the in-memory persistence module even existed, but it became impossible to ignore once the problem was clearly framed: you can't build a working alternative to JPA persistence if the service layer is wired directly to the database. Any in-memory implementation would handle every save correctly, then fail silently the moment someone called `treeRoot()`.

The fix was extracting `LedgerMerkleFrontierRepository` as a proper SPI — `findBySubjectId` and `replace`, used by both `LedgerVerificationService` and the Merkle update path in `save()`. `JpaLedgerMerkleFrontierRepository` provides the JPA implementation; `InMemoryLedgerMerkleFrontierRepository` provides a `ConcurrentHashMap`-backed alternative.

Once that was in place, Claude and I built out the rest of `casehub-ledger-memory` following the `casehub-eidos-memory` pattern. Six `@Alternative @Priority(1)` implementations — four blocking stores, two reactive delegates wired through `@IfBuildProperty(name = "casehub.ledger.reactive.enabled", stringValue = "true")`. The module needs a Jandex index in the JAR; without one, Quarkus's build-time augmentation can't discover `@Alternative` beans from external JARs regardless of Maven scope.

Two things caught us mid-build.

The first: `InMemoryKeyRotationRepository` reads from `InMemoryLedgerEntryRepository`, since key rotation entries are a `LedgerEntry` subclass and share the same map. The obvious approach was direct field access into the other class's `ConcurrentHashMap`. That compiles and runs without error. It also silently returns an empty stream every time, because `@ApplicationScoped` CDI beans are client proxies — direct field reads hit the proxy's own uninitialized field, not the underlying singleton. Only method calls dispatch. A four-line `allEntries()` method fixed it.

The second: `@Entity` parent fields declared `public` in source become `protected` in the bytecode-enhanced classloader that `@QuarkusTest` runs under. Test classes that aren't subclasses of the entity get `IllegalAccessError` at runtime — no hint in the stack trace about why a `public` field is inaccessible. The pattern: a package-private subclass in the test package, with accessor methods on the subclass to expose the fields the test needs.

Both are non-obvious enough that they'll recur. The CDI proxy one especially — it looks correct, compiles, and fails invisibly.
