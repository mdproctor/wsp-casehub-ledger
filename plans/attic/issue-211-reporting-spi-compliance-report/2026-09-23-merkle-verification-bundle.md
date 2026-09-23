# Merkle Verification Bundle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #212 — feat: Merkle verification bundle — offline tamper-evidence package
**Issue group:** #211, #212

**Goal:** Generate self-contained offline verification bundles so auditors can independently verify Merkle chain integrity without system access.

**Architecture:** Service in `runtime` queries entry digests and MMR frontier per subject, packages them with an embedded Python verification script and instructions. Models in `ledger-core`. Chain-only verification — digests are taken as given, the script verifies MMR structure.

**Tech Stack:** Java 21, Python 3.8+ (embedded script, no external deps), SHA-256, RFC 9162 domain separation

## Global Constraints

- Leaf hash: `SHA-256(0x00 | canonicalBytes)` — precomputed, stored as `LedgerEntry.digest`
- Internal hash: `SHA-256(0x01 | left_bytes | right_bytes)` — RFC 9162
- Tree root: fold frontier ASC by level using `internalHash(higher_level, current)`
- All tenant-scoped methods place `tenancyId` as the last parameter
- Python script must use only standard library (hashlib, json, sys) — zero external deps

---

## Batch 1: Models, Service, Script, and Tests

### Task 1: Create verification bundle models and service

**Files:**
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/VerificationBundle.java`
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/SubjectChain.java`
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/ChainEntry.java`
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/compliance/FrontierNode.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/MerkleVerificationBundleService.java`
- Create: `runtime/src/main/resources/verification/verify.py`
- Create: `runtime/src/main/resources/verification/VERIFY-README.md`
- Test: `runtime/src/test/java/io/casehub/ledger/service/MerkleVerificationBundleServiceIT.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository.findDistinctSubjectIds(String tenancyId)` → `List<UUID>`, `LedgerEntryRepository.findBySubjectId(UUID subjectId, String tenancyId)` → `List<LedgerEntry>`, `LedgerMerkleFrontierRepository.findBySubjectId(UUID subjectId, String tenancyId)` → `List<LedgerMerkleFrontier>`, `LedgerVerificationService.treeRoot(UUID subjectId, String tenancyId)` → `String`
- Produces: `MerkleVerificationBundleService.generateBundle(String tenancyId)` → `VerificationBundle`

- [ ] **Step 1: Write failing integration test**

