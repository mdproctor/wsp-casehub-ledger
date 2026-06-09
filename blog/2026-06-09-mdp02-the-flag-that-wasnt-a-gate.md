---
layout: post
title: "The Flag That Wasn't a Gate"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [trust, multi-tenancy, spi]
---

## The Flag That Wasn't a Gate

Three small issues, one branch. The kind of batch where each change is
individually boring — an SPI method, a config flag, a record component — but
the design conversations around them turn out to be more interesting than the
code.

## The Query That Wasn't Slower

Issue #124 asked for per-capability database queries so `ComputedTrustScoreSource`
wouldn't load all attestations for an actor when only one capability was being
queried. Sounds reasonable until you look at the cache: `ComputedTrustScoreSource`
already loads everything once per actor and serves all subsequent calls from a
`ConcurrentHashMap`. The per-capability query would bypass the cache or require
partial-cache invalidation — both worse than what exists.

What the issue actually described was a different problem: the two-step
`findEventsByActorId` → extract IDs → `findAttestationsForEntries` pattern that
forced every caller to know how attestations are fetched via entry IDs. A leaky
abstraction, not a performance problem. We added `findAttestationsByActorId` as a
single SPI method — JPQL subquery under the hood, because `LedgerAttestation` has
no `@ManyToOne` mapping to `LedgerEntry` — and updated the two callers. The
number of SQL queries stays at two. The improvement is that callers no longer
bridge them manually.

## Materialization Is Not the Opposite of Computation

Issue #125 proposed excluding `TrustScoreJob` and `IncrementalTrustUpdateObserver`
via build-time CDI exclusion when `ComputedTrustScoreSource` is active. I started
designing that and got challenged: why are you trying to exclude it? Are they
mutually exclusive at runtime?

They're not. `ComputedTrustScoreSource` reads from raw attestation history.
`TrustScoreJob` writes to `ActorTrustScoreRepository`. `TrustExportService` reads
from `ActorTrustScoreRepository` — not through `TrustScoreSource`. A deployment
could legitimately want computed reads for low-latency trust queries while keeping
the batch job running for federation exports.

The right answer was a runtime config flag —
`casehub.ledger.trust-score.materialization.enabled` — that lets deployments opt
out of the write path without removing the beans from the CDI graph. Two guard
clauses, one new config interface. The build-time CDI exclusion would have
created a false coupling between source selection and persistence lifecycle.

## Tenancy Through the Async Gap

Issue #129 was the straightforward one. `ActorIdentityBindingObserver` receives
identity validation events via `@ObservesAsync` and persists binding entries —
but the async handler has no CDI request scope, so `CurrentPrincipal.tenancyId()`
is unreachable. The interim fix defaulted to `DEFAULT_TENANT_ID` in the repository.

The proper fix threads `tenancyId` through the event records themselves:
`AgentIdentityValidatedEvent` and `AgentIdentityViolationEvent` in
`casehub-platform-api` each gain `tenancyId` as their second component,
per the platform convention. The enricher reads it from the entry being persisted.
The observer reads it from the event. The repository fallback goes away — a null
tenancyId at save time is now a bug, not a default.

The cross-repo search confirmed zero consumers outside ledger, so the record
component addition is a contained compile break.
