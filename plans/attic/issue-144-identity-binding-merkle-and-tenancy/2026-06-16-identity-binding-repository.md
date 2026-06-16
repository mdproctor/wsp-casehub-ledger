# Identity Binding Repository — Merkle Frontier + Tenancy Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove `save()` from `ActorIdentityBindingRepository` and route binding entry persistence through `LedgerEntryRepository.save()`, mirroring the established `KeyRotationRepository` pattern — fixing missing Merkle frontier updates (#144) and adding `tenancyId` to read methods (#145).

**Architecture:** `ActorIdentityBindingRepository` becomes a read-only SPI (two methods, no save). `ActorIdentityBindingObserver` injects `LedgerEntryRepository` and calls `ledgerRepo.save(entry, tenancyId)` — the full pipeline (sequence allocation, hash, Merkle frontier update, enrichers) runs automatically. A guard in `ActorDIDEnricher` (`instanceof ActorIdentityBindingEntry`) prevents DID resolution on binding entries, which eliminates the event loop unconditionally and prevents `boundDid`/`actorDid` discrepancy. `InMemoryActorIdentityBindingRepository` delegates reads to `InMemoryLedgerEntryRepository.allEntries()`, mirroring `InMemoryKeyRotationRepository`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (Hibernate ORM), H2 (tests), CDI, AssertJ, Awaitility, JUnit 5 / `@QuarkusTest`

**Spec:** `specs/2026-06-16-identity-binding-repository-design.md`

---

## File Map

| Action | File |
|--------|------|
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorIdentityBindingRepository.java` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/NoOpActorIdentityBindingRepository.java` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorIdentityBindingEntry.java` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorIdentityBindingRepository.java` |
| Modify | `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryActorIdentityBindingRepository.java` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorIdentityBindingObserver.java` |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorDIDEnricher.java` |
| Modify | `runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityBindingEntryIT.java` |
| Modify | `runtime/src/test/java/io/casehub/ledger/service/identity/NoOpActorIdentityBindingRepositoryIT.java` |
| Modify | `runtime/src/test/resources/application.properties` |
| Create | `runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityBindingTenancyIT.java` |
| Create | `runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityBindingMerkleIT.java` |

---

## Task 1: Break the SPI — remove `save()`, add `tenancyId`

This is the anchor change. Everything downstream in Tasks 2–6 restores compilation. Do not run tests between Task 1 and Task 6.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorIdentityBindingRepository.java`

- [ ] **Replace the entire file with the new read-only SPI:**

```java
package io.casehub.ledger.runtime.repository;

import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
import java.util.List;
import java.util.Optional;

/**
 * SPI for querying {@link ActorIdentityBindingEntry} records.
 *
 * <p>Binding entries are persisted via {@link io.casehub.ledger.runtime.repository.LedgerEntryRepository#save},
 * which ensures Merkle chain inclusion, sequence allocation, enricher pipeline execution,
 * and agent signing. {@code ActorIdentityBindingObserver} is the sole writer.
 *
 * <p>Implementations must filter results by both {@code actorId} and {@code tenancyId} —
 * two tenants can share the same {@code actorId} (e.g. a shared LLM persona like
 * {@code claude:reviewer@v1}); returning cross-tenant results is a data isolation violation.
 */
public interface ActorIdentityBindingRepository {

    /**
     * Return the most recent binding entry for the given actor and tenant,
     * ordered by {@code sequenceNumber} descending.
     */
    Optional<ActorIdentityBindingEntry> latestBindingFor(String actorId, String tenancyId);

    /**
     * Return all binding entries for the given actor and tenant,
     * ordered by {@code sequenceNumber} ascending.
     */
    List<ActorIdentityBindingEntry> bindingHistoryFor(String actorId, String tenancyId);
}
```

---

## Task 2: Fix `NoOpActorIdentityBindingRepository`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/NoOpActorIdentityBindingRepository.java`

- [ ] **Replace the entire file:**

