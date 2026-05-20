# AgentSignatureVerificationService Extraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract agent signature verification out of `LedgerVerificationService` into a dedicated `AgentSignatureVerificationService` (blocking) and rename `ReactiveLedgerVerificationService` → `ReactiveAgentSignatureVerificationService`, eliminating a verbatim code duplication and giving each bean one clear concern.

**Architecture:** A new package-private static utility `AgentCryptographicVerifier` holds the Ed25519 verification logic used by both tiers, mirroring the `LedgerMerkleTree` pattern. `LedgerVerificationService` retains only Merkle operations; `AgentSignatureVerificationService` holds the blocking signature pipeline; `ReactiveAgentSignatureVerificationService` holds the reactive pipeline calling into the shared utility.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jakarta CDI, JPA, Mutiny (Uni), JUnit 5, AssertJ, Quarkus Test, Mockito.

---

## Task 1: AgentCryptographicVerifier utility

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifier.java`
- Create: `runtime/src/test/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifierTest.java`

- [ ] **Step 1.1: Write the failing test**

`runtime/src/test/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifierTest.java`:

```java
package io.casehub.ledger.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.time.Instant;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.service.model.VerificationResult;
import io.casehub.ledger.service.supplement.TestEntry;

class AgentCryptographicVerifierTest {

    private KeyPair keyPair;
    private TestEntry entry;

    @BeforeEach
    void setUp() throws Exception {
        keyPair = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        entry = new TestEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = "claude:reviewer@v1";
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Reviewer";
        entry.occurredAt = Instant.now();
    }

    @Test
    void validSignature_returnsValid() throws Exception {
        final byte[] canonical = LedgerMerkleTree.canonicalBytes(entry);
        final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
        sig.initSign(keyPair.getPrivate());
        sig.update(canonical);
        entry.agentSignature = sig.sign();
        entry.agentPublicKey = keyPair.getPublic().getEncoded();

        assertThat(AgentCryptographicVerifier.verifyCryptographic(entry))
                .isEqualTo(VerificationResult.VALID);
    }

    @Test
    void tamperedSignature_returnsInvalid() throws Exception {
        final byte[] canonical = LedgerMerkleTree.canonicalBytes(entry);
        final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
        sig.initSign(keyPair.getPrivate());
        sig.update(canonical);
        final byte[] signature = sig.sign();
        signature[0] ^= 0xFF;
        entry.agentSignature = signature;
        entry.agentPublicKey = keyPair.getPublic().getEncoded();

        assertThat(AgentCryptographicVerifier.verifyCryptographic(entry))
                .isEqualTo(VerificationResult.INVALID);
    }

    @Test
    void missingPublicKey_returnsInvalid() throws Exception {
        entry.agentSignature = new byte[]{0x01};
        entry.agentPublicKey = null;

        assertThat(AgentCryptographicVerifier.verifyCryptographic(entry))
                .isEqualTo(VerificationResult.INVALID);
    }

    @Test
    void mutatedCanonicalField_returnsInvalid() throws Exception {
        final byte[] canonical = LedgerMerkleTree.canonicalBytes(entry);
        final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
        sig.initSign(keyPair.getPrivate());
        sig.update(canonical);
        entry.agentSignature = sig.sign();
        entry.agentPublicKey = keyPair.getPublic().getEncoded();

        // Mutate actorId after signing — canonical bytes will differ
        entry.actorId = "impersonator@v1";

        assertThat(AgentCryptographicVerifier.verifyCryptographic(entry))
                .isEqualTo(VerificationResult.INVALID);
    }
}
```

- [ ] **Step 1.2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentCryptographicVerifierTest 2>&1 | tail -20
```

Expected: compilation error — `AgentCryptographicVerifier` does not exist.

- [ ] **Step 1.3: Create AgentCryptographicVerifier**

`runtime/src/main/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifier.java`:

