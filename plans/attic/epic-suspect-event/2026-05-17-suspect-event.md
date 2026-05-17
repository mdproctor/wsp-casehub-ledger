# AgentSignatureSuspectEvent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fire a CDI event whenever `verifyAgentSignature()` (or its new async twin) detects a SUSPECT signature — giving consumers a real-time hook for alerting without polling.

**Architecture:** `AgentSignatureSuspectEvent` is a plain CDI record following the `LedgerGapDetected` pattern. The sync path fires `event.fire()`; the new `verifyAgentSignatureAsync()` fires `event.fireAsync()`. A `verifyCryptographic()` helper is extracted to avoid duplication across the two paths. A blocking bridge (`Uni.createFrom().item()`) calls `KeyRotationService.compromisedWindows()` from the reactive path (see #86 for the full reactive variant).

**Tech Stack:** Java 21, Quarkus 3.32.2, SmallRye Mutiny (`Uni`), CDI `Event<T>` / `@ObservesAsync`, `ReactiveLedgerEntryRepository`

---

## Task 1: `AgentSignatureSuspectEvent` record + `AgentSuspectEventCapture` test observer

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureSuspectEvent.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/AgentSuspectEventCapture.java`

- [ ] **Step 1: Create `AgentSignatureSuspectEvent`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureSuspectEvent.java`:

```java
package io.casehub.ledger.runtime.service;

import java.time.Instant;
import java.util.UUID;

/**
 * CDI event fired when {@link LedgerVerificationService#verifyAgentSignature(UUID)} or
 * {@link LedgerVerificationService#verifyAgentSignatureAsync(UUID)} returns
 * {@link io.casehub.ledger.runtime.service.model.VerificationResult#SUSPECT}.
 *
 * <p>
 * SUSPECT means the signature is cryptographically valid but was produced by a key
 * subsequently reported {@link io.casehub.ledger.api.model.KeyRotationReason#COMPROMISED}
 * within the applicable time window.
 *
 * <p>
 * Consumers:
 * <pre>{@code
 * // Synchronous consumer
 * void onSuspect(@Observes AgentSignatureSuspectEvent e) { ... }
 *
 * // Asynchronous consumer (non-blocking)
 * CompletionStage<Void> onSuspect(@ObservesAsync AgentSignatureSuspectEvent e) { ... }
 * }</pre>
 *
 * @param entryId        the entry whose signature is SUSPECT
 * @param actorId        the actor who signed the entry
 * @param keyRef         the key generation reported COMPROMISED
 * @param occurredAt     when the entry was signed
 * @param effectiveSince the earliest matching compromise window — "compromised since when"
 */
public record AgentSignatureSuspectEvent(
        UUID entryId,
        String actorId,
        String keyRef,
        Instant occurredAt,
        Instant effectiveSince) {}
```

- [ ] **Step 2: Create `AgentSuspectEventCapture` test CDI bean**

Create `runtime/src/test/java/io/casehub/ledger/service/AgentSuspectEventCapture.java`:

```java
package io.casehub.ledger.service;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.enterprise.event.Observes;

import io.casehub.ledger.runtime.service.AgentSignatureSuspectEvent;

/**
 * Test CDI bean that captures {@link AgentSignatureSuspectEvent} fired during tests.
 * Registered automatically by the Quarkus test container.
 */
@ApplicationScoped
public class AgentSuspectEventCapture {

    private final List<AgentSignatureSuspectEvent> syncEvents = new CopyOnWriteArrayList<>();
    private volatile AgentSignatureSuspectEvent lastAsyncEvent;
    private volatile CountDownLatch asyncLatch = new CountDownLatch(1);

    void onSuspectSync(@Observes final AgentSignatureSuspectEvent event) {
        syncEvents.add(event);
    }

    CompletionStage<Void> onSuspectAsync(@ObservesAsync final AgentSignatureSuspectEvent event) {
        lastAsyncEvent = event;
        asyncLatch.countDown();
        return CompletableFuture.completedFuture(null);
    }

    public List<AgentSignatureSuspectEvent> syncEvents() {
        return syncEvents;
    }

    public AgentSignatureSuspectEvent lastAsyncEvent() {
        return lastAsyncEvent;
    }

    public CountDownLatch asyncLatch() {
        return asyncLatch;
    }

    public void reset() {
        syncEvents.clear();
        lastAsyncEvent = null;
        asyncLatch = new CountDownLatch(1);
    }
}
```

- [ ] **Step 3: Verify compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureSuspectEvent.java \
        runtime/src/test/java/io/casehub/ledger/service/AgentSuspectEventCapture.java
git commit -m "feat(#83): AgentSignatureSuspectEvent record + AgentSuspectEventCapture test observer"
```

---

## Task 2: Extract `verifyCryptographic()` + update sync `verifyAgentSignature()` — TDD

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java`

- [ ] **Step 1: Write failing tests for event firing**

Add these imports to `LedgerVerificationServiceIT.java`:
```java
import io.casehub.ledger.runtime.service.AgentSignatureSuspectEvent;
import io.casehub.ledger.service.AgentSuspectEventCapture;
```

Add field: `@Inject AgentSuspectEventCapture eventCapture;`

Add these tests at the end of the agent signature section (after the existing SUSPECT/VALID/SCHEDULED tests). Note that `seedEntry` already exists as a helper.

```java
@Test
@Transactional
void verifyAgentSignature_suspect_firesSuspectEvent() throws Exception {
    eventCapture.reset();
    final java.security.KeyPairGenerator gen = java.security.KeyPairGenerator.getInstance("Ed25519");
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

    final Instant compromisedSince = e.occurredAt.minusSeconds(60);
    rotationService.recordRotation("claude:reviewer@v1", sk.keyRef(), null,
            KeyRotationReason.COMPROMISED, compromisedSince);

    verificationService.verifyAgentSignature(e.id);

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
    final java.security.KeyPairGenerator gen = java.security.KeyPairGenerator.getInstance("Ed25519");
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

    verificationService.verifyAgentSignature(e.id);

    assertThat(eventCapture.syncEvents()).isEmpty();
}

@Test
@Transactional
void verifyAgentSignature_invalid_doesNotFireEvent() throws Exception {
    eventCapture.reset();
    final java.security.KeyPairGenerator gen = java.security.KeyPairGenerator.getInstance("Ed25519");
    final SigningKey sk = SigningKey.of(gen.generateKeyPair());
    final UUID sub = UUID.randomUUID();
    final TestEntry e = seedEntry(sub, 1, "claude:reviewer@v1");
    final byte[] canonical = LedgerMerkleTree.canonicalBytes(e);
    final java.security.Signature sig = java.security.Signature.getInstance("Ed25519");
    sig.initSign(sk.keyPair().getPrivate());
    sig.update(canonical);
    final byte[] signature = sig.sign();
    signature[0] ^= 0xFF; // corrupt
    e.agentSignature = signature;
    e.agentPublicKey = sk.keyPair().getPublic().getEncoded();
    e.agentKeyRef = sk.keyRef();
    repo.save(e);

    verificationService.verifyAgentSignature(e.id);

    assertThat(eventCapture.syncEvents()).isEmpty();
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerVerificationServiceIT -q 2>&1 | tail -5
```

Expected: compile error — `AgentSignatureSuspectEvent` not yet injected in service.

- [ ] **Step 3: Update `LedgerVerificationService`**

Add these imports to `LedgerVerificationService.java`:
```java
import java.util.Optional;
import jakarta.enterprise.event.Event;
import io.smallrye.mutiny.Uni;
import io.casehub.ledger.runtime.repository.ReactiveLedgerEntryRepository;
import io.casehub.ledger.runtime.service.model.CompromisedWindow;
```

Add injections after the existing `@Inject KeyRotationService keyRotationService;`:
```java
    @Inject
    Event<AgentSignatureSuspectEvent> suspectEvent;

    @Inject
    ReactiveLedgerEntryRepository reactiveLedgerRepo;
```

Add this private helper method before `verifyAgentSignature`:
```java
    /**
     * Ed25519 signature verification against stored public key bytes.
     * Returns {@link VerificationResult#VALID} or {@link VerificationResult#INVALID}.
     * Does NOT check compromise windows.
     */
    private VerificationResult verifyCryptographic(final LedgerEntry entry) {
        if (entry.agentPublicKey == null) {
            LOG.warnf("Entry %s has agentSignature but no agentPublicKey — record is corrupt", entry.id);
            return VerificationResult.INVALID;
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

Replace the body of `verifyAgentSignature` with:
```java
    @Transactional
    public VerificationResult verifyAgentSignature(final UUID entryId) {
        final LedgerEntry entry = ledgerRepo.findEntryById(entryId)
                .orElseThrow(() -> new IllegalArgumentException("Entry not found: " + entryId));

        if (entry.agentSignature == null) {
            return VerificationResult.UNSIGNED;
        }

        final VerificationResult cryptoResult = verifyCryptographic(entry);
        if (cryptoResult != VerificationResult.VALID) {
            return cryptoResult;
        }

        if (entry.agentKeyRef != null && entry.actorId != null) {
            final Optional<Instant> effectiveSince = keyRotationService
                    .compromisedWindows(entry.actorId, entry.agentKeyRef)
                    .stream()
                    .filter(w -> !entry.occurredAt.isBefore(w.effectiveSince()))
                    .map(CompromisedWindow::effectiveSince)
                    .min(Instant::compareTo);
            if (effectiveSince.isPresent()) {
                suspectEvent.fire(new AgentSignatureSuspectEvent(
                        entryId, entry.actorId, entry.agentKeyRef,
                        entry.occurredAt, effectiveSince.get()));
                return VerificationResult.SUSPECT;
            }
        }

        return VerificationResult.VALID;
    }
```

Add required import: `import java.time.Instant;`

- [ ] **Step 4: Run tests — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerVerificationServiceIT -q
```

Expected: BUILD SUCCESS, all tests pass (existing + 3 new).

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java \
        runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java
git commit -m "feat(#83): extract verifyCryptographic(), fire AgentSignatureSuspectEvent on SUSPECT"
```

---

## Task 3: Add `verifyAgentSignatureAsync()` — TDD

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/SuspectEventIT.java`

- [ ] **Step 1: Write the integration test**

Create `runtime/src/test/java/io/casehub/ledger/service/SuspectEventIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.when;

import java.security.KeyPairGenerator;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.AgentKeyProvider;
import io.casehub.ledger.runtime.service.AgentSignatureSuspectEvent;
import io.casehub.ledger.runtime.service.KeyRotationService;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.SigningKey;
import io.casehub.ledger.runtime.service.model.VerificationResult;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class SuspectEventIT {

    @Inject LedgerEntryRepository repo;
    @Inject LedgerVerificationService verificationService;
    @Inject KeyRotationService rotationService;
    @Inject AgentSuspectEventCapture eventCapture;

    @InjectMock
    AgentKeyProvider agentKeyProvider;

    private SigningKey testKey;

    @BeforeEach
    void setUp() throws Exception {
        testKey = SigningKey.of(KeyPairGenerator.getInstance("Ed25519").generateKeyPair());
        when(agentKeyProvider.signingKey(anyString())).thenReturn(Optional.empty());
        when(agentKeyProvider.signingKey("claude:reviewer@v1"))
                .thenReturn(Optional.of(testKey));
        eventCapture.reset();
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
    void verifyAgentSignature_suspect_firesSyncEvent() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedSigned(sub, 1);

        rotationService.recordRotation("claude:reviewer@v1", testKey.keyRef(), null,
                KeyRotationReason.COMPROMISED, e.occurredAt.minusSeconds(60));

        final VerificationResult result = verificationService.verifyAgentSignature(e.id);

        assertThat(result).isEqualTo(VerificationResult.SUSPECT);
        assertThat(eventCapture.syncEvents()).hasSize(1);
        final AgentSignatureSuspectEvent event = eventCapture.syncEvents().get(0);
        assertThat(event.entryId()).isEqualTo(e.id);
        assertThat(event.actorId()).isEqualTo("claude:reviewer@v1");
        assertThat(event.keyRef()).isEqualTo(testKey.keyRef());
    }

    @Test
    @Transactional
    void verifyAgentSignatureAsync_unsigned_returnsUnsigned() {
        final TestEntry e = new TestEntry();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "system:noop";
        e.actorType = ActorType.SYSTEM;
        e.actorRole = "System";
        final TestEntry saved = (TestEntry) repo.save(e);

        final VerificationResult result =
                verificationService.verifyAgentSignatureAsync(saved.id).await().indefinitely();

        assertThat(result).isEqualTo(VerificationResult.UNSIGNED);
        assertThat(eventCapture.syncEvents()).isEmpty();
    }

    @Test
    @Transactional
    void verifyAgentSignatureAsync_valid_returnsValid() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedSigned(sub, 1);

        final VerificationResult result =
                verificationService.verifyAgentSignatureAsync(e.id).await().indefinitely();

        assertThat(result).isEqualTo(VerificationResult.VALID);
        assertThat(eventCapture.syncEvents()).isEmpty();
    }

    @Test
    @Transactional
    void verifyAgentSignatureAsync_suspect_firesAsyncEvent() throws Exception {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedSigned(sub, 1);

        rotationService.recordRotation("claude:reviewer@v1", testKey.keyRef(), null,
                KeyRotationReason.COMPROMISED, e.occurredAt.minusSeconds(60));

        final VerificationResult result =
                verificationService.verifyAgentSignatureAsync(e.id).await().indefinitely();

        assertThat(result).isEqualTo(VerificationResult.SUSPECT);

        // Wait for async event delivery
        final boolean received = eventCapture.asyncLatch().await(5, TimeUnit.SECONDS);
        assertThat(received).as("async event must be received within 5 seconds").isTrue();

        final AgentSignatureSuspectEvent event = eventCapture.lastAsyncEvent();
        assertThat(event).isNotNull();
        assertThat(event.entryId()).isEqualTo(e.id);
        assertThat(event.actorId()).isEqualTo("claude:reviewer@v1");
        assertThat(event.keyRef()).isEqualTo(testKey.keyRef());
    }

    @Test
    @Transactional
    void verifyAgentSignatureAsync_invalid_returnsInvalid() {
        final UUID sub = UUID.randomUUID();
        final TestEntry e = seedSigned(sub, 1);
        // Corrupt the stored signature
        final io.casehub.ledger.runtime.model.LedgerEntry stored =
                repo.findEntryById(e.id).orElseThrow();
        stored.agentSignature[0] ^= 0xFF;
        repo.save(stored);

        final VerificationResult result =
                verificationService.verifyAgentSignatureAsync(e.id).await().indefinitely();

        assertThat(result).isEqualTo(VerificationResult.INVALID);
        assertThat(eventCapture.syncEvents()).isEmpty();
    }
}
```

- [ ] **Step 2: Run — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SuspectEventIT -q 2>&1 | tail -5
```

Expected: compile error — `verifyAgentSignatureAsync` not yet defined.

- [ ] **Step 3: Add `verifyAgentSignatureAsync()` to `LedgerVerificationService`**

Add this method after `verifyAgentSignature()`:

```java
    /**
     * Reactive variant of {@link #verifyAgentSignature(UUID)}.
     *
     * <p>
     * Uses {@link ReactiveLedgerEntryRepository} for the entry lookup.
     * {@link KeyRotationService#compromisedWindows} is called via a blocking bridge
     * pending full reactive {@link KeyRotationService} variants (see casehubio/ledger#86).
     *
     * <p>
     * Fires {@link AgentSignatureSuspectEvent} asynchronously via {@code event.fireAsync()}
     * when the result is {@link VerificationResult#SUSPECT}.
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

                    final VerificationResult cryptoResult = verifyCryptographic(entry);
                    if (cryptoResult != VerificationResult.VALID) {
                        return Uni.createFrom().item(cryptoResult);
                    }

                    if (entry.agentKeyRef == null || entry.actorId == null) {
                        return Uni.createFrom().item(VerificationResult.VALID);
                    }

                    // Blocking bridge — see #86 for reactive KeyRotationService
                    final Optional<Instant> effectiveSince = keyRotationService
                            .compromisedWindows(entry.actorId, entry.agentKeyRef)
                            .stream()
                            .filter(w -> !entry.occurredAt.isBefore(w.effectiveSince()))
                            .map(CompromisedWindow::effectiveSince)
                            .min(Instant::compareTo);

                    if (effectiveSince.isPresent()) {
                        return Uni.createFrom().completionStage(
                                () -> suspectEvent.fireAsync(new AgentSignatureSuspectEvent(
                                        entryId, entry.actorId, entry.agentKeyRef,
                                        entry.occurredAt, effectiveSince.get())))
                                .replaceWith(VerificationResult.SUSPECT);
                    }

                    return Uni.createFrom().item(VerificationResult.VALID);
                });
    }
```

- [ ] **Step 4: Run IT — expect pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=SuspectEventIT -q
```

Expected: BUILD SUCCESS, 5 tests pass.

- [ ] **Step 5: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java \
        runtime/src/test/java/io/casehub/ledger/service/SuspectEventIT.java
git commit -m "feat(#83): verifyAgentSignatureAsync() — reactive path with fireAsync on SUSPECT"
```

---

## Task 4: Code review + doc sync + install + epic close

- [ ] **Step 1: Invoke `superpowers:requesting-code-review`**

Review all changes for #83 before the final commit.

- [ ] **Step 2: Fix any findings Minor or above**

Any finding not fixed this session → GitHub issue. Batch related nits into one issue.

- [ ] **Step 3: Run full install**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Invoke `implementation-doc-sync`**

Sync CLAUDE.md, DESIGN.md for new components: `AgentSignatureSuspectEvent`, `verifyAgentSignatureAsync`, `verifyCryptographic` extraction.

- [ ] **Step 5: Push epic branch and close issue**

```bash
git push -u origin epic-suspect-event
gh issue close 83 --repo casehubio/ledger --comment "Implemented on epic-suspect-event branch."
```

- [ ] **Step 6: Invoke `/epic close`**

Close the epic: promote artifacts, merge journal, clean up branches.
