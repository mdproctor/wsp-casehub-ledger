---
layout: post
title: "How provider-agnostic really works"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [reactive, mutiny, quarkus, testing, code-review]
---

The `KeyRotationService` had blocking methods. The platform rule says all ledger service methods must ship both blocking and `Uni<T>` reactive variants. Adding the reactive variants looked routine — three methods to mirror, known patterns, done.

The first question that stopped me was architectural. Does this design genuinely support multiple persistence backends — in-memory, PostgreSQL, MongoDB?

It does, but the answer isn't obvious until you look at how `ReactiveLedgerEntryRepository` actually works. There is no JPA reactive implementation bundled in `casehub-ledger`. The interface is a pure SPI contract; consumers provide their own implementation using whatever reactive stack they run. PostgreSQL: Hibernate Reactive. MongoDB: a reactive Mongo client. The extension provides the contract and stays out of the persistence business.

Tests are a different problem. The `@QuarkusTest` suite runs against H2/JDBC — no Vert.x reactive datasource anywhere. When you add a reactive injection point and nothing satisfies it, CDI fails to start. The fix is a `@DefaultBean` shim in test sources that wraps the blocking JPA implementation with `Uni.createFrom().item()`:

```java
@DefaultBean
@ApplicationScoped
class BlockingReactiveKeyRotationRepository implements ReactiveKeyRotationRepository {

    @Inject
    KeyRotationRepository blocking;

    @Override
    public Uni<List<KeyRotationEntry>> findByActorId(String actorId) {
        return Uni.createFrom().item(() -> blocking.findByActorId(actorId));
    }
}
```

`@DefaultBean` means it's invisible in production — any real implementation takes precedence. It stays in `src/test/java` and never ships.

The more interesting problem surfaced in code review. `compromisedEffectiveSinceAsync` is the reactive private helper that mirrors the blocking `compromisedEffectiveSince`. The blocking version does this:

```java
.filter(w -> !occurredAt.isBefore(w.effectiveSince()))
.min(Instant::compareTo)
```

We'd written the reactive version with `.findFirst()` instead of `.min()`. Both produce the same result right now, because the named query orders by `effectiveSince ASC`. But `.min(Instant::compareTo)` is order-independent. `.findFirst()` silently depends on `ORDER BY`. If the query ordering ever changes, the blocking path stays correct; the reactive path breaks without warning. Claude caught it in the review. We switched to `.min()`.

The same review flagged `isAssignableFrom` in the structural SPI parity test. The structural check verifies that every method on the reactive SPI returns `Uni<T>`. We'd written `m.getReturnType().isAssignableFrom(Uni.class)` — which is always false, because `Uni` isn't a supertype of anything. The correct direction is `Uni.class.isAssignableFrom(m.getReturnType())`. The test compiled, ran green, and checked exactly nothing.

The reactive port is now complete. `verifyAgentSignatureAsync` no longer has a blocking bridge — it chains through `compromisedWindowsAsync` end-to-end.
