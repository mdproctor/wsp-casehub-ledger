# Bilateral Entry Signing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add optional per-actorId Ed25519 agent signatures to `LedgerEntry`, enabling non-repudiation: the agent that authored an entry proves it by signing the canonical bytes, and the signature is verifiable forever from the stored public key.

**Architecture:** Two nullable columns (`agent_signature BYTEA`, `agent_public_key BYTEA`) are added to `ledger_entry`. An `AgentKeyProvider` SPI supplies `KeyPair` per actorId; the default implementation reads PKCS#8 private key + X.509 public key PEM paths from config. An `AgentSignatureEnricher` (existing `LedgerEntryEnricher` pipeline) signs at `@PrePersist`. `LedgerVerificationService.verifyAgentSignature()` verifies self-contained from stored bytes. Unsigned entries carry nulls — zero overhead for unconfigured actors.

**Tech Stack:** Java 21, Quarkus 3.32.2, Ed25519 (`java.security.Signature`), SmallRye Config `@ConfigMapping`, Flyway, JUnit 5, AssertJ, Mockito (`@InjectMock`)

---

## Task 1: Promote `canonicalBytes()` to `public static`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerkleTree.java:169`

This is the authoritative canonical form shared by Merkle leaf hashes and agent signatures. Making it public static removes the duplication risk.

- [ ] **Step 1: Change visibility**

In `LedgerMerkleTree.java`, change line 169:

```java
// Before
private static byte[] canonicalBytes(final LedgerEntry entry) {

// After
public static byte[] canonicalBytes(final LedgerEntry entry) {
```

- [ ] **Step 2: Verify existing tests still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerMerkleTreeTest -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerkleTree.java
git commit -m "refactor(#79): promote canonicalBytes() to public static — shared by Merkle and agent signing"
```

---

## Task 2: V1005 migration + `LedgerEntry` fields + `VerificationResult` enum

**Files:**
- Create: `runtime/src/main/resources/db/migration/V1005__agent_signature.sql`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/model/VerificationResult.java`

- [ ] **Step 1: Write the migration**

Create `runtime/src/main/resources/db/migration/V1005__agent_signature.sql`:

```sql
ALTER TABLE ledger_entry
    ADD COLUMN agent_signature  BYTEA,
    ADD COLUMN agent_public_key BYTEA;
```

- [ ] **Step 2: Add fields to `LedgerEntry`**

In `LedgerEntry.java`, add after the `supplementJson` field (before `// ── Supplement helpers`):

```java
// ── Agent signing ─────────────────────────────────────────────────────────

/**
 * Ed25519 signature of {@link io.casehub.ledger.runtime.service.LedgerMerkleTree#canonicalBytes(LedgerEntry)}
 * by the agent identified in {@link #actorId}.
 * Null when the actor is not configured for bilateral signing.
 */
@Column(name = "agent_signature")
public byte[] agentSignature;

/**
 * X.509-encoded Ed25519 public key of the signing agent.
 * Stored alongside the signature for self-contained verification —
 * entries remain verifiable without any external key management system.
 * Null when {@link #agentSignature} is null.
 */
@Column(name = "agent_public_key")
public byte[] agentPublicKey;
```

- [ ] **Step 3: Create `VerificationResult` enum**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/model/VerificationResult.java`:

```java
package io.casehub.ledger.runtime.service.model;

/**
 * Result of an agent signature verification on a {@link io.casehub.ledger.runtime.model.LedgerEntry}.
 */
public enum VerificationResult {

    /** No agent signature is stored on this entry — the actor did not sign. */
    UNSIGNED,

    /** Signature is present and verified against the stored public key and canonical bytes. */
    VALID,

    /** Signature is present but verification failed — possible tampering or key mismatch. */
    INVALID
}
```

- [ ] **Step 4: Verify schema applies cleanly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=PlainEntityTest -q
```