```java
package io.casehub.ledger.runtime.repository;

import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.List;
import java.util.Optional;

/**
 * No-op {@link ActorIdentityBindingRepository} — satisfies the CDI injection point when
 * neither {@code JpaActorIdentityBindingRepository} nor the in-memory alternative is active.
 *
 * <p>Activation priority (lowest to highest):
 * <ol>
 * <li>This {@code @DefaultBean} — active when nothing else is present</li>
 * <li>{@code JpaActorIdentityBindingRepository @Alternative} — activate via
 *     {@code quarkus.arc.selected-alternatives}</li>
 * <li>{@code InMemoryActorIdentityBindingRepository @Alternative @Priority(1)} —
 *     active when {@code casehub-ledger-memory} is on the classpath</li>
 * </ol>
 *
 * <p>Writes go through {@code LedgerEntryRepository} regardless of which implementation
 * is active here. This bean governs read access only.
 */
@DefaultBean
@ApplicationScoped
public class NoOpActorIdentityBindingRepository implements ActorIdentityBindingRepository {

    @Override
    public Optional<ActorIdentityBindingEntry> latestBindingFor(final String actorId,
            final String tenancyId) {
        return Optional.empty();
    }

    @Override
    public List<ActorIdentityBindingEntry> bindingHistoryFor(final String actorId,
            final String tenancyId) {
        return List.of();
    }
}
```

---

## Task 3: Update named queries on `ActorIdentityBindingEntry`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorIdentityBindingEntry.java`

- [ ] **Replace the two `@NamedQuery` annotations at the top of the class** (lines 34–41):

```java
@NamedQueries({
    @NamedQuery(
        name = "ActorIdentityBindingEntry.findLatestByActorId",
        query = "SELECT e FROM ActorIdentityBindingEntry e "
              + "WHERE e.actorId = :actorId AND e.tenancyId = :tenancyId "
              + "ORDER BY e.sequenceNumber DESC"),
    @NamedQuery(
        name = "ActorIdentityBindingEntry.findHistoryByActorId",
        query = "SELECT e FROM ActorIdentityBindingEntry e "
              + "WHERE e.actorId = :actorId AND e.tenancyId = :tenancyId "
              + "ORDER BY e.sequenceNumber ASC")
})
```

No other changes to this file.

---

## Task 4: Simplify `JpaActorIdentityBindingRepository`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorIdentityBindingRepository.java`

- [ ] **Replace the entire file:**

```java
package io.casehub.ledger.runtime.repository.jpa;

import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.repository.ActorIdentityBindingRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;

import java.util.List;
import java.util.Optional;

/**
 * JPA implementation of {@link ActorIdentityBindingRepository}.
 *
 * <p>Read-only — saves go through {@link io.casehub.ledger.runtime.repository.LedgerEntryRepository#save}.
 *
 * <p>Activate via {@code quarkus.arc.selected-alternatives=...JpaActorIdentityBindingRepository}.
 */
@Alternative
@ApplicationScoped
public class JpaActorIdentityBindingRepository implements ActorIdentityBindingRepository {

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @Override
    public Optional<ActorIdentityBindingEntry> latestBindingFor(final String actorId,
            final String tenancyId) {
        return em.createNamedQuery("ActorIdentityBindingEntry.findLatestByActorId",
                    ActorIdentityBindingEntry.class)
                .setParameter("actorId", actorId)
                .setParameter("tenancyId", tenancyId)
                .setMaxResults(1)
                .getResultStream()
                .findFirst();
    }

    @Override
    public List<ActorIdentityBindingEntry> bindingHistoryFor(final String actorId,
            final String tenancyId) {
        return em.createNamedQuery("ActorIdentityBindingEntry.findHistoryByActorId",
                    ActorIdentityBindingEntry.class)
                .setParameter("actorId", actorId)
                .setParameter("tenancyId", tenancyId)
                .getResultList();
    }
}
```

---

## Task 5: Replace `InMemoryActorIdentityBindingRepository`

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryActorIdentityBindingRepository.java`

- [ ] **Replace the entire file:**

```java
package io.casehub.ledger.memory;

import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
import io.casehub.ledger.runtime.repository.ActorIdentityBindingRepository;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;

/**
 * In-memory {@link ActorIdentityBindingRepository}.
 *
 * <p>Read-only — delegates to {@link InMemoryLedgerEntryRepository#allEntries()} filtered by
 * type, actorId, and tenancyId. Saves go through {@link InMemoryLedgerEntryRepository#save},
 * which assigns {@code sequenceNumber}, computes {@code digest}, and updates the Merkle frontier.
 *
 * <p>Mirrors the {@code InMemoryKeyRotationRepository} pattern.
 */
