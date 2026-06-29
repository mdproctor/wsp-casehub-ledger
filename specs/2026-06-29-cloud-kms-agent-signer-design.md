# Cloud KMS AgentSigner Adapters — Design Spec

**Issue:** casehubio/ledger#102
**Date:** 2026-06-29
**Status:** Approved

## Summary

Production-grade signing adapters for AWS KMS, GCP Cloud KMS, and Azure Key Vault,
plus promotion of the existing Vault Transit example to a first-class module. Each
adapter follows a two-layer architecture: pure Java signing client (framework-free)
with a Quarkus CDI integration layer on top.

## Scope

- Four signing providers: Vault Transit (promoted), AWS KMS, GCP Cloud KMS, Azure Key Vault
- Eight new modules under `signing/` (pure Java + Quarkus per provider)
- Verification infrastructure changes: EC algorithm support in `AgentCryptographicVerifier` and `LedgerPemUtil`
- `AgentSigner.keyMaterial()` SPI addition to avoid wasted cloud KMS sign calls
- Thin getting-started examples under `examples/` for each provider
- Follow-up issue for promoting other example capabilities (otel-trace-wiring, prov-dm-export, eigentrust-mesh, trust-score-routing)

**Scope deviation from issue #102:** The issue places adapters in `examples/` submodules.
This spec promotes them to `signing/` as first-class modules. Production signing
adapters should not live in `examples/` — they carry cloud SDK dependencies, require
CI testing, and will be Maven dependencies for real deployments. The `examples/`
directory retains thin getting-started samples that depend on the `signing/` modules.
Issue #102 should be updated to reflect this agreed scope.

## Module Structure

```
signing/
  vault-transit/              → io.casehub:casehub-ledger-vault-transit
  vault-transit-quarkus/      → io.casehub:casehub-ledger-vault-transit-quarkus
  aws-kms/                    → io.casehub:casehub-ledger-aws-kms
  aws-kms-quarkus/            → io.casehub:casehub-ledger-aws-kms-quarkus
  gcp-kms/                    → io.casehub:casehub-ledger-gcp-kms
  gcp-kms-quarkus/            → io.casehub:casehub-ledger-gcp-kms-quarkus
  azure-keyvault/             → io.casehub:casehub-ledger-azure-keyvault
  azure-keyvault-quarkus/     → io.casehub:casehub-ledger-azure-keyvault-quarkus
```

All eight modules are children of `casehub-ledger-parent`, gated behind a
`with-signing` Maven profile. Structure:

- `signing/pom.xml` — intermediate aggregator POM listing all eight modules
- `casehub-ledger-parent/pom.xml` — `<profile id="with-signing">` activates `signing/` as a module
- Consumers depend on the specific provider artifact (e.g., `casehub-ledger-aws-kms-quarkus`) — the profile controls build scope, not consumer dependency resolution
- Individual providers can be built in isolation via `mvn -pl signing/aws-kms` during development

**Package roots:**
- Pure Java: `io.casehub.ledger.signing.vault`, `.aws`, `.gcp`, `.azure`
- Quarkus: `io.casehub.ledger.signing.vault.quarkus`, `.aws.quarkus`, `.gcp.quarkus`, `.azure.quarkus`

**Dependencies:**
- Pure Java modules depend only on their cloud SDK — no dependency on any casehub-ledger module. They return `byte[]` signatures and `PublicKey` objects; the Quarkus adapter constructs `AgentSignature`
- Quarkus modules depend on their pure Java sibling + `casehub-ledger` runtime (for `AbstractCachingAgentSigner`, `AgentSignature`, `AgentKeyRotatedEvent`) + `quarkus-arc` + `quarkus-scheduler`

## Pure Java Layer

Each pure Java module contains three things:

### Signing client

The core class. Takes a config POJO in the constructor, provides two operations:

```java
public class AwsKmsSigningClient {
    public AwsKmsSigningClient(AwsKmsSigningConfig config) { ... }
    public PublicKey fetchPublicKey(String keyArn);
    public byte[] sign(String keyArn, byte[] data);
}
```

- No CDI, no Quarkus, no `AbstractCachingAgentSigner`
- Usable from Spring, Micronaut, or plain `main()`
- Handles provider-specific public key parsing internally (DER, PEM, or JWK depending on provider)

### Config POJO

Plain Java record — no framework annotations:

```java
public record AwsKmsSigningConfig(
    String region,
    Map<String, String> keyMapping  // actorId → KMS key ARN
) {}
```

Credentials are not in the config — cloud SDKs use their standard credential chains
(env vars, instance metadata, config files).

### Context record

Cached per-actorId by the Quarkus layer:

```java
public record AwsKmsContext(String keyArn, PublicKey publicKey) {}
```

## Quarkus Layer

Each Quarkus module wraps its pure Java sibling with CDI wiring. Three classes:

### Config mapping

Bridges Quarkus config to the pure Java POJO:

```java
@ConfigMapping(prefix = "casehub.ledger.aws-kms")
public interface AwsKmsConfig {
    @WithDefault("us-east-1")
    String region();
    Map<String, String> keyMapping();
    @WithDefault("5m")
    String refreshInterval();
}
```

### CDI adapter

Extends `AbstractCachingAgentSigner`, delegates to the pure Java client:

```java
@ApplicationScoped
@Alternative
@Priority(1)
public class AwsKmsAgentSigner extends AbstractCachingAgentSigner<AwsKmsContext> {

    private final AwsKmsSigningClient client;
    private final AwsKmsConfig config;

    @Inject
    public AwsKmsAgentSigner(AwsKmsConfig config) {
        this.config = config;
        this.client = new AwsKmsSigningClient(
            new AwsKmsSigningConfig(config.region(), config.keyMapping()));
    }

    @Override
    protected Optional<AwsKmsContext> loadContext(String actorId) {
        String keyArn = config.keyMapping().get(actorId);
        if (keyArn == null) return Optional.empty();
        PublicKey pub = client.fetchPublicKey(keyArn);
        return Optional.of(new AwsKmsContext(keyArn, pub));
    }

    @Override
    protected AgentSignature performSign(String actorId, AwsKmsContext ctx, byte[] data) {
        byte[] sigBytes = client.sign(ctx.keyArn(), data);
        byte[] pubEncoded = ctx.publicKey().getEncoded();
        String keyRef = computeKeyRef(pubEncoded);
        return new AgentSignature(sigBytes, pubEncoded, keyRef);
    }

    @Scheduled(every = "${casehub.ledger.aws-kms.refresh-interval:5m}")
    void refreshCache() { invalidateAll(); }

    @Observes
    public void onKeyRotated(AgentKeyRotatedEvent event) {
        super.onKeyRotated(event);
    }
}
```

### keyRef computation

`Base64URL(SHA-256(publicKey))` — used by all four Quarkus adapters. Already exists as
inline code in `AgentSignature.signWith()` and the Vault adapter. With four adapters now,
extract to a `public static` utility method on `AgentSignature` (in the `runtime` module —
Quarkus adapter modules depend on `casehub-ledger` runtime, so this is accessible):

```java
public static String computeKeyRef(byte[] publicKeyEncoded)
```

All adapters and `signWith()` delegate to this single method.

### AgentSigner.keyMaterial() — key-only retrieval

`AgentEntrySigner.prepareKey()` currently calls `sign(actorId, new byte[0])` to
obtain key material. For cloud KMS adapters, this triggers a paid signing API call
whose result is discarded — each entry save incurs one wasted KMS sign call. Add a
`default` method to the `AgentSigner` SPI for key-only retrieval:

```java
default Optional<AgentKeyMaterial> keyMaterial(String actorId) {
    return sign(actorId, new byte[0])
            .map(sig -> new AgentKeyMaterial(sig.publicKey(), sig.keyRef()));
}
```