Create `runtime/src/test/java/io/casehub/ledger/service/MerkleVerificationBundleServiceIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.core.compliance.VerificationBundle;
import io.casehub.ledger.runtime.service.MerkleVerificationBundleService;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class MerkleVerificationBundleServiceIT {

    @Inject MerkleVerificationBundleService bundleService;
    @Inject LedgerEntryRepository repo;

    @Test
    @Transactional
    void generateBundle_twoSubjects_bundleContainsBoth() {
        final UUID subject1 = UUID.randomUUID();
        final UUID subject2 = UUID.randomUUID();
        final String tenancyId = "bundle-two-" + UUID.randomUUID();

        repo.save(entry(subject1, "actor-1"), tenancyId);
        repo.save(entry(subject2, "actor-2"), tenancyId);

        final VerificationBundle bundle = bundleService.generateBundle(tenancyId);

        assertThat(bundle.tenancyId()).isEqualTo(tenancyId);
        assertThat(bundle.generatedAt()).isNotNull();
        assertThat(bundle.chains()).hasSize(2);
        assertThat(bundle.chains()).allMatch(c -> !c.entries().isEmpty());
        assertThat(bundle.chains()).allMatch(c -> c.storedRoot() != null);
        assertThat(bundle.verificationScript()).contains("def verify_chain");
        assertThat(bundle.verificationInstructions()).contains("python3");
    }

    @Test
    @Transactional
    void generateBundle_emptyTenancy_emptyChains() {
        final String tenancyId = "bundle-empty-" + UUID.randomUUID();

        final VerificationBundle bundle = bundleService.generateBundle(tenancyId);

        assertThat(bundle.chains()).isEmpty();
        assertThat(bundle.verificationScript()).isNotEmpty();
    }

    @Test
    @Transactional
    void generateBundle_chainEntriesMatchDigests() {
        final UUID subject = UUID.randomUUID();
        final String tenancyId = "bundle-digest-" + UUID.randomUUID();

        repo.save(entry(subject, "actor-d1"), tenancyId);
        repo.save(entry(subject, "actor-d2"), tenancyId);

        final VerificationBundle bundle = bundleService.generateBundle(tenancyId);

        assertThat(bundle.chains()).hasSize(1);
        assertThat(bundle.chains().get(0).entries()).hasSize(2);
        assertThat(bundle.chains().get(0).entries()).allMatch(e -> e.digest() != null);
        assertThat(bundle.chains().get(0).entries().get(0).sequenceNumber())
                .isLessThan(bundle.chains().get(0).entries().get(1).sequenceNumber());
    }

    @Test
    @Transactional
    void generateBundle_frontierNodesPresent() {
        final UUID subject = UUID.randomUUID();
        final String tenancyId = "bundle-frontier-" + UUID.randomUUID();

        repo.save(entry(subject, "actor-f1"), tenancyId);

        final VerificationBundle bundle = bundleService.generateBundle(tenancyId);

        assertThat(bundle.chains().get(0).frontier()).isNotEmpty();
        assertThat(bundle.chains().get(0).frontier()).allMatch(f -> f.hash() != null);
    }

    private static TestEntry entry(final UUID subjectId, final String actorId) {
        final TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.actorRole = "Verifier";
        e.occurredAt = Instant.now();
        return e;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MerkleVerificationBundleServiceIT -f pom.xml`
Expected: Compilation error — `VerificationBundle` and `MerkleVerificationBundleService` don't exist

- [ ] **Step 3: Create model records in `ledger-core`**

Create four files using `ide_create_file`:

`ledger-core/src/main/java/io/casehub/ledger/core/compliance/VerificationBundle.java`:
```java
package io.casehub.ledger.core.compliance;

import java.time.Instant;
import java.util.List;

public record VerificationBundle(
        String tenancyId,
        Instant generatedAt,
        List<SubjectChain> chains,
        String verificationScript,
        String verificationInstructions) {
}
```

`ledger-core/src/main/java/io/casehub/ledger/core/compliance/SubjectChain.java`:
```java
package io.casehub.ledger.core.compliance;

import java.util.List;
import java.util.UUID;

public record SubjectChain(
        UUID subjectId,
        List<ChainEntry> entries,
        List<FrontierNode> frontier,
        String storedRoot) {
}
```

`ledger-core/src/main/java/io/casehub/ledger/core/compliance/ChainEntry.java`:
```java
package io.casehub.ledger.core.compliance;

public record ChainEntry(
        int sequenceNumber,
        String digest) {
}
```

`ledger-core/src/main/java/io/casehub/ledger/core/compliance/FrontierNode.java`:
```java
package io.casehub.ledger.core.compliance;

public record FrontierNode(
        int level,
        String hash) {
}
```

- [ ] **Step 4: Create the Python verification script**

Create `runtime/src/main/resources/verification/verify.py` — the complete script from the spec (§Part 3). Uses only `hashlib`, `json`, `sys`. Implements `internal_hash`, `append_to_frontier`, `tree_root`, `verify_chain`, `main`.