@ApplicationScoped
@Alternative
@Priority(1)
public class InMemoryActorIdentityBindingRepository implements ActorIdentityBindingRepository {

    @Inject
    InMemoryLedgerEntryRepository blocking;

    @Override
    public Optional<ActorIdentityBindingEntry> latestBindingFor(final String actorId,
            final String tenancyId) {
        return blocking.allEntries().stream()
            .filter(e -> e instanceof ActorIdentityBindingEntry)
            .map(e -> (ActorIdentityBindingEntry) e)
            .filter(e -> actorId.equals(e.actorId) && tenancyId.equals(e.tenancyId))
            .max(Comparator.comparingInt(e -> e.sequenceNumber));
    }

    @Override
    public List<ActorIdentityBindingEntry> bindingHistoryFor(final String actorId,
            final String tenancyId) {
        return blocking.allEntries().stream()
            .filter(e -> e instanceof ActorIdentityBindingEntry)
            .map(e -> (ActorIdentityBindingEntry) e)
            .filter(e -> actorId.equals(e.actorId) && tenancyId.equals(e.tenancyId))
            .sorted(Comparator.comparingInt(e -> e.sequenceNumber))
            .toList();
    }
}
```

---

## Task 6: Reroute `ActorIdentityBindingObserver`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorIdentityBindingObserver.java`

- [ ] **Replace the entire file:**

```java
package io.casehub.ledger.runtime.service.identity;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.AgentIdentityValidatedEvent;
import io.casehub.platform.api.identity.AgentIdentityViolationEvent;
import io.casehub.platform.api.identity.CredentialValidationResult;
import io.casehub.platform.api.identity.IdentityBindingStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;

import static jakarta.transaction.Transactional.TxType.REQUIRES_NEW;

/**
 * Persists {@link ActorIdentityBindingEntry} in response to async identity validation events.
 *
 * <p>Routes through {@link LedgerEntryRepository#save} — the full save pipeline runs:
 * sequence allocation, hash computation, Merkle frontier update, enricher pipeline, agent signing.
 *
 * <p>Uses {@code REQUIRES_NEW} so the binding entry commits in its own transaction,
 * independent of the parent entry's lifecycle. Failure is logged and swallowed.
 */
@ApplicationScoped
public class ActorIdentityBindingObserver {

    private static final Logger LOG = Logger.getLogger(ActorIdentityBindingObserver.class);

    @Inject
    LedgerEntryRepository ledgerRepo;

    void onValidated(@ObservesAsync final AgentIdentityValidatedEvent event) {
        persistBinding(
            event.tenancyId(), event.actorId(), event.actorDid(), event.status(),
            event.alsoKnownAsVerified(), event.keyMatchVerified(),
            event.verifiedKeyRef(), event.credentialResult(), event.didMethod());
    }

    void onViolation(@ObservesAsync final AgentIdentityViolationEvent event) {
        persistBinding(
            event.tenancyId(), event.actorId(), event.actorDid(), event.status(),
            false, false, null, null, extractDidMethod(event.actorDid()));
    }

    @Transactional(REQUIRES_NEW)
    void persistBinding(
            final String tenancyId,
            final String actorId,
            final String actorDid,
            final IdentityBindingStatus status,
            final boolean alsoKnownAsVerified,
            final boolean keyMatchVerified,
            final String verifiedKeyRef,
            final CredentialValidationResult credentialResult,
            final String didMethod) {
        try {
            final ActorIdentityBindingEntry entry = new ActorIdentityBindingEntry();
            entry.id = UUID.randomUUID();
            entry.subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(StandardCharsets.UTF_8));
            entry.actorId = actorId;
            entry.actorType = io.casehub.platform.api.identity.ActorType.AGENT;
            entry.actorRole = "identity-binding";
            entry.entryType = LedgerEntryType.EVENT;
            entry.occurredAt = Instant.now();
            entry.boundDid = actorDid;
            entry.validationResult = status;
            entry.alsoKnownAsVerified = alsoKnownAsVerified;
            entry.keyMatchVerified = keyMatchVerified;
            entry.verifiedKeyRef = verifiedKeyRef;
            entry.credentialResult = credentialResult;
            entry.didMethod = didMethod;
            // tenancyId is set by ledgerRepo.save() — do not set entry.tenancyId here
            ledgerRepo.save(entry, tenancyId);
        } catch (final Exception e) {
            LOG.warnf("ActorIdentityBindingObserver failed to persist binding for %s: %s",
                actorId, e.getMessage());
        }
    }

    private String extractDidMethod(final String did) {
        if (did == null || !did.startsWith("did:")) return null;
        final String[] parts = did.split(":", 3);
        return parts.length > 1 ? parts[1] : null;
    }
}
```

