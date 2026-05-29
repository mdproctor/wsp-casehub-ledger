---
layout: post
title: "Signing Doesn't Belong in the Enricher"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [ledger]
tags: [architecture, spi, vault, cryptography, agent-signing]
---

The question that opened this session: can `AgentKeyProvider` work with Vault Transit?

The answer is no — and it's structural. `AgentKeyProvider` returns a `KeyPair`. The enricher then calls `Signature.getInstance(algo).initSign(privateKey)` and signs locally. That whole model assumes the application has the private key. Vault Transit's entire point is that the private key never leaves Vault. You send data; you get a signature back. The SPI had the wrong shape.

I'd left this known but deferred since ADR 0011 was written. Vault Transit, Cloud KMS, HSMs via REST — they all break the same assumption. This session fixed it.

The change is a one-method rewrite. Instead of `AgentKeyProvider.signingKey(actorId) → Optional<SigningKey>`, the new SPI is `AgentSigner.sign(actorId, data) → Optional<AgentSignature>`. The enricher stops caring how signing happens. It gets a result back, sets three fields on the entry, and moves on. The signing boundary moved from the enricher into the SPI.

Two concrete things fell out of that shift:

`AgentSignature` is a record with defensive copies. The compact constructor clones both byte array fields — without that, any implementation that caches its own `AgentSignature` and the enricher share the same array reference. The initial review caught a null guard gap and a `catch (Exception e)` that was too broad. Both fixed before the first commit.

`AbstractCachingAgentSigner<C>` is the base class for external providers. The type parameter is whatever context the implementation needs per actorId — a `KeyPair` for extractable-key providers, a `VaultTransitContext` record for remote signers. The tricky bit was the cache implementation. I went with `putIfAbsent` rather than `computeIfAbsent` — `computeIfAbsent` holds the bucket lock for the entire duration of the mapping function. For a network call (fetching a public key from Vault), that blocks all map operations on keys that hash to the same bucket. The duplicate-load race from `putIfAbsent` is the safer trade-off.

The Vault Transit example (`examples/vault-transit-signing/`) demonstrated the pattern end-to-end: `GET /v1/transit/keys/<name>` to cache the public key, `POST /v1/transit/sign/<name>` to sign, strip the `vault:v1:` prefix from the response. Four WireMock tests — happy path, cache hit (verifying only one GET issued), unmapped actor, 403 not cached on retry. Claude got the initial version wrong in four places: WireMock's `verify()` takes `RequestPatternBuilder` not `MappingBuilder` (compiles fine, fails at runtime), missing `@Alternative @Priority(1)`, no `@Scheduled` cache refresh, and an inline PEM trial-load instead of `LedgerPemUtil.parsePublicKey()` which already exists and knows about the full algorithm list. A second pass fixed all four.

The deletion step found more than expected. The plan listed three files to remove — `AgentKeyProvider`, `SigningKey`, `ConfiguredAgentKeyProvider`. Compilation after deletion revealed four more test files (`KeyRotationIT`, `SuspectEventIT`, `KeyRotationServiceIT`, `ReactiveKeyRotationServiceIT`) that all referenced `SigningKey` for computing `keyRef` strings. None of them were in the original list. They were updated and the full suite went green: 463 runtime tests, all passing.

Nine commits, issue #85 closed.

The ledger now supports remote signing without any changes to how entries are written. An operator adds a Vault Transit key mapping to config and activates the example's `VaultTransitAgentSigner`. The rest of the pipeline — enrichment, Merkle chaining, verification — is unchanged.
