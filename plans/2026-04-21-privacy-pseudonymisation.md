# Privacy / Pseudonymisation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement pluggable actor-identity pseudonymisation and decision-context sanitisation so any organisation can satisfy GDPR Art.17 (right to erasure) without breaking the immutable Merkle hash chain.

**Architecture:** Two CDI SPIs (`ActorIdentityProvider`, `DecisionContextSanitiser`) are injected into `JpaLedgerEntryRepository`. Defaults are pass-through — existing consumers see zero behaviour change. A built-in `InternalActorIdentityProvider` (config-gated via `quarkus.ledger.identity.tokenisation.enabled=true`) stores UUID tokens in a new `actor_identity` table and provides erasure via `LedgerErasureService`. A `LedgerPrivacyProducer` selects the right implementation at startup based on config. Custom beans replace the defaults via standard CDI `@DefaultBean` semantics.

**Tech Stack:** Java 21, Quarkus 3.32.2 Arc, JPA/Hibernate, H2 (tests), JUnit 5, AssertJ

**Issue:** All commits reference `Refs #29`

---

## File Map

| File | Change |
|---|---|
| `runtime/src/main/java/.../privacy/ActorIdentityProvider.java` | **Create** — SPI |
| `runtime/src/main/java/.../privacy/DecisionContextSanitiser.java` | **Create** — SPI |
| `runtime/src/main/java/.../privacy/PassThroughActorIdentityProvider.java` | **Create** — default impl |
| `runtime/src/main/java/.../privacy/PassThroughDecisionContextSanitiser.java` | **Create** — default impl |
| `runtime/src/main/java/.../privacy/InternalActorIdentityProvider.java` | **Create** — built-in token impl |
| `runtime/src/main/java/.../privacy/LedgerPrivacyProducer.java` | **Create** — CDI producer |
| `runtime/src/main/java/.../privacy/LedgerErasureService.java` | **Create** — erasure CDI bean |
| `runtime/src/main/java/.../model/ActorIdentity.java` | **Create** — JPA entity |
| `runtime/src/main/resources/db/migration/V1004__actor_identity.sql` | **Create** — migration |
| `runtime/src/main/java/.../config/LedgerConfig.java` | Modify — add `IdentityConfig` |
| `runtime/src/main/java/.../repository/jpa/JpaLedgerEntryRepository.java` | Modify — wire SPIs |
| `runtime/src/test/java/.../privacy/PassThroughPrivacyTest.java` | **Create** — unit tests |
| `runtime/src/test/java/.../privacy/InternalActorIdentityProviderIT.java` | **Create** — IT |
| `runtime/src/test/java/.../privacy/LedgerErasureServiceIT.java` | **Create** — IT |
| `runtime/src/test/java/.../privacy/LedgerPrivacyWiringIT.java` | **Create** — IT |
| `runtime/src/test/resources/application.properties` | Modify — add pseudonymisation-test profile |
| `docs/AUDITABILITY.md` | Modify — mark Axiom 7 addressed |
| `docs/DESIGN.md` | Modify — add privacy section |

All paths under `runtime/src/main/java/io/casehub/ledger/runtime/` and tests under `runtime/src/test/java/io/casehub/ledger/`.

---

## Task 1: SPI interfaces and pass-through defaults

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/ActorIdentityProvider.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/DecisionContextSanitiser.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/PassThroughActorIdentityProvider.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/PassThroughDecisionContextSanitiser.java`
- Create: `runtime/src/test/java/io/casehub/ledger/privacy/PassThroughPrivacyTest.java`

- [ ] **Step 1: Write failing unit tests**

```java
package io.casehub.ledger.privacy;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Optional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.privacy.PassThroughActorIdentityProvider;
import io.casehub.ledger.runtime.privacy.PassThroughDecisionContextSanitiser;

class PassThroughPrivacyTest {

    private final PassThroughActorIdentityProvider provider = new PassThroughActorIdentityProvider();
    private final PassThroughDecisionContextSanitiser sanitiser = new PassThroughDecisionContextSanitiser();

    // ── ActorIdentityProvider ─────────────────────────────────────────────────

    @Test
    void tokenise_returnsRawActorId_unchanged() {
        assertThat(provider.tokenise("alice@example.com")).isEqualTo("alice@example.com");
    }

    @Test
    void tokenise_nullSafe_returnsNull() {
        assertThat(provider.tokenise(null)).isNull();
    }

    @Test
    void tokeniseForQuery_returnsRawActorId_unchanged() {
        assertThat(provider.tokeniseForQuery("alice@example.com")).isEqualTo("alice@example.com");
    }

    @Test
    void resolve_returnsTokenAsIdentity() {
        assertThat(provider.resolve("some-token")).isEqualTo(Optional.of("some-token"));
    }

    @Test
    void resolve_null_returnsEmpty() {
        assertThat(provider.resolve(null)).isEmpty();
    }

    @Test
    void erase_isNoOp_noException() {
        provider.erase("alice@example.com"); // must not throw
    }

    // ── DecisionContextSanitiser ──────────────────────────────────────────────

    @Test
    void sanitise_returnsJson_unchanged() {
        final String json = "{\"name\":\"Alice\",\"riskScore\":42}";
        assertThat(sanitiser.sanitise(json)).isEqualTo(json);
    }

    @Test
    void sanitise_null_returnsNull() {
        assertThat(sanitiser.sanitise(null)).isNull();
    }
}
```

- [ ] **Step 2: Run — verify 8 tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=PassThroughPrivacyTest -q 2>&1 | grep -E "Tests run:|BUILD|ERROR"
```

