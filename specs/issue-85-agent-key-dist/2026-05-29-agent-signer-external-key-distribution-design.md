# Design Spec — AgentSigner: External Key Distribution (HSM / TUF / PKI)

**Issue:** casehubio/ledger#85  
**Branch:** issue-85-agent-key-dist  
**Date:** 2026-05-29  
**Status:** Approved (rev 2 — post-review)

---

## Problem

`ConfiguredAgentKeyProvider` reads PEM files from local disk at startup. This model is
incompatible with:

- **Vault Transit / Cloud KMS** — private key never leaves the KMS; signing happens via API call
- **HSM via non-JCA API** — hardware signing requires a remote call, not local `Signature.getInstance()`
- **Restart-free key refresh** — PEM-based provider requires restart to pick up rotated keys

The current SPI (`AgentKeyProvider.signingKey(actorId) → Optional<SigningKey>`) assumes the
caller will do local JCA signing. This assumption breaks for remote-signing models.

**What this PR delivers on key refresh:** `AbstractCachingAgentSigner` exposes an
`invalidate(actorId)` hook that subclasses call from a scheduled job — giving bounded-staleness
refresh (up to the configured interval). Event-driven invalidation on rotation (true hot swap,
zero staleness) is tracked separately in #103.

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
     * hardware without exporting key material. Load the key pair via
     * {@code KeyStore.getInstance("PKCS11")}, extend
     * {@link AbstractCachingAgentSigner}&lt;KeyPair&gt;, and call
     * {@link AgentSignature#signWith(KeyPair, byte[])}.
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

`@FunctionalInterface` is intentionally absent. `AgentSigner` has one abstract method and
supports lambda construction in tests (SAM interface — no annotation required). Omitting the
annotation leaves room for `default` methods in future and avoids implying that lambda
construction is the intended production pattern.

### `AgentSignature` (replaces `SigningKey` in public API)

```java
public record AgentSignature(byte[] signature, byte[] publicKey, String keyRef) {

    // Defensive copies — callers and implementations cannot mutate fields after construction.
    public AgentSignature {
        signature = signature.clone();
        publicKey = publicKey.clone();
    }

    /**
     * Algorithm-transparent local signing factory.
     *
     * Derives the signing algorithm from {@code keyPair.getPrivate().getAlgorithm()} —
     * no hardcoded algorithm string. Computes
     * {@code keyRef = Base64URL(SHA-256(publicKey.getEncoded()))}.
     *
     * Used by {@code ConfiguredAgentSigner} and test lambdas.
     */
    public static AgentSignature signWith(KeyPair keyPair, byte[] data) { ... }
}
```

`signature` — raw bytes. For JCA local signing: `Signature.sign()` output. For Vault Transit:
base64-decoded payload after stripping the `vault:v1:` prefix. Must be standard format for the
algorithm (ed25519: 64 raw bytes; ECDSA: ASN.1 DER; RSA: raw PKCS#1).  
`publicKey` — X.509 DER-encoded. Embeds the algorithm OID; drives algorithm detection in
`AgentCryptographicVerifier`. Unchanged from current `agentPublicKey` column semantics.  
`keyRef` — `Base64URL(SHA-256(publicKey))`. Unchanged from current `agentKeyRef` semantics.

---

## Changes to Existing Runtime Classes

### `LedgerPemUtil` — new public method

`LedgerPemUtil` is currently package-private. The Vault Transit example needs to parse a PEM
public key from a string (not a file path). Add one public method:

```java
// New — parses a PEM-encoded public key from a string (e.g. a Vault Transit API response).
// Trial-loads through SUPPORTED_ALGORITHMS, same as loadPublicKey(String pemPath).
public static PublicKey parsePublicKey(String pemContent) throws Exception { ... }
```

`loadPublicKey(String pemPath)` (file path variant) stays unchanged and package-private.
`decodePem(String pem, String type)` stays package-private.
No other visibility changes to `LedgerPemUtil`.

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
- Eager `@PostConstruct` loading: parses all PEM files at startup, logs errors for failed actors
- Preserves the `failedActors` sentinel from `ConfiguredAgentKeyProvider`: actors whose key
  failed to load at startup are tracked in `Set<String> failedActors`. `sign()` logs a single
  warning at startup for each failed actor and returns `Optional.empty()` on subsequent calls
  — no per-call logging storm for misconfigured actors.
- Signs locally via `AgentSignature.signWith(keyPair, data)` — algorithm-transparent
- `SigningKey` deleted; `KeyPair` cached directly in `Map<String, KeyPair>`

### Deleted

- `AgentKeyProvider` — deleted
- `SigningKey` — deleted

---

## New Runtime Class: `AbstractCachingAgentSigner<C>`

Abstract base for external providers that have per-call overhead (network, hardware).

```
C = per-actorId context type
    Extractable-key providers (TUF, Vault KV): C = KeyPair
    Remote-signing providers (Vault Transit):  C = VaultTransitContext(pubKey, keyRef, keyName)
```

**`loadContext` signature:**

```java
/**
 * Loads the signing context for {@code actorId}.
 *
 * Return {@link Optional#empty()} if this actorId is not configured for signing —
 * the result is cached and no further calls are made for this actor.
 *
 * Throw {@link RuntimeException} for transient failures (network, auth) —
 * the failure is NOT cached; the next {@code sign()} call retries.
 */
protected abstract Optional<C> loadContext(String actorId);
```

Returning `Optional<C>` (not `null`) makes the "not configured" vs "transient error"
distinction explicit at the type level. A `null` return would be ambiguous.

**Cache semantics:**

The cache is `ConcurrentHashMap<String, Optional<C>>`:
- `map.get(actorId) == null` → not yet loaded; `loadContext` will be called
- `map.get(actorId) == Optional.empty()` → loaded, actor not configured; no further attempts
- `map.get(actorId).isPresent()` → loaded, actor has context; use it directly

Load failure (throw from `loadContext`) is not cached — the entry stays absent from the map so
the next call retries.

**Concurrent load trade-off:** the implementation uses `putIfAbsent`, not
`computeIfAbsent`. Two threads hitting the same unconfigured actor simultaneously both call
`loadContext` — for a Vault HTTP call that is two network round-trips. `computeIfAbsent` would
prevent the duplicate, but it blocks the map bucket for the duration of the compute function
and has reentrancy constraints that make it unsafe with slow external calls. The duplicate-load
cost on cold start (typically one round-trip per actor, bounded by actor count) is preferable
to the bucket-blocking risk.

**Template method:**

```java
protected abstract AgentSignature performSign(String actorId, C context, byte[] data);
```

**Cache management hooks (subclasses call from `@Scheduled` or rotation event observers):**

```java
protected void invalidateAll()               // full cache clear — triggers reload on next sign()
protected void invalidate(String actorId)    // single-actor eviction
```

No CDI annotations on the abstract class. Subclasses provide `@ApplicationScoped`.

---

## New Example: `examples/vault-transit-signing/`

Standalone Maven module (same structure as `examples/order-processing/`).

### What it demonstrates

Remote signing via Vault Transit Secrets Engine: private key never leaves Vault.
Signing happens via a Vault API call; the application holds only the public key.

### `VaultTransitAgentSigner`

```java
@ApplicationScoped
@Alternative
@Priority(1)
public class VaultTransitAgentSigner extends AbstractCachingAgentSigner<VaultTransitContext> { ... }
```

`VaultTransitContext` (private record): `vaultKeyName`, `publicKey` (bytes), `keyRef`.

**`loadContext(actorId)` — returns `Optional<C>`:**
1. Look up Vault key name from config mapping (`casehub.ledger.vault-transit.key-mapping."<actorId>"`)
   — return `Optional.empty()` immediately if actorId not in config (not configured, do not contact Vault)
2. `GET /v1/transit/keys/<key-name>` with `X-Vault-Token`
3. Parse public key from JSON response using `LedgerPemUtil.parsePublicKey(String)` (new public method)
4. Compute `keyRef = Base64URL(SHA-256(publicKey.getEncoded()))`
5. Return `Optional.of(new VaultTransitContext(keyName, pubKey.getEncoded(), keyRef))`

**`performSign(actorId, ctx, data)`:**
1. `POST /v1/transit/sign/<key-name>` body `{ "input": "<base64(data)>" }`
2. Parse `data.signature` from JSON response
3. Strip `vault:v1:` prefix; decode base64 → raw signature bytes
4. Return `new AgentSignature(rawBytes, ctx.publicKey(), ctx.keyRef())`

**Signature format note:** Vault Transit ed25519 signatures are 64 raw bytes after base64
decode. Vault Transit ECDSA (ecdsa-p256) signatures are ASN.1 DER-encoded — the same format
JCA's `Signature.verify()` expects for ECDSA, so no conversion is needed. Future adapters for
ECDSA keys work without format translation.

**Algorithm transparency:** The Vault key type (ed25519, ecdsa-p256, rsa-2048) is encoded in
the public key's X.509 OID bytes stored on the ledger entry. `AgentCryptographicVerifier`
detects the algorithm from those bytes. No algorithm string appears in `VaultTransitAgentSigner`.

**Scheduled cache refresh:**
```java
@Scheduled(every = "${casehub.ledger.vault-transit.refresh-interval:5m}")
void refresh() { invalidateAll(); }
```

This gives bounded-staleness refresh — new keys take effect within the interval. For
zero-latency rotation see #103.

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
- actorId not in config: returns `Optional.empty()` without contacting Vault
- Vault 403: `loadContext` throws; enricher swallows; entry unsigned
- Vault 500: `loadContext` throws; not cached; retry succeeds on second call

---

## Test Updates (existing runtime tests)

### `AgentSignatureEnricherTest`

Lambda construction changes (SAM, no annotation required):
```java
// was: actorId -> Optional.of(SigningKey.of(kp))
(actorId, data) -> Optional.of(AgentSignature.signWith(testKeyPair, data))
```
Test assertions unchanged.

`signatureVerifiesAgainstStoredPublicKey` — JCA verification using `Signature.getInstance()`
stays; it explicitly confirms the SPI contract produces a verifiable signature.

### `AgentSigningIT`

```java
@InjectMock AgentSigner agentSigner;  // was AgentKeyProvider

// Catch-all first, specific override second — simpler than argThat ordering
when(agentSigner.sign(anyString(), any())).thenReturn(Optional.empty());
when(agentSigner.sign(eq("claude:reviewer@v1"), any()))
    .thenAnswer(inv -> Optional.of(AgentSignature.signWith(testKeyPair, inv.getArgument(1))));
```

### `AgentSignatureVerificationServiceIT`

Heavy user of `SigningKey`. All `SigningKey.of(kp)` calls replaced:
- `SigningKey.of(kp).keyRef()` → inline computation: `Base64.getUrlEncoder().withoutPadding().encodeToString(MessageDigest.getInstance("SHA-256").digest(kp.getPublic().getEncoded()))`. Or extract to a test helper `keyRef(KeyPair)`.
- `SigningKey sk = SigningKey.of(...)` → replaced with `KeyPair` directly; use `AgentSignature.signWith()` where signing is needed; compute `keyRef` inline or via helper.

No mock of `AgentKeyProvider` in this test — verification reads from stored entry fields, not from the signer SPI. No `@InjectMock` changes needed here.

### `ReactiveAgentSignatureVerificationServiceIT`

- `@InjectMock AgentKeyProvider agentKeyProvider` → `@InjectMock AgentSigner agentSigner`
- Same `SigningKey` replacement pattern as above

### New: `AbstractCachingAgentSignerTest`

Pure unit test with a `TestSigner extends AbstractCachingAgentSigner<String>`:
- Cache hit: `loadContext` called once; `performSign` called twice for same actorId
- `Optional.empty()` returned and cached for unconfigured actors (null from `loadContext`)
- Throw from `loadContext` not cached (retry path): second call re-invokes `loadContext`
- `invalidateAll()` forces `loadContext` to be called again on next `sign()`
- `invalidate(actorId)` evicts only that actor; others remain cached

---

## ADR

New ADR extends (does not supersede) ADR 0011. ADR 0011's per-actorId key model is correct
and unchanged — the new ADR replaces the SPI contract that implements it. Records:

- Why signing belongs in the SPI (remote-signing models require it; `AgentKeyProvider`'s
  `KeyPair` return type assumes local JCA signing)
- Why `AgentKeyProvider` → `AgentSigner` is the minimal correct break
- The two-tier structure: `ConfiguredAgentSigner` (eager, direct) vs `AbstractCachingAgentSigner<C>` (lazy, cached)
- `loadContext() → Optional<C>` semantics vs null convention — explicit type-level contract
- Vault Transit as the reference implementation for the remote-signing pattern
- Deferred: Cloud KMS adapters (#102), rotation-triggered invalidation (#103),
  InMemoryAgentSigner (#104)

---

## Deferred (filed as issues)

| Issue | Description |
|-------|-------------|
| #101 | Vault AppRole / OIDC auth for `VaultTransitAgentSigner` |
| #102 | Cloud KMS adapters (AWS KMS, GCP KMS, Azure Key Vault) |
| #103 | Rotation-triggered cache invalidation via CDI event (true hot swap) |
| #104 | `InMemoryAgentSigner` in `persistence-memory/` |

---

## Platform Coherence Check

- **SPI types:** `AgentSigner` uses only `byte[]` and `String` — no SDK types in contract. ✅
- **Algorithm transparency:** protocol PP-20260523-e7b577 — no hardcoded algorithm strings. `AgentSignature.signWith()` derives from key; `AgentCryptographicVerifier` detects from X.509 bytes; `VaultTransitAgentSigner` stores raw bytes with no algorithm assumption. ✅
- **CDI pattern:** `ConfiguredAgentSigner @DefaultBean`; external providers `@Alternative @Priority(1)` — Pattern B from `alternative-extension-patterns.md`. ✅
- **Example module structure:** follows existing `examples/` conventions (standalone Quarkus app, own pom, WireMock for I/O). ✅
- **No platform-level doc update needed:** this is entirely within `casehub-ledger`'s existing ownership of agent signing. ✅