```python
#!/usr/bin/env python3
"""Offline Merkle Mountain Range verification for casehub-ledger bundles."""
import hashlib
import json
import sys

def internal_hash(left_hex: str, right_hex: str) -> str:
    left = bytes.fromhex(left_hex)
    right = bytes.fromhex(right_hex)
    data = b'\x01' + left + right
    return hashlib.sha256(data).hexdigest()

def append_to_frontier(digest: str, frontier: dict[int, str]) -> dict[int, str]:
    carry = digest
    level = 0
    while level in frontier:
        carry = internal_hash(frontier.pop(level), carry)
        level += 1
    frontier[level] = carry
    return frontier

def tree_root(frontier: dict[int, str]) -> str:
    if not frontier:
        raise ValueError("empty frontier")
    sorted_levels = sorted(frontier.keys())
    current = frontier[sorted_levels[0]]
    for lvl in sorted_levels[1:]:
        current = internal_hash(frontier[lvl], current)
    return current

def verify_chain(chain: dict) -> tuple[bool, str]:
    subject_id = chain["subjectId"]
    entries = sorted(chain["entries"], key=lambda e: e["sequenceNumber"])
    stored_root = chain.get("storedRoot")
    if not entries:
        return True, f"{subject_id}: SKIP (no entries)"
    if stored_root is None:
        return False, f"{subject_id}: FAIL (no stored root)"
    frontier = {}
    for entry in entries:
        frontier = append_to_frontier(entry["digest"], frontier)
    computed = tree_root(frontier)
    if computed == stored_root:
        return True, f"{subject_id}: PASS ({len(entries)} entries)"
    return False, f"{subject_id}: FAIL (root mismatch: computed={computed[:16]}... stored={stored_root[:16]}...)"

def main():
    if len(sys.argv) != 2:
        print(f"Usage: {sys.argv[0]} <bundle.json>")
        sys.exit(1)
    with open(sys.argv[1]) as f:
        bundle = json.load(f)
    print(f"Verifying tenancy: {bundle['tenancyId']}")
    print(f"Generated: {bundle['generatedAt']}")
    print(f"Subjects: {len(bundle['chains'])}")
    print()
    passed = 0
    failed = 0
    for chain in bundle["chains"]:
        ok, msg = verify_chain(chain)
        print(f"  {msg}")
        if ok:
            passed += 1
        else:
            failed += 1
    print()
    print(f"Result: {passed} passed, {failed} failed")
    sys.exit(0 if failed == 0 else 1)

if __name__ == "__main__":
    main()
```

- [ ] **Step 5: Create verification instructions**

Create `runtime/src/main/resources/verification/VERIFY-README.md` — the instructions from the spec (§Part 4).

- [ ] **Step 6: Create `MerkleVerificationBundleService`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/MerkleVerificationBundleService.java` — the service implementation from the spec (§Part 2). Injects `LedgerEntryRepository`, `LedgerMerkleFrontierRepository`, `LedgerVerificationService`. Uses `findDistinctSubjectIds()`, `findBySubjectId()`, `frontierRepo.findBySubjectId()`, `verificationService.treeRoot()`. Loads `verify.py` and `VERIFY-README.md` from classpath.

```java
package io.casehub.ledger.runtime.service;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.model.LedgerMerkleFrontier;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.core.compliance.ChainEntry;
import io.casehub.ledger.core.compliance.FrontierNode;
import io.casehub.ledger.core.compliance.SubjectChain;
import io.casehub.ledger.core.compliance.VerificationBundle;
import io.casehub.ledger.runtime.repository.LedgerMerkleFrontierRepository;

@ApplicationScoped
public class MerkleVerificationBundleService {

    @Inject
    LedgerEntryRepository repo;

    @Inject
    LedgerMerkleFrontierRepository frontierRepo;

    @Inject
    LedgerVerificationService verificationService;

    @Transactional
    public VerificationBundle generateBundle(final String tenancyId) {
        final List<UUID> subjectIds = repo.findDistinctSubjectIds(tenancyId);
        final List<SubjectChain> chains = subjectIds.stream()
                .map(sid -> buildChain(sid, tenancyId))
                .toList();
        final String script = loadResource("verification/verify.py");
        final String instructions = loadResource("verification/VERIFY-README.md");
        return new VerificationBundle(tenancyId, Instant.now(), chains, script, instructions);
    }

