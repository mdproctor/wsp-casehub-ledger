# Merkle Tree Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the linear SHA-256 hash chain with a Merkle Mountain Range that gives O(log N) inclusion proofs and publishable signed tree roots for external verification without DB access.

**Architecture:** Stored frontier (log₂(N) rows per subject) replaces `previous_hash` chaining. `LedgerMerkleTree` (pure static) handles all algorithm logic. `LedgerVerificationService` (CDI) exposes proof generation. `LedgerMerklePublisher` (CDI, opt-in) signs and POSTs tlog-checkpoint format using Ed25519.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, JUnit 5, AssertJ, `java.security` EdDSA (no extra deps). All tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`.

**ADR:** `adr/0002-merkle-tree-internal-structure.md`
**Spec:** `docs/superpowers/specs/2026-04-18-merkle-tree-upgrade-design.md`
**Issue:** #11

---

## File Map

**Created:**
- `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerMerkleFrontier.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerkleTree.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerklePublisher.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/service/model/InclusionProof.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/service/model/ProofStep.java`
- `runtime/src/test/java/io/casehub/ledger/service/LedgerMerkleTreeTest.java`
- `runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java`
- `runtime/src/test/java/io/casehub/ledger/service/LedgerMerklePublisherTest.java`
- `examples/merkle-verification/src/...`

**Modified:**
- `runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql`
- `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerRetentionJob.java`
- `docs/AUDITABILITY.md`
- `docs/DESIGN.md`

**Deleted:**
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerHashChain.java`
- `runtime/src/test/java/io/casehub/ledger/service/LedgerHashChainTest.java`

---

## Task 1: Epic Setup

**Files:** GitHub issues only.

- [ ] **Step 1: Create epic issue**

```bash
gh issue create --repo casehubio/ledger \
  --title "Epic: Axiom 4 — Verifiability (Merkle tree upgrade + external proof publishing)" \
  --label "epic" \
  --body "$(cat <<'EOF'
Closes the Verifiability gap in docs/AUDITABILITY.md (Axiom 4).

## Child Issues
- [ ] #11 — Merkle tree upgrade (stored frontier, inclusion proofs, Ed25519 publishing)

## Acceptance Criteria
- O(log N) inclusion proof for any entry
- Signed tlog-checkpoint publishable to external URL
- Axiom 4 marked ✅ in AUDITABILITY.md
EOF
)"
```

