# Cloud KMS AgentSigner Adapters — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Production-grade signing adapters for AWS KMS, GCP Cloud KMS, Azure Key Vault, and Vault Transit (promoted from examples/), with EC verification infrastructure and SPI evolution.

**Architecture:** Two-layer architecture per provider: pure Java signing client (no framework deps) + Quarkus CDI integration layer. Verification infrastructure gains EC support. `AgentSigner` SPI gains `keyMaterial()` to avoid wasted cloud KMS sign calls.

**Tech Stack:** Java 21, Quarkus 3.32.2, AWS SDK v2, Google Cloud KMS, Azure Key Vault Keys SDK, WireMock 3.4.2, JUnit 5, AssertJ

## Global Constraints

- Java 21 (`maven.compiler.release=21`), Quarkus 3.32.2
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn ...` for all builds
- Pure Java modules: ZERO casehub-ledger dependencies. Return `byte[]` and `PublicKey`, not `AgentSignature`
- Quarkus modules: depend on sibling pure Java + `casehub-ledger` runtime
- Cloud KMS: EC keys only. RSA is out of scope
- Algorithm transparency: never hardcode a cryptographic algorithm string (PP-20260523-e7b577)
- Protocol PP-20260523-e7b577 applies to `io.casehub.ledger.runtime.service` — provider-specific packages are outside its scope
- SPI evolution via `default` methods (protocol `spi-evolution-default-methods`)
- All JPQL against LedgerEntry must use `@NamedQuery` (protocol `ledger-entry-named-query`)
- LedgerEntry subclass repositories are read-only (protocol `ledger-subclass-repo-readonly`)
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all class lookups and navigation — never bash grep/find for classes
- TDD: write failing test first, then implement
- Every commit references issue #102: `Refs #102`
- Read the spec at `~/claude/public/casehub/ledger/specs/2026-06-29-cloud-kms-agent-signer-design.md` — it contains the review-hardened design with all edge cases resolved
- Read `docs/PLATFORM.md` (fetch from `https://raw.githubusercontent.com/casehubio/parent/main/docs/PLATFORM.md`) before implementing

---

### Task 1: EC Verification Infrastructure

Add EC algorithm support to the verification and signing infrastructure. Three callers of `Signature.getInstance(key.getAlgorithm())` break with EC keys because `ECKey.getAlgorithm()` returns `"EC"`, not a valid `Signature` algorithm. Fix all three with a shared `signatureAlgorithm(Key)` utility, and add `"EC"` to both `SUPPORTED_ALGORITHMS` lists.

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/SignatureAlgorithms.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifier.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignature.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerPemUtil.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerklePublisher.java`
- Create: `runtime/src/test/java/io/casehub/ledger/runtime/service/SignatureAlgorithmsTest.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifierTest.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/runtime/service/LedgerPemUtilTest.java`

**Interfaces:**
- Produces: `SignatureAlgorithms.signatureAlgorithm(java.security.Key) → String` — shared by AgentCryptographicVerifier, AgentSignature, LedgerMerklePublisher
- Produces: `LedgerPemUtil.SUPPORTED_ALGORITHMS` gains `"EC"` — enables EC key loading for ConfiguredAgentSigner

**Implementation details:**

`SignatureAlgorithms` is a package-private utility class in `io.casehub.ledger.runtime.service`:

```java
package io.casehub.ledger.runtime.service;

import java.security.Key;
import java.security.interfaces.ECKey;

final class SignatureAlgorithms {
    private SignatureAlgorithms() {}