```java
package io.casehub.ledger.runtime.service;

import java.security.KeyFactory;
import java.security.PublicKey;
import java.security.Signature;
import java.security.spec.X509EncodedKeySpec;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.service.model.VerificationResult;

/**
 * Static Ed25519 signature verification utility.
 * Shared by both the blocking and reactive signature verification beans.
 * Mirrors the {@link LedgerMerkleTree} pattern: pure Java, no IO, no CDI.
 */
class AgentCryptographicVerifier {

    private static final Logger LOG = Logger.getLogger(AgentCryptographicVerifier.class);

    private AgentCryptographicVerifier() {}

    /**
     * Verifies the Ed25519 signature stored on {@code entry} against its stored
     * public key bytes, using the canonical form defined by
     * {@link LedgerMerkleTree#canonicalBytes(LedgerEntry)}.
     *
     * <p>Does NOT check key compromise windows. Returns:
     * <ul>
     *   <li>{@link VerificationResult#INVALID} if the public key is absent (corrupt record)</li>
     *   <li>{@link VerificationResult#VALID} if the signature verifies</li>
     *   <li>{@link VerificationResult#INVALID} if the signature fails or key data is malformed</li>
     * </ul>
     */
    static VerificationResult verifyCryptographic(final LedgerEntry entry) {
        if (entry.agentPublicKey == null) {
            LOG.warnf("Entry %s has agentSignature but no agentPublicKey — record is corrupt", entry.id);
            return VerificationResult.INVALID;
        }
        try {
            final KeyFactory kf = KeyFactory.getInstance("Ed25519");
            final PublicKey pub = kf.generatePublic(new X509EncodedKeySpec(entry.agentPublicKey));
            final Signature sig = Signature.getInstance("Ed25519");
            sig.initVerify(pub);
            sig.update(LedgerMerkleTree.canonicalBytes(entry));
            return sig.verify(entry.agentSignature) ? VerificationResult.VALID : VerificationResult.INVALID;
        } catch (final Exception e) {
            LOG.debugf("Ed25519 verify failed for entry %s (%s) — likely corrupt key data or JVM config issue",
                    entry.id, e.getMessage());
            return VerificationResult.INVALID;
        }
    }
}
```

- [ ] **Step 1.4: Run test — expect 4 passing**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=AgentCryptographicVerifierTest 2>&1 | tail -10
```

Expected: `Tests run: 4, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 1.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifier.java runtime/src/test/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifierTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#93): AgentCryptographicVerifier — extract Ed25519 verify utility"
```

---

## Task 2: AgentSignatureVerificationService + purity tests

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureVerificationService.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/BlockingTierPurityTest.java`

- [ ] **Step 2.1: Add purity test assertions for the new bean**

Add these two tests to `BlockingTierPurityTest` (after the existing `keyRotationService_doesNotInjectReactiveSpi` test):

```java
    @Test
    void agentSignatureVerificationService_hasNoUniMethods() {
        final List<String> uniMethods = uniMethodNames(AgentSignatureVerificationService.class);
        assertThat(uniMethods)
                .as("AgentSignatureVerificationService must contain no Uni<T>-returning methods " +
                        "(reactive variants belong in ReactiveAgentSignatureVerificationService)")
                .isEmpty();
    }

    @Test
    void agentSignatureVerificationService_doesNotInjectReactiveSpi() {
        final List<String> reactiveFields = reactiveFieldNames(AgentSignatureVerificationService.class);
        assertThat(reactiveFields)
                .as("AgentSignatureVerificationService must not inject reactive SPI types " +
                        "(reactive dependencies belong in ReactiveAgentSignatureVerificationService)")
                .isEmpty();
    }
```

Add the import at the top of `BlockingTierPurityTest`:

```java
import io.casehub.ledger.runtime.service.AgentSignatureVerificationService;
```

- [ ] **Step 2.2: Run purity test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BlockingTierPurityTest 2>&1 | tail -10
```

Expected: compilation error — `AgentSignatureVerificationService` does not exist.

- [ ] **Step 2.3: Create AgentSignatureVerificationService**

`runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureVerificationService.java`:

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.model.CompromisedWindow;
import io.casehub.ledger.runtime.service.model.VerificationResult;

/**
 * Blocking-tier CDI bean for agent signature verification.
 *
 * <p>
 * Covers the full verification pipeline: unsigned check, Ed25519 cryptographic
 * verification (via {@link AgentCryptographicVerifier}), key compromise window
 * check (via {@link KeyRotationService}), and {@link AgentSignatureSuspectEvent}
 * firing. Auto-activated — no consumer configuration required.
 *
 * <p>
 * For the reactive variant see {@link ReactiveAgentSignatureVerificationService}.
 */
@ApplicationScoped
public class AgentSignatureVerificationService {

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    KeyRotationService keyRotationService;

    @Inject
    Event<AgentSignatureSuspectEvent> suspectEvent;

    /**
     * Verifies the agent signature on the given entry.
     *
     * @param entryId the entry to verify
     * @return {@link VerificationResult#UNSIGNED} if no signature stored;
     *         {@link VerificationResult#VALID} if the signature verifies and the key is not compromised;
     *         {@link VerificationResult#SUSPECT} if the signature verifies but the key was subsequently
     *         reported {@link io.casehub.ledger.api.model.KeyRotationReason#COMPROMISED} within the
     *         applicable time window — fires {@link AgentSignatureSuspectEvent};
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

        final VerificationResult cryptoResult = AgentCryptographicVerifier.verifyCryptographic(entry);
        if (cryptoResult != VerificationResult.VALID) {
            return cryptoResult;
        }

        if (entry.agentKeyRef != null && entry.actorId != null) {
            final Optional<Instant> effectiveSince =
                    compromisedEffectiveSince(entry.actorId, entry.agentKeyRef, entry.occurredAt);
            if (effectiveSince.isPresent()) {
                suspectEvent.fire(new AgentSignatureSuspectEvent(
                        entryId, entry.actorId, entry.agentKeyRef,
                        entry.occurredAt, effectiveSince.get()));
                return VerificationResult.SUSPECT;
            }
        }

        return VerificationResult.VALID;
    }

    private Optional<Instant> compromisedEffectiveSince(
            final String actorId, final String keyRef, final Instant occurredAt) {
        return keyRotationService.compromisedWindows(actorId, keyRef)
                .stream()
                .filter(w -> !occurredAt.isBefore(w.effectiveSince()))
                .map(CompromisedWindow::effectiveSince)
                .min(Instant::compareTo);
    }
}
```

