---
layout: post
title: "Routing Signals, a Health Check, and the Claude That Went Off-Script"
date: 2026-04-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [cdi, quarkus, health-check, trust-scoring]
---

I wanted to ship the trust score routing signals and run a health check before
thinking about Quarkiverse submission. I brought Claude in on the routing signals
first. That part went cleanly. The health check was more revealing.

## Routing signals: CDI as a strategy selector

The problem with routing signals is that "routing" means different things to
different consumers. CaseHub task assignment might want a full ranked list after
every computation. A lighter consumer might only care which actors changed
meaningfully. A monitoring layer might just want a "scores refreshed" timestamp.
I didn't want a configuration enum to pick one.

We landed on using distinct Java record types as the CDI event payload:

```java
TrustScoreFullPayload    // all current scores — rebuild a full ranked list
TrustScoreDeltaPayload   // only changed actors — incremental update
TrustScoreComputedAt     // timestamp + count — lightweight signal
```

Consumers `@Observes` whichever type they need. No config, no annotation
strategy. CDI type dispatch handles the routing. A consumer wanting deltas
just declares:

```java
void onDelta(@Observes TrustScoreDeltaPayload payload) { ... }
```

The one non-obvious piece: CDI 4.x `event.fire()` delivers only to `@Observes`
(synchronous) observers. To reach `@ObservesAsync` observers, you need a separate
`event.fireAsync()` call. The two paths are completely independent — if you only
call `fire()`, async observers are silently never notified. We hit this during
integration tests and spent a moment assuming the test setup was wrong before
checking the spec.

The publisher uses `BeanManager.resolveObserverMethods()` in `@PostConstruct` to
detect whether any observers are registered for each payload type at startup,
caching the result. If no delta observer is registered, the database pre-read for
delta computation is skipped entirely. For a nightly batch job this matters — the
job should be cheap when nothing is listening.

## A health check that was overdue

With 192 tests green, I ran a tier 4 project health check expecting minor doc
drift. Four of the six example modules didn't compile.

`mvn install` only runs the default Maven profile. The examples live under
`with-examples` — a clean install passes even when they're broken. The root causes
were accumulated drift:

The `correlationId` field was renamed to `traceId` in the entity several sessions
back, but `OrderResource.java` still referenced `e.correlationId`. The
`LedgerEntryRepository` SPI had gained a `listAll()` method that the example
repositories — all of which implemented the SPI directly from scratch — had never
added. And those implementations used Panache static calls (`LedgerEntry.list()`,
`entry.persist()`) that stopped working when we removed Panache from the entities.

The fix was switching the example repositories from `implements LedgerEntryRepository`
to `extends JpaLedgerEntryRepository`. The subclass pattern inherits every SPI
method automatically and gets the Merkle frontier handling inside `save()` for free.
The integration guide Step 4 now shows this pattern. The old version showed
`LedgerEntry.findById(id)` — a Panache call on a plain `@Entity` that simply
doesn't exist.

`correlationId` had also crept into five documentation files — README, integration
guide, CAPABILITIES, AUDITABILITY, prov-dm-mapping — all referencing a field name
that no longer exists. That was a straightforward find-and-replace across docs.

## The Claude that went off-script

While I was working on the broken examples, Claude silently modified seven runtime
classes — `JpaLedgerEntryRepository`, `LedgerErasureService`, `LedgerRetentionJob`,
and others — adding `@PersistenceUnit("qhorus")` annotations to every `EntityManager`
injection point. The commit message was superficially plausible: *"feat: add
@PersistenceUnit("qhorus") to all EntityManager injection points — qualifies
EntityManager injections so beans resolve correctly when only the qhorus datasource
is configured."*

This is precisely the wrong thing to do. `casehub-ledger` is a generic extension
with no knowledge of Qhorus. Qualifying the default `EntityManager` injection with a
named persistence unit breaks every deployment that isn't Qhorus. The commit looked
like it might be valid because it didn't break the runtime tests — which weren't
re-run after that particular task completed.

I caught it during the issue-tracking session when `git log` showed an unfamiliar
commit at HEAD. Reset, force push, done. But it's a useful reminder that subagent
scope can drift silently in directions that look coherent at the commit level but
are architecturally wrong for the project.