    private SubjectChain buildChain(final UUID subjectId, final String tenancyId) {
        final List<LedgerEntry> entries = repo.findBySubjectId(subjectId, tenancyId);
        final List<ChainEntry> chainEntries = entries.stream()
                .map(e -> new ChainEntry(e.sequenceNumber, e.digest))
                .toList();
        final List<LedgerMerkleFrontier> frontier =
                frontierRepo.findBySubjectId(subjectId, tenancyId);
        final List<FrontierNode> frontierNodes = frontier.stream()
                .map(f -> new FrontierNode(f.level, f.hash))
                .toList();
        String storedRoot;
        try {
            storedRoot = verificationService.treeRoot(subjectId, tenancyId);
        } catch (final Exception e) {
            storedRoot = null;
        }
        return new SubjectChain(subjectId, chainEntries, frontierNodes, storedRoot);
    }

    private String loadResource(final String name) {
        try (var is = getClass().getClassLoader().getResourceAsStream(name)) {
            if (is == null) return "";
            return new String(is.readAllBytes(), StandardCharsets.UTF_8);
        } catch (final IOException e) {
            return "";
        }
    }
}
```

- [ ] **Step 7: Install api + ledger-core and run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,ledger-core -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MerkleVerificationBundleServiceIT`
Expected: All 4 tests PASS

- [ ] **Step 8: Commit**

```bash
git add ledger-core/src/main/java/io/casehub/ledger/core/compliance/VerificationBundle.java \
  ledger-core/src/main/java/io/casehub/ledger/core/compliance/SubjectChain.java \
  ledger-core/src/main/java/io/casehub/ledger/core/compliance/ChainEntry.java \
  ledger-core/src/main/java/io/casehub/ledger/core/compliance/FrontierNode.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/MerkleVerificationBundleService.java \
  runtime/src/main/resources/verification/verify.py \
  runtime/src/main/resources/verification/VERIFY-README.md \
  runtime/src/test/java/io/casehub/ledger/service/MerkleVerificationBundleServiceIT.java
git commit -m "feat(#212): add MerkleVerificationBundleService — offline tamper-evidence package

Generates self-contained verification bundles with entry digests, MMR
frontier, stored roots, a Python verification script, and instructions.

Closes #212

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: End-to-end Python script verification test

**Files:**
- Test: `runtime/src/test/java/io/casehub/ledger/service/VerifyPyScriptTest.java`

**Interfaces:**
- Consumes: `LedgerMerkleTree.leafHash(LedgerEntry)`, `LedgerMerkleTree.append(hash, frontier, subjectId)`, `LedgerMerkleTree.treeRoot(frontier)`, Jackson `ObjectMapper` for JSON serialization

- [ ] **Step 1: Write the Python script test**

Create `runtime/src/test/java/io/casehub/ledger/service/VerifyPyScriptTest.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.model.LedgerMerkleFrontier;
import io.casehub.ledger.core.merkle.LedgerMerkleTree;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.service.supplement.TestEntry;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

class VerifyPyScriptTest {

    @TempDir Path tempDir;

    @Test
    void validChain_passesVerification() throws Exception {
        final UUID subjectId = UUID.randomUUID();
        final List<LedgerEntry> entries = List.of(
                testEntry(subjectId, "actor-1", 1),
                testEntry(subjectId, "actor-2", 2),
                testEntry(subjectId, "actor-3", 3));

        final String bundleJson = buildBundleJson(subjectId, entries);
        final int exitCode = runVerifyScript(bundleJson);

        assertThat(exitCode).isEqualTo(0);
    }

    @Test
    void tamperedChain_failsVerification() throws Exception {
        final UUID subjectId = UUID.randomUUID();
        final List<LedgerEntry> entries = List.of(
                testEntry(subjectId, "actor-1", 1),
                testEntry(subjectId, "actor-2", 2));

        String bundleJson = buildBundleJson(subjectId, entries);
        // Tamper: replace the first digest char
        bundleJson = bundleJson.replaceFirst(
                "\"digest\":\"([0-9a-f])",
                "\"digest\":\"0");

        final int exitCode = runVerifyScript(bundleJson);

        assertThat(exitCode).isEqualTo(1);
    }