- [ ] **Step 2.4: Run purity tests — expect 6 passing**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=BlockingTierPurityTest 2>&1 | tail -10
```

Expected: `Tests run: 6, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Step 2.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureVerificationService.java runtime/src/test/java/io/casehub/ledger/service/BlockingTierPurityTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#93): AgentSignatureVerificationService — blocking signature verification bean"
```

---

## Task 3: AgentSignatureVerificationServiceIT + strip signature tests from LedgerVerificationServiceIT

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureVerificationServiceIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java`

- [ ] **Step 3.1: Create AgentSignatureVerificationServiceIT**

This file contains the signature tests moved verbatim from `LedgerVerificationServiceIT`, updated to inject `AgentSignatureVerificationService`:

`runtime/src/test/java/io/casehub/ledger/service/AgentSignatureVerificationServiceIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.time.Instant;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.AgentSignatureVerificationService;
import io.casehub.ledger.runtime.service.AgentSignatureSuspectEvent;
import io.casehub.ledger.runtime.service.KeyRotationService;
import io.casehub.ledger.runtime.service.LedgerMerkleTree;
import io.casehub.ledger.runtime.service.SigningKey;
import io.casehub.ledger.runtime.service.model.VerificationResult;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class AgentSignatureVerificationServiceIT {

    @Inject
    AgentSignatureVerificationService signatureService;
    @Inject
    LedgerEntryRepository repo;
    @Inject
    KeyRotationService rotationService;
    @Inject
    AgentSuspectEventCapture eventCapture;

    private TestEntry seedEntry(final UUID subjectId, final int seq, final String actorId) {
        final TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = seq;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.SYSTEM;
        e.actorRole = "Tester";
        return (TestEntry) repo.save(e);
    }

    @Test
    @Transactional
    void verifyAgentSignature_unsignedEntry_returnsUnsigned() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedEntry(sub, 1, "unsigned-actor");

        assertThat(signatureService.verifyAgentSignature(e.id))
                .isEqualTo(VerificationResult.UNSIGNED);
    }

    @Test
    @Transactional
    void verifyAgentSignature_validSignature_returnsValid() throws Exception {
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedEntry(sub, 1, "signed-actor");

        final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
        final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
        sig.initSign(kp.getPrivate());
        sig.update(canonical);
        e.agentSignature = sig.sign();
        e.agentPublicKey = kp.getPublic().getEncoded();
        e.agentKeyRef = SigningKey.of(kp).keyRef();
        repo.save(e);

        assertThat(signatureService.verifyAgentSignature(e.id))
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
        signature[0] ^= 0xFF;
        e.agentSignature = signature;
        e.agentPublicKey = kp.getPublic().getEncoded();
        e.agentKeyRef = SigningKey.of(kp).keyRef();
        repo.save(e);

        assertThat(signatureService.verifyAgentSignature(e.id))
                .isEqualTo(VerificationResult.INVALID);
    }

    @Test
    @Transactional
    void verifyAgentSignature_unknownEntry_throwsIllegalArgument() {
        final UUID nonexistent = UUID.randomUUID();
        org.junit.jupiter.api.Assertions.assertThrows(
                IllegalArgumentException.class,
                () -> signatureService.verifyAgentSignature(nonexistent));
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
        e.agentKeyRef = SigningKey.of(kp).keyRef();
        e.actorId = "impersonator-actor";
        repo.save(e);

        assertThat(signatureService.verifyAgentSignature(e.id))
                .isEqualTo(VerificationResult.INVALID);
    }

    @Test
    @Transactional
    void verifyAgentSignature_compromisedKey_afterEffectiveSince_returnsSuspect() throws Exception {
        final SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
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

        final Instant compromisedSince = e.occurredAt.minusSeconds(60);
        rotationService.recordRotation("claude:reviewer@v1", sk.keyRef(), null,
                KeyRotationReason.COMPROMISED, compromisedSince);

        assertThat(signatureService.verifyAgentSignature(e.id))
                .isEqualTo(VerificationResult.SUSPECT);
    }

    @Test
    @Transactional
    void verifyAgentSignature_compromisedKey_beforeEffectiveSince_returnsValid() throws Exception {
        final SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
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

        final Instant compromisedSince = e.occurredAt.plusSeconds(3600);
        rotationService.recordRotation("claude:reviewer@v1", sk.keyRef(), null,
                KeyRotationReason.COMPROMISED, compromisedSince);

        assertThat(signatureService.verifyAgentSignature(e.id))
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

        rotationService.recordRotation("claude:reviewer@v1", sk.keyRef(),
                SigningKey.of(gen.generateKeyPair()).keyRef(),
                KeyRotationReason.SCHEDULED, e.occurredAt.minusSeconds(60));

        assertThat(signatureService.verifyAgentSignature(e.id))
                .isEqualTo(VerificationResult.VALID);
    }

    @Test
    @Transactional
    void verifyAgentSignature_suspect_firesSuspectEvent() throws Exception {
        eventCapture.reset();
        final SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
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

        final Instant compromisedSince = e.occurredAt.minusSeconds(60);
        rotationService.recordRotation("claude:reviewer@v1", sk.keyRef(), null,
                KeyRotationReason.COMPROMISED, compromisedSince);

        signatureService.verifyAgentSignature(e.id);

        assertThat(eventCapture.syncEvents()).hasSize(1);
        final AgentSignatureSuspectEvent event = eventCapture.syncEvents().get(0);
        assertThat(event.entryId()).isEqualTo(e.id);
        assertThat(event.actorId()).isEqualTo("claude:reviewer@v1");
        assertThat(event.keyRef()).isEqualTo(sk.keyRef());
        assertThat(event.occurredAt()).isEqualTo(e.occurredAt);
        assertThat(event.effectiveSince()).isEqualTo(compromisedSince);
    }

    @Test
    @Transactional
    void verifyAgentSignature_valid_doesNotFireEvent() throws Exception {
        eventCapture.reset();
        final SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
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

        signatureService.verifyAgentSignature(e.id);

        assertThat(eventCapture.syncEvents()).isEmpty();
    }

    @Test
    @Transactional
    void verifyAgentSignature_invalid_doesNotFireEvent() throws Exception {
        eventCapture.reset();
        final SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedEntry(sub, 1, "claude:reviewer@v1");
        final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
        final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
        sig.initSign(sk.keyPair().getPrivate());
        sig.update(canonical);
        final byte[] signature = sig.sign();
        signature[0] ^= 0xFF;
        e.agentSignature = signature;
        e.agentPublicKey = sk.keyPair().getPublic().getEncoded();
        e.agentKeyRef = sk.keyRef();
        repo.save(e);

        signatureService.verifyAgentSignature(e.id);

        assertThat(eventCapture.syncEvents()).isEmpty();
    }
}
```