---

## Task 7: Add `ActorDIDEnricher` guard

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorDIDEnricher.java`

- [ ] **Replace the `enrich()` method body only** (lines 28–35):

```java
@Override
public void enrich(final LedgerEntry entry) {
    if (entry.actorId == null || entry.actorDid != null) return;
    if (entry instanceof ActorIdentityBindingEntry) return;
    try {
        provider.didFor(entry.actorId).ifPresent(did -> entry.actorDid = did);
    } catch (final Exception e) {
        LOG.warnf("ActorDIDEnricher failed for actor %s: %s", entry.actorId, e.getMessage());
    }
}
```

Add the import at the top of the file:
```java
import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
```

---

## Task 8: Compile check

- [ ] **Verify everything compiles:**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime,persistence-memory -am -q
```

Expected: `BUILD SUCCESS`. No errors.

If there are compilation errors, they are from other files that reference the old `save()` or the old `latestBindingFor(String)` / `bindingHistoryFor(String)` signatures. Fix them before continuing.

---

## Task 9: Update `ActorIdentityBindingEntryIT`

The two helper methods call the old single-argument signatures. Add `DEFAULT_TENANT_ID` as the second argument.

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityBindingEntryIT.java`

- [ ] **Replace `readLatestBinding()` and `readBindingHistory()` helper methods** (lines 184–193):

```java
/** Reads latest binding in its own transaction. */
private java.util.Optional<ActorIdentityBindingEntry> readLatestBinding(final String actorId) {
    return QuarkusTransaction.requiringNew()
        .call(() -> bindingRepo.latestBindingFor(actorId, DEFAULT_TENANT_ID));
}

/** Reads binding history in its own transaction. */
private List<ActorIdentityBindingEntry> readBindingHistory(final String actorId) {
    return QuarkusTransaction.requiringNew()
        .call(() -> bindingRepo.bindingHistoryFor(actorId, DEFAULT_TENANT_ID));
}
```

- [ ] **Run existing binding tests to confirm they still pass:**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=ActorIdentityBindingEntryIT -q
```

Expected: `Tests run: 3, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorIdentityBindingRepository.java \
  runtime/src/main/java/io/casehub/ledger/runtime/repository/NoOpActorIdentityBindingRepository.java \
  runtime/src/main/java/io/casehub/ledger/runtime/model/ActorIdentityBindingEntry.java \
  runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorIdentityBindingRepository.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorIdentityBindingObserver.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorDIDEnricher.java \
  runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityBindingEntryIT.java
git -C /Users/mdproctor/claude/casehub/ledger add \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryActorIdentityBindingRepository.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "$(cat <<'EOF'
feat(#144,#145): route ActorIdentityBindingEntry through LedgerEntryRepository

- Remove save() from ActorIdentityBindingRepository SPI — now read-only, mirrors KeyRotationRepository
- ActorIdentityBindingObserver: inject LedgerEntryRepository, call ledgerRepo.save(entry, tenancyId)
- ActorDIDEnricher: guard binding entries (instanceof) — prevents event loop and boundDid/actorDid discrepancy
- JpaActorIdentityBindingRepository: remove save() + supporting injections; read-only
- InMemoryActorIdentityBindingRepository: delegate reads to InMemoryLedgerEntryRepository.allEntries()
- Named queries: add tenancyId filter (Refs #145), switch ordering to sequenceNumber (Refs #144, #145)
- NoOpActorIdentityBindingRepository: remove save(), update read signatures

Refs #144, #145
EOF
)"
```

---

## Task 10: Repurpose `NoOpActorIdentityBindingRepositoryIT`