Expected: compilation error (classes don't exist yet).

- [ ] **Step 3: Create `ActorIdentityProvider.java`**

```java
package io.casehub.ledger.runtime.privacy;

import java.util.Optional;

/**
 * SPI for pseudonymising actor identities written to the ledger.
 *
 * <p>
 * The default implementation is pass-through — existing consumers see zero behaviour change.
 * Replace with a custom CDI bean to plug in any pseudonymisation strategy.
 * The built-in {@link InternalActorIdentityProvider} activates when
 * {@code quarkus.ledger.identity.tokenisation.enabled=true}.
 */
public interface ActorIdentityProvider {

    /**
     * Returns a token to store in place of {@code rawActorId} on write.
     * Creates a new mapping if one does not yet exist.
     * Called on every {@code save()} and {@code saveAttestation()}.
     *
     * @param rawActorId the real actor identity; may be {@code null}
     * @return token to store, or {@code null} if input is {@code null}
     */
    String tokenise(String rawActorId);

    /**
     * Returns the existing token for {@code rawActorId} without creating one.
     * Returns {@code rawActorId} unchanged if no mapping exists.
     * Called on read queries ({@code findByActorId}) to avoid spurious token creation.
     *
     * @param rawActorId the real actor identity
     * @return existing token, or {@code rawActorId} if unmapped
     */
    String tokeniseForQuery(String rawActorId);

    /**
     * Maps a stored token back to the real identity.
     * Returns {@link Optional#empty()} if the mapping has been severed by erasure
     * or never existed.
     *
     * @param token the stored token
     * @return the real identity, or empty if unresolvable
     */
    Optional<String> resolve(String token);

    /**
     * Severs the token→identity mapping for {@code rawActorId}.
     * After this call, {@link #resolve(String)} for the actor's token returns empty.
     * Ledger entries retaining the token become permanently anonymous.
     *
     * @param rawActorId the real actor identity whose mapping to sever
     */
    void erase(String rawActorId);
}
```

- [ ] **Step 4: Create `DecisionContextSanitiser.java`**

```java
package io.casehub.ledger.runtime.privacy;

/**
 * SPI for sanitising {@code ComplianceSupplement.decisionContext} JSON before persist.
 *
 * <p>
 * The default implementation is pass-through. Replace with a custom CDI bean to strip
 * PII from decision context blobs before they reach the ledger.
 */
public interface DecisionContextSanitiser {

    /**
     * Sanitise a decision context JSON string before it is persisted.
     *
     * @param decisionContextJson the raw JSON; may be {@code null}
     * @return sanitised JSON, or {@code null} if input is {@code null}
     */
    String sanitise(String decisionContextJson);
}
```

- [ ] **Step 5: Create `PassThroughActorIdentityProvider.java`**

```java
package io.casehub.ledger.runtime.privacy;

import java.util.Optional;

/** Pass-through implementation — stores raw actor identities unchanged. */
public class PassThroughActorIdentityProvider implements ActorIdentityProvider {

    @Override
    public String tokenise(final String rawActorId) {
        return rawActorId;
    }

    @Override
    public String tokeniseForQuery(final String rawActorId) {
        return rawActorId;
    }

    @Override
    public Optional<String> resolve(final String token) {
        return Optional.ofNullable(token);
    }

    @Override
    public void erase(final String rawActorId) {
        // pass-through: no mapping to sever
    }
}
```

- [ ] **Step 6: Create `PassThroughDecisionContextSanitiser.java`**

```java
package io.casehub.ledger.runtime.privacy;

/** Pass-through implementation — stores decision context JSON unchanged. */
public class PassThroughDecisionContextSanitiser implements DecisionContextSanitiser {

    @Override
    public String sanitise(final String decisionContextJson) {
        return decisionContextJson;
    }
}
```

- [ ] **Step 7: Run — verify 8 tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=PassThroughPrivacyTest -q 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: `Tests run: 8, Failures: 0` and `BUILD SUCCESS`.

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/privacy/ \
        runtime/src/test/java/io/casehub/ledger/privacy/PassThroughPrivacyTest.java
git commit -m "$(cat <<'EOF'
feat(privacy): add ActorIdentityProvider + DecisionContextSanitiser SPIs with pass-through defaults

Zero behaviour change — pass-through impls store raw identities unchanged.
Refs #29
EOF
)"
```

---

## Task 2: `ActorIdentity` entity and V1004 migration

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorIdentity.java`
- Create: `runtime/src/main/resources/db/migration/V1004__actor_identity.sql`

- [ ] **Step 1: Create `V1004__actor_identity.sql`**

```sql
-- CaseHub Ledger — actor identity pseudonymisation table (V1004)
-- Compatible with H2 (dev/test) and PostgreSQL (production)
--
-- actor_identity: optional token-to-identity mapping for pseudonymisation.
-- Always created. Empty when quarkus.ledger.identity.tokenisation.enabled=false.
-- UNIQUE (actor_id) ensures one stable token per rawActorId.
-- Erasure deletes the row; stored tokens become permanently unresolvable.

CREATE TABLE actor_identity (
    token      VARCHAR(255) NOT NULL,
    actor_id   VARCHAR(255) NOT NULL,
    created_at TIMESTAMP    NOT NULL,
    CONSTRAINT pk_actor_identity PRIMARY KEY (token),
    CONSTRAINT uq_actor_identity_actor_id UNIQUE (actor_id)
);
```

- [ ] **Step 2: Create `ActorIdentity.java`**

