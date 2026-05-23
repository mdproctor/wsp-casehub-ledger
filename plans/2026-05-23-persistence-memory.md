# persistence-memory Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `persistence-memory/` module (`casehub-ledger-memory`) with in-memory implementations of all ledger persistence SPIs, plus three runtime refactors that make the module structurally correct.

**Architecture:** Extract `LedgerEnricherPipeline` CDI bean and `LedgerMerkleFrontierRepository` SPI from the runtime module, refactor `JpaLedgerEntryRepository` and `LedgerVerificationService` to use them, then implement six in-memory stores in the new module. Reactive stores delegate to blocking stores via CDI injection.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI/Arc, SmallRye Mutiny, JUnit 5, AssertJ, H2 (test only)

---

## File Map

**New in `runtime/`:**
- `runtime/.../service/LedgerEnricherPipeline.java` — extracted enricher iteration
- `runtime/.../repository/LedgerMerkleFrontierRepository.java` — new SPI
- `runtime/.../repository/jpa/JpaLedgerMerkleFrontierRepository.java` — JPA impl

**Modified in `runtime/`:**
- `runtime/.../service/LedgerTraceListener.java` — delegate to `LedgerEnricherPipeline`
- `runtime/.../repository/jpa/JpaLedgerEntryRepository.java` — inject frontier SPI
- `runtime/.../service/LedgerVerificationService.java` — inject frontier SPI
- `runtime/src/test/resources/application.properties` — add `JpaLedgerMerkleFrontierRepository` to selected-alternatives

**New module `persistence-memory/`:**
- `persistence-memory/pom.xml`
- `persistence-memory/.../memory/InMemoryLedgerMerkleFrontierRepository.java`
- `persistence-memory/.../memory/InMemoryLedgerEntryRepository.java`
- `persistence-memory/.../memory/InMemoryActorTrustScoreRepository.java`
- `persistence-memory/.../memory/InMemoryKeyRotationRepository.java`
- `persistence-memory/.../memory/InMemoryReactiveLedgerEntryRepository.java`
- `persistence-memory/.../memory/InMemoryReactiveKeyRotationRepository.java`
- `persistence-memory/src/test/.../memory/InMemoryLedgerEntryRepositoryTest.java`
- `persistence-memory/src/test/.../memory/InMemoryActorTrustScoreRepositoryTest.java`
- `persistence-memory/src/test/.../memory/InMemoryKeyRotationRepositoryTest.java`
- `persistence-memory/src/test/resources/application.properties`

**Modified:**
- `pom.xml` (root) — add `<module>persistence-memory</module>`
- `CLAUDE.md` — add `persistence-memory/` to project structure

---

## Task 1: Extract `LedgerEnricherPipeline`

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEnricherPipeline.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerTraceListener.java`

- [ ] **Step 1: Create `LedgerEnricherPipeline.java`**

```java
package io.casehub.ledger.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.ledger.runtime.model.LedgerEntry;

@ApplicationScoped
public class LedgerEnricherPipeline {

    private static final Logger log = Logger.getLogger(LedgerEnricherPipeline.class);

    @Inject
    @Any
    Instance<LedgerEntryEnricher> enrichers;

    public void enrich(final LedgerEntry entry) {
        for (final LedgerEntryEnricher enricher : enrichers) {
            try {
                enricher.enrich(entry);
            } catch (final Exception ex) {
                log.warnf("LedgerEntryEnricher %s failed — entry will still be saved: %s",
                        enricher.getClass().getSimpleName(), ex.getMessage());
            }
        }
    }
}
```

- [ ] **Step 2: Refactor `LedgerTraceListener` to delegate**

Replace the full file content:

```java
package io.casehub.ledger.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.PrePersist;

import io.casehub.ledger.runtime.model.LedgerEntry;

@ApplicationScoped
public class LedgerTraceListener {

    @Inject
    LedgerEnricherPipeline enricherPipeline;

    @PrePersist
    public void prePersist(final Object entity) {
        if (!(entity instanceof LedgerEntry entry)) {
            return;
        }
        enricherPipeline.enrich(entry);
    }
}
```

- [ ] **Step 3: Run runtime tests — all must still pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: BUILD SUCCESS, same test count as before.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEnricherPipeline.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerTraceListener.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "refactor(#91): extract LedgerEnricherPipeline — shared by JPA and in-memory paths"
```

---

## Task 2: `LedgerMerkleFrontierRepository` SPI + refactors

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerMerkleFrontierRepository.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerMerkleFrontierRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Step 1: Create the SPI**

```java
package io.casehub.ledger.runtime.repository;

import java.util.List;
import java.util.UUID;

import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;

public interface LedgerMerkleFrontierRepository {

    List<LedgerMerkleFrontier> findBySubjectId(UUID subjectId);

    void replace(UUID subjectId, List<LedgerMerkleFrontier> newFrontier);
}
```

- [ ] **Step 2: Create `JpaLedgerMerkleFrontierRepository`**

```java
package io.casehub.ledger.runtime.repository.jpa;

import java.util.List;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.repository.LedgerMerkleFrontierRepository;

@ApplicationScoped
@Alternative
public class JpaLedgerMerkleFrontierRepository implements LedgerMerkleFrontierRepository {

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @Override
    public List<LedgerMerkleFrontier> findBySubjectId(final UUID subjectId) {
        return em.createNamedQuery("LedgerMerkleFrontier.findBySubjectId", LedgerMerkleFrontier.class)
                .setParameter("subjectId", subjectId)
                .getResultList();
    }

    @Override
    @Transactional
    public void replace(final UUID subjectId, final List<LedgerMerkleFrontier> newFrontier) {
        final Set<Integer> newLevels = newFrontier.stream()
                .map(n -> n.level)
                .collect(Collectors.toSet());

        for (final LedgerMerkleFrontier old : findBySubjectId(subjectId)) {
            if (!newLevels.contains(old.level)) {
                em.createNamedQuery("LedgerMerkleFrontier.deleteBySubjectAndLevel")
                        .setParameter("subjectId", subjectId)
                        .setParameter("level", old.level)
                        .executeUpdate();
            }
        }

        for (final LedgerMerkleFrontier node : newFrontier) {
            em.createNamedQuery("LedgerMerkleFrontier.deleteBySubjectAndLevel")
                    .setParameter("subjectId", subjectId)
                    .setParameter("level", node.level)
                    .executeUpdate();
            em.persist(node);
        }
    }
}
```

- [ ] **Step 3: Refactor `JpaLedgerEntryRepository` — inject frontier SPI**

Add field after `merklePublisher`:
```java
@Inject
LedgerMerkleFrontierRepository frontierRepo;
```

Replace the `updateMerkleFrontier` method:
```java
private void updateMerkleFrontier(final LedgerEntry entry) {
    final List<LedgerMerkleFrontier> currentFrontier = frontierRepo.findBySubjectId(entry.subjectId);
    final List<LedgerMerkleFrontier> newFrontier = LedgerMerkleTree.append(
            entry.digest, currentFrontier, entry.subjectId);
    frontierRepo.replace(entry.subjectId, newFrontier);
    final String newRoot = LedgerMerkleTree.treeRoot(newFrontier);
    merklePublisher.publish(entry.subjectId, entry.sequenceNumber, newRoot);
}
```

Remove the `import jakarta.transaction.Transactional;` from the import if it is now unused (the `@Transactional` annotation was only on `save()` which keeps it, so keep the import).

- [ ] **Step 4: Refactor `LedgerVerificationService` — drop `EntityManager`, inject SPI**

Replace the full file:

```java
package io.casehub.ledger.runtime.service;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.repository.LedgerMerkleFrontierRepository;
import io.casehub.ledger.runtime.service.model.InclusionProof;

@ApplicationScoped
public class LedgerVerificationService {

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    LedgerMerkleFrontierRepository frontierRepo;

    @Transactional
    public String treeRoot(final UUID subjectId) {
        final List<LedgerMerkleFrontier> frontier = frontierRepo.findBySubjectId(subjectId);
        if (frontier.isEmpty()) {
            throw new IllegalStateException("No entries for subject " + subjectId);
        }
        return LedgerMerkleTree.treeRoot(frontier);
    }

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

- [ ] **Step 5: Update `runtime/src/test/resources/application.properties`**

Change the default selected-alternatives line from:
```
quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository
```
to:
```
quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository
```

Also update the three profile-specific overrides that include `JpaLedgerEntryRepository` explicitly — add `JpaLedgerMerkleFrontierRepository` to each:

`%federation-import-test`:
```
%federation-import-test.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository,\
  io.casehub.ledger.runtime.service.federation.JpaTrustImportService
```

`%federation-bootstrap-test`:
```
%federation-bootstrap-test.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository,\
  io.casehub.ledger.runtime.service.federation.JpaTrustImportService,\
  io.casehub.ledger.service.federation.TrustBootstrapServiceIT$CapturingBootstrapSource
```

`%trust-score-bootstrap-test`:
```
%trust-score-bootstrap-test.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository,\
  io.casehub.ledger.runtime.service.federation.JpaTrustImportService,\
  io.casehub.ledger.service.TrustScoreBootstrapIT$SeedingBootstrapSource
```

- [ ] **Step 6: Run all runtime tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

Expected: BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerMerkleFrontierRepository.java \
  runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerMerkleFrontierRepository.java \
  runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java \
  runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/ledger commit -m "refactor(#91): extract LedgerMerkleFrontierRepository SPI — LedgerVerificationService no longer requires EntityManager"
```

---

## Task 3: Module scaffolding

**Files:**
- Create: `persistence-memory/pom.xml`
- Create: `persistence-memory/src/test/resources/application.properties`
- Modify: `pom.xml` (root)

- [ ] **Step 1: Create `persistence-memory/pom.xml`**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ledger-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-ledger-memory</artifactId>
  <name>CaseHub Ledger - In-Memory Persistence</name>
  <description>In-memory implementations of all ledger persistence SPIs — zero datasource, for tests and ephemeral installs</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger</artifactId>
      <version>${project.version}</version>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <executions>
          <execution>
            <id>make-index</id>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Add module to root `pom.xml`**

In `/Users/mdproctor/claude/casehub/ledger/pom.xml`, change:
```xml
  <modules>
    <module>api</module>
    <module>runtime</module>
    <module>deployment</module>
  </modules>
```
to:
```xml
  <modules>
    <module>api</module>
    <module>runtime</module>
    <module>deployment</module>
    <module>persistence-memory</module>
  </modules>
```

- [ ] **Step 3: Create test `application.properties`**

Create `persistence-memory/src/test/resources/application.properties`:

```properties
# H2 satisfies the ledger extension's Hibernate ORM setup.
# No tables are created — all persistence goes through in-memory stores.
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:ledgermemtestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.datasource.username=sa
quarkus.datasource.password=
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.enabled=false

quarkus.arc.selected-alternatives=\
  io.casehub.ledger.memory.InMemoryLedgerEntryRepository,\
  io.casehub.ledger.memory.InMemoryLedgerMerkleFrontierRepository,\
  io.casehub.ledger.memory.InMemoryActorTrustScoreRepository,\
  io.casehub.ledger.memory.InMemoryKeyRotationRepository

casehub.ledger.enabled=true
casehub.ledger.hash-chain.enabled=true
casehub.ledger.reactive.enabled=false
```

- [ ] **Step 4: Verify the module resolves**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn validate -pl persistence-memory
```

Expected: BUILD SUCCESS (no sources yet, just structure validation).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  persistence-memory/pom.xml \
  persistence-memory/src/test/resources/application.properties \
  pom.xml
git -C /Users/mdproctor/claude/casehub/ledger commit -m "chore(#91): scaffold persistence-memory module"
```

---

## Task 4: `InMemoryLedgerMerkleFrontierRepository`

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerMerkleFrontierRepository.java`

- [ ] **Step 1: Write a plain JUnit 5 unit test (no Quarkus)**

Create `persistence-memory/src/test/java/io/casehub/ledger/memory/InMemoryLedgerMerkleFrontierRepositoryTest.java`:

```java
package io.casehub.ledger.memory;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;

class InMemoryLedgerMerkleFrontierRepositoryTest {

    private InMemoryLedgerMerkleFrontierRepository repo;

    @BeforeEach
    void setUp() {
        repo = new InMemoryLedgerMerkleFrontierRepository();
    }

    @Test
    void findBySubjectId_returnsEmptyWhenNoneStored() {
        assertThat(repo.findBySubjectId(UUID.randomUUID())).isEmpty();
    }

    @Test
    void replace_storesAndRetrievesFrontier() {
        UUID subjectId = UUID.randomUUID();
        LedgerMerkleFrontier node = frontier(subjectId, 0, "abc");

        repo.replace(subjectId, List.of(node));

        List<LedgerMerkleFrontier> result = repo.findBySubjectId(subjectId);
        assertThat(result).hasSize(1);
        assertThat(result.get(0).hash).isEqualTo("abc");
    }

    @Test
    void replace_overwritesPreviousFrontier() {
        UUID subjectId = UUID.randomUUID();
        repo.replace(subjectId, List.of(frontier(subjectId, 0, "first")));
        repo.replace(subjectId, List.of(frontier(subjectId, 0, "second")));

        assertThat(repo.findBySubjectId(subjectId)).hasSize(1);
        assertThat(repo.findBySubjectId(subjectId).get(0).hash).isEqualTo("second");
    }

    @Test
    void findBySubjectId_returnsDefensiveCopy() {
        UUID subjectId = UUID.randomUUID();
        repo.replace(subjectId, List.of(frontier(subjectId, 0, "abc")));

        List<LedgerMerkleFrontier> result = repo.findBySubjectId(subjectId);
        result.clear(); // mutate returned list

        assertThat(repo.findBySubjectId(subjectId)).hasSize(1); // original unchanged
    }

    @Test
    void clear_removesAllFrontiers() {
        UUID s1 = UUID.randomUUID();
        UUID s2 = UUID.randomUUID();
        repo.replace(s1, List.of(frontier(s1, 0, "a")));
        repo.replace(s2, List.of(frontier(s2, 0, "b")));

        repo.clear();

        assertThat(repo.findBySubjectId(s1)).isEmpty();
        assertThat(repo.findBySubjectId(s2)).isEmpty();
    }

    private LedgerMerkleFrontier frontier(UUID subjectId, int level, String hash) {
        LedgerMerkleFrontier f = new LedgerMerkleFrontier();
        f.id = UUID.randomUUID();
        f.subjectId = subjectId;
        f.level = level;
        f.hash = hash;
        return f;
    }
}
```

- [ ] **Step 2: Run — expect compilation failure (class missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory 2>&1 | tail -5
```

Expected: compilation error — `InMemoryLedgerMerkleFrontierRepository` does not exist.

- [ ] **Step 3: Implement `InMemoryLedgerMerkleFrontierRepository`**

```java
package io.casehub.ledger.memory;

import java.util.Collections;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.repository.LedgerMerkleFrontierRepository;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryLedgerMerkleFrontierRepository implements LedgerMerkleFrontierRepository {

    private final ConcurrentHashMap<UUID, List<LedgerMerkleFrontier>> frontierBySubject =
            new ConcurrentHashMap<>();

    @Override
    public List<LedgerMerkleFrontier> findBySubjectId(final UUID subjectId) {
        return List.copyOf(frontierBySubject.getOrDefault(subjectId, Collections.emptyList()));
    }

    @Override
    public void replace(final UUID subjectId, final List<LedgerMerkleFrontier> newFrontier) {
        frontierBySubject.put(subjectId, List.copyOf(newFrontier));
    }

    public void clear() {
        frontierBySubject.clear();
    }
}
```

