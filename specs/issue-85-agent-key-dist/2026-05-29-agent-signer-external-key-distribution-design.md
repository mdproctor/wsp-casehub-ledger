# Design Spec — AgentSigner: External Key Distribution (HSM / TUF / PKI)

**Issue:** casehubio/ledger#85  
**Branch:** issue-85-agent-key-dist  
**Date:** 2026-05-29  
**Status:** Approved

---

## Problem

`ConfiguredAgentKeyProvider` reads PEM files from local disk at startup. This model is
incompatible with:

- **Vault Transit / Cloud KMS** — private key never leaves the KMS; signing happens via API call
- **HSM via non-JCA API** — hardware signing requires a remote call, not local `Signature.getInstance()`
- **Hot key rotation** — PEM-based provider requires restart to pick up new keys

The current SPI (`AgentKeyProvider.signingKey(actorId) → Optional<SigningKey>`) assumes the
caller will do local JCA signing. This assumption breaks for remote-signing models.

---

## Core Design Decision

**The signing responsibility belongs in the SPI, not the caller.**

Replace `AgentKeyProvider` with `AgentSigner`. Instead of returning a key pair and leaving
signing to `AgentSignatureEnricher`, the SPI performs (or delegates) the signing and returns
the complete result. This is the minimal change that makes remote-signing implementations
possible without any workaround.

`AgentKeyProvider` and `SigningKey` are deleted. This is a deliberate breaking change.

---

## New Types

### `AgentSigner` (replaces `AgentKeyProvider`)

```java
@FunctionalInterface
public interface AgentSigner {
    /**
     * Signs {@code data} on behalf of {@code actorId} and returns the complete signature
     * result — raw signature bytes, X.509-encoded public key, and keyRef.
     *
     * <p>Return {@link Optional#empty()} for actors that do not participate in bilateral
     * signing — their entries will be persisted unsigned.
     *
     * <p><strong>Thread safety:</strong> implementations must be safe for concurrent calls
     * from multiple threads. {@code AgentSignatureEnricher} calls this from JPA
     * {@code @PrePersist} listeners, which may run concurrently.
     *
     * <p><strong>Error handling:</strong> throw any {@link RuntimeException} to signal a
     * transient failure (key store unreachable, network error). The enricher swallows the
     * exception and leaves the entry unsigned — a non-fatal operational failure. Do NOT
     * return {@link Optional#empty()} for errors; reserve empty for "actor not configured".
     *
     * <p><strong>Performance:</strong> implementations are expected to cache per-actorId
     * context (public key bytes, remote client setup). See {@link AbstractCachingAgentSigner}.
     *
     * <p><strong>JCA / PKCS#11 HSMs:</strong> hardware security modules that expose a JCA
     * {@code Provider} return a {@code PrivateKey} that proxies signing operations into the
     * hardware. Load the key pair via {@code KeyStore.getInstance("PKCS11")}, extend
     * {@link AbstractCachingAgentSigner}&lt;KeyPair&gt;, and call
     * {@link AgentSignature#sign(KeyPair, byte[])}. JCA routes the signing call into the
     * hardware without exporting key material.
     *
     * <p><strong>Algorithm transparency:</strong> implementations must not hardcode a
     * cryptographic algorithm string. See protocol PP-20260523-e7b577.
     *
     * @param actorId the actor identity (e.g. {@code "claude:reviewer@v1"})
     * @param data    the canonical bytes to sign (from {@link LedgerMerkleTree#canonicalBytes})
     * @return signed result, or empty if this actor does not sign entries
     */
    Optional<AgentSignature> sign(String actorId, byte[] data);
}
```

### `AgentSignature` (replaces `SigningKey` in public API)

```java
public record AgentSignature(byte[] signature, byte[] publicKey, String keyRef) {
    /**
     * Algorithm-transparent local signing factory.
     *
     * Derives the signing algorithm from {@code keyPair.getPrivate().getAlgorithm()}.
     * Computes {@code keyRef = Base64URL(SHA-256(publicKey.getEncoded()))}.
     *
     * Used by {@code ConfiguredAgentSigner} and test lambdas.
     */
    public static AgentSignature sign(KeyPair keyPair, byte[] data) { ... }
}
```