Expected: BUILD SUCCESS (Flyway applies V1005, Hibernate maps new columns)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/resources/db/migration/V1005__agent_signature.sql \
        runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/model/VerificationResult.java
git commit -m "feat(#79): V1005 agent_signature columns, LedgerEntry fields, VerificationResult enum"
```

---

## Task 3: `AgentKeyProvider` SPI + `ConfiguredAgentKeyProvider`

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyProvider.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/ConfiguredAgentKeyProvider.java`

- [ ] **Step 1: Create `AgentKeyProvider` SPI**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyProvider.java`:

```java
package io.casehub.ledger.runtime.service;

import java.security.KeyPair;
import java.util.Optional;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * SPI: supplies the Ed25519 {@link KeyPair} used to sign a {@link LedgerEntry}
 * on behalf of a given actorId.
 *
 * <p>
 * Return {@link Optional#empty()} for actors that do not participate in
 * bilateral signing — those entries will be persisted unsigned.
 *
 * <p>
 * Implementations must be {@code @ApplicationScoped} CDI beans. The default
 * implementation ({@link ConfiguredAgentKeyProvider}) reads key paths from
 * {@code casehub.ledger.agent-signing.keys.*} config.
 */
public interface AgentKeyProvider {

    /**
     * Returns the signing key pair for the given actorId, or empty if this
     * actor does not sign ledger entries.
     *
     * @param actorId the actor identity string (e.g. {@code "claude:reviewer@v1"})
     * @return signing key pair, or empty for unsigned actors
     */
    Optional<KeyPair> signingKeyPair(String actorId);
}
```

- [ ] **Step 2: Add `AgentSigningConfig` to `LedgerConfig`**

In `LedgerConfig.java`, add a new method after `health()`:

```java
/**
 * Per-actorId Ed25519 signing key configuration for bilateral entry signing.
 *
 * @return the agent signing sub-configuration
 */
AgentSigningConfig agentSigning();
```

And add the nested interface at the end of `LedgerConfig` (before the closing brace):

```java
/** Per-actorId bilateral signing key configuration. */
interface AgentSigningConfig {

    /**
     * Map of actorId → key file paths.
     * Key: actorId (e.g. {@code "claude:reviewer@v1"}).
     * Value: {@link ActorKeyConfig} with paths to the private and public key PEM files.
     *
     * <p>
     * Example configuration:
     * <pre>
     * casehub.ledger.agent-signing.keys."claude:reviewer@v1".private-key=/secrets/reviewer.private.pem
     * casehub.ledger.agent-signing.keys."claude:reviewer@v1".public-key=/secrets/reviewer.public.pem
     * </pre>
     */
    java.util.Map<String, ActorKeyConfig> keys();

    /** PEM file paths for one actor's signing key pair. */
    interface ActorKeyConfig {

        /**
         * Path to the PKCS#8 PEM file containing the Ed25519 private key.
         * Generate with: {@code openssl genpkey -algorithm Ed25519 -out actor.private.pem}
         */
        String privateKey();

        /**
         * Path to the X.509 PEM file containing the Ed25519 public key.
         * Generate with: {@code openssl pkey -in actor.private.pem -pubout -out actor.public.pem}
         */
        String publicKey();
    }
}
```

- [ ] **Step 3: Create `ConfiguredAgentKeyProvider`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/ConfiguredAgentKeyProvider.java`:

```java
package io.casehub.ledger.runtime.service;

import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyFactory;
import java.security.KeyPair;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.X509EncodedKeySpec;
import java.util.Base64;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.config.LedgerConfig;

/**
 * Default {@link AgentKeyProvider} — loads Ed25519 key pairs from PEM files
 * configured under {@code casehub.ledger.agent-signing.keys.*}.
 *
 * <p>
 * Returns {@link Optional#empty()} for any actorId not present in config,
 * making signing effectively opt-in per actor.
 */
@DefaultBean
@ApplicationScoped
public class ConfiguredAgentKeyProvider implements AgentKeyProvider {

    private static final Logger LOG = Logger.getLogger(ConfiguredAgentKeyProvider.class);

    @Inject
    LedgerConfig config;

    private final Map<String, KeyPair> keyPairs = new ConcurrentHashMap<>();

    @PostConstruct
    void loadKeys() {
        config.agentSigning().keys().forEach((actorId, keyConfig) -> {
            try {
                final PrivateKey priv = loadPrivateKey(keyConfig.privateKey());
                final PublicKey pub = loadPublicKey(keyConfig.publicKey());
                keyPairs.put(actorId, new KeyPair(pub, priv));
                LOG.infof("Loaded signing key pair for actor: %s", actorId);
            } catch (final Exception e) {
                LOG.errorf("Failed to load signing key for actor %s: %s", actorId, e.getMessage());
            }
        });
    }

    @Override
    public Optional<KeyPair> signingKeyPair(final String actorId) {
        return Optional.ofNullable(keyPairs.get(actorId));
    }

    private static PrivateKey loadPrivateKey(final String pemPath) throws Exception {
        final String pem = Files.readString(Path.of(pemPath));
        final byte[] keyBytes = decodePem(pem, "PRIVATE KEY");
        return KeyFactory.getInstance("Ed25519").generatePrivate(new PKCS8EncodedKeySpec(keyBytes));
    }

    private static PublicKey loadPublicKey(final String pemPath) throws Exception {
        final String pem = Files.readString(Path.of(pemPath));
        final byte[] keyBytes = decodePem(pem, "PUBLIC KEY");
        return KeyFactory.getInstance("Ed25519").generatePublic(new X509EncodedKeySpec(keyBytes));
    }

    private static byte[] decodePem(final String pem, final String type) {
        return Base64.getDecoder().decode(
            pem.replace("-----BEGIN " + type + "-----", "")
               .replace("-----END " + type + "-----", "")
               .replaceAll("\\s", ""));
    }
}
```

- [ ] **Step 4: Verify it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyProvider.java \
        runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/ConfiguredAgentKeyProvider.java
git commit -m "feat(#79): AgentKeyProvider SPI + ConfiguredAgentKeyProvider — per-actorId Ed25519 key loading"
```

---

## Task 4: `AgentSignatureEnricher` — tests first, then implementation

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java`

- [ ] **Step 1: Write failing tests**

Create `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java`:

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

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.AgentKeyProvider;
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
        enricher = new AgentSignatureEnricher(actorId -> Optional.of(kp));

        final TestEntry e = entry("claude:reviewer@v1");
        enricher.enrich(e);

        assertThat(e.agentSignature).isNotNull().hasSizeGreaterThan(0);
        assertThat(e.agentPublicKey).isNotNull()
            .isEqualTo(kp.getPublic().getEncoded());
    }

    @Test
    void signatureVerifiesAgainstStoredPublicKey() throws Exception {
        enricher = new AgentSignatureEnricher(actorId -> Optional.of(testKeyPair));

        final TestEntry e = entry("claude:reviewer@v1");
        enricher.enrich(e);

        final Signature sig = Signature.getInstance("Ed25519");
        sig.initVerify(testKeyPair.getPublic());
        sig.update(LedgerMerkleTree.canonicalBytes(e));
        assertThat(sig.verify(e.agentSignature)).isTrue();
    }

    @Test
    void leavesFieldsNull_whenActorHasNoKey() {
        enricher = new AgentSignatureEnricher(actorId -> Optional.empty());

        final TestEntry e = entry("unknown-actor");
        enricher.enrich(e);

        assertThat(e.agentSignature).isNull();
        assertThat(e.agentPublicKey).isNull();
    }

    @Test
    void leavesFieldsNull_whenActorIdIsNull() {
        enricher = new AgentSignatureEnricher(actorId -> Optional.of(testKeyPair));

        final TestEntry e = entry(null);
        enricher.enrich(e);

        assertThat(e.agentSignature).isNull();
        assertThat(e.agentPublicKey).isNull();
    }

    @Test
    void isIdempotent_calledTwice() {
        enricher = new AgentSignatureEnricher(actorId -> Optional.of(testKeyPair));

        final TestEntry e = entry("claude:reviewer@v1");
        enricher.enrich(e);
        final byte[] firstSig = e.agentSignature.clone();

        enricher.enrich(e);

        assertThat(e.agentSignature).isEqualTo(firstSig);
    }

    @Test
    void doesNotThrow_whenKeyProviderThrows() {
        enricher = new AgentSignatureEnricher(actorId -> {
            throw new RuntimeException("key store unavailable");
        });

        final TestEntry e = entry("claude:reviewer@v1");
        assertThatCode(() -> enricher.enrich(e)).doesNotThrowAnyException();
        assertThat(e.agentSignature).isNull();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=AgentSignatureEnricherTest -q 2>&1 | tail -5
```

Expected: compilation failure — `AgentSignatureEnricher` does not exist yet.

- [ ] **Step 3: Implement `AgentSignatureEnricher`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java`:

```java
package io.casehub.ledger.runtime.service;

import java.security.KeyPair;
import java.security.Signature;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * {@link LedgerEntryEnricher} that signs each entry with the actorId's Ed25519 private key.
 *
 * <p>
 * Signs {@link LedgerMerkleTree#canonicalBytes(LedgerEntry)} — the same canonical form
 * used for Merkle leaf hashes, guaranteeing that the signature covers the exact fields
 * that appear in the tamper-evident chain.
 *
 * <p>
 * No-op when the actor has no configured key pair. Non-fatal — exceptions are swallowed
 * so a key store failure never blocks a ledger write.
 */
@ApplicationScoped
public class AgentSignatureEnricher implements LedgerEntryEnricher {

    private static final Logger LOG = Logger.getLogger(AgentSignatureEnricher.class);

    private final AgentKeyProvider keyProvider;

    @Inject
    public AgentSignatureEnricher(final AgentKeyProvider keyProvider) {
        this.keyProvider = keyProvider;
    }

    @Override
    public void enrich(final LedgerEntry entry) {
        if (entry.actorId == null) return;
        try {
            keyProvider.signingKeyPair(entry.actorId).ifPresent(kp -> sign(entry, kp));
        } catch (final Exception e) {
            LOG.warnf("AgentSignatureEnricher failed for actor %s — entry will be unsigned: %s",
                    entry.actorId, e.getMessage());
        }
    }

    private static void sign(final LedgerEntry entry, final KeyPair kp) {
        try {
            final byte[] canonical = LedgerMerkleTree.canonicalBytes(entry);
            final Signature sig = Signature.getInstance("Ed25519");
            sig.initSign(kp.getPrivate());
            sig.update(canonical);
            entry.agentSignature = sig.sign();
            entry.agentPublicKey = kp.getPublic().getEncoded();
        } catch (final Exception e) {
            throw new IllegalStateException("Ed25519 signing failed for actor " + entry.actorId, e);
        }
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=AgentSignatureEnricherTest -q
```

Expected: BUILD SUCCESS, 6 tests pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java
git commit -m "feat(#79): AgentSignatureEnricher — signs canonical bytes at @PrePersist via AgentKeyProvider"
```

---

## Task 5: `LedgerVerificationService.verifyAgentSignature` — tests first, then implementation

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`

- [ ] **Step 1: Add failing tests to `LedgerVerificationServiceIT`**

Add these tests at the end of `LedgerVerificationServiceIT.java` (before the closing brace). Add required imports: `java.security.KeyPair`, `java.security.KeyPairGenerator`, `io.casehub.ledger.runtime.service.model.VerificationResult`.

```java
// ── Agent signature verification ──────────────────────────────────────────

@Test
@Transactional
void verifyAgentSignature_unsignedEntry_returnsUnsigned() {
    final UUID sub = UUID.randomUUID();
    final TestEntry e = seedEntry(sub, 1, "unsigned-actor");
    // agentSignature and agentPublicKey remain null

    assertThat(verificationService.verifyAgentSignature(e.id))
            .isEqualTo(VerificationResult.UNSIGNED);
}

@Test
@Transactional
void verifyAgentSignature_validSignature_returnsValid() throws Exception {
    final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
    final UUID sub = UUID.randomUUID();
    final TestEntry e = seedEntry(sub, 1, "signed-actor");

    // Manually attach signature to simulate what AgentSignatureEnricher does
    final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
    final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
    sig.initSign(kp.getPrivate());
    sig.update(canonical);
    e.agentSignature = sig.sign();
    e.agentPublicKey = kp.getPublic().getEncoded();
    repo.save(e);

    assertThat(verificationService.verifyAgentSignature(e.id))
            .isEqualTo(VerificationResult.VALID);
}

@Test
@Transactional
void verifyAgentSignature_tamperedSignature_returnsInvalid() throws Exception {
    final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
    final UUID sub = UUID.randomUUID();
    final TestEntry e = seedEntry(sub, 1, "signed-actor");

    final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
    final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
    sig.initSign(kp.getPrivate());
    sig.update(canonical);
    final byte[] signature = sig.sign();
    signature[0] ^= 0xFF; // flip bits to corrupt
    e.agentSignature = signature;
    e.agentPublicKey = kp.getPublic().getEncoded();
    repo.save(e);

    assertThat(verificationService.verifyAgentSignature(e.id))
            .isEqualTo(VerificationResult.INVALID);
}

@Test
@Transactional
void verifyAgentSignature_mutatedActorId_returnsInvalid() throws Exception {
    final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
    final UUID sub = UUID.randomUUID();
    final TestEntry e = seedEntry(sub, 1, "original-actor");

    final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
    final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
    sig.initSign(kp.getPrivate());
    sig.update(canonical);
    e.agentSignature = sig.sign();
    e.agentPublicKey = kp.getPublic().getEncoded();
    e.actorId = "impersonator-actor"; // mutate after signing
    repo.save(e);

    assertThat(verificationService.verifyAgentSignature(e.id))
            .isEqualTo(VerificationResult.INVALID);
}
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerVerificationServiceIT -q 2>&1 | tail -10
```

Expected: compilation failure — `verifyAgentSignature` does not exist yet.

- [ ] **Step 3: Implement `verifyAgentSignature`**

In `LedgerVerificationService.java`, add imports: `java.security.KeyFactory`, `java.security.PublicKey`, `java.security.spec.X509EncodedKeySpec`, `io.casehub.ledger.runtime.service.model.VerificationResult`.

Add this method after `verify()`:

```java
/**
 * Verifies the agent signature on the given entry.
 *
 * @param entryId the entry to verify
 * @return {@link VerificationResult#UNSIGNED} if no signature stored;
 *         {@link VerificationResult#VALID} if the signature verifies;
 *         {@link VerificationResult#INVALID} if verification fails
 * @throws IllegalArgumentException if the entry does not exist
 */
@Transactional
public VerificationResult verifyAgentSignature(final UUID entryId) {
    final LedgerEntry entry = ledgerRepo.findEntryById(entryId)
            .orElseThrow(() -> new IllegalArgumentException("Entry not found: " + entryId));

    if (entry.agentSignature == null) {
        return VerificationResult.UNSIGNED;
    }

    try {
        final KeyFactory kf = KeyFactory.getInstance("Ed25519");
        final PublicKey pub = kf.generatePublic(new X509EncodedKeySpec(entry.agentPublicKey));

        final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
        sig.initVerify(pub);
        sig.update(LedgerMerkleTree.canonicalBytes(entry));

        return sig.verify(entry.agentSignature) ? VerificationResult.VALID : VerificationResult.INVALID;
    } catch (final Exception e) {
        return VerificationResult.INVALID;
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerVerificationServiceIT -q
```

Expected: BUILD SUCCESS, all tests (existing + 4 new) pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java
git commit -m "feat(#79): LedgerVerificationService.verifyAgentSignature — UNSIGNED/VALID/INVALID"
```

---

## Task 6: `AgentSigningIT` — end-to-end integration test

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/AgentSigningIT.java`

Tests the full enricher pipeline end-to-end: entry persisted → enricher fires → signature stored → verification service confirms VALID. Uses `@InjectMock` to provide a test key pair without PEM files.

- [ ] **Step 1: Write the integration test**

Create `runtime/src/test/java/io/casehub/ledger/service/AgentSigningIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.when;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.AgentKeyProvider;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.model.VerificationResult;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class AgentSigningIT {

    @Inject
    LedgerEntryRepository repo;

    @Inject
    LedgerVerificationService verificationService;

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @InjectMock
    AgentKeyProvider agentKeyProvider;

    private KeyPair testKeyPair;

    @BeforeEach
    void setUp() throws Exception {
        testKeyPair = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        when(agentKeyProvider.signingKeyPair("claude:reviewer@v1"))
                .thenReturn(Optional.of(testKeyPair));
        when(agentKeyProvider.signingKeyPair(anyString()))
                .thenReturn(Optional.empty());
    }

    private TestEntry seedSigned(final UUID subjectId, final int seq) {
        final TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = seq;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "claude:reviewer@v1";
        e.actorType = ActorType.AGENT;
        e.actorRole = "Reviewer";
        return (TestEntry) repo.save(e);
    }

    @Test
    @Transactional
    void signedEntry_enrichedByPipeline_verifiesValid() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedSigned(sub, 1);

        assertThat(e.agentSignature).as("enricher should have populated agentSignature").isNotNull();
        assertThat(e.agentPublicKey).as("enricher should have populated agentPublicKey").isNotNull();
        assertThat(verificationService.verifyAgentSignature(e.id)).isEqualTo(VerificationResult.VALID);
    }

    @Test
    @Transactional
    void unsignedEntry_noKeyConfigured_returnsUnsigned() {
        final TestEntry e = new TestEntry();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "system:noop";
        e.actorType = ActorType.SYSTEM;
        e.actorRole = "System";
        final TestEntry saved = (TestEntry) repo.save(e);

        assertThat(saved.agentSignature).isNull();
        assertThat(verificationService.verifyAgentSignature(saved.id))
                .isEqualTo(VerificationResult.UNSIGNED);
    }

    @Test
    @Transactional
    void tamperedSignatureBytes_returnsInvalid() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedSigned(sub, 1);

        // Corrupt the stored signature directly
        final LedgerEntry stored = repo.findEntryById(e.id).orElseThrow();
        stored.agentSignature[0] ^= 0xFF;
        em.flush();

        assertThat(verificationService.verifyAgentSignature(e.id)).isEqualTo(VerificationResult.INVALID);
    }

    @Test
    @Transactional
    void signedEntry_merkleChainStillValid() {
        final UUID sub = UUID.randomUUID();
        seedSigned(sub, 1);
        seedSigned(sub, 2);

        assertThat(verificationService.verify(sub)).isTrue();
    }
}
```

- [ ] **Step 2: Run the integration test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=AgentSigningIT -q
```

Expected: BUILD SUCCESS, 4 tests pass.

- [ ] **Step 3: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q
```

Expected: BUILD SUCCESS, all tests pass (390+ → now more).

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/AgentSigningIT.java
git commit -m "test(#79): AgentSigningIT — end-to-end bilateral signing via enricher pipeline"
```

---

## Task 7: ADR 0011

**Files:**
- Create: `adr/0011-per-actorid-signing-key-model.md` (workspace staging)

- [ ] **Step 1: Write the ADR**

Create `/Users/mdproctor/claude/public/casehub/ledger/adr/0011-per-actorid-signing-key-model.md`:

```markdown
# ADR 0011 — Per-actorId Signing Key Model for Agent Non-Repudiation

**Status:** Accepted
**Date:** 2026-05-15
**Issue:** casehubio/ledger#79

## Context

Bilateral entry signing requires each `LedgerEntry` to be signed by the agent that authored it. The key architectural question is the granularity of signing keys: one key per deployment, or one key pair per actorId.

## Decision

**Per-actorId key pairs.** Each distinct actorId is associated with its own Ed25519 key pair. The `AgentKeyProvider` SPI supplies the appropriate `KeyPair` at signing time. The default implementation (`ConfiguredAgentKeyProvider`) reads PEM file paths keyed by actorId from `casehub.ledger.agent-signing.keys.*`.

## Rationale

**Per-deployment keys are rejected** because they undermine the entire premise of non-repudiation. If all agents share one signing key, any agent (or any actor with access to the deployment) could have produced any signature. The ledger proves the entry was signed, but not *which* agent signed it. This is tamper-evidence, not non-repudiation.

**Per-actorId keys** mean a SOUND attestation from `claude:reviewer@v1` is cryptographically distinguishable from one produced by any other actor. When a key is rotated (see #80), old entries remain verifiable via their stored public key bytes — the self-contained verification model ensures historical entries are never compromised by key rotation.

Industry consensus (Keyfactor, 2025; Aembit, 2025) is that per-agent cryptographic identities are the correct model. Shared deployment credentials create attribution blind spots and dramatically increase blast radius on key compromise.

## Consequences

- Operators configure one key pair per signing agent (private + public PEM files)
- Actors without a configured key pair produce unsigned entries — signing is opt-in
- Key management (generation, rotation, revocation) is out of scope for this issue
- The `AgentKeyProvider` SPI allows consumers to provide per-actorId keys from any source (HSM, Vault, PKI) by replacing the default implementation

## Supersedes

Nothing. First decision in this space.

## Related

- #80 — key rotation pattern
- #81 — agent DID/VC identity binding (the follow-on that adds cryptographic identity verification)
- ADR 0004 — actorId format for LLM agents (`model:persona@major`)
```

- [ ] **Step 2: Commit ADR to workspace**

```bash
git -C /Users/mdproctor/claude/public/casehub/ledger add adr/0011-per-actorid-signing-key-model.md
git -C /Users/mdproctor/claude/public/casehub/ledger commit -m "adr(#79): ADR 0011 — per-actorId signing key model"
```

---

## Task 8: Code review, doc sync, full install

- [ ] **Step 1: Invoke `superpowers:requesting-code-review`**

Review all changes introduced in this issue before the final commit.

- [ ] **Step 2: Fix any findings Minor or above**

Any finding that cannot be fixed this session must be captured as a GitHub issue in `casehubio/ledger` before sign-off. Batch related nits into a single issue.

- [ ] **Step 3: Run full test suite one final time**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 4: Install to .m2**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -q
```

- [ ] **Step 5: Invoke `implementation-doc-sync`**

Sync DESIGN.md to reflect the new `AgentKeyProvider`, `AgentSignatureEnricher`, `VerificationResult`, and `verifyAgentSignature` additions.

- [ ] **Step 6: Promote ADR to project repo**

```bash
cp /Users/mdproctor/claude/public/casehub/ledger/adr/0011-per-actorid-signing-key-model.md \
   /Users/mdproctor/claude/casehub/ledger/docs/adr/0011-per-actorid-signing-key-model.md
git add docs/adr/0011-per-actorid-signing-key-model.md
git commit -m "docs(#79): ADR 0011 — per-actorId signing key model Closes #79"
```