**`AgentKeyMaterial` record** — top-level in `io.casehub.ledger.runtime.service`,
consistent with `AgentSignature`. Defensive copy on `publicKey` per platform
value-type convention:

```java
public record AgentKeyMaterial(byte[] publicKey, String keyRef) {
    public AgentKeyMaterial {
        Objects.requireNonNull(keyRef, "keyRef must not be null");
        publicKey = publicKey.clone();
    }
}
```

`AgentEntrySigner.prepareKey()` calls `keyMaterial()` instead of `sign()`.
`ConfiguredAgentSigner` uses the default (local JCA signing is free). Per protocol
`spi-evolution-default-methods`, this is a backward-compatible SPI evolution via
`default` method.

**`AbstractCachingAgentSigner` override** — two additions enable
`keyMaterial()` to return cached key material without calling `performSign()`:

1. **`resolveContext(String actorId)`** — `protected` helper factored out of
   `sign()`, encapsulates the cache-lookup-or-load logic. Both `sign()` and
   `keyMaterial()` call it, eliminating duplication:

```java
protected Optional<C> resolveContext(String actorId) {
    Optional<C> cached = contextCache.get(actorId);
    if (cached == null) {
        final Optional<C> loaded = loadContext(actorId);
        final Optional<C> racing = contextCache.putIfAbsent(actorId, loaded);
        cached = racing != null ? racing : loaded;
    }
    return cached;
}
```

2. **`contextPublicKey(C context)`** — new abstract method returning the
   `PublicKey` held by the provider-specific context type. Each adapter
   implements it as a one-liner (e.g., `return ctx.publicKey()`):

```java
protected abstract PublicKey contextPublicKey(C context);
```

The concrete `keyMaterial()` override assembles `AgentKeyMaterial` from the cached
context without triggering `performSign()` — zero KMS API calls for key retrieval:

```java
@Override
public Optional<AgentKeyMaterial> keyMaterial(String actorId) {
    return resolveContext(actorId).map(ctx -> {
        byte[] pub = contextPublicKey(ctx).getEncoded();
        return new AgentKeyMaterial(pub, AgentSignature.computeKeyRef(pub));
    });
}
```

### Consumer activation

```properties
quarkus.arc.selected-alternatives=io.casehub.ledger.signing.aws.quarkus.AwsKmsAgentSigner
```

## Provider-Specific Details

### AWS KMS
- **SDK:** `software.amazon.awssdk:kms`
- **Sign:** `kmsClient.sign(SignRequest)` — adapter maps key spec to signing algorithm:
  - `ECC_NIST_P256` → `ECDSA_SHA_256`
  - `ECC_NIST_P384` → `ECDSA_SHA_384`
  - `ECC_NIST_P521` → `ECDSA_SHA_512`
- **Supported key types:** EC only (P-256, P-384, P-521). RSA key specs are rejected at `loadContext()` time
- **Public key:** `kmsClient.getPublicKey()` → DER-encoded `SubjectPublicKeyInfo` bytes. Feed to `KeyFactory.getInstance("EC")`
- **Auth:** Default credential provider chain (env vars, `~/.aws/credentials`, instance metadata, ECS task role)
- **Signature format:** DER-encoded. Store as-is
- **Key type validation:** `loadContext()` reads `GetPublicKeyResponse.keySpec()` before EC parsing. Non-EC key specs (e.g., `RSA_2048`) → log ERROR ("key {arn} is {keySpec} but this adapter requires EC") and return `Optional.empty()` (cached as absent — permanent misconfiguration, no retry)
- **Error classification:** `NotFoundException` → return `Optional.empty()`; `KmsException` with 401/403 → throw (transient — credential refresh may resolve); network timeout, throttling (429), 5xx → throw (transient)

