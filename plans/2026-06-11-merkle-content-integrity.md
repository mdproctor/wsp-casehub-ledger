# Merkle Content Integrity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expand the Merkle leaf hash and agent signature to cover all tamper-critical fields — structural metadata, supplements, and subclass domain content — so the chain proves what entries contain, not just that they exist.

**Architecture:** Move `canonicalBytes()` from `LedgerMerkleTree` (static) to `LedgerEntry` (final instance method). Add `domainContentBytes()` override point for subclass join-table fields. Extract `AgentSignatureEnricher` from the enricher pipeline into a standalone `AgentEntrySigner` CDI bean. Restructure the save pipeline to: enrich → hash → sign → persist.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (Hibernate), Ed25519 signing, SHA-256 (RFC 9162)

**Spec:** `specs/issue-128-merkle-content-integrity/2026-06-11-merkle-content-integrity-design.md`

---

### Task 1: Add `canonicalBytes()` and `domainContentBytes()` to `LedgerEntry`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/CanonicalBytesTest.java`

- [ ] **Step 1: Write failing tests for the expanded canonical form**

Create `CanonicalBytesTest.java` — pure JUnit 5 (no Quarkus) using `TestEntry`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.service.supplement.TestEntry;
import io.casehub.platform.api.identity.ActorType;

class CanonicalBytesTest {

    private static final Instant FIXED_TIME = Instant.parse("2026-04-18T10:00:00Z");

    private TestEntry entry() {
        final TestEntry e = new TestEntry();
        e.subjectId = UUID.fromString("00000000-0000-0000-0000-000000000001");
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "actor-1";
        e.actorRole = "Tester";
        e.occurredAt = FIXED_TIME;
        e.tenancyId = "tenant-a";
        e.actorType = ActorType.AGENT;
        e.causedByEntryId = UUID.fromString("00000000-0000-0000-0000-000000000099");
        return e;
    }

    @Test
    void includesNewBaseFields() {
        final String canonical = new String(entry().canonicalBytes(), StandardCharsets.UTF_8);
        assertThat(canonical).contains("tenant-a");
        assertThat(canonical).contains("AGENT");
        assertThat(canonical).contains("00000000-0000-0000-0000-000000000099");
    }

    @Test
    void includesSupplementJson_whenPresent() {
        final TestEntry e = entry();
        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.rationale = "approved by policy";
        e.attach(cs);

        final String canonical = new String(e.canonicalBytes(), StandardCharsets.UTF_8);
        assertThat(canonical).contains("approved by policy");
    }

    @Test
    void excludesSupplementJson_whenAbsent() {
        final TestEntry e = entry();
        final String canonical = new String(e.canonicalBytes(), StandardCharsets.UTF_8);
        // 9 pipe-delimited fields, no trailing supplement
        assertThat(canonical.split("\\|")).hasSize(9);
    }

    @Test
    void isDeterministic() {
        final TestEntry e = entry();
        assertThat(e.canonicalBytes()).isEqualTo(e.canonicalBytes());
    }

    @Test
    void mutatingTenancyId_changesHash() {
        final TestEntry e = entry();
        final byte[] before = e.canonicalBytes();
        e.tenancyId = "tenant-b";
        assertThat(e.canonicalBytes()).isNotEqualTo(before);
    }

    @Test
    void mutatingActorType_changesHash() {
        final TestEntry e = entry();
        final byte[] before = e.canonicalBytes();
        e.actorType = ActorType.HUMAN;
        assertThat(e.canonicalBytes()).isNotEqualTo(before);
    }

    @Test
    void mutatingCausedByEntryId_changesHash() {
        final TestEntry e = entry();
        final byte[] before = e.canonicalBytes();
        e.causedByEntryId = UUID.randomUUID();
        assertThat(e.canonicalBytes()).isNotEqualTo(before);
    }

    @Test
    void mutatingSupplementField_afterRefresh_changesHash() {
        final TestEntry e = entry();
        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.confidenceScore = 0.3;
        e.attach(cs);
        final byte[] before = e.canonicalBytes();

        cs.confidenceScore = 0.95;
        e.refreshSupplementJson();
        assertThat(e.canonicalBytes()).isNotEqualTo(before);
    }

