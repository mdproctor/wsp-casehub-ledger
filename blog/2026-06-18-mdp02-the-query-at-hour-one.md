---
layout: post
title: "The Query That Only Failed at Hour One"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [hibernate, jpql, cdi, quarkus, architecture]
---

The health job's gap-detection query had been wrong for a while. Not subtly wrong — wrong in a way that Hibernate 6 explicitly refuses to execute. The HAVING clause did arithmetic on `MIN()` and `MAX()` aggregates, and on PostgreSQL, Hibernate's semantic checker classifies those aggregate results as `Object` rather than a numeric type. `Operand of - is of type 'java.lang.Object'` — clear enough. What made it interesting is that H2 accepted the same query without complaint. All the tests passed. The failure was waiting in production, six hours in, when the health job fired for the first time.

I had assumed the bug was dialect-independent — that Hibernate's JPQL semantic analysis ran on the AST before any dialect was involved, so H2 and PostgreSQL would fail identically. That turned out to be wrong. The H2 dialect's aggregate type resolution takes a different path than PostgreSQL's, and in this case H2 lands on something numeric while PostgreSQL lands on Object. The fix ended up being simple — remove the HAVING clause, filter the healthy subjects in Java — but the diagnosis changed what I thought I understood about where JPQL validation happens.

The arithmetic had been in the HAVING clause as a premature optimisation anyway. The same calculation was already sitting in the loop body for logging. Moving the filter there was three lines of work. The harder question was why the buggy query had survived this long, and the answer was the one I didn't want: it was an inline `em.createQuery()` call. Had it been a `@NamedQuery` on `LedgerEntry`, Hibernate would have thrown at startup — not at hour one. The convention exists for exactly this reason.

So the fix expanded. We converted the four remaining inline queries in `JpaCrossTenantLedgerEntryRepository` to `@NamedQuery` as well, and moved the gap-detection query into `CrossTenantLedgerEntryRepository` as a proper SPI method returning a typed `SubjectSequenceStats` record. The health job had been injecting `EntityManager` directly — the only scheduled job in the codebase doing that. Not any more.

The third issue on the branch (#155) was about transaction isolation. The old `LedgerHealthJob.run()` was `@Transactional`, which meant both checks — gap detection and reconciliation — shared a single transaction. For a large ledger with many subjects, that holds a result set in memory while unrelated work runs. The fix uses CDI self-injection: `@Inject LedgerHealthJob self`, then `self.checkSequenceGaps()` and `self.checkReconciliation()` called through the proxy. Each call starts its own transaction. The method visibility matters here — private methods can't be intercepted by Arc's proxy subclass, so the check methods had to become package-private. It's a known CDI pattern but one I forget exists until the problem makes it necessary again.

The `@NamedQuery` guarantee is the thing worth remembering. A query that fails at boot is annoying. A query that fails at hour one of production, on PostgreSQL only, because your test suite runs H2 — that's the kind of bug that erodes trust in the health check you added to build trust.