    static String signatureAlgorithm(final Key key) {
        if (!"EC".equals(key.getAlgorithm())) return key.getAlgorithm();
        final ECKey ec = (ECKey) key;
        return switch (ec.getParams().getOrder().bitLength()) {
            case 256 -> "SHA256withECDSA";
            case 384 -> "SHA384withECDSA";
            case 521 -> "SHA512withECDSA";
            default -> throw new IllegalArgumentException(
                "Unsupported EC curve order: " + ec.getParams().getOrder().bitLength());
        };
    }
}
```

Both `SUPPORTED_ALGORITHMS` lists gain `"EC"` after `"Ed25519"`:
```java
List.of("Ed25519", "EC", "ML-DSA-44", "ML-DSA-65", "ML-DSA-87");
```

Three callers updated to use `SignatureAlgorithms.signatureAlgorithm(key)` instead of `key.getAlgorithm()`:
1. `AgentCryptographicVerifier.verifyCryptographic()` line 59: `Signature.getInstance(SignatureAlgorithms.signatureAlgorithm(pub))`
2. `AgentSignature.signWith()` line 28: `Signature.getInstance(SignatureAlgorithms.signatureAlgorithm(keyPair.getPrivate()))`
3. `LedgerMerklePublisher.signCheckpoint()` line 59: `Signature.getInstance(SignatureAlgorithms.signatureAlgorithm(privateKey))`

**Test scenarios:**
- `SignatureAlgorithmsTest`: P-256 → `SHA256withECDSA`, P-384 → `SHA384withECDSA`, P-521 → `SHA512withECDSA`, Ed25519 → `Ed25519` (passthrough), unknown curve → `IllegalArgumentException`
- `AgentCryptographicVerifierTest`: generate EC P-256 keypair, sign entry with ECDSA, verify → VALID; tamper → INVALID
- `LedgerPemUtilTest`: load EC P-256 public/private PEM files via trial-load, verify parsed key type
- `AgentSignature.signWith()`: generate EC P-256 keypair, call signWith(), verify returned signature can be verified with the public key

- [ ] Write failing tests for `SignatureAlgorithms` (P-256, P-384, P-521, Ed25519, unknown curve)
- [ ] Run tests — verify they fail (class doesn't exist)
- [ ] Implement `SignatureAlgorithms.java`
- [ ] Run tests — verify they pass
- [ ] Write failing test for EC signature verification in `AgentCryptographicVerifierTest`
- [ ] Add `"EC"` to `AgentCryptographicVerifier.SUPPORTED_ALGORITHMS`, update `verifyCryptographic()` to use `SignatureAlgorithms.signatureAlgorithm(pub)`
- [ ] Run test — verify it passes
- [ ] Write failing test for EC PEM loading in `LedgerPemUtilTest`
- [ ] Add `"EC"` to `LedgerPemUtil.SUPPORTED_ALGORITHMS`
- [ ] Run test — verify it passes
- [ ] Write failing test for EC `AgentSignature.signWith()` — generate EC P-256 keypair, sign, verify
- [ ] Update `AgentSignature.signWith()` to use `SignatureAlgorithms.signatureAlgorithm(keyPair.getPrivate())`
- [ ] Run test — verify it passes
- [ ] Update `LedgerMerklePublisher.signCheckpoint()` to use `SignatureAlgorithms.signatureAlgorithm(privateKey)` — add test for EC checkpoint signing in `LedgerMerklePublisherTest`
- [ ] Run full `mvn test -pl runtime` — verify all existing tests still pass
- [ ] Commit: `feat(service): add EC algorithm support to verification and signing infrastructure — Refs #102`

---

### Task 2: SPI Evolution — keyMaterial(), computeKeyRef(), resolveContext(), contextPublicKey()

Extract `computeKeyRef()` to `AgentSignature`, add `AgentKeyMaterial` record, add `keyMaterial()` default method to `AgentSigner` SPI, refactor `AbstractCachingAgentSigner` with `resolveContext()` + `contextPublicKey()`, update `AgentEntrySigner.prepareKey()`.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignature.java` — add `computeKeyRef(byte[])`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyMaterial.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSigner.java` — add `default keyMaterial()`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AbstractCachingAgentSigner.java` — add `resolveContext()`, `contextPublicKey()`, override `keyMaterial()`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentEntrySigner.java` — `prepareKey()` calls `keyMaterial()` instead of `sign()`
- Create: `runtime/src/test/java/io/casehub/ledger/service/AgentSignerContractTest.java` — SPI contract test for `keyMaterial()` default
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AbstractCachingAgentSignerTest.java` — test `resolveContext()`, `contextPublicKey()`, `keyMaterial()` override
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AgentEntrySignerTest.java` — verify `prepareKey()` calls `keyMaterial()` not `sign()`

**Interfaces:**
- Consumes: `AgentSignature` (from Task 1, with `signWith()` EC support)
- Produces: `AgentSignature.computeKeyRef(byte[] publicKeyEncoded) → String` — used by all four Quarkus adapters
- Produces: `AgentKeyMaterial(byte[] publicKey, String keyRef)` — returned by `keyMaterial()`
- Produces: `AgentSigner.keyMaterial(String actorId) → Optional<AgentKeyMaterial>` — default method
- Produces: `AbstractCachingAgentSigner.resolveContext(String actorId) → Optional<C>` — protected, used by `sign()` and `keyMaterial()`
- Produces: `AbstractCachingAgentSigner.contextPublicKey(C context) → PublicKey` — abstract, each adapter returns its cached public key

**Implementation details:**

`AgentSignature.computeKeyRef(byte[])`:
```java
public static String computeKeyRef(final byte[] publicKeyEncoded) {
    try {
        final byte[] hash = MessageDigest.getInstance("SHA-256").digest(publicKeyEncoded);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(hash);
    } catch (final java.security.NoSuchAlgorithmException e) {
        throw new IllegalStateException("SHA-256 not available", e);
    }
}
```
Update `signWith()` to call `computeKeyRef(pubEncoded)` instead of inline code.

`AgentKeyMaterial`:
```java
package io.casehub.ledger.runtime.service;

import java.util.Objects;

public record AgentKeyMaterial(byte[] publicKey, String keyRef) {
    public AgentKeyMaterial {
        Objects.requireNonNull(keyRef, "keyRef must not be null");
        publicKey = publicKey.clone();
    }
}
```

