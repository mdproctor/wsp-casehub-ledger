# Agent Signing Key Rotation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add key versioning (`keyRef`), rotation event records (`KeyRotationEntry`), and compromise detection (`SUSPECT` verification result) to the bilateral entry signing infrastructure.

**Architecture:** `keyRef` is self-derived as `Base64URL(SHA-256(publicKey.getEncoded()))` — zero operator configuration. A `KeyRotationEntry` (first-class `LedgerEntry` subclass) records each rotation with reason and `effectiveSince`. `LedgerVerificationService.verifyAgentSignature()` returns `SUSPECT` when a cryptographically valid entry was signed under a key subsequently reported `COMPROMISED` within the applicable time window.

**Tech Stack:** Java 21, Quarkus 3.32.2, Ed25519 (`java.security`), Hibernate ORM (JOINED inheritance), Flyway, JUnit 5, AssertJ, Mockito (`@InjectMock`)

---

## Task 1: `SigningKey` record + `AgentKeyProvider` SPI rename + `ConfiguredAgentKeyProvider` update

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/SigningKey.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyProvider.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/ConfiguredAgentKeyProvider.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AgentSigningIT.java`

- [ ] **Step 1: Create `SigningKey` record**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/SigningKey.java`:

```java
package io.casehub.ledger.runtime.service;

import java.security.KeyPair;
import java.security.MessageDigest;
import java.util.Base64;

/**
 * A signing key pair with a self-derived stable identifier.
 *
 * <p>
 * The {@code keyRef} is {@code Base64URL(SHA-256(publicKey.getEncoded()))} —
 * derived entirely from the public key bytes. Zero operator configuration.
 * Any party with the public key can independently compute and verify the keyRef.
 * Old entries can retroactively derive their keyRef from stored {@code agentPublicKey} bytes.
 */
public record SigningKey(String keyRef, KeyPair keyPair) {

    public static SigningKey of(final KeyPair keyPair) {
        try {
            final byte[] encoded = keyPair.getPublic().getEncoded();
            final byte[] hash = MessageDigest.getInstance("SHA-256").digest(encoded);
            final String keyRef = Base64.getUrlEncoder().withoutPadding().encodeToString(hash);
            return new SigningKey(keyRef, keyPair);
        } catch (final Exception e) {
            throw new IllegalStateException("Failed to derive keyRef from public key", e);
        }
    }
}
```

- [ ] **Step 2: Update `AgentKeyProvider` SPI**

Replace the content of `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyProvider.java`:

```java
package io.casehub.ledger.runtime.service;

import java.util.Optional;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * SPI: supplies the {@link SigningKey} used to sign a {@link LedgerEntry}
 * on behalf of a given actorId.
 *
 * <p>
 * Return {@link Optional#empty()} for actors that do not participate in
 * bilateral signing — those entries will be persisted unsigned.
 *
 * <p>
 * The {@link SigningKey} carries a self-derived {@code keyRef}
 * ({@code Base64URL(SHA-256(publicKey.getEncoded()))}) that is stored alongside
 * the signature on each entry, enabling key-generation attribution and
 * compromise detection.
 *
 * <p>
 * Implementations must be {@code @ApplicationScoped} CDI beans. The default
 * implementation ({@link ConfiguredAgentKeyProvider}) reads key paths from
 * {@code casehub.ledger.agent-signing.keys.*} config.
 */
public interface AgentKeyProvider {

    /**
     * Returns the signing key for the given actorId, or empty if this
     * actor does not sign ledger entries.
     *
     * @param actorId the actor identity string (e.g. {@code "claude:reviewer@v1"})
     * @return signing key, or empty for unsigned actors
     */
    Optional<SigningKey> signingKey(String actorId);
}
```

- [ ] **Step 3: Update `ConfiguredAgentKeyProvider`**

In `ConfiguredAgentKeyProvider.java`, change the `keyPairs` map type and `loadKeys` + `signingKeyPair` method. Replace the class body with:

```java
    private final Map<String, SigningKey> signingKeys = new ConcurrentHashMap<>();
    private final Set<String> failedActors = ConcurrentHashMap.newKeySet();

    @PostConstruct
    void loadKeys() {
        config.agentSigning().keys().forEach((actorId, keyConfig) -> {
            try {
                final PrivateKey priv = loadPrivateKey(keyConfig.privateKey());
                final PublicKey pub = loadPublicKey(keyConfig.publicKey());
                final SigningKey signingKey = SigningKey.of(new KeyPair(pub, priv));
                signingKeys.put(actorId, signingKey);
                LOG.infof("Loaded signing key for actor %s — keyRef: %s", actorId, signingKey.keyRef());
            } catch (final Exception e) {
                failedActors.add(actorId);
                LOG.errorf("Failed to load signing key for actor %s: %s — entries for this actor will be unsigned",
                        actorId, e.getMessage());
            }
        });
    }

    @Override
    public Optional<SigningKey> signingKey(final String actorId) {
        if (failedActors.contains(actorId)) {
            LOG.warnf("Actor %s was configured for signing but key failed to load — entry will be unsigned", actorId);
        }
        return Optional.ofNullable(signingKeys.get(actorId));
    }
```

Also add `import io.casehub.ledger.runtime.service.SigningKey;` and remove the unused `import java.security.KeyPair;` — `KeyPair` is used internally but `SigningKey` wraps it. Keep `KeyPair` import since `new KeyPair(pub, priv)` is still used in `loadKeys`.

- [ ] **Step 4: Update `AgentSignatureEnricherTest` for new API**

In `AgentSignatureEnricherTest.java`, update all lambdas that return `Optional.of(testKeyPair)` to return `Optional.of(SigningKey.of(testKeyPair))`. Add import `import io.casehub.ledger.runtime.service.SigningKey;`.