The test previously asserted that no rows are written. After our change, binding entries ARE written via `JpaLedgerEntryRepository` — the assertion inverts. The test now validates the read/write split: writes don't require `JpaActorIdentityBindingRepository` in `selected-alternatives`, but reads via the SPI return empty (no-op active).

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/identity/NoOpActorIdentityBindingRepositoryIT.java`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Replace `NoOpActorIdentityBindingRepositoryIT.java`:**

```java
package io.casehub.ledger.service.identity;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.when;

import java.time.Duration;
import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.repository.ActorIdentityBindingRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.IdentityBindingStatus;
import io.casehub.ledger.runtime.service.AgentSigner;
import io.casehub.ledger.runtime.service.identity.ActorIdentityValidationEnricher;
import io.casehub.ledger.service.supplement.TestEntry;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

import static io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;

/**
 * Verifies the read/write split introduced by routing binding entry saves through
 * {@code LedgerEntryRepository} rather than {@code ActorIdentityBindingRepository}.
 *
 * <p>The {@code noop-test} profile selects {@code JpaLedgerEntryRepository} and
 * {@code JpaLedgerMerkleFrontierRepository} but deliberately omits
 * {@code JpaActorIdentityBindingRepository}.
 *
 * <p>Write path: {@code ActorIdentityBindingObserver} calls {@code ledgerRepo.save()} →
 * {@code JpaLedgerEntryRepository} → binding entry IS written to {@code actor_identity_binding}.
 * Writes do NOT require {@code JpaActorIdentityBindingRepository} in selected-alternatives.
 *
 * <p>Read path: {@code ActorIdentityBindingRepository} resolves to {@code @DefaultBean}
 * no-op → {@code latestBindingFor()} returns empty. Reads require {@code JpaActorIdentityBindingRepository}
 * to be in selected-alternatives.
 */
@QuarkusTest
@TestProfile(NoOpActorIdentityBindingRepositoryIT.Profile.class)
class NoOpActorIdentityBindingRepositoryIT {

    public static class Profile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "noop-test";
        }
    }

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    ActorIdentityBindingRepository bindingRepo;

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @Inject
    ActorIdentityValidationEnricher identityEnricher;

    @InjectMock
    AgentSigner agentSigner;

    @BeforeEach
    void setUp() {
        identityEnricher.invalidateAll();
        when(agentSigner.sign(anyString(), any())).thenReturn(Optional.empty());
    }

    /**
     * Saves an entry with {@code actorDid} set. The enricher fires
     * {@code AgentIdentityViolationEvent} (DID unresolvable — no resolver registered).
     * The observer calls {@code ledgerRepo.save(bindingEntry)} — the JPA ledger repo
     * writes the row, even though {@code JpaActorIdentityBindingRepository} is absent.
     *
     * <p>Assert 1: row is written (write path does not need {@code JpaActorIdentityBindingRepository}).
     * Assert 2: {@code bindingRepo.latestBindingFor()} returns empty (no-op read is active).
     */
    @Test
    void writePathDoesNotNeedJpaBindingRepo_readPathRequiresIt() {
        final String actorId = "claude:noop-test-" + UUID.randomUUID();

        final LedgerEntry[] saved = new LedgerEntry[1];
        QuarkusTransaction.requiringNew().run(() -> {
            final TestEntry e = new TestEntry();
            e.subjectId = UUID.randomUUID();
            e.entryType = LedgerEntryType.EVENT;
            e.actorId = actorId;
            e.actorType = ActorType.AGENT;
            e.actorRole = "noop-it";
            e.actorDid = "did:web:noop-test.example.com";
            saved[0] = ledgerRepo.save(e, DEFAULT_TENANT_ID);
        });

        assertThat(saved[0].pendingIdentityStatus).isEqualTo(IdentityBindingStatus.DID_UNRESOLVABLE);

        // Assert 1: binding entry IS written via JpaLedgerEntryRepository
        await().during(Duration.ofMillis(500)).atMost(Duration.ofSeconds(3))
            .untilAsserted(() -> {
                final Long count = QuarkusTransaction.requiringNew().call(() ->
                    (Long) em.createNativeQuery(
                        "SELECT COUNT(*) FROM actor_identity_binding aib " +
                        "JOIN ledger_entry le ON le.id = aib.id " +
                        "WHERE le.actor_id = :id")
                        .setParameter("id", actorId)
                        .getSingleResult()
                );
                assertThat(count).isPositive();
            });

        // Assert 2: SPI read via @DefaultBean no-op returns empty
        final Optional<ActorIdentityBindingEntry> viaRepo = QuarkusTransaction.requiringNew()
            .call(() -> bindingRepo.latestBindingFor(actorId, DEFAULT_TENANT_ID));
        assertThat(viaRepo).isEmpty();
    }
}
```

- [ ] **Update the noop-test comment in `application.properties`** (lines 191–201). Replace those lines with:

```properties
# No-op repository test profile (used by NoOpActorIdentityBindingRepositoryIT)
# Isolated DB — verifies the read/write split: binding entries ARE written via JpaLedgerEntryRepository
# even when JpaActorIdentityBindingRepository is absent from selected-alternatives (write path
# uses LedgerEntryRepository only). latestBindingFor() returns empty because NoOpActorIdentityBindingRepository
# @DefaultBean is the active read implementation.
%noop-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:nooptestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%noop-test.quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository
```

- [ ] **Run the repurposed test:**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=NoOpActorIdentityBindingRepositoryIT -q
```

