---
layout: post
title: "The Annotation That Wasn't There"
date: 2026-09-23
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [quarkus, virtual-threads, jpa, transactional, code-generation]
series: issue-351-transactional-vthread-fix
---

# The Annotation That Wasn't There

The APT generator we built for `@McpDomain` had been emitting `@RunOnVirtualThread`
on every generated REST endpoint since the streaming work landed. What it wasn't
emitting was `@Transactional`. On platform threads that doesn't matter — Hibernate
binds the JPA session to the request context automatically. On virtual threads,
that binding doesn't happen. JPA queries execute without a persistence context
and silently return empty results. No exception. No warning. Just wrong data.

AML found this the hard way — `findEvidence()` returning empty from a generated
endpoint that worked fine when called directly. They fixed it by adding
`@Transactional` to their service method, but every consumer would have to
discover this independently.

The deeper question was where the fix belongs. Three candidate layers: the
generated endpoint, the SPI implementation, or the JPA repository. Tracing
ledger's own call chain made the answer obvious. `DefaultLedgerEntryApi` has
no `@Transactional`. `JpaLedgerEntryRepository` has it on write methods
(`save`, `saveAttestation`) but not on any of the fifteen-plus read methods.
The generated endpoint is the only layer that knows about thread dispatch —
it emits `@RunOnVirtualThread`, so it should emit the corresponding
`@Transactional`. These are paired concerns: virtual thread dispatch removes
the implicit session binding, and `@Transactional` restores it by creating
the JTA transaction Hibernate needs.

While investigating the REST fix, a second gap surfaced. The generated GraphQL
resolvers had neither annotation — they were running on SmallRye GraphQL's
default `@Blocking` dispatch, which pins them to a fixed-size platform thread
pool. The REST endpoints scale with virtual threads; the GraphQL endpoints
don't. Same SPI, different scalability characteristics. SmallRye GraphQL has
supported `@RunOnVirtualThread` since Quarkus 3.8, so the fix was
straightforward: emit both annotations on GraphQL `@Query` and `@Mutation`
methods too. Skip for streams, webhooks, subscriptions, and `Uni` returns —
those are reactive and don't need either.

The actual change was four edits in `GraphQLResolverProcessor.java`. Adding
the import alongside `RunOnVirtualThread`, emitting `@Transactional` before
`@RunOnVirtualThread` under the same guard, and repeating both for the GraphQL
method generator. The `@Transactional(REQUIRED)` default cooperates cleanly
with any downstream transaction annotation — it's a no-op when no JPA is
involved, joins an existing transaction if one is present, and creates a new
one if not. No scenario where it causes harm.

The paired-concern insight is the thing worth remembering. If you dispatch
to a virtual thread, you've moved off the platform thread where the container
manages session lifecycle. You need to explicitly establish what the platform
thread gave you implicitly. `@Transactional` is how you do that in Quarkus.
