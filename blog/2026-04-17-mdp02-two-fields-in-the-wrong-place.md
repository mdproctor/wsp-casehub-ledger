---
layout: post
title: "Two Fields in the Wrong Place"
date: 2026-04-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, jpa, architecture, supplements, causality, art12]
---

This session was the longest one yet on casehub-ledger. We covered a lot of ground —
research into ten possible directions, a compliance sprint through three of the top four,
and then a small architectural correction that turned out to be more interesting than it
first appeared.

## The Research Pass

I started by searching the web and Google Scholar for guidance on each of the ten
directions I'd identified: Merkle trees, forgiveness mechanisms in EigenTrust, EU AI Act
specifics, W3C PROV-DM, causality in distributed systems, and the rest. The forgiveness
research was particularly good — multiple independent papers (Binmad & Li 2016, the
e-business agent models) converge on a two-parameter model: recency and frequency.
Severity appears in theoretical frameworks but not in models that were actually evaluated
empirically. That finding directly shaped the ADR we wrote for the forgiveness mechanism:
severity is already implicit in the `decisionScore` majority calculation, adding it
explicitly would double-count the same signal.

## The Sprint

We built issues #7, #8, and #9 back-to-back. The LedgerSupplement architecture (#7)
turned out to be the biggest structural change in the project's short life — moving ten
optional fields off `LedgerEntry` into three supplement entities
(`ComplianceSupplement`, `ProvenanceSupplement`, `ObservabilitySupplement`) backed by
separate joined tables, lazily loaded, never written unless a consumer explicitly attaches
one. The forgiveness mechanism (#8) was cleaner — a two-parameter function applied
per-negative-decision inside `TrustScoreComputer.compute()`, disabled by default.
The Art.12 compliance surface (#9) was the most operationally useful: a
retention job that archives and deletes entries past their window, and three new auditor
query methods (`findByActorId`, `findByActorRole`, `findByTimeRange`) using `Instant`
parameters so queries can't silently slide across timezone boundaries.

## The Supplement Question

Designing the causality query API (#10) started simply: `ObservabilitySupplement`
already had `causedByEntryId`. All we needed was a `findCausedBy(UUID)` query method.

Then I stopped. Why is a causality field in a supplement?

Supplements exist for optional cross-cutting enrichment — GDPR Art.22 decision fields
that only AI decision systems need, provenance tracking that only workflow-driven
consumers need. The test for whether something belongs in core is: is this field
relevant to every consumer, every entry, every time? `causedByEntryId` fails it — not
every entry has a causal predecessor. But so does `actorId`. Nobody questions whether
`actorId` belongs in core. The difference is that `actorId` is always *potentially*
applicable, whereas `causedByEntryId` is null for most entries. That's not the same
as being optional enrichment — it's a structural relationship that might not always
have a value but is always meaningful when it does.

The same argument applies to `correlationId`. OTel is built into Quarkus. Every REST
call and agent invocation in the ecosystem has a trace ID. Linking an audit entry to
its distributed trace is not enrichment — it's the bridge between the audit layer and
observability infrastructure.

With both fields moving to `LedgerEntry` core, `ObservabilitySupplement` had no
fields left. The right conclusion was obvious: delete it. We went from three supplement
types to two, and the remaining two (`ComplianceSupplement` and `ProvenanceSupplement`)
are genuinely optional, domain-specific enrichment. The supplement architecture is now
cleaner than it was before.

The implementation was straightforward: edit V1000 to add an index on
`caused_by_entry_id` (the column was already there from before V1002 dropped it),
edit V1002 to stop dropping it, add the fields to `LedgerEntry`, delete the supplement
class, remove the OBSERVABILITY branch from the serialiser. Since we have no deployed
instances, editing migration files in place is entirely valid — no need for ALTER TABLE
ceremony.

## A Bug in the Repository

We discovered during the Art.12 sprint that the Flyway SQL files — V1000, V1001,
V1002 — had **never been committed to git**. They existed on disk, and tests passed
because H2 in-memory databases with `DB_CLOSE_DELAY=-1` cache the schema within the
JVM process. A fresh `mvn clean test` from a different terminal (or CI) failed with
`Table "LEDGER_ENTRY" not found`. The files had been created during development and
simply never added. Three `git add` commands later, the repository is reproducible.

## What's Next

Issue #10 (causality) is closed. The remaining work on the current research list is
#11: the Merkle tree upgrade to the hash chain. The current linear chain requires
O(N) work to verify any single entry — you need every entry from genesis. A Merkle
tree gives O(log N) inclusion proofs, closing Axiom 4 (Verifiability) and making the
extension credible as public infrastructure. It's the architecture play before
Quarkiverse submission. That's the next session.