`signature` — raw JCA signature bytes (or raw bytes from remote KMS, stripped of any
provider-specific prefix).  
`publicKey` — X.509 DER-encoded public key bytes. Embeds the algorithm OID; drives
algorithm detection in `AgentCryptographicVerifier`. Unchanged from current `agentPublicKey`
column semantics.  
`keyRef` — `Base64URL(SHA-256(publicKey))`. Unchanged from current `agentKeyRef` semantics.

---

## Changes to Existing Runtime Classes

### `AgentSignatureEnricher`

Inject `AgentSigner` instead of `AgentKeyProvider`. One method call covers all three fields:

```java
signer.sign(entry.actorId, LedgerMerkleTree.canonicalBytes(entry))
      .ifPresent(sig -> {
          entry.agentSignature = sig.signature();
          entry.agentPublicKey = sig.publicKey();
          entry.agentKeyRef    = sig.keyRef();
      });
```

Exception handling stays the same — `try/catch` around the whole block, non-fatal.

### `ConfiguredAgentSigner` (replaces `ConfiguredAgentKeyProvider`)

- `@DefaultBean @ApplicationScoped`
- Direct implementation of `AgentSigner` — does NOT extend `AbstractCachingAgentSigner`
- Eager `@PostConstruct` loading: parses all PEM files at startup, logs errors for failed
  actors. Startup failure visibility is an operational property worth keeping.
- Signs locally via `AgentSignature.sign(keyPair, data)` — algorithm-transparent
- `SigningKey` becomes a private implementation detail (or removed; logic inlined)

### Deleted

- `AgentKeyProvider` — deleted
- `SigningKey` — deleted or made private to `ConfiguredAgentSigner`

---

## New Runtime Class: `AbstractCachingAgentSigner<C>`

Abstract base for external key providers that have per-call overhead (network, hardware).

```
C = per-actorId context type
    For extractable-key providers (TUF, Vault KV): C = KeyPair
    For remote-signing providers (Vault Transit): C = VaultTransitContext(pubKey, keyRef, keyName)
```

**Cache semantics:**
- Cache is `ConcurrentHashMap<String, Optional<C>>` — `Optional.empty()` is the "not configured"
  sentinel, preventing repeated `loadContext` calls for unknown actors
- `loadContext` returning `null` → cached as `Optional.empty()`
- `loadContext` throwing → NOT cached; next call retries (transient failure path)
- Concurrent load race: `putIfAbsent` pattern; duplicate loads are idempotent

**Template methods:**

```java
// Returns null if actorId is not configured for signing.
// Throws RuntimeException for transient failures (network, auth).
protected abstract C loadContext(String actorId);

// Called only when context is present. Must not cache the signing result.
protected abstract AgentSignature performSign(String actorId, C context, byte[] data);
```

**Cache management hooks (subclasses call from `@Scheduled` or event observers):**

```java
protected void invalidateAll()               // full cache clear
protected void invalidate(String actorId)    // single-actor eviction
```

No CDI annotations on the abstract class. Subclasses provide `@ApplicationScoped`.

---

## New Example: `examples/vault-transit-signing/`

Standalone Maven module (same structure as `examples/order-processing/`).

### What it demonstrates

Remote signing via Vault Transit Secrets Engine: private key never leaves Vault.
Signing is performed by a Vault API call; the application holds only the public key.

### `VaultTransitAgentSigner`

```java
@ApplicationScoped
@Alternative
@Priority(1)
public class VaultTransitAgentSigner extends AbstractCachingAgentSigner<VaultTransitContext> { ... }
```

`VaultTransitContext` (private record): `vaultKeyName`, `publicKey` (bytes), `keyRef`.

**`loadContext(actorId)`:**
1. Resolve Vault key name from config mapping (`casehub.ledger.vault-transit.key-mapping."<actorId>"`)
2. `GET /v1/transit/keys/<key-name>` with `X-Vault-Token`
3. Parse public key from response (PEM → X.509 bytes via `LedgerPemUtil`)
4. Compute `keyRef`
5. Return `VaultTransitContext`

**`performSign(actorId, ctx, data)`:**
1. `POST /v1/transit/sign/<key-name>` with `{ "input": "<base64(data)>" }`
2. Parse `data.signature` from response; strip `vault:v1:` prefix; decode base64
3. Return `AgentSignature(rawBytes, ctx.publicKey(), ctx.keyRef())`