    @Test
    void domainContentBytes_defaultIsEmpty() {
        assertThat(entry().domainContentBytes()).isEmpty();
    }

    @Test
    void nullBaseFields_produceEmptyStringsInCanonical() {
        final TestEntry e = new TestEntry();
        e.sequenceNumber = 0;
        // All nullable fields are null — should not NPE
        final String canonical = new String(e.canonicalBytes(), StandardCharsets.UTF_8);
        assertThat(canonical).isNotNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="CanonicalBytesTest" -q`
Expected: compilation failure — `canonicalBytes()` does not exist on `LedgerEntry`

- [ ] **Step 3: Implement `canonicalBytes()` and `domainContentBytes()` on `LedgerEntry`**

Add to `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`:

```java
import java.nio.charset.StandardCharsets;
import java.time.temporal.ChronoUnit;

private static final byte[] EMPTY_BYTES = new byte[0];

/**
 * Returns the canonical byte representation of this entry for Merkle hashing
 * and agent signing.
 *
 * <p>Covers: structural metadata (subjectId, seqNum, entryType, actorId,
 * actorRole, occurredAt, tenancyId, actorType, causedByEntryId), supplement
 * content (supplementJson), and subclass domain content
 * ({@link #domainContentBytes()}).
 *
 * <p>{@code final} — the structural encoding is a sealed contract. Subclasses
 * extend content through {@link #domainContentBytes()} only.
 */
public final byte[] canonicalBytes() {
    final String structural = String.join("|",
        subjectId != null ? subjectId.toString() : "",
        String.valueOf(sequenceNumber),
        entryType != null ? entryType.name() : "",
        actorId != null ? actorId : "",
        actorRole != null ? actorRole : "",
        occurredAt != null ? occurredAt.truncatedTo(ChronoUnit.MILLIS).toString() : "",
        tenancyId != null ? tenancyId : "",
        actorType != null ? actorType.name() : "",
        causedByEntryId != null ? causedByEntryId.toString() : "");

    final String withSupplements = supplementJson != null
        ? structural + "|" + supplementJson
        : structural;

    final byte[] base = withSupplements.getBytes(StandardCharsets.UTF_8);
    final byte[] domain = domainContentBytes();

    if (domain.length == 0) {
        return base;
    }

    final byte[] combined = new byte[base.length + 1 + domain.length];
    System.arraycopy(base, 0, combined, 0, base.length);
    combined[base.length] = (byte) '|';
    System.arraycopy(domain, 0, combined, base.length + 1, domain.length);
    return combined;
}

/**
 * Returns domain-specific content bytes for hash protection.
 *
 * <p>Subclasses that declare persistent fields on join tables MUST override
 * this method to include those fields. The returned bytes are appended to the
 * canonical form used by both the Merkle leaf hash and the agent signature.
 *
 * <p>Build-time enforcement: {@code LedgerProcessor} produces a deployment error
 * if a {@code LedgerEntry} subclass declares persistent fields (non-{@code @Transient})
 * but does not override this method.
 *
 * @return domain content bytes; empty array if no domain fields exist
 */
protected byte[] domainContentBytes() {
    return EMPTY_BYTES;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="CanonicalBytesTest" -q`
Expected: all 10 tests PASS

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java \
       runtime/src/test/java/io/casehub/ledger/service/CanonicalBytesTest.java
git commit -m "feat(#128): add canonicalBytes() and domainContentBytes() to LedgerEntry"
```

---

### Task 2: Implement `domainContentBytes()` on internal subclasses

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/KeyRotationEntry.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorIdentityBindingEntry.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/DomainContentBytesTest.java`

- [ ] **Step 1: Write failing tests for domain content bytes**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.ledger.runtime.model.PlainLedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.IdentityBindingStatus;

class DomainContentBytesTest {

    @Test
    void plainLedgerEntry_domainContentBytesIsEmpty() {
        final PlainLedgerEntry e = new PlainLedgerEntry();
        assertThat(e.domainContentBytes()).isEmpty();
    }

    @Test
    void keyRotationEntry_includesAllFields() {
        final KeyRotationEntry e = new KeyRotationEntry();
        e.previousKeyRef = "old-key-ref";
        e.newKeyRef = "new-key-ref";
        e.reason = KeyRotationReason.COMPROMISED;
        e.effectiveSince = Instant.parse("2026-06-01T00:00:00Z");

        final String content = new String(e.domainContentBytes(), StandardCharsets.UTF_8);
        assertThat(content).contains("old-key-ref");
        assertThat(content).contains("new-key-ref");
        assertThat(content).contains("COMPROMISED");
        assertThat(content).contains("2026-06-01");
    }

    @Test
    void keyRotationEntry_mutatingReason_changesCanonicalBytes() {
        final KeyRotationEntry e = new KeyRotationEntry();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "agent";
        e.actorRole = "KeyManager";
        e.occurredAt = Instant.now();
        e.tenancyId = "t";
        e.actorType = ActorType.SYSTEM;
        e.previousKeyRef = "old";
        e.newKeyRef = "new";
        e.reason = KeyRotationReason.SCHEDULED;
        e.effectiveSince = Instant.now();

        final byte[] before = e.canonicalBytes();
        e.reason = KeyRotationReason.COMPROMISED;
        assertThat(e.canonicalBytes()).isNotEqualTo(before);
    }

    @Test
    void identityBindingEntry_includesAllFields() {
        final ActorIdentityBindingEntry e = new ActorIdentityBindingEntry();
        e.boundDid = "did:web:example.com";
        e.validationResult = IdentityBindingStatus.VERIFIED;
        e.alsoKnownAsVerified = true;
        e.keyMatchVerified = true;
        e.verifiedKeyRef = "key-ref-abc";
        e.didMethod = "web";

        final String content = new String(e.domainContentBytes(), StandardCharsets.UTF_8);
        assertThat(content).contains("did:web:example.com");
        assertThat(content).contains("VERIFIED");
        assertThat(content).contains("true");
        assertThat(content).contains("key-ref-abc");
        assertThat(content).contains("web");
    }

    @Test
    void identityBindingEntry_mutatingBoundDid_changesCanonicalBytes() {
        final ActorIdentityBindingEntry e = new ActorIdentityBindingEntry();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "agent";
        e.actorRole = "Identity";
        e.occurredAt = Instant.now();
        e.tenancyId = "t";
        e.actorType = ActorType.AGENT;
        e.boundDid = "did:web:old.com";
        e.validationResult = IdentityBindingStatus.VERIFIED;
        e.alsoKnownAsVerified = true;
        e.keyMatchVerified = true;
        e.effectiveSince = Instant.now();

        final byte[] before = e.canonicalBytes();
        e.boundDid = "did:web:tampered.com";
        assertThat(e.canonicalBytes()).isNotEqualTo(before);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="DomainContentBytesTest" -q`
Expected: FAIL — `keyRotationEntry_includesAllFields` and `identityBindingEntry_includesAllFields` fail because `domainContentBytes()` returns empty bytes

- [ ] **Step 3: Add `domainContentBytes()` to `KeyRotationEntry`**

Add to `KeyRotationEntry.java`:

```java
import java.nio.charset.StandardCharsets;
import java.time.temporal.ChronoUnit;

@Override
protected byte[] domainContentBytes() {
    return String.join("|",
        previousKeyRef != null ? previousKeyRef : "",
        newKeyRef != null ? newKeyRef : "",
        reason != null ? reason.name() : "",
        effectiveSince != null ? effectiveSince.truncatedTo(ChronoUnit.MILLIS).toString() : ""
    ).getBytes(StandardCharsets.UTF_8);
}
```

- [ ] **Step 4: Add `domainContentBytes()` to `ActorIdentityBindingEntry`**

Add to `ActorIdentityBindingEntry.java`:

```java
import java.nio.charset.StandardCharsets;

@Override
protected byte[] domainContentBytes() {
    return String.join("|",
        boundDid != null ? boundDid : "",
        validationResult != null ? validationResult.name() : "",
        String.valueOf(alsoKnownAsVerified),
        String.valueOf(keyMatchVerified),
        verifiedKeyRef != null ? verifiedKeyRef : "",
        credentialResult != null ? credentialResult.name() : "",
        didMethod != null ? didMethod : ""
    ).getBytes(StandardCharsets.UTF_8);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="DomainContentBytesTest" -q`
Expected: all 5 tests PASS

- [ ] **Step 6: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/model/KeyRotationEntry.java \
       runtime/src/main/java/io/casehub/ledger/runtime/model/ActorIdentityBindingEntry.java \
       runtime/src/test/java/io/casehub/ledger/service/DomainContentBytesTest.java
git commit -m "feat(#128): add domainContentBytes() to KeyRotationEntry and ActorIdentityBindingEntry"
```

---

### Task 3: Delete `canonicalBytes()` from `LedgerMerkleTree`, migrate all call sites

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerkleTree.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifier.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSigner.java` (javadoc only)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java` (javadoc only)
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerMerkleTreeTest.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifierTest.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureVerificationServiceIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/ReactiveAgentSignatureVerificationServiceIT.java`

Note: `AgentSignatureEnricher` is also a call site but is deleted in Task 4 — skip it here.

- [ ] **Step 1: Delete `canonicalBytes()` from `LedgerMerkleTree`**

Remove lines 169-180 (`public static byte[] canonicalBytes(...)`) from `LedgerMerkleTree.java`.

Update `leafHash()` to call the instance method:

```java
public static String leafHash(final LedgerEntry entry) {
    final byte[] canonical = entry.canonicalBytes();
    final byte[] input = new byte[1 + canonical.length];
    input[0] = 0x00;
    System.arraycopy(canonical, 0, input, 1, canonical.length);
    return sha256hex(input);
}
```

- [ ] **Step 2: Update `AgentCryptographicVerifier`**

Line 61: change `sig.update(LedgerMerkleTree.canonicalBytes(entry))` to `sig.update(entry.canonicalBytes())`.

Update javadoc at line 41: change `{@link LedgerMerkleTree#canonicalBytes(LedgerEntry)}` to `{@link LedgerEntry#canonicalBytes()}`.

- [ ] **Step 3: Update javadoc references**

`AgentSigner.java` line 29: change `{@link LedgerMerkleTree#canonicalBytes}` to `{@link LedgerEntry#canonicalBytes()}`.

`LedgerEntry.java` line 161 (`agentSignature` field javadoc): change `{@link io.casehub.ledger.runtime.service.LedgerMerkleTree#canonicalBytes(LedgerEntry)}` to `{@link #canonicalBytes()}`.

- [ ] **Step 4: Update test call sites**

`AgentCryptographicVerifierTest.java` (3 sites at lines 39, 52, 85): change `LedgerMerkleTree.canonicalBytes(entry)` to `entry.canonicalBytes()`.

`AgentSignatureVerificationServiceIT.java` line 56: change `LedgerMerkleTree.canonicalBytes(e)` to `e.canonicalBytes()`.

`ReactiveAgentSignatureVerificationServiceIT.java` line 95: change `LedgerMerkleTree.canonicalBytes(e)` to `e.canonicalBytes()`.

`LedgerMerkleTreeTest.java`: the test helper `entry()` needs to set the new base fields (`tenancyId`, `actorType`) so hashes are deterministic. Add to the `entry()` method:

```java
e.tenancyId = "default-tenant";
e.actorType = ActorType.AGENT;
```

Add import: `import io.casehub.platform.api.identity.ActorType;`

- [ ] **Step 5: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`
Expected: all tests PASS (some hash values will have changed — tests that compare hash equality against themselves are fine; tests that used hardcoded hash strings would fail but there are none)

- [ ] **Step 6: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerkleTree.java \
       runtime/src/main/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifier.java \
       runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSigner.java \
       runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java \
       runtime/src/test/java/io/casehub/ledger/service/LedgerMerkleTreeTest.java \
       runtime/src/test/java/io/casehub/ledger/runtime/service/AgentCryptographicVerifierTest.java \
       runtime/src/test/java/io/casehub/ledger/service/AgentSignatureVerificationServiceIT.java \
       runtime/src/test/java/io/casehub/ledger/service/ReactiveAgentSignatureVerificationServiceIT.java
git commit -m "refactor(#128): delete LedgerMerkleTree.canonicalBytes(), migrate all call sites to entry.canonicalBytes()"
```

---

### Task 4: Extract signing from enricher pipeline — create `AgentEntrySigner`

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentEntrySigner.java`
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java` → rename to `AgentEntrySignerTest.java`

- [ ] **Step 1: Create `AgentEntrySigner`**

```java
package io.casehub.ledger.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.model.LedgerEntry;

@ApplicationScoped
public class AgentEntrySigner {

    private static final Logger LOG = Logger.getLogger(AgentEntrySigner.class);

    private final AgentSigner signer;

    @Inject
    public AgentEntrySigner(final AgentSigner signer) {
        this.signer = signer;
    }

    public void sign(final LedgerEntry entry) {
        if (entry.actorId == null || entry.agentSignature != null) return;
        try {
            signer.sign(entry.actorId, entry.canonicalBytes())
                    .ifPresent(sig -> {
                        entry.agentSignature = sig.signature();
                        entry.agentPublicKey = sig.publicKey();
                        entry.agentKeyRef = sig.keyRef();
                    });
        } catch (final Exception e) {
            LOG.warnf("AgentEntrySigner failed for actor %s — entry will be unsigned: %s",
                    entry.actorId, e.getMessage());
        }
    }
}
```

- [ ] **Step 2: Rename and update the test**

Rename `AgentSignatureEnricherTest.java` to `AgentEntrySignerTest.java`. Update:
- Class name: `AgentEntrySignerTest`
- Field: `AgentEntrySigner signer` (was `AgentSignatureEnricher enricher`)
- Construction: `new AgentEntrySigner(...)` (was `new AgentSignatureEnricher(...)`)
- All `enricher.enrich(e)` calls → `signer.sign(e)`
- Line 69: `LedgerMerkleTree.canonicalBytes(e)` → `e.canonicalBytes()`
- Remove import of `AgentSignatureEnricher` and `LedgerMerkleTree`

- [ ] **Step 3: Delete `AgentSignatureEnricher.java`**

Delete `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java`.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="AgentEntrySignerTest" -q`
Expected: all 7 tests PASS

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/AgentEntrySigner.java \
       runtime/src/test/java/io/casehub/ledger/service/AgentEntrySignerTest.java
git rm runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureEnricher.java \
       runtime/src/test/java/io/casehub/ledger/service/AgentSignatureEnricherTest.java
git commit -m "refactor(#128): extract AgentEntrySigner from enricher pipeline — signing is sealing, not enrichment"
```

---

### Task 5: Fix missing `@Priority` annotations on enrichers

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TraceIdEnricher.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/intercept/ProvenanceCaptureEnricher.java`

- [ ] **Step 1: Add `@Priority(10)` to `TraceIdEnricher`**

Add `import jakarta.annotation.Priority;` and `@Priority(10)` annotation to the class.

- [ ] **Step 2: Add `@Priority(30)` to `ProvenanceCaptureEnricher`**

Add `import jakarta.annotation.Priority;` and `@Priority(30)` annotation to the class.

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`
Expected: all tests PASS

- [ ] **Step 4: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TraceIdEnricher.java \
       runtime/src/main/java/io/casehub/ledger/runtime/service/intercept/ProvenanceCaptureEnricher.java
git commit -m "fix(#128): add missing @Priority(10) and @Priority(30) to TraceIdEnricher and ProvenanceCaptureEnricher"
```

---

### Task 6: Restructure save pipeline — enrich → hash → sign → persist

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerTraceListener.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryEnricher.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/PipelineOrderingIT.java`

- [ ] **Step 1: Write failing integration test for pipeline ordering**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.PlainLedgerEntry;
import io.casehub.ledger.runtime.model.supplement.ProvenanceSupplement;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerMerkleTree;
import io.casehub.ledger.runtime.service.intercept.ProvenanceContext;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class PipelineOrderingIT {

    @Inject
    LedgerEntryRepository repo;

    @Inject
    ProvenanceContext provenanceContext;

    @Test
    void provenanceSupplement_isReflectedInDigest() {
        provenanceContext.push("WorkItem", UUID.randomUUID(), "casehub-work");
        try {
            final PlainLedgerEntry e = new PlainLedgerEntry();
            e.subjectId = UUID.randomUUID();
            e.entryType = LedgerEntryType.EVENT;
            e.actorId = "test-actor";
            e.actorType = ActorType.AGENT;
            e.actorRole = "Tester";

            final var saved = repo.save(e, TenancyConstants.DEFAULT_TENANT_ID);

            assertThat(saved.supplementJson).isNotNull();
            assertThat(saved.supplementJson).contains("casehub-work");

            // The digest must have been computed AFTER the enricher added the supplement
            final String recomputedDigest = LedgerMerkleTree.leafHash(saved);
            assertThat(saved.digest).isEqualTo(recomputedDigest);
        } finally {
            provenanceContext.pop();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="PipelineOrderingIT" -q`
Expected: FAIL — `saved.digest` was computed before provenance enrichment, so it does not match `recomputedDigest` which includes `supplementJson`

- [ ] **Step 3: Restructure `JpaLedgerEntryRepository.save()`**

Inject `AgentEntrySigner` and `LedgerEnricherPipeline`. Change the save method ordering:

```java
@Inject
AgentEntrySigner agentEntrySigner;

@Inject
LedgerEnricherPipeline enricherPipeline;

@Override
@Transactional
public LedgerEntry save(final LedgerEntry entry, final String tenancyId) {
    entry.tenancyId = tenancyId;

    if (entry.subjectId == null) {
        throw new IllegalArgumentException("LedgerEntry.subjectId must not be null");
    }
    if (entry.occurredAt == null) {
        entry.occurredAt = Instant.now();
    }
    if (entry.actorId != null) {
        entry.actorId = actorIdentityProvider.tokenise(entry.actorId);
    }
    entry.compliance().ifPresent(cs -> {
        if (cs.decisionContext != null) {
            cs.decisionContext = decisionContextSanitiser.sanitise(cs.decisionContext);
            entry.refreshSupplementJson();
        }
    });

    entry.sequenceNumber = sequenceAllocator.nextSequenceNumber(entry.subjectId);

    enricherPipeline.enrich(entry);                       // content enrichment

    if (ledgerConfig.hashChain().enabled()) {
        entry.digest = LedgerMerkleTree.leafHash(entry);  // hash AFTER enrichment
    }

    agentEntrySigner.sign(entry);                         // seal

    em.persist(entry);                                     // persist — no enricher pipeline

    if (ledgerConfig.hashChain().enabled()) {
        updateMerkleFrontier(entry, tenancyId);
    }

    return entry;
}
```

- [ ] **Step 4: Restructure `InMemoryLedgerEntryRepository.save()`**

Change the save method: split existing `enricherPipeline.enrich(entry)` and inject `AgentEntrySigner`. Add the same three-phase sequence:

```java
@Inject
AgentEntrySigner agentEntrySigner;

// In save():
// ... existing stamp/validation code ...

entry.sequenceNumber = sequenceCounters
        .computeIfAbsent(entry.subjectId, k -> new AtomicInteger(0))
        .incrementAndGet();

enricherPipeline.enrich(entry);                       // content enrichment

if (ledgerConfig.hashChain().enabled()) {
    entry.digest = LedgerMerkleTree.leafHash(entry);  // hash AFTER enrichment
}

agentEntrySigner.sign(entry);                         // seal

entries.put(entry.id, entry);
// ... existing Merkle frontier update ...
```

- [ ] **Step 5: Update `LedgerTraceListener` — replace enricher call with defensive check**

```java
@ApplicationScoped
public class LedgerTraceListener {

    @Inject
    LedgerConfig ledgerConfig;

    @PrePersist
    public void prePersist(final Object entity) {
        if (!(entity instanceof LedgerEntry entry)) return;
        if (ledgerConfig.hashChain().enabled() && entry.digest == null) {
            throw new IllegalStateException(
                "LedgerEntry must be persisted through LedgerEntryRepository, "
                + "which handles sequence allocation, enrichment, hashing, and signing. "
                + "Direct em.persist() bypasses the entire save pipeline.");
        }
    }
}
```

Remove the `LedgerEnricherPipeline` injection from `LedgerTraceListener`.

- [ ] **Step 6: Rewrite `LedgerEntryEnricher` javadoc**

Replace the entire javadoc on the `LedgerEntryEnricher` interface with the contract from the spec (Section 5). Key changes: pipeline position is "BEFORE hashing and signing", enrichers MAY attach supplements, must use `attach()` or `refreshSupplementJson()`.

- [ ] **Step 7: Run the pipeline ordering test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="PipelineOrderingIT" -q`
Expected: PASS

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`
Expected: all tests PASS

- [ ] **Step 9: Run persistence-memory tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,runtime,deployment -q -DskipTests && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -q`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java \
       runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerTraceListener.java \
       runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryEnricher.java \
       persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java \
       runtime/src/test/java/io/casehub/ledger/service/PipelineOrderingIT.java
git commit -m "feat(#128): restructure save pipeline — enrich → hash → sign → persist"
```

---

### Task 7: Add build-time guard for `domainContentBytes()` override enforcement

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java`
- Create: `runtime/src/test/java/io/casehub/ledger/deployment/DomainContentBytesGuardTest.java`

- [ ] **Step 1: Write failing test**

This tests the build-time guard. Create a test that verifies the guard fires for a subclass with persistent fields and no override. The simplest approach: test the guard method directly as a unit test, passing mock Jandex data.

```java
package io.casehub.ledger.deployment;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.jboss.jandex.Index;
import org.jboss.jandex.Indexer;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.PlainLedgerEntry;

class DomainContentBytesGuardTest {

    @Test
    void plainLedgerEntry_noFields_passes() throws Exception {
        final Index index = buildIndex(LedgerEntry.class, PlainLedgerEntry.class);
        assertThatCode(() -> LedgerProcessor.validateDomainContentBytes(index))
                .doesNotThrowAnyException();
    }

    @Test
    void keyRotationEntry_withOverride_passes() throws Exception {
        final Index index = buildIndex(LedgerEntry.class, KeyRotationEntry.class);
        assertThatCode(() -> LedgerProcessor.validateDomainContentBytes(index))
                .doesNotThrowAnyException();
    }

    private static Index buildIndex(final Class<?>... classes) throws Exception {
        final Indexer indexer = new Indexer();
        for (final Class<?> c : classes) {
            indexer.indexClass(c);
        }
        return indexer.complete();
    }
}
```

Note: Testing the failure case (subclass with fields but no override) requires a test @Entity class that deliberately omits the override. Since `CaseLedgerEntry` (engine) is not on the test classpath, and creating a fake @Entity subclass with persistent fields but no override just for the test is possible but requires the class to be on the Jandex index. The implementation test should verify: (1) PlainLedgerEntry passes, (2) KeyRotationEntry (has override) passes. The negative case is tested implicitly when engine picks up the new ledger SNAPSHOT — their build fails until they add the override.

- [ ] **Step 2: Implement the build-time guard in `LedgerProcessor`**

Add to `LedgerProcessor.java`:

```java
import org.jboss.jandex.ClassInfo;
import org.jboss.jandex.DotName;
import org.jboss.jandex.FieldInfo;
import org.jboss.jandex.IndexView;

private static final DotName ENTITY = DotName.createSimple("jakarta.persistence.Entity");
private static final DotName TRANSIENT = DotName.createSimple("jakarta.persistence.Transient");
private static final DotName LEDGER_ENTRY = DotName.createSimple(
        "io.casehub.ledger.runtime.model.LedgerEntry");

@BuildStep
void validateDomainContentBytes(CombinedIndexBuildItem index) {
    validateDomainContentBytes(index.getIndex());
}

// package-private for testing
static void validateDomainContentBytes(final IndexView index) {
    for (final ClassInfo subclass : index.getAllKnownSubclasses(LEDGER_ENTRY)) {
        if (!subclass.hasAnnotation(ENTITY)) continue;

        final boolean hasPersistentFields = subclass.fields().stream()
                .anyMatch(f -> !f.hasAnnotation(TRANSIENT));

        if (!hasPersistentFields) continue;

        final boolean overrides = subclass.method("domainContentBytes") != null;

        if (!overrides) {
            final String fieldNames = subclass.fields().stream()
                    .filter(f -> !f.hasAnnotation(TRANSIENT))
                    .map(FieldInfo::name)
                    .collect(java.util.stream.Collectors.joining(", "));
            throw new jakarta.enterprise.inject.spi.DeploymentException(
                    subclass.name().withoutPackagePrefix()
                    + " declares persistent fields (" + fieldNames
                    + ") but does not override domainContentBytes(). "
                    + "These fields are not hash-protected. Override domainContentBytes() "
                    + "to include them in the Merkle leaf hash and agent signature.");
        }
    }
}
```

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl deployment -q`
Expected: PASS

- [ ] **Step 4: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q`
Expected: BUILD SUCCESS — all internal subclasses have overrides (or no persistent fields)

- [ ] **Step 5: Commit**

```
git add deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java \
       runtime/src/test/java/io/casehub/ledger/deployment/DomainContentBytesGuardTest.java
git commit -m "feat(#128): add build-time guard — @Entity subclasses with persistent fields must override domainContentBytes()"
```

---

### Task 8: Update documentation — DESIGN.md, CLAUDE.md, LedgerEntryEnricher javadoc

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update DESIGN.md**

Replace the "Hash chain canonical form" section (around line 213) with:

```markdown
### Hash chain canonical form

The leaf hash covers all tamper-critical fields: structural metadata (`subjectId`,
`seqNum`, `entryType`, `actorId`, `actorRole`, `occurredAt`, `tenancyId`, `actorType`,
`causedByEntryId`), base-table supplements (`supplementJson`), and subclass domain
content via `domainContentBytes()`.

`canonicalBytes()` is a `public final` instance method on `LedgerEntry` — the canonical
form is a property of the entry, not of the Merkle tree utility. `final` seals the
structural encoding; subclasses extend content through `domainContentBytes()` only.
A build-time guard in `LedgerProcessor` enforces this for `@Entity` subclasses with
persistent fields.

The save pipeline runs in four phases: content enrichment → hashing → agent signing →
persist. `AgentEntrySigner` is a direct call in the save pipeline, not an enricher —
signing seals the entry, it does not add content.
```

Also update the enricher pipeline section to remove `AgentSignatureEnricher` from the enricher list and note it's now `AgentEntrySigner` (direct call).

- [ ] **Step 2: Update CLAUDE.md**

Update the project structure tree:
- Replace `AgentSignatureEnricher.java` reference with `AgentEntrySigner.java` and update its description
- Update the `canonicalBytes()` reference in the "Merkle leaf hash canonical form" section to note it's now an instance method on `LedgerEntry`

- [ ] **Step 3: Commit**

```
git add docs/DESIGN.md CLAUDE.md
git commit -m "docs(#128): update DESIGN.md and CLAUDE.md for Merkle content integrity"
```

---

### Task 9: Final verification — full build and test

- [ ] **Step 1: Full clean build with all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS with 0 failures (excluding Docker-dependent PostgreSQL tests)

- [ ] **Step 2: Verify test count hasn't decreased**

Check that the test count is >= 714 (current count on main).

- [ ] **Step 3: Review all changes**

Run: `git log --oneline main..HEAD` — should show 8 commits, one per task.
Run: `git diff main..HEAD --stat` — review the full delta.