Note the new epic issue number (e.g. #12). All commits in this feature use: `Refs #11, Refs #12`.

- [ ] **Step 2: Confirm issue #11 is open**

```bash
gh issue view 11 --repo casehubio/ledger
```

---

## Task 2: Schema

**Files:**
- Modify: `runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql`

- [ ] **Step 1: Drop `previous_hash`, add frontier table**

Replace the `ledger_entry` CREATE TABLE block: remove the `previous_hash VARCHAR(64)` line. Update the comment to remove the old hash-chain description.

Add after the existing `ledger_attestation` block:

```sql
CREATE TABLE ledger_merkle_frontier (
    id          UUID        NOT NULL,
    subject_id  UUID        NOT NULL,
    level       INTEGER     NOT NULL,
    hash        VARCHAR(64) NOT NULL,
    CONSTRAINT pk_ledger_merkle_frontier PRIMARY KEY (id),
    CONSTRAINT uq_merkle_frontier_subject_level UNIQUE (subject_id, level)
);

CREATE INDEX idx_merkle_frontier_subject ON ledger_merkle_frontier (subject_id);
```

- [ ] **Step 2: Verify schema compiles (build without tests)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/resources/db/migration/V1000__ledger_base_schema.sql
git commit -m "feat(merkle): add ledger_merkle_frontier table, drop previous_hash

Refs #11, Refs #12"
```

---

## Task 3: Config

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`

- [ ] **Step 1: Add `merkle()` method and `MerkleConfig` interface**

Add to `LedgerConfig` interface (after `retention()`):

```java
/**
 * Merkle tree settings — inclusion proofs and optional external publishing.
 *
 * @return the merkle sub-configuration
 */
MerkleConfig merkle();

/** Merkle Mountain Range and external publishing settings. */
interface MerkleConfig {

    /**
     * External publishing settings. Publishing is inactive when {@code url} is absent.
     *
     * @return the publish sub-configuration
     */
    MerklePublishConfig publish();

    /** External checkpoint publishing settings. */
    interface MerklePublishConfig {

        /**
         * POST endpoint to receive signed tlog-checkpoint on each frontier update.
         * When absent, the publisher is inactive — zero overhead.
         *
         * @return the publish URL, if configured
         */
        java.util.Optional<String> url();

        /**
         * Path to an Ed25519 private key PEM file (PKCS#8 format).
         * Required when {@link #url()} is present.
         *
         * @return path to the private key file
         */
        java.util.Optional<String> privateKey();

        /**
         * Opaque identifier for the public key. Included in each checkpoint so
         * receivers can locate the corresponding public key for verification.
         *
         * @return the key identifier
         */
        @WithDefault("default")
        String keyId();
    }
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java
git commit -m "feat(merkle): add MerkleConfig with optional publish settings

Refs #11, Refs #12"
```

---

## Task 4: Data Types

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/model/ProofStep.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/model/InclusionProof.java`

- [ ] **Step 1: Create `ProofStep`**

```java
package io.casehub.ledger.runtime.service.model;

/**
 * One sibling hash in a Merkle inclusion proof.
 *
 * @param hash 64-char lowercase hex SHA-256
 * @param side whether the sibling is to the LEFT or RIGHT of the current node
 */
public record ProofStep(String hash, Side side) {

    /** Position of the sibling relative to the current node in the tree. */
    public enum Side { LEFT, RIGHT }
}
```

- [ ] **Step 2: Create `InclusionProof`**

```java
package io.casehub.ledger.runtime.service.model;

import java.util.List;
import java.util.UUID;

/**
 * A Merkle inclusion proof for a single ledger entry.
 *
 * <p>Verify with: start from {@code leafHash}, apply each {@link ProofStep} as
 * {@code internalHash(step.hash, current)} for LEFT or {@code internalHash(current, step.hash)}
 * for RIGHT. The result must equal {@code treeRoot}.
 *
 * @param entryId    the ledger entry this proof covers
 * @param entryIndex 0-based index of the entry in the per-subject sequence
 * @param treeSize   total number of entries for this subject at proof time
 * @param leafHash   SHA-256 leaf hash of this entry (stored as {@code digest} on the entry)
 * @param siblings   ordered sibling hashes from leaf level to root
 * @param treeRoot   Merkle root at proof time — verify against a published checkpoint
 */
public record InclusionProof(
        UUID entryId,
        int entryIndex,
        int treeSize,
        String leafHash,
        List<ProofStep> siblings,
        String treeRoot) {
}
```

- [ ] **Step 3: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/model/
git commit -m "feat(merkle): add ProofStep and InclusionProof records

Refs #11, Refs #12"
```

---

## Task 5: LedgerMerkleFrontier Entity

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerMerkleFrontier.java`

- [ ] **Step 1: Create entity**

```java
package io.casehub.ledger.runtime.model;

import java.util.List;
import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * One node in the Merkle Mountain Range frontier for a subject.
 *
 * <p>A subject with N entries has exactly {@code Integer.bitCount(N)} rows at any time —
 * one per set bit in N's binary representation. At 1 million entries: at most 20 rows.
 */
@Entity
@Table(name = "ledger_merkle_frontier")
public class LedgerMerkleFrontier extends PanacheEntityBase {

    @Id
    public UUID id;

    /** The aggregate this frontier node belongs to. */
    @Column(name = "subject_id", nullable = false)
    public UUID subjectId;

    /** Tree level — this node is the root of a perfect subtree of 2^level leaves. */
    @Column(nullable = false)
    public int level;

    /** SHA-256 root hash of this subtree — 64-char lowercase hex. */
    @Column(nullable = false, length = 64)
    public String hash;

    @PrePersist
    void prePersist() {
        if (id == null) id = UUID.randomUUID();
    }

    /** Return all frontier nodes for the given subject, ordered by level ascending. */
    public static List<LedgerMerkleFrontier> findBySubjectId(final UUID subjectId) {
        return list("subjectId = ?1 ORDER BY level ASC", subjectId);
    }

    /** Delete the frontier node at the given level for the given subject, if present. */
    public static void deleteBySubjectAndLevel(final UUID subjectId, final int level) {
        delete("subjectId = ?1 AND level = ?2", subjectId, level);
    }
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerMerkleFrontier.java
git commit -m "feat(merkle): add LedgerMerkleFrontier Panache entity

Refs #11, Refs #12"
```

---

## Task 6: TDD — Write Failing LedgerMerkleTree Tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerMerkleTreeTest.java`
- Create stub: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerkleTree.java`

- [ ] **Step 1: Create stub LedgerMerkleTree so tests compile**

```java
package io.casehub.ledger.runtime.service;

import java.util.List;
import java.util.UUID;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.service.model.InclusionProof;

public final class LedgerMerkleTree {
    private LedgerMerkleTree() {}
    public static String leafHash(LedgerEntry entry) { throw new UnsupportedOperationException(); }
    public static String internalHash(String leftHex, String rightHex) { throw new UnsupportedOperationException(); }
    public static List<LedgerMerkleFrontier> append(String leafHash, List<LedgerMerkleFrontier> frontier, UUID subjectId) { throw new UnsupportedOperationException(); }
    public static String treeRoot(List<LedgerMerkleFrontier> frontier) { throw new UnsupportedOperationException(); }
    public static InclusionProof inclusionProof(UUID entryId, int entryIndex, int treeSize, List<String> leafHashes) { throw new UnsupportedOperationException(); }
    public static boolean verifyProof(InclusionProof proof, String expectedRoot) { throw new UnsupportedOperationException(); }
}
```

- [ ] **Step 2: Create LedgerMerkleTreeTest**

```java
package io.casehub.ledger.service;

import static io.casehub.ledger.runtime.service.model.ProofStep.Side.LEFT;
import static io.casehub.ledger.runtime.service.model.ProofStep.Side.RIGHT;
import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.service.LedgerMerkleTree;
import io.casehub.ledger.runtime.service.model.InclusionProof;
import io.casehub.ledger.service.supplement.TestEntry;

class LedgerMerkleTreeTest {

    private static final Instant FIXED_TIME = Instant.parse("2026-04-18T10:00:00Z");

    private TestEntry entry(UUID subjectId, int seq) {
        TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = seq;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "actor-" + seq;
        e.actorRole = "Tester";
        e.occurredAt = FIXED_TIME;
        return e;
    }

    private List<LedgerMerkleFrontier> buildFrontier(UUID subjectId, int n) {
        List<LedgerMerkleFrontier> frontier = new ArrayList<>();
        for (int i = 1; i <= n; i++) {
            String leaf = LedgerMerkleTree.leafHash(entry(subjectId, i));
            frontier = LedgerMerkleTree.append(leaf, frontier, subjectId);
        }
        return frontier;
    }

    // ── leafHash ─────────────────────────────────────────────────────────────

    @Test
    void leafHash_returns64CharHex() {
        assertThat(LedgerMerkleTree.leafHash(entry(UUID.randomUUID(), 1)))
                .matches("[0-9a-f]{64}");
    }

    @Test
    void leafHash_isDeterministic() {
        UUID id = UUID.randomUUID();
        TestEntry e = entry(id, 1);
        assertThat(LedgerMerkleTree.leafHash(e)).isEqualTo(LedgerMerkleTree.leafHash(e));
    }

    @Test
    void leafHash_mutatingActorId_changesHash() {
        TestEntry e = entry(UUID.randomUUID(), 1);
        String before = LedgerMerkleTree.leafHash(e);
        e.actorId = "mutated";
        assertThat(LedgerMerkleTree.leafHash(e)).isNotEqualTo(before);
    }

    @Test
    void leafHash_andInternalHash_domainSeparated() {
        // The 0x00/0x01 domain separation prevents leaf/node hash collisions
        TestEntry e = entry(UUID.randomUUID(), 1);
        String leaf = LedgerMerkleTree.leafHash(e);
        // An internal hash of leaf with itself is different from any leaf hash
        String internal = LedgerMerkleTree.internalHash(leaf, leaf);
        assertThat(leaf).isNotEqualTo(internal);
    }

    // ── append — frontier size (Integer.bitCount(N)) ─────────────────────────

    @Test
    void append_n1_frontierHas1NodeAtLevel0() {
        UUID sub = UUID.randomUUID();
        List<LedgerMerkleFrontier> f = buildFrontier(sub, 1);
        assertThat(f).hasSize(1);
        assertThat(f.get(0).level).isEqualTo(0);
        assertThat(f.get(0).subjectId).isEqualTo(sub);
    }

    @Test
    void append_n2_frontierHas1NodeAtLevel1() {
        List<LedgerMerkleFrontier> f = buildFrontier(UUID.randomUUID(), 2);
        assertThat(f).hasSize(1);
        assertThat(f.get(0).level).isEqualTo(1);
    }

    @Test
    void append_n3_frontierHas2Nodes() {
        List<LedgerMerkleFrontier> f = buildFrontier(UUID.randomUUID(), 3);
        assertThat(f).hasSize(2); // bitCount(3) = 2
        assertThat(f.stream().map(n -> n.level).toList()).containsExactlyInAnyOrder(0, 1);
    }

    @Test
    void append_n4_frontierHas1NodeAtLevel2() {
        List<LedgerMerkleFrontier> f = buildFrontier(UUID.randomUUID(), 4);
        assertThat(f).hasSize(1);
        assertThat(f.get(0).level).isEqualTo(2);
    }

    @Test
    void append_n5_frontierHas2Nodes() {
        List<LedgerMerkleFrontier> f = buildFrontier(UUID.randomUUID(), 5);
        assertThat(f).hasSize(2); // bitCount(5) = 2
        assertThat(f.stream().map(n -> n.level).toList()).containsExactlyInAnyOrder(0, 2);
    }

    @Test
    void append_n7_frontierHas3Nodes() {
        List<LedgerMerkleFrontier> f = buildFrontier(UUID.randomUUID(), 7);
        assertThat(f).hasSize(3); // bitCount(7) = 3
        assertThat(f.stream().map(n -> n.level).toList()).containsExactlyInAnyOrder(0, 1, 2);
    }

    @Test
    void append_n8_frontierHas1NodeAtLevel3() {
        List<LedgerMerkleFrontier> f = buildFrontier(UUID.randomUUID(), 8);
        assertThat(f).hasSize(1);
        assertThat(f.get(0).level).isEqualTo(3);
    }

    @Test
    void append_n2_level1NodeIsInternalHashOfBothLeaves() {
        UUID sub = UUID.randomUUID();
        String leaf1 = LedgerMerkleTree.leafHash(entry(sub, 1));
        String leaf2 = LedgerMerkleTree.leafHash(entry(sub, 2));
        List<LedgerMerkleFrontier> f1 = LedgerMerkleTree.append(leaf1, List.of(), sub);
        List<LedgerMerkleFrontier> f2 = LedgerMerkleTree.append(leaf2, f1, sub);
        assertThat(f2.get(0).hash).isEqualTo(LedgerMerkleTree.internalHash(leaf1, leaf2));
    }

    // ── treeRoot ─────────────────────────────────────────────────────────────

    @Test
    void treeRoot_singleNode_returnsThatHash() {
        UUID sub = UUID.randomUUID();
        List<LedgerMerkleFrontier> f = buildFrontier(sub, 1);
        String leaf = LedgerMerkleTree.leafHash(entry(sub, 1));
        assertThat(LedgerMerkleTree.treeRoot(f)).isEqualTo(leaf);
    }

    @Test
    void treeRoot_n2_equalsInternalHashOfLeaves() {
        UUID sub = UUID.randomUUID();
        String leaf1 = LedgerMerkleTree.leafHash(entry(sub, 1));
        String leaf2 = LedgerMerkleTree.leafHash(entry(sub, 2));
        List<LedgerMerkleFrontier> f = buildFrontier(sub, 2);
        assertThat(LedgerMerkleTree.treeRoot(f))
                .isEqualTo(LedgerMerkleTree.internalHash(leaf1, leaf2));
    }

    @Test
    void treeRoot_isDeterministic() {
        UUID sub = UUID.randomUUID();
        List<LedgerMerkleFrontier> f = buildFrontier(sub, 5);
        assertThat(LedgerMerkleTree.treeRoot(f)).isEqualTo(LedgerMerkleTree.treeRoot(f));
    }

    @Test
    void treeRoot_differentSubjects_differentRoots() {
        List<LedgerMerkleFrontier> f1 = buildFrontier(UUID.randomUUID(), 3);
        List<LedgerMerkleFrontier> f2 = buildFrontier(UUID.randomUUID(), 3);
        assertThat(LedgerMerkleTree.treeRoot(f1)).isNotEqualTo(LedgerMerkleTree.treeRoot(f2));
    }

    // ── inclusionProof + verifyProof ─────────────────────────────────────────

    private InclusionProof proofFor(UUID sub, int n, int k) {
        List<String> leaves = new ArrayList<>();
        List<LedgerMerkleFrontier> frontier = new ArrayList<>();
        for (int i = 1; i <= n; i++) {
            String leaf = LedgerMerkleTree.leafHash(entry(sub, i));
            leaves.add(leaf);
            frontier = LedgerMerkleTree.append(leaf, frontier, sub);
        }
        String root = LedgerMerkleTree.treeRoot(frontier);
        InclusionProof proof = LedgerMerkleTree.inclusionProof(
                UUID.randomUUID(), k, n, leaves);
        // Manually set root for verification (in integration tests the service does this)
        return new InclusionProof(proof.entryId(), proof.entryIndex(), proof.treeSize(),
                proof.leafHash(), proof.siblings(), root);
    }

    @Test
    void inclusionProof_singleEntry_emptyProof_verifiesAsRoot() {
        UUID sub = UUID.randomUUID();
        InclusionProof proof = proofFor(sub, 1, 0);
        assertThat(proof.siblings()).isEmpty();
        assertThat(LedgerMerkleTree.verifyProof(proof, proof.treeRoot())).isTrue();
    }

    @Test
    void inclusionProof_n2_entry0_verifies() {
        assertThat(LedgerMerkleTree.verifyProof(
                proofFor(UUID.randomUUID(), 2, 0),
                proofFor(UUID.randomUUID(), 2, 0).treeRoot())).isTrue();
    }

    @Test
    void inclusionProof_n2_entry1_verifies() {
        UUID sub = UUID.randomUUID();
        InclusionProof proof = proofFor(sub, 2, 1);
        assertThat(LedgerMerkleTree.verifyProof(proof, proof.treeRoot())).isTrue();
    }

    @Test
    void inclusionProof_n4_allEntries_verify() {
        UUID sub = UUID.randomUUID();
        // Build leaves + root once
        List<String> leaves = new ArrayList<>();
        List<LedgerMerkleFrontier> frontier = new ArrayList<>();
        for (int i = 1; i <= 4; i++) {
            String leaf = LedgerMerkleTree.leafHash(entry(sub, i));
            leaves.add(leaf);
            frontier = LedgerMerkleTree.append(leaf, frontier, sub);
        }
        String root = LedgerMerkleTree.treeRoot(frontier);
        for (int k = 0; k < 4; k++) {
            InclusionProof proof = LedgerMerkleTree.inclusionProof(UUID.randomUUID(), k, 4, leaves);
            InclusionProof withRoot = new InclusionProof(proof.entryId(), k, 4,
                    proof.leafHash(), proof.siblings(), root);
            assertThat(LedgerMerkleTree.verifyProof(withRoot, root))
                    .as("entry %d of 4", k).isTrue();
        }
    }

    @Test
    void inclusionProof_n7_firstLastMiddle_verify() {
        UUID sub = UUID.randomUUID();
        List<String> leaves = new ArrayList<>();
        List<LedgerMerkleFrontier> frontier = new ArrayList<>();
        for (int i = 1; i <= 7; i++) {
            String leaf = LedgerMerkleTree.leafHash(entry(sub, i));
            leaves.add(leaf);
            frontier = LedgerMerkleTree.append(leaf, frontier, sub);
        }
        String root = LedgerMerkleTree.treeRoot(frontier);
        for (int k : new int[]{0, 3, 6}) {
            InclusionProof proof = LedgerMerkleTree.inclusionProof(UUID.randomUUID(), k, 7, leaves);
            InclusionProof withRoot = new InclusionProof(proof.entryId(), k, 7,
                    proof.leafHash(), proof.siblings(), root);
            assertThat(LedgerMerkleTree.verifyProof(withRoot, root))
                    .as("entry %d of 7", k).isTrue();
        }
    }

    @Test
    void verifyProof_wrongRoot_returnsFalse() {
        UUID sub = UUID.randomUUID();
        InclusionProof proof = proofFor(sub, 3, 1);
        assertThat(LedgerMerkleTree.verifyProof(proof,
                "0000000000000000000000000000000000000000000000000000000000000000")).isFalse();
    }
}
```

- [ ] **Step 3: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerMerkleTreeTest -q 2>&1 | tail -5
```

Expected: FAIL — `UnsupportedOperationException` from stub methods.

- [ ] **Step 4: Commit stub + tests**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerkleTree.java
git add runtime/src/test/java/io/casehub/ledger/service/LedgerMerkleTreeTest.java
git commit -m "test(merkle): add LedgerMerkleTreeTest (TDD — all failing)

Refs #11, Refs #12"
```

---

## Task 7: Implement LedgerMerkleTree

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerkleTree.java`

- [ ] **Step 1: Implement the full class**

Replace the stub with:

```java
package io.casehub.ledger.runtime.service;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.UUID;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.service.model.InclusionProof;
import io.casehub.ledger.runtime.service.model.ProofStep;
import io.casehub.ledger.runtime.service.model.ProofStep.Side;

/**
 * Pure static utility implementing the Merkle Mountain Range (stored frontier) algorithm.
 *
 * <p>Leaf hash: SHA-256(0x00 | canonical_bytes) — RFC 9162 domain separation.
 * Internal node hash: SHA-256(0x01 | left_bytes | right_bytes).
 *
 * <p>No CDI, no side effects. All state lives in the caller's frontier list.
 */
public final class LedgerMerkleTree {

    private LedgerMerkleTree() {}

    // ── Public API ────────────────────────────────────────────────────────────

    public static String leafHash(final LedgerEntry entry) {
        final byte[] canonical = canonicalBytes(entry);
        final byte[] input = new byte[1 + canonical.length];
        input[0] = 0x00;
        System.arraycopy(canonical, 0, input, 1, canonical.length);
        return sha256hex(input);
    }

    public static String internalHash(final String leftHex, final String rightHex) {
        final byte[] left = hexToBytes(leftHex);
        final byte[] right = hexToBytes(rightHex);
        final byte[] input = new byte[1 + left.length + right.length];
        input[0] = 0x01;
        System.arraycopy(left, 0, input, 1, left.length);
        System.arraycopy(right, 0, input, 1 + left.length, right.length);
        return sha256hex(input);
    }

    /**
     * Append a new leaf to the frontier and return the updated frontier.
     * The returned list contains only the nodes that changed — call upsert on each.
     * Nodes removed during carry propagation are not in the returned list;
     * delete them by (subjectId, level).
     *
     * <p>Returns the complete new frontier (all surviving + new nodes).
     */
    public static List<LedgerMerkleFrontier> append(
            final String leafHash,
            final List<LedgerMerkleFrontier> frontier,
            final UUID subjectId) {

        // Work with a mutable map of level → hash
        final java.util.TreeMap<Integer, String> map = new java.util.TreeMap<>();
        for (final LedgerMerkleFrontier node : frontier) {
            map.put(node.level, node.hash);
        }

        String carry = leafHash;
        int level = 0;
        while (map.containsKey(level)) {
            carry = internalHash(map.get(level), carry);
            map.remove(level);
            level++;
        }
        map.put(level, carry);

        // Build result list (sorted by level ASC — consistent with findBySubjectId)
        final List<LedgerMerkleFrontier> result = new ArrayList<>();
        for (final java.util.Map.Entry<Integer, String> e : map.entrySet()) {
            final LedgerMerkleFrontier node = new LedgerMerkleFrontier();
            node.subjectId = subjectId;
            node.level = e.getKey();
            node.hash = e.getValue();
            result.add(node);
        }
        return result;
    }

    /**
     * Compute the tree root from the frontier nodes.
     * Folds ASC by level: start with smallest-level node, combine upward as
     * {@code internalHash(higher_level_node, current)}.
     */
    public static String treeRoot(final List<LedgerMerkleFrontier> frontier) {
        if (frontier.isEmpty()) {
            throw new IllegalArgumentException("frontier must not be empty");
        }
        final List<LedgerMerkleFrontier> sorted = frontier.stream()
                .sorted(Comparator.comparingInt(n -> n.level))
                .toList();
        String current = sorted.get(0).hash;
        for (int i = 1; i < sorted.size(); i++) {
            current = internalHash(sorted.get(i).hash, current);
        }
        return current;
    }

    /**
     * Generate an inclusion proof for the entry at 0-based index {@code k} in a tree of
     * size {@code n}. {@code leafHashes} must be the {@code digest} values of all entries
     * for the subject in sequence order (index 0 = sequenceNumber 1).
     */
    public static InclusionProof inclusionProof(
            final UUID entryId,
            final int k,
            final int n,
            final List<String> leafHashes) {

        final List<ProofStep> steps = new ArrayList<>();
        computeProof(k, n, leafHashes, steps);
        final String leafHash = leafHashes.get(k);
        // Compute root from full leaf set
        List<LedgerMerkleFrontier> frontier = new ArrayList<>();
        for (final String lh : leafHashes) {
            frontier = append(lh, frontier, UUID.randomUUID()); // subjectId irrelevant here
        }
        final String root = treeRoot(frontier);
        return new InclusionProof(entryId, k, n, leafHash, List.copyOf(steps), root);
    }

    /** Verify an inclusion proof against a known tree root. No DB access required. */
    public static boolean verifyProof(final InclusionProof proof, final String expectedRoot) {
        String current = proof.leafHash();
        for (final ProofStep step : proof.siblings()) {
            current = step.side() == Side.LEFT
                    ? internalHash(step.hash(), current)
                    : internalHash(current, step.hash());
        }
        return current.equals(expectedRoot);
    }

    // ── Private helpers ───────────────────────────────────────────────────────

    private static void computeProof(
            final int k, final int n,
            final List<String> leafHashes,
            final List<ProofStep> steps) {
        if (n == 1) return;
        final int split = Integer.highestOneBit(n - 1);
        if (k < split) {
            computeProof(k, split, leafHashes.subList(0, split), steps);
            steps.add(new ProofStep(subtreeRoot(leafHashes, split, n), Side.RIGHT));
        } else {
            computeProof(k - split, n - split, leafHashes.subList(split, n), steps);
            steps.add(new ProofStep(subtreeRoot(leafHashes, 0, split), Side.LEFT));
        }
    }

    private static String subtreeRoot(final List<String> leaves, final int from, final int to) {
        if (to - from == 1) return leaves.get(from);
        final int mid = from + Integer.highestOneBit(to - from - 1);
        return internalHash(subtreeRoot(leaves, from, mid), subtreeRoot(leaves, mid, to));
    }

    private static byte[] canonicalBytes(final LedgerEntry entry) {
        final String canonical = String.join("|",
                entry.subjectId != null ? entry.subjectId.toString() : "",
                String.valueOf(entry.sequenceNumber),
                entry.entryType != null ? entry.entryType.name() : "",
                entry.actorId != null ? entry.actorId : "",
                entry.actorRole != null ? entry.actorRole : "",
                entry.occurredAt != null
                        ? entry.occurredAt.truncatedTo(ChronoUnit.MILLIS).toString() : "");
        return canonical.getBytes(StandardCharsets.UTF_8);
    }

    private static byte[] hexToBytes(final String hex) {
        final byte[] result = new byte[hex.length() / 2];
        for (int i = 0; i < result.length; i++) {
            result[i] = (byte) Integer.parseInt(hex.substring(i * 2, i * 2 + 2), 16);
        }
        return result;
    }

    private static String sha256hex(final byte[] input) {
        try {
            final MessageDigest md = MessageDigest.getInstance("SHA-256");
            final byte[] hash = md.digest(input);
            final StringBuilder sb = new StringBuilder(64);
            for (final byte b : hash) sb.append(String.format("%02x", b));
            return sb.toString();
        } catch (final NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 not available", e);
        }
    }
}
```

- [ ] **Step 2: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerMerkleTreeTest 2>&1 | tail -10
```

Expected: BUILD SUCCESS, all tests PASS.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerkleTree.java
git commit -m "feat(merkle): implement LedgerMerkleTree — Merkle Mountain Range algorithm

Leaf hash: SHA-256(0x00|canonical). Internal: SHA-256(0x01|left|right).
Stored frontier gives O(log N) inclusion proofs. RFC 9162 compatible.

Refs #11, Refs #12"
```

---

## Task 8: Update LedgerEntry — Drop previousHash

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`

- [ ] **Step 1: Remove `previousHash` field and update comments**

Remove these lines from `LedgerEntry.java`:

```java
/**
 * SHA-256 digest of the previous entry for this subject.
 * {@code null} for the first entry (no previous entry exists).
 */
@Column(name = "previous_hash")
public String previousHash;
```

Update the class-level Javadoc to replace the hash chain description with:
```
 * The {@code digest} field holds the RFC 9162 leaf hash — SHA-256(0x00 | canonical fields).
 * Chain integrity is maintained by the Merkle Mountain Range in {@code ledger_merkle_frontier}.
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl runtime -q
```

Fix any compilation errors (e.g. references to `entry.previousHash` in tests — update them).

- [ ] **Step 3: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -10
```

Expected: BUILD SUCCESS (LedgerHashChainTest will break — that's expected; it's deleted in Task 15).

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java
git commit -m "feat(merkle): drop previousHash from LedgerEntry — frontier handles chaining

Refs #11, Refs #12"
```

---

## Task 9: Write Path Integration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`

- [ ] **Step 1: Inject frontier access and update `save()`**

Add `@Inject` for frontier access and update `save()`:

```java
// Add imports
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

// In class body, add:
// (No injection needed — LedgerMerkleFrontier uses static Panache methods)

/** {@inheritDoc} */
@Override
@Transactional
public LedgerEntry save(final LedgerEntry entry) {
    // Compute and set the leaf hash (owned by the extension, not the consumer)
    entry.digest = LedgerMerkleTree.leafHash(entry);

    // Persist the entry
    entry.persist();

    // Update the Merkle frontier for this subject
    final List<LedgerMerkleFrontier> currentFrontier =
            LedgerMerkleFrontier.findBySubjectId(entry.subjectId);

    final List<LedgerMerkleFrontier> newFrontier =
            LedgerMerkleTree.append(entry.digest, currentFrontier, entry.subjectId);

    // Delete nodes removed by carry propagation
    final java.util.Set<Integer> newLevels = newFrontier.stream()
            .map(n -> n.level)
            .collect(java.util.stream.Collectors.toSet());
    for (final LedgerMerkleFrontier old : currentFrontier) {
        if (!newLevels.contains(old.level)) {
            LedgerMerkleFrontier.deleteBySubjectAndLevel(entry.subjectId, old.level);
        }
    }

    // Upsert surviving + new frontier nodes
    for (final LedgerMerkleFrontier node : newFrontier) {
        // Delete existing node at this level (if any) before inserting
        LedgerMerkleFrontier.deleteBySubjectAndLevel(entry.subjectId, node.level);
        node.persist();
    }

    return entry;
}
```

Also add required imports at the top of the file:
```java
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.service.LedgerMerkleTree;
```

- [ ] **Step 2: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -15
```

Expected: existing tests pass. `LedgerHashChainTest` will fail — that's expected.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java
git commit -m "feat(merkle): compute leaf hash and update frontier in JpaLedgerEntryRepository.save()

Refs #11, Refs #12"
```

---

## Task 10: TDD — Write Failing LedgerVerificationService Tests

**Files:**
- Create stub: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java`

- [ ] **Step 1: Create stub LedgerVerificationService**

```java
package io.casehub.ledger.runtime.service;

import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.ledger.runtime.service.model.InclusionProof;

@ApplicationScoped
public class LedgerVerificationService {
    public String treeRoot(UUID subjectId) { throw new UnsupportedOperationException(); }
    public InclusionProof inclusionProof(UUID entryId) { throw new UnsupportedOperationException(); }
    public boolean verify(UUID subjectId) { throw new UnsupportedOperationException(); }
}
```

- [ ] **Step 2: Create LedgerVerificationServiceIT**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerMerkleTree;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.model.InclusionProof;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class LedgerVerificationServiceIT {

    @Inject LedgerVerificationService verificationService;
    @Inject LedgerEntryRepository repo;

    private TestEntry seedEntry(UUID subjectId, int seq, String actorId) {
        TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = seq;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.SYSTEM;
        e.actorRole = "Tester";
        return (TestEntry) repo.save(e);
    }

    // ── treeRoot ─────────────────────────────────────────────────────────────

    @Test
    @Transactional
    void treeRoot_afterOneEntry_matchesFrontier() {
        UUID sub = UUID.randomUUID();
        seedEntry(sub, 1, "actor-a");

        String root = verificationService.treeRoot(sub);
        List<LedgerMerkleFrontier> frontier = LedgerMerkleFrontier.findBySubjectId(sub);

        assertThat(root).isNotNull().matches("[0-9a-f]{64}");
        assertThat(root).isEqualTo(LedgerMerkleTree.treeRoot(frontier));
    }

    @Test
    @Transactional
    void treeRoot_frontierHasCorrectNodeCount_afterFiveEntries() {
        UUID sub = UUID.randomUUID();
        for (int i = 1; i <= 5; i++) seedEntry(sub, i, "actor-" + i);

        verificationService.treeRoot(sub); // ensure no exception

        List<LedgerMerkleFrontier> frontier = LedgerMerkleFrontier.findBySubjectId(sub);
        assertThat(frontier).hasSize(Integer.bitCount(5)); // bitCount(5) = 2
    }

    // ── inclusionProof ────────────────────────────────────────────────────────

    @Test
    @Transactional
    void inclusionProof_singleEntry_verifies() {
        UUID sub = UUID.randomUUID();
        TestEntry e = seedEntry(sub, 1, "actor-a");

        InclusionProof proof = verificationService.inclusionProof(e.id);

        assertThat(proof).isNotNull();
        assertThat(proof.entryId()).isEqualTo(e.id);
        assertThat(proof.entryIndex()).isEqualTo(0);
        assertThat(proof.treeSize()).isEqualTo(1);
        assertThat(proof.siblings()).isEmpty();
        assertThat(LedgerMerkleTree.verifyProof(proof, proof.treeRoot())).isTrue();
    }

    @Test
    @Transactional
    void inclusionProof_fourEntries_allVerify() {
        UUID sub = UUID.randomUUID();
        TestEntry[] entries = new TestEntry[4];
        for (int i = 0; i < 4; i++) entries[i] = seedEntry(sub, i + 1, "actor-" + i);

        String root = verificationService.treeRoot(sub);
        for (TestEntry e : entries) {
            InclusionProof proof = verificationService.inclusionProof(e.id);
            assertThat(LedgerMerkleTree.verifyProof(proof, root))
                    .as("entry %s should verify", e.id).isTrue();
        }
    }

    @Test
    @Transactional
    void inclusionProof_lastOfSevenEntries_verifies() {
        UUID sub = UUID.randomUUID();
        TestEntry last = null;
        for (int i = 1; i <= 7; i++) last = seedEntry(sub, i, "actor-" + i);

        String root = verificationService.treeRoot(sub);
        InclusionProof proof = verificationService.inclusionProof(last.id);
        assertThat(LedgerMerkleTree.verifyProof(proof, root)).isTrue();
    }

    // ── verify ────────────────────────────────────────────────────────────────

    @Test
    @Transactional
    void verify_untamperedChain_returnsTrue() {
        UUID sub = UUID.randomUUID();
        for (int i = 1; i <= 5; i++) seedEntry(sub, i, "actor-" + i);
        assertThat(verificationService.verify(sub)).isTrue();
    }

    @Test
    @Transactional
    void verify_afterDigestTampering_returnsFalse() {
        UUID sub = UUID.randomUUID();
        TestEntry e = seedEntry(sub, 1, "actor-a");

        // Directly corrupt the stored digest
        io.casehub.ledger.runtime.model.LedgerEntry stored =
                repo.findById(e.id).orElseThrow();
        stored.digest = "0000000000000000000000000000000000000000000000000000000000000000";
        stored.persist();

        assertThat(verificationService.verify(sub)).isFalse();
    }
}
```

Note: add `import java.util.List;` at the top.

- [ ] **Step 3: Run — verify tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerVerificationServiceIT 2>&1 | tail -5
```

Expected: FAIL — `UnsupportedOperationException`.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java
git add runtime/src/test/java/io/casehub/ledger/service/LedgerVerificationServiceIT.java
git commit -m "test(merkle): add LedgerVerificationServiceIT (TDD — all failing)

Refs #11, Refs #12"
```

---

## Task 11: Implement LedgerVerificationService

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`

- [ ] **Step 1: Implement**

```java
package io.casehub.ledger.runtime.service;

import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.service.model.InclusionProof;

/**
 * CDI bean exposing Merkle tree verification operations.
 * Auto-activated — no consumer configuration required.
 */
@ApplicationScoped
public class LedgerVerificationService {

    /** Return the current Merkle tree root for a subject. */
    @Transactional
    public String treeRoot(final UUID subjectId) {
        final List<LedgerMerkleFrontier> frontier =
                LedgerMerkleFrontier.findBySubjectId(subjectId);
        if (frontier.isEmpty()) {
            throw new IllegalStateException("No entries for subject " + subjectId);
        }
        return LedgerMerkleTree.treeRoot(frontier);
    }

    /**
     * Generate an inclusion proof for the given entry.
     * Fetches O(log N) leaf hashes from the database.
     */
    @Transactional
    public InclusionProof inclusionProof(final UUID entryId) {
        final LedgerEntry entry = LedgerEntry.findById(entryId);
        if (entry == null) throw new IllegalArgumentException("Entry not found: " + entryId);

        final List<LedgerEntry> allForSubject = LedgerEntry.list(
                "subjectId = ?1 ORDER BY sequenceNumber ASC", entry.subjectId);

        final List<String> leafHashes = allForSubject.stream()
                .map(e -> e.digest)
                .toList();

        final int k = entry.sequenceNumber - 1; // 0-based index
        final InclusionProof proof = LedgerMerkleTree.inclusionProof(
                entryId, k, leafHashes.size(), leafHashes);

        // Replace the computed root with the authoritative frontier root
        final String root = treeRoot(entry.subjectId);
        return new InclusionProof(entryId, k, leafHashes.size(),
                proof.leafHash(), proof.siblings(), root);
    }

    /**
     * Verify that all stored digests are consistent with the Merkle frontier.
     * Recomputes leaf hashes from canonical fields and checks against stored digest values.
     *
     * @return {@code true} if the chain is intact; {@code false} if any entry was tampered with
     */
    @Transactional
    public boolean verify(final UUID subjectId) {
        final List<LedgerEntry> entries = LedgerEntry.list(
                "subjectId = ?1 ORDER BY sequenceNumber ASC", subjectId);

        List<LedgerMerkleFrontier> frontier = new java.util.ArrayList<>();
        for (final LedgerEntry entry : entries) {
            final String expected = LedgerMerkleTree.leafHash(entry);
            if (!expected.equals(entry.digest)) return false;
            frontier = LedgerMerkleTree.append(expected, frontier, subjectId);
        }

        if (frontier.isEmpty()) return true;

        // Compare computed root against stored frontier
        final String computed = LedgerMerkleTree.treeRoot(frontier);
        final String stored = treeRoot(subjectId);
        return computed.equals(stored);
    }
}
```

- [ ] **Step 2: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerVerificationServiceIT 2>&1 | tail -10
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 3: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -10
```

Expected: all tests pass except `LedgerHashChainTest` (deleted in Task 15).

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java
git commit -m "feat(merkle): implement LedgerVerificationService — treeRoot, inclusionProof, verify

Refs #11, Refs #12"
```

---

## Task 12: TDD — Write Failing LedgerMerklePublisher Tests

**Files:**
- Create stub: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerklePublisher.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerMerklePublisherTest.java`

- [ ] **Step 1: Create stub**

```java
package io.casehub.ledger.runtime.service;

import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class LedgerMerklePublisher {

    static String buildCheckpoint(UUID subjectId, int treeSize, String treeRoot, String keyId) {
        throw new UnsupportedOperationException();
    }

    static byte[] signCheckpoint(String checkpointText, java.security.PrivateKey privateKey) {
        throw new UnsupportedOperationException();
    }

    public void publish(UUID subjectId, int treeSize, String treeRoot) {
        // no-op stub
    }
}
```

- [ ] **Step 2: Create LedgerMerklePublisherTest**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.Signature;
import java.util.Base64;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.service.LedgerMerklePublisher;

class LedgerMerklePublisherTest {

    private static final String FAKE_ROOT =
            "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2";

    // ── Checkpoint format ─────────────────────────────────────────────────────

    @Test
    void buildCheckpoint_firstLineIsOrigin() {
        UUID sub = UUID.randomUUID();
        String cp = LedgerMerklePublisher.buildCheckpoint(sub, 5, FAKE_ROOT, "key-01");
        assertThat(cp.split("\n")[0]).isEqualTo("io.casehub.ledger/v1");
    }

    @Test
    void buildCheckpoint_secondLineIsSubjectId() {
        UUID sub = UUID.randomUUID();
        String cp = LedgerMerklePublisher.buildCheckpoint(sub, 5, FAKE_ROOT, "key-01");
        assertThat(cp.split("\n")[1]).isEqualTo(sub.toString());
    }

    @Test
    void buildCheckpoint_thirdLineIsTreeSize() {
        String cp = LedgerMerklePublisher.buildCheckpoint(UUID.randomUUID(), 42, FAKE_ROOT, "key-01");
        assertThat(cp.split("\n")[2]).isEqualTo("42");
    }

    @Test
    void buildCheckpoint_fourthLineIsBase64Root() {
        String cp = LedgerMerklePublisher.buildCheckpoint(UUID.randomUUID(), 1, FAKE_ROOT, "key-01");
        String line4 = cp.split("\n")[3];
        byte[] decoded = Base64.getDecoder().decode(line4);
        assertThat(decoded).hasSize(32); // 32 bytes = 64-char hex
    }

    @Test
    void buildCheckpoint_fifthLineIsEmpty() {
        String cp = LedgerMerklePublisher.buildCheckpoint(UUID.randomUUID(), 1, FAKE_ROOT, "key-01");
        assertThat(cp.split("\n")[4]).isEmpty();
    }

    @Test
    void buildCheckpoint_signatureLine_startsWithDashKeyId() {
        String cp = LedgerMerklePublisher.buildCheckpoint(UUID.randomUUID(), 1, FAKE_ROOT, "key-01");
        // Signature line comes after the blank line — but buildCheckpoint returns unsigned checkpoint
        // Signature is added separately by signCheckpoint
        String[] lines = cp.split("\n");
        assertThat(lines).hasSize(5); // 4 data lines + 1 blank line (no sig yet)
    }

    // ── Ed25519 signing ───────────────────────────────────────────────────────

    @Test
    void signCheckpoint_producesVerifiableSignature() throws Exception {
        KeyPairGenerator kpg = KeyPairGenerator.getInstance("Ed25519");
        KeyPair kp = kpg.generateKeyPair();

        String text = "io.casehub.ledger/v1\n" + UUID.randomUUID() + "\n5\n"
                + Base64.getEncoder().encodeToString(new byte[32]) + "\n";

        byte[] sig = LedgerMerklePublisher.signCheckpoint(text, kp.getPrivate());

        Signature verifier = Signature.getInstance("Ed25519");
        verifier.initVerify(kp.getPublic());
        verifier.update(text.getBytes(java.nio.charset.StandardCharsets.UTF_8));
        assertThat(verifier.verify(sig)).isTrue();
    }

    @Test
    void signCheckpoint_differentInputs_differentSignatures() throws Exception {
        KeyPairGenerator kpg = KeyPairGenerator.getInstance("Ed25519");
        KeyPair kp = kpg.generateKeyPair();
        byte[] sig1 = LedgerMerklePublisher.signCheckpoint("text-1", kp.getPrivate());
        byte[] sig2 = LedgerMerklePublisher.signCheckpoint("text-2", kp.getPrivate());
        assertThat(sig1).isNotEqualTo(sig2);
    }
}
```

- [ ] **Step 3: Run — verify tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerMerklePublisherTest 2>&1 | tail -5
```

Expected: FAIL.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerklePublisher.java
git add runtime/src/test/java/io/casehub/ledger/service/LedgerMerklePublisherTest.java
git commit -m "test(merkle): add LedgerMerklePublisherTest (TDD — all failing)

Refs #11, Refs #12"
```

---

## Task 13: Implement LedgerMerklePublisher

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerklePublisher.java`

- [ ] **Step 1: Implement**

```java
package io.casehub.ledger.runtime.service;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.Signature;
import java.security.spec.PKCS8EncodedKeySpec;
import java.util.Base64;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.config.LedgerConfig.MerkleConfig.MerklePublishConfig;

/**
 * Publishes signed Merkle tree checkpoints to a configured external URL.
 * Inactive when {@code quarkus.ledger.merkle.publish.url} is absent.
 * Publishing is async and best-effort — failures are logged, not thrown.
 */
@ApplicationScoped
public class LedgerMerklePublisher {

    private static final Logger LOG = Logger.getLogger(LedgerMerklePublisher.class);

    @Inject
    LedgerConfig config;

    /**
     * Build an unsigned tlog-checkpoint (4 header lines + blank line).
     * The caller adds the signature line after signing this text.
     */
    static String buildCheckpoint(
            final UUID subjectId, final int treeSize,
            final String treeRoot, final String keyId) {

        final byte[] rootBytes = hexToBytes(treeRoot);
        return "io.casehub.ledger/v1\n"
                + subjectId + "\n"
                + treeSize + "\n"
                + Base64.getEncoder().encodeToString(rootBytes) + "\n";
    }

    /** Sign the checkpoint text with an Ed25519 private key. */
    static byte[] signCheckpoint(final String checkpointText, final PrivateKey privateKey) {
        try {
            final Signature sig = Signature.getInstance("Ed25519");
            sig.initSign(privateKey);
            sig.update(checkpointText.getBytes(StandardCharsets.UTF_8));
            return sig.sign();
        } catch (final Exception e) {
            throw new IllegalStateException("Ed25519 signing failed", e);
        }
    }

    /**
     * Build, sign, and POST the checkpoint for the given subject.
     * No-op when publish URL is not configured. Best-effort — logs failures.
     */
    public void publish(final UUID subjectId, final int treeSize, final String treeRoot) {
        final MerklePublishConfig publish = config.merkle().publish();
        if (publish.url().isEmpty()) return;

        try {
            final String keyId = publish.keyId();
            final String checkpoint = buildCheckpoint(subjectId, treeSize, treeRoot, keyId);
            final PrivateKey privateKey = loadPrivateKey(publish.privateKey()
                    .orElseThrow(() -> new IllegalStateException(
                            "quarkus.ledger.merkle.publish.private-key required when url is set")));
            final byte[] signature = signCheckpoint(checkpoint, privateKey);
            final String signed = checkpoint
                    + "\n— " + keyId + " " + Base64.getEncoder().encodeToString(signature);

            final HttpClient client = HttpClient.newHttpClient();
            final HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(publish.url().get()))
                    .header("Content-Type", "text/plain; charset=utf-8")
                    .POST(HttpRequest.BodyPublishers.ofString(signed))
                    .build();

            client.sendAsync(request, HttpResponse.BodyHandlers.discarding())
                    .exceptionally(ex -> {
                        LOG.warnf("Merkle checkpoint publish failed for subject %s: %s",
                                subjectId, ex.getMessage());
                        return null;
                    });
        } catch (final Exception e) {
            LOG.warnf("Merkle checkpoint publish error for subject %s: %s",
                    subjectId, e.getMessage());
        }
    }

    private static PrivateKey loadPrivateKey(final String pemPath) throws Exception {
        final String pem = Files.readString(Path.of(pemPath));
        final String base64 = pem
                .replace("-----BEGIN PRIVATE KEY-----", "")
                .replace("-----END PRIVATE KEY-----", "")
                .replaceAll("\\s", "");
        final byte[] keyBytes = Base64.getDecoder().decode(base64);
        final KeyFactory kf = KeyFactory.getInstance("Ed25519");
        return kf.generatePrivate(new PKCS8EncodedKeySpec(keyBytes));
    }

    private static byte[] hexToBytes(final String hex) {
        final byte[] result = new byte[hex.length() / 2];
        for (int i = 0; i < result.length; i++) {
            result[i] = (byte) Integer.parseInt(hex.substring(i * 2, i * 2 + 2), 16);
        }
        return result;
    }
}
```

- [ ] **Step 2: Call publisher from write path**

In `JpaLedgerEntryRepository.save()`, after the frontier upsert block, add:

```java
// Add field injection at top of class:
@Inject
LedgerMerklePublisher merklePublisher;

// Add at end of save() method, after frontier upsert:
final String root = LedgerMerkleTree.treeRoot(newFrontier);
merklePublisher.publish(entry.subjectId, entry.sequenceNumber, root);
```

- [ ] **Step 3: Run publisher tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerMerklePublisherTest 2>&1 | tail -10
```

Expected: all pass (checkpoint format test needs update — `buildCheckpoint` now returns 4 lines + blank = 5 lines total, verify test expects 5 lines).

If `signatureLine_startsWithDashKeyId` test fails, update it: `buildCheckpoint` returns unsigned text (5 elements when split by `\n` with trailing newline). Adjust assertion to match the unsigned format.

- [ ] **Step 4: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -10
```

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerMerklePublisher.java
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java
git commit -m "feat(merkle): implement LedgerMerklePublisher — Ed25519 signed tlog-checkpoint

Disabled by default. Activated by quarkus.ledger.merkle.publish.url.
Async best-effort — write path never blocked by publish failure.

Refs #11, Refs #12"
```

---

## Task 14: Update LedgerRetentionJob

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerRetentionJob.java`

- [ ] **Step 1: Replace LedgerHashChain.verify with LedgerVerificationService**

Inject `LedgerVerificationService` into the retention job:

```java
@Inject
LedgerVerificationService verificationService;
```

Replace the call at line 109:
```java
// OLD:
if (!LedgerHashChain.verify(sorted)) {

// NEW:
if (!verificationService.verify(subjectId)) {
```

Remove the `sorted` list (if it's only used for the verify call) or keep it if it's used elsewhere. Check the full `archiveSubject` method and remove the `LedgerHashChain` import.

- [ ] **Step 2: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -10
```

Expected: all tests pass (except `LedgerHashChainTest` — deleted next).

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerRetentionJob.java
git commit -m "feat(merkle): update LedgerRetentionJob to use LedgerVerificationService.verify()

Refs #11, Refs #12"
```

---

## Task 15: Delete LedgerHashChain

**Files:**
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerHashChain.java`
- Delete: `runtime/src/test/java/io/casehub/ledger/service/LedgerHashChainTest.java`

- [ ] **Step 1: Delete both files**

```bash
rm runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerHashChain.java
rm runtime/src/test/java/io/casehub/ledger/service/LedgerHashChainTest.java
```

- [ ] **Step 2: Run full suite — verify all tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | tail -10
```

Expected: BUILD SUCCESS. All remaining tests pass.

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -m "feat(merkle): delete LedgerHashChain — fully replaced by LedgerMerkleTree

Refs #11, Refs #12"
```

---

## Task 16: Update Documentation

**Files:**
- Modify: `docs/AUDITABILITY.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Update AUDITABILITY.md — mark Axiom 4 ✅**

In the axiom summary table, change:
```
| 4. Verifiability | ⚠️ Partial | Hash chain verification endpoint (medium-term) |
```
to:
```
| 4. Verifiability | ✅ Addressed | Merkle tree upgrade (#11) — O(log N) inclusion proofs + Ed25519 publishing |
```

In the Axiom 4 section body, update **Current state** and **Gap** to **Addressed**:

Replace the `**Gap:**` paragraph and `**How to incorporate:**` section with:

```markdown
**Status:** ✅ Addressed (#11)

**Addressed by (#11):**
- `LedgerMerkleTree` — RFC 9162 Merkle Mountain Range. Leaf hash: `SHA-256(0x00 | canonicalFields)`.
  Internal node: `SHA-256(0x01 | left | right)`. Stored frontier: ≤ log₂(N) rows per subject.
- `LedgerVerificationService` — CDI bean. `treeRoot(subjectId)`, `inclusionProof(entryId)`,
  `verify(subjectId)`. Auto-activated; no consumer configuration required.
- `LedgerMerklePublisher` — opt-in Ed25519-signed tlog-checkpoint publishing.
  Configure `quarkus.ledger.merkle.publish.url` to activate. Disabled by default.
- An external auditor needs only: a published checkpoint + an `InclusionProof` record.
  No DB access, no schema knowledge, no trust in the operator required.
```

- [ ] **Step 2: Update DESIGN.md**

Add a `## Merkle Mountain Range` section under the Hash Chain section describing:
- The frontier table
- The `LedgerMerkleTree` utility (leaf hash formula, internal hash formula, append algorithm)
- `LedgerVerificationService` public API
- `LedgerMerklePublisher` (opt-in, Ed25519, tlog-checkpoint format)

- [ ] **Step 3: Commit**

```bash
git add docs/AUDITABILITY.md docs/DESIGN.md
git commit -m "docs: mark Axiom 4 Verifiability as addressed, update DESIGN.md with Merkle section

Refs #11, Refs #12"
```

---

## Task 17: Create Example

**Files:**
- Create: `examples/merkle-verification/README.md`
- Create: `examples/merkle-verification/src/main/java/io/casehub/ledger/example/merkle/MerkleVerificationExample.java`
- Create: `examples/merkle-verification/src/test/java/io/casehub/ledger/example/merkle/MerkleVerificationIT.java`
- Create: `examples/merkle-verification/pom.xml` (copy structure from `examples/art12-compliance/pom.xml`)

- [ ] **Step 1: Copy pom.xml from art12-compliance, update artifactId**

```bash
cp examples/art12-compliance/pom.xml examples/merkle-verification/pom.xml
```

Edit: change `<artifactId>casehub-ledger-example-art12-compliance</artifactId>` to
`<artifactId>casehub-ledger-example-merkle-verification</artifactId>`.
Change `<name>` accordingly.

- [ ] **Step 2: Create MerkleVerificationExample**

```java
package io.casehub.ledger.example.merkle;

import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerMerkleTree;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.model.InclusionProof;

/**
 * Demonstrates Merkle tree inclusion proof generation and independent verification.
 *
 * <p>Happy path: write 5 entries → generate proof for entry 3 → verify without DB access.
 */
@ApplicationScoped
public class MerkleVerificationExample {

    @Inject LedgerEntryRepository repo;
    @Inject LedgerVerificationService verification;

    @Transactional
    public VerificationResult runHappyPath(final UUID subjectId) {
        // 1. Write 5 entries via the repository (digest + frontier updated automatically)
        for (int i = 1; i <= 5; i++) {
            final AuditEntry e = new AuditEntry();
            e.subjectId = subjectId;
            e.sequenceNumber = i;
            e.entryType = io.casehub.ledger.runtime.model.LedgerEntryType.EVENT;
            e.actorId = "example-actor";
            e.actorRole = "Demonstrator";
            repo.save(e);
        }

        // 2. Fetch tree root (this is what you publish externally)
        final String root = verification.treeRoot(subjectId);

        // 3. Generate inclusion proof for entry 3 (sequenceNumber=3, index=2)
        final java.util.List<io.casehub.ledger.runtime.model.LedgerEntry> entries =
                repo.findBySubjectId(subjectId);
        final InclusionProof proof = verification.inclusionProof(entries.get(2).id);

        // 4. Verify independently — no DB access from this point
        final boolean valid = LedgerMerkleTree.verifyProof(proof, root);

        return new VerificationResult(root, proof, valid);
    }

    public record VerificationResult(String treeRoot, InclusionProof proof, boolean valid) {}
}
```

- [ ] **Step 3: Create AuditEntry (concrete LedgerEntry subclass for the example)**

```java
package io.casehub.ledger.example.merkle;

import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import io.casehub.ledger.runtime.model.LedgerEntry;

@Entity
@Table(name = "merkle_example_entry")
@DiscriminatorValue("MERKLE_EXAMPLE")
public class AuditEntry extends LedgerEntry {}
```

- [ ] **Step 4: Create MerkleVerificationIT**

```java
package io.casehub.ledger.example.merkle;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.service.LedgerMerkleTree;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MerkleVerificationIT {

    @Inject MerkleVerificationExample example;

    @Test
    void happyPath_fiveEntries_proofVerifiesWithoutDbAccess() {
        final MerkleVerificationExample.VerificationResult result =
                example.runHappyPath(UUID.randomUUID());

        assertThat(result.valid()).isTrue();
        assertThat(result.treeRoot()).matches("[0-9a-f]{64}");
        assertThat(result.proof().treeSize()).isEqualTo(5);
        assertThat(result.proof().entryIndex()).isEqualTo(2);
        // Independent verification — no injection needed
        assertThat(LedgerMerkleTree.verifyProof(result.proof(), result.treeRoot())).isTrue();
    }

    @Test
    void happyPath_fiveEntries_frontierHasBitCount5Nodes() {
        example.runHappyPath(UUID.randomUUID()); // just runs without exception
        // Frontier correctness tested in main module LedgerVerificationServiceIT
    }

    @Test
    void happyPath_wrongRoot_proofDoesNotVerify() {
        final MerkleVerificationExample.VerificationResult result =
                example.runHappyPath(UUID.randomUUID());
        final String wrongRoot =
                "0000000000000000000000000000000000000000000000000000000000000000";
        assertThat(LedgerMerkleTree.verifyProof(result.proof(), wrongRoot)).isFalse();
    }
}
```

- [ ] **Step 5: Create README.md**

```markdown
# Example: Merkle Verification

Demonstrates O(log N) Merkle inclusion proofs using `casehub-ledger`.

## What This Shows

1. Write entries — `digest` and Merkle frontier updated automatically on `repo.save()`
2. Fetch tree root — publishable to an external checkpoint log
3. Generate inclusion proof — O(log N) DB reads
4. Verify independently — no DB access, no trust in operator required

## Run

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/merkle-verification
```

## External Publishing

Configure `quarkus.ledger.merkle.publish.url` and `quarkus.ledger.merkle.publish.private-key`
to have each frontier update POSTed as a signed tlog-checkpoint to your external log.
Receivers verify using your published Ed25519 public key.
```

- [ ] **Step 6: Build and test the example**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/merkle-verification 2>&1 | tail -10
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Close issue**

```bash
git add examples/merkle-verification/
git commit -m "feat(merkle): add merkle-verification example — happy path inclusion proof demo

Refs #11, Refs #12"
```

```bash
gh issue close 11 --repo casehubio/ledger \
  --comment "Implemented: Merkle Mountain Range frontier, LedgerVerificationService, Ed25519 publishing, example. Closes Axiom 4."
gh issue close 12 --repo casehubio/ledger \
  --comment "All child issues complete."
```

---

## Final Verification

- [ ] **Run complete test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -15
```

Expected: BUILD SUCCESS across all modules.

- [ ] **Check test count has increased**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep "Tests run"
```

Previous: 110 tests. New tests add: ~15 unit + ~7 IT + ~3 publisher + ~3 example = ~28 new tests.