- [ ] **Step 3.2: Strip all verifyAgentSignature_* tests from LedgerVerificationServiceIT**

In `runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java`, delete everything from the comment `// ── Agent signature verification ──────────────────────────────────────────` to the end of the file (including that comment and all `verifyAgentSignature_*` test methods).

Also remove these now-unused imports:
```java
import io.casehub.ledger.runtime.service.AgentSignatureSuspectEvent;
import io.casehub.ledger.runtime.service.KeyRotationService;
import io.casehub.ledger.runtime.service.SigningKey;
```

And remove these now-unused fields:
```java
@Inject
KeyRotationService rotationService;
@Inject
AgentSuspectEventCapture eventCapture;
```

- [ ] **Step 3.3: Run both test files — expect all passing**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="AgentSignatureVerificationServiceIT,LedgerVerificationServiceIT" 2>&1 | tail -15
```

Expected: All Merkle tests pass in `LedgerVerificationServiceIT`; all 11 signature tests pass in `AgentSignatureVerificationServiceIT`.

- [ ] **Step 3.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/test/java/io/casehub/ledger/service/AgentSignatureVerificationServiceIT.java runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test(#93): AgentSignatureVerificationServiceIT — move signature tests from LedgerVerificationServiceIT"
```

---

## Task 4: Strip LedgerVerificationService + update blocking-side callers

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AgentSigningIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/SuspectEventIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/KeyRotationIT.java`

- [ ] **Step 4.1: Strip LedgerVerificationService**

Replace the full content of `LedgerVerificationService.java` with the Merkle-only version:

```java
package io.casehub.ledger.runtime.service;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.model.InclusionProof;

/**
 * CDI bean exposing Merkle tree verification operations.
 * Auto-activated — no consumer configuration required.
 *
 * <p>
 * For agent signature verification see {@link AgentSignatureVerificationService}.
 */