```java
package io.casehub.ledger.runtime.model;

import java.time.Instant;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.NamedQuery;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

/**
 * Token-to-identity mapping for actor pseudonymisation.
 * Plain {@code @Entity} — queries via {@code @NamedQuery} + EntityManager.
 *
 * <p>
 * Each row maps one UUID token (stored in ledger entries) to one real actor identity.
 * Deleting a row severs the link — the token in existing entries becomes unresolvable.
 */
@Entity
@Table(name = "actor_identity")
@NamedQuery(
        name = "ActorIdentity.findByActorId",
        query = "SELECT a FROM ActorIdentity a WHERE a.actorId = :actorId")
@NamedQuery(
        name = "ActorIdentity.deleteByActorId",
        query = "DELETE FROM ActorIdentity a WHERE a.actorId = :actorId")
public class ActorIdentity {

    /** UUID token stored in ledger entries in place of the real identity. */
    @Id
    @Column(name = "token")
    public String token;

    /** The real actor identity this token represents. */
    @Column(name = "actor_id", nullable = false)
    public String actorId;

    /** When this mapping was created. */
    @Column(name = "created_at", nullable = false)
    public Instant createdAt;

    @PrePersist
    void prePersist() {
        if (createdAt == null) {
            createdAt = Instant.now();
        }
    }
}
```

- [ ] **Step 3: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | grep -E "ERROR|BUILD"
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/ActorIdentity.java \
        runtime/src/main/resources/db/migration/V1004__actor_identity.sql
git commit -m "$(cat <<'EOF'
feat(schema): add actor_identity table (V1004) and ActorIdentity entity

Always created. Empty when tokenisation disabled. UNIQUE(actor_id) ensures
one stable token per identity. Erasure deletes the row.
Refs #29
EOF
)"
```

---

## Task 3: `LedgerConfig.IdentityConfig` and `LedgerPrivacyProducer`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/InternalActorIdentityProvider.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/LedgerPrivacyProducer.java`

- [ ] **Step 1: Add `IdentityConfig` to `LedgerConfig.java`**

Add after the existing `merkle()` accessor (before the final closing brace of `LedgerConfig`):

```java
    /**
     * Actor identity pseudonymisation settings.
     *
     * @return the identity sub-configuration
     */
    IdentityConfig identity();

    /** Actor identity pseudonymisation settings. */
    interface IdentityConfig {

        /**
         * Tokenisation settings.
         *
         * @return the tokenisation sub-configuration
         */
        TokenisationConfig tokenisation();

        /** Token-based pseudonymisation settings. */
        interface TokenisationConfig {

            /**
             * When {@code true}, actor identities are stored as UUID tokens backed by
             * the {@code actor_identity} table. On erasure, the token→identity mapping
             * is deleted — ledger entries retain the token but it becomes unresolvable.
             * Off by default — zero behaviour change when disabled.
             *
             * <p>
             * Organisations with their own identity management systems should leave this
             * off and provide a custom {@link io.casehub.ledger.runtime.privacy.ActorIdentityProvider}
             * CDI bean instead.
             *
             * @return {@code true} if built-in tokenisation is active; {@code false} by default
             */
            @WithDefault("false")
            boolean enabled();
        }
    }
```

- [ ] **Step 2: Create `InternalActorIdentityProvider.java`**

```java
package io.casehub.ledger.runtime.privacy;

import java.util.Optional;
import java.util.UUID;

import jakarta.persistence.EntityManager;

import io.casehub.ledger.runtime.model.ActorIdentity;

/**
 * Built-in token-based actor identity provider backed by the {@code actor_identity} table.
 *
 * <p>
 * Not a CDI bean — constructed by {@link LedgerPrivacyProducer} when
 * {@code quarkus.ledger.identity.tokenisation.enabled=true}. The EntityManager
 * it receives is a CDI proxy that resolves to the current transaction's session
 * when its methods are called — all callers of this class operate within
 * an existing {@code @Transactional} boundary.
 */
public class InternalActorIdentityProvider implements ActorIdentityProvider {

    private final EntityManager em;

    public InternalActorIdentityProvider(final EntityManager em) {
        this.em = em;
    }

    /**
     * Returns the existing token for {@code rawActorId}, creating one if absent.
     * {@code null} input returns {@code null} (caller guards against null actorId).
     */
    @Override
    public String tokenise(final String rawActorId) {
        if (rawActorId == null) {
            return null;
        }
        return em.createNamedQuery("ActorIdentity.findByActorId", ActorIdentity.class)
                .setParameter("actorId", rawActorId)
                .getResultStream()
                .map(a -> a.token)
                .findFirst()
                .orElseGet(() -> {
                    final ActorIdentity identity = new ActorIdentity();
                    identity.token = UUID.randomUUID().toString();
                    identity.actorId = rawActorId;
                    em.persist(identity);
                    return identity.token;
                });
    }

    /**
     * Returns the existing token for {@code rawActorId} without creating one.
     * Returns {@code rawActorId} unchanged if no mapping exists — read queries
     * will correctly return empty results.
     */
    @Override
    public String tokeniseForQuery(final String rawActorId) {
        if (rawActorId == null) {
            return null;
        }
        return em.createNamedQuery("ActorIdentity.findByActorId", ActorIdentity.class)
                .setParameter("actorId", rawActorId)
                .getResultStream()
                .map(a -> a.token)
                .findFirst()
                .orElse(rawActorId);
    }

    /** Returns the real identity for a token, or empty if the mapping was erased. */
    @Override
    public Optional<String> resolve(final String token) {
        if (token == null) {
            return Optional.empty();
        }
        return Optional.ofNullable(em.find(ActorIdentity.class, token))
                .map(a -> a.actorId);
    }

    /** Deletes the token→identity mapping. The token in existing entries becomes unresolvable. */
    @Override
    public void erase(final String rawActorId) {
        em.createNamedQuery("ActorIdentity.deleteByActorId")
                .setParameter("actorId", rawActorId)
                .executeUpdate();
    }
}
```

- [ ] **Step 3: Create `LedgerPrivacyProducer.java`**