### GCP Cloud KMS
- **SDK:** `com.google.cloud:google-cloud-kms`
- **Sign:** `asymmetricSign(CryptoKeyVersionName, Digest)` — adapter reads `CryptoKeyVersion.algorithm` and selects the matching digest:
  - `EC_SIGN_P256_SHA256` → SHA-256 digest
  - `EC_SIGN_P384_SHA384` → SHA-384 digest
- **Supported key types:** EC only. RSA and secp256k1 key algorithms are rejected at `loadContext()` time
- **Public key:** `getPublicKey()` → PEM string. Strip headers, base64-decode, feed to `KeyFactory.getInstance("EC")`
- **Auth:** Application Default Credentials (`GOOGLE_APPLICATION_CREDENTIALS` or GCE metadata)
- **Signature format:** DER-encoded ASN.1 for EC keys. Store as-is
- **Key type validation:** `loadContext()` reads `CryptoKeyVersion.algorithm` before EC parsing. Non-EC algorithms (e.g., `RSA_SIGN_PKCS1_2048_SHA256`, `CRYPTO_KEY_VERSION_ALGORITHM_UNSPECIFIED`) → log ERROR ("key {versionName} is {algorithm} but this adapter requires EC") and return `Optional.empty()` (cached as absent — permanent misconfiguration, no retry)
- **Error classification:** `NOT_FOUND` → return `Optional.empty()`; `PERMISSION_DENIED`/`UNAUTHENTICATED` → throw (transient — token refresh may resolve); `UNAVAILABLE`/`DEADLINE_EXCEEDED`/`RESOURCE_EXHAUSTED` → throw (transient)

### Azure Key Vault
- **SDK:** `com.azure:azure-security-keyvault-keys`
- **Sign:** `CryptographyClient.sign(SignatureAlgorithm, byte[])` — adapter maps curve to algorithm and computes the appropriate digest:
  - P-256 → `ES256` (SHA-256 digest)
  - P-384 → `ES384` (SHA-384 digest)
  - P-521 → `ES512` (SHA-512 digest)
- **Supported key types:** EC only. RSA keys are rejected at `loadContext()` time
- **Public key:** `KeyClient.getKey()` → `JsonWebKey`. Call `toEc()` for JCA `ECPublicKey`
- **Auth:** `DefaultAzureCredential` (env vars, managed identity, Azure CLI)
- **Signature format:** Raw R‖S bytes for EC keys — **needs conversion to DER before storage** so `AgentCryptographicVerifier` can verify. Component sizes are curve-dependent (P-256: 32 bytes each, P-384: 48, P-521: 66). The pure Java client splits R‖S at the midpoint, constructs DER-encoded ASN.1 `SEQUENCE { INTEGER r, INTEGER s }` with appropriate leading-zero handling via `BigInteger`. No BouncyCastle dependency required
- **Key type validation:** `loadContext()` reads `JsonWebKey.getKeyType()` before EC parsing. Non-EC key types (e.g., `RSA`, `RSA-HSM`) → log ERROR ("key {keyId} is {keyType} but this adapter requires EC") and return `Optional.empty()` (cached as absent — permanent misconfiguration, no retry)
- **Error classification:** `ResourceNotFoundException` → return `Optional.empty()`; 401/403 → throw (transient — credential refresh may resolve); network timeout, 429, 5xx → throw (transient)