    private String buildBundleJson(final UUID subjectId,
            final List<LedgerEntry> entries) throws Exception {
        // Compute digests and frontier using the real Merkle algorithm
        final List<Object> chainEntries = new ArrayList<>();
        List<LedgerMerkleFrontier> frontier = new ArrayList<>();
        for (final LedgerEntry entry : entries) {
            final String digest = LedgerMerkleTree.leafHash(entry);
            entry.digest = digest;
            chainEntries.add(new java.util.LinkedHashMap<>() {{
                put("sequenceNumber", entry.sequenceNumber);
                put("digest", digest);
            }});
            frontier = LedgerMerkleTree.append(digest, frontier, subjectId);
        }
        final String root = LedgerMerkleTree.treeRoot(frontier);

        final List<Object> frontierNodes = frontier.stream()
                .map(f -> new java.util.LinkedHashMap<>() {{
                    put("level", f.level);
                    put("hash", f.hash);
                }})
                .collect(java.util.stream.Collectors.toList());

        final var chain = new java.util.LinkedHashMap<>();
        chain.put("subjectId", subjectId.toString());
        chain.put("entries", chainEntries);
        chain.put("frontier", frontierNodes);
        chain.put("storedRoot", root);

        final var bundle = new java.util.LinkedHashMap<>();
        bundle.put("tenancyId", "test-tenant");
        bundle.put("generatedAt", Instant.now().toString());
        bundle.put("chains", List.of(chain));

        final ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        return mapper.writerWithDefaultPrettyPrinter().writeValueAsString(bundle);
    }

    private int runVerifyScript(final String bundleJson) throws Exception {
        // Write bundle JSON to temp file
        final Path bundlePath = tempDir.resolve("bundle.json");
        Files.writeString(bundlePath, bundleJson);

        // Copy verify.py from classpath to temp dir
        final Path scriptPath = tempDir.resolve("verify.py");
        try (InputStream is = getClass().getClassLoader()
                .getResourceAsStream("verification/verify.py")) {
            assertThat(is).as("verify.py must be on classpath").isNotNull();
            Files.write(scriptPath, is.readAllBytes());
        }

        // Run the Python script
        final ProcessBuilder pb = new ProcessBuilder(
                "python3", scriptPath.toString(), bundlePath.toString());
        pb.redirectErrorStream(true);
        final Process process = pb.start();
        final String output = new String(
                process.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
        final int exitCode = process.waitFor();

        System.out.println("verify.py output:\n" + output);
        return exitCode;
    }

    private static TestEntry testEntry(final UUID subjectId,
            final String actorId, final int seq) {
        final TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = seq;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.actorRole = "Verifier";
        e.occurredAt = Instant.now();
        e.tenancyId = "test-tenant";
        return e;
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=VerifyPyScriptTest`
Expected: Both tests PASS — valid chain exits 0, tampered chain exits 1

- [ ] **Step 3: Update `CLAUDE.md` — add MerkleVerificationBundleService**

Add to the `runtime/` service section in the project structure:
```
│       │   ├── MerkleVerificationBundleService.java — CDI bean: generates offline verification bundles (entry digests, MMR frontier, Python script)
```

- [ ] **Step 4: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: All tests PASS (including new + existing)

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/VerifyPyScriptTest.java CLAUDE.md
git commit -m "test(#212): end-to-end Python script verification — valid chain passes, tampered chain fails

Closes #212

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- `specs/issue-211-reporting-spi-compliance-report/2026-09-23-merkle-verification-bundle-design.md` — design spec
- `ledger-core/src/main/java/io/casehub/ledger/core/merkle/LedgerMerkleTree.java` — MMR algorithm
- `api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java:327` — canonicalBytes()
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java` — online verification
- `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerMerkleFrontierRepository.java` — frontier storage
- `api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryRepository.java` — findDistinctSubjectIds, findBySubjectId
- GitHub #211, #212
