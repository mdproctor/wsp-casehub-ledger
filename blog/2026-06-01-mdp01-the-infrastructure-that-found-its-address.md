---
layout: post
title: "The Infrastructure That Found Its Address"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [identity, did, platform, refactor, quarkus-extension, cdi]
---

The last few sessions assembled ledger's identity pipeline — DID binding, SCIM lookup,
key rotation, VC validation. It worked. It just happened to live in entirely the wrong place.

The question I kept sidestepping was whether ledger actually *owns* identity or whether
it happened to be the first repo that needed it. Once stated plainly, the answer was
obvious: ledger's domain is audit. It cares about *who* wrote an entry — but resolving
that identity isn't its responsibility. Any module wanting to verify an agent's identity
shouldn't have to pull in Merkle trees, trust scores, and Flyway migrations to do it.

I formalised this as a platform rule: a module owns infrastructure only if it is the
definitive provider of that capability. Development order doesn't determine ownership.

## Phase 1: the pieces with clean boundaries

We moved the pieces with no `LedgerEntry` in their API surface: the SPIs
(`ActorDIDProvider`, `DIDResolver`, `AgentCredentialValidator`), the model types,
the CDI events, and all the implementations — no-op defaults, `KeyDIDResolver`,
`WebDIDResolver`, `ScimActorDIDProvider`, the cache base. These landed in
`casehub-platform-identity`, a new module in the platform repo.

One constraint appeared during the move. `ScimActorDIDProvider` had an
`@Observes AgentKeyRotatedEvent` to evict its cache on key rotation. In platform
that's backwards — the platform module can't observe a ledger event without creating
a dependency in the wrong direction. The fix was a thin `IdentityCacheInvalidator`
bean in ledger: it observes the event and calls `provider.invalidate(actorId)` if
the active provider is an `AbstractCachingIdentityProvider`. The event stays in its
owning domain; the domain provides the bridge.

## Phase 2: cutting the LedgerEntry dependency

`AgentIdentityVerificationService` still took a `LedgerEntry` parameter. The
verification logic — resolve the DID document, check `alsoKnownAs`, match the
public key — has nothing to do with ledger. It only needed `actorId`, `actorDid`,
and `agentPublicKey`. We put the service in platform with those three primitives and
left thin adapter classes in ledger that extract the fields and delegate. Consumers
that inject by the ledger type continue to work unchanged.

## The build gotcha

After updating `deployment/pom.xml` to include the new deployment artifacts, the build
still failed with "missing dependencies" on the deployment module — even though the
source was correct. Claude identified the root cause: the Quarkus extension descriptor
validation reads the *installed* deployment JAR from `.m2`, not the current source.
The error message points at the deployment artifact and says it's missing deps, which
is true of the cached version. Nothing in the error mentions the cache. Fix: install
the deployment module first.

Both repos are on main. `casehub-platform-identity` now provides the SPIs and all
standard implementations. Ledger depends on it and wires identity into the audit
pipeline via its enrichers. Any future module needing agent identity verification
won't need to touch ledger.
