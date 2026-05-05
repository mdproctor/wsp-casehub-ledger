---
layout: post
title: "A Clean Entity"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, jpa, panache, reactive, documentation]
---

The integration guide still showed consumers calling `LedgerHashChain.compute(previousHash, entry)`.
That class was deleted two sessions ago. A consumer following the guide would hit a compile error
with no clue what replaced it.

The fix was blunt: remove every `LedgerHashChain` reference, drop `previousHash` from the base
fields table, simplify the capture service to what it actually does now:

```java
// Merkle leaf hash and frontier update handled automatically by save()
ledgerRepo.save(entry);
```

The write path handles everything. The docs should have said that from the start of the Merkle sprint.

## Why LedgerEntry Had to Change

The reactive migration request from Qhorus forced a decision I should have made earlier.
`LedgerEntry extends PanacheEntityBase` meant any reactive subclass was stuck — Hibernate Reactive
and blocking Hibernate ORM don't share a Panache inheritance hierarchy. Qhorus's
`AgentMessageLedgerEntry` couldn't extend something tied to blocking ORM.

The fix is clean: make `LedgerEntry` a plain `@Entity` POJO. Consumers subclass it however
they like — blocking Panache active record, reactive Panache repository, plain JPA. The entity
has no opinion.

## What Panache Actually Requires

The rough part was `JpaLedgerEntryRepository`. My first instinct: implement
`PanacheRepository<LedgerEntry, UUID>`. The Quarkus docs explicitly describe this as the
"repository pattern" for plain JPA entities — no `PanacheEntityBase` required.

At runtime, `listAll()` threw:

```
java.lang.IllegalStateException: This method is normally automatically overridden
in subclasses: did you forget to annotate your entity with @Entity?
```

The entity had `@Entity`. The error message is completely wrong. The actual problem: Panache's
bytecode enhancement for repository methods only fires when the entity type went through
Panache's own build-time processor. A plain `@Entity` never does. The methods are never enhanced,
the stubs remain, and the runtime blows up.

Fix: inject `EntityManager` and use JPQL. More verbose, but guaranteed.

One more consequence: `PanacheRepositoryBase.findById(Id)` returns `Entity` (nullable), but our
`LedgerEntryRepository.findById(UUID)` returned `Optional<LedgerEntry>`. Java can't satisfy both
from one method. We renamed ours to `findEntryById(UUID)`. Any consumer calling `repo.findById(id)`
needs to update.

## The Reactive Dep Trap

A subagent added `quarkus-hibernate-reactive-panache` as `<optional>true</optional>` and a
pre-built `ReactiveJpaLedgerEntryRepository`. I let it go — seemed useful.

Then the tests failed. The Hibernate Reactive extension activated, demanded a Vert.x reactive
datasource, and H2 doesn't provide one.

My first attempt was `quarkus.arc.exclude-types` — exclude the reactive bean from CDI scan.
Didn't help. Extension activation is a build-time classpath concern, not a bean discovery
concern. The extension activates the moment the jar is present, regardless of whether any
CDI beans use it.

The right answer was simpler: provide only the `ReactiveLedgerEntryRepository` SPI interface.
`Uni<T>` is already on the Quarkus classpath via Mutiny. Qhorus implements the interface in
their own module using `quarkus-hibernate-reactive-panache` — which they were adding anyway.
The pre-built implementation has no business being in this module.

`casehub-ledger` is now installed with `LedgerEntry` as a plain entity. Qhorus can subclass
it reactively.