Affected lambdas — replace throughout:
```java
// Before
actorId -> Optional.of(testKeyPair)
actorId -> Optional.of(kp)
actorId -> Optional.of(otherPair)

// After
actorId -> Optional.of(SigningKey.of(testKeyPair))
actorId -> Optional.of(SigningKey.of(kp))
actorId -> Optional.of(SigningKey.of(otherPair))
```

- [ ] **Step 5: Update `AgentSigningIT` for new API**

In `AgentSigningIT.java`, change the mock setup from `Optional.of(testKeyPair)` to `Optional.of(SigningKey.of(testKeyPair))`. Add import `import io.casehub.ledger.runtime.service.SigningKey;`.

```java
// Before
when(agentKeyProvider.signingKeyPair("claude:reviewer@v1")).thenReturn(Optional.of(testKeyPair));
// After
when(agentKeyProvider.signingKey("claude:reviewer@v1")).thenReturn(Optional.of(SigningKey.of(testKeyPair)));
```

Also change the `when(agentKeyProvider.signingKeyPair(anyString()))` line:
```java
when(agentKeyProvider.signingKey(anyString())).thenReturn(Optional.empty());
```

- [ ] **Step 6: Verify compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: BUILD SUCCESS. (Tests will fail until Task 2 updates the enricher.)

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/SigningKey.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyProvider.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/ConfiguredAgentKeyProvider.java \
        runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java \
        runtime/src/test/java/io/casehub/ledger/service/AgentSigningIT.java
git commit -m "refactor(#80): SigningKey record + AgentKeyProvider.signingKey() returning Optional<SigningKey>"
```

---

## Task 2: V1006 migration + `LedgerEntry.agentKeyRef` + `AgentSignatureEnricher` update + tests

**Files:**
- Create: `runtime/src/main/resources/db/migration/V1006__agent_key_ref.sql`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java`
- Test: `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java`

- [ ] **Step 1: Write failing test for `agentKeyRef` population**

In `AgentSignatureEnricherTest.java`, add this test after `populatesSignatureAndPublicKey_whenActorHasKey`:

```java
@Test
void populatesAgentKeyRef_matchingSigningKeyRef() {
    final SigningKey sk = SigningKey.of(testKeyPair);
    enricher = new AgentSignatureEnricher(actorId -> Optional.of(sk));

    final TestEntry e = entry("claude:reviewer@v1");
    enricher.enrich(e);

    assertThat(e.agentKeyRef).isEqualTo(sk.keyRef());
}

@Test
void agentKeyRef_isNullWhenUnsigned() {
    enricher = new AgentSignatureEnricher(actorId -> Optional.empty());

    final TestEntry e = entry("unknown-actor");
    enricher.enrich(e);

    assertThat(e.agentKeyRef).isNull();
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentSignatureEnricherTest -q 2>&1 | tail -5
```

Expected: compile error — `agentKeyRef` field doesn't exist yet.

- [ ] **Step 3: Create V1006 migration**

Create `runtime/src/main/resources/db/migration/V1006__agent_key_ref.sql`:

```sql
ALTER TABLE ledger_entry ADD COLUMN agent_key_ref TEXT;

ALTER TABLE ledger_entry ADD CONSTRAINT chk_agent_key_ref_nullability
    CHECK ((agent_key_ref IS NULL) = (agent_signature IS NULL));
```

- [ ] **Step 4: Add `agentKeyRef` to `LedgerEntry`**

In `LedgerEntry.java`, add after the `agentPublicKey` field (inside the `// ── Agent signing` section):

```java
/**
 * Self-derived identifier for the key generation that produced {@link #agentSignature}.
 * Value: {@code Base64URL(SHA-256(agentPublicKey))} — computable from stored bytes.
 * Null when {@link #agentSignature} is null.
 */
@Column(name = "agent_key_ref")
public String agentKeyRef;
```

- [ ] **Step 5: Update `AgentSignatureEnricher` to use `SigningKey` and store `agentKeyRef`**

Replace the `enrich` and `sign` methods in `AgentSignatureEnricher.java`:

```java
    @Override
    public void enrich(final LedgerEntry entry) {
        if (entry.actorId == null || entry.agentSignature != null) return;
        try {
            keyProvider.signingKey(entry.actorId).ifPresent(sk -> sign(entry, sk));
        } catch (final Exception e) {
            LOG.warnf("AgentSignatureEnricher failed for actor %s — entry will be unsigned: %s",
                    entry.actorId, e.getMessage());
        }
    }

    private static void sign(final LedgerEntry entry, final SigningKey sk) {
        try {
            final byte[] canonical = LedgerMerkleTree.canonicalBytes(entry);
            final Signature sig = Signature.getInstance("Ed25519");
            sig.initSign(sk.keyPair().getPrivate());
            sig.update(canonical);
            entry.agentSignature = sig.sign();
            entry.agentPublicKey = sk.keyPair().getPublic().getEncoded();
            entry.agentKeyRef = sk.keyRef();
        } catch (final Exception e) {
            throw new IllegalStateException("Ed25519 signing failed for actor " + entry.actorId, e);
        }
    }
```

Also remove the unused `import java.security.KeyPair;` and add `import io.casehub.ledger.runtime.service.SigningKey;`.

- [ ] **Step 6: Run all enricher tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentSignatureEnricherTest -q
```

Expected: BUILD SUCCESS, all tests pass (original 6 + 2 new = 8 tests).

- [ ] **Step 7: Run full suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/resources/db/migration/V1006__agent_key_ref.sql \
        runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java \
        runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java
git commit -m "feat(#80): V1006 agent_key_ref column, LedgerEntry.agentKeyRef, AgentSignatureEnricher stores keyRef"
```

---

## Task 3: `SigningKeyTest` — unit tests for `SigningKey.of()`

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/SigningKeyTest.java`

- [ ] **Step 1: Write tests**

Create `runtime/src/test/java/io/casehub/ledger/service/SigningKeyTest.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.util.Base64;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.service.SigningKey;