`AgentSigner.keyMaterial()` default:
```java
default Optional<AgentKeyMaterial> keyMaterial(String actorId) {
    return sign(actorId, new byte[0])
            .map(sig -> new AgentKeyMaterial(sig.publicKey(), sig.keyRef()));
}
```

`AbstractCachingAgentSigner` changes:
- Factor cache-lookup logic from `sign()` into `protected Optional<C> resolveContext(String actorId)`
- `sign()` calls `resolveContext()` then `performSign()`
- Add `protected abstract PublicKey contextPublicKey(C context)`
- Override `keyMaterial()` to call `resolveContext()` + `contextPublicKey()` without `performSign()`

`AgentEntrySigner.prepareKey()`:
```java
public void prepareKey(final LedgerEntry entry) {
    if (entry.actorId == null || entry.agentPublicKey != null) return;
    try {
        signer.keyMaterial(entry.actorId)
                .ifPresent(km -> {
                    entry.agentPublicKey = km.publicKey();
                    entry.agentKeyRef = km.keyRef();
                });
    } catch (final Exception e) {
        LOG.warnf("AgentEntrySigner.prepareKey failed for actor %s: %s",
                entry.actorId, e.getMessage());
    }
}
```

**Test scenarios:**
- `AgentSignerContractTest`: lambda `AgentSigner` — `keyMaterial()` returns present when `sign()` returns present; returns empty when `sign()` returns empty; `keyRef` matches
- `AbstractCachingAgentSignerTest`: concrete test subclass with `contextPublicKey()` — `keyMaterial()` returns key without calling `performSign()`; `resolveContext()` caches; `sign()` still works
- `AgentEntrySignerTest`: mock `AgentSigner`, verify `prepareKey()` calls `keyMaterial()` (not `sign()`)