Expected: `Tests run: 1, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/test/java/io/casehub/ledger/service/identity/NoOpActorIdentityBindingRepositoryIT.java \
  runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test(#144,#145): repurpose noop-test — assert write/read split, invert count assertion"
```

---

## Task 11: New cross-tenant isolation test

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityBindingTenancyIT.java`

- [ ] **Create the file:**

```java
package io.casehub.ledger.service.identity;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.when;

import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
import io.casehub.ledger.runtime.repository.ActorIdentityBindingRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.AgentSigner;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.CredentialValidationResult;
import io.casehub.platform.api.identity.IdentityBindingStatus;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

/**
 * Verifies per-tenant isolation in {@link ActorIdentityBindingRepository} read methods.
 *
 * <p>Two tenants can share the same {@code actorId} — e.g. a shared LLM persona like
 * {@code claude:reviewer@v1}. Binding entries from tenant A must not appear in queries
 * scoped to tenant B, and vice versa.
 *
 * <p>Uses an isolated DB to prevent cross-test contamination from REQUIRES_NEW commits.
 */
@QuarkusTest
@TestProfile(ActorIdentityBindingTenancyIT.TenancyProfile.class)
class ActorIdentityBindingTenancyIT {

    public static class TenancyProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "identity-binding-tenancy-test";
        }
    }

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    ActorIdentityBindingRepository bindingRepo;

    @InjectMock
    AgentSigner agentSigner;

    @BeforeEach
    void setUp() {
        when(agentSigner.sign(anyString(), any())).thenReturn(Optional.empty());
    }

    @Test
    void latestBindingForIsIsolatedByTenant() {
        final String actorId = "claude:tenancy-isolation-" + UUID.randomUUID();
        final String tenantA = "tenant-a-" + UUID.randomUUID();
        final String tenantB = "tenant-b-" + UUID.randomUUID();

        final ActorIdentityBindingEntry entryA = buildBindingEntry(actorId,
                "did:web:test-a.example.com", IdentityBindingStatus.VALID);
        QuarkusTransaction.requiringNew().run(() -> ledgerRepo.save(entryA, tenantA));

        // tenantA can read their own entry
        final Optional<ActorIdentityBindingEntry> forA = QuarkusTransaction.requiringNew()
            .call(() -> bindingRepo.latestBindingFor(actorId, tenantA));
        assertThat(forA).isPresent();
        assertThat(forA.get().tenancyId).isEqualTo(tenantA);

        // tenantB sees nothing — same actorId, different tenant
        final Optional<ActorIdentityBindingEntry> forB = QuarkusTransaction.requiringNew()
            .call(() -> bindingRepo.latestBindingFor(actorId, tenantB));
        assertThat(forB).isEmpty();
    }

    @Test
    void bindingHistoryForIsIsolatedByTenant() {
        final String actorId = "claude:history-isolation-" + UUID.randomUUID();
        final String tenantA = "tenant-ha-" + UUID.randomUUID();
        final String tenantB = "tenant-hb-" + UUID.randomUUID();

        final ActorIdentityBindingEntry e1 = buildBindingEntry(actorId,
                "did:web:history-a.example.com", IdentityBindingStatus.VALID);
        final ActorIdentityBindingEntry e2 = buildBindingEntry(actorId,
                "did:web:history-b.example.com", IdentityBindingStatus.DID_UNRESOLVABLE);
        QuarkusTransaction.requiringNew().run(() -> {
            ledgerRepo.save(e1, tenantA);
            ledgerRepo.save(e2, tenantB);
        });

        // tenantA sees exactly 1 entry
        assertThat(QuarkusTransaction.requiringNew()
            .call(() -> bindingRepo.bindingHistoryFor(actorId, tenantA))).hasSize(1);
        // tenantB sees exactly 1 entry
        assertThat(QuarkusTransaction.requiringNew()
            .call(() -> bindingRepo.bindingHistoryFor(actorId, tenantB))).hasSize(1);
    }

    private static ActorIdentityBindingEntry buildBindingEntry(
            final String actorId, final String did, final IdentityBindingStatus status) {
        final ActorIdentityBindingEntry e = new ActorIdentityBindingEntry();
        e.id = UUID.randomUUID();
        e.subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(StandardCharsets.UTF_8));
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.actorRole = "tenancy-test";
        e.entryType = LedgerEntryType.EVENT;
        e.occurredAt = Instant.now();
        e.boundDid = did;
        e.validationResult = status;
        e.alsoKnownAsVerified = status == IdentityBindingStatus.VALID;
        e.keyMatchVerified = status == IdentityBindingStatus.VALID;
        return e;
    }
}
```

- [ ] **Add the test profile to `application.properties`** (append at the end of the file):

```properties
# Identity binding tenancy test profile (used by ActorIdentityBindingTenancyIT)
# Isolated DB — verifies per-tenant isolation in latestBindingFor() and bindingHistoryFor().
%identity-binding-tenancy-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:identitybindingtenancytestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
```

- [ ] **Run the new test:**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=ActorIdentityBindingTenancyIT -q
```