class SigningKeyTest {

    @Test
    void keyRef_isDerivedFromPublicKey() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final SigningKey sk = SigningKey.of(kp);

        assertThat(sk.keyRef()).isNotNull().isNotEmpty();
        assertThat(sk.keyPair()).isSameAs(kp);
    }

    @Test
    void keyRef_isSameForSamePublicKey() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();

        final SigningKey sk1 = SigningKey.of(kp);
        final SigningKey sk2 = SigningKey.of(kp);

        assertThat(sk1.keyRef()).isEqualTo(sk2.keyRef());
    }

    @Test
    void keyRef_differsBetweenDistinctKeys() throws Exception {
        final KeyPairGenerator gen = KeyPairGenerator.getInstance("Ed25519");
        final KeyPair kp1 = gen.generateKeyPair();
        final KeyPair kp2 = gen.generateKeyPair();

        assertThat(SigningKey.of(kp1).keyRef())
                .isNotEqualTo(SigningKey.of(kp2).keyRef());
    }

    @Test
    void keyRef_isBase64UrlEncoded() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final String keyRef = SigningKey.of(kp).keyRef();

        // Base64URL without padding: only A-Z, a-z, 0-9, -, _
        assertThat(keyRef).matches("[A-Za-z0-9_-]+");
        // SHA-256 → 32 bytes → 43 Base64URL chars (no padding)
        assertThat(keyRef).hasSize(43);
    }

    @Test
    void keyRef_derivableFromPublicKeyBytes() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final SigningKey sk = SigningKey.of(kp);

        // Simulate retroactive computation from stored agentPublicKey bytes
        final byte[] pubKeyBytes = kp.getPublic().getEncoded();
        final java.security.MessageDigest sha256 = java.security.MessageDigest.getInstance("SHA-256");
        final byte[] hash = sha256.digest(pubKeyBytes);
        final String computed = Base64.getUrlEncoder().withoutPadding().encodeToString(hash);

        assertThat(sk.keyRef()).isEqualTo(computed);
    }
}
```

- [ ] **Step 2: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SigningKeyTest -q
```

Expected: BUILD SUCCESS, 5 tests pass.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/SigningKeyTest.java
git commit -m "test(#80): SigningKeyTest — keyRef derivation and Base64URL encoding"
```

---

## Task 4: `KeyRotationReason` enum + `KeyRotationEntry` subclass + V1007 migration

**Files:**
- Create: `api/src/main/java/io/casehub/ledger/api/model/KeyRotationReason.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/KeyRotationEntry.java`
- Create: `runtime/src/main/resources/db/migration/V1007__key_rotation_entry.sql`

- [ ] **Step 1: Create `KeyRotationReason` enum**

Create `api/src/main/java/io/casehub/ledger/api/model/KeyRotationReason.java`:

```java
package io.casehub.ledger.api.model;

/**
 * Why an agent signing key was rotated.
 *
 * <p>
 * Per NIST SP 800-57, key rotation (planned) and key revocation (compromise)
 * are distinct lifecycle events requiring distinct records and response procedures.
 */
public enum KeyRotationReason {

    /**
     * Planned rotation due to cryptoperiod policy — no compromise assumed.
     * Entries signed by the previous key remain VALID.
     */
    SCHEDULED,

    /**
     * Emergency rotation because the key was leaked or the actor was misbehaving.
     * Entries signed by the previous key on or after {@code effectiveSince} are SUSPECT.
     */
    COMPROMISED
}
```

- [ ] **Step 2: Create V1007 migration**

Create `runtime/src/main/resources/db/migration/V1007__key_rotation_entry.sql`:

```sql
CREATE TABLE key_rotation_entry (
    id               UUID PRIMARY KEY REFERENCES ledger_entry(id),
    previous_key_ref TEXT,
    new_key_ref      TEXT,
    reason           TEXT NOT NULL,
    effective_since  TIMESTAMP WITH TIME ZONE NOT NULL
);
```

- [ ] **Step 3: Create `KeyRotationEntry` subclass**

Create `runtime/src/main/java/io/casehub/ledger/runtime/model/KeyRotationEntry.java`:

```java
package io.casehub.ledger.runtime.model;

import java.time.Instant;

import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;

import io.casehub.ledger.api.model.KeyRotationReason;

/**
 * A first-class immutable ledger entry recording a signing key rotation or revocation.
 *
 * <p>
 * {@link #subjectId} is derived deterministically as
 * {@code UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))}, making the full
 * key lifecycle for an actor queryable as an ordered ledger sequence.
 *
 * <p>
 * {@link #previousKeyRef} is the keyRef of the key being retired.
 * {@link #newKeyRef} is the keyRef of the replacement key (null for pure revocation).
 * {@link #effectiveSince} marks the earliest {@code occurredAt} of entries that are
 * SUSPECT when {@code reason == COMPROMISED}.
 */
@Entity
@Table(name = "key_rotation_entry")
@DiscriminatorValue("KEY_ROTATION")
public class KeyRotationEntry extends LedgerEntry {

    /** keyRef of the key being retired. Null when the previous key is unknown. */
    @Column(name = "previous_key_ref")
    public String previousKeyRef;

    /**
     * keyRef of the replacement key.
     * Null for pure revocation without a replacement (actor suspended).
     */
    @Column(name = "new_key_ref")
    public String newKeyRef;

    /** Why the key was rotated. */
    @Enumerated(EnumType.STRING)
    @Column(name = "reason", nullable = false)
    public KeyRotationReason reason;

