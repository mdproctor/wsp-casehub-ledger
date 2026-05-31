---
layout: post
title: "The Cache That Knew When to Forget"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [cdi, events, cache-invalidation, scim2, did, identity, quarkus-arc]
---

The branch before this one proved that ledger entries can be cryptographically bound to W3C DIDs. This one asked the harder operational question: what happens to the caches when a key rotates?

## The direct call that was always a smell

`KeyRotationService.recordRotation()` used to call `identityEnricher.invalidate(actorId)` directly after persisting the rotation entry. The comment even admitted it: *Issue #103 will replace this direct call with CDI event-driven invalidation.* I'd written that comment and then shipped the branch.

When we started building `ScimActorDIDProvider` — a SCIM2-backed implementation of `ActorDIDProvider` that caches per-actorId DID lookups — the problem became concrete. `KeyRotationService` would need a second direct call: `scimProvider.invalidate(actorId)`. A third implementation would need a third. That's not a pattern, it's a list.

The fix was already described in the issue: a CDI event fired after persist. Observers handle their own caches.

```java
public record AgentKeyRotatedEvent(String actorId, String previousKeyRef, String newKeyRef) {}
```

`KeyRotationService` fires it synchronously — within the transaction, before returning. `ReactiveKeyRotationService` fires via `fireAsync()` after the Uni completes, fire-and-forget. `ActorIdentityValidationEnricher` and `ScimActorDIDProvider` each observe it and call `invalidate(event.actorId())`. The rotation concern no longer bleeds into the identity layer.

## The ARC behaviour that wasn't in the spec

I wanted to put the `@Observes` method in `AbstractCachingAgentSigner` — the abstract base class — so concrete CDI subclasses would inherit it automatically. CDI 4.0 §3.5 says observer methods are inherited by bean subclasses, so this should work.

It doesn't in Quarkus ARC 3.32.2.

When `AbstractCachingAgentSigner` (which implements `AgentSigner`, a CDI SPI interface) has an `@Observes` method, ARC registers the abstract class itself as a `@Dependent` managed bean. This makes it a second candidate for the `AgentSigner` injection point alongside `ConfiguredAgentKeyProvider`, and 12 `@InjectMock AgentSigner` integration tests fail with `AmbiguousResolutionException`. The symptom points at the injection point, not at the abstract class — it takes a while to work out where the extra bean came from.

The fix: remove `@Observes`. Leave `onKeyRotated(AgentKeyRotatedEvent event)` as a plain public method. Concrete CDI beans wire it as `@Observes` explicitly. There are no concrete CDI subclasses of `AbstractCachingAgentSigner` in production right now, so the method sits as a hook for when they appear.

## SCIM2 and the `+` that shouldn't be there

`ScimActorDIDProvider implements ActorDIDProvider` — it resolves `actorId → DID URI` by querying a SCIM2 `Agent` endpoint. The lookup is `GET /scim/v2/Agents?filter=externalId eq "{actorId}"`, where `actorId` values like `claude:reviewer@v1` must be percent-encoded in the filter value.

`URLEncoder.encode(actorId, StandardCharsets.UTF_8)` handles the colons and `@` correctly. It also encodes spaces as `+`, which is correct for HTML form encoding but wrong for URL query parameters. SCIM filter values need `%20`, not `+`.

```java
String encodedActorId = URLEncoder.encode(actorId, StandardCharsets.UTF_8)
        .replace("+", "%20");
```

That's not a thing most people know until it breaks.

The CDI pattern for an optional enterprise integration is `@ApplicationScoped @Alternative` — activated via `quarkus.arc.selected-alternatives` in `application.properties`, dormant otherwise. `ConfiguredActorDIDProvider @ApplicationScoped` stays the simpler config-based alternative; `ScimActorDIDProvider` replaces it when SCIM is in play.

The config keys `endpoint` and `authToken` are `Optional<String>` in the `@ConfigMapping` interface. I'd initially tried `@WithDefault("")`, which seemed right. SmallRye rejects it: empty string is treated as null for plain `String` mappings, and SmallRye Config throws `SRCFG00040` at startup even when the provider isn't activated. `Optional<String>` is the correct shape for absent-by-default credentials.

The spec review caught the `publicKeyBytes` issue before any code was written: SCIM `x509Certificates[0].value` is a DER-encoded X.509 certificate, not the `SubjectPublicKeyInfo` bytes that `LedgerEntry.agentPublicKey` stores. Extracting the actual public key requires `CertificateFactory.getInstance("X.509")`. Since nothing currently needs that field from the SCIM resource, we dropped it from `ScimAgentResource` entirely and filed a separate issue for when there's a concrete consumer.

The CDI integration test had a subtle assertion problem. The initial version asserted `totalCalls > callsBeforeRotation` after triggering a rotation. That passes even if the cache wasn't evicted — `ActorDIDEnricher` calls `didFor()` as a side effect of every `ledgerRepo.save()`, and `KeyRotationService.recordRotation()` calls `ledgerRepo.save()`. The count increases regardless. We fixed it by swapping the WireMock stub to return a different DID after rotation and asserting the new value is returned — if the cache wasn't evicted, the old DID comes back and the assertion fails.

## The reactive bridge

`ReactiveAgentIdentityVerificationService` wraps the blocking `AgentIdentityVerificationService` for reactive consumers. The blocking service only uses `DIDResolver` (in-memory cache + HTTP), so this is a pure bridge with no Hibernate Reactive dependency.

```java
@DefaultBean
@ApplicationScoped
@Unremovable
public class ReactiveAgentIdentityVerificationService {

    @Inject AgentIdentityVerificationService blockingService;

    public Uni<IdentityVerificationResult> verifyIdentityBindingAsync(LedgerEntry entry) {
        return Uni.createFrom()
                .item(() -> blockingService.verifyIdentityBinding(entry))
                .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

`@Unremovable` is required because no code within the extension itself injects this type. ARC's dead-code elimination removes it at build time without it — consumers get `UnsatisfiedResolutionException` at their augmentation step. A pure bridge with no internal injection point always needs `@Unremovable`.
