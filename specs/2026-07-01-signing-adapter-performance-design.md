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

### API call reduction

| Path | Before | After |
|------|--------|-------|
| GCP `loadContext()` | 2 calls (`getPublicKey` + `getCryptoKeyVersion`) | 1 call (`getPublicKey`) |
| GCP `sign()` | 2 calls (`getCryptoKeyVersion` + `asymmetricSign`) | 1 call (`asymmetricSign`) |
| Azure `loadContext()` | 1 call (`getKey`) | 1 call (`getKey`) |
| Azure `sign()` | 2 calls (`getKey` + `sign`) | 1 call (`sign`) |

GCP `loadContext()` currently calls `fetchPublicKey()` (→ `getPublicKey()` API) and
then `fetchAlgorithm()` (→ `getCryptoKeyVersion()` API). The `getPublicKey()` response
already carries the algorithm at `pubKeyResp.getAlgorithm()` — it's validated for EC
but then discarded, only to be re-fetched via a separate API call. After the change,
`fetchPublicKey()` returns `GcpKmsContext` with the algorithm included, eliminating the
second call.

## Design

Follow the AWS pattern: pure Java `sign()` accepts metadata from cached context.
Clients stay stateless. Caching remains the CDI adapter's responsibility.

ASSUMPTION: Azure SDK `KeyClient`, `CryptographyClient`, and `DefaultAzureCredential`
are thread-safe and designed for concurrent reuse from a shared singleton.

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
- Constructor: no-arg creates `DefaultGcpKmsClientWrapper` internally (stays
  package-private). Wrapper constructor for test injection. Config parameter removed
  from both — unused by the client.

**`GcpKmsClientWrapper`:**
- Remove `getCryptoKeyVersion()` method. No caller needs it.

**`GcpKmsAgentSigner`:**
- `loadContext()` simplifies to a single `client.fetchPublicKey()` call returning the
  full context. Removes `fetchAlgorithm()` private method — the algorithm now comes
  from the `fetchPublicKey()` response.
- `performSign()` passes `context.algorithm()` to `client.sign()`.

### Azure Key Vault

**`AzureKeyVaultSigningClient`:**
- `sign(vaultUrl, keyName, algorithm, data)` accepts `SignatureAlgorithm` from context.
  No `getKey()` re-fetch. Sign path: compute digest → derive componentSize from
  algorithm → wrapper.sign() → rawToDer(). One API call instead of two.
- `fetchPublicKey()` stores `SignatureAlgorithm` in context using existing
  `mapCurveToAlgorithm(ECPublicKey)`. `computeComponentSize()` moves out of the
  fetch path — componentSize is derived from algorithm inside `sign()` where it's
  needed for DER conversion.
- Constructor: no-arg creates `DefaultAzureKeyVaultClientWrapper` internally.
  Wrapper constructor for test injection. Config parameter removed — unused by the
  client.

**`AzureKeyVaultContext`:**
- Replace `componentSize` field with `SignatureAlgorithm algorithm`. The algorithm
  is the primary abstraction; componentSize is a DER encoding detail derived inside
  `sign()`.

**`DefaultAzureKeyVaultClientWrapper`:**
- Cache `KeyClient` per vault URL in `ConcurrentHashMap<String, KeyClient>`.
- Cache `CryptographyClient` per key identifier in
  `ConcurrentHashMap<String, CryptographyClient>`.
- Create `DefaultAzureCredential` once in constructor — it's designed for reuse;
  per-call construction scans multiple credential providers each time.

**`AzureKeyVaultAgentSigner`:**
- `performSign()` passes `context.vaultUrl()`, `context.keyName()`,
  `context.algorithm()` to `client.sign()`.

### Not changed

**AWS KMS** — already correct. `sign(keyArn, data, signingAlgorithm)` accepts
algorithm from context. `KmsClient` created once in constructor.

**Vault Transit** — Ed25519 has no algorithm choice at sign time. `HttpClient`
created once in constructor. No per-call waste.

## Impact

Breaking changes to pure Java client `sign()` signatures. No external consumers
exist (platform has no end users). Migration is mechanical: callers pass context
fields instead of relying on internal re-fetches.

### Test migration

**GCP — `GcpKmsSigningClientTest`:**
- `sign_p256Key_usesCorrectDigest`: remove `getCryptoKeyVersion()` mock, pass
  algorithm to `sign()`.
- `sign_p384Key_usesCorrectDigest`: same — remove mock, pass algorithm.
- `sign_permissionDenied_throwsRuntimeException`: update `sign()` call signature to
  include algorithm parameter. (Note: this test currently never reaches
  `asymmetricSign` — `getCryptoKeyVersion()` returns null from an unmocked mock,
  causing NPE before the sign call. After the change, it will correctly test the
  permission denied path.)
- `fetchPublicKey_*` tests (4 methods): return type changes from `PublicKey` to
  `GcpKmsContext` — assertions change to check context fields.

**Azure — `AzureKeyVaultSigningClientTest`:**
- `signProducesDEREncodedSignature`: remove `wrapper.getKey()` mock from sign path,
  change `sign()` call to pass `SignatureAlgorithm`.
- `signThrowsExceptionOnServiceError`: same — remove mock, update call signature.
- `fetchPublicKey*` tests: assertions change from `componentSize` to `algorithm`.

**`GcpKmsClientWrapper`** loses `getCryptoKeyVersion()` — all mocks of this method
are removed. Net simpler: fewer mock stubs per test.

## Out of scope

- Vault AppRole/OIDC auth (#101) — separate issue
- Pre-existing `KeyDIDResolverTest` failure (doubled X.509 header bytes) — tracked
  as #166