Expected: `Tests run: 2, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityBindingTenancyIT.java \
  runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test(#145): cross-tenant isolation for latestBindingFor and bindingHistoryFor"
```

---

## Task 12: New Merkle proof coverage test

This is the #144 regression guard. It verifies that the Merkle frontier is correctly maintained for `ActorIdentityBindingEntry` subjects — that `sequenceNumber` and `digest` are set, and that `LedgerVerificationService.verify()` returns `true`.

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityBindingMerkleIT.java`

- [ ] **Create the file:**

```java
package io.casehub.ledger.service.identity;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.when;

import java.nio.charset.StandardCharsets;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.util.List;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.ActorIdentityBindingEntry;
import io.casehub.ledger.runtime.repository.ActorIdentityBindingRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.AgentSignature;
import io.casehub.ledger.runtime.service.AgentSigner;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.identity.ActorIdentityValidationEnricher;
import io.casehub.ledger.service.supplement.TestEntry;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.DIDDocument;
import io.casehub.platform.api.identity.VerificationMethod;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

import static io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;

/**
 * Regression guard for #144: verifies that saving an {@link ActorIdentityBindingEntry}
 * correctly updates the Merkle frontier so that {@link LedgerVerificationService#verify}
 * returns {@code true}.
 *
 * <p>Before the fix: {@code JpaActorIdentityBindingRepository.save()} never called
 * {@code frontierRepo.replace()}, leaving the frontier empty. {@code verify()} threw
 * {@code IllegalStateException} from the internal {@code treeRoot()} call.
 *
 * <p>After the fix: saves route through {@code JpaLedgerEntryRepository.save()}, which
 * updates the frontier. {@code verify()} returns {@code true}.
 */
@QuarkusTest
@TestProfile(ActorIdentityBindingMerkleIT.MerkleProfile.class)
class ActorIdentityBindingMerkleIT {

