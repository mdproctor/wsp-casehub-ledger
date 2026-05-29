# AgentSigner External Key Distribution — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `AgentKeyProvider` SPI with `AgentSigner` SPI so signing can happen locally (JCA) or remotely (Vault Transit, HSM), and deliver a Vault Transit example demonstrating remote signing.

**Architecture:** `AgentSigner.sign(actorId, data) → Optional<AgentSignature>` places signing responsibility in the SPI. `ConfiguredAgentSigner` replaces `ConfiguredAgentKeyProvider` with eager PEM loading. `AbstractCachingAgentSigner<C>` provides a template-method base for external providers. A new `examples/vault-transit-signing/` Maven module implements Vault Transit remote signing with WireMock tests.

**Tech Stack:** Java 21, Quarkus 3.32.2, JCA (java.security), java.net.http.HttpClient, WireMock 3.x, JUnit 5, AssertJ, Mockito

---

## File Map

**Create (runtime/src/main/java/io/casehub/ledger/runtime/service/):**
- `AgentSignature.java` — record: `(signature, publicKey, keyRef)` + compact constructor (defensive copies) + `signWith(KeyPair, byte[])` factory
- `AgentSigner.java` — interface: `sign(String actorId, byte[] data) → Optional<AgentSignature>`
- `AbstractCachingAgentSigner.java` — abstract generic base: per-actorId context cache, `loadContext()` / `performSign()` template methods, `invalidateAll()` / `invalidate(actorId)` hooks
- `ConfiguredAgentSigner.java` — `@DefaultBean @ApplicationScoped`, constructor-injected `LedgerConfig`, eager `@PostConstruct` PEM loading, `failedActors` sentinel

**Modify (runtime/src/main/java/io/casehub/ledger/runtime/service/):**
- `LedgerPemUtil.java` — add `public static PublicKey parsePublicKey(String pemContent) throws Exception`
- `AgentSignatureEnricher.java` — inject `AgentSigner` instead of `AgentKeyProvider`; single `signer.sign()` call sets all three entry fields

**Delete:**
- `AgentKeyProvider.java`
- `SigningKey.java`
- `ConfiguredAgentKeyProvider.java`

**Create (runtime/src/test/java/io/casehub/ledger/service/):**
- `AgentSignatureTest.java` — replaces `SigningKeyTest.java`; tests defensive copies, `signWith()` factory, keyRef derivation
- `AbstractCachingAgentSignerTest.java` — cache hit, unconfigured actor cached, error not cached, `invalidateAll()`, `invalidate(actorId)`
- `ConfiguredAgentSignerTest.java` — PEM loading, successful sign, unconfigured actor → empty, failed load → empty with no per-call logging

**Delete:**
- `SigningKeyTest.java` — replaced by `AgentSignatureTest.java`

**Modify (runtime/src/test/java/io/casehub/ledger/service/):**
- `AgentSignatureEnricherTest.java` — update lambda: `(actorId, data) -> Optional.of(AgentSignature.signWith(kp, data))`
- `AgentSigningIT.java` — `@InjectMock AgentSigner`; update mock setup and lambda
- `AgentSignatureVerificationServiceIT.java` — replace all `SigningKey.of(kp)` usage with `AgentSignature.signWith(kp, canonical)` and `agentSig.keyRef()`
- `ReactiveAgentSignatureVerificationServiceIT.java` — `@InjectMock AgentSigner`; replace `AgentKeyProvider` mock + `SigningKey` usage

**Modify (runtime/src/test/java/io/casehub/ledger/runtime/service/):**
- `LedgerPemUtilTest.java` — add `parsePublicKey_roundTrip_fromPemString()` test

**Create (docs/adr/):**
- `0014-agent-signer-spi.md` — extends ADR 0011; records SPI rename rationale, two-tier design

**Create (examples/vault-transit-signing/):**
- `pom.xml` — standalone Quarkus app, WireMock test dep
- `src/main/java/io/casehub/ledger/examples/vault/VaultTransitConfig.java` — `@ConfigMapping`
- `src/main/java/io/casehub/ledger/examples/vault/VaultTransitAgentSigner.java` — extends `AbstractCachingAgentSigner<VaultTransitContext>`
- `src/main/resources/application.properties`
- `src/test/java/io/casehub/ledger/examples/vault/VaultTransitAgentSignerIT.java` — WireMock stubs
- `src/test/resources/application.properties`
- `README.md`

**Modify (root):**
- `pom.xml` — add `examples/vault-transit-signing` to `with-examples` profile

---

## Task 1: `AgentSignature` record

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignature.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureTest.java`
- Delete: `runtime/src/test/java/io/casehub/ledger/service/SigningKeyTest.java`

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureTest.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.MessageDigest;
import java.security.Signature;
import java.util.Base64;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.service.AgentSignature;

class AgentSignatureTest {

    @Test
    void signWith_producesVerifiableSignature() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final byte[] data = "canonical bytes".getBytes();

        final AgentSignature sig = AgentSignature.signWith(kp, data);

        final Signature verifier = Signature.getInstance("Ed25519");
        verifier.initVerify(kp.getPublic());
        verifier.update(data);
        assertThat(verifier.verify(sig.signature())).isTrue();
    }

    @Test
    void signWith_publicKeyIsX509Encoded() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final AgentSignature sig = AgentSignature.signWith(kp, new byte[]{1, 2, 3});
        assertThat(sig.publicKey()).isEqualTo(kp.getPublic().getEncoded());
    }

    @Test
    void signWith_keyRefIsSha256OfPublicKeyBase64Url() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final AgentSignature sig = AgentSignature.signWith(kp, new byte[]{1, 2, 3});

        final byte[] hash = MessageDigest.getInstance("SHA-256").digest(kp.getPublic().getEncoded());
        final String expected = Base64.getUrlEncoder().withoutPadding().encodeToString(hash);
        assertThat(sig.keyRef()).isEqualTo(expected);
    }

    @Test
    void keyRef_isBase64UrlNoPadding_43chars() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final AgentSignature sig = AgentSignature.signWith(kp, new byte[]{1});
        assertThat(sig.keyRef()).matches("[A-Za-z0-9_-]+").hasSize(43);
    }

    @Test
    void compactConstructor_defensiveCopy_signature() {
        final byte[] sig = {1, 2, 3};
        final byte[] pub = {4, 5, 6};
        final AgentSignature as = new AgentSignature(sig, pub, "keyref");
        sig[0] = 99;
        assertThat(as.signature()[0]).isEqualTo((byte) 1);
    }

    @Test
    void compactConstructor_defensiveCopy_publicKey() {
        final byte[] sig = {1, 2, 3};
        final byte[] pub = {4, 5, 6};
        final AgentSignature as = new AgentSignature(sig, pub, "keyref");
        pub[0] = 99;
        assertThat(as.publicKey()[0]).isEqualTo((byte) 4);
    }
}
```

- [ ] **Step 2: Run to verify it fails (compilation error — class not yet created)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentSignatureTest -q 2>&1 | tail -5
```

Expected: compilation error referencing missing `AgentSignature`.

- [ ] **Step 3: Create `AgentSignature.java`**

```java
package io.casehub.ledger.runtime.service;

import java.security.KeyPair;
import java.security.MessageDigest;
import java.security.Signature;
import java.util.Base64;

public record AgentSignature(byte[] signature, byte[] publicKey, String keyRef) {

    public AgentSignature {
        signature = signature.clone();
        publicKey = publicKey.clone();
    }