@ApplicationScoped
public class LedgerVerificationService {

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    /** Return the current Merkle tree root for a subject. */
    @Transactional
    public String treeRoot(final UUID subjectId) {
        final List<LedgerMerkleFrontier> frontier = em
                .createNamedQuery("LedgerMerkleFrontier.findBySubjectId", LedgerMerkleFrontier.class)
                .setParameter("subjectId", subjectId)
                .getResultList();
        if (frontier.isEmpty()) {
            throw new IllegalStateException("No entries for subject " + subjectId);
        }
        return LedgerMerkleTree.treeRoot(frontier);
    }

    /**
     * Generate an inclusion proof for the given entry.
     * Fetches all leaf hashes for the subject from the database (ordered by sequenceNumber).
     * The returned proof carries the authoritative root from the stored frontier.
     */
    @Transactional
    public InclusionProof inclusionProof(final UUID entryId) {
        final LedgerEntry entry = ledgerRepo.findEntryById(entryId).orElse(null);
        if (entry == null)
            throw new IllegalArgumentException("Entry not found: " + entryId);

        final List<LedgerEntry> allForSubject = ledgerRepo.findBySubjectId(entry.subjectId);

        final List<String> leafHashes = allForSubject.stream()
                .map(e -> e.digest)
                .toList();

        final int k = entry.sequenceNumber - 1;
        final String root = treeRoot(entry.subjectId);
        final InclusionProof proof = LedgerMerkleTree.inclusionProof(
                entryId, k, leafHashes.size(), leafHashes);

        return new InclusionProof(entryId, k, leafHashes.size(),
                proof.leafHash(), proof.siblings(), root);
    }

    /**
     * Verify that all stored digests are consistent with recomputed leaf hashes.
     * Returns false if any entry's stored digest doesn't match its canonical hash.
     */
    @Transactional
    public boolean verify(final UUID subjectId) {
        final List<LedgerEntry> entries = ledgerRepo.findBySubjectId(subjectId);

        List<LedgerMerkleFrontier> frontier = new ArrayList<>();
        for (final LedgerEntry entry : entries) {
            final String expected = LedgerMerkleTree.leafHash(entry);
            if (!expected.equals(entry.digest))
                return false;
            frontier = LedgerMerkleTree.append(expected, frontier, subjectId);
        }

        if (frontier.isEmpty())
            return true;

        final String computed = LedgerMerkleTree.treeRoot(frontier);
        final String stored = treeRoot(subjectId);
        return computed.equals(stored);
    }
}
```

- [ ] **Step 4.2: Run full runtime tests — expect failures in 3 test files**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "ERROR|FAIL|verifyAgentSignature" | head -20
```

Expected: Compilation or test failures in `AgentSigningIT`, `SuspectEventIT`, `KeyRotationIT` — they still reference `verifyAgentSignature` on `LedgerVerificationService`.

- [ ] **Step 4.3: Update AgentSigningIT**

Replace the import and field declarations for the verification service, and update the three signature call sites. The full updated file:

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
import io.casehub.ledger.runtime.service.AgentSignatureVerificationService;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.SigningKey;
import io.casehub.ledger.runtime.service.model.VerificationResult;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class AgentSigningIT {

    @Inject
    LedgerEntryRepository repo;

    @Inject
    AgentSignatureVerificationService signatureService;

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
        when(agentKeyProvider.signingKey(anyString())).thenReturn(Optional.empty());
        when(agentKeyProvider.signingKey("claude:reviewer@v1"))
                .thenReturn(Optional.of(SigningKey.of(testKeyPair)));
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
        assertThat(signatureService.verifyAgentSignature(e.id)).isEqualTo(VerificationResult.VALID);
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
        assertThat(signatureService.verifyAgentSignature(saved.id))
                .isEqualTo(VerificationResult.UNSIGNED);
    }

    @Test
    @Transactional
    void tamperedSignatureBytes_returnsInvalid() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedSigned(sub, 1);

        final LedgerEntry stored = repo.findEntryById(e.id).orElseThrow();
        stored.agentSignature[0] ^= 0xFF;
        em.flush();

        assertThat(signatureService.verifyAgentSignature(e.id)).isEqualTo(VerificationResult.INVALID);
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

- [ ] **Step 4.4: Update SuspectEventIT — blocking injection only**

Change the import and field for the blocking service. Replace:
```java
import io.casehub.ledger.runtime.service.LedgerVerificationService;
```
with:
```java
import io.casehub.ledger.runtime.service.AgentSignatureVerificationService;
```

Replace the field:
```java
@Inject LedgerVerificationService verificationService;
```
with:
```java
@Inject AgentSignatureVerificationService verificationService;
```

Leave `@Inject ReactiveLedgerVerificationService reactiveVerificationService;` unchanged — that will be updated in Task 5.

- [ ] **Step 4.5: Update KeyRotationIT**