### Vault Transit (promoted from examples/)
- **No SDK** — `java.net.http.HttpClient` + Jackson
- **Sign:** `POST /v1/transit/sign/<keyName>` → strip `vault:v1:` prefix, base64-decode
- **Public key:** `GET /v1/transit/keys/<keyName>` → PEM in response JSON. Parse the `keys` map and select the **highest-numbered version** (existing code bug: `keys.fields().next()` returns version 1, the oldest — see Vault Transit Promotion below)
- **PEM parsing:** The pure Java client includes its own PEM parsing (strip headers, base64-decode, `KeyFactory.getInstance("Ed25519")`). It does NOT depend on `LedgerPemUtil` — pure Java modules have no casehub-ledger dependency. The direct `KeyFactory` call (no trial-load) is correct: Vault Transit is Ed25519-only, and the provider-specific package `io.casehub.ledger.signing.vault` is outside protocol PP-20260523-e7b577's `applies_to` scope (`io.casehub.ledger.runtime.service`). Algorithm transparency belongs in the shared verification infrastructure, not in provider-specific clients
- **Supported key types:** Ed25519 only (existing scope; no verification infrastructure changes needed)
- **Auth:** `X-Vault-Token` header (static token). #101 adds AppRole/OIDC
- **Signature format:** Raw Ed25519 bytes. Store as-is
- **Error classification:** HTTP 404 (key not found) → return `Optional.empty()`; HTTP 403 (token revoked/expired) → throw (transient); network timeout, 5xx → throw (transient)

## Verification Infrastructure Changes

Cloud KMS adapters produce EC signatures (P-256, P-384, P-521). The existing
verification infrastructure (`AgentCryptographicVerifier`, `LedgerPemUtil`) supports
only Ed25519 and ML-DSA. Two changes are required per protocol PP-20260523-e7b577:

### Add EC to the supported-algorithm trial list

Both `AgentCryptographicVerifier.SUPPORTED_ALGORITHMS` and
`LedgerPemUtil.SUPPORTED_ALGORITHMS` gain `"EC"`:

```java
private static final List<String> SUPPORTED_ALGORITHMS =
    List.of("Ed25519", "EC", "ML-DSA-44", "ML-DSA-65", "ML-DSA-87");
```

### EC signature algorithm mapping

`AgentCryptographicVerifier.verifyCryptographic()` currently calls
`Signature.getInstance(pub.getAlgorithm())`. For Ed25519 and ML-DSA,
`pub.getAlgorithm()` returns a valid `Signature` algorithm name. For EC keys,
`pub.getAlgorithm()` returns `"EC"` — not a valid `Signature` algorithm. A mapping
function derives the correct algorithm from the EC key's curve parameters:

| Curve | Order bit length | Signature algorithm |
|---|---|---|
| P-256 (secp256r1) | 256 | `SHA256withECDSA` |
| P-384 (secp384r1) | 384 | `SHA384withECDSA` |
| P-521 (secp521r1) | 521 | `SHA512withECDSA` |

The curve information is embedded in the X.509 SubjectPublicKeyInfo — the mapping
is deterministic from the key material, consistent with the algorithm-transparency
protocol.

### RSA is out of scope

Cloud KMS adapters are restricted to EC keys. RSA signature algorithms are not
deterministic from the key material — the hash and padding scheme are signing-time
choices not encoded in the public key. Supporting RSA would require either storing
the signature algorithm alongside the signature (schema change) or trial-verifying
with multiple algorithms (fragile). EC is universally available across all three
cloud providers and is the natural complement to the platform's Ed25519 default.

## Vault Transit Promotion

The existing `examples/vault-transit-signing/` splits into two modules:

| Current file | Destination | Changes |
|---|---|---|
| `VaultTransitAgentSigner.java` | Split into `VaultTransitSigningClient` (pure Java) + `VaultTransitAgentSigner` (Quarkus) | HTTP/JSON logic → client; CDI/config/caching → adapter |
| `VaultTransitContext.java` | `signing/vault-transit/` | Package rename only |
| `VaultTransitConfig.java` | `signing/vault-transit-quarkus/` | Package rename only |
| `VaultTransitAgentSignerIT.java` | Split — client unit test + Quarkus integration test | |

**Bug fixes during promotion:**

1. **`fetchPublicKey` version selection** — existing code uses `keys.fields().next().getValue()` which returns version 1 (the oldest). After key rotation, Vault signs with the latest version but the stored public key is from version 1 → verification fails. Fix: iterate the `keys` map and select the entry with the highest integer key.

