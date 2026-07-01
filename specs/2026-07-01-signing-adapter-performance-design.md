# Signing Adapter Performance — Design Spec

**Issue:** #164
**Date:** 2026-07-01
**Status:** Approved

## Problem

GCP and Azure pure Java signing clients re-fetch metadata on every `sign()` call
that the Quarkus adapter already cached during `loadContext()`. Azure additionally
creates new SDK clients (KeyClient, CryptographyClient, DefaultAzureCredential) on
every API call. AWS and Vault Transit don't have these problems.

The root cause is that the GCP and Azure `sign()` methods don't accept cached context
— they take only a key reference string and must re-derive algorithm/curve info from
the cloud API each time.

## Design

Follow the AWS pattern: pure Java `sign()` accepts metadata from cached context.
Clients stay stateless. Caching remains the CDI adapter's responsibility.

### GCP Cloud KMS

**`GcpKmsSigningClient`:**
- `fetchPublicKey(versionName)` returns `GcpKmsContext` (key + algorithm) instead of
  `PublicKey`. The `getPublicKey()` response already carries the algorithm — stop
  discarding it.
- `sign(versionName, algorithm, data)` accepts algorithm from context. Removes the
  `getCryptoKeyVersion()` call entirely. Sign path: compute digest → asymmetricSign.
  One API call instead of two.
- Remove `getWrapper()` getter — only existed for the adapter to reach
  `getCryptoKeyVersion()`.
- Constructor takes only the wrapper, not config. Config is unused by the client —
  the adapter owns key mapping lookup.

**`GcpKmsClientWrapper`:**
- Remove `getCryptoKeyVersion()` method. No caller needs it.

**`GcpKmsAgentSigner`:**
- `loadContext()` simplifies to a single `client.fetchPublicKey()` call returning the
  full context. Removes `fetchAlgorithm()` private method.
- `performSign()` passes `context.algorithm()` to `client.sign()`.

### Azure Key Vault

**`AzureKeyVaultSigningClient`:**
- `sign(vaultUrl, keyName, componentSize, data)` accepts metadata from context. No
  `getKey()` re-fetch. Sign path: derive algorithm from componentSize → compute
  digest → wrapper.sign(). One API call instead of two.
- New `mapComponentSizeToAlgorithm(int)` replaces `mapCurveToAlgorithm(ECPublicKey)`
  — derives algorithm from componentSize directly (32→ES256, 48→ES384, 66→ES512).
- Constructor takes only the wrapper, not config. Config is unused by the client.

**`DefaultAzureKeyVaultClientWrapper`:**
- Cache `KeyClient` per vault URL in `ConcurrentHashMap<String, KeyClient>`.
- Cache `CryptographyClient` per key identifier in
  `ConcurrentHashMap<String, CryptographyClient>`.
- Create `DefaultAzureCredential` once in constructor — it's designed for reuse;
  per-call construction scans multiple credential providers each time.

**`AzureKeyVaultAgentSigner`:**
- `performSign()` passes `context.vaultUrl()`, `context.keyName()`,
  `context.componentSize()` to `client.sign()`.

### Not changed

**AWS KMS** — already correct. `sign(keyArn, data, signingAlgorithm)` accepts
algorithm from context. `KmsClient` created once in constructor.

**Vault Transit** — Ed25519 has no algorithm choice at sign time. `HttpClient`
created once in constructor. No per-call waste.

## Impact

Breaking changes to pure Java client `sign()` signatures. No external consumers
exist (platform has no end users). Migration is mechanical: callers pass context
fields instead of relying on internal re-fetches.

`GcpKmsClientWrapper` loses `getCryptoKeyVersion()` — tests that mock this method
need updating. Net simpler: fewer mock stubs per test.

## Out of scope

- Vault AppRole/OIDC auth (#101) — separate issue
- Pre-existing `KeyDIDResolverTest` failure (doubled X.509 header bytes) — unrelated
  bug, file a separate issue