```java
package io.casehub.ledger.runtime.privacy;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.quarkus.arc.DefaultBean;

/**
 * CDI producer for {@link ActorIdentityProvider} and {@link DecisionContextSanitiser}.
 *
 * <p>
 * Both producers are annotated {@link DefaultBean} — a consumer-supplied CDI bean of the
 * same type silently replaces the default without any configuration change.
 *
 * <p>
 * {@link ActorIdentityProvider}: returns {@link InternalActorIdentityProvider} when
 * {@code quarkus.ledger.identity.tokenisation.enabled=true}; otherwise pass-through.
 * {@link DecisionContextSanitiser}: always returns pass-through; replace with a
 * custom bean to scrub PII from decision context blobs.
 */
@ApplicationScoped
public class LedgerPrivacyProducer {

    @Inject
    LedgerConfig config;

    @Inject
    EntityManager em;

    @Produces
    @DefaultBean
    @ApplicationScoped
    public ActorIdentityProvider actorIdentityProvider() {
        if (config.identity().tokenisation().enabled()) {
            return new InternalActorIdentityProvider(em);
        }
        return new PassThroughActorIdentityProvider();
    }

    @Produces
    @DefaultBean
    @ApplicationScoped
    public DecisionContextSanitiser decisionContextSanitiser() {
        return new PassThroughDecisionContextSanitiser();
    }
}
```

- [ ] **Step 4: Run full test suite — all 129 pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `Tests run: 129, Failures: 0` and `BUILD SUCCESS`.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java \
        runtime/src/main/java/io/casehub/ledger/runtime/privacy/InternalActorIdentityProvider.java \
        runtime/src/main/java/io/casehub/ledger/runtime/privacy/LedgerPrivacyProducer.java
git commit -m "$(cat <<'EOF'
feat(privacy): add InternalActorIdentityProvider and LedgerPrivacyProducer

Config key quarkus.ledger.identity.tokenisation.enabled activates built-in
token impl. DefaultBean semantics mean custom CDI beans replace without config.
Refs #29
EOF
)"
```

---

## Task 4: `LedgerErasureService`

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/LedgerErasureService.java`

- [ ] **Step 1: Create `LedgerErasureService.java`**

```java
package io.casehub.ledger.runtime.privacy;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.ActorIdentity;

/**
 * CDI bean for processing GDPR Art.17 erasure requests.
 *
 * <p>
 * Severs the token→identity mapping for the given actor. Ledger entries retaining
 * the token become permanently anonymous — the hash chain is intact; the personal
 * data link is gone. Returns an {@link ErasureResult} with diagnostic information.
 */
@ApplicationScoped
public class LedgerErasureService {

    @Inject
    ActorIdentityProvider actorIdentityProvider;

    @Inject
    EntityManager em;

    /**
     * The outcome of an erasure request.
     *
     * @param rawActorId the identity that was requested for erasure
     * @param mappingFound {@code true} if a token→identity mapping existed and was severed
     * @param affectedEntryCount number of ledger entries whose {@code actorId} was the severed token;
     *        informational only — entries are not deleted
     */
    public record ErasureResult(String rawActorId, boolean mappingFound, long affectedEntryCount) {
    }

    /**
     * Process an erasure request for the given actor identity.
     *
     * <p>
     * If no mapping exists (tokenisation was never enabled for this actor, or the identity
     * was already erased), returns {@code mappingFound=false} with count 0.
     *
     * @param rawActorId the real actor identity to erase
     * @return the erasure result
     */
    @Transactional
    public ErasureResult erase(final String rawActorId) {
        final List<ActorIdentity> existing = em
                .createNamedQuery("ActorIdentity.findByActorId", ActorIdentity.class)
                .setParameter("actorId", rawActorId)
                .getResultList();

        if (existing.isEmpty()) {
            return new ErasureResult(rawActorId, false, 0L);
        }

        final String token = existing.get(0).token;

        final long count = em
                .createQuery("SELECT COUNT(e) FROM LedgerEntry e WHERE e.actorId = :token", Long.class)
                .setParameter("token", token)
                .getSingleResult();

        actorIdentityProvider.erase(rawActorId);

        return new ErasureResult(rawActorId, true, count);
    }
}
```

- [ ] **Step 2: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | grep -E "ERROR|BUILD"
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/privacy/LedgerErasureService.java
git commit -m "$(cat <<'EOF'
feat(privacy): add LedgerErasureService — GDPR Art.17 erasure with ErasureResult

Severs token→identity mapping, counts affected entries, delegates to
ActorIdentityProvider.erase(). Hash chain intact after erasure.
Refs #29
EOF
)"
```

---

## Task 5: Wire SPIs into `JpaLedgerEntryRepository`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`

- [ ] **Step 1: Add SPI injections and wire into `save()`, `saveAttestation()`, `findByActorId()`**

Add two `@Inject` fields after the existing `@Inject LedgerMerklePublisher merklePublisher;`:

```java
    @Inject
    ActorIdentityProvider actorIdentityProvider;

    @Inject
    DecisionContextSanitiser decisionContextSanitiser;
```

Add the import at the top:
```java
import io.casehub.ledger.runtime.privacy.ActorIdentityProvider;
import io.casehub.ledger.runtime.privacy.DecisionContextSanitiser;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
```

Replace the `save()` method body — insert tokenisation and sanitisation **before** computing the digest and before `em.persist(entry)`:

```java
    @Override
    @Transactional
    public LedgerEntry save(final LedgerEntry entry) {
        if (entry.occurredAt == null) {
            entry.occurredAt = Instant.now();
        }

        // Pseudonymise actor identity before computing the leaf hash.
        // The hash chain covers the token, not the raw identity.
        if (entry.actorId != null) {
            entry.actorId = actorIdentityProvider.tokenise(entry.actorId);
        }

        // Sanitise decisionContext in any attached ComplianceSupplement.
        entry.compliance().ifPresent(cs -> {
            if (cs.decisionContext != null) {
                cs.decisionContext = decisionContextSanitiser.sanitise(cs.decisionContext);
                entry.refreshSupplementJson();
            }
        });

        if (ledgerConfig.hashChain().enabled()) {
            entry.digest = LedgerMerkleTree.leafHash(entry);
        }
        em.persist(entry);

        if (ledgerConfig.hashChain().enabled()) {
            final List<LedgerMerkleFrontier> currentFrontier = em
                    .createNamedQuery("LedgerMerkleFrontier.findBySubjectId", LedgerMerkleFrontier.class)
                    .setParameter("subjectId", entry.subjectId)
                    .getResultList();

            final List<LedgerMerkleFrontier> newFrontier = LedgerMerkleTree.append(entry.digest, currentFrontier,
                    entry.subjectId);

            final Set<Integer> newLevels = newFrontier.stream()
                    .map(n -> n.level)
                    .collect(Collectors.toSet());
            for (final LedgerMerkleFrontier old : currentFrontier) {
                if (!newLevels.contains(old.level)) {
                    em.createNamedQuery("LedgerMerkleFrontier.deleteBySubjectAndLevel")
                            .setParameter("subjectId", entry.subjectId)
                            .setParameter("level", old.level)
                            .executeUpdate();
                }
            }

            for (final LedgerMerkleFrontier node : newFrontier) {
                em.createNamedQuery("LedgerMerkleFrontier.deleteBySubjectAndLevel")
                        .setParameter("subjectId", entry.subjectId)
                        .setParameter("level", node.level)
                        .executeUpdate();
                em.persist(node);
            }

            final String newRoot = LedgerMerkleTree.treeRoot(newFrontier);
            merklePublisher.publish(entry.subjectId, entry.sequenceNumber, newRoot);
        }

        return entry;
    }
```

Replace `saveAttestation()` — tokenise `attestorId` before persist:

```java
    @Override
    public LedgerAttestation saveAttestation(final LedgerAttestation attestation) {
        if (attestation.attestorId != null) {
            attestation.attestorId = actorIdentityProvider.tokenise(attestation.attestorId);
        }
        em.persist(attestation);
        return attestation;
    }
```

Replace `findByActorId()` — translate raw actorId to token for query:

```java
    @Override
    public List<LedgerEntry> findByActorId(final String actorId,
            final Instant from, final Instant to) {
        final String token = actorIdentityProvider.tokeniseForQuery(actorId);
        return em.createQuery(
                "SELECT e FROM LedgerEntry e WHERE e.actorId = :actorId" +
                        " AND e.occurredAt >= :from AND e.occurredAt <= :to ORDER BY e.occurredAt ASC",
                LedgerEntry.class)
                .setParameter("actorId", token)
                .setParameter("from", from)
                .setParameter("to", to)
                .getResultList();
    }
```

- [ ] **Step 2: Run full test suite — all 129 pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `Tests run: 129, Failures: 0` and `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java
git commit -m "$(cat <<'EOF'
feat(privacy): wire ActorIdentityProvider and DecisionContextSanitiser into JpaLedgerEntryRepository

Tokenises actorId and attestorId on write. Sanitises decisionContext before persist.
Translates actorId to token in findByActorId(). Pass-through default = zero change.
Refs #29
EOF
)"
```

---

## Task 6: Integration tests — `InternalActorIdentityProviderIT`

**Files:**
- Modify: `runtime/src/test/resources/application.properties`
- Create: `runtime/src/test/java/io/casehub/ledger/privacy/InternalActorIdentityProviderIT.java`

- [ ] **Step 1: Add pseudonymisation-test profile to `application.properties`**

Append at the end of the file:

```properties
# Pseudonymisation test profile (used by InternalActorIdentityProviderIT, LedgerErasureServiceIT, LedgerPrivacyWiringIT)
# Isolated DB — prevents token PK collisions with other test classes
%pseudonymisation-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:pseudonymisationtestdb;DB_CLOSE_DELAY=-1
%pseudonymisation-test.quarkus.ledger.identity.tokenisation.enabled=true
```

- [ ] **Step 2: Create `InternalActorIdentityProviderIT.java`**