    /**
     * Algorithm-transparent local signing factory.
     * Derives the algorithm from {@code keyPair.getPrivate().getAlgorithm()} — never hardcodes a string.
     * Computes {@code keyRef = Base64URL(SHA-256(publicKey.getEncoded()))}.
     */
    public static AgentSignature signWith(final KeyPair keyPair, final byte[] data) {
        try {
            final byte[] pubEncoded = keyPair.getPublic().getEncoded();
            final Signature sig = Signature.getInstance(keyPair.getPrivate().getAlgorithm());
            sig.initSign(keyPair.getPrivate());
            sig.update(data);
            final byte[] sigBytes = sig.sign();
            final byte[] hash = MessageDigest.getInstance("SHA-256").digest(pubEncoded);
            final String keyRef = Base64.getUrlEncoder().withoutPadding().encodeToString(hash);
            return new AgentSignature(sigBytes, pubEncoded, keyRef);
        } catch (final Exception e) {
            throw new IllegalStateException("Local signing failed", e);
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentSignatureTest -q
```

Expected: BUILD SUCCESS, 6 tests pass.

- [ ] **Step 5: Delete `SigningKeyTest.java`**

```bash
rm runtime/src/test/java/io/casehub/ledger/service/SigningKeyTest.java
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignature.java runtime/src/test/java/io/casehub/ledger/service/AgentSignatureTest.java runtime/src/test/java/io/casehub/ledger/service/SigningKeyTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#85): add AgentSignature record — defensive copies, signWith() factory, keyRef derivation"
```

---

## Task 2: `AgentSigner` interface + `AbstractCachingAgentSigner<C>`

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSigner.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AbstractCachingAgentSigner.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/AbstractCachingAgentSignerTest.java`

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/ledger/service/AbstractCachingAgentSignerTest.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.security.KeyPairGenerator;
import java.util.Optional;
import java.util.concurrent.atomic.AtomicInteger;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.service.AbstractCachingAgentSigner;
import io.casehub.ledger.runtime.service.AgentSignature;

class AbstractCachingAgentSignerTest {

    // Concrete test subclass — context type is String (a key name or null if unconfigured)
    static class TestSigner extends AbstractCachingAgentSigner<String> {
        final AtomicInteger loadCount = new AtomicInteger();
        volatile String contextToReturn = "context";
        volatile boolean throwOnLoad = false;

        @Override
        protected Optional<String> loadContext(final String actorId) {
            loadCount.incrementAndGet();
            if (throwOnLoad) throw new RuntimeException("simulated failure");
            return Optional.ofNullable(contextToReturn);
        }

        @Override
        protected AgentSignature performSign(final String actorId, final String context, final byte[] data) {
            try {
                return AgentSignature.signWith(
                        KeyPairGenerator.getInstance("Ed25519").generateKeyPair(), data);
            } catch (final Exception e) {
                throw new RuntimeException(e);
            }
        }
    }

    @Test
    void cachesContextAfterFirstLoad() {
        final TestSigner signer = new TestSigner();
        signer.sign("actor1", new byte[]{1});
        signer.sign("actor1", new byte[]{2});
        assertThat(signer.loadCount.get()).isEqualTo(1);
    }

    @Test
    void returnsEmptyForUnconfiguredActor_andCachesAbsence() {
        final TestSigner signer = new TestSigner();
        signer.contextToReturn = null;
        assertThat(signer.sign("unknown", new byte[]{1})).isEmpty();
        assertThat(signer.sign("unknown", new byte[]{1})).isEmpty();
        assertThat(signer.loadCount.get()).isEqualTo(1);
    }

    @Test
    void transientError_notCached_retriesOnNextCall() {
        final TestSigner signer = new TestSigner();
        signer.throwOnLoad = true;
        assertThatThrownBy(() -> signer.sign("actor1", new byte[]{1}))
                .isInstanceOf(RuntimeException.class).hasMessage("simulated failure");

        signer.throwOnLoad = false;
        final Optional<AgentSignature> result = signer.sign("actor1", new byte[]{1});
        assertThat(result).isPresent();
        assertThat(signer.loadCount.get()).isEqualTo(2);
    }

    @Test
    void invalidateAll_forcesReloadOnNextSign() {
        final TestSigner signer = new TestSigner();
        signer.sign("actor1", new byte[]{1});
        signer.invalidateAll();
        signer.sign("actor1", new byte[]{1});
        assertThat(signer.loadCount.get()).isEqualTo(2);
    }

    @Test
    void invalidate_evictsOnlyTargetActor() {
        final TestSigner signer = new TestSigner();
        signer.sign("actor1", new byte[]{1});
        signer.sign("actor2", new byte[]{1});
        signer.invalidate("actor1");
        signer.sign("actor1", new byte[]{1});
        signer.sign("actor2", new byte[]{1});
        // actor1 loaded twice, actor2 once
        assertThat(signer.loadCount.get()).isEqualTo(3);
    }

    @Test
    void returnsPresent_whenContextPresent() {
        final TestSigner signer = new TestSigner();
        assertThat(signer.sign("actor1", new byte[]{1})).isPresent();
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AbstractCachingAgentSignerTest -q 2>&1 | tail -5
```

Expected: compilation error — `AbstractCachingAgentSigner` not found.

- [ ] **Step 3: Create `AgentSigner.java`**

```java
package io.casehub.ledger.runtime.service;

import java.util.Optional;

/**
 * SPI: performs (or delegates) the signing of a {@link io.casehub.ledger.runtime.model.LedgerEntry}
 * on behalf of the given actorId, returning the complete signature result.
 *
 * <p>Return {@link Optional#empty()} for actors that do not participate in bilateral signing —
 * their entries will be persisted unsigned.
 *
 * <p><strong>Error handling:</strong> throw {@link RuntimeException} for transient failures
 * (network, auth). The enricher swallows it and leaves the entry unsigned. Do NOT return
 * empty to signal an error — reserve empty for "actor not configured".
 *
 * <p><strong>Thread safety:</strong> implementations must be safe for concurrent calls.
 *
 * <p><strong>Algorithm transparency:</strong> never hardcode a cryptographic algorithm string.
 * See protocol PP-20260523-e7b577.
 *
 * <p>This interface has one abstract method and supports lambda construction in tests
 * (SAM interface). {@code @FunctionalInterface} is intentionally absent to allow
 * future {@code default} methods.
 */
public interface AgentSigner {

    /**
     * @param actorId the actor identity (e.g. {@code "claude:reviewer@v1"})
     * @param data    canonical bytes to sign ({@link LedgerMerkleTree#canonicalBytes})
     * @return signed result, or empty if this actor does not sign entries
     */
    Optional<AgentSignature> sign(String actorId, byte[] data);
}
```

- [ ] **Step 4: Create `AbstractCachingAgentSigner.java`**

```java
package io.casehub.ledger.runtime.service;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Abstract base for {@link AgentSigner} implementations with per-actorId context caching.
 *
 * <p>Designed for external providers (Vault Transit, Cloud KMS, HSM via non-JCA API) where
 * {@link #loadContext} involves network or hardware I/O. The cache avoids redundant calls
 * on every {@code @PrePersist}.
 *
 * <p><strong>Cache semantics:</strong>
 * <ul>
 *   <li>{@link #loadContext} returns {@code Optional.empty()} → cached as absent; no further calls for this actor</li>
 *   <li>{@link #loadContext} throws → NOT cached; next {@link #sign} call retries</li>
 *   <li>Two threads hitting the same unconfigured actor simultaneously both call {@link #loadContext}
 *       (putIfAbsent, not computeIfAbsent). This is a deliberate trade-off: computeIfAbsent blocks
 *       the ConcurrentHashMap bucket for the duration of a network call and has reentrancy constraints;
 *       a duplicate load on cold start is cheaper than blocking unrelated actors.</li>
 * </ul>
 *
 * @param <C> per-actorId context type (e.g. {@code KeyPair} for extractable-key providers,
 *            {@code VaultTransitContext} for remote-signing providers)
 */
public abstract class AbstractCachingAgentSigner<C> implements AgentSigner {

    private final ConcurrentHashMap<String, Optional<C>> contextCache = new ConcurrentHashMap<>();

    @Override
    public final Optional<AgentSignature> sign(final String actorId, final byte[] data) {
        Optional<C> cached = contextCache.get(actorId);
        if (cached == null) {
            // loadContext throws on transient failure → not cached, caller retries next time
            final Optional<C> loaded = loadContext(actorId);
            final Optional<C> racing = contextCache.putIfAbsent(actorId, loaded);
            cached = racing != null ? racing : loaded;
        }
        return cached.map(ctx -> performSign(actorId, ctx, data));
    }

    /**
     * Loads signing context for {@code actorId}.
     *
     * @return {@code Optional.empty()} if not configured for signing (cached, no retry)
     * @throws RuntimeException for transient failures — NOT cached; next call retries
     */
    protected abstract Optional<C> loadContext(String actorId);

    /**
     * Performs the signing operation using the cached context.
     * Called only when context is present. Must not cache the result.
     */
    protected abstract AgentSignature performSign(String actorId, C context, byte[] data);

    /** Evicts all cached contexts. Next {@link #sign} call reloads from the source. */
    public void invalidateAll() {
        contextCache.clear();
    }

    /** Evicts the cached context for one actor. Next {@link #sign} call for this actor reloads. */
    public void invalidate(final String actorId) {
        contextCache.remove(actorId);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AbstractCachingAgentSignerTest -q
```

Expected: BUILD SUCCESS, 6 tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSigner.java runtime/src/main/java/io/casehub/ledger/runtime/service/AbstractCachingAgentSigner.java runtime/src/test/java/io/casehub/ledger/service/AbstractCachingAgentSignerTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#85): add AgentSigner SPI and AbstractCachingAgentSigner<C> base"
```

---

## Task 3: `LedgerPemUtil.parsePublicKey(String)`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerPemUtil.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/runtime/service/LedgerPemUtilTest.java`

- [ ] **Step 1: Add failing test to `LedgerPemUtilTest`**

Add this test to the existing `LedgerPemUtilTest` class (after existing tests):

```java
@Test
void parsePublicKey_fromPemString_roundTrip() throws Exception {
    final KeyPairGenerator kpg = KeyPairGenerator.getInstance("Ed25519");
    final KeyPair kp = kpg.generateKeyPair();
    final String pem = "-----BEGIN PUBLIC KEY-----\n"
            + Base64.getMimeEncoder(64, new byte[]{'\n'}).encodeToString(kp.getPublic().getEncoded())
            + "\n-----END PUBLIC KEY-----\n";

    final PublicKey loaded = LedgerPemUtil.parsePublicKey(pem);

    assertThat(loaded.getEncoded()).isEqualTo(kp.getPublic().getEncoded());
}
```

Add the missing import at the top: `import java.security.PublicKey;`

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerPemUtilTest -q 2>&1 | tail -5
```

Expected: compilation error — `parsePublicKey` not found.

- [ ] **Step 3: Add `parsePublicKey` to `LedgerPemUtil`**

Add this method to `LedgerPemUtil` (after the existing `loadPublicKey(String pemPath)` method):

```java
/**
 * Parses a PEM-encoded public key from a string (e.g. a Vault Transit API response).
 * Trial-loads through supported algorithms, same as {@link #loadPublicKey(String)}.
 *
 * @param pemContent PEM string containing {@code -----BEGIN PUBLIC KEY-----} header
 * @return parsed public key
 * @throws InvalidKeyException if no supported algorithm recognises the key
 */
public static PublicKey parsePublicKey(final String pemContent) throws Exception {
    final byte[] keyBytes = decodePem(pemContent, "PUBLIC KEY");
    final X509EncodedKeySpec spec = new X509EncodedKeySpec(keyBytes);
    for (final String algo : SUPPORTED_ALGORITHMS) {
        try {
            return KeyFactory.getInstance(algo).generatePublic(spec);
        } catch (final NoSuchAlgorithmException | InvalidKeySpecException ignored) {
        }
    }
    throw new InvalidKeyException(
            "Public key PEM does not match any supported algorithm: " + SUPPORTED_ALGORITHMS);
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerPemUtilTest -q
```

Expected: BUILD SUCCESS, 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerPemUtil.java runtime/src/test/java/io/casehub/ledger/runtime/service/LedgerPemUtilTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#85): add LedgerPemUtil.parsePublicKey(String) — parse from PEM string not file path"
```

---

## Task 4: `ConfiguredAgentSigner`

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/ConfiguredAgentSigner.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/ConfiguredAgentSignerTest.java`

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/ledger/service/ConfiguredAgentSignerTest.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.Signature;
import java.util.Base64;
import java.util.Map;
import java.util.Optional;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.service.AgentSignature;
import io.casehub.ledger.runtime.service.ConfiguredAgentSigner;

class ConfiguredAgentSignerTest {

    @TempDir Path tempDir;

    private Path writeKeyPem(final String filename, final String type, final byte[] encoded)
            throws Exception {
        final String pem = "-----BEGIN " + type + "-----\n"
                + Base64.getMimeEncoder(64, new byte[]{'\n'}).encodeToString(encoded)
                + "\n-----END " + type + "-----\n";
        final Path file = tempDir.resolve(filename);
        Files.writeString(file, pem);
        return file;
    }

    private LedgerConfig mockConfig(final Map<String, LedgerConfig.AgentSigningConfig.ActorKeyConfig> keys) {
        final LedgerConfig.AgentSigningConfig signingConfig = mock(LedgerConfig.AgentSigningConfig.class);
        when(signingConfig.keys()).thenReturn(keys);
        final LedgerConfig config = mock(LedgerConfig.class);
        when(config.agentSigning()).thenReturn(signingConfig);
        return config;
    }

    @Test
    void sign_returnsValidSignature_forConfiguredActor() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final Path privPath = writeKeyPem("priv.pem", "PRIVATE KEY", kp.getPrivate().getEncoded());
        final Path pubPath = writeKeyPem("pub.pem", "PUBLIC KEY", kp.getPublic().getEncoded());

        final LedgerConfig.AgentSigningConfig.ActorKeyConfig actorConfig =
                mock(LedgerConfig.AgentSigningConfig.ActorKeyConfig.class);
        when(actorConfig.privateKey()).thenReturn(privPath.toString());
        when(actorConfig.publicKey()).thenReturn(pubPath.toString());

        final ConfiguredAgentSigner signer =
                new ConfiguredAgentSigner(mockConfig(Map.of("claude:reviewer@v1", actorConfig)));
        signer.loadKeys();

        final byte[] data = "test canonical bytes".getBytes();
        final Optional<AgentSignature> result = signer.sign("claude:reviewer@v1", data);

        assertThat(result).isPresent();
        final Signature verifier = Signature.getInstance("Ed25519");
        verifier.initVerify(kp.getPublic());
        verifier.update(data);
        assertThat(verifier.verify(result.get().signature())).isTrue();
        assertThat(result.get().publicKey()).isEqualTo(kp.getPublic().getEncoded());
    }

    @Test
    void sign_returnsEmpty_forUnconfiguredActor() {
        final ConfiguredAgentSigner signer = new ConfiguredAgentSigner(mockConfig(Map.of()));
        signer.loadKeys();
        assertThat(signer.sign("unknown-actor", new byte[]{1})).isEmpty();
    }

    @Test
    void sign_returnsEmpty_forFailedKeyLoad_neverThrows() throws Exception {
        final LedgerConfig.AgentSigningConfig.ActorKeyConfig actorConfig =
                mock(LedgerConfig.AgentSigningConfig.ActorKeyConfig.class);
        when(actorConfig.privateKey()).thenReturn("/does/not/exist.pem");
        when(actorConfig.publicKey()).thenReturn("/does/not/exist.pem");

        final ConfiguredAgentSigner signer =
                new ConfiguredAgentSigner(mockConfig(Map.of("bad-actor", actorConfig)));
        signer.loadKeys();  // logs error, adds to failedActors

        // Subsequent sign() calls return empty without throwing
        assertThat(signer.sign("bad-actor", new byte[]{1})).isEmpty();
        assertThat(signer.sign("bad-actor", new byte[]{2})).isEmpty();
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ConfiguredAgentSignerTest -q 2>&1 | tail -5
```

Expected: compilation error — `ConfiguredAgentSigner` not found.

- [ ] **Step 3: Create `ConfiguredAgentSigner.java`**

```java
package io.casehub.ledger.runtime.service;

import java.security.KeyPair;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.quarkus.arc.DefaultBean;

import io.casehub.ledger.runtime.config.LedgerConfig;

/**
 * Default {@link AgentSigner} — loads key pairs from PEM files configured under
 * {@code casehub.ledger.agent-signing.keys.*} at startup.
 *
 * <p>Signing is opt-in per actor. Actors without a configured key pair produce unsigned entries.
 * Actors whose key files fail to load at startup are tracked in {@code failedActors}:
 * a single error is logged at startup, and subsequent {@link #sign} calls return empty without
 * further logging — no per-call log storm for misconfigured actors.
 */
@DefaultBean
@ApplicationScoped
public class ConfiguredAgentSigner implements AgentSigner {

    private static final Logger LOG = Logger.getLogger(ConfiguredAgentSigner.class);

    private final LedgerConfig config;
    private final Map<String, KeyPair> signingKeys = new ConcurrentHashMap<>();
    private final Set<String> failedActors = ConcurrentHashMap.newKeySet();

    @Inject
    public ConfiguredAgentSigner(final LedgerConfig config) {
        this.config = config;
    }

    @PostConstruct
    void loadKeys() {
        config.agentSigning().keys().forEach((actorId, keyConfig) -> {
            try {
                final PrivateKey priv = LedgerPemUtil.loadPrivateKey(keyConfig.privateKey());
                final PublicKey pub = LedgerPemUtil.loadPublicKey(keyConfig.publicKey());
                signingKeys.put(actorId, new KeyPair(pub, priv));
                LOG.infof("Loaded signing key for actor %s", actorId);
            } catch (final Exception e) {
                failedActors.add(actorId);
                LOG.errorf("Failed to load signing key for actor %s: %s — entries will be unsigned",
                        actorId, e.getMessage());
            }
        });
    }

    @Override
    public Optional<AgentSignature> sign(final String actorId, final byte[] data) {
        if (failedActors.contains(actorId)) {
            return Optional.empty();
        }
        final KeyPair kp = signingKeys.get(actorId);
        if (kp == null) {
            return Optional.empty();
        }
        return Optional.of(AgentSignature.signWith(kp, data));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ConfiguredAgentSignerTest -q
```

Expected: BUILD SUCCESS, 3 tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/ConfiguredAgentSigner.java runtime/src/test/java/io/casehub/ledger/service/ConfiguredAgentSignerTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#85): add ConfiguredAgentSigner — PEM-based AgentSigner default bean"
```

---

## Task 5: Wire up `AgentSignatureEnricher` + update all affected tests

These changes must be done together — updating the enricher without updating the tests that mock `AgentKeyProvider` leaves the tests broken. Compile and run at the end.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AgentSigningIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureVerificationServiceIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/ReactiveAgentSignatureVerificationServiceIT.java`

- [ ] **Step 1: Update `AgentSignatureEnricher.java`**

Replace the entire file content:

```java
package io.casehub.ledger.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * {@link LedgerEntryEnricher} that signs each entry via the configured {@link AgentSigner}.
 *
 * <p>Signs {@link LedgerMerkleTree#canonicalBytes(LedgerEntry)} — the same canonical form
 * used for Merkle leaf hashes, so the signature covers exactly the tamper-evident fields.
 *
 * <p>No-op when the actor has no configured signing key. Non-fatal — exceptions from
 * the signer are swallowed so a key store failure never blocks a ledger write.
 */
@ApplicationScoped
public class AgentSignatureEnricher implements LedgerEntryEnricher {

    private static final Logger LOG = Logger.getLogger(AgentSignatureEnricher.class);

    private final AgentSigner signer;

    @Inject
    public AgentSignatureEnricher(final AgentSigner signer) {
        this.signer = signer;
    }

    @Override
    public void enrich(final LedgerEntry entry) {
        if (entry.actorId == null || entry.agentSignature != null) return;
        try {
            signer.sign(entry.actorId, LedgerMerkleTree.canonicalBytes(entry))
                    .ifPresent(sig -> {
                        entry.agentSignature = sig.signature();
                        entry.agentPublicKey = sig.publicKey();
                        entry.agentKeyRef = sig.keyRef();
                    });
        } catch (final Exception e) {
            LOG.warnf("AgentSignatureEnricher failed for actor %s — entry will be unsigned: %s",
                    entry.actorId, e.getMessage());
        }
    }
}
```

- [ ] **Step 2: Update `AgentSignatureEnricherTest.java`**

Replace the file content (update imports and lambda constructions):

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.Signature;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.AgentSignature;
import io.casehub.ledger.runtime.service.AgentSignatureEnricher;
import io.casehub.ledger.runtime.service.LedgerMerkleTree;
import io.casehub.ledger.service.supplement.TestEntry;

class AgentSignatureEnricherTest {

    private KeyPair testKeyPair;
    private AgentSignatureEnricher enricher;

    @BeforeEach
    void setUp() throws Exception {
        testKeyPair = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
    }

    private TestEntry entry(final String actorId) {
        final TestEntry e = new TestEntry();
        e.id = UUID.randomUUID();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.actorRole = "Reviewer";
        e.occurredAt = Instant.now();
        return e;
    }

    @Test
    void populatesSignatureAndPublicKey_whenActorHasKey() {
        final KeyPair kp = testKeyPair;
        enricher = new AgentSignatureEnricher(
                (actorId, data) -> Optional.of(AgentSignature.signWith(kp, data)));

        final TestEntry e = entry("claude:reviewer@v1");
        enricher.enrich(e);

        assertThat(e.agentSignature).isNotNull().hasSizeGreaterThan(0);
        assertThat(e.agentPublicKey).isNotNull().isEqualTo(kp.getPublic().getEncoded());
    }

    @Test
    void signatureVerifiesAgainstStoredPublicKey() throws Exception {
        enricher = new AgentSignatureEnricher(
                (actorId, data) -> Optional.of(AgentSignature.signWith(testKeyPair, data)));

        final TestEntry e = entry("claude:reviewer@v1");
        enricher.enrich(e);

        final Signature sig = Signature.getInstance("Ed25519");
        sig.initVerify(testKeyPair.getPublic());
        sig.update(LedgerMerkleTree.canonicalBytes(e));
        assertThat(sig.verify(e.agentSignature)).isTrue();
    }

    @Test
    void leavesFieldsNull_whenActorHasNoKey() {
        enricher = new AgentSignatureEnricher((actorId, data) -> Optional.empty());

        final TestEntry e = entry("unknown-actor");
        enricher.enrich(e);

        assertThat(e.agentSignature).isNull();
        assertThat(e.agentPublicKey).isNull();
    }

    @Test
    void leavesFieldsNull_whenActorIdIsNull() {
        enricher = new AgentSignatureEnricher(
                (actorId, data) -> Optional.of(AgentSignature.signWith(testKeyPair, data)));

        final TestEntry e = entry(null);
        enricher.enrich(e);

        assertThat(e.agentSignature).isNull();
        assertThat(e.agentPublicKey).isNull();
    }

    @Test
    void isIdempotent_secondCallIsNoOp() throws Exception {
        enricher = new AgentSignatureEnricher(
                (actorId, data) -> Optional.of(AgentSignature.signWith(testKeyPair, data)));

        final TestEntry e = entry("claude:reviewer@v1");
        enricher.enrich(e);
        final byte[] firstSig = e.agentSignature.clone();
        final byte[] firstKey = e.agentPublicKey.clone();

        final KeyPair otherPair = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        enricher = new AgentSignatureEnricher(
                (actorId, data) -> Optional.of(AgentSignature.signWith(otherPair, data)));
        enricher.enrich(e);

        assertThat(e.agentSignature).isEqualTo(firstSig);
        assertThat(e.agentPublicKey).isEqualTo(firstKey);
    }

    @Test
    void doesNotThrow_whenSignerThrows() {
        enricher = new AgentSignatureEnricher((actorId, data) -> {
            throw new RuntimeException("key store unavailable");
        });

        final TestEntry e = entry("claude:reviewer@v1");
        assertThatCode(() -> enricher.enrich(e)).doesNotThrowAnyException();
        assertThat(e.agentSignature).isNull();
    }

    @Test
    void populatesAgentKeyRef_fromSignerResult() {
        enricher = new AgentSignatureEnricher(
                (actorId, data) -> Optional.of(AgentSignature.signWith(testKeyPair, data)));

        final TestEntry e = entry("claude:reviewer@v1");
        enricher.enrich(e);

        assertThat(e.agentKeyRef).isNotNull().hasSize(43);
        assertThat(e.agentKeyRef).matches("[A-Za-z0-9_-]+");
    }

    @Test
    void agentKeyRef_isNullWhenUnsigned() {
        enricher = new AgentSignatureEnricher((actorId, data) -> Optional.empty());

        final TestEntry e = entry("unknown-actor");
        enricher.enrich(e);

        assertThat(e.agentKeyRef).isNull();
    }
}
```

- [ ] **Step 3: Update `AgentSigningIT.java`**

Replace imports and mock setup. Find and replace these blocks:

Remove import: `import io.casehub.ledger.runtime.service.AgentKeyProvider;`
Remove import: `import io.casehub.ledger.runtime.service.SigningKey;`
Add import: `import io.casehub.ledger.runtime.service.AgentSignature;`
Add import: `import io.casehub.ledger.runtime.service.AgentSigner;`
Add import: `import static org.mockito.ArgumentMatchers.any;`

Replace the field declaration:
```java
// Remove:
@InjectMock
AgentKeyProvider agentKeyProvider;

// Add:
@InjectMock
AgentSigner agentSigner;
```

Replace `@BeforeEach setUp()`:
```java
@BeforeEach
void setUp() throws Exception {
    testKeyPair = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
    when(agentSigner.sign(anyString(), any())).thenReturn(Optional.empty());
    when(agentSigner.sign(eq("claude:reviewer@v1"), any()))
            .thenAnswer(inv -> Optional.of(AgentSignature.signWith(testKeyPair, inv.getArgument(1))));
}
```

Add `import static org.mockito.ArgumentMatchers.eq;` if not present.

- [ ] **Step 4: Update `AgentSignatureVerificationServiceIT.java`**

Add a private helper method (before the first `@Test`):
```java
private static AgentSignature signEntry(final TestEntry e, final KeyPair kp) {
    final AgentSignature sig = AgentSignature.signWith(kp, LedgerMerkleTree.canonicalBytes(e));
    e.agentSignature = sig.signature();
    e.agentPublicKey = sig.publicKey();
    e.agentKeyRef = sig.keyRef();
    return sig;
}
```

Remove import: `import io.casehub.ledger.runtime.service.SigningKey;`
Add import: `import io.casehub.ledger.runtime.service.AgentSignature;`

For each test that currently does:
```java
final SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
// then manually signs and sets entry fields
```

Replace with:
```java
final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
final AgentSignature agentSig = signEntry(e, kp);
```

Then replace `sk.keyRef()` with `agentSig.keyRef()` in `rotationService.recordRotation(...)` calls.

For the tampered-signature test that does `signature[0] ^= 0xFF` before setting on the entry:
```java
// Old pattern:
final byte[] signature = sig.sign();
signature[0] ^= 0xFF;
e.agentSignature = signature;

// New pattern — sign normally, set fields, then tamper the entity field:
final AgentSignature agentSig = signEntry(e, kp);
e.agentSignature[0] ^= 0xFF;  // tamper the entity's copy directly
```

Note: `e.agentSignature` is a plain `byte[]` field on the entity — mutating it is correct.

- [ ] **Step 5: Update `ReactiveAgentSignatureVerificationServiceIT.java`**

Remove imports:
```java
import io.casehub.ledger.runtime.service.AgentKeyProvider;
import io.casehub.ledger.runtime.service.SigningKey;
```

Add imports:
```java
import io.casehub.ledger.runtime.service.AgentSignature;
import io.casehub.ledger.runtime.service.AgentSigner;
import static org.mockito.ArgumentMatchers.any;
```

Replace field:
```java
// Remove:
@InjectMock
AgentKeyProvider agentKeyProvider;

// Add:
@InjectMock
AgentSigner agentSigner;
```

Replace `@BeforeEach setUp()`:
```java
@BeforeEach
void setUp() throws Exception {
    when(agentSigner.sign(anyString(), any())).thenReturn(Optional.empty());
    eventCapture.reset();
}
```

Add private helper (same as above):
```java
private static AgentSignature signEntry(final TestEntry e, final KeyPair kp) {
    final AgentSignature sig = AgentSignature.signWith(kp, LedgerMerkleTree.canonicalBytes(e));
    e.agentSignature = sig.signature();
    e.agentPublicKey = sig.publicKey();
    e.agentKeyRef = sig.keyRef();
    return sig;
}
```

Replace all `SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair())` with:
```java
final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
```

Then replace the manual JCA signing + field-setting blocks with `signEntry(e, kp)`, and replace `sk.keyRef()` with the returned `AgentSignature`'s `.keyRef()` where used in `rotationService.recordRotation(...)`.

- [ ] **Step 6: Compile to verify no errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile test-compile -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Run affected tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="AgentSignatureEnricherTest,AgentSigningIT,AgentSignatureVerificationServiceIT,ReactiveAgentSignatureVerificationServiceIT" -q
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java runtime/src/test/java/io/casehub/ledger/service/AgentSigningIT.java runtime/src/test/java/io/casehub/ledger/service/AgentSignatureVerificationServiceIT.java runtime/src/test/java/io/casehub/ledger/service/ReactiveAgentSignatureVerificationServiceIT.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#85): wire AgentSignatureEnricher to AgentSigner; update all signing tests"
```

---

## Task 6: Delete old types + full test run

**Files:**
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyProvider.java`
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/service/SigningKey.java`
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/service/ConfiguredAgentKeyProvider.java`

- [ ] **Step 1: Delete old source files**

```bash
rm /Users/mdproctor/claude/casehub/ledger/runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyProvider.java
rm /Users/mdproctor/claude/casehub/ledger/runtime/src/main/java/io/casehub/ledger/runtime/service/SigningKey.java
rm /Users/mdproctor/claude/casehub/ledger/runtime/src/main/java/io/casehub/ledger/runtime/service/ConfiguredAgentKeyProvider.java
```

- [ ] **Step 2: Compile to check for missed references**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile test-compile -pl runtime -q 2>&1 | grep "error:" | head -20
```

Expected: no output (BUILD SUCCESS). If errors appear, search for remaining references:
```bash
grep -r "AgentKeyProvider\|SigningKey\|ConfiguredAgentKeyProvider" /Users/mdproctor/claude/casehub/ledger/runtime/src --include="*.java"
```

Fix any remaining references before continuing.

- [ ] **Step 3: Run full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```

Expected: BUILD SUCCESS. Note the test count — should be ≥ previous count (new tests added).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add -u runtime/src/main/java/io/casehub/ledger/runtime/service/
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#85): delete AgentKeyProvider, SigningKey, ConfiguredAgentKeyProvider — replaced by AgentSigner SPI"
```

---

## Task 7: ADR 0014

**Files:**
- Create: `docs/adr/0014-agent-signer-spi.md`

- [ ] **Step 1: Write the ADR**

Create `docs/adr/0014-agent-signer-spi.md`:

```markdown
# ADR 0014 — AgentSigner SPI: Signing Responsibility Belongs in the SPI

**Status:** Accepted  
**Date:** 2026-05-29  
**Issue:** casehubio/ledger#85

## Context

ADR 0011 established per-actorId key pairs via `AgentKeyProvider.signingKey(actorId) →
Optional<SigningKey>`. `AgentSignatureEnricher` then performed local JCA signing with the
returned `KeyPair`. This model assumes the caller can access the private key — it breaks
for:

- **Vault Transit** — private key never leaves Vault; signing is a remote API call
- **Cloud KMS** (AWS KMS, GCP KMS, Azure Key Vault) — same remote-signing model
- **HSMs via non-JCA API** — hardware signing requires an API call, not local `Signature.getInstance()`

PKCS#11-backed HSMs that expose a JCA `Provider` return a `PrivateKey` handle that routes
signing into hardware without exporting key material — these are compatible with `AgentKeyProvider`.
The incompatible case is remote REST-based signers (Vault Transit, Cloud KMS).

## Decision

Replace `AgentKeyProvider` with `AgentSigner`:

```java
public interface AgentSigner {
    Optional<AgentSignature> sign(String actorId, byte[] data);
}

public record AgentSignature(byte[] signature, byte[] publicKey, String keyRef) { ... }
```

`AgentSignatureEnricher` calls `signer.sign()` and receives the complete result —
signature bytes, X.509 public key, and keyRef. The signing responsibility moves from
the enricher into the SPI.

## Two-Tier Implementation Structure

**`ConfiguredAgentSigner`** (`@DefaultBean`) — replaces `ConfiguredAgentKeyProvider`.
Direct implementation, NOT extending the abstract base. Eager `@PostConstruct` PEM loading
preserves startup-time misconfiguration visibility. Signs locally via `AgentSignature.signWith(KeyPair, byte[])`.

**`AbstractCachingAgentSigner<C>`** — abstract base for external providers with network/hardware overhead.
Per-actorId context cache (`ConcurrentHashMap<String, Optional<C>>`). Template methods:
`loadContext(actorId) → Optional<C>` (null = not configured, cached; throw = transient error, not cached)
and `performSign(actorId, context, data) → AgentSignature`. Invalidation hooks: `invalidateAll()`,
`invalidate(actorId)`.

## Vault Transit Reference Implementation

`VaultTransitAgentSigner` in `examples/vault-transit-signing/` extends `AbstractCachingAgentSigner<VaultTransitContext>`.
`loadContext`: `GET /v1/transit/keys/<key-name>` → parses public key, caches.
`performSign`: `POST /v1/transit/sign/<key-name>` → strips `vault:v1:` prefix, returns `AgentSignature`.
Scheduled `invalidateAll()` for bounded-staleness refresh (default 5m).

## Breaking Change

`AgentKeyProvider`, `SigningKey`, and `ConfiguredAgentKeyProvider` are deleted. This is
intentional — there are no external consumers of `casehub-ledger` 0.2-SNAPSHOT.

## Extends

ADR 0011 — the per-actorId key model is unchanged and correct. This ADR replaces only the
SPI contract that implements it. The `signWith(KeyPair, byte[])` factory on `AgentSignature`
is algorithm-transparent (derives algorithm from `keyPair.getPrivate().getAlgorithm()`),
consistent with protocol PP-20260523-e7b577.

## Deferred

- #101 Vault AppRole/OIDC auth  
- #102 Cloud KMS adapters (AWS KMS, GCP, Azure)  
- #103 Rotation-triggered cache invalidation via CDI event  
- #104 `InMemoryAgentSigner` in `persistence-memory/`

## Related

- ADR 0011 — per-actorId key model
- ADR 0012 — key rotation design
- ADR 0013 — post-quantum algorithm migration
- Protocol PP-20260523-e7b577 — algorithm-transparent signing
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add docs/adr/0014-agent-signer-spi.md
git -C /Users/mdproctor/claude/casehub/ledger commit -m "docs(#85): ADR 0014 — AgentSigner SPI (extends ADR 0011)"
```

---

## Task 8: Example module — `vault-transit-signing` — bootstrap and WireMock tests

**Files:**
- Create: `examples/vault-transit-signing/pom.xml`
- Create: `examples/vault-transit-signing/src/main/resources/application.properties`
- Create: `examples/vault-transit-signing/src/test/resources/application.properties`
- Create: `examples/vault-transit-signing/src/test/java/io/casehub/ledger/examples/vault/VaultTransitAgentSignerIT.java`
- Create: `examples/vault-transit-signing/README.md`
- Modify: `pom.xml` (root — add to `with-examples` profile)

- [ ] **Step 1: Create `examples/vault-transit-signing/pom.xml`**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>io.casehub.ledger.examples</groupId>
  <artifactId>casehub-ledger-example-vault-transit-signing</artifactId>
  <version>0.2-SNAPSHOT</version>

  <name>CaseHub Ledger - Example: Vault Transit Signing</name>
  <description>
    Demonstrates remote signing via HashiCorp Vault Transit Secrets Engine.
    The private key never leaves Vault; signing happens via REST API call.
    AbstractCachingAgentSigner caches the public key per actorId with scheduled refresh.
    Run tests with: mvn test
  </description>

  <properties>
    <quarkus.version>3.32.2</quarkus.version>
    <casehub-ledger.version>0.2-SNAPSHOT</casehub-ledger.version>
    <wiremock.version>3.4.2</wiremock.version>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <surefire-plugin.version>3.2.5</surefire-plugin.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>io.quarkus.platform</groupId>
        <artifactId>quarkus-bom</artifactId>
        <version>${quarkus.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger</artifactId>
      <version>${casehub-ledger.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-scheduler</artifactId>
    </dependency>
    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.wiremock</groupId>
      <artifactId>wiremock</artifactId>
      <version>${wiremock.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <repositories>
    <repository>
      <id>github</id>
      <url>https://maven.pkg.github.com/casehubio/*</url>
      <snapshots><enabled>true</enabled></snapshots>
    </repository>
  </repositories>

  <build>
    <plugins>
      <plugin>
        <groupId>io.quarkus.platform</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${quarkus.version}</version>
        <extensions>true</extensions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>${surefire-plugin.version}</version>
        <configuration>
          <includes>
            <include>**/*Test.java</include>
            <include>**/*IT.java</include>
          </includes>
          <systemPropertyVariables>
            <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
          </systemPropertyVariables>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Create `src/main/resources/application.properties`**

```properties
# Vault Transit AgentSigner configuration
# Override these for your Vault deployment
casehub.ledger.vault-transit.address=http://localhost:8200
casehub.ledger.vault-transit.token=root
casehub.ledger.vault-transit.refresh-interval=5m
# Key mapping: actorId -> Vault Transit key name
# casehub.ledger.vault-transit.key-mapping."claude:reviewer@v1"=reviewer-signing-key
```

- [ ] **Step 3: Create `src/test/resources/application.properties`**

```properties
# WireMock server runs on port 8099 (started in VaultTransitAgentSignerIT)
casehub.ledger.vault-transit.address=http://localhost:8099
casehub.ledger.vault-transit.token=test-token
casehub.ledger.vault-transit.key-mapping."claude:reviewer@v1"=reviewer-key
casehub.ledger.vault-transit.refresh-interval=24h
```

- [ ] **Step 4: Write the failing integration test**

Create `src/test/java/io/casehub/ledger/examples/vault/VaultTransitAgentSignerIT.java`:

```java
package io.casehub.ledger.examples.vault;

import static com.github.tomakehurst.wiremock.client.WireMock.anyRequestedFor;
import static com.github.tomakehurst.wiremock.client.WireMock.anyUrl;
import static com.github.tomakehurst.wiremock.client.WireMock.equalTo;
import static com.github.tomakehurst.wiremock.client.WireMock.forbidden;
import static com.github.tomakehurst.wiremock.client.WireMock.get;
import static com.github.tomakehurst.wiremock.client.WireMock.okJson;
import static com.github.tomakehurst.wiremock.client.WireMock.post;
import static com.github.tomakehurst.wiremock.client.WireMock.urlEqualTo;
import static com.github.tomakehurst.wiremock.client.WireMock.verify;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.Signature;
import java.util.Base64;
import java.util.Optional;

import jakarta.inject.Inject;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.github.tomakehurst.wiremock.WireMockServer;

import io.casehub.ledger.runtime.service.AgentSignature;
import io.casehub.ledger.runtime.service.AgentSigner;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class VaultTransitAgentSignerIT {

    static WireMockServer wireMock;

    @BeforeAll
    static void startWireMock() {
        wireMock = new WireMockServer(8099);
        wireMock.start();
    }

    @AfterAll
    static void stopWireMock() {
        wireMock.stop();
    }

    @BeforeEach
    void resetWireMock() {
        wireMock.resetAll();
        // Invalidate cache between tests so each test gets a fresh loadContext() call
        ((VaultTransitAgentSigner) agentSigner).invalidateAll();
    }

    @Inject
    AgentSigner agentSigner;

    // Helpers for building Vault API responses

    /** Returns the public key PEM as a Java string with real newlines. */
    private static String publicKeyPem(final KeyPair kp) {
        return "-----BEGIN PUBLIC KEY-----\n"
                + Base64.getMimeEncoder(64, new byte[]{'\n'})
                        .encodeToString(kp.getPublic().getEncoded())
                + "\n-----END PUBLIC KEY-----\n";
    }

    /**
     * Builds a Vault Transit key-info JSON response body.
     * Newlines in the PEM are escaped as {@code \n} (JSON escape) so the JSON is valid.
     * Jackson's {@code asText()} will un-escape them back to real newlines when parsed.
     */
    private static String keyInfoResponse(final KeyPair kp) {
        final String pemJsonSafe = publicKeyPem(kp)
                .replace("\\", "\\\\")
                .replace("\"", "\\\"")
                .replace("\n", "\\n");
        return "{\"data\":{\"keys\":{\"1\":{\"public_key\":\"" + pemJsonSafe + "\"}}}}";
    }

    private static String signResponse(final byte[] sigBytes) {
        return "{\"data\":{\"signature\":\"vault:v1:" +
                Base64.getEncoder().encodeToString(sigBytes) + "\"}}";
    }

    private static void stubKeyInfo(final KeyPair kp) {
        wireMock.stubFor(get(urlEqualTo("/v1/transit/keys/reviewer-key"))
                .withHeader("X-Vault-Token", equalTo("test-token"))
                .willReturn(okJson(keyInfoResponse(kp))));
    }

    private static byte[] realSign(final KeyPair kp, final byte[] data) throws Exception {
        final Signature sig = Signature.getInstance("Ed25519");
        sig.initSign(kp.getPrivate());
        sig.update(data);
        return sig.sign();
    }

    private static void stubSign(final KeyPair kp, final byte[] data) throws Exception {
        final byte[] sigBytes = realSign(kp, data);
        wireMock.stubFor(post(urlEqualTo("/v1/transit/sign/reviewer-key"))
                .withHeader("X-Vault-Token", equalTo("test-token"))
                .willReturn(okJson(signResponse(sigBytes))));
    }

    @Test
    void signsData_viaVaultTransitApi() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final byte[] data = "canonical ledger bytes".getBytes();
        stubKeyInfo(kp);
        stubSign(kp, data);

        final Optional<AgentSignature> result = agentSigner.sign("claude:reviewer@v1", data);

        assertThat(result).isPresent();
        assertThat(result.get().publicKey()).isEqualTo(kp.getPublic().getEncoded());
        // Verify the returned signature is valid Ed25519
        final Signature verifier = Signature.getInstance("Ed25519");
        verifier.initVerify(kp.getPublic());
        verifier.update(data);
        assertThat(verifier.verify(result.get().signature())).isTrue();
    }

    @Test
    void cachesPublicKey_secondSignCallDoesNotRefetchKeyInfo() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final byte[] data1 = "data one".getBytes();
        final byte[] data2 = "data two".getBytes();
        stubKeyInfo(kp);
        // Stub sign for both calls
        wireMock.stubFor(post(urlEqualTo("/v1/transit/sign/reviewer-key"))
                .willReturn(okJson(signResponse(realSign(kp, data1)))));

        agentSigner.sign("claude:reviewer@v1", data1);
        agentSigner.sign("claude:reviewer@v1", data2);

        // Key info endpoint should only be called once (context cached)
        wireMock.verify(1, get(urlEqualTo("/v1/transit/keys/reviewer-key")));
    }

    @Test
    void returnsEmpty_forUnmappedActor() {
        final Optional<AgentSignature> result = agentSigner.sign("unmapped-actor", new byte[]{1});
        assertThat(result).isEmpty();
        // No Vault calls for unmapped actors
        wireMock.verify(0, anyRequestedFor(anyUrl()));
    }

    @Test
    void throwsOnVault403_notCached_retrySucceeds() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        wireMock.stubFor(get(urlEqualTo("/v1/transit/keys/reviewer-key"))
                .willReturn(forbidden()));

        assertThatThrownBy(() -> agentSigner.sign("claude:reviewer@v1", new byte[]{1}))
                .isInstanceOf(RuntimeException.class);

        // Reset and stub success — error was not cached so retry should work
        wireMock.resetAll();
        final byte[] data = "retry data".getBytes();
        stubKeyInfo(kp);
        stubSign(kp, data);

        final Optional<AgentSignature> result = agentSigner.sign("claude:reviewer@v1", data);
        assertThat(result).isPresent();
    }
}
```

- [ ] **Step 5: Add example to root `pom.xml` `with-examples` profile**

In `/Users/mdproctor/claude/casehub/ledger/pom.xml`, find the `with-examples` profile and add:

```xml
<module>examples/vault-transit-signing</module>
```

The profile block should look like:
```xml
<profile>
  <id>with-examples</id>
  <modules>
    <module>examples/order-processing</module>
    <module>examples/trust-score-routing</module>
    <module>examples/vault-transit-signing</module>
  </modules>
</profile>
```

- [ ] **Step 6: Create minimal `README.md`**

Create `examples/vault-transit-signing/README.md`:

```markdown
# Example: Vault Transit Remote Signing

Demonstrates `VaultTransitAgentSigner` — an `AgentSigner` implementation where the private
key never leaves HashiCorp Vault. Signing happens via the Vault Transit Secrets Engine REST API.

## Pattern

`VaultTransitAgentSigner` extends `AbstractCachingAgentSigner<VaultTransitContext>`:

- `loadContext(actorId)` → `GET /v1/transit/keys/<key-name>` — fetches and caches public key
- `performSign(actorId, context, data)` → `POST /v1/transit/sign/<key-name>` — remote sign

The private key **never leaves Vault**. Only the public key is cached locally for storage
on `LedgerEntry.agentPublicKey` (needed for self-contained verification via `AgentCryptographicVerifier`).

## Signature format

Vault Transit ed25519: 64 raw bytes after stripping `vault:v1:` prefix and base64-decoding.
Vault Transit ECDSA (ecdsa-p256): ASN.1 DER — same format JCA expects, no conversion needed.

## Auth

This example uses a static token (`casehub.ledger.vault-transit.token`).
Production deployments should use AppRole or OIDC — see issue #101.

## PKCS#11 HSMs via JCA

For HSMs exposing a JCA Provider, you do NOT need `VaultTransitAgentSigner`. Use
`KeyStore.getInstance("PKCS11")` to load a `KeyPair` where the `PrivateKey` is an HSM-backed
handle. JCA routes signing into hardware without exporting key material. Extend
`AbstractCachingAgentSigner<KeyPair>` and call `AgentSignature.signWith(keyPair, data)`.
```

- [ ] **Step 7: Compile the example module to verify pom is correct**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl examples/vault-transit-signing -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS (source directories exist but no Java files yet — that's fine).

---

## Task 9: Implement `VaultTransitAgentSigner`

**Files:**
- Create: `examples/vault-transit-signing/src/main/java/io/casehub/ledger/examples/vault/VaultTransitConfig.java`
- Create: `examples/vault-transit-signing/src/main/java/io/casehub/ledger/examples/vault/VaultTransitAgentSigner.java`

- [ ] **Step 1: Run the IT to confirm it fails** (class not found)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/vault-transit-signing -Dtest=VaultTransitAgentSignerIT -q 2>&1 | tail -5
```

Expected: compilation error — `VaultTransitAgentSigner` not found.

- [ ] **Step 2: Create `VaultTransitConfig.java`**

```java
package io.casehub.ledger.examples.vault;

import java.util.Map;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.ledger.vault-transit")
public interface VaultTransitConfig {

    /** Base URL of the Vault server, e.g. {@code http://vault:8200}. */
    String address();

    /** Vault token for authentication. See issue #101 for AppRole/OIDC alternatives. */
    String token();

    /**
     * Map of actorId → Vault Transit key name.
     * Example: {@code casehub.ledger.vault-transit.key-mapping."claude:reviewer@v1"=reviewer-key}
     */
    Map<String, String> keyMapping();

    /**
     * Cache invalidation interval. Context cache is cleared on this schedule so rotated
     * keys take effect within this window. See issue #103 for event-driven invalidation.
     */
    @WithDefault("5m")
    String refreshInterval();
}
```

- [ ] **Step 3: Create `VaultTransitAgentSigner.java`**

```java
package io.casehub.ledger.examples.vault;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.security.MessageDigest;
import java.security.PublicKey;
import java.util.Base64;
import java.util.Optional;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.ledger.runtime.service.AbstractCachingAgentSigner;
import io.casehub.ledger.runtime.service.AgentSignature;
import io.casehub.ledger.runtime.service.LedgerPemUtil;
import io.quarkus.scheduler.Scheduled;

/**
 * {@link io.casehub.ledger.runtime.service.AgentSigner} backed by HashiCorp Vault Transit.
 *
 * <p>Private key never leaves Vault. This signer:
 * <ol>
 *   <li>Fetches the public key from {@code GET /v1/transit/keys/<key-name>} on first use per actorId</li>
 *   <li>Signs by calling {@code POST /v1/transit/sign/<key-name>}</li>
 *   <li>Strips the {@code vault:v1:} prefix from the returned signature</li>
 * </ol>
 *
 * <p><strong>Auth:</strong> static token only. See issue #101 for AppRole/OIDC.
 *
 * <p><strong>Signature format:</strong>
 * Ed25519 — 64 raw bytes after stripping prefix and base64-decoding.
 * ECDSA (ecdsa-p256) — ASN.1 DER, same format JCA expects, no conversion needed.
 *
 * <p><strong>Algorithm transparency:</strong> no algorithm string is hardcoded.
 * The stored {@code agentPublicKey} X.509 bytes carry the OID; {@code AgentCryptographicVerifier}
 * detects the algorithm from those bytes.
 */
@ApplicationScoped
@Alternative
@Priority(1)
public class VaultTransitAgentSigner extends AbstractCachingAgentSigner<VaultTransitAgentSigner.VaultTransitContext> {

    private static final Logger LOG = Logger.getLogger(VaultTransitAgentSigner.class);
    private static final String VAULT_SIG_PREFIX = "vault:v1:";

    record VaultTransitContext(String vaultKeyName, byte[] publicKey, String keyRef) {}

    private final VaultTransitConfig config;
    private final HttpClient httpClient;
    private final ObjectMapper objectMapper;

    @Inject
    public VaultTransitAgentSigner(final VaultTransitConfig config, final ObjectMapper objectMapper) {
        this.config = config;
        this.httpClient = HttpClient.newHttpClient();
        this.objectMapper = objectMapper;
    }

    /** Clears context cache on schedule so rotated keys take effect within the refresh interval. */
    @Scheduled(every = "${casehub.ledger.vault-transit.refresh-interval:5m}")
    void refreshCache() {
        LOG.debug("Invalidating Vault Transit context cache");
        invalidateAll();
    }

    @Override
    protected Optional<VaultTransitContext> loadContext(final String actorId) {
        final String keyName = config.keyMapping().get(actorId);
        if (keyName == null) {
            return Optional.empty();
        }
        try {
            final String url = config.address() + "/v1/transit/keys/" + keyName;
            final HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(url))
                    .header("X-Vault-Token", config.token())
                    .GET()
                    .build();
            final HttpResponse<String> response =
                    httpClient.send(request, HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() != 200) {
                throw new RuntimeException(
                        "Vault key info failed for actor " + actorId + ": HTTP " + response.statusCode());
            }
            final String pemKey = parsePublicKeyPem(response.body());
            final PublicKey publicKey = LedgerPemUtil.parsePublicKey(pemKey);
            final byte[] pubEncoded = publicKey.getEncoded();
            final byte[] hash = MessageDigest.getInstance("SHA-256").digest(pubEncoded);
            final String keyRef = Base64.getUrlEncoder().withoutPadding().encodeToString(hash);
            LOG.infof("Loaded Vault Transit context for actor %s (key: %s)", actorId, keyName);
            return Optional.of(new VaultTransitContext(keyName, pubEncoded, keyRef));
        } catch (final RuntimeException e) {
            throw e;
        } catch (final Exception e) {
            throw new RuntimeException("Failed to load Vault Transit context for actor " + actorId, e);
        }
    }

    @Override
    protected AgentSignature performSign(
            final String actorId, final VaultTransitContext ctx, final byte[] data) {
        try {
            final String b64Input = Base64.getEncoder().encodeToString(data);
            final String requestBody = "{\"input\":\"" + b64Input + "\"}";
            final String url = config.address() + "/v1/transit/sign/" + ctx.vaultKeyName();
            final HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(url))
                    .header("X-Vault-Token", config.token())
                    .header("Content-Type", "application/json")
                    .POST(HttpRequest.BodyPublishers.ofString(requestBody))
                    .build();
            final HttpResponse<String> response =
                    httpClient.send(request, HttpResponse.BodyHandlers.ofString());
            if (response.statusCode() != 200) {
                throw new RuntimeException(
                        "Vault sign failed for actor " + actorId + ": HTTP " + response.statusCode());
            }
            final byte[] sigBytes = parseSignatureBytes(response.body());
            return new AgentSignature(sigBytes, ctx.publicKey(), ctx.keyRef());
        } catch (final RuntimeException e) {
            throw e;
        } catch (final Exception e) {
            throw new RuntimeException("Vault Transit signing failed for actor " + actorId, e);
        }
    }

    /** Parses the latest version public key PEM from a Vault Transit key info response. */
    private String parsePublicKeyPem(final String responseBody) throws Exception {
        final JsonNode root = objectMapper.readTree(responseBody);
        final JsonNode keys = root.path("data").path("keys");
        // Find the highest version number
        String latestPem = null;
        int latestVersion = -1;
        final var it = keys.fields();
        while (it.hasNext()) {
            final var entry = it.next();
            final int version = Integer.parseInt(entry.getKey());
            if (version > latestVersion) {
                latestVersion = version;
                latestPem = entry.getValue().path("public_key").asText();
            }
        }
        if (latestPem == null || latestPem.isBlank()) {
            throw new RuntimeException("No public key found in Vault Transit key info response");
        }
        return latestPem;
    }

    /** Strips the {@code vault:v1:} prefix and base64-decodes to raw signature bytes. */
    private byte[] parseSignatureBytes(final String responseBody) throws Exception {
        final JsonNode root = objectMapper.readTree(responseBody);
        final String vaultSig = root.path("data").path("signature").asText();
        if (!vaultSig.startsWith(VAULT_SIG_PREFIX)) {
            throw new RuntimeException("Unexpected Vault signature format: " + vaultSig);
        }
        return Base64.getDecoder().decode(vaultSig.substring(VAULT_SIG_PREFIX.length()));
    }
}
```

- [ ] **Step 4: Run the integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/vault-transit-signing -q
```

Expected: BUILD SUCCESS, all 4 `VaultTransitAgentSignerIT` tests pass.

`invalidateAll()` is `public` on `AbstractCachingAgentSigner` (set in Task 2), so the cast and call compile correctly from the IT class.

- [ ] **Step 5: Commit example module**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add examples/vault-transit-signing/ pom.xml
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#85): add vault-transit-signing example — remote signing via Vault Transit REST API"
```

---

## Task 10: Final verification

- [ ] **Step 1: Run complete runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 2: Run all modules including examples**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-examples -q
```

Expected: BUILD SUCCESS across all modules.

- [ ] **Step 3: Verify issue #85 is referenced in all commits**

```bash
git -C /Users/mdproctor/claude/casehub/ledger log --oneline origin/main..HEAD
```

All commits should reference `#85` in the message.
