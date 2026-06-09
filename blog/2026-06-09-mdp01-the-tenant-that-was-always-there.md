---
layout: post
title: "The Tenant That Was Always There"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [multi-tenancy, spi, cdi, architecture]
---

Engine had already solved this problem. When we added tenancyId to casehub-engine a week ago, the first instinct was elegant: inject `CurrentPrincipal`, read `tenancyId()` in each repository, filter silently. The callers stay clean. The protocol says exactly this — PP-20260520-e6a5f0 says tenancyId binding belongs in the data access layer.

What killed it was context. `TrustScoreJob` runs on a Quartz timer. `IncrementalTrustUpdateObserver` fires via `@ObservesAsync`. `LedgerRetentionJob` is `@Scheduled`. None of these have an active CDI request scope. `currentPrincipal.tenancyId()` throws `ContextNotActiveException` at runtime, not at compile time — the worst kind of failure, because it passes code review.

So the design is explicit parameters, same as engine: `String tenancyId` on every tenant-scoped SPI method. The SPI break is the point. Every caller is forced to be explicit about which tenant they're operating on. The migration is mechanical; the breakage is the forcing function.

The interesting decision was what *doesn't* get tenancyId. Trust scores are global. An actor's reputation is the same regardless of which tenant asks — `ActorTrustScoreRepository` doesn't split into tenant-scoped and cross-tenant halves; it's entirely cross-tenant. Same for key rotations and identity bindings. These are per-actor, not per-tenant. The Merkle frontier is scoped by `subjectId` which is already unique per tenant, so the tenancy filter there is defence in depth, not a functional requirement.

The cross-tenant operations — `findAllEvents()`, `listAll()`, `findByTimeRange()` — moved to a separate `CrossTenantLedgerEntryRepository` interface, produced via a `@CrossTenant` CDI qualifier. A `CrossTenantProducer` wraps the repository bean and guards it with `isCrossTenantAdmin()`. The pattern mirrors engine's `CrossTenantProducer` and `@EngineSystem` exactly — `@LedgerSystem` marks the system-level `CurrentPrincipal` so the producer can inject it without competing with the request-scoped principal.

One surprise. `ActorIdentityBindingObserver` persists `ActorIdentityBindingEntry` via `@ObservesAsync` — no request scope, so no tenant context. The NOT NULL constraint on `tenancy_id` caught it immediately, which is exactly the kind of failure that validates the unconditional-filtering protocol. The interim fix defaults to `DEFAULT_TENANT_ID` in the JPA save; the proper fix carries tenancyId through the CDI event record, which needs a platform-api change.

The downstream cascade is real. Three issues filed: work#260, qhorus#263, engine#459. Each consumer that implements `LedgerEntryRepository` needs to add `tenancyId` to every method. The NoOps in engine are trivial; the real implementations in work and qhorus need their own tenancy specs before implementation. We're producing all three specs independently, then reviewing for cross-repo coherence before anyone writes code.