```java
package io.casehub.ledger.privacy;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Optional;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.privacy.ActorIdentityProvider;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration tests for {@link io.casehub.ledger.runtime.privacy.InternalActorIdentityProvider}.
 *
 * <p>
 * Runs with tokenisation enabled against an isolated H2 database.
 * Verifies token creation, idempotency, resolution, and erasure behaviour.
 */
@QuarkusTest
@TestProfile(InternalActorIdentityProviderIT.PseudonymisationProfile.class)
class InternalActorIdentityProviderIT {

    public static class PseudonymisationProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "pseudonymisation-test";
        }
    }

    @Inject
    ActorIdentityProvider provider;

    // ── Happy path: tokenise creates a token ──────────────────────────────────

    @Test
    @Transactional
    void tokenise_createsToken_differentFromRawActorId() {
        final String token = provider.tokenise("alice@example.com");
        assertThat(token).isNotNull().isNotEqualTo("alice@example.com");
    }

    // ── Correctness: same actorId always maps to same token ───────────────────

    @Test
    @Transactional
    void tokenise_sameActorId_returnsSameToken() {
        final String token1 = provider.tokenise("bob@example.com");
        final String token2 = provider.tokenise("bob@example.com");
        assertThat(token1).isEqualTo(token2);
    }

    // ── Correctness: different actorIds get different tokens ──────────────────

    @Test
    @Transactional
    void tokenise_differentActorIds_returnsDifferentTokens() {
        final String tokenAlice = provider.tokenise("alice-" + java.util.UUID.randomUUID());
        final String tokenBob = provider.tokenise("bob-" + java.util.UUID.randomUUID());
        assertThat(tokenAlice).isNotEqualTo(tokenBob);
    }

    // ── Happy path: null input returns null ───────────────────────────────────

    @Test
    @Transactional
    void tokenise_null_returnsNull() {
        assertThat(provider.tokenise(null)).isNull();
    }

    // ── Happy path: tokeniseForQuery returns existing token ───────────────────

    @Test
    @Transactional
    void tokeniseForQuery_existingActor_returnsToken() {
        final String actorId = "carol-" + java.util.UUID.randomUUID();
        final String token = provider.tokenise(actorId);
        assertThat(provider.tokeniseForQuery(actorId)).isEqualTo(token);
    }

    // ── Correctness: tokeniseForQuery does not create for unknown actor ────────

    @Test
    @Transactional
    void tokeniseForQuery_unknownActor_returnsRawActorId() {
        final String unknown = "unknown-" + java.util.UUID.randomUUID();
        assertThat(provider.tokeniseForQuery(unknown)).isEqualTo(unknown);
    }

    // ── Happy path: resolve returns the real identity ─────────────────────────

    @Test
    @Transactional
    void resolve_existingToken_returnsRealIdentity() {
        final String actorId = "dave-" + java.util.UUID.randomUUID();
        final String token = provider.tokenise(actorId);
        assertThat(provider.resolve(token)).isEqualTo(Optional.of(actorId));
    }

    // ── Correctness: resolve returns empty for unknown token ──────────────────

    @Test
    @Transactional
    void resolve_unknownToken_returnsEmpty() {
        assertThat(provider.resolve("no-such-token")).isEmpty();
    }

    // ── Correctness: resolve returns empty for null ───────────────────────────

    @Test
    @Transactional
    void resolve_null_returnsEmpty() {
        assertThat(provider.resolve(null)).isEmpty();
    }

    // ── Happy path: erase severs the mapping ─────────────────────────────────

    @Test
    @Transactional
    void erase_severed_resolveReturnsEmpty() {
        final String actorId = "eve-" + java.util.UUID.randomUUID();
        final String token = provider.tokenise(actorId);

        provider.erase(actorId);

        assertThat(provider.resolve(token)).isEmpty();
    }

    // ── Correctness: erase of unknown actorId does not throw ─────────────────

    @Test
    @Transactional
    void erase_unknownActor_noException() {
        provider.erase("never-registered-" + java.util.UUID.randomUUID()); // must not throw
    }

    // ── Correctness: tokeniseForQuery after erase returns raw actorId ─────────

    @Test
    @Transactional
    void tokeniseForQuery_afterErase_returnsRawActorId() {
        final String actorId = "frank-" + java.util.UUID.randomUUID();
        provider.tokenise(actorId); // create mapping
        provider.erase(actorId);   // sever it

        assertThat(provider.tokeniseForQuery(actorId)).isEqualTo(actorId);
    }
}
```

- [ ] **Step 3: Run**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=InternalActorIdentityProviderIT 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: `Tests run: 11, Failures: 0` and `BUILD SUCCESS`.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/resources/application.properties \
        runtime/src/test/java/io/casehub/ledger/privacy/InternalActorIdentityProviderIT.java
git commit -m "$(cat <<'EOF'
test(privacy): InternalActorIdentityProviderIT — 11 tests covering tokenise, resolve, erase

Happy path, correctness, edge cases: idempotent tokenise, null safety,
tokeniseForQuery non-creating, erase severs mapping.
Refs #29
EOF
)"
```

---

## Task 7: Integration tests — `LedgerErasureServiceIT` and `LedgerPrivacyWiringIT`

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/privacy/LedgerErasureServiceIT.java`
- Create: `runtime/src/test/java/io/casehub/ledger/privacy/LedgerPrivacyWiringIT.java`

- [ ] **Step 1: Create `LedgerErasureServiceIT.java`**

```java
package io.casehub.ledger.privacy;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.privacy.LedgerErasureService;
import io.casehub.ledger.runtime.privacy.LedgerErasureService.ErasureResult;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * End-to-end integration tests for {@link LedgerErasureService}.
 *
 * <p>
 * Verifies the full erasure pipeline: save entries → check count → erase →
 * confirm mapping severed and findByActorId returns empty.
 */
@QuarkusTest
@TestProfile(InternalActorIdentityProviderIT.PseudonymisationProfile.class)
class LedgerErasureServiceIT {

    @Inject
    LedgerErasureService erasureService;

    @Inject
    LedgerEntryRepository repo;

    // ── Happy path: erase known actor ─────────────────────────────────────────

    @Test
    @Transactional
    void erase_knownActor_mappingFound_correctCount() {
        final String actorId = "actor-" + UUID.randomUUID();

        // Save 3 entries for this actor
        saveEntry(actorId);
        saveEntry(actorId);
        saveEntry(actorId);

        final ErasureResult result = erasureService.erase(actorId);

        assertThat(result.rawActorId()).isEqualTo(actorId);
        assertThat(result.mappingFound()).isTrue();
        assertThat(result.affectedEntryCount()).isEqualTo(3L);
    }

    // ── Happy path: erase unknown actor ───────────────────────────────────────

    @Test
    @Transactional
    void erase_unknownActor_mappingNotFound_zeroCount() {
        final ErasureResult result = erasureService.erase("never-saved-" + UUID.randomUUID());

        assertThat(result.mappingFound()).isFalse();
        assertThat(result.affectedEntryCount()).isZero();
    }

    // ── End-to-end: findByActorId returns empty after erase ───────────────────

    @Test
    @Transactional
    void findByActorId_afterErase_returnsEmpty() {
        final String actorId = "erasable-" + UUID.randomUUID();
        final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
        final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

        saveEntry(actorId);

        // Before erase: entry is findable
        assertThat(repo.findByActorId(actorId, from, to)).hasSize(1);

        erasureService.erase(actorId);

        // After erase: findByActorId with raw actorId returns empty
        // (tokeniseForQuery now returns raw actorId, which has no entries)
        assertThat(repo.findByActorId(actorId, from, to)).isEmpty();
    }

    // ── Correctness: erase twice is idempotent ────────────────────────────────

    @Test
    @Transactional
    void erase_twice_secondReturnsNotFound() {
        final String actorId = "double-erase-" + UUID.randomUUID();
        saveEntry(actorId);

        erasureService.erase(actorId);
        final ErasureResult second = erasureService.erase(actorId);

        assertThat(second.mappingFound()).isFalse();
        assertThat(second.affectedEntryCount()).isZero();
    }

    // ── Fixture ───────────────────────────────────────────────────────────────

    private void saveEntry(final String actorId) {
        final TestEntry entry = new TestEntry();
        entry.subjectId = UUID.randomUUID();
        entry.sequenceNumber = 1;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = Instant.now();
        repo.save(entry);
    }
}
```