- [ ] Write `AgentSignerContractTest` — failing (method doesn't exist)
- [ ] Extract `AgentSignature.computeKeyRef()`, add `AgentKeyMaterial` record, add `AgentSigner.keyMaterial()` default
- [ ] Run contract test — verify it passes
- [ ] Write failing test in `AbstractCachingAgentSignerTest` — `keyMaterial()` returns key without calling `performSign()`
- [ ] Refactor `AbstractCachingAgentSigner`: extract `resolveContext()`, add `contextPublicKey()` abstract, override `keyMaterial()`
- [ ] Run test — verify it passes
- [ ] Update `AgentEntrySignerTest` — verify `prepareKey()` calls `keyMaterial()` not `sign()`
- [ ] Update `AgentEntrySigner.prepareKey()` to call `keyMaterial()`
- [ ] Run test — verify it passes
- [ ] Update `AgentSignature.signWith()` to use `computeKeyRef()` instead of inline code
- [ ] Run full `mvn test -pl runtime` — verify all existing tests pass
- [ ] Commit: `feat(spi): add keyMaterial() to AgentSigner, extract computeKeyRef(), refactor AbstractCachingAgentSigner — Refs #102`

---

### Task 3: Vault Transit Promotion + Maven Structure

Create the `signing/` directory structure with aggregator POM and `with-signing` parent profile. Split `examples/vault-transit-signing/` into `signing/vault-transit/` (pure Java) + `signing/vault-transit-quarkus/` (Quarkus CDI adapter). Fix two bugs: version selection (`fetchPublicKey` returns version 1 instead of latest) and missing `AgentKeyRotatedEvent` observer.

**Files:**
- Create: `signing/pom.xml` — aggregator POM
- Create: `signing/vault-transit/pom.xml`
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/VaultTransitSigningConfig.java`
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/VaultTransitSigningClient.java`
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/VaultTransitContext.java`
- Create: `signing/vault-transit/src/test/java/io/casehub/ledger/signing/vault/VaultTransitSigningClientTest.java`
- Create: `signing/vault-transit-quarkus/pom.xml`
- Create: `signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitConfig.java`
- Create: `signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitAgentSigner.java`
- Create: `signing/vault-transit-quarkus/src/test/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitAgentSignerIT.java`
- Modify: `pom.xml` — add `with-signing` profile
- Modify: `examples/vault-transit-signing/pom.xml` — change to depend on `casehub-ledger-vault-transit-quarkus`

**Interfaces:**
- Consumes: `AbstractCachingAgentSigner.resolveContext()`, `.contextPublicKey()` (from Task 2)
- Consumes: `AgentSignature.computeKeyRef()` (from Task 2)
- Produces: `VaultTransitSigningClient.fetchPublicKey(String keyName) → PublicKey`
- Produces: `VaultTransitSigningClient.sign(String keyName, byte[] data) → byte[]`
- Produces: `VaultTransitSigningConfig(String address, String token, Map<String,String> keyMapping)`
- Produces: `VaultTransitAgentSigner extends AbstractCachingAgentSigner<VaultTransitContext>` — reference Quarkus adapter pattern for Tasks 4-6

**Implementation details:**

`signing/pom.xml` — aggregator listing `vault-transit`, `vault-transit-quarkus`, and placeholders for Tasks 4-6:
```xml
<groupId>io.casehub</groupId>
<artifactId>casehub-ledger-signing</artifactId>
<packaging>pom</packaging>
<modules>
    <module>vault-transit</module>
    <module>vault-transit-quarkus</module>
</modules>
```

Parent POM gains:
```xml
<profile>
    <id>with-signing</id>
    <modules>
        <module>signing</module>
    </modules>
</profile>
```

**Pure Java module** (`signing/vault-transit/`):
- Standalone POM, `groupId=io.casehub`, `artifactId=casehub-ledger-vault-transit`
- Dependencies: `jackson-databind` (for JSON parsing). No casehub-ledger deps
- `VaultTransitSigningClient` — extracts HTTP logic from existing `VaultTransitAgentSigner`:
  - `fetchPublicKey(String keyName)`: GET `/v1/transit/keys/<keyName>`, parse `data.keys`, select **highest-numbered version** (bug fix: existing code uses `keys.fields().next()` which returns version 1)
  - `sign(String keyName, byte[] data)`: POST `/v1/transit/sign/<keyName>`, strip `vault:v1:` prefix, base64-decode
  - Key type validation: read `data.type`, reject non-`ed25519`
  - PEM parsing: inline strip-headers + base64-decode + `KeyFactory.getInstance("Ed25519")` (no `LedgerPemUtil` dep)
- `VaultTransitSigningConfig` — plain Java record: `address`, `token`, `keyMapping`
- `VaultTransitContext` — record: `keyName`, `publicKey` (moved from examples/)

**Quarkus module** (`signing/vault-transit-quarkus/`):
- POM depends on `casehub-ledger-vault-transit` + `casehub-ledger` runtime + `quarkus-arc` + `quarkus-scheduler`
- `VaultTransitConfig` — `@ConfigMapping(prefix = "casehub.ledger.vault-transit")` (moved from examples/)
- `VaultTransitAgentSigner extends AbstractCachingAgentSigner<VaultTransitContext>`:
  - `loadContext()` → delegates to `VaultTransitSigningClient.fetchPublicKey()`
  - `performSign()` → delegates to `VaultTransitSigningClient.sign()`
  - `contextPublicKey(VaultTransitContext ctx)` → `return ctx.publicKey()`
  - `@Scheduled` refresh
  - `@Observes onKeyRotated()` (bug fix: missing in existing code)

**Test scenarios:**
- `VaultTransitSigningClientTest` (WireMock, no CDI):
  - Sign: WireMock returns `vault:v1:` prefixed base64 → client returns decoded bytes
  - Fetch public key: WireMock returns multi-version `keys` map → client selects highest version (regression test for bug fix)
  - Key not found: WireMock 404 → client throws
  - Wrong key type: `data.type = "rsa-2048"` → client rejects
- `VaultTransitAgentSignerIT` (@QuarkusTest, WireMock):
  - Full save pipeline: entry persisted with valid Ed25519 signature
  - Round-trip: sign then verify with `AgentCryptographicVerifier`
  - `keyMaterial()` returns key without calling sign API
  - Key rotation event → cache invalidated

- [ ] Create `signing/` directory, `signing/pom.xml` aggregator
- [ ] Create `signing/vault-transit/pom.xml`
- [ ] Write failing `VaultTransitSigningClientTest` — sign and fetchPublicKey with WireMock
- [ ] Implement `VaultTransitSigningConfig`, `VaultTransitContext`, `VaultTransitSigningClient`
- [ ] Run client tests — verify they pass
- [ ] Write test for highest-version selection (bug fix regression test)
- [ ] Verify it passes (implemented above)
- [ ] Write test for wrong key type rejection
- [ ] Verify it passes
- [ ] Create `signing/vault-transit-quarkus/pom.xml`
- [ ] Write failing `VaultTransitAgentSignerIT` — full save pipeline with WireMock
- [ ] Implement `VaultTransitConfig`, `VaultTransitAgentSigner`
- [ ] Run integration tests — verify they pass
- [ ] Write test for `keyMaterial()` — verify no sign API call
- [ ] Verify it passes
- [ ] Update parent `pom.xml` — add `with-signing` profile
- [ ] Update `examples/vault-transit-signing/pom.xml` — depend on `casehub-ledger-vault-transit-quarkus`, remove signing logic
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-signing` — verify signing modules build
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime` — verify core still passes
- [ ] Commit: `feat(signing): promote Vault Transit to first-class module with bug fixes — Refs #102`

---

### Task 4: AWS KMS Adapter

Create `signing/aws-kms/` (pure Java) and `signing/aws-kms-quarkus/` (Quarkus CDI adapter).

**Files:**
- Create: `signing/aws-kms/pom.xml`
- Create: `signing/aws-kms/src/main/java/io/casehub/ledger/signing/aws/AwsKmsSigningConfig.java`
- Create: `signing/aws-kms/src/main/java/io/casehub/ledger/signing/aws/AwsKmsSigningClient.java`
- Create: `signing/aws-kms/src/main/java/io/casehub/ledger/signing/aws/AwsKmsContext.java`
- Create: `signing/aws-kms/src/test/java/io/casehub/ledger/signing/aws/AwsKmsSigningClientTest.java`
- Create: `signing/aws-kms-quarkus/pom.xml`
- Create: `signing/aws-kms-quarkus/src/main/java/io/casehub/ledger/signing/aws/quarkus/AwsKmsConfig.java`
- Create: `signing/aws-kms-quarkus/src/main/java/io/casehub/ledger/signing/aws/quarkus/AwsKmsAgentSigner.java`
- Create: `signing/aws-kms-quarkus/src/test/java/io/casehub/ledger/signing/aws/quarkus/AwsKmsAgentSignerIT.java`
- Modify: `signing/pom.xml` — add `aws-kms`, `aws-kms-quarkus` modules

**Interfaces:**
- Consumes: `AbstractCachingAgentSigner<C>`, `AgentSignature.computeKeyRef()`, `AgentKeyMaterial` (from Tasks 1-2)
- Consumes: Vault Transit module as pattern reference (from Task 3)
- Produces: `AwsKmsSigningClient.fetchPublicKey(String keyArn) → PublicKey`
- Produces: `AwsKmsSigningClient.sign(String keyArn, byte[] data) → byte[]`
- Produces: `AwsKmsAgentSigner extends AbstractCachingAgentSigner<AwsKmsContext>`

**Implementation details:**

**Pure Java module** (`signing/aws-kms/`):
- Dependencies: `software.amazon.awssdk:kms`
- `AwsKmsSigningConfig` — record: `region`, `keyMapping` (actorId → key ARN)
- `AwsKmsSigningClient`:
  - Constructor: creates `KmsClient` with configured region + default credential provider
  - `fetchPublicKey(String keyArn)`: calls `kmsClient.getPublicKey()`, checks `keySpec()` is EC (reject RSA with error log + return null), parses DER bytes with `KeyFactory.getInstance("EC")`
  - `sign(String keyArn, byte[] data)`: maps key spec to signing algorithm (`ECC_NIST_P256 → ECDSA_SHA_256`, `ECC_NIST_P384 → ECDSA_SHA_384`, `ECC_NIST_P521 → ECDSA_SHA_512`), calls `kmsClient.sign()`, returns signature bytes
  - Error handling: `NotFoundException` → null; `KmsException` 401/403/429/5xx → throw RuntimeException
- `AwsKmsContext` — record: `keyArn`, `publicKey`

**Quarkus module** (`signing/aws-kms-quarkus/`):
- Dependencies: `casehub-ledger-aws-kms` + `casehub-ledger` runtime + `quarkus-arc` + `quarkus-scheduler`
- `AwsKmsConfig` — `@ConfigMapping(prefix = "casehub.ledger.aws-kms")`: `region()`, `keyMapping()`, `refreshInterval()`
- `AwsKmsAgentSigner extends AbstractCachingAgentSigner<AwsKmsContext>`:
  - `loadContext()`: get ARN from config, call `client.fetchPublicKey()`, wrap in `AwsKmsContext`; null return → `Optional.empty()` (permanent error — RSA key or not found)
  - `performSign()`: call `client.sign()`, construct `AgentSignature` with `computeKeyRef()`
  - `contextPublicKey(AwsKmsContext ctx)` → `return ctx.publicKey()`
  - `@Scheduled` refresh, `@Observes onKeyRotated()`
- `@ApplicationScoped @Alternative @Priority(1)`

**Test scenarios:**
- `AwsKmsSigningClientTest` (mock `KmsClient` — it's a Java interface):
  - Sign P-256 key: mock returns signature bytes → client returns them
  - Fetch public key P-256: mock returns DER bytes + `ECC_NIST_P256` spec → client returns EC PublicKey
  - RSA key rejection: mock returns `RSA_2048` spec → client returns null + logs error
  - Not found: mock throws `NotFoundException` → client returns null
  - Transient error: mock throws `KmsException` 429 → client throws RuntimeException
- `AwsKmsAgentSignerIT` (@QuarkusTest, mocked `KmsClient`):
  - Full save pipeline with EC P-256 key
  - Round-trip: sign then verify with `AgentCryptographicVerifier`
  - `keyMaterial()` returns key without calling KMS sign
  - Unconfigured actor → unsigned entry

- [ ] Create `signing/aws-kms/pom.xml` with AWS SDK dependency
- [ ] Write failing `AwsKmsSigningClientTest` — sign and fetchPublicKey with mocked KmsClient
- [ ] Implement `AwsKmsSigningConfig`, `AwsKmsContext`, `AwsKmsSigningClient`
- [ ] Run client tests — verify they pass
- [ ] Write RSA rejection test, not-found test, transient error test
- [ ] Verify they pass
- [ ] Create `signing/aws-kms-quarkus/pom.xml`
- [ ] Write failing `AwsKmsAgentSignerIT` — full save pipeline
- [ ] Implement `AwsKmsConfig`, `AwsKmsAgentSigner`
- [ ] Run integration tests — verify they pass
- [ ] Add modules to `signing/pom.xml`
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-signing` — full signing build
- [ ] Commit: `feat(signing): add AWS KMS AgentSigner adapter — Refs #102`

---

### Task 5: GCP Cloud KMS Adapter

Create `signing/gcp-kms/` (pure Java) and `signing/gcp-kms-quarkus/` (Quarkus CDI adapter).

**Files:**
- Create: `signing/gcp-kms/pom.xml`
- Create: `signing/gcp-kms/src/main/java/io/casehub/ledger/signing/gcp/GcpKmsSigningConfig.java`
- Create: `signing/gcp-kms/src/main/java/io/casehub/ledger/signing/gcp/GcpKmsSigningClient.java`
- Create: `signing/gcp-kms/src/main/java/io/casehub/ledger/signing/gcp/GcpKmsContext.java`
- Create: `signing/gcp-kms/src/test/java/io/casehub/ledger/signing/gcp/GcpKmsSigningClientTest.java`
- Create: `signing/gcp-kms-quarkus/pom.xml`
- Create: `signing/gcp-kms-quarkus/src/main/java/io/casehub/ledger/signing/gcp/quarkus/GcpKmsConfig.java`
- Create: `signing/gcp-kms-quarkus/src/main/java/io/casehub/ledger/signing/gcp/quarkus/GcpKmsAgentSigner.java`
- Create: `signing/gcp-kms-quarkus/src/test/java/io/casehub/ledger/signing/gcp/quarkus/GcpKmsAgentSignerIT.java`
- Modify: `signing/pom.xml` — add `gcp-kms`, `gcp-kms-quarkus` modules

**Interfaces:**
- Consumes: Same as Task 4
- Produces: `GcpKmsSigningClient.fetchPublicKey(String versionName) → PublicKey`
- Produces: `GcpKmsSigningClient.sign(String versionName, byte[] data) → byte[]`
- Produces: `GcpKmsAgentSigner extends AbstractCachingAgentSigner<GcpKmsContext>`

**Implementation details:**

**Pure Java module** (`signing/gcp-kms/`):
- Dependencies: `com.google.cloud:google-cloud-kms`
- `GcpKmsSigningConfig` — record: `keyMapping` (actorId → CryptoKeyVersion resource name)
- `GcpKmsSigningClient`:
  - Constructor: creates `KeyManagementServiceClient` with default credentials
  - `fetchPublicKey(String versionName)`: calls `getPublicKey()`, checks algorithm is EC (reject RSA/secp256k1), parses PEM → inline strip-headers + base64-decode + `KeyFactory.getInstance("EC")`
  - `sign(String versionName, byte[] data)`: reads `CryptoKeyVersion.algorithm`, selects digest algorithm (`EC_SIGN_P256_SHA256 → SHA-256`, `EC_SIGN_P384_SHA384 → SHA-384`), computes digest, calls `asymmetricSign()`
  - Error handling: `NOT_FOUND` → null; `PERMISSION_DENIED`/`UNAUTHENTICATED` → throw; `UNAVAILABLE`/`DEADLINE_EXCEEDED`/`RESOURCE_EXHAUSTED` → throw
- `GcpKmsContext` — record: `versionName`, `publicKey`, `algorithm` (cached for sign-time digest selection)

**Quarkus module** (`signing/gcp-kms-quarkus/`):
- Same pattern as Task 4's AWS adapter
- `@ConfigMapping(prefix = "casehub.ledger.gcp-kms")`: `keyMapping()`, `refreshInterval()`

**Test scenarios:**
- `GcpKmsSigningClientTest` (WireMock at gRPC/HTTP level — `KeyManagementServiceClient` is concrete):
  - Sign P-256: correct digest used (SHA-256), returns signature bytes
  - Fetch public key P-256: PEM parsed correctly
  - RSA rejection: non-EC algorithm → returns null
  - Digest selection: P-384 key → SHA-384 digest (not SHA-256)
- `GcpKmsAgentSignerIT`: same pattern as Task 4

- [ ] Create `signing/gcp-kms/pom.xml` with GCP KMS SDK dependency
- [ ] Write failing `GcpKmsSigningClientTest` — sign and fetchPublicKey
- [ ] Implement `GcpKmsSigningConfig`, `GcpKmsContext`, `GcpKmsSigningClient`
- [ ] Run client tests — verify they pass
- [ ] Write RSA rejection test, digest selection test for P-384
- [ ] Verify they pass
- [ ] Create `signing/gcp-kms-quarkus/pom.xml`
- [ ] Write failing `GcpKmsAgentSignerIT` — full save pipeline
- [ ] Implement `GcpKmsConfig`, `GcpKmsAgentSigner`
- [ ] Run integration tests — verify they pass
- [ ] Add modules to `signing/pom.xml`
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-signing`
- [ ] Commit: `feat(signing): add GCP Cloud KMS AgentSigner adapter — Refs #102`

---

### Task 6: Azure Key Vault Adapter

Create `signing/azure-keyvault/` (pure Java) and `signing/azure-keyvault-quarkus/` (Quarkus CDI adapter). Includes the R||S → DER conversion for EC signatures.

**Files:**
- Create: `signing/azure-keyvault/pom.xml`
- Create: `signing/azure-keyvault/src/main/java/io/casehub/ledger/signing/azure/AzureKeyVaultSigningConfig.java`
- Create: `signing/azure-keyvault/src/main/java/io/casehub/ledger/signing/azure/AzureKeyVaultSigningClient.java`
- Create: `signing/azure-keyvault/src/main/java/io/casehub/ledger/signing/azure/AzureKeyVaultContext.java`
- Create: `signing/azure-keyvault/src/main/java/io/casehub/ledger/signing/azure/EcSignatureConverter.java`
- Create: `signing/azure-keyvault/src/test/java/io/casehub/ledger/signing/azure/AzureKeyVaultSigningClientTest.java`
- Create: `signing/azure-keyvault/src/test/java/io/casehub/ledger/signing/azure/EcSignatureConverterTest.java`
- Create: `signing/azure-keyvault-quarkus/pom.xml`
- Create: `signing/azure-keyvault-quarkus/src/main/java/io/casehub/ledger/signing/azure/quarkus/AzureKeyVaultConfig.java`
- Create: `signing/azure-keyvault-quarkus/src/main/java/io/casehub/ledger/signing/azure/quarkus/AzureKeyVaultAgentSigner.java`
- Create: `signing/azure-keyvault-quarkus/src/test/java/io/casehub/ledger/signing/azure/quarkus/AzureKeyVaultAgentSignerIT.java`
- Modify: `signing/pom.xml` — add `azure-keyvault`, `azure-keyvault-quarkus` modules

**Interfaces:**
- Consumes: Same as Task 4
- Produces: `AzureKeyVaultSigningClient.fetchPublicKey(String keyName, String vaultUrl) → PublicKey`
- Produces: `AzureKeyVaultSigningClient.sign(String keyName, String vaultUrl, byte[] data) → byte[]`
- Produces: `EcSignatureConverter.rawToDer(byte[] rawSig, int componentSize) → byte[]` — converts R||S to DER
- Produces: `AzureKeyVaultAgentSigner extends AbstractCachingAgentSigner<AzureKeyVaultContext>`

**Implementation details:**

**Pure Java module** (`signing/azure-keyvault/`):
- Dependencies: `com.azure:azure-security-keyvault-keys`, `com.azure:azure-identity`
- `AzureKeyVaultSigningConfig` — record: `keyMapping` (actorId → `vaultUrl#keyName`)
- `EcSignatureConverter` — standalone utility:
  - `rawToDer(byte[] rawSig, int componentSize)`: splits R||S at midpoint, constructs DER `SEQUENCE { INTEGER r, INTEGER s }` using `BigInteger` with leading-zero handling. No BouncyCastle
  - Component sizes: P-256 → 32, P-384 → 48, P-521 → 66
  - `componentSizeForCurve(ECParameterSpec params)`: derives from `params.getOrder().bitLength()`
- `AzureKeyVaultSigningClient`:
  - `fetchPublicKey()`: creates `KeyClient`, calls `getKey()`, checks `getKeyType()` is EC (reject RSA), calls `toEc()` for JCA `ECPublicKey`
  - `sign()`: maps curve to `SignatureAlgorithm` (P-256 → ES256, P-384 → ES384, P-521 → ES512), computes digest (SHA-256/384/512), calls `CryptographyClient.sign()`, converts R||S → DER via `EcSignatureConverter`
  - Error handling: `ResourceNotFoundException` → null; 401/403 → throw; timeout/429/5xx → throw
- `AzureKeyVaultContext` — record: `keyName`, `vaultUrl`, `publicKey`, `componentSize` (cached for sign-time DER conversion)

**Test scenarios:**
- `EcSignatureConverterTest`:
  - P-256 (32-byte components): known R||S → expected DER
  - P-384 (48-byte components): correct DER output
  - Leading-zero handling: R with high bit set → DER pads with 0x00
  - Round-trip: generate EC signature with JCA, convert DER→raw→DER, verify equal
- `AzureKeyVaultSigningClientTest` (WireMock — SDK clients are concrete):
  - Sign P-256: correct algorithm used (ES256), R||S converted to DER
  - Fetch public key: JWK parsed to ECPublicKey
  - RSA rejection
  - DER output verifiable with JCA `Signature.verify()`
- `AzureKeyVaultAgentSignerIT`: same pattern as Task 4

- [ ] Write `EcSignatureConverterTest` — P-256, P-384, leading-zero, round-trip
- [ ] Implement `EcSignatureConverter`
- [ ] Run converter tests — verify they pass
- [ ] Create `signing/azure-keyvault/pom.xml` with Azure SDK dependencies
- [ ] Write failing `AzureKeyVaultSigningClientTest`
- [ ] Implement `AzureKeyVaultSigningConfig`, `AzureKeyVaultContext`, `AzureKeyVaultSigningClient`
- [ ] Run client tests — verify they pass
- [ ] Write RSA rejection test
- [ ] Verify it passes
- [ ] Create `signing/azure-keyvault-quarkus/pom.xml`
- [ ] Write failing `AzureKeyVaultAgentSignerIT` — full save pipeline
- [ ] Implement `AzureKeyVaultConfig`, `AzureKeyVaultAgentSigner`
- [ ] Run integration tests — verify they pass
- [ ] Add modules to `signing/pom.xml`
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-signing`
- [ ] Commit: `feat(signing): add Azure Key Vault AgentSigner adapter — Refs #102`

---

### Task 7: Examples Update

Slim down `examples/vault-transit-signing/` to a thin consumer example. Create three new examples for AWS KMS, GCP KMS, and Azure Key Vault.

**Files:**
- Modify: `examples/vault-transit-signing/pom.xml` — depend on `casehub-ledger-vault-transit-quarkus`
- Modify: `examples/vault-transit-signing/src/` — remove all signing logic, keep only config + test
- Create: `examples/aws-kms-signing/pom.xml`
- Create: `examples/aws-kms-signing/src/main/resources/application.properties`
- Create: `examples/aws-kms-signing/src/test/java/io/casehub/ledger/examples/aws/AwsKmsExampleIT.java`
- Create: `examples/aws-kms-signing/README.md`
- Create: `examples/gcp-kms-signing/pom.xml`
- Create: `examples/gcp-kms-signing/src/main/resources/application.properties`
- Create: `examples/gcp-kms-signing/src/test/java/io/casehub/ledger/examples/gcp/GcpKmsExampleIT.java`
- Create: `examples/gcp-kms-signing/README.md`
- Create: `examples/azure-keyvault-signing/pom.xml`
- Create: `examples/azure-keyvault-signing/src/main/resources/application.properties`
- Create: `examples/azure-keyvault-signing/src/test/java/io/casehub/ledger/examples/azure/AzureKeyVaultExampleIT.java`
- Create: `examples/azure-keyvault-signing/README.md`

**Interfaces:**
- Consumes: All four `*-quarkus` signing modules (from Tasks 3-6)

**Implementation details:**

Each example is a standalone POM (same pattern as existing `examples/vault-transit-signing/`):
- Depends on the `-quarkus` module + `casehub-ledger-memory` (test scope)
- `application.properties`: config for WireMock-friendly defaults
- One `@QuarkusTest` demonstrating a signed entry save + verify
- `README.md`: IAM/RBAC permissions for production use (AWS IAM policy, GCP IAM roles, Azure RBAC roles)

The slimmed-down `examples/vault-transit-signing/`:
- Delete `VaultTransitAgentSigner.java`, `VaultTransitContext.java`, `VaultTransitConfig.java`
- Keep the test, updated to use the `-quarkus` module's classes
- `pom.xml` depends on `casehub-ledger-vault-transit-quarkus` instead of building signing logic

Update parent POM: add new examples to `with-examples` profile (alongside `order-processing` and `trust-score-routing`).

- [ ] Slim `examples/vault-transit-signing/` — remove signing classes, depend on `-quarkus` module
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/vault-transit-signing` — verify example still works
- [ ] Create `examples/aws-kms-signing/` — POM, properties, README, test
- [ ] Create `examples/gcp-kms-signing/` — POM, properties, README, test
- [ ] Create `examples/azure-keyvault-signing/` — POM, properties, README, test
- [ ] Add all four to `with-examples` profile in parent POM
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-examples,with-signing` — all examples pass
- [ ] Commit: `feat(examples): add Cloud KMS signing examples, slim Vault Transit example — Refs #102`

---

### Task 8: Follow-Up Issues, Issue #102 Update, Documentation

Create follow-up GitHub issues, update issue #102 body to reflect scope deviation, update CLAUDE.md and ARC42STORIES.MD.

**Files:**
- Modify: `CLAUDE.md` — update Project Structure section to include `signing/` modules
- Modify: `ARC42STORIES.MD` — update relevant architecture sections

**Actions:**
- [ ] Update issue #102 body to note the `examples/` → `signing/` scope deviation (already agreed in spec)
- [ ] Create GitHub issue: "Promote example capabilities to first-class modules" — covers otel-trace-wiring, prov-dm-export, eigentrust-mesh, trust-score-routing
- [ ] Update `CLAUDE.md` Project Structure section — add `signing/` module hierarchy, Maven coordinates, package roots
- [ ] Update `ARC42STORIES.MD` — update §5 Building Block View and §9 to reflect signing module architecture, EC support, keyMaterial() SPI
- [ ] Run final coherence check: re-read `docs/PLATFORM.md` and all protocols, verify implementation is consistent
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test` — full default build passes
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-signing` — signing modules pass
- [ ] Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-examples,with-signing` — all examples pass
- [ ] Commit: `docs: update CLAUDE.md and ARC42STORIES.MD for signing modules — Refs #102`