2. **Missing `AgentKeyRotatedEvent` observer** — `AbstractCachingAgentSigner` Javadoc requires concrete CDI subclasses to expose `onKeyRotated()` as a CDI observer. The existing Vault Transit example omits this. The promoted Quarkus adapter adds the `@Observes` method (consistent with the spec's CDI adapter pattern for all four providers).

The old `examples/vault-transit-signing/` becomes a thin getting-started example that
depends on `casehub-ledger-vault-transit-quarkus`. Contains only `application.properties`
showing configuration and a minimal test demonstrating usage.

## Testing Strategy

Three test levels:

### SPI contract test

Per protocol `spi-default-method-contract-test` (PP-20260513-2ce9e1), an anonymous
`AgentSigner` implementation verifies the `keyMaterial()` default method contract:

```java
AgentSigner anon = (actorId, data) ->
    Optional.of(new AgentSignature(data, new byte[]{1}, "ref"));
assertThat(anon.keyMaterial("actor")).isPresent();
assertThat(anon.keyMaterial("actor").get().keyRef()).isEqualTo("ref");
```

Plus negative case: `sign()` returns empty → `keyMaterial()` returns empty. No
existing `AgentSignerContractTest` exists — this spec creates one in
`runtime/src/test/java`.

### Pure Java client tests (unit tests, no CDI)
- WireMock for Vault Transit (HTTP API)
- AWS: `KmsClient` is a Java interface — Mockito mock directly
- GCP: `KeyManagementServiceClient` is a concrete class — WireMock at the HTTP/gRPC level (consistent with Vault Transit approach, tests full SDK construction path)
- Azure: `CryptographyClient` and `KeyClient` are concrete classes — WireMock at the HTTP level
- Coverage: successful sign, successful public key fetch, key not found (empty), transient failure (RuntimeException), malformed response, wrong key type (RSA → permanent error)
- Azure: explicit test for R‖S → DER conversion correctness

### Quarkus integration tests (`@QuarkusTest`)
- WireMock or mocked SDK injected via Quarkus test config
- `casehub-ledger-memory` as test dependency
- Coverage: full save pipeline (entry persisted with signature), cache invalidation on `AgentKeyRotatedEvent`, scheduled refresh, unconfigured actorId produces unsigned entry
- One round-trip test per provider: sign then verify with `AgentCryptographicVerifier`

No Docker required. No real cloud credentials in CI.

## Examples

Each provider gets a thin example under `examples/`:

```
examples/
  vault-transit-signing/    (existing — slimmed to consumer-only)
  aws-kms-signing/          (new)
  gcp-kms-signing/          (new)
  azure-keyvault-signing/   (new)
```

Each contains:
- Standalone `pom.xml` depending on the `-quarkus` module
- `application.properties` with WireMock/localstack-friendly defaults
- A single `@QuarkusTest` demonstrating configuration and a signed entry
- `README.md` with IAM/RBAC permission requirements for production use

## Known Limitations

- **Single active signing provider:** All four Quarkus adapters use `@Alternative @Priority(1)`. Only one can be active per deployment — selecting two produces an ambiguous `AgentSigner` dependency. A deployment routes all actors through the same provider. Multi-provider routing (e.g., some actors to AWS KMS, others to Vault) requires a composite `AgentSigner` that dispatches by actorId — a different design, out of scope for this spec.

- **RSA keys not supported:** Cloud KMS adapters are restricted to EC keys. RSA signature algorithms are not deterministic from the key material. See Verification Infrastructure Changes above.

## Follow-Up

File a separate issue for promoting other example capabilities to first-class modules
using the same pattern (capability module + thin example):
- `examples/otel-trace-wiring/`
- `examples/prov-dm-export/`
- `examples/eigentrust-mesh/`
- `examples/trust-score-routing/`