    public static class MerkleProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "identity-binding-merkle-test";
        }
    }

    @Inject LedgerEntryRepository ledgerRepo;
    @Inject ActorIdentityBindingRepository bindingRepo;
    @Inject LedgerVerificationService verificationService;
    @Inject ActorIdentityValidationEnricher identityEnricher;
    @Inject InjectableTestDIDResolver testResolver;

    @InjectMock
    AgentSigner agentSigner;

    private KeyPair keyPair;

    @BeforeEach
    void setUp() throws Exception {
        identityEnricher.invalidateAll();
        testResolver.clear();
        keyPair = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        when(agentSigner.sign(anyString(), any())).thenReturn(Optional.empty());
    }

    @Test
    void bindingEntryMaintainsMerkleFrontier() throws Exception {
        final String actorId = "claude:merkle-binding-" + UUID.randomUUID();
        final String did = "did:web:merkle-test.example.com";

        // Configure a resolvable DID so ActorIdentityValidationEnricher fires AgentIdentityValidatedEvent
        final byte[] pubKey = keyPair.getPublic().getEncoded();
        final var vm = new VerificationMethod(did + "#key-1", "Ed25519", pubKey);
        final var doc = new DIDDocument(did, List.of(vm), List.of(actorId));
        testResolver.register(did, doc);
        when(agentSigner.sign(eq(actorId), any()))
            .thenAnswer(inv -> Optional.of(
                AgentSignature.signWith(keyPair, inv.getArgument(1))));

        // Save a normal entry with actorDid set — this triggers the identity validation pipeline
        QuarkusTransaction.requiringNew().run(() -> {
            final TestEntry e = new TestEntry();
            e.subjectId = UUID.randomUUID();
            e.entryType = LedgerEntryType.EVENT;
            e.actorId = actorId;
            e.actorType = ActorType.AGENT;
            e.actorRole = "merkle-it";
            e.actorDid = did;
            ledgerRepo.save(e, DEFAULT_TENANT_ID);
        });

        // The binding entry's subjectId is deterministic: nameUUIDFromBytes(actorId)
        final UUID bindingSubjectId = UUID.nameUUIDFromBytes(
            actorId.getBytes(StandardCharsets.UTF_8));

        // Wait for the async observer to commit the binding entry
        await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            final Optional<ActorIdentityBindingEntry> binding = QuarkusTransaction.requiringNew()
                .call(() -> bindingRepo.latestBindingFor(actorId, DEFAULT_TENANT_ID));
            assertThat(binding).isPresent();
            assertThat(binding.get().sequenceNumber).isPositive();
            assertThat(binding.get().digest).isNotNull();
        });

        // verify() recomputes all leaf hashes for bindingSubjectId and compares against
        // the stored Merkle frontier. Returns true only when the frontier was correctly
        // updated during save(). Throws IllegalStateException if the frontier is empty.
        final boolean verified = QuarkusTransaction.requiringNew()
            .call(() -> verificationService.verify(bindingSubjectId, DEFAULT_TENANT_ID));
        assertThat(verified).isTrue();
    }
}
```

- [ ] **Add the test profile to `application.properties`** (append at the end):

```properties
# Identity binding Merkle test profile (used by ActorIdentityBindingMerkleIT)
# Isolated DB — verifies that binding entries maintain the Merkle frontier (#144 regression guard).
%identity-binding-merkle-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:identitybindingmerkletestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
```

- [ ] **Run the new test:**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=ActorIdentityBindingMerkleIT -q
```

Expected: `Tests run: 1, Failures: 0, Errors: 0, Skipped: 0`

- [ ] **Commit:**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityBindingMerkleIT.java \
  runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test(#144): Merkle proof coverage for ActorIdentityBindingEntry subjects"
```

---

## Task 13: Full test suite

- [ ] **Run all tests (H2 only — no Docker required):**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```

Expected: All tests pass. The output line to look for:
```
[INFO] Tests run: NNN, Failures: 0, Errors: 0, Skipped: 0
```

Note: Two test classes require Docker (PostgreSQL via Testcontainers): `JpaSequenceNumberPgIT` and `LedgerHealthJobPgIT`. Skip them if Docker is not available by running only the H2 tests:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test \
  -Dtest='!JpaSequenceNumberPgIT,!LedgerHealthJobPgIT'
```

- [ ] **Commit the final squash message or tag if all green:**

No additional commit needed — all changes are already committed in Tasks 9, 10, 11, 12.
