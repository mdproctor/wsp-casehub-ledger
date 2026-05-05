---
layout: post
title: "No Panache in the Model"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, jpa, panache, architecture, entities]
---

Earlier in this session we converted `LedgerEntry` to a plain `@Entity` to unblock Qhorus's reactive migration. That was the forcing function. But once we'd done one entity, the question became: why stop there?

The answer I kept returning to: `casehub-ledger` is an extension library. Forcing `quarkus-hibernate-orm-panache` on every consumer is a dependency they didn't ask for. An application can choose Panache, reactive, plain JPA, or Spring Data — but if the base entity extends `PanacheEntityBase`, the choice is made for them. That's wrong for a library.

## The Panache Shorthand Question

Before committing to the full cleanup, I wanted to be honest about what we'd lose. Panache's main value here is query shorthand:

```java
// Panache
LedgerAttestation.list("ledgerEntryId = ?1 ORDER BY occurredAt ASC", id)

// EntityManager
em.createNamedQuery("LedgerAttestation.findByEntryId", LedgerAttestation.class)
  .setParameter("entryId", id)
  .getResultList()
```

Three times more verbose. But we have maybe fifteen queries across all repositories. The verbosity cost is low. And the @NamedQuery approach recovers something: Hibernate validates every named query at startup. A typo fails the boot, not the first runtime call. That's a genuine win.

## The JOINED Inheritance Surprise

During the entity migration, we also had to fix the retention job's deletion logic. The original approach — `em.merge(detachedEntity)` then `em.remove()` — seemed like the standard "safe re-attachment" pattern. For single-table entities it is. For JOINED inheritance, it silently loses the concrete subclass type on merge, so Hibernate can't find the subclass table row to delete.

The error it throws — `OptimisticLockException: Unexpected row count` — points entirely in the wrong direction. Nothing in the message suggests polymorphic type resolution. The fix is `em.find()` first (which issues a proper polymorphic SELECT), then `em.remove()`.

We caught this because the retention test failed in the full suite. A subagent found the root cause by reading the deletion logic carefully and recognising the cross-EntityManager context issue.

## What the Extension Now Looks Like

Seven internal entities — `LedgerMerkleFrontier`, `LedgerAttestation`, `ActorTrustScore`, `LedgerEntryArchiveRecord`, and the supplement stack — are all plain `@Entity`. `quarkus-hibernate-orm-panache` is gone from both pom files. Repositories use `EntityManager` + `@NamedQuery`. The `PlainEntityTest` structural test guards against regression.

Consumers still use Panache freely in their own code — they just can't inherit active-record methods from `LedgerEntry` anymore. That's the right tradeoff for a library: the base is dependency-minimal; consumers add what they need on top.