    /**
     * For {@link KeyRotationReason#COMPROMISED}: entries signed by {@link #previousKeyRef}
     * on or after this timestamp are SUSPECT.
     * For {@link KeyRotationReason#SCHEDULED}: informational only — no entries are flagged.
     */
    @Column(name = "effective_since", nullable = false)
    public Instant effectiveSince;
}
```

- [ ] **Step 4: Verify schema applies**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=PlainEntityTest -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/model/KeyRotationReason.java \
        runtime/src/main/java/io/casehub/ledger/runtime/model/KeyRotationEntry.java \
        runtime/src/main/resources/db/migration/V1007__key_rotation_entry.sql
git commit -m "feat(#80): KeyRotationReason enum, KeyRotationEntry subclass, V1007 migration"
```

---

## Task 5: `CompromisedWindow` + `KeyRotationRepository` SPI + `JpaKeyRotationRepository`

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/model/CompromisedWindow.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/repository/KeyRotationRepository.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaKeyRotationRepository.java`

Note: `KeyRotationEntry` needs `@NamedQuery` annotations for the repository queries. Add them in this task.

- [ ] **Step 1: Add `@NamedQuery` to `KeyRotationEntry`**

In `KeyRotationEntry.java`, add these annotations before `@Entity`:

```java
import jakarta.persistence.NamedQueries;
import jakarta.persistence.NamedQuery;

@NamedQueries({
    @NamedQuery(
        name = "KeyRotationEntry.findByActorId",
        query = "SELECT e FROM KeyRotationEntry e WHERE e.actorId = :actorId ORDER BY e.occurredAt ASC"),
    @NamedQuery(
        name = "KeyRotationEntry.findCompromisedByActorIdAndKeyRef",
        query = "SELECT e FROM KeyRotationEntry e " +
                "WHERE e.actorId = :actorId " +
                "AND e.previousKeyRef = :keyRef " +
                "AND e.reason = io.casehub.ledger.api.model.KeyRotationReason.COMPROMISED " +
                "ORDER BY e.effectiveSince ASC")
})
@Entity
```

- [ ] **Step 2: Create `CompromisedWindow` record**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/model/CompromisedWindow.java`:

```java
package io.casehub.ledger.runtime.service.model;

import java.time.Instant;

/**
 * A time window during which a specific key generation was reported compromised.
 * Entries signed by {@link #keyRef} on or after {@link #effectiveSince} are SUSPECT.
 */
public record CompromisedWindow(String keyRef, Instant effectiveSince) {}
```

- [ ] **Step 3: Create `KeyRotationRepository` SPI**

Create `runtime/src/main/java/io/casehub/ledger/runtime/repository/KeyRotationRepository.java`:

```java
package io.casehub.ledger.runtime.repository;

import java.util.List;

import io.casehub.ledger.runtime.model.KeyRotationEntry;

/**
 * SPI for persisting and querying {@link KeyRotationEntry} records.
 */
public interface KeyRotationRepository {

    /**
     * Persist a key rotation entry.
     *
     * @param entry the entry to persist; must not be null
     * @return the persisted entry (post-{@code @PrePersist})
     */
    KeyRotationEntry save(KeyRotationEntry entry);

    /**
     * All rotation events for an actor, ordered by {@code occurredAt} ascending.
     */
    List<KeyRotationEntry> findByActorId(String actorId);

    /**
     * All {@code COMPROMISED} rotation events for a specific actor and keyRef,
     * ordered by {@code effectiveSince} ascending.
     */
    List<KeyRotationEntry> findCompromisedByActorIdAndKeyRef(String actorId, String keyRef);
}
```

- [ ] **Step 4: Create `JpaKeyRotationRepository`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaKeyRotationRepository.java`:

```java
package io.casehub.ledger.runtime.repository.jpa;

import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.repository.KeyRotationRepository;

@ApplicationScoped
public class JpaKeyRotationRepository implements KeyRotationRepository {

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @Override
    @Transactional
    public KeyRotationEntry save(final KeyRotationEntry entry) {
        if (entry.id == null) {
            entry.id = UUID.randomUUID();
        }
        if (entry.subjectId == null && entry.actorId != null) {
            entry.subjectId = UUID.nameUUIDFromBytes(
                    entry.actorId.getBytes(StandardCharsets.UTF_8));
        }
        em.persist(entry);
        return entry;
    }

    @Override
    public List<KeyRotationEntry> findByActorId(final String actorId) {
        return em.createNamedQuery("KeyRotationEntry.findByActorId", KeyRotationEntry.class)
                .setParameter("actorId", actorId)
                .getResultList();
    }

    @Override
    public List<KeyRotationEntry> findCompromisedByActorIdAndKeyRef(
            final String actorId, final String keyRef) {
        return em.createNamedQuery(
                "KeyRotationEntry.findCompromisedByActorIdAndKeyRef", KeyRotationEntry.class)
                .setParameter("actorId", actorId)
                .setParameter("keyRef", keyRef)
                .getResultList();
    }
}
```

- [ ] **Step 5: Verify compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/KeyRotationEntry.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/model/CompromisedWindow.java \
        runtime/src/main/java/io/casehub/ledger/runtime/repository/KeyRotationRepository.java \
        runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaKeyRotationRepository.java
git commit -m "feat(#80): CompromisedWindow, KeyRotationRepository SPI, JpaKeyRotationRepository"
```

---