- [ ] **Step 2: Create `LedgerPrivacyWiringIT.java`**

```java
package io.casehub.ledger.privacy;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration tests verifying the write-path and query-path wiring of
 * {@link io.casehub.ledger.runtime.privacy.ActorIdentityProvider} and
 * {@link io.casehub.ledger.runtime.privacy.DecisionContextSanitiser}
 * inside {@link io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository}.
 */
@QuarkusTest
@TestProfile(InternalActorIdentityProviderIT.PseudonymisationProfile.class)
class LedgerPrivacyWiringIT {

    @Inject
    LedgerEntryRepository repo;

    @Inject
    EntityManager em;

    // ── Happy path: actorId stored as token, not raw ──────────────────────────

    @Test
    @Transactional
    void save_actorIdStoredAsToken_notRawIdentity() {
        final String rawActorId = "alice-" + UUID.randomUUID();
        final TestEntry entry = entry(rawActorId);
        repo.save(entry);

        final LedgerEntry stored = em.find(LedgerEntry.class, entry.id);
        assertThat(stored.actorId)
                .isNotNull()
                .isNotEqualTo(rawActorId); // stored as UUID token
    }

    // ── Happy path: findByActorId translates raw → token transparently ─────────

    @Test
    @Transactional
    void findByActorId_withRawActorId_returnsEntry() {
        final String rawActorId = "bob-" + UUID.randomUUID();
        final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
        final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

        repo.save(entry(rawActorId));

        final List<LedgerEntry> results = repo.findByActorId(rawActorId, from, to);
        assertThat(results).hasSize(1);
    }

    // ── Correctness: attestorId tokenised on saveAttestation ─────────────────

    @Test
    @Transactional
    void saveAttestation_attestorIdStoredAsToken() {
        final String rawActorId = "carol-" + UUID.randomUUID();
        final String rawAttestorId = "attestor-" + UUID.randomUUID();
        final TestEntry entry = entry(rawActorId);
        repo.save(entry);

        final LedgerAttestation att = new LedgerAttestation();
        att.id = UUID.randomUUID();
        att.ledgerEntryId = entry.id;
        att.subjectId = entry.subjectId;
        att.attestorId = rawAttestorId;
        att.attestorType = ActorType.AGENT;
        att.verdict = AttestationVerdict.SOUND;
        att.confidence = 0.9;
        att.occurredAt = Instant.now();
        repo.saveAttestation(att);

        final LedgerAttestation stored = em.find(LedgerAttestation.class, att.id);
        assertThat(stored.attestorId)
                .isNotNull()
                .isNotEqualTo(rawAttestorId); // stored as UUID token
    }

    // ── Happy path: decisionContext passed through pass-through sanitiser ─────

    @Test
    @Transactional
    void save_decisionContext_storedUnchanged_withPassThroughSanitiser() {
        // Default DecisionContextSanitiser is pass-through — content unchanged
        final String rawActorId = "dave-" + UUID.randomUUID();
        final TestEntry entry = entry(rawActorId);
        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.decisionContext = "{\"riskScore\":42,\"region\":\"EU\"}";
        entry.attach(cs);

        repo.save(entry);

        final LedgerEntry stored = em.find(LedgerEntry.class, entry.id);
        stored.supplements.size(); // force load
        assertThat(stored.compliance())
                .isPresent()
                .hasValueSatisfying(c ->
                        assertThat(c.decisionContext).isEqualTo("{\"riskScore\":42,\"region\":\"EU\"}")
                );
    }

    // ── Correctness: same actorId on two entries → same token ─────────────────

    @Test
    @Transactional
    void save_sameActorId_twoEntries_sameTokenStored() {
        final String rawActorId = "eve-" + UUID.randomUUID();
        final TestEntry e1 = entry(rawActorId);
        final TestEntry e2 = entry(rawActorId);
        repo.save(e1);
        repo.save(e2);

        final String token1 = em.find(LedgerEntry.class, e1.id).actorId;
        final String token2 = em.find(LedgerEntry.class, e2.id).actorId;
        assertThat(token1).isEqualTo(token2);
    }

    // ── Correctness: findByActorId for unknown actorId returns empty ──────────

    @Test
    @Transactional
    void findByActorId_unknownActorId_returnsEmpty() {
        final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
        final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

        assertThat(repo.findByActorId("never-saved-" + UUID.randomUUID(), from, to)).isEmpty();
    }

    // ── Fixture ───────────────────────────────────────────────────────────────

    private TestEntry entry(final String actorId) {
        final TestEntry e = new TestEntry();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.actorRole = "Classifier";
        e.occurredAt = Instant.now();
        return e;
    }
}
```

- [ ] **Step 3: Run all privacy ITs**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest="InternalActorIdentityProviderIT,LedgerErasureServiceIT,LedgerPrivacyWiringIT" \
  2>&1 | grep -E "Tests run:|BUILD"