Replace the import:
```java
import io.casehub.ledger.runtime.service.LedgerVerificationService;
```
with:
```java
import io.casehub.ledger.runtime.service.AgentSignatureVerificationService;
```

Replace the field:
```java
@Inject LedgerVerificationService verificationService;
```
with:
```java
@Inject AgentSignatureVerificationService verificationService;
```

All four test methods call `verificationService.verifyAgentSignature(...)` — method name is unchanged, no body edits needed.

- [ ] **Step 4.6: Run full runtime tests — expect all passing**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`. All tests pass.

- [ ] **Step 4.7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java \
  runtime/src/test/java/io/casehub/ledger/service/AgentSigningIT.java \
  runtime/src/test/java/io/casehub/ledger/service/SuspectEventIT.java \
  runtime/src/test/java/io/casehub/ledger/service/KeyRotationIT.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "refactor(#93): strip LedgerVerificationService to Merkle-only; update caller injection points"
```

---

## Task 5: ReactiveAgentSignatureVerificationService + LedgerProcessor + reactive test rename

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/ReactiveAgentSignatureVerificationService.java`
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/service/ReactiveLedgerVerificationService.java`
- Modify: `deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/SuspectEventIT.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/ReactiveAgentSignatureVerificationServiceIT.java`
- Delete: `runtime/src/test/java/io/casehub/ledger/service/ReactiveLedgerVerificationServiceIT.java`

- [ ] **Step 5.1: Create ReactiveAgentSignatureVerificationService**

`runtime/src/main/java/io/casehub/ledger/runtime/service/ReactiveAgentSignatureVerificationService.java`:

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;

import io.smallrye.mutiny.Uni;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.ReactiveLedgerEntryRepository;
import io.casehub.ledger.runtime.service.model.CompromisedWindow;
import io.casehub.ledger.runtime.service.model.VerificationResult;

/**
 * Reactive-tier CDI bean for agent signature verification.
 *
 * <p>
 * Present only when {@code casehub.ledger.reactive.enabled=true} — excluded by
 * {@code LedgerProcessor} otherwise. Consumers on a JDBC-only datasource must not
 * depend on this bean.
 *
 * <p>
 * Cryptographic verification delegates to {@link AgentCryptographicVerifier#verifyCryptographic},
 * shared with the blocking tier {@link AgentSignatureVerificationService}.
 */
@ApplicationScoped
public class ReactiveAgentSignatureVerificationService {

    @Inject
    ReactiveLedgerEntryRepository reactiveLedgerRepo;

    @Inject
    ReactiveKeyRotationService reactiveKeyRotationService;

    @Inject
    Event<AgentSignatureSuspectEvent> suspectEvent;

    /**
     * Reactive variant of {@link AgentSignatureVerificationService#verifyAgentSignature(UUID)}.
     *
     * <p>
     * Uses {@link ReactiveLedgerEntryRepository} for the entry lookup and
     * {@link ReactiveKeyRotationService#compromisedWindowsAsync} for the compromise
     * window check. Fires {@link AgentSignatureSuspectEvent} asynchronously via
     * {@code event.fireAsync()} when the result is {@link VerificationResult#SUSPECT}.
     *
     * @param entryId the entry to verify
     * @return a {@link Uni} completing with UNSIGNED, VALID, INVALID, or SUSPECT
     * @throws IllegalArgumentException if the entry does not exist
     */
    public Uni<VerificationResult> verifyAgentSignatureAsync(final UUID entryId) {
        return reactiveLedgerRepo.findEntryById(entryId)
                .map(opt -> opt.orElseThrow(
                        () -> new IllegalArgumentException("Entry not found: " + entryId)))
                .chain(entry -> {
                    if (entry.agentSignature == null) {
                        return Uni.createFrom().item(VerificationResult.UNSIGNED);
                    }

                    final VerificationResult cryptoResult =
                            AgentCryptographicVerifier.verifyCryptographic(entry);
                    if (cryptoResult != VerificationResult.VALID) {
                        return Uni.createFrom().item(cryptoResult);
                    }

                    if (entry.agentKeyRef == null || entry.actorId == null) {
                        return Uni.createFrom().item(VerificationResult.VALID);
                    }

                    return compromisedEffectiveSinceAsync(entry.actorId, entry.agentKeyRef, entry.occurredAt)
                            .chain(effectiveSince -> {
                                if (effectiveSince.isPresent()) {
                                    return Uni.createFrom().completionStage(
                                            () -> suspectEvent.fireAsync(new AgentSignatureSuspectEvent(
                                                    entryId, entry.actorId, entry.agentKeyRef,
                                                    entry.occurredAt, effectiveSince.get())))
                                            .replaceWith(VerificationResult.SUSPECT);
                                }
                                return Uni.createFrom().item(VerificationResult.VALID);
                            });
                });
    }