- [ ] **Step 4: Run — expect all tests to pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory -Dtest=InMemoryLedgerMerkleFrontierRepositoryTest
```

Expected: BUILD SUCCESS, 5 tests passing.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerMerkleFrontierRepository.java \
  persistence-memory/src/test/java/io/casehub/ledger/memory/InMemoryLedgerMerkleFrontierRepositoryTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#91): InMemoryLedgerMerkleFrontierRepository"
```

---

## Task 5: `InMemoryLedgerEntryRepository`

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java`
- Create: `persistence-memory/src/test/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepositoryTest.java`

- [ ] **Step 1: Write the test class**

```java
package io.casehub.ledger.memory;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class InMemoryLedgerEntryRepositoryTest {

    @Inject
    InMemoryLedgerEntryRepository repo;

    @BeforeEach
    void setUp() {
        repo.clear();
    }

    // ── save ─────────────────────────────────────────────────────────────────

    @Test
    void save_assignsId() {
        MemoryTestEntry e = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        e.id = null;
        repo.save(e);
        assertThat(e.id).isNotNull();
    }

    @Test
    void save_assignsOccurredAt() {
        MemoryTestEntry e = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        e.occurredAt = null;
        repo.save(e);
        assertThat(e.occurredAt).isNotNull();
    }

    @Test
    void save_assignsSequenceNumber() {
        UUID subjectId = UUID.randomUUID();
        MemoryTestEntry e1 = entry(subjectId, LedgerEntryType.EVENT);
        MemoryTestEntry e2 = entry(subjectId, LedgerEntryType.EVENT);
        repo.save(e1);
        repo.save(e2);
        assertThat(e1.sequenceNumber).isEqualTo(1);
        assertThat(e2.sequenceNumber).isEqualTo(2);
    }

    @Test
    void save_sequenceNumberIsPerSubject() {
        UUID s1 = UUID.randomUUID();
        UUID s2 = UUID.randomUUID();
        MemoryTestEntry a = entry(s1, LedgerEntryType.EVENT);
        MemoryTestEntry b = entry(s2, LedgerEntryType.EVENT);
        MemoryTestEntry c = entry(s1, LedgerEntryType.EVENT);
        repo.save(a);
        repo.save(b);
        repo.save(c);
        assertThat(a.sequenceNumber).isEqualTo(1);
        assertThat(b.sequenceNumber).isEqualTo(1);
        assertThat(c.sequenceNumber).isEqualTo(2);
    }

    // ── findBySubjectId ───────────────────────────────────────────────────────

    @Test
    void findBySubjectId_returnsOrderedBySequenceNumber() {
        UUID subjectId = UUID.randomUUID();
        repo.save(entry(subjectId, LedgerEntryType.EVENT));
        repo.save(entry(subjectId, LedgerEntryType.COMMAND));
        repo.save(entry(subjectId, LedgerEntryType.EVENT));

        List<LedgerEntry> results = repo.findBySubjectId(subjectId);
        assertThat(results).hasSize(3);
        assertThat(results.get(0).sequenceNumber).isEqualTo(1);
        assertThat(results.get(1).sequenceNumber).isEqualTo(2);
        assertThat(results.get(2).sequenceNumber).isEqualTo(3);
    }

    @Test
    void findBySubjectId_returnsEmptyForUnknownSubject() {
        assertThat(repo.findBySubjectId(UUID.randomUUID())).isEmpty();
    }

    // ── findEntryById ─────────────────────────────────────────────────────────

    @Test
    void findEntryById_returnsEntry() {
        MemoryTestEntry e = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        repo.save(e);
        Optional<LedgerEntry> result = repo.findEntryById(e.id);
        assertThat(result).isPresent();
        assertThat(result.get().id).isEqualTo(e.id);
    }

    @Test
    void findEntryById_returnsEmptyForUnknown() {
        assertThat(repo.findEntryById(UUID.randomUUID())).isEmpty();
    }

    // ── findLatestBySubjectId ─────────────────────────────────────────────────

    @Test
    void findLatestBySubjectId_returnsHighestSequenceNumber() {
        UUID subjectId = UUID.randomUUID();
        repo.save(entry(subjectId, LedgerEntryType.EVENT));
        MemoryTestEntry last = entry(subjectId, LedgerEntryType.COMMAND);
        repo.save(last);

        Optional<LedgerEntry> result = repo.findLatestBySubjectId(subjectId);
        assertThat(result).isPresent();
        assertThat(result.get().id).isEqualTo(last.id);
    }

    // ── time range queries ────────────────────────────────────────────────────

    @Test
    void findBySubjectIdAndTimeRange_filtersInclusively() {
        UUID subjectId = UUID.randomUUID();
        Instant t1 = Instant.parse("2026-01-01T00:00:00Z");
        Instant t2 = Instant.parse("2026-06-01T00:00:00Z");
        Instant t3 = Instant.parse("2026-12-01T00:00:00Z");

        MemoryTestEntry e1 = entry(subjectId, LedgerEntryType.EVENT);
        e1.occurredAt = t1;
        MemoryTestEntry e2 = entry(subjectId, LedgerEntryType.EVENT);
        e2.occurredAt = t2;
        MemoryTestEntry e3 = entry(subjectId, LedgerEntryType.EVENT);
        e3.occurredAt = t3;
        repo.save(e1); repo.save(e2); repo.save(e3);

        List<LedgerEntry> results = repo.findBySubjectIdAndTimeRange(subjectId, t1, t2);
        assertThat(results).hasSize(2);
    }

    @Test
    void findByTimeRange_acrossAllSubjects() {
        Instant t1 = Instant.parse("2026-01-01T00:00:00Z");
        Instant t2 = Instant.parse("2026-06-01T00:00:00Z");

        MemoryTestEntry in1 = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        in1.occurredAt = t1;
        MemoryTestEntry in2 = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        in2.occurredAt = t2;
        MemoryTestEntry out = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        out.occurredAt = Instant.parse("2026-12-01T00:00:00Z");
        repo.save(in1); repo.save(in2); repo.save(out);

        List<LedgerEntry> results = repo.findByTimeRange(t1, t2);
        assertThat(results).hasSize(2);
    }

    // ── actor queries ─────────────────────────────────────────────────────────

    @Test
    void findByActorId_filtersAndOrdersByOccurredAt() {
        String actor = "actor-" + UUID.randomUUID();
        Instant t1 = Instant.parse("2026-01-01T00:00:00Z");
        Instant t2 = Instant.parse("2026-06-01T00:00:00Z");

        MemoryTestEntry e1 = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        e1.actorId = actor; e1.occurredAt = t2;
        MemoryTestEntry e2 = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        e2.actorId = actor; e2.occurredAt = t1;
        MemoryTestEntry other = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        other.actorId = "other";
        repo.save(e1); repo.save(e2); repo.save(other);

        List<LedgerEntry> results = repo.findByActorId(
                actor, Instant.parse("2025-01-01T00:00:00Z"), Instant.parse("2027-01-01T00:00:00Z"));
        assertThat(results).hasSize(2);
        assertThat(results.get(0).occurredAt).isEqualTo(t1);
    }

    @Test
    void findByActorRole_filters() {
        String role = "Classifier-" + UUID.randomUUID();
        MemoryTestEntry e = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        e.actorRole = role;
        repo.save(e);

        List<LedgerEntry> results = repo.findByActorRole(
                role, Instant.EPOCH, Instant.parse("2099-01-01T00:00:00Z"));
        assertThat(results).hasSize(1);
    }

    // ── causal + type queries ─────────────────────────────────────────────────

    @Test
    void findCausedBy_returnsDownstreamEntries() {
        MemoryTestEntry cause = entry(UUID.randomUUID(), LedgerEntryType.COMMAND);
        repo.save(cause);
        MemoryTestEntry effect = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        effect.causedByEntryId = cause.id;
        repo.save(effect);

        List<LedgerEntry> results = repo.findCausedBy(cause.id);
        assertThat(results).hasSize(1);
        assertThat(results.get(0).id).isEqualTo(effect.id);
    }

    @Test
    void findAllEvents_returnsOnlyEvents() {
        repo.save(entry(UUID.randomUUID(), LedgerEntryType.EVENT));
        repo.save(entry(UUID.randomUUID(), LedgerEntryType.COMMAND));
        repo.save(entry(UUID.randomUUID(), LedgerEntryType.EVENT));

        List<LedgerEntry> results = repo.findAllEvents();
        assertThat(results).hasSize(2);
        assertThat(results).allMatch(e -> e.entryType == LedgerEntryType.EVENT);
    }

    @Test
    void listAll_returnsAllEntries() {
        repo.save(entry(UUID.randomUUID(), LedgerEntryType.EVENT));
        repo.save(entry(UUID.randomUUID(), LedgerEntryType.COMMAND));
        assertThat(repo.listAll()).hasSize(2);
    }

    // ── attestations ──────────────────────────────────────────────────────────

    @Test
    void saveAttestation_andFindByEntryId() {
        MemoryTestEntry e = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        repo.save(e);

        LedgerAttestation a = attestation(e.id, e.subjectId, "attesting-actor", "*");
        repo.saveAttestation(a);

        List<LedgerAttestation> results = repo.findAttestationsByEntryId(e.id);
        assertThat(results).hasSize(1);
        assertThat(results.get(0).attestorId).isEqualTo("attesting-actor");
    }

    @Test
    void findAttestationsForEntries_groupsByEntryId() {
        MemoryTestEntry e1 = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        MemoryTestEntry e2 = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        repo.save(e1); repo.save(e2);
        repo.saveAttestation(attestation(e1.id, e1.subjectId, "a1", "*"));
        repo.saveAttestation(attestation(e1.id, e1.subjectId, "a2", "*"));
        repo.saveAttestation(attestation(e2.id, e2.subjectId, "a3", "*"));

        Map<UUID, List<LedgerAttestation>> grouped =
                repo.findAttestationsForEntries(Set.of(e1.id, e2.id));
        assertThat(grouped.get(e1.id)).hasSize(2);
        assertThat(grouped.get(e2.id)).hasSize(1);
    }

    @Test
    void findAttestationsByEntryIdAndCapabilityTag_filters() {
        MemoryTestEntry e = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        repo.save(e);
        repo.saveAttestation(attestation(e.id, e.subjectId, "a1", "review"));
        repo.saveAttestation(attestation(e.id, e.subjectId, "a2", "*"));

        List<LedgerAttestation> results =
                repo.findAttestationsByEntryIdAndCapabilityTag(e.id, "review");
        assertThat(results).hasSize(1);
        assertThat(results.get(0).attestorId).isEqualTo("a1");
    }

    @Test
    void findAttestationsByEntryIdGlobal_returnsOnlyStar() {
        MemoryTestEntry e = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        repo.save(e);
        repo.saveAttestation(attestation(e.id, e.subjectId, "a1", "*"));
        repo.saveAttestation(attestation(e.id, e.subjectId, "a2", "review"));

        List<LedgerAttestation> results = repo.findAttestationsByEntryIdGlobal(e.id);
        assertThat(results).hasSize(1);
        assertThat(results.get(0).capabilityTag).isEqualTo("*");
    }

    @Test
    void findAttestationsByAttestorIdAndCapabilityTag_filters() {
        MemoryTestEntry e = entry(UUID.randomUUID(), LedgerEntryType.EVENT);
        repo.save(e);
        repo.saveAttestation(attestation(e.id, e.subjectId, "claude:v1", "review"));
        repo.saveAttestation(attestation(e.id, e.subjectId, "claude:v1", "*"));
        repo.saveAttestation(attestation(e.id, e.subjectId, "other", "review"));

        List<LedgerAttestation> results =
                repo.findAttestationsByAttestorIdAndCapabilityTag("claude:v1", "review");
        assertThat(results).hasSize(1);
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    private MemoryTestEntry entry(UUID subjectId, LedgerEntryType type) {
        MemoryTestEntry e = new MemoryTestEntry();
        e.subjectId = subjectId;
        e.entryType = type;
        e.actorId = "test-actor";
        e.actorType = ActorType.AGENT;
        e.actorRole = "TestRole";
        return e;
    }

    private LedgerAttestation attestation(UUID entryId, UUID subjectId,
            String attestorId, String capabilityTag) {
        LedgerAttestation a = new LedgerAttestation();
        a.ledgerEntryId = entryId;
        a.subjectId = subjectId;
        a.attestorId = attestorId;
        a.attestorType = ActorType.AGENT;
        a.verdict = AttestationVerdict.SOUND;
        a.confidence = 1.0;
        a.capabilityTag = capabilityTag;
        return a;
    }

    /** Concrete non-entity subclass for in-memory testing. */
    static class MemoryTestEntry extends LedgerEntry {
    }
}
```

- [ ] **Step 2: Run — expect compilation failure (class missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory \
  -Dtest=InMemoryLedgerEntryRepositoryTest 2>&1 | tail -5
```

Expected: compilation error.

- [ ] **Step 3: Implement `InMemoryLedgerEntryRepository`**

```java
package io.casehub.ledger.memory;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerMerkleFrontier;
import io.casehub.ledger.runtime.privacy.ActorIdentityProvider;
import io.casehub.ledger.runtime.privacy.DecisionContextSanitiser;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.repository.LedgerMerkleFrontierRepository;
import io.casehub.ledger.runtime.service.LedgerEnricherPipeline;
import io.casehub.ledger.runtime.service.LedgerMerklePublisher;
import io.casehub.ledger.runtime.service.LedgerMerkleTree;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryLedgerEntryRepository implements LedgerEntryRepository {

    @Inject
    LedgerMerkleFrontierRepository frontierRepo;

    @Inject
    LedgerEnricherPipeline enricherPipeline;

    @Inject
    ActorIdentityProvider actorIdentityProvider;

    @Inject
    DecisionContextSanitiser decisionContextSanitiser;

    @Inject
    LedgerMerklePublisher merklePublisher;

    @Inject
    LedgerConfig ledgerConfig;

    // package-private so InMemoryKeyRotationRepository can read it
    final ConcurrentHashMap<UUID, LedgerEntry> entries = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<UUID, LedgerAttestation> attestations = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<UUID, AtomicInteger> sequenceCounters = new ConcurrentHashMap<>();

    @Override
    public LedgerEntry save(final LedgerEntry entry) {
        if (entry.id == null) {
            entry.id = UUID.randomUUID();
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

        enricherPipeline.enrich(entry);

        entry.sequenceNumber = sequenceCounters
                .computeIfAbsent(entry.subjectId, k -> new AtomicInteger(0))
                .incrementAndGet();

        if (ledgerConfig.hashChain().enabled()) {
            entry.digest = LedgerMerkleTree.leafHash(entry);
        }

        entries.put(entry.id, entry);

        if (ledgerConfig.hashChain().enabled()) {
            final List<LedgerMerkleFrontier> current = frontierRepo.findBySubjectId(entry.subjectId);
            final List<LedgerMerkleFrontier> newFrontier =
                    LedgerMerkleTree.append(entry.digest, current, entry.subjectId);
            frontierRepo.replace(entry.subjectId, newFrontier);
            final String newRoot = LedgerMerkleTree.treeRoot(newFrontier);
            merklePublisher.publish(entry.subjectId, entry.sequenceNumber, newRoot);
        }

        return entry;
    }

    @Override
    public List<LedgerEntry> findBySubjectId(final UUID subjectId) {
        return entries.values().stream()
                .filter(e -> subjectId.equals(e.subjectId))
                .sorted(Comparator.comparingInt(e -> e.sequenceNumber))
                .collect(Collectors.toList());
    }

    @Override
    public List<LedgerEntry> findBySubjectIdAndTimeRange(final UUID subjectId,
            final Instant from, final Instant to) {
        return entries.values().stream()
                .filter(e -> subjectId.equals(e.subjectId))
                .filter(e -> !e.occurredAt.isBefore(from) && !e.occurredAt.isAfter(to))
                .sorted(Comparator.comparing(e -> e.occurredAt))
                .collect(Collectors.toList());
    }

    @Override
    public Optional<LedgerEntry> findLatestBySubjectId(final UUID subjectId) {
        return entries.values().stream()
                .filter(e -> subjectId.equals(e.subjectId))
                .max(Comparator.comparingInt(e -> e.sequenceNumber));
    }

    @Override
    public Optional<LedgerEntry> findEntryById(final UUID id) {
        return Optional.ofNullable(entries.get(id));
    }

    @Override
    public List<LedgerAttestation> findAttestationsByEntryId(final UUID ledgerEntryId) {
        return attestations.values().stream()
                .filter(a -> ledgerEntryId.equals(a.ledgerEntryId))
                .sorted(Comparator.comparing(a -> a.occurredAt))
                .collect(Collectors.toList());
    }

    @Override
    public LedgerAttestation saveAttestation(final LedgerAttestation attestation) {
        if (attestation.id == null) {
            attestation.id = UUID.randomUUID();
        }
        if (attestation.occurredAt == null) {
            attestation.occurredAt = Instant.now();
        }
        if (attestation.attestorId != null) {
            attestation.attestorId = actorIdentityProvider.tokenise(attestation.attestorId);
        }
        attestations.put(attestation.id, attestation);
        return attestation;
    }

    @Override
    public List<LedgerEntry> listAll() {
        return new ArrayList<>(entries.values());
    }

    @Override
    public List<LedgerEntry> findAllEvents() {
        return entries.values().stream()
                .filter(e -> LedgerEntryType.EVENT.equals(e.entryType))
                .collect(Collectors.toList());
    }

    @Override
    public Map<UUID, List<LedgerAttestation>> findAttestationsForEntries(final Set<UUID> entryIds) {
        if (entryIds.isEmpty()) {
            return Collections.emptyMap();
        }
        return attestations.values().stream()
                .filter(a -> entryIds.contains(a.ledgerEntryId))
                .collect(Collectors.groupingBy(a -> a.ledgerEntryId));
    }

    @Override
    public List<LedgerEntry> findByActorId(final String actorId,
            final Instant from, final Instant to) {
        final String token = actorIdentityProvider.tokeniseForQuery(actorId);
        return entries.values().stream()
                .filter(e -> token.equals(e.actorId))
                .filter(e -> !e.occurredAt.isBefore(from) && !e.occurredAt.isAfter(to))
                .sorted(Comparator.comparing(e -> e.occurredAt))
                .collect(Collectors.toList());
    }

    @Override
    public List<LedgerEntry> findByActorRole(final String actorRole,
            final Instant from, final Instant to) {
        return entries.values().stream()
                .filter(e -> actorRole.equals(e.actorRole))
                .filter(e -> !e.occurredAt.isBefore(from) && !e.occurredAt.isAfter(to))
                .sorted(Comparator.comparing(e -> e.occurredAt))
                .collect(Collectors.toList());
    }

    @Override
    public List<LedgerEntry> findByTimeRange(final Instant from, final Instant to) {
        return entries.values().stream()
                .filter(e -> !e.occurredAt.isBefore(from) && !e.occurredAt.isAfter(to))
                .sorted(Comparator.comparing(e -> e.occurredAt))
                .collect(Collectors.toList());
    }

    @Override
    public List<LedgerEntry> findCausedBy(final UUID entryId) {
        return entries.values().stream()
                .filter(e -> entryId.equals(e.causedByEntryId))
                .sorted(Comparator.comparing(e -> e.occurredAt))
                .collect(Collectors.toList());
    }

    @Override
    public List<LedgerAttestation> findAttestationsByEntryIdAndCapabilityTag(
            final UUID entryId, final String capabilityTag) {
        return attestations.values().stream()
                .filter(a -> entryId.equals(a.ledgerEntryId))
                .filter(a -> capabilityTag.equals(a.capabilityTag))
                .sorted(Comparator.comparing(a -> a.occurredAt))
                .collect(Collectors.toList());
    }

    @Override
    public List<LedgerAttestation> findAttestationsByEntryIdGlobal(final UUID entryId) {
        return findAttestationsByEntryIdAndCapabilityTag(entryId,
                io.casehub.ledger.api.model.CapabilityTag.GLOBAL);
    }

    @Override
    public List<LedgerAttestation> findAttestationsByAttestorIdAndCapabilityTag(
            final String attestorId, final String capabilityTag) {
        final String token = actorIdentityProvider.tokeniseForQuery(attestorId);
        return attestations.values().stream()
                .filter(a -> token.equals(a.attestorId))
                .filter(a -> capabilityTag.equals(a.capabilityTag))
                .sorted(Comparator.comparing(a -> a.occurredAt))
                .collect(Collectors.toList());
    }

    public void clear() {
        entries.clear();
        attestations.clear();
        sequenceCounters.clear();
    }
}
```

- [ ] **Step 4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory \
  -Dtest=InMemoryLedgerEntryRepositoryTest
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java \
  persistence-memory/src/test/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepositoryTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#91): InMemoryLedgerEntryRepository"
```

---

## Task 6: `InMemoryActorTrustScoreRepository`

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryActorTrustScoreRepository.java`
- Create: `persistence-memory/src/test/java/io/casehub/ledger/memory/InMemoryActorTrustScoreRepositoryTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.ledger.memory;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class InMemoryActorTrustScoreRepositoryTest {

    @Inject
    InMemoryActorTrustScoreRepository repo;

    @BeforeEach
    void setUp() {
        repo.clear();
    }

    @Test
    void upsert_global_andFindByActorId() {
        String actorId = "actor-" + UUID.randomUUID();
        upsert(actorId, ScoreType.GLOBAL, null, null, 0.8);

        assertThat(repo.findByActorId(actorId)).isPresent();
        assertThat(repo.findByActorId(actorId).get().trustScore).isEqualTo(0.8);
    }

    @Test
    void upsert_capability_andFindCapabilityScore() {
        String actorId = "actor-" + UUID.randomUUID();
        upsert(actorId, ScoreType.CAPABILITY, "review", null, 0.7);

        assertThat(repo.findCapabilityScore(actorId, "review")).isPresent();
        assertThat(repo.findCapabilityScore(actorId, "review").get().trustScore).isEqualTo(0.7);
        assertThat(repo.findCapabilityScore(actorId, "other")).isEmpty();
    }

    @Test
    void upsert_dimension_andFindDimensionScore() {
        String actorId = "actor-" + UUID.randomUUID();
        upsert(actorId, ScoreType.DIMENSION, null, "accuracy", 0.9);

        assertThat(repo.findDimensionScore(actorId, "accuracy")).isPresent();
        assertThat(repo.findDimensionScore(actorId, "other")).isEmpty();
    }

    @Test
    void upsert_capabilityDimension_andFind() {
        String actorId = "actor-" + UUID.randomUUID();
        upsert(actorId, ScoreType.CAPABILITY_DIMENSION, "review", "accuracy", 0.85);

        assertThat(repo.findCapabilityDimension(actorId, "review", "accuracy")).isPresent();
        assertThat(repo.findCapabilityDimension(actorId, "review", "other")).isEmpty();
    }

    @Test
    void upsert_isIdempotent_updatesExistingRow() {
        String actorId = "actor-" + UUID.randomUUID();
        upsert(actorId, ScoreType.GLOBAL, null, null, 0.5);
        upsert(actorId, ScoreType.GLOBAL, null, null, 0.9);

        assertThat(repo.findAll()).hasSize(1);
        assertThat(repo.findByActorId(actorId).get().trustScore).isEqualTo(0.9);
    }

    @Test
    void updateGlobalTrustScore_updatesField() {
        String actorId = "actor-" + UUID.randomUUID();
        upsert(actorId, ScoreType.GLOBAL, null, null, 0.5);

        repo.updateGlobalTrustScore(actorId, 0.99);

        assertThat(repo.findByActorId(actorId).get().globalTrustScore).isEqualTo(0.99);
    }

    @Test
    void findCapabilityDimensions_returnsAllForCapability() {
        String actorId = "actor-" + UUID.randomUUID();
        upsert(actorId, ScoreType.CAPABILITY_DIMENSION, "review", "accuracy", 0.8);
        upsert(actorId, ScoreType.CAPABILITY_DIMENSION, "review", "speed", 0.6);
        upsert(actorId, ScoreType.CAPABILITY_DIMENSION, "other", "accuracy", 0.7);

        List<ActorTrustScore> results = repo.findCapabilityDimensions(actorId, "review");
        assertThat(results).hasSize(2);
    }

    @Test
    void findAll_returnsEverything() {
        upsert("a1", ScoreType.GLOBAL, null, null, 0.5);
        upsert("a2", ScoreType.GLOBAL, null, null, 0.6);
        assertThat(repo.findAll()).hasSize(2);
    }

    @Test
    void findAllByLastComputedAtAfter_filters() {
        Instant threshold = Instant.parse("2026-06-01T00:00:00Z");
        upsertAt("a1", ScoreType.GLOBAL, null, null, 0.5, Instant.parse("2026-01-01T00:00:00Z"));
        upsertAt("a2", ScoreType.GLOBAL, null, null, 0.6, Instant.parse("2026-12-01T00:00:00Z"));

        List<ActorTrustScore> results = repo.findAllByLastComputedAtAfter(threshold);
        assertThat(results).hasSize(1);
        assertThat(results.get(0).actorId).isEqualTo("a2");
    }

    @Test
    void findByActorIdAndScoreType_returnsMatchingRows() {
        String actorId = "actor-" + UUID.randomUUID();
        upsert(actorId, ScoreType.GLOBAL, null, null, 0.5);
        upsert(actorId, ScoreType.CAPABILITY, "review", null, 0.7);
        upsert(actorId, ScoreType.CAPABILITY, "code", null, 0.8);

        List<ActorTrustScore> results = repo.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY);
        assertThat(results).hasSize(2);
    }

    private void upsert(String actorId, ScoreType type, String cap, String dim, double score) {
        upsertAt(actorId, type, cap, dim, score, Instant.now());
    }

    private void upsertAt(String actorId, ScoreType type, String cap, String dim,
            double score, Instant at) {
        repo.upsert(actorId, type, cap, dim, ActorType.AGENT,
                score, 10, 1, 9.0, 2.0, 8, 2, at);
    }
}
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory \
  -Dtest=InMemoryActorTrustScoreRepositoryTest 2>&1 | tail -5
```

- [ ] **Step 3: Implement `InMemoryActorTrustScoreRepository`**

```java
package io.casehub.ledger.memory;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;

import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.platform.api.identity.ActorType;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryActorTrustScoreRepository
        implements io.casehub.ledger.runtime.repository.ActorTrustScoreRepository {

    private final ConcurrentHashMap<String, ActorTrustScore> store = new ConcurrentHashMap<>();

    private static String key(String actorId, ScoreType type, String cap, String dim) {
        return actorId + "|" + type + "|" + nvl(cap) + "|" + nvl(dim);
    }

    private static String nvl(String s) {
        return s != null ? s : "";
    }

    @Override
    public Optional<ActorTrustScore> findByActorId(final String actorId) {
        return Optional.ofNullable(store.get(key(actorId, ScoreType.GLOBAL, null, null)));
    }

    @Override
    public Optional<ActorTrustScore> findCapabilityScore(final String actorId,
            final String capabilityTag) {
        return Optional.ofNullable(store.get(key(actorId, ScoreType.CAPABILITY, capabilityTag, null)));
    }

    @Override
    public Optional<ActorTrustScore> findDimensionScore(final String actorId,
            final String dimension) {
        return Optional.ofNullable(store.get(key(actorId, ScoreType.DIMENSION, null, dimension)));
    }

    @Override
    public Optional<ActorTrustScore> findCapabilityDimension(final String actorId,
            final String capabilityTag, final String dimension) {
        return Optional.ofNullable(
                store.get(key(actorId, ScoreType.CAPABILITY_DIMENSION, capabilityTag, dimension)));
    }

    @Override
    public List<ActorTrustScore> findCapabilityDimensions(final String actorId,
            final String capabilityTag) {
        return store.values().stream()
                .filter(s -> actorId.equals(s.actorId))
                .filter(s -> ScoreType.CAPABILITY_DIMENSION.equals(s.scoreType))
                .filter(s -> capabilityTag.equals(s.capabilityKey))
                .collect(Collectors.toList());
    }

    @Override
    public List<ActorTrustScore> findByActorIdAndScoreType(final String actorId,
            final ScoreType scoreType) {
        return store.values().stream()
                .filter(s -> actorId.equals(s.actorId))
                .filter(s -> scoreType.equals(s.scoreType))
                .collect(Collectors.toList());
    }

    @Override
    public void upsert(final String actorId, final ScoreType scoreType,
            final String capabilityKey, final String dimensionKey,
            final ActorType actorType, final double trustScore,
            final int decisionCount, final int overturnedCount,
            final double alpha, final double beta,
            final int attestationPositive, final int attestationNegative,
            final Instant lastComputedAt) {

        final String k = key(actorId, scoreType, capabilityKey, dimensionKey);
        ActorTrustScore score = store.get(k);
        if (score == null) {
            score = new ActorTrustScore();
            score.id = UUID.randomUUID();
            score.actorId = actorId;
            score.scoreType = scoreType;
            score.capabilityKey = capabilityKey;
            score.dimensionKey = dimensionKey;
        }
        score.actorType = actorType;
        score.trustScore = trustScore;
        score.alpha = alpha;
        score.beta = beta;
        score.decisionCount = decisionCount;
        score.overturnedCount = overturnedCount;
        score.attestationPositive = attestationPositive;
        score.attestationNegative = attestationNegative;
        score.lastComputedAt = lastComputedAt;
        store.put(k, score);
    }

    @Override
    public void updateGlobalTrustScore(final String actorId, final double globalTrustScore) {
        final String k = key(actorId, ScoreType.GLOBAL, null, null);
        final ActorTrustScore score = store.get(k);
        if (score != null) {
            score.globalTrustScore = globalTrustScore;
        }
    }

    @Override
    public List<ActorTrustScore> findAll() {
        return new ArrayList<>(store.values());
    }

    @Override
    public List<ActorTrustScore> findAllByLastComputedAtAfter(final Instant since) {
        return store.values().stream()
                .filter(s -> s.lastComputedAt != null && s.lastComputedAt.isAfter(since))
                .collect(Collectors.toList());
    }

    public void clear() {
        store.clear();
    }
}
```

- [ ] **Step 4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory \
  -Dtest=InMemoryActorTrustScoreRepositoryTest
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryActorTrustScoreRepository.java \
  persistence-memory/src/test/java/io/casehub/ledger/memory/InMemoryActorTrustScoreRepositoryTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#91): InMemoryActorTrustScoreRepository"
```

---

## Task 7: `InMemoryKeyRotationRepository`

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryKeyRotationRepository.java`
- Create: `persistence-memory/src/test/java/io/casehub/ledger/memory/InMemoryKeyRotationRepositoryTest.java`

- [ ] **Step 1: Write the test**

```java
package io.casehub.ledger.memory;

import static org.assertj.core.api.Assertions.assertThat;

import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class InMemoryKeyRotationRepositoryTest {

    @Inject
    InMemoryLedgerEntryRepository entryRepo;

    @Inject
    InMemoryKeyRotationRepository rotationRepo;

    @BeforeEach
    void setUp() {
        entryRepo.clear();
    }

    @Test
    void findByActorId_returnsRotationsOrderedByOccurredAt() {
        String actorId = "claude:reviewer@v1";
        Instant t1 = Instant.parse("2026-01-01T00:00:00Z");
        Instant t2 = Instant.parse("2026-06-01T00:00:00Z");

        KeyRotationEntry r1 = rotation(actorId, "keyRef-A", "keyRef-B",
                KeyRotationReason.SCHEDULED, t1, t1);
        KeyRotationEntry r2 = rotation(actorId, "keyRef-B", "keyRef-C",
                KeyRotationReason.SCHEDULED, t2, t2);
        entryRepo.save(r1);
        entryRepo.save(r2);

        List<KeyRotationEntry> results = rotationRepo.findByActorId(actorId);
        assertThat(results).hasSize(2);
        assertThat(results.get(0).occurredAt).isEqualTo(t1);
        assertThat(results.get(1).occurredAt).isEqualTo(t2);
    }

    @Test
    void findByActorId_doesNotReturnOtherActors() {
        entryRepo.save(rotation("actor-A", "k1", "k2", KeyRotationReason.SCHEDULED,
                Instant.now(), Instant.now()));
        entryRepo.save(rotation("actor-B", "k1", "k2", KeyRotationReason.SCHEDULED,
                Instant.now(), Instant.now()));

        assertThat(rotationRepo.findByActorId("actor-A")).hasSize(1);
    }

    @Test
    void findCompromisedByActorIdAndKeyRef_filtersCorrectly() {
        String actorId = "claude:reviewer@v1";
        Instant effectiveSince = Instant.parse("2026-03-01T00:00:00Z");

        entryRepo.save(rotation(actorId, "bad-key", "new-key",
                KeyRotationReason.COMPROMISED, Instant.now(), effectiveSince));
        entryRepo.save(rotation(actorId, "other-key", "another",
                KeyRotationReason.COMPROMISED, Instant.now(), effectiveSince));
        entryRepo.save(rotation(actorId, "bad-key", "new-key2",
                KeyRotationReason.SCHEDULED, Instant.now(), effectiveSince));

        List<KeyRotationEntry> results =
                rotationRepo.findCompromisedByActorIdAndKeyRef(actorId, "bad-key");
        assertThat(results).hasSize(1);
        assertThat(results.get(0).reason).isEqualTo(KeyRotationReason.COMPROMISED);
        assertThat(results.get(0).previousKeyRef).isEqualTo("bad-key");
    }

    private KeyRotationEntry rotation(String actorId, String prevKey, String newKey,
            KeyRotationReason reason, Instant occurredAt, Instant effectiveSince) {
        KeyRotationEntry e = new KeyRotationEntry();
        e.subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(StandardCharsets.UTF_8));
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.occurredAt = occurredAt;
        e.previousKeyRef = prevKey;
        e.newKeyRef = newKey;
        e.reason = reason;
        e.effectiveSince = effectiveSince;
        return e;
    }
}
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory \
  -Dtest=InMemoryKeyRotationRepositoryTest 2>&1 | tail -5
```

- [ ] **Step 3: Implement `InMemoryKeyRotationRepository`**

```java
package io.casehub.ledger.memory;

import java.util.Comparator;
import java.util.List;
import java.util.stream.Collectors;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.casehub.ledger.api.model.KeyRotationReason;
import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.ledger.runtime.repository.KeyRotationRepository;

@Alternative
@Priority(1)
@ApplicationScoped
public class InMemoryKeyRotationRepository implements KeyRotationRepository {

    @Inject
    InMemoryLedgerEntryRepository blocking;

    @Override
    public List<KeyRotationEntry> findByActorId(final String actorId) {
        return blocking.entries.values().stream()
                .filter(e -> e instanceof KeyRotationEntry)
                .map(e -> (KeyRotationEntry) e)
                .filter(e -> actorId.equals(e.actorId))
                .sorted(Comparator.comparing(e -> e.occurredAt))
                .collect(Collectors.toList());
    }

    @Override
    public List<KeyRotationEntry> findCompromisedByActorIdAndKeyRef(
            final String actorId, final String keyRef) {
        return blocking.entries.values().stream()
                .filter(e -> e instanceof KeyRotationEntry)
                .map(e -> (KeyRotationEntry) e)
                .filter(e -> actorId.equals(e.actorId))
                .filter(e -> keyRef.equals(e.previousKeyRef))
                .filter(e -> KeyRotationReason.COMPROMISED.equals(e.reason))
                .sorted(Comparator.comparing(e -> e.effectiveSince))
                .collect(Collectors.toList());
    }
}
```

- [ ] **Step 4: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory \
  -Dtest=InMemoryKeyRotationRepositoryTest
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryKeyRotationRepository.java \
  persistence-memory/src/test/java/io/casehub/ledger/memory/InMemoryKeyRotationRepositoryTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#91): InMemoryKeyRotationRepository"
```

---

## Task 8: Reactive delegates

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryReactiveLedgerEntryRepository.java`
- Create: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryReactiveKeyRotationRepository.java`

- [ ] **Step 1: Implement `InMemoryReactiveLedgerEntryRepository`**

```java
package io.casehub.ledger.memory;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.ReactiveLedgerEntryRepository;
import io.quarkus.arc.properties.IfBuildProperty;
import io.smallrye.mutiny.Uni;

@Alternative
@Priority(1)
@ApplicationScoped
@IfBuildProperty(name = "casehub.ledger.reactive.enabled", stringValue = "true")
public class InMemoryReactiveLedgerEntryRepository implements ReactiveLedgerEntryRepository {

    @Inject
    InMemoryLedgerEntryRepository blocking;

    @Override
    public Uni<LedgerEntry> save(final LedgerEntry entry) {
        return Uni.createFrom().item(() -> blocking.save(entry));
    }

    @Override
    public Uni<List<LedgerEntry>> listAll() {
        return Uni.createFrom().item(blocking::listAll);
    }

    @Override
    public Uni<List<LedgerEntry>> findBySubjectId(final UUID subjectId) {
        return Uni.createFrom().item(() -> blocking.findBySubjectId(subjectId));
    }

    @Override
    public Uni<List<LedgerEntry>> findBySubjectIdAndTimeRange(final UUID subjectId,
            final Instant from, final Instant to) {
        return Uni.createFrom().item(() -> blocking.findBySubjectIdAndTimeRange(subjectId, from, to));
    }

    @Override
    public Uni<Optional<LedgerEntry>> findLatestBySubjectId(final UUID subjectId) {
        return Uni.createFrom().item(() -> blocking.findLatestBySubjectId(subjectId));
    }

    @Override
    public Uni<Optional<LedgerEntry>> findEntryById(final UUID id) {
        return Uni.createFrom().item(() -> blocking.findEntryById(id));
    }

    @Override
    public Uni<List<LedgerEntry>> findAllEvents() {
        return Uni.createFrom().item(blocking::findAllEvents);
    }

    @Override
    public Uni<List<LedgerEntry>> findByActorId(final String actorId,
            final Instant from, final Instant to) {
        return Uni.createFrom().item(() -> blocking.findByActorId(actorId, from, to));
    }

    @Override
    public Uni<List<LedgerEntry>> findByActorRole(final String actorRole,
            final Instant from, final Instant to) {
        return Uni.createFrom().item(() -> blocking.findByActorRole(actorRole, from, to));
    }

    @Override
    public Uni<List<LedgerEntry>> findByTimeRange(final Instant from, final Instant to) {
        return Uni.createFrom().item(() -> blocking.findByTimeRange(from, to));
    }

    @Override
    public Uni<List<LedgerEntry>> findCausedBy(final UUID entryId) {
        return Uni.createFrom().item(() -> blocking.findCausedBy(entryId));
    }

    @Override
    public Uni<LedgerAttestation> saveAttestation(final LedgerAttestation attestation) {
        return Uni.createFrom().item(() -> blocking.saveAttestation(attestation));
    }

    @Override
    public Uni<List<LedgerAttestation>> findAttestationsByEntryId(final UUID ledgerEntryId) {
        return Uni.createFrom().item(() -> blocking.findAttestationsByEntryId(ledgerEntryId));
    }

    @Override
    public Uni<Map<UUID, List<LedgerAttestation>>> findAttestationsForEntries(
            final Set<UUID> entryIds) {
        return Uni.createFrom().item(() -> blocking.findAttestationsForEntries(entryIds));
    }

    @Override
    public Uni<List<LedgerAttestation>> findAttestationsByEntryIdAndCapabilityTag(
            final UUID entryId, final String capabilityTag) {
        return Uni.createFrom().item(
                () -> blocking.findAttestationsByEntryIdAndCapabilityTag(entryId, capabilityTag));
    }

    @Override
    public Uni<List<LedgerAttestation>> findAttestationsByEntryIdGlobal(final UUID entryId) {
        return Uni.createFrom().item(() -> blocking.findAttestationsByEntryIdGlobal(entryId));
    }

    @Override
    public Uni<List<LedgerAttestation>> findAttestationsByAttestorIdAndCapabilityTag(
            final String attestorId, final String capabilityTag) {
        return Uni.createFrom().item(
                () -> blocking.findAttestationsByAttestorIdAndCapabilityTag(attestorId, capabilityTag));
    }
}
```

- [ ] **Step 2: Implement `InMemoryReactiveKeyRotationRepository`**

```java
package io.casehub.ledger.memory;

import java.util.List;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.casehub.ledger.runtime.model.KeyRotationEntry;
import io.casehub.ledger.runtime.repository.ReactiveKeyRotationRepository;
import io.quarkus.arc.properties.IfBuildProperty;
import io.smallrye.mutiny.Uni;

@Alternative
@Priority(1)
@ApplicationScoped
@IfBuildProperty(name = "casehub.ledger.reactive.enabled", stringValue = "true")
public class InMemoryReactiveKeyRotationRepository implements ReactiveKeyRotationRepository {

    @Inject
    InMemoryKeyRotationRepository blocking;

    @Override
    public Uni<List<KeyRotationEntry>> findByActorId(final String actorId) {
        return Uni.createFrom().item(() -> blocking.findByActorId(actorId));
    }

    @Override
    public Uni<List<KeyRotationEntry>> findCompromisedByActorIdAndKeyRef(
            final String actorId, final String keyRef) {
        return Uni.createFrom().item(() -> blocking.findCompromisedByActorIdAndKeyRef(actorId, keyRef));
    }
}
```

- [ ] **Step 3: Run all persistence-memory tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory
```

Expected: BUILD SUCCESS, all tests passing.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryReactiveLedgerEntryRepository.java \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryReactiveKeyRotationRepository.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#91): reactive in-memory delegates (build-gated)"
```

---

## Task 9: Full build + platform docs

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Run the full build across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: BUILD SUCCESS across api, runtime, deployment, persistence-memory.

- [ ] **Step 2: Update `PLATFORM.md` — Capability Ownership table**

In `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`, find the row:
```
| Immutable entry chain (Merkle Mountain Range) | `casehub-ledger` | Domain-agnostic; consumers extend `LedgerEntry` via JPA JOINED |
```
Add a new row immediately after it:
```
| In-memory persistence (zero datasource / ephemeral install) | `casehub-ledger` | `casehub-ledger-memory` — `@Alternative @Priority(1)` impls of all persistence SPIs; add as compile dep for `@QuarkusTest` isolation |
```

Also update the Repository Map entry for `casehub-ledger`:
```
| `casehub-ledger` | [casehubio/ledger](https://github.com/casehubio/ledger) | Immutable tamper-evident audit ledger + trust scoring. Modules: `api`, `runtime`, `deployment`, `persistence-memory` (`casehub-ledger-memory` — zero-datasource in-memory SPIs) | Foundation |
```

- [ ] **Step 3: Update `CLAUDE.md` — project structure section**

In `/Users/mdproctor/claude/casehub/ledger/CLAUDE.md`, add `persistence-memory/` to the project structure tree after `deployment/`:

```
└── persistence-memory/
    └── src/main/java/io/casehub/ledger/memory/
        ├── InMemoryLedgerEntryRepository.java        — @Alternative @Priority(1); save pipeline mirrors JPA; entries field package-private
        ├── InMemoryLedgerMerkleFrontierRepository.java — @Alternative @Priority(1); ConcurrentHashMap-backed
        ├── InMemoryActorTrustScoreRepository.java    — @Alternative @Priority(1); composite key: actorId|scoreType|cap|dim
        ├── InMemoryKeyRotationRepository.java        — @Alternative @Priority(1); reads from InMemoryLedgerEntryRepository
        ├── InMemoryReactiveLedgerEntryRepository.java — @IfBuildProperty(reactive.enabled=true); delegates to blocking
        └── InMemoryReactiveKeyRotationRepository.java — @IfBuildProperty(reactive.enabled=true); delegates to blocking
```

Also add to the Maven Coordinates table:
```
| persistence-memory artifactId | `casehub-ledger-memory` |
```

- [ ] **Step 4: Commit platform doc changes**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(#91): add casehub-ledger-memory to capability ownership"
git -C /Users/mdproctor/claude/casehub/ledger add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/ledger commit -m "docs(#91): add persistence-memory to project structure"
```

---

## Self-Review

**Spec coverage check:**
- ✅ `LedgerEnricherPipeline` extracted (Task 1)
- ✅ `LedgerMerkleFrontierRepository` SPI + JPA impl (Task 2)
- ✅ `LedgerVerificationService` refactored to use SPI (Task 2)
- ✅ `JpaLedgerEntryRepository` refactored to use SPI (Task 2)
- ✅ Module scaffolding + pom (Task 3)
- ✅ `InMemoryLedgerMerkleFrontierRepository` (Task 4)
- ✅ `InMemoryLedgerEntryRepository` (Task 5)
- ✅ `InMemoryActorTrustScoreRepository` (Task 6)
- ✅ `InMemoryKeyRotationRepository` (Task 7)
- ✅ `InMemoryReactiveLedgerEntryRepository` build-gated (Task 8)
- ✅ `InMemoryReactiveKeyRotationRepository` build-gated (Task 8)
- ✅ Platform docs (Task 9)
- ✅ `clear()` on all stores (in each implementation)
- ✅ `@BeforeEach repo.clear()` in all tests (GE-20260518-896005)

**Type consistency:** All SPI method signatures match exactly. `KeyRotationEntry.effectiveSince` used in `findCompromisedByActorIdAndKeyRef` sort — verify field name at implementation time against the actual class (`KeyRotationEntry` has `effectiveSince` per CLAUDE.md).

**Placeholder scan:** None found. All steps contain complete code.

**Note on H2 datasource in tests:** The persistence-memory `@QuarkusTest` uses H2 to satisfy the ledger extension's Hibernate ORM setup. No JPA operations execute — all persistence routes through in-memory stores. If startup fails due to entity scanning, add `quarkus.hibernate-orm.packages=io.casehub.ledger.runtime.model` to scope entity discovery, or switch to plain JUnit 5 (non-QuarkusTest) for the store tests.