**Algorithm transparency:** Vault's key type (ed25519, ecdsa-p256, rsa-2048) determines the
algorithm. The stored `publicKey` X.509 bytes carry the OID. `AgentCryptographicVerifier`
detects the algorithm from those bytes — no algorithm string appears in the signer.

**Scheduled cache refresh:**
```java
@Scheduled(every = "${casehub.ledger.vault-transit.refresh-interval:5m}")
void refresh() { invalidateAll(); }
```

**HTTP client:** `java.net.http.HttpClient` — no Vault Java SDK dependency.

### Configuration

```properties
casehub.ledger.vault-transit.address=http://vault:8200
casehub.ledger.vault-transit.token=<token>
casehub.ledger.vault-transit.key-mapping."claude:reviewer@v1"=reviewer-signing-key
casehub.ledger.vault-transit.refresh-interval=5m
```

### Tests (`VaultTransitAgentSignerIT`)

WireMock stubs the Vault HTTP API (test scope, no real Vault):
- Happy path: sign returns expected bytes; public key loaded from key info response
- Cache hit: second `sign()` issues only one `GET /v1/transit/keys/...`
- `invalidateAll()`: next `sign()` re-fetches (second GET issued)
- Vault 403: `loadContext` throws; enricher swallows; entry unsigned
- Vault 500: `loadContext` throws; not cached; retry succeeds on second call

---

## Test Updates (existing runtime tests)

### `AgentSignatureEnricherTest`

Lambda construction changes from `actorId -> Optional.of(SigningKey.of(kp))` to:
```java
(actorId, data) -> Optional.of(AgentSignature.sign(testKeyPair, data))
```
Test assertions unchanged.

`signatureVerifiesAgainstStoredPublicKey` — verification using JCA `Signature.getInstance()`
stays; the test explicitly confirms the SPI contract produces a verifiable signature.

### `AgentSigningIT`

```java
@InjectMock AgentSigner agentSigner;  // was AgentKeyProvider

when(agentSigner.sign(eq("claude:reviewer@v1"), any()))
    .thenAnswer(inv -> Optional.of(AgentSignature.sign(testKeyPair, inv.getArgument(1))));
when(agentSigner.sign(argThat(id -> !"claude:reviewer@v1".equals(id)), any()))
    .thenReturn(Optional.empty());
```

### New: `AbstractCachingAgentSignerTest`

Pure unit test with a `TestSigner extends AbstractCachingAgentSigner<String>`:
- Cache hit verified via call count on `loadContext`
- `Optional.empty()` returned and cached for unconfigured actors
- Throw from `loadContext` not cached (retry path)
- `invalidateAll()` forces reload

---

## ADR

New ADR supersedes ADR 0011 (per-actorId key model). Records:
- Why signing belongs in the SPI (remote-signing models require it)
- Why `AgentKeyProvider` → `AgentSigner` is the right break
- The two-tier structure: `ConfiguredAgentSigner` (eager, direct) vs `AbstractCachingAgentSigner<C>` (lazy, cached)
- Vault Transit as the reference implementation for the remote-signing pattern
- Deferred: Cloud KMS adapters (#102), rotation-triggered invalidation (#103), InMemoryAgentSigner (#104)

---

## Deferred (filed as issues)

| Issue | Description |
|-------|-------------|
| #101 | Vault AppRole / OIDC auth for `VaultTransitAgentSigner` |
| #102 | Cloud KMS adapters (AWS KMS, GCP KMS, Azure Key Vault) |
| #103 | Rotation-triggered cache invalidation via CDI event |
| #104 | `InMemoryAgentSigner` in `persistence-memory/` |

---

## Platform Coherence Check

- **SPI types:** `AgentSigner` uses only `byte[]` and `String` — no SDK types in contract. ✅
- **Algorithm transparency:** protocol PP-20260523-e7b577 — no hardcoded algorithm strings anywhere. `AgentSignature.sign()` derives from key; `AgentCryptographicVerifier` detects from X.509 bytes. ✅
- **CDI pattern:** `ConfiguredAgentSigner @DefaultBean`; external providers `@Alternative @Priority(1)` — Pattern B from `alternative-extension-patterns.md`. ✅
- **Example module structure:** follows existing `examples/` conventions (standalone Quarkus app, own pom, WireMock for I/O). ✅
- **No platform-level doc update needed:** this is entirely within `casehub-ledger`'s existing ownership of agent signing. ✅