    private Uni<Optional<Instant>> compromisedEffectiveSinceAsync(
            final String actorId, final String keyRef, final Instant occurredAt) {
        return reactiveKeyRotationService.compromisedWindowsAsync(actorId, keyRef)
                .map(windows -> windows.stream()
                        .filter(w -> !occurredAt.isBefore(w.effectiveSince()))
                        .map(CompromisedWindow::effectiveSince)
                        .min(Instant::compareTo));
    }
}
```

- [ ] **Step 5.2: Update LedgerProcessor**

In `deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java`:

Replace import:
```java
import io.casehub.ledger.runtime.service.ReactiveLedgerVerificationService;
```
with:
```java
import io.casehub.ledger.runtime.service.ReactiveAgentSignatureVerificationService;
```

Replace the `ExcludedTypeBuildItem` line:
```java
new ExcludedTypeBuildItem(ReactiveLedgerVerificationService.class.getName()));
```
with:
```java
new ExcludedTypeBuildItem(ReactiveAgentSignatureVerificationService.class.getName()));
```

- [ ] **Step 5.3: Update SuspectEventIT — reactive injection**

Replace import:
```java
import io.casehub.ledger.runtime.service.ReactiveLedgerVerificationService;
```
with:
```java
import io.casehub.ledger.runtime.service.ReactiveAgentSignatureVerificationService;
```

Replace field:
```java
@Inject ReactiveLedgerVerificationService reactiveVerificationService;
```
with:
```java
@Inject ReactiveAgentSignatureVerificationService reactiveVerificationService;
```

- [ ] **Step 5.4: Create ReactiveAgentSignatureVerificationServiceIT**

`runtime/src/test/java/io/casehub/ledger/service/ReactiveAgentSignatureVerificationServiceIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.security.KeyPairGenerator;
import java.time.Duration;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.AgentSignatureSuspectEvent;
import io.casehub.ledger.runtime.service.KeyRotationService;
import io.casehub.ledger.runtime.service.LedgerMerkleTree;
import io.casehub.ledger.runtime.service.ReactiveAgentSignatureVerificationService;
import io.casehub.ledger.runtime.service.SigningKey;
import io.casehub.ledger.runtime.service.model.VerificationResult;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;

/**
 * Integration tests for {@link ReactiveAgentSignatureVerificationService}.
 *
 * <p>
 * {@code @Transactional} on each test method is required for the setup operations
 * that use the blocking {@link LedgerEntryRepository} and {@link KeyRotationService}.
 * The reactive verification calls go through {@link BlockingReactiveLedgerEntryRepository},
 * which delegates to the blocking repo on the same thread and participates in the same
 * JTA transaction. In production the reactive path runs outside any JTA context.
 */
@QuarkusTest
class ReactiveAgentSignatureVerificationServiceIT {

    @Inject
    ReactiveAgentSignatureVerificationService reactiveVerificationService;

    @Inject
    LedgerEntryRepository repo;

    @Inject
    KeyRotationService rotationService;

    @Inject
    AgentSuspectEventCapture eventCapture;

    private TestEntry seedEntry(final UUID subjectId, final int seq, final String actorId) {
        final TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = seq;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.SYSTEM;
        e.actorRole = "Tester";
        return (TestEntry) repo.save(e);
    }

    @Test
    @Transactional
    void verifyAgentSignatureAsync_unsignedEntry_returnsUnsigned() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedEntry(sub, 1, "unsigned-actor");

        final VerificationResult result = reactiveVerificationService
                .verifyAgentSignatureAsync(e.id)
                .await().atMost(Duration.ofSeconds(5));

