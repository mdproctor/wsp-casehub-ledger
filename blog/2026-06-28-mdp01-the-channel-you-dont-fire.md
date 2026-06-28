---
layout: post
title: "The Channel You Don't Fire"
date: 2026-06-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [cdi, quarkus, events, dual-channel]
---

CDI 4.x has two completely disjoint event delivery channels. `fire()` reaches `@Observes` observers. `fireAsync()` reaches `@ObservesAsync` observers. Neither delivers to the other, and there's no warning when an observer exists on the wrong channel — it simply never fires.

I knew this in theory. The garden has an entry for it (GE-20260423-daef97). But knowing it and auditing every event producer in a codebase are different things. The issue asked us to normalise two event types to fire both channels. I wanted to verify that "two" was actually the right scope.

It wasn't.

We traced every `Event<>` injection point in ledger — 11 distinct event types across 10 producer classes. Only one (`TrustScoreComputedAt`) already fired both channels. The rest were single-channel. Some fired `fire()` only, leaving future `@ObservesAsync` consumers dead on arrival. Others — the reactive services — fired `fireAsync()` only, which is worse: it means *existing* sync observers silently miss events.

That's where the real bug was hiding. `ReactiveKeyRotationService.recordRotationAsync()` fires `fireAsync()`, but `IdentityCacheInvalidator` and `ActorIdentityValidationEnricher` both use `@Observes`. When a key is rotated through the reactive path, their caches are never invalidated. The sync observers are registered, they exist in the CDI container, and they never fire. No error, no log line, nothing.

The second bug was subtler. `ReactiveAgentSignatureVerificationService` was awaiting the `CompletionStage` returned by `fireAsync()` via `Uni.createFrom().completionStage()`. This looks like normal Mutiny bridging — it compiles, it passes tests, it works. Until an async observer throws. Then the `CompletionStage` fails, the Uni fails, and the caller gets an error instead of `VerificationResult.SUSPECT`. The SUSPECT verdict — a fact about cryptographic state and key compromise windows — becomes coupled to whether an observer successfully processed a notification. The fix was straightforward: `.invoke()` instead of `.completionStage()`, making the event dispatch a side effect rather than a pipeline dependency.

I initially thought the identity enricher was an exception — that `ActorIdentityValidationEnricher` couldn't safely call `fire()` because it runs inside `@PrePersist`, and sync observers would cause re-entrant persistence. That turned out to be wrong. The enricher runs from `JpaLedgerEntryRepository.save()`, not from a JPA entity listener. `LedgerTraceListener` — the actual `@PrePersist` callback — is just a guard that throws if you bypass the save pipeline. The recursion concern was also unfounded: `ActorDIDEnricher` already skips `ActorIdentityBindingEntry` via an instanceof guard, breaking the cycle before it starts.

So no genuine exception exists. Every producer fires both channels. The pattern is mechanical — `fire()` first, then `fireAsync()` with `.exceptionally()` — and the ordering matters: if the sync observer throws, the async dispatch is skipped, which prevents notifying about events whose transaction will roll back.

The disjoint-channel design is a CDI spec choice, not a bug, but it creates a class of silent failure that's almost impossible to catch through testing. Your test observers typically don't throw. Your test fixtures typically don't register on the "wrong" channel. The failure only surfaces in production when a real observer is added to the channel nobody fires — and then the symptom is "observer never invoked", which looks like a wiring problem, not a missing `fireAsync()` call.
