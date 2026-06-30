---
layout: post
title: "The save() that forgot to count"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [jpa, sequence, h2, merge, joined-inheritance, concurrency]
---

`JpaLedgerEntryRepository.save()` never assigned `sequenceNumber`. Every JPA-persisted entry got 0 — the primitive default. The in-memory implementation had it right from the start: `ConcurrentHashMap<UUID, AtomicInteger>`, one counter per subject. The JPA path just called `em.persist()` and hoped for the best.

The damage was worse than broken ordering. `sequenceNumber` is a canonical field — it feeds into `canonicalBytes()`, which feeds into both the Merkle leaf hash and the agent signature. Every JPA entry had its cryptographic integrity computed against `sequenceNumber = 0`. The signature was technically valid (it matched the stored value), but it proved nothing about the entry's actual position in the subject's history.

### Three syntaxes, one that works

The obvious fix was PostgreSQL's `INSERT ON CONFLICT DO UPDATE RETURNING` — one atomic statement, returns the allocated value. H2 rejected it. `MODE=PostgreSQL` provides DDL compatibility but not DML extensions. The error message is a generic syntax error with no hint about which clause is unsupported.

The fallback looked promising: H2's proprietary `MERGE INTO table KEY(col) VALUES(...)`. It appeared to work on the first call. The second call silently reset the counter to its seed value. H2's `MERGE KEY` replaces the *entire row* on match — it's not an upsert with selective update, it's a full-row overwrite. A counter table that resets to 1 on every call is worse than no counter at all, because the test passes once and the bug is invisible.

SQL-standard `MERGE INTO ... USING ... WHEN MATCHED THEN UPDATE SET ... WHEN NOT MATCHED THEN INSERT` works on both H2 2.4+ and PostgreSQL 15+. The `WHEN MATCHED` clause controls exactly which columns change. It took three attempts to find the syntax that's both correct and portable.

### The invisible save path

The UNIQUE constraint on `(subject_id, sequence_number)` surfaced a second bug. `ActorIdentityBindingEntry` extends `LedgerEntry` (JOINED inheritance — same base table), but it's persisted through `JpaActorIdentityBindingRepository.save()`, not `JpaLedgerEntryRepository.save()`. That repository calls `em.persist()` directly, with no sequence assignment. Before the constraint, both entries silently got `sequence_number = 0`. After it, the second binding entry for the same actor blew up with a constraint violation.

This is the architectural lesson. JPA JOINED inheritance stores all subclass rows in the same base table. Any invariant enforced in the base repository's `save()` is silently bypassed when a subclass has its own repository. There's no JPA mechanism that forces all subclasses through the same pipeline. The fix was extracting `nextSequenceNumber()` into a shared `LedgerSequenceAllocator` CDI bean that both repositories inject. The constraint is what made the invisible save path visible — without it, the data corruption would have been silent and permanent.