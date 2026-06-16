---
layout: post
title: "The Repository That Stopped Short"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [merkle, tenancy, identity-binding, repository-design]
---

The previous session closed three issues and left two open. Issue #144: `JpaActorIdentityBindingRepository.save()` never called `frontierRepo.replace()`. Issue #145: `latestBindingFor` and `bindingHistoryFor` had no `tenancyId` parameter. Both were filed by that session and deferred.

Looking at them now, they share a root cause: `ActorIdentityBindingRepository` had its own `save()` method. That method did just enough — allocating a sequence number, computing the leaf hash — but not everything. No Merkle frontier update. No enricher pipeline. And the read methods, written before the tenancy work landed, had never been updated to filter by tenant.

The fix was already in the codebase. `KeyRotationRepository` has no `save()` at all. Its Javadoc says it plainly: *"persisted via `LedgerEntryRepository#save`."* `KeyRotationService.recordRotation()` calls `ledgerRepo.save(entry, tenancyId)`, and that's it — the full pipeline runs, the frontier gets updated, the sequence number is allocated. `InMemoryKeyRotationRepository` delegates reads to `InMemoryLedgerEntryRepository.allEntries()` filtered by `instanceof`.

We hadn't applied that pattern to `ActorIdentityBindingRepository`, so it had built its own incomplete version. The fix was to remove `save()` from the SPI entirely and route the observer through `LedgerEntryRepository.save()`.

There was one wrinkle: the event loop. If `ActorIdentityBindingEntry` goes through the full save pipeline, `ActorDIDEnricher` resolves the actor's DID and sets `actorDid` on the entry. Then `ActorIdentityValidationEnricher` sees a non-null `actorDid`, fires `AgentIdentityValidatedEvent`, and `ActorIdentityBindingObserver` receives it and saves another binding entry — which goes through the pipeline again. On paper, infinite loop.

In practice it isn't quite that, because `ActorIdentityValidationEnricher` gates on a per-actor status cache. The loop fires once on a cache miss, then the cache hit on the second entry prevents the event from firing again. So without any guard it would produce one extra binding entry per cache miss, and restart whenever the cache is invalidated — on key rotation, for instance.

The actual fix was to guard `ActorDIDEnricher` itself with an `instanceof ActorIdentityBindingEntry` check. If DID resolution is never attempted on binding entries, `actorDid` stays null, `ActorIdentityValidationEnricher` short-circuits on its existing null check, and no event fires at all. The loop becomes unconditionally impossible rather than cache-bounded.

There's a secondary reason for that guard. `ActorIdentityBindingEntry` has a `boundDid` field: the DID that was actually validated in this binding event. If `ActorDIDEnricher` ran on binding entries, it would populate `actorDid` from the provider's current DID at async-commit time — which might differ from `boundDid` if the actor rotated their DID in the window between the validation event firing and the binding entry committing. Two DID fields on the same audit record that can disagree is a quiet integrity issue, and it has no upside. Better to skip the enricher entirely. Binding entries are self-describing.

For the in-memory side: `InMemoryActorIdentityBindingRepository` had been silently assigning `sequenceNumber=0` and leaving `digest` null on every binding entry. Its `save()` just appended to a `CopyOnWriteArrayList` with no pipeline at all. That's now gone — replaced with the delegation pattern from `InMemoryKeyRotationRepository`. Binding entries go through `InMemoryLedgerEntryRepository.save()`, get proper sequence numbers under the per-subject lock, and the in-memory Merkle frontier gets updated alongside everything else.

The read-side fix was mechanical: add `tenancyId` to both named queries and all three implementations, switch ordering from `occurredAt` to `sequenceNumber`. Named queries were previously ordering by timestamp, which is non-deterministic for entries committed within the same millisecond — easy to hit in test suites that flush the cache mid-test and trigger a second validation immediately.

What was filed as two separate bugs was one structural decision made at the wrong time.