        assertThat(result).isEqualTo(VerificationResult.UNSIGNED);
    }

    @Test
    @Transactional
    void verifyAgentSignatureAsync_validSignature_returnsValid() throws Exception {
        final SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedEntry(sub, 1, "signed-actor");
        final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
        final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
        sig.initSign(sk.keyPair().getPrivate());
        sig.update(canonical);
        e.agentSignature = sig.sign();
        e.agentPublicKey = sk.keyPair().getPublic().getEncoded();
        e.agentKeyRef = sk.keyRef();
        repo.save(e);

        final VerificationResult result = reactiveVerificationService
                .verifyAgentSignatureAsync(e.id)
                .await().atMost(Duration.ofSeconds(5));

        assertThat(result).isEqualTo(VerificationResult.VALID);
    }

    @Test
    @Transactional
    void verifyAgentSignatureAsync_tamperedSignature_returnsInvalid() throws Exception {
        final SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedEntry(sub, 1, "signed-actor");
        final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
        final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
        sig.initSign(sk.keyPair().getPrivate());
        sig.update(canonical);
        final byte[] signature = sig.sign();
        signature[0] ^= 0xFF;
        e.agentSignature = signature;
        e.agentPublicKey = sk.keyPair().getPublic().getEncoded();
        e.agentKeyRef = sk.keyRef();
        repo.save(e);

        final VerificationResult result = reactiveVerificationService
                .verifyAgentSignatureAsync(e.id)
                .await().atMost(Duration.ofSeconds(5));

        assertThat(result).isEqualTo(VerificationResult.INVALID);
    }

    @Test
    @Transactional
    void verifyAgentSignatureAsync_suspectEntry_firesEventViaReactivePath() throws Exception {
        eventCapture.reset();
        final SigningKey sk = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
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

        final java.time.Instant compromisedSince = e.occurredAt.minusSeconds(60);
        rotationService.recordRotation("claude:reviewer@v1", sk.keyRef(), null,
                KeyRotationReason.COMPROMISED, compromisedSince);

        final VerificationResult result = reactiveVerificationService
                .verifyAgentSignatureAsync(e.id)
                .await().atMost(Duration.ofSeconds(5));

        assertThat(result).isEqualTo(VerificationResult.SUSPECT);
        assertThat(eventCapture.asyncLatch().await(5, TimeUnit.SECONDS)).isTrue();
        final AgentSignatureSuspectEvent event = eventCapture.lastAsyncEvent();
        assertThat(event.entryId()).isEqualTo(e.id);
        assertThat(event.effectiveSince()).isEqualTo(compromisedSince);
    }
}
```

- [ ] **Step 5.5: Delete old reactive files**

```bash
git -C /Users/mdproctor/claude/casehub/ledger rm \
  runtime/src/main/java/io/casehub/ledger/runtime/service/ReactiveLedgerVerificationService.java \
  runtime/src/test/java/io/casehub/ledger/service/ReactiveLedgerVerificationServiceIT.java
```

- [ ] **Step 5.6: Run full runtime tests — expect all passing**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -15
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 5.7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/service/ReactiveAgentSignatureVerificationService.java \
  deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java \
  runtime/src/test/java/io/casehub/ledger/service/SuspectEventIT.java \
  runtime/src/test/java/io/casehub/ledger/service/ReactiveAgentSignatureVerificationServiceIT.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "refactor(#93): rename ReactiveLedgerVerificationService → ReactiveAgentSignatureVerificationService; eliminate verifyCryptographic duplication"
```

---

## Task 6: Javadoc updates

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureSuspectEvent.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/KeyRotationService.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/ReactiveKeyRotationService.java`

- [ ] **Step 6.1: Update AgentSignatureSuspectEvent class Javadoc**

In `AgentSignatureSuspectEvent.java`, the class-level Javadoc currently references:
```java
* CDI event fired when {@link LedgerVerificationService#verifyAgentSignature(UUID)} or
* {@link LedgerVerificationService#verifyAgentSignatureAsync(UUID)} returns
```

Replace with:
```java
* CDI event fired when {@link AgentSignatureVerificationService#verifyAgentSignature(UUID)} or
* {@link ReactiveAgentSignatureVerificationService#verifyAgentSignatureAsync(UUID)} returns
```

- [ ] **Step 6.2: Update KeyRotationService Javadoc**

In `KeyRotationService.java`, find the Javadoc on `compromisedWindows`:
```java
* Used by {@link LedgerVerificationService} to detect SUSPECT signatures.
```
Replace with:
```java
* Used by {@link AgentSignatureVerificationService} to detect SUSPECT signatures.
```

- [ ] **Step 6.3: Update ReactiveKeyRotationService Javadoc**

In `ReactiveKeyRotationService.java`, find the Javadoc on `compromisedWindowsAsync`:
```java
* Used by {@link ReactiveLedgerVerificationService} to detect SUSPECT signatures.
```
Replace with:
```java
* Used by {@link ReactiveAgentSignatureVerificationService} to detect SUSPECT signatures.
```

- [ ] **Step 6.4: Build to confirm no compile errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureSuspectEvent.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/KeyRotationService.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/ReactiveKeyRotationService.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "docs(#93): update @link references to AgentSignatureVerificationService / ReactiveAgentSignatureVerificationService"
```

---

## Task 7: Full build verification

- [ ] **Step 7.1: Run the complete multi-module build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`. Test count must equal or exceed 450 (was 450 before this branch).

- [ ] **Step 7.2: Confirm test count did not drop**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep "Tests run:" | tail -3
```

The total test count across all `Tests run:` lines should be ≥ 450. (We added `AgentCryptographicVerifierTest` +4 and `AgentSignatureVerificationServiceIT` +11 and `ReactiveAgentSignatureVerificationServiceIT` +4; net increase of ~15 since the duplicate `verifyCryptographic` tests in `ReactiveLedgerVerificationServiceIT` are replaced 1-for-1.)
