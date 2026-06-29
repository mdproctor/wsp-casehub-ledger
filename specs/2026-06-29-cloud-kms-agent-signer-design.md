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
- Thin getting-started examples under `examples/` for each provider
- Follow-up issue for promoting other example capabilities (otel-trace-wiring, prov-dm-export, eigentrust-mesh, trust-score-routing)

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
`with-signing` Maven profile.

**Package roots:**
- Pure Java: `io.casehub.ledger.signing.vault`, `.aws`, `.gcp`, `.azure`
- Quarkus: `io.casehub.ledger.signing.vault.quarkus`, `.aws.quarkus`, `.gcp.quarkus`, `.azure.quarkus`

**Dependencies:**
- Pure Java modules depend on `casehub-ledger-api` (for `AgentSignature`) + cloud SDK
- Quarkus modules depend on their pure Java sibling + `casehub-ledger` runtime (for `AbstractCachingAgentSigner`, `AgentKeyRotatedEvent`) + `quarkus-arc` + `quarkus-scheduler`

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
extract to a `public static` utility method on `AgentSignature`:

```java
public static String computeKeyRef(byte[] publicKeyEncoded)
```

All adapters and `signWith()` delegate to this single method.

### Consumer activation

```properties
quarkus.arc.selected-alternatives=io.casehub.ledger.signing.aws.quarkus.AwsKmsAgentSigner
```

## Provider-Specific Details

### AWS KMS
- **SDK:** `software.amazon.awssdk:kms`
- **Sign:** `kmsClient.sign(SignRequest)` — signing algorithm determined by KMS key type
- **Public key:** `kmsClient.getPublicKey()` → DER-encoded `SubjectPublicKeyInfo` bytes. Feed to `KeyFactory` with algorithm from `KeySpec`
- **Auth:** Default credential provider chain (env vars, `~/.aws/credentials`, instance metadata, ECS task role)
- **Signature format:** DER-encoded. Store as-is

### GCP Cloud KMS
- **SDK:** `com.google.cloud:google-cloud-kms`
- **Sign:** `asymmetricSign(CryptoKeyVersionName, Digest)` — caller computes SHA-256 digest first
- **Public key:** `getPublicKey()` → PEM string. Strip headers, base64-decode, feed to `KeyFactory`
- **Auth:** Application Default Credentials (`GOOGLE_APPLICATION_CREDENTIALS` or GCE metadata)
- **Signature format:** DER-encoded ASN.1 for EC keys. Store as-is

### Azure Key Vault
- **SDK:** `com.azure:azure-security-keyvault-keys`
- **Sign:** `CryptographyClient.sign(SignatureAlgorithm, byte[])` — takes raw digest bytes
- **Public key:** `KeyClient.getKey()` → `JsonWebKey`. Call `toEc()`/`toRsa()` for JCA `PublicKey`
- **Auth:** `DefaultAzureCredential` (env vars, managed identity, Azure CLI)
- **Signature format:** Raw R‖S bytes for EC keys — **needs conversion to DER before storage** so `AgentCryptographicVerifier` can verify. Pure Java client handles conversion internally

### Vault Transit (promoted from examples/)
- **No SDK** — `java.net.http.HttpClient` + Jackson
- **Sign:** `POST /v1/transit/sign/<keyName>` → strip `vault:v1:` prefix, base64-decode
- **Public key:** `GET /v1/transit/keys/<keyName>` → PEM in response JSON
- **Auth:** `X-Vault-Token` header (static token). #101 adds AppRole/OIDC
- **Signature format:** Raw Ed25519 bytes. Store as-is

## Vault Transit Promotion

The existing `examples/vault-transit-signing/` splits into two modules:

| Current file | Destination | Changes |
|---|---|---|
| `VaultTransitAgentSigner.java` | Split into `VaultTransitSigningClient` (pure Java) + `VaultTransitAgentSigner` (Quarkus) | HTTP/JSON logic → client; CDI/config/caching → adapter |
| `VaultTransitContext.java` | `signing/vault-transit/` | Package rename only |
| `VaultTransitConfig.java` | `signing/vault-transit-quarkus/` | Package rename only |
| `VaultTransitAgentSignerIT.java` | Split — client unit test + Quarkus integration test | |

The old `examples/vault-transit-signing/` becomes a thin getting-started example that
depends on `casehub-ledger-vault-transit-quarkus`. Contains only `application.properties`
showing configuration and a minimal test demonstrating usage.

## Testing Strategy

Two test levels per provider, matching the two layers:

### Pure Java client tests (unit tests, no CDI)
- WireMock for Vault (HTTP API)
- Mocked SDK clients for cloud providers (AWS `KmsClient` is an interface; GCP/Azure similarly)
- Coverage: successful sign, successful public key fetch, key not found (empty), transient failure (RuntimeException), malformed response
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

## Follow-Up

File a separate issue for promoting other example capabilities to first-class modules
using the same pattern (capability module + thin example):
- `examples/otel-trace-wiring/`
- `examples/prov-dm-export/`
- `examples/eigentrust-mesh/`
- `examples/trust-score-routing/`