```

Expected: all tests pass, `BUILD SUCCESS`.

- [ ] **Step 4: Run full suite — all tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `BUILD SUCCESS`. Total count will be 129 + new tests.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/privacy/LedgerErasureServiceIT.java \
        runtime/src/test/java/io/casehub/ledger/privacy/LedgerPrivacyWiringIT.java
git commit -m "$(cat <<'EOF'
test(privacy): LedgerErasureServiceIT (4 tests) + LedgerPrivacyWiringIT (6 tests)

Erasure: known actor, unknown actor, findByActorId empty after erase, idempotent erase.
Wiring: actorId tokenised, findByActorId translates, attestorId tokenised,
decisionContext pass-through, same actorId = same token, unknown = empty.
Refs #29
EOF
)"
```

---

## Task 8: Update docs — AUDITABILITY.md, DESIGN.md

**Files:**
- Modify: `docs/AUDITABILITY.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Update `AUDITABILITY.md` — mark Axiom 7 addressed**

Find the Axiom 7 row in the summary table:
```
| 7. Privacy Compatibility | ❌ Gap | Pseudonymisation strategy (not yet designed) |
```

Replace with:
```
| 7. Privacy Compatibility | ✅ Addressed | `ActorIdentityProvider` + `DecisionContextSanitiser` SPIs. Built-in tokenisation via `quarkus.ledger.identity.tokenisation.enabled`. `LedgerErasureService` for GDPR Art.17 requests. |
```

Also find the Axiom 7 assessment section (the full text block under `### 7. Privacy Compatibility`) and update it to describe the implemented solution. Replace the ❌ Gap description with:

```markdown
### 7. Privacy Compatibility ✅ Addressed

**What it means:**
Audit records must coexist with privacy rights — including the GDPR right to erasure (Art.17).
An immutable ledger cannot delete entries without breaking tamper evidence. The solution is
pseudonymisation: store tokens instead of raw identities, backed by a detachable mapping.

**Current state:**
Two SPIs in `io.casehub.ledger.runtime.privacy`:

- `ActorIdentityProvider` — tokenises `actorId` (write) and `attestorId` on every persist.
  `tokenise()` creates a UUID token if none exists. `tokeniseForQuery()` does a read-only
  lookup. `erase()` deletes the mapping — the token in existing entries becomes unresolvable.
- `DecisionContextSanitiser` — sanitises `ComplianceSupplement.decisionContext` JSON before
  persist. Default is pass-through; consumers supply a custom CDI bean to strip PII.

Built-in implementation (`InternalActorIdentityProvider`) backed by the `actor_identity`
table (V1004). Activated via `quarkus.ledger.identity.tokenisation.enabled=true`.

`LedgerErasureService.erase(String rawActorId)` processes GDPR Art.17 requests: locates
the token, counts affected entries (informational), severs the mapping, returns `ErasureResult`.

Organisations with external identity management supply a custom `ActorIdentityProvider` CDI
bean — it replaces the default via `@DefaultBean` semantics, no config changes needed.

**Remaining gap:**
`decisionContext` PII scrubbing is the consumer's responsibility — the `DecisionContextSanitiser`
SPI provides the hook, but the extension cannot validate JSON content. `subjectId` (the
aggregate UUID) is typically not personal data; consumers who use PII as their `subjectId`
must address this outside the extension.
```

- [ ] **Step 2: Update `DESIGN.md` — add Privacy section**

Find the section in DESIGN.md that describes the supplement stack or the configuration table. After the trust scoring section (or the configuration table), add:

```markdown
### Privacy and Pseudonymisation

Actor identities (`actorId`, `attestorId`) and decision context blobs are intercepted on
every write by two SPIs in `io.casehub.ledger.runtime.privacy`:

| SPI | Default | Purpose |
|---|---|---|
| `ActorIdentityProvider` | Pass-through | Tokenise actor identities; resolve tokens back to real identities; sever mappings on erasure |
| `DecisionContextSanitiser` | Pass-through | Strip PII from `ComplianceSupplement.decisionContext` before persist |

Both defaults produce zero behaviour change. Supply a custom CDI bean to replace either.

**Built-in tokenisation** (`InternalActorIdentityProvider`) activates when
`quarkus.ledger.identity.tokenisation.enabled=true`. Tokens are UUID strings stored in the
`actor_identity` table (V1004). Erasure deletes the mapping row — the token in existing
entries becomes permanently unresolvable but the Merkle hash chain is intact.

**`LedgerErasureService`** processes GDPR Art.17 erasure requests. Returns `ErasureResult`
with the actor identity, whether a mapping was found, and how many ledger entries were
affected (entries are not deleted).

**Config:**

| Key | Default | Description |
|---|---|---|
| `quarkus.ledger.identity.tokenisation.enabled` | `false` | Activate built-in UUID token pseudonymisation |
```

- [ ] **Step 3: Run full test suite — final green check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 4: Commit**

```bash
git add docs/AUDITABILITY.md docs/DESIGN.md
git commit -m "$(cat <<'EOF'
docs: mark Axiom 7 (Privacy) addressed — ActorIdentityProvider, DecisionContextSanitiser, LedgerErasureService

AUDITABILITY.md updated with implemented solution. DESIGN.md gains
privacy/pseudonymisation section with config table.

Closes #29
EOF
)"
```

---

## Verification

After all tasks complete:

```bash
# All tests pass
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep "Tests run:"

# No raw actorId storage path bypasses the provider
grep -r "em\.persist(entry)" runtime/src/main --include="*.java"
# Only one hit — in JpaLedgerEntryRepository.save(), after tokenisation

# No ForgivenessParams remnants and no actorId hardcoded in tests
grep -r "ForgivenessParams\|rawActorId.*=.*em\." runtime/src --include="*.java"

# All commits reference #29
git log --format="%s%n%b" -9 | grep "#29"
```
