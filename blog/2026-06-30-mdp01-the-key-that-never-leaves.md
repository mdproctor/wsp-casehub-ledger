---
layout: post
title: "The key that never leaves"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [signing, kms, ec, architecture]
series: issue-102-cloud-kms-agent-signer
---

The ledger has always supported bilateral signing — an agent signs each entry it
creates, and the signature is verified on read. Until now, that meant the private
key lived on disk, loaded from PEM files at startup by `ConfiguredAgentSigner`. Fine
for development. Not great for production, where keys belong in a hardware security
module or cloud KMS that never releases them.

I wanted four providers: AWS KMS, GCP Cloud KMS, Azure Key Vault, and HashiCorp Vault
Transit (which was already in `examples/` but not promoted to a real module). The
design question that shaped everything else was whether these should be examples or
first-class modules. Examples are copy-and-paste — consumers vendor them and maintain
their own fork. That's the wrong answer for production signing infrastructure that
carries cloud SDK dependencies and needs CI testing. So we promoted them.

The architecture is two layers per provider. A pure Java signing client at the bottom —
no CDI, no Quarkus, no casehub-ledger dependency. It takes a config POJO, talks to the
cloud API, returns `byte[]` and `PublicKey`. Usable from Spring, Micronaut, or a plain
`main()`. A Quarkus CDI adapter on top extends `AbstractCachingAgentSigner`, adds
`@ConfigMapping`, scheduled cache refresh, and key rotation event handling. Consumers
add one Maven dependency and set `quarkus.arc.selected-alternatives` — done.

The interesting constraint was EC-only. RSA signature algorithms aren't deterministic
from the key material — `SHA256withRSA` vs `SHA256withRSAandMGF1` is a signing-time
choice not encoded in the public key. Supporting RSA would mean either storing the
algorithm alongside the signature (schema change) or trial-verifying with multiple
algorithms (fragile). EC keys embed the curve OID in the X.509 SubjectPublicKeyInfo,
so the mapping is deterministic: P-256 → `SHA256withECDSA`, P-384 → `SHA384withECDSA`,
P-521 → `SHA512withECDSA`. Clean.

But the existing verification infrastructure didn't know about EC at all —
`AgentCryptographicVerifier` and `LedgerPemUtil` only supported Ed25519 and ML-DSA.
Worse, three callers used `Signature.getInstance(key.getAlgorithm())` directly, which
works for Ed25519 but fails for EC because `ECKey.getAlgorithm()` returns `"EC"`, not
a valid `Signature` algorithm name. A shared `signatureAlgorithm(Key)` utility now
derives the correct algorithm from the EC curve parameters.

The other SPI change worth noting is `keyMaterial()`. The existing save pipeline called
`sign(actorId, new byte[0])` in `prepareKey()` just to extract the public key and
keyRef — the signature was discarded. For local signing that's free. For cloud KMS,
that's a paid API call thrown away on every entry persist. A new `default` method on
`AgentSigner` returns key material without triggering a sign operation. Cloud adapters
override it to return the cached public key directly.

The design review caught a subtle bug I'd have missed: Azure Key Vault returns EC
signatures in raw R‖S format (R and S concatenated), not DER. The `EcSignatureConverter`
handles the conversion, but the initial implementation wrote the DER SEQUENCE length as
a single byte. For P-256 and P-384, total content stays under 127 bytes — fine. For
P-521 with 66-byte components, the total can reach 138 bytes, requiring long-form DER
encoding. Without the fix, P-521 signatures would silently fail verification — no
exception, just `Signature.verify()` returning `false`. The kind of bug you'd chase for
hours before suspecting the encoding.

Eight new modules under `signing/`, four getting-started examples, a follow-up issue
for promoting the other example capabilities (otel-trace-wiring, prov-dm-export,
eigentrust-mesh, trust-score-routing) using the same pattern. The Vault Transit promotion
also fixed two latent bugs: `fetchPublicKey()` was selecting key version 1 (the oldest)
instead of the latest, and the `AgentKeyRotatedEvent` observer was missing entirely.

The two-layer split feels right. The pure Java clients are testable with plain Mockito
(AWS) or thin wrapper interfaces (GCP, Azure — their SDK clients are concrete classes).
No WireMock, no Docker, no cloud credentials in CI. The Quarkus layer is thin enough
that its tests are about CDI wiring, not signing logic.