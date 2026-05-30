---
layout: post
title: "Signing Proves Someone Signed — DID Proves Who"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [did, vc, identity, enricher, cdi]
---

The bilateral signing added in the previous branch proves that *someone* signed a ledger
entry with the private key for a given actorId. It does not prove that the person holding
that key is actually `claude:reviewer@v1`. Anyone who has read the Javadoc can set that
string. The signing binds a key to an entry; it doesn't bind an identity to a key.

That's what this branch was for.

## The string that couldn't prove itself

`actorId` has always been a convention. The format (`{model-family}:{persona}@{major}`) is
a good one — it encodes versioning semantics and accumulates trust correctly. But it's still
just an assertion. The ledger records whatever the calling code sends.

I wanted to bind `actorId` to a W3C DID so that the signing key is publicly attested and
third parties can verify agent identity without trusting the ledger's key store. The question
was how to do this without breaking the trust accumulation model, which depends on `actorId`
being stable across sessions.

The answer turned out to be two fields for two different concerns. `actorId` stays as the
trust accumulation key — the convention string is actually the right design for that purpose,
ADR 0004 reasoning intact. `actorDid` is the new cryptographic binding: a DID URI that
resolves to a document containing the signing key and an `alsoKnownAs` claim that asserts
the actorId.

## The divergence attack

The `alsoKnownAs` requirement isn't decoration. Without it, an operator could bind any DID
to any actorId — the DID document and the actorId are just two strings that happen to be
configured together. With it, the DID document itself must assert the binding, and the
validation rejects anything that doesn't match. If I configure `did:web:attacker.com` for
`claude:reviewer@v1` but the document doesn't list `claude:reviewer@v1` in `alsoKnownAs`,
validation returns `IDENTITY_MISMATCH` before the key check even runs.

The full pipeline runs in five steps: resolve the DID → check `alsoKnownAs` → check whether
there's a public key to cross-check → compare the stored `agentPublicKey` against the DID
document's verification methods → optionally validate a VC. Each step has its own result
status (`DID_UNRESOLVABLE`, `IDENTITY_MISMATCH`, `UNSIGNED`, `KEY_MISMATCH`,
`CREDENTIAL_INVALID`) so the specific failure mode is recorded in the audit entry.

## SCIM2 showed up

While designing the `ActorDIDProvider` SPI — the piece that maps an actorId to a DID — I
found that IETF published `draft-abbey-scim-agent-extension-00` in late 2025, defining
first-class SCIM 2.0 resource types for AI agents. The schema maps naturally: `Agent.externalId`
is the actorId convention string; a custom schema extension carries the DID URI.

That's the enterprise path: the operator provisions an agent in their IAM (Azure AD, Okta,
Keycloak), and `ScimActorDIDProvider` resolves the actorId → DID via
`GET /scim/v2/Agents?filter=externalId eq "claude:reviewer@v1"`. The casehub-specific SCIM2
schema extension and lookup protocol are now captured in casehubio/parent (issue #107 for
the raggable doc; protocol PP-20260530-bf919d). `ScimActorDIDProvider` itself is deferred to
#107 — the SPI and the simple config-based default are there; the enterprise integration is
the next layer.

The `did:key` method is the zero-infrastructure dev path: encode the public key directly in
the DID string, no HTTP required. `KeyDIDResolver` decodes it back to verify the key match.
The complication is that `did:key` documents are derived deterministically from the key bytes
and don't support `alsoKnownAs` — that's by spec design. Tests that need `alsoKnownAs` use
`TestDIDResolver`, a map-backed stub with no CDI overhead.

## CDI gave us trouble in the pipeline

The hardest implementation problem wasn't the DID logic — it was getting the enricher
ordering to work correctly. `ActorIdentityValidationEnricher` needs `agentPublicKey` to be
present before it runs the key match. `agentPublicKey` is set by `AgentSignatureEnricher`.
So the order matters: signing enricher first, then identity validation.

`LedgerEnricherPipeline` was sorting enrichers in unspecified CDI order. The natural fix is
`@Priority` — except using `e.getClass().getAnnotation(Priority.class)` returns null in
Quarkus because Arc wraps beans in proxy classes. The proxy class doesn't carry the original
annotations. We had to use `InjectableInstance.handlesStream()` and
`h.getBean() instanceof InjectableBean<?> ib → ib.getPriority()` to read priority from CDI
bean metadata rather than from the proxy class. Not in any user-facing docs; discoverable
only from Arc's API.

The other pipeline wrinkle: `ActorIdentityValidationEnricher` can't simply extend the
caching base class we built, because `loadContext(String key)` can only receive the cache
key — not the `LedgerEntry` needed to run the validation. The fix was to compose the cache
as a field and expose a `put()` bypass method, overriding `loadContext` with a no-op sentinel.
The enricher calls `cache.get(actorId)` to check for a hit, computes the status externally
when it misses, and calls `cache.put(actorId, status)` to store it. The base class handles
TTL and eviction from there.

## Persistence is decoupled

`ActorIdentityBindingEntry` is a `LedgerEntry` subclass — same pattern as `KeyRotationEntry`,
using `UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))` as the subject ID so key rotation
events and identity binding events form a unified actor lifecycle sequence per actorId.

Persisting it from inside the enricher's `@PrePersist` context would be unsafe — the JPA
spec doesn't guarantee flush ordering when you call `em.persist()` on a different entity
during a `@PrePersist` callback. So the enricher fires an async CDI event and leaves; a
separate `ActorIdentityBindingObserver` picks it up with `@ObservesAsync` and persists the
binding entry in a `@Transactional(REQUIRES_NEW)` context. The audit record commits
independently of the parent entry — it survives a parent rollback and failures don't affect
the main write.

ENFORCE mode works the same way: not from inside the enricher (enrichers can't throw), but
from a separate `LedgerIdentityEnforcementListener` registered as a `@EntityListeners` on
`LedgerEntry`. It reads the transient `pendingIdentityStatus` field that the enricher sets,
and throws if the mode is `ENFORCE` and the result is non-VALID. `@EntityListeners` is
JPA-only — it doesn't fire in the in-memory repository — so ENFORCE is a JPA-only concern.

523 tests passing across all four modules.