## Task 6: `KeyRotationService` CDI bean — TDD

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/KeyRotationServiceIT.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/KeyRotationService.java`

- [ ] **Step 1: Write failing integration tests**

Create `runtime/src/test/java/io/casehub/ledger/service/KeyRotationServiceIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.nio.charset.StandardCharsets;
import java.security.KeyPairGenerator;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.ledger.runtime.service.KeyRotationService;
import io.casehub.ledger.runtime.service.SigningKey;
import io.casehub.ledger.runtime.service.model.CompromisedWindow;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class KeyRotationServiceIT {

    @Inject
    KeyRotationService rotationService;

    private SigningKey newKey() throws Exception {
        return SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
    }

    @Test
    @Transactional
    void recordRotation_persistsEntry() throws Exception {
        final String actorId = "claude:reviewer@v1";
        final SigningKey oldKey = newKey();
        final SigningKey newKey = newKey();

        final KeyRotationEntry entry = rotationService.recordRotation(
                actorId, oldKey.keyRef(), newKey.keyRef(),
                KeyRotationReason.SCHEDULED, Instant.now());

        assertThat(entry.id).isNotNull();
        assertThat(entry.actorId).isEqualTo(actorId);
        assertThat(entry.previousKeyRef).isEqualTo(oldKey.keyRef());
        assertThat(entry.newKeyRef).isEqualTo(newKey.keyRef());
        assertThat(entry.reason).isEqualTo(KeyRotationReason.SCHEDULED);
        assertThat(entry.entryType).isEqualTo(LedgerEntryType.COMMAND);
    }

    @Test
    @Transactional
    void recordRotation_subjectIdIsDeterministicFromActorId() throws Exception {
        final String actorId = "claude:auditor@v1";
        final SigningKey oldKey = newKey();

        final KeyRotationEntry entry = rotationService.recordRotation(
                actorId, oldKey.keyRef(), null,
                KeyRotationReason.COMPROMISED, Instant.now());

        final UUID expectedSubjectId = UUID.nameUUIDFromBytes(
                actorId.getBytes(StandardCharsets.UTF_8));
        assertThat(entry.subjectId).isEqualTo(expectedSubjectId);
    }

    @Test
    @Transactional
    void rotationHistory_returnsAllEventsOrdered() throws Exception {
        final String actorId = "claude:reviewer@v2-" + UUID.randomUUID();
        final SigningKey k1 = newKey();
        final SigningKey k2 = newKey();

        final Instant t1 = Instant.now();
        final Instant t2 = t1.plusSeconds(60);

        rotationService.recordRotation(actorId, k1.keyRef(), k2.keyRef(),
                KeyRotationReason.SCHEDULED, t1);
        rotationService.recordRotation(actorId, k2.keyRef(), null,
                KeyRotationReason.COMPROMISED, t2);

        final List<KeyRotationEntry> history = rotationService.rotationHistory(actorId);
        assertThat(history).hasSize(2);
        assertThat(history.get(0).reason).isEqualTo(KeyRotationReason.SCHEDULED);
        assertThat(history.get(1).reason).isEqualTo(KeyRotationReason.COMPROMISED);
    }

    @Test
    @Transactional
    void compromisedWindows_onlyReturnsCompromisedReason() throws Exception {
        final String actorId = "claude:reviewer@v3-" + UUID.randomUUID();
        final SigningKey oldKey = newKey();
        final SigningKey newKey = newKey();
        final Instant compromisedSince = Instant.now().minusSeconds(3600);

        // scheduled rotation — should NOT appear in compromised windows
        rotationService.recordRotation(actorId, oldKey.keyRef(), newKey.keyRef(),
                KeyRotationReason.SCHEDULED, Instant.now());
        // compromised rotation
        rotationService.recordRotation(actorId, oldKey.keyRef(), null,
                KeyRotationReason.COMPROMISED, compromisedSince);

        final List<CompromisedWindow> windows =
                rotationService.compromisedWindows(actorId, oldKey.keyRef());

        assertThat(windows).hasSize(1);
        assertThat(windows.get(0).keyRef()).isEqualTo(oldKey.keyRef());
        assertThat(windows.get(0).effectiveSince()).isEqualTo(compromisedSince);
    }

    @Test
    @Transactional
    void compromisedWindows_emptyWhenNoCompromiseRecord() throws Exception {
        final String actorId = "claude:reviewer@v4-" + UUID.randomUUID();
        final SigningKey key = newKey();

        final List<CompromisedWindow> windows =
                rotationService.compromisedWindows(actorId, key.keyRef());

        assertThat(windows).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests — expect failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=KeyRotationServiceIT -q 2>&1 | tail -5
```

Expected: compile error — `KeyRotationService` does not exist.

- [ ] **Step 3: Implement `KeyRotationService`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/KeyRotationService.java`:

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.ledger.runtime.repository.KeyRotationRepository;
import io.casehub.ledger.runtime.service.model.CompromisedWindow;

/**
 * CDI bean for recording and querying signing key rotation events.
 *
 * <p>
 * Each rotation is persisted as a {@link KeyRotationEntry} — a first-class
 * immutable ledger entry in the tamper-evident chain, queryable per actor.
 */
@ApplicationScoped
public class KeyRotationService {

    @Inject
    KeyRotationRepository repository;

    /**
     * Record a signing key rotation event.
     *
     * @param actorId        the actor whose key is being rotated
     * @param previousKeyRef the keyRef of the key being retired; null if unknown
     * @param newKeyRef      the keyRef of the replacement key; null for pure revocation
     * @param reason         {@link KeyRotationReason#SCHEDULED} or {@link KeyRotationReason#COMPROMISED}
     * @param effectiveSince for COMPROMISED: entries signed on or after this time are SUSPECT;
     *                       for SCHEDULED: informational only
     * @return the persisted {@link KeyRotationEntry}
     */
    @Transactional
    public KeyRotationEntry recordRotation(
            final String actorId,
            final String previousKeyRef,
            final String newKeyRef,
            final KeyRotationReason reason,
            final Instant effectiveSince) {

        final KeyRotationEntry entry = new KeyRotationEntry();
        entry.actorId = actorId;
        entry.actorRole = "KeyManager";
        entry.entryType = LedgerEntryType.COMMAND;
        entry.previousKeyRef = previousKeyRef;
        entry.newKeyRef = newKeyRef;
        entry.reason = reason;
        entry.effectiveSince = effectiveSince;
        return repository.save(entry);
    }

    /**
     * All rotation events for an actor, ordered by {@code occurredAt} ascending.
     */
    @Transactional
    public List<KeyRotationEntry> rotationHistory(final String actorId) {
        return repository.findByActorId(actorId);
    }

    /**
     * Returns all time windows during which a specific key was reported compromised
     * for a given actor. Used by {@link LedgerVerificationService} for SUSPECT detection.
     */
    @Transactional
    public List<CompromisedWindow> compromisedWindows(
            final String actorId, final String keyRef) {
        return repository.findCompromisedByActorIdAndKeyRef(actorId, keyRef)
                .stream()
                .map(e -> new CompromisedWindow(e.previousKeyRef, e.effectiveSince))
                .toList();
    }
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=KeyRotationServiceIT -q
```

Expected: BUILD SUCCESS, 5 tests pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/KeyRotationServiceIT.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/KeyRotationService.java
git commit -m "feat(#80): KeyRotationService — recordRotation, rotationHistory, compromisedWindows"
```

---

## Task 7: `VerificationResult.SUSPECT` + `LedgerVerificationService` update — TDD

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/model/VerificationResult.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java`

- [ ] **Step 1: Write failing SUSPECT tests in `LedgerVerificationServiceIT`**

Add these imports to `LedgerVerificationServiceIT.java`:
```java
import java.security.KeyPairGenerator;
import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.runtime.service.KeyRotationService;
import io.casehub.ledger.runtime.service.SigningKey;
```

Add `@Inject KeyRotationService rotationService;` field.

Add these tests at the end of the agent signature section:

```java
@Test
@Transactional
void verifyAgentSignature_compromisedKey_afterEffectiveSince_returnsSuspect() throws Exception {
    final KeyPairGenerator gen = KeyPairGenerator.getInstance("Ed25519");
    final SigningKey sk = SigningKey.of(gen.generateKeyPair());
    final UUID sub = UUID.randomUUID();

    // Persist a signed entry
    final TestEntry e = seedEntry(sub, 1, "claude:reviewer@v1");
    final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
    final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
    sig.initSign(sk.keyPair().getPrivate());
    sig.update(canonical);
    e.agentSignature = sig.sign();
    e.agentPublicKey = sk.keyPair().getPublic().getEncoded();
    e.agentKeyRef = sk.keyRef();
    repo.save(e);

    // Record a COMPROMISED rotation with effectiveSince BEFORE the entry's occurredAt
    final Instant compromisedSince = e.occurredAt.minusSeconds(60);
    rotationService.recordRotation("claude:reviewer@v1", sk.keyRef(), null,
            KeyRotationReason.COMPROMISED, compromisedSince);

    assertThat(verificationService.verifyAgentSignature(e.id))
            .isEqualTo(VerificationResult.SUSPECT);
}

@Test
@Transactional
void verifyAgentSignature_compromisedKey_beforeEffectiveSince_returnsValid() throws Exception {
    final KeyPairGenerator gen = KeyPairGenerator.getInstance("Ed25519");
    final SigningKey sk = SigningKey.of(gen.generateKeyPair());
    final UUID sub = UUID.randomUUID();

    final TestEntry e = seedEntry(sub, 1, "claude:reviewer@v1");
    final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
    final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
    sig.initSign(sk.keyPair().getPrivate());
    sig.update(canonical);
    e.agentSignature = sig.sign();
    e.agentPublicKey = sk.keyPair().getPublic().getEncoded();
    e.agentKeyRef = sk.keyRef();
    repo.save(e);

    // Record a COMPROMISED rotation with effectiveSince AFTER the entry's occurredAt
    final Instant compromisedSince = e.occurredAt.plusSeconds(3600);
    rotationService.recordRotation("claude:reviewer@v1", sk.keyRef(), null,
            KeyRotationReason.COMPROMISED, compromisedSince);

    // Entry was signed before the compromise window — still VALID
    assertThat(verificationService.verifyAgentSignature(e.id))
            .isEqualTo(VerificationResult.VALID);
}

@Test
@Transactional
void verifyAgentSignature_scheduledRotation_doesNotProduceSuspect() throws Exception {
    final KeyPairGenerator gen = KeyPairGenerator.getInstance("Ed25519");
    final SigningKey sk = SigningKey.of(gen.generateKeyPair());
    final UUID sub = UUID.randomUUID();

    final TestEntry e = seedEntry(sub, 1, "claude:reviewer@v1");
    final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
    final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
    sig.initSign(sk.keyPair().getPrivate());
    sig.update(canonical);
    e.agentSignature = sig.sign();
    e.agentPublicKey = sk.keyPair().getPublic().getEncoded();
    e.agentKeyRef = sk.keyRef();
    repo.save(e);

    // SCHEDULED rotation — never produces SUSPECT
    rotationService.recordRotation("claude:reviewer@v1", sk.keyRef(),
            SigningKey.of(gen.generateKeyPair()).keyRef(),
            KeyRotationReason.SCHEDULED, e.occurredAt.minusSeconds(60));

    assertThat(verificationService.verifyAgentSignature(e.id))
            .isEqualTo(VerificationResult.VALID);
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerVerificationServiceIT -q 2>&1 | tail -5
```

Expected: compile error — `SUSPECT` not yet on `VerificationResult`.

- [ ] **Step 3: Add `SUSPECT` to `VerificationResult`**

Replace `VerificationResult.java` with:

```java
package io.casehub.ledger.runtime.service.model;

/**
 * Result of an agent signature verification on a {@link io.casehub.ledger.runtime.model.LedgerEntry}.
 */
public enum VerificationResult {

    /** No agent signature is stored on this entry — the actor did not sign. */
    UNSIGNED,

    /** Signature is present and cryptographically verified; key not compromised. */
    VALID,

    /** Signature is present but cryptographic verification failed — possible tampering. */
    INVALID,

    /**
     * Signature is cryptographically VALID but was produced by a key subsequently
     * reported {@link io.casehub.ledger.api.model.KeyRotationReason#COMPROMISED}
     * and the entry's {@code occurredAt} falls within the compromised window.
     * The entry content is intact; the signing actor's trustworthiness is in question.
     */
    SUSPECT
}
```

- [ ] **Step 4: Update `LedgerVerificationService.verifyAgentSignature`**

Inject `KeyRotationService` into `LedgerVerificationService`. Add field:
```java
    @Inject
    KeyRotationService keyRotationService;
```

Replace the body of `verifyAgentSignature` (lines 113–138) with:

```java
    @Transactional
    public VerificationResult verifyAgentSignature(final UUID entryId) {
        final LedgerEntry entry = ledgerRepo.findEntryById(entryId)
                .orElseThrow(() -> new IllegalArgumentException("Entry not found: " + entryId));

        if (entry.agentSignature == null) {
            return VerificationResult.UNSIGNED;
        }

        if (entry.agentPublicKey == null) {
            LOG.warnf("Entry %s has agentSignature but no agentPublicKey — record is corrupt", entryId);
            return VerificationResult.INVALID;
        }

        try {
            final KeyFactory kf = KeyFactory.getInstance("Ed25519");
            final PublicKey pub = kf.generatePublic(new X509EncodedKeySpec(entry.agentPublicKey));

            final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
            sig.initVerify(pub);
            sig.update(LedgerMerkleTree.canonicalBytes(entry));

            if (!sig.verify(entry.agentSignature)) {
                return VerificationResult.INVALID;
            }
        } catch (final Exception e) {
            return VerificationResult.INVALID;
        }

        // Cryptographic check passed — check for compromise windows
        if (entry.agentKeyRef != null && entry.actorId != null) {
            final boolean suspect = keyRotationService
                    .compromisedWindows(entry.actorId, entry.agentKeyRef)
                    .stream()
                    .anyMatch(w -> !entry.occurredAt.isBefore(w.effectiveSince()));
            if (suspect) {
                return VerificationResult.SUSPECT;
            }
        }

        return VerificationResult.VALID;
    }
```

Also add import: `import io.casehub.ledger.runtime.service.KeyRotationService;`

- [ ] **Step 5: Run — expect all IT tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerVerificationServiceIT -q
```

Expected: BUILD SUCCESS (existing tests + 3 new SUSPECT tests).

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/model/VerificationResult.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java \
        runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java
git commit -m "feat(#80): VerificationResult.SUSPECT + compromise window check in verifyAgentSignature"
```

---

## Task 8: `KeyRotationIT` — end-to-end integration test

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/KeyRotationIT.java`

- [ ] **Step 1: Write the IT**

Create `runtime/src/test/java/io/casehub/ledger/service/KeyRotationIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.when;

import java.security.KeyPairGenerator;
import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.AgentKeyProvider;
import io.casehub.ledger.runtime.service.KeyRotationService;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.SigningKey;
import io.casehub.ledger.runtime.service.model.VerificationResult;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class KeyRotationIT {

    @Inject LedgerEntryRepository repo;
    @Inject LedgerVerificationService verificationService;
    @Inject KeyRotationService rotationService;

    @InjectMock
    AgentKeyProvider agentKeyProvider;

    private SigningKey currentKey;
    private SigningKey nextKey;

    @BeforeEach
    void setUp() throws Exception {
        final KeyPairGenerator gen = KeyPairGenerator.getInstance("Ed25519");
        currentKey = SigningKey.of(gen.generateKeyPair());
        nextKey = SigningKey.of(gen.generateKeyPair());

        when(agentKeyProvider.signingKey(anyString())).thenReturn(Optional.empty());
        when(agentKeyProvider.signingKey("claude:reviewer@v1"))
                .thenReturn(Optional.of(currentKey));
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
    void scheduledRotation_existingEntries_remainValid() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedSigned(sub, 1);

        assertThat(e.agentKeyRef).isEqualTo(currentKey.keyRef());
        assertThat(verificationService.verifyAgentSignature(e.id)).isEqualTo(VerificationResult.VALID);

        rotationService.recordRotation("claude:reviewer@v1",
                currentKey.keyRef(), nextKey.keyRef(),
                KeyRotationReason.SCHEDULED, Instant.now());

        // Entry signed before scheduled rotation is still VALID
        assertThat(verificationService.verifyAgentSignature(e.id)).isEqualTo(VerificationResult.VALID);
    }

    @Test
    @Transactional
    void compromisedRotation_entryAfterEffectiveSince_isSuspect() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedSigned(sub, 1);

        assertThat(verificationService.verifyAgentSignature(e.id)).isEqualTo(VerificationResult.VALID);

        // Rotate as COMPROMISED — effectiveSince is before the entry's occurredAt
        rotationService.recordRotation("claude:reviewer@v1",
                currentKey.keyRef(), null,
                KeyRotationReason.COMPROMISED,
                e.occurredAt.minusSeconds(60));

        assertThat(verificationService.verifyAgentSignature(e.id)).isEqualTo(VerificationResult.SUSPECT);
    }

    @Test
    @Transactional
    void rotationHistory_recordsAllEvents() {
        rotationService.recordRotation("claude:reviewer@v1",
                currentKey.keyRef(), nextKey.keyRef(),
                KeyRotationReason.SCHEDULED, Instant.now());
        rotationService.recordRotation("claude:reviewer@v1",
                nextKey.keyRef(), null,
                KeyRotationReason.COMPROMISED, Instant.now());

        final List<KeyRotationEntry> history =
                rotationService.rotationHistory("claude:reviewer@v1");

        assertThat(history).hasSizeGreaterThanOrEqualTo(2);
        assertThat(history.stream().map(e -> e.reason))
                .contains(KeyRotationReason.SCHEDULED, KeyRotationReason.COMPROMISED);
    }

    @Test
    @Transactional
    void keyRotationEntry_subjectId_isDeterministic() {
        final KeyRotationEntry entry = rotationService.recordRotation(
                "claude:reviewer@v1", currentKey.keyRef(), nextKey.keyRef(),
                KeyRotationReason.SCHEDULED, Instant.now());

        // Same actorId always produces same subjectId
        final UUID expected = UUID.nameUUIDFromBytes(
                "claude:reviewer@v1".getBytes(java.nio.charset.StandardCharsets.UTF_8));
        assertThat(entry.subjectId).isEqualTo(expected);
    }
}
```

- [ ] **Step 2: Run the IT**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=KeyRotationIT -q
```

Expected: BUILD SUCCESS, 4 tests pass.

- [ ] **Step 3: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/KeyRotationIT.java
git commit -m "test(#80): KeyRotationIT — end-to-end rotation and compromise detection"
```

---

## Task 9: ADR 0012

**Files:**
- Create: `adr/0012-key-rotation-design.md` (workspace staging)
- Promote to: `docs/adr/0012-key-rotation-design.md` (project repo)

- [ ] **Step 1: Write ADR**

Create `/Users/mdproctor/claude/public/casehub/ledger/adr/0012-key-rotation-design.md`:

```markdown
# ADR 0012 — Agent Signing Key Rotation: Self-Derived keyRef and First-Class Rotation Events

**Status:** Accepted
**Date:** 2026-05-17
**Issue:** casehubio/ledger#80

## Context

Bilateral entry signing (#79) stores raw Ed25519 public key bytes alongside each signed `LedgerEntry`. This enables self-contained verification. What was missing: key identity attribution (which generation signed this entry?), rotation records, and compromise detection.

## Decisions

### 1. Self-derived `keyRef`

`keyRef = Base64URL(SHA-256(publicKey.getEncoded()))` — derived entirely from the public key bytes.

**Rejected:** operator-configured key IDs (misconfiguration risk, coordination required across deployments), sequence counters (requires stateful ledger bookkeeping).

**Rationale:** Aligns with Sigstore/Rekor v2's key identity model. Zero operator configuration. Any party with the public key can independently compute and verify the keyRef. Old entries can retroactively derive their keyRef from stored `agentPublicKey` bytes, enabling backfill without re-signing.

### 2. `KeyRotationEntry` as first-class `LedgerEntry` subclass

Rotation events are immutable ledger entries in the tamper-evident chain, not a separate audit table.

**Rejected:** separate `ledger_key_rotation` table (no Merkle participation, no sequence), `KeyRotationSupplement` (blurs purpose of attached entry).

**Rationale:** Key rotation is an observable event in the actor's lifecycle. It belongs in the chain. The `subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))` derivation makes the full key lifecycle queryable as an ordered ledger sequence per actor — the same query pattern used for all other subjects.

### 3. NIST SP 800-57 distinction: SCHEDULED vs COMPROMISED

Two distinct `KeyRotationReason` values, not a single "rotate" event.

**Rationale:** NIST SP 800-57 explicitly separates rotation (planned, cryptoperiod-driven) from revocation (compromise response, uses Compromised Key Lists). Only COMPROMISED records with `effectiveSince` produce SUSPECT entries in `verifyAgentSignature`. SCHEDULED rotations never retroactively affect existing entries.

### 4. `VerificationResult.SUSPECT` — not `INVALID`

A cryptographically valid signature from a subsequently compromised key returns SUSPECT, not INVALID.

**Rationale:** The entry content is intact and was genuinely signed by that actor at that time. INVALID would be misleading — it implies tampering. SUSPECT correctly conveys "the signature is authentic but the signing actor's trustworthiness is in question." Auditors can act on SUSPECT without destroying the evidentiary value of the entry.

## Consequences

- `AgentKeyProvider.signingKey()` returns `Optional<SigningKey>` (was `Optional<KeyPair>`)
- `LedgerEntry` gains `agentKeyRef TEXT` field (V1006 migration)
- `KeyRotationEntry` subclass table (V1007 migration)
- `LedgerVerificationService.verifyAgentSignature()` queries compromise windows on VALID signatures

## Related

- #79 — bilateral entry signing (foundation)
- #83 — CDI event on SUSPECT detection
- #84 — post-quantum algorithm migration
- #85 — external key distribution (TUF/HSM/PKI)
- ADR 0011 — per-actorId signing key model
```

- [ ] **Step 2: Commit to workspace and promote to project**

```bash
git -C /Users/mdproctor/claude/public/casehub/ledger add adr/0012-key-rotation-design.md
git -C /Users/mdproctor/claude/public/casehub/ledger commit -m "adr(#80): ADR 0012 — key rotation design"

cp /Users/mdproctor/claude/public/casehub/ledger/adr/0012-key-rotation-design.md \
   /Users/mdproctor/claude/casehub/ledger/docs/adr/0012-key-rotation-design.md
git add docs/adr/0012-key-rotation-design.md
git commit -m "docs(#80): ADR 0012 — agent signing key rotation"
```

---

## Task 10: Code review, doc sync, install, push

- [ ] **Step 1: Invoke `superpowers:requesting-code-review`**

Review all changes for #80 before the final commit.

- [ ] **Step 2: Fix any findings Minor or above**

Any finding not fixed this session → GitHub issue in `casehubio/ledger`. Batch related nits into one issue.

- [ ] **Step 3: Run full test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Install to .m2**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -q
```

- [ ] **Step 5: Invoke `implementation-doc-sync`**

Sync CLAUDE.md and DESIGN.md for new components: `SigningKey`, `KeyRotationReason`, `KeyRotationEntry`, `KeyRotationRepository`, `JpaKeyRotationRepository`, `KeyRotationService`, `CompromisedWindow`, `VerificationResult.SUSPECT`, V1006/V1007 migrations.

- [ ] **Step 6: Promote ADR** (done in Task 9 Step 2)

- [ ] **Step 7: Close #80 and push**

```bash
gh issue close 80 --repo casehubio/ledger --comment "Implemented and merged to main."
git push origin main
```
