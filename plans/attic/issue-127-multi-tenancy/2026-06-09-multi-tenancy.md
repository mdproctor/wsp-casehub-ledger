# Multi-Tenancy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `tenancyId` column to `ledger_entry`, thread it through all SPI methods, and split cross-tenant operations into separate `@CrossTenant`-qualified repository interfaces.

**Architecture:** Explicit `String tenancyId` parameter on every tenant-scoped SPI method. Cross-tenant methods (trust computation, health, retention) split to separate `CrossTenant*Repository` interfaces produced via a `@CrossTenant` CDI qualifier, guarded by `LedgerSystemCurrentPrincipal.isCrossTenantAdmin()`. Full parity with casehub-engine's tenancy pattern (#299, #405, #406).

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM (EntityManager + JPQL), H2 test DB, Flyway migrations, CDI qualifiers.

**Spec:** `specs/issue-127-multi-tenancy/2026-06-08-multi-tenancy-design.md`

**Governing protocols:** PP-20260520-439daf (unconditional filtering), PP-20260520-e6a5f0 (tenancyId in data access only), PP-20260511-ledger-spi (SPI propagation), PP-20260519-3f2ea2 (reactive SPI shim).

---

## Task 1: Entity Model — Add `tenancyId` to `LedgerEntry`

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`
- Modify: `runtime/src/main/resources/db/ledger/migration/V1000__ledger_base_schema.sql`

- [ ] **Step 1: Add `tenancyId` field to api `LedgerEntry`**

In `api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java`, add after the `public UUID subjectId;` field:

```java
/**
 * Tenant that owns this entry. Non-null, defaults to
 * {@link io.casehub.platform.api.identity.TenancyConstants#DEFAULT_TENANT_ID}.
 * Set at persist time by the repository — callers do not set this directly.
 */
public String tenancyId;
```

The api module already depends on `casehub-platform-api` (for `ActorType`, `ActorTypeResolver`), so `TenancyConstants` is available.

- [ ] **Step 2: Add `tenancyId` JPA column to runtime `LedgerEntry`**

In `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`, add after the `public UUID subjectId;` field (with its `@Column` annotation):

```java
@Column(name = "tenancy_id", nullable = false)
public String tenancyId;
```

- [ ] **Step 3: Update V1000 migration**

In `runtime/src/main/resources/db/ledger/migration/V1000__ledger_base_schema.sql`, add to the `ledger_entry` table definition, after the `subject_id` column:

```sql
tenancy_id VARCHAR(255) NOT NULL DEFAULT '278776f9-e1b0-46fb-9032-8bddebdcf9ce',
```

Add after the existing indexes:

```sql
CREATE INDEX idx_ledger_entry_tenancy ON ledger_entry (tenancy_id);
```

- [ ] **Step 4: Build api + runtime to verify entity compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,runtime -q`
Expected: BUILD SUCCESS (queries will fail at test time due to SPI signature mismatch — that's expected and fixed in later tasks)

- [ ] **Step 5: Commit**

```
git add api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java \
       runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java \
       runtime/src/main/resources/db/ledger/migration/V1000__ledger_base_schema.sql
git commit -m "feat(#127): add tenancyId field to LedgerEntry entity and V1000 migration (Refs #127)"
```

---

## Task 2: SPI Split — Blocking Repositories

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/repository/CrossTenantLedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerMerkleFrontierRepository.java`

- [ ] **Step 1: Add `String tenancyId` to all tenant-scoped methods on `LedgerEntryRepository`**

Every method on `LedgerEntryRepository` gets `String tenancyId` as its **last** parameter, EXCEPT for the five methods that move to cross-tenant. The methods that STAY (with tenancyId added):

```java
LedgerEntry save(LedgerEntry entry, String tenancyId);
List<LedgerEntry> findBySubjectId(UUID subjectId, String tenancyId);
List<LedgerEntry> findBySubjectIdAndTimeRange(UUID subjectId, Instant from, Instant to, String tenancyId);
Optional<LedgerEntry> findLatestBySubjectId(UUID subjectId, String tenancyId);
Optional<LedgerEntry> findEntryById(UUID id, String tenancyId);
List<LedgerAttestation> findAttestationsByEntryId(UUID ledgerEntryId, String tenancyId);
LedgerAttestation saveAttestation(LedgerAttestation attestation, String tenancyId);
List<LedgerEntry> findByActorId(String actorId, Instant from, Instant to, String tenancyId);
List<LedgerEntry> findByActorRole(String actorRole, Instant from, Instant to, String tenancyId);
List<LedgerEntry> findCausedBy(UUID entryId, String tenancyId);
List<LedgerAttestation> findAttestationsByEntryIdAndCapabilityTag(UUID entryId, String capabilityTag, String tenancyId);
List<LedgerAttestation> findAttestationsByEntryIdGlobal(UUID entryId, String tenancyId);
List<LedgerAttestation> findAttestationsByAttestorIdAndCapabilityTag(String attestorId, String capabilityTag, String tenancyId);
```

REMOVE these five methods from `LedgerEntryRepository` (they move to cross-tenant):
- `listAll()`
- `findAllEvents()`
- `findEventsByActorId(String actorId)`
- `findByTimeRange(Instant from, Instant to)`
- `findAttestationsForEntries(Set<UUID> entryIds)`

- [ ] **Step 2: Create `CrossTenantLedgerEntryRepository`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/repository/CrossTenantLedgerEntryRepository.java`:

```java
package io.casehub.ledger.runtime.repository;

import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

/**
 * Cross-tenant read operations for trust computation, health checks, retention,
 * and compliance export. Produced via {@code @CrossTenant} CDI qualifier — only
 * injectable by system-level beans with cross-tenant authority.
 */
public interface CrossTenantLedgerEntryRepository {

    List<LedgerEntry> listAll();

    List<LedgerEntry> findAllEvents();

    List<LedgerEntry> findEventsByActorId(String actorId);

    List<LedgerEntry> findByTimeRange(Instant from, Instant to);

    Map<UUID, List<LedgerAttestation>> findAttestationsForEntries(Set<UUID> entryIds);
}
```

- [ ] **Step 3: Add `String tenancyId` to `LedgerMerkleFrontierRepository`**

Update both methods:

```java
List<LedgerMerkleFrontier> findBySubjectId(UUID subjectId, String tenancyId);
void replace(UUID subjectId, List<LedgerMerkleFrontier> newFrontier, String tenancyId);
```

- [ ] **Step 4: Compile to verify interfaces**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | head -30`
Expected: Compilation errors from implementations not matching — that confirms the SPI break is real.

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/
git commit -m "feat(#127): split LedgerEntryRepository — tenant-scoped + CrossTenant, add tenancyId to frontier repo (Refs #127)"
```

---

## Task 3: SPI Split — Reactive Repositories

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/ReactiveLedgerEntryRepository.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/repository/CrossTenantReactiveLedgerEntryRepository.java`

- [ ] **Step 1: Add `String tenancyId` to tenant-scoped methods on `ReactiveLedgerEntryRepository`**

Same pattern as blocking. Add `String tenancyId` as last parameter to every method that stays. Remove the five that move to cross-tenant:

Methods that STAY (with tenancyId):
```java
Uni<LedgerEntry> save(LedgerEntry entry, String tenancyId);
Uni<List<LedgerEntry>> findBySubjectId(UUID subjectId, String tenancyId);
Uni<List<LedgerEntry>> findBySubjectIdAndTimeRange(UUID subjectId, Instant from, Instant to, String tenancyId);
Uni<Optional<LedgerEntry>> findLatestBySubjectId(UUID subjectId, String tenancyId);
Uni<Optional<LedgerEntry>> findEntryById(UUID id, String tenancyId);
Uni<List<LedgerEntry>> findByActorId(String actorId, Instant from, Instant to, String tenancyId);
Uni<List<LedgerEntry>> findByActorRole(String actorRole, Instant from, Instant to, String tenancyId);
Uni<List<LedgerEntry>> findCausedBy(UUID entryId, String tenancyId);
Uni<LedgerAttestation> saveAttestation(LedgerAttestation attestation, String tenancyId);
Uni<List<LedgerAttestation>> findAttestationsByEntryId(UUID ledgerEntryId, String tenancyId);
Uni<List<LedgerAttestation>> findAttestationsByEntryIdAndCapabilityTag(UUID entryId, String capabilityTag, String tenancyId);
Uni<List<LedgerAttestation>> findAttestationsByEntryIdGlobal(UUID entryId, String tenancyId);
Uni<List<LedgerAttestation>> findAttestationsByAttestorIdAndCapabilityTag(String attestorId, String capabilityTag, String tenancyId);
```

REMOVE these methods (move to cross-tenant — mirrors blocking split per spec):
- `listAll()`
- `findAllEvents()`
- `findEventsByActorId(String actorId)`
- `findByTimeRange(Instant, Instant)` (but see note below)
- `findAttestationsForEntries(Set<UUID>)` (but see note below)

Note: if a reactive consumer needs tenant-scoped `findByTimeRange` or `findAttestationsForEntries`, add them back with `String tenancyId` on the tenant-scoped interface. For now, following the blocking split exactly.

- [ ] **Step 2: Create `CrossTenantReactiveLedgerEntryRepository`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/repository/CrossTenantReactiveLedgerEntryRepository.java`:

```java
package io.casehub.ledger.runtime.repository;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.smallrye.mutiny.Uni;

import java.util.List;

/**
 * Reactive cross-tenant read operations. Build-gated via
 * {@code casehub.ledger.reactive.enabled=true}. Produced via {@code @CrossTenant}.
 */
public interface CrossTenantReactiveLedgerEntryRepository {

    Uni<List<LedgerEntry>> listAll();

    Uni<List<LedgerEntry>> findAllEvents();

    Uni<List<LedgerEntry>> findEventsByActorId(String actorId);

    Uni<List<LedgerEntry>> findByTimeRange(Instant from, Instant to);

    Uni<Map<UUID, List<LedgerAttestation>>> findAttestationsForEntries(Set<UUID> entryIds);
}
```

- [ ] **Step 3: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/
git commit -m "feat(#127): split ReactiveLedgerEntryRepository — tenant-scoped + CrossTenant (Refs #127)"
```

---

## Task 4: CDI Infrastructure

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/qualifier/CrossTenant.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/qualifier/LedgerSystem.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/LedgerSystemCurrentPrincipal.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/CrossTenantProducer.java`

- [ ] **Step 1: Create `@CrossTenant` qualifier**

Create `runtime/src/main/java/io/casehub/ledger/runtime/qualifier/CrossTenant.java`:

```java
package io.casehub.ledger.runtime.qualifier;

import jakarta.inject.Qualifier;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks injection points that require cross-tenant data access. Convention-based
 * marker — enforced by code review, not CDI machinery.
 */
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.TYPE, ElementType.PARAMETER})
public @interface CrossTenant {}
```

- [ ] **Step 2: Create `@LedgerSystem` qualifier**

Create `runtime/src/main/java/io/casehub/ledger/runtime/qualifier/LedgerSystem.java`:

```java
package io.casehub.ledger.runtime.qualifier;

import jakarta.inject.Qualifier;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks the system-level {@link io.casehub.platform.api.identity.CurrentPrincipal}
 * for ledger-internal use. Analogous to engine's {@code @EngineSystem}.
 */
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.TYPE, ElementType.PARAMETER})
public @interface LedgerSystem {}
```

- [ ] **Step 3: Create `LedgerSystemCurrentPrincipal`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/LedgerSystemCurrentPrincipal.java`:

```java
package io.casehub.ledger.runtime.service.identity;

import io.casehub.ledger.runtime.qualifier.LedgerSystem;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Set;

/**
 * Ledger-internal system-actor CurrentPrincipal. Always isCrossTenantAdmin().
 *
 * <p>Not @DefaultBean — accessed only via @LedgerSystem qualifier from CrossTenantProducer.
 *
 * <p>Interim: delete when casehub-platform ships a platform-level system-actor principal
 * with isCrossTenantAdmin() = true.
 */
@ApplicationScoped
@LedgerSystem
public class LedgerSystemCurrentPrincipal implements CurrentPrincipal {

    @Override
    public String actorId() {
        return "system";
    }

    @Override
    public Set<String> groups() {
        return Set.of();
    }

    @Override
    public String tenancyId() {
        return TenancyConstants.DEFAULT_TENANT_ID;
    }

    @Override
    public boolean isCrossTenantAdmin() {
        return true;
    }
}
```

- [ ] **Step 4: Create `CrossTenantProducer`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/CrossTenantProducer.java`:

```java
package io.casehub.ledger.runtime.service.identity;

import io.casehub.ledger.runtime.qualifier.CrossTenant;
import io.casehub.ledger.runtime.qualifier.LedgerSystem;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.repository.ActorIdentityBindingRepository;
import io.casehub.ledger.runtime.repository.CrossTenantLedgerEntryRepository;
import io.casehub.ledger.runtime.repository.KeyRotationRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;

/**
 * Produces @CrossTenant-qualified cross-tenant repository beans.
 *
 * <p>The @LedgerSystem LedgerSystemCurrentPrincipal check is a contract assertion:
 * if isCrossTenantAdmin() ever returns false, this producer fails at startup
 * rather than silently granting access.
 */
@ApplicationScoped
public class CrossTenantProducer {

    @Inject @LedgerSystem LedgerSystemCurrentPrincipal systemPrincipal;
    @Inject CrossTenantLedgerEntryRepository ledgerRepo;
    @Inject ActorTrustScoreRepository trustRepo;
    @Inject KeyRotationRepository keyRotationRepo;
    @Inject ActorIdentityBindingRepository identityBindingRepo;

    @Produces
    @CrossTenant
    @ApplicationScoped
    public CrossTenantLedgerEntryRepository produceLedgerRepo() {
        assertCrossTenant();
        return ledgerRepo;
    }

    @Produces
    @CrossTenant
    @ApplicationScoped
    public ActorTrustScoreRepository produceTrustRepo() {
        assertCrossTenant();
        return trustRepo;
    }

    @Produces
    @CrossTenant
    @ApplicationScoped
    public KeyRotationRepository produceKeyRotationRepo() {
        assertCrossTenant();
        return keyRotationRepo;
    }

    @Produces
    @CrossTenant
    @ApplicationScoped
    public ActorIdentityBindingRepository produceIdentityBindingRepo() {
        assertCrossTenant();
        return identityBindingRepo;
    }

    private void assertCrossTenant() {
        if (!systemPrincipal.isCrossTenantAdmin()) {
            throw new IllegalStateException(
                    "LedgerSystemCurrentPrincipal.isCrossTenantAdmin() must return true — ledger#127");
        }
    }
}
```

Note: Reactive cross-tenant producers are added in a later task, gated by `casehub.ledger.reactive.enabled`.

- [ ] **Step 5: Compile CDI infrastructure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | head -10`
Expected: Compilation errors from implementations — CDI infrastructure itself should compile.

- [ ] **Step 6: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/qualifier/ \
       runtime/src/main/java/io/casehub/ledger/runtime/service/identity/LedgerSystemCurrentPrincipal.java \
       runtime/src/main/java/io/casehub/ledger/runtime/service/identity/CrossTenantProducer.java
git commit -m "feat(#127): CDI infrastructure — @CrossTenant, @LedgerSystem, LedgerSystemCurrentPrincipal, CrossTenantProducer (Refs #127)"
```

---

## Task 5: JPA Implementations — Tenant-Scoped + Cross-Tenant

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaCrossTenantLedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerMerkleFrontierRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorIdentityBindingRepository.java`

- [ ] **Step 1: Update `JpaLedgerEntryRepository` — add tenancyId to `save()`**

Update the `save` method signature to `save(final LedgerEntry entry, final String tenancyId)`. Add tenancyId stamping before the existing `subjectId` null check:

```java
@Override
@Transactional
public LedgerEntry save(final LedgerEntry entry, final String tenancyId) {
    if (entry.subjectId == null) {
        throw new IllegalArgumentException("LedgerEntry.subjectId must not be null");
    }
    entry.tenancyId = tenancyId;
    // ... rest of existing save logic unchanged
```

- [ ] **Step 2: Update `JpaLedgerEntryRepository` — add tenancyId to all query methods**

For every tenant-scoped query method, add `String tenancyId` as last parameter and add `AND e.tenancyId = :tenancyId` to the JPQL WHERE clause plus `.setParameter("tenancyId", tenancyId)`.

Example — `findBySubjectId`:
```java
@Override
public List<LedgerEntry> findBySubjectId(final UUID subjectId, final String tenancyId) {
    return em.createQuery(
            "SELECT e FROM LedgerEntry e WHERE e.subjectId = :subjectId AND e.tenancyId = :tenancyId ORDER BY e.sequenceNumber ASC",
            LedgerEntry.class)
            .setParameter("subjectId", subjectId)
            .setParameter("tenancyId", tenancyId)
            .getResultList();
}
```

Apply the same pattern to: `findBySubjectIdAndTimeRange`, `findLatestBySubjectId`, `findEntryById` (use JPQL query instead of `em.find` to add tenancy filter), `findAttestationsByEntryId`, `saveAttestation`, `findByActorId`, `findByActorRole`, `findCausedBy`, `findAttestationsByEntryIdAndCapabilityTag`, `findAttestationsByEntryIdGlobal`, `findAttestationsByAttestorIdAndCapabilityTag`.

For `findEntryById`, replace `em.find()` with a JPQL query:
```java
@Override
public Optional<LedgerEntry> findEntryById(final UUID id, final String tenancyId) {
    return em.createQuery(
            "SELECT e FROM LedgerEntry e WHERE e.id = :id AND e.tenancyId = :tenancyId",
            LedgerEntry.class)
            .setParameter("id", id)
            .setParameter("tenancyId", tenancyId)
            .getResultStream()
            .findFirst();
}
```

For attestation queries that use `@NamedQuery`, convert to inline JPQL with the tenancyId join:
```java
@Override
public List<LedgerAttestation> findAttestationsByEntryId(final UUID ledgerEntryId, final String tenancyId) {
    return em.createQuery(
            "SELECT a FROM LedgerAttestation a JOIN LedgerEntry e ON a.ledgerEntryId = e.id " +
            "WHERE a.ledgerEntryId = :entryId AND e.tenancyId = :tenancyId ORDER BY a.occurredAt ASC",
            LedgerAttestation.class)
            .setParameter("entryId", ledgerEntryId)
            .setParameter("tenancyId", tenancyId)
            .getResultList();
}
```

For `saveAttestation`, validate the referenced entry belongs to the tenant:
```java
@Override
public LedgerAttestation saveAttestation(final LedgerAttestation attestation, final String tenancyId) {
    // Validate the referenced entry belongs to this tenant
    final LedgerEntry entry = em.createQuery(
            "SELECT e FROM LedgerEntry e WHERE e.id = :id AND e.tenancyId = :tenancyId",
            LedgerEntry.class)
            .setParameter("id", attestation.ledgerEntryId)
            .setParameter("tenancyId", tenancyId)
            .getResultStream()
            .findFirst()
            .orElse(null);
    if (entry == null) {
        throw new IllegalArgumentException(
                "LedgerEntry " + attestation.ledgerEntryId + " not found in tenant " + tenancyId);
    }

    if (attestation.attestorId != null) {
        attestation.attestorId = actorIdentityProvider.tokenise(attestation.attestorId);
    }
    em.persist(attestation);

    if (entry.actorId != null) {
        attestationRecordedEvent.fire(
                new AttestationRecordedEvent(entry.actorId, entry.id, attestation.id));
    }

    return attestation;
}
```

REMOVE the five methods that moved to cross-tenant: `listAll()`, `findAllEvents()`, `findEventsByActorId()`, `findByTimeRange()`, `findAttestationsForEntries()`.

- [ ] **Step 3: Create `JpaCrossTenantLedgerEntryRepository`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaCrossTenantLedgerEntryRepository.java`:

```java
package io.casehub.ledger.runtime.repository.jpa;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.CrossTenantLedgerEntryRepository;
import io.casehub.ledger.runtime.repository.jpa.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.privacy.ActorIdentityProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;

import java.time.Instant;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
public class JpaCrossTenantLedgerEntryRepository implements CrossTenantLedgerEntryRepository {

    @Inject
    @LedgerPersistenceUnit
    EntityManager em;

    @Inject
    ActorIdentityProvider actorIdentityProvider;

    @Override
    public List<LedgerEntry> listAll() {
        return em.createQuery("SELECT e FROM LedgerEntry e", LedgerEntry.class)
                .getResultList();
    }

    @Override
    public List<LedgerEntry> findAllEvents() {
        return em.createQuery(
                "SELECT e FROM LedgerEntry e WHERE e.entryType = :type",
                LedgerEntry.class)
                .setParameter("type", LedgerEntryType.EVENT)
                .getResultList();
    }

    @Override
    public List<LedgerEntry> findEventsByActorId(final String actorId) {
        final String token = actorIdentityProvider.tokeniseForQuery(actorId);
        return em.createQuery(
                "SELECT e FROM LedgerEntry e WHERE e.actorId = :actorId AND e.entryType = :type",
                LedgerEntry.class)
                .setParameter("actorId", token)
                .setParameter("type", LedgerEntryType.EVENT)
                .getResultList();
    }

    @Override
    public List<LedgerEntry> findByTimeRange(final Instant from, final Instant to) {
        return em.createQuery(
                "SELECT e FROM LedgerEntry e WHERE e.occurredAt >= :from AND e.occurredAt <= :to" +
                        " ORDER BY e.occurredAt ASC",
                LedgerEntry.class)
                .setParameter("from", from)
                .setParameter("to", to)
                .getResultList();
    }

    @Override
    public Map<UUID, List<LedgerAttestation>> findAttestationsForEntries(final Set<UUID> entryIds) {
        if (entryIds.isEmpty()) {
            return Collections.emptyMap();
        }
        final List<LedgerAttestation> all = em
                .createNamedQuery("LedgerAttestation.findByEntryIds", LedgerAttestation.class)
                .setParameter("entryIds", entryIds)
                .getResultList();
        return all.stream().collect(Collectors.groupingBy(a -> a.ledgerEntryId));
    }
}
```

- [ ] **Step 4: Update `JpaLedgerMerkleFrontierRepository`**

Add `String tenancyId` to both methods. The implementation can pass it through without using it in the WHERE clause (frontier is keyed by subjectId, which is unique per tenant):

```java
@Override
public List<LedgerMerkleFrontier> findBySubjectId(final UUID subjectId, final String tenancyId) {
    // tenancyId accepted for SPI parity; subjectId is unique per tenant
    return em.createQuery(/* ... existing query unchanged ... */)
            .setParameter("subjectId", subjectId)
            .getResultList();
}

@Override
@Transactional
public void replace(final UUID subjectId, final List<LedgerMerkleFrontier> newFrontier, final String tenancyId) {
    // tenancyId accepted for SPI parity; subjectId is unique per tenant
    // ... existing implementation unchanged
}
```

- [ ] **Step 5: Compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | head -20`
Expected: Remaining errors from service layer, InMemory impls, and tests — JPA layer should compile.

- [ ] **Step 6: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/
git commit -m "feat(#127): JPA implementations — tenant-scoped filtering + JpaCrossTenantLedgerEntryRepository (Refs #127)"
```

---

## Task 6: InMemory Implementations

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java`
- Create: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantLedgerEntryRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerMerkleFrontierRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryReactiveLedgerEntryRepository.java`
- Create: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantReactiveLedgerEntryRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryReactiveKeyRotationRepository.java`

- [ ] **Step 1: Update `InMemoryLedgerEntryRepository`**

Add `String tenancyId` to `save()` — stamp `entry.tenancyId = tenancyId` before existing logic.

Add `String tenancyId` to all tenant-scoped find methods — filter by `tenancyId.equals(e.tenancyId)`.

Remove the five cross-tenant methods (`listAll`, `findAllEvents`, `findEventsByActorId`, `findByTimeRange`, `findAttestationsForEntries`).

Keep the `allEntries()` method as an internal accessor for the cross-tenant delegate.

- [ ] **Step 2: Create `InMemoryCrossTenantLedgerEntryRepository`**

Create `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantLedgerEntryRepository.java`:

```java
package io.casehub.ledger.memory;

import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.CrossTenantLedgerEntryRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.annotation.Priority;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
@Alternative
@Priority(1)
public class InMemoryCrossTenantLedgerEntryRepository implements CrossTenantLedgerEntryRepository {

    @Inject
    InMemoryLedgerEntryRepository blocking;

    @Override
    public List<LedgerEntry> listAll() {
        return blocking.allEntries();
    }

    @Override
    public List<LedgerEntry> findAllEvents() {
        return blocking.allEntries().stream()
                .filter(e -> e.entryType == LedgerEntryType.EVENT)
                .toList();
    }

    @Override
    public List<LedgerEntry> findEventsByActorId(final String actorId) {
        return blocking.allEntries().stream()
                .filter(e -> e.entryType == LedgerEntryType.EVENT)
                .filter(e -> actorId.equals(e.actorId))
                .toList();
    }

    @Override
    public List<LedgerEntry> findByTimeRange(final Instant from, final Instant to) {
        return blocking.allEntries().stream()
                .filter(e -> !e.occurredAt.isBefore(from) && !e.occurredAt.isAfter(to))
                .sorted(java.util.Comparator.comparing(e -> e.occurredAt))
                .toList();
    }

    @Override
    public Map<UUID, List<LedgerAttestation>> findAttestationsForEntries(final Set<UUID> entryIds) {
        if (entryIds.isEmpty()) {
            return Collections.emptyMap();
        }
        return blocking.allAttestations().stream()
                .filter(a -> entryIds.contains(a.ledgerEntryId))
                .collect(Collectors.groupingBy(a -> a.ledgerEntryId));
    }
}
```

Note: `allAttestations()` needs to be exposed on `InMemoryLedgerEntryRepository` as an internal accessor (same pattern as `allEntries()`).

- [ ] **Step 3: Update `InMemoryLedgerMerkleFrontierRepository`**

Add `String tenancyId` to both method signatures. Ignore the parameter in the implementation (frontier is keyed by subjectId).

- [ ] **Step 4: Update `InMemoryReactiveLedgerEntryRepository`**

Update all delegating methods to pass `tenancyId` through to the blocking counterpart. Remove the three cross-tenant methods.

- [ ] **Step 5: Create `InMemoryCrossTenantReactiveLedgerEntryRepository`**

Create at `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantReactiveLedgerEntryRepository.java`. Annotate with `@IfBuildProperty(name = "casehub.ledger.reactive.enabled", stringValue = "true")`. Delegate to `InMemoryCrossTenantLedgerEntryRepository` wrapped in `Uni.createFrom().item()`.

- [ ] **Step 6: Compile persistence-memory module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl persistence-memory -q`

- [ ] **Step 7: Commit**

```
git add persistence-memory/src/main/java/io/casehub/ledger/memory/
git commit -m "feat(#127): InMemory implementations — tenant filtering + cross-tenant delegates (Refs #127)"
```

---

## Task 7: Service Layer — Thread tenancyId

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerVerificationService.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/KeyRotationService.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/ReactiveKeyRotationService.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerComplianceReportService.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvExportService.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/AgentSignatureVerificationService.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryArchiver.java`

- [ ] **Step 1: Update `LedgerVerificationService`**

Add `String tenancyId` as last parameter to `treeRoot(UUID subjectId)`, `inclusionProof(UUID subjectId, int index)`, and `verify(UUID subjectId)`. Pass through to `frontierRepo.findBySubjectId(subjectId, tenancyId)` and `ledgerRepo.findBySubjectId(subjectId, tenancyId)`.

- [ ] **Step 2: Update `KeyRotationService`**

`recordRotation()` calls `ledgerRepo.save()` — add `String tenancyId` parameter, pass through.
`rotationHistory()` and `compromisedWindows()` call `KeyRotationRepository` which is cross-tenant (actor-scoped) — no tenancyId change on those methods.

- [ ] **Step 3: Update `ReactiveKeyRotationService`**

Same pattern as blocking. `recordRotationAsync()` gains `String tenancyId`.

- [ ] **Step 4: Update `LedgerComplianceReportService`**

`reportForActor()` and `reportForSubject()` gain `String tenancyId`. Pass through to `ledgerRepo.findByActorId(..., tenancyId)` and `ledgerRepo.findBySubjectId(..., tenancyId)`.

- [ ] **Step 5: Update `LedgerProvExportService`**

`exportProvenance()` gains `String tenancyId`. Pass through to entry repo calls.

- [ ] **Step 6: Update `AgentSignatureVerificationService`**

`verifyAgentSignature()` takes a `LedgerEntry` — the entry is already loaded (caller has it). Verification is pure computation on the entry bytes. The method also calls `KeyRotationRepository` for compromise window checks — that's cross-tenant (actor-scoped). No tenancyId change needed.

Actually, check if it calls `ledgerRepo` for anything. If not, no change. If it does, add tenancyId.

- [ ] **Step 7: Update remaining services**

Scan for any other service that injects `LedgerEntryRepository` and calls methods whose signatures changed. Update each with `String tenancyId` parameter threading.

Key services to check: `LedgerRetentionJob` (already handled by @CrossTenant in Task 9), `TrustScoreJob` (Task 9), `IncrementalTrustUpdateObserver` (Task 9).

- [ ] **Step 8: Compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | head -20`

- [ ] **Step 9: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/
git commit -m "feat(#127): service layer — thread tenancyId through verification, rotation, compliance, provenance (Refs #127)"
```

---

## Task 8: Job Updates — Switch to `@CrossTenant`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/PerActorTrustComputer.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/IncrementalTrustUpdateObserver.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerRetentionJob.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustExportService.java` (federation/)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustBootstrapService.java` (federation/)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/MaterializedTrustScoreSource.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/CachedTrustScoreSource.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/ComputedTrustScoreSource.java`

- [ ] **Step 1: Update `TrustScoreJob`**

Change injection:
```java
@Inject
@CrossTenant
CrossTenantLedgerEntryRepository ledgerRepo;

@Inject
@CrossTenant
ActorTrustScoreRepository trustRepo;
```

Update method calls: `ledgerRepo.findAllEvents()`, `ledgerRepo.findAttestationsForEntries(entryIds)`, `trustRepo.findAll()`, etc. — these methods have unchanged signatures on the cross-tenant interfaces.

- [ ] **Step 2: Update `PerActorTrustComputer`**

Same pattern — inject `@CrossTenant` repos. This bean is used by both `TrustScoreJob` (no request scope) and `IncrementalTrustUpdateObserver` (`@ObservesAsync`, no request scope).

- [ ] **Step 3: Update `IncrementalTrustUpdateObserver`**

Switch to `@CrossTenant CrossTenantLedgerEntryRepository`. The observer uses `findEventsByActorId()` which is on the cross-tenant interface.

- [ ] **Step 4: Update `LedgerRetentionJob`**

Switch to `@CrossTenant CrossTenantLedgerEntryRepository`. Uses `listAll()` and `findByTimeRange()` for retention sweep.

- [ ] **Step 5: Update trust score sources**

`MaterializedTrustScoreSource`, `CachedTrustScoreSource`, `ComputedTrustScoreSource` — these read trust scores which are cross-tenant. Switch `ActorTrustScoreRepository` injection to `@CrossTenant`.

- [ ] **Step 6: Update `TrustExportService` and `TrustBootstrapService`**

Switch to `@CrossTenant ActorTrustScoreRepository`.

- [ ] **Step 7: Compile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q 2>&1 | head -20`

- [ ] **Step 8: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/
git commit -m "feat(#127): jobs + trust sources — switch to @CrossTenant injection (Refs #127)"
```

---

## Task 9: Fix All Existing Tests

**Files:**
- Modify: all test files in `runtime/src/test/java/` that call changed SPI methods
- Modify: `runtime/src/test/java/io/casehub/ledger/service/BlockingReactiveLedgerEntryRepository.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/BlockingReactiveKeyRotationRepository.java`
- Modify: `persistence-memory/src/test/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepositoryTest.java`

- [ ] **Step 1: Add `TenancyConstants.DEFAULT_TENANT_ID` to all test call sites**

Every test that calls `save()`, `findBySubjectId()`, or any changed SPI method needs `TenancyConstants.DEFAULT_TENANT_ID` as the last argument. This is mechanical — add the import and the parameter.

Pattern:
```java
import static io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID;

// Before:
repo.save(entry);
// After:
repo.save(entry, DEFAULT_TENANT_ID);
```

- [ ] **Step 2: Update test shims**

`BlockingReactiveLedgerEntryRepository` — update all method signatures to match the new `ReactiveLedgerEntryRepository`. Remove cross-tenant methods. Pass tenancyId through to blocking delegate.

`BlockingReactiveKeyRotationRepository` — no signature change (cross-tenant).

- [ ] **Step 3: Update `InMemoryLedgerEntryRepositoryTest`**

Add `DEFAULT_TENANT_ID` to all method calls. Test that filtering by different tenancyId returns empty.

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,persistence-memory`
Expected: All existing tests pass with the new tenancyId parameter (using DEFAULT_TENANT_ID).

- [ ] **Step 5: Commit**

```
git add runtime/src/test/ persistence-memory/src/test/
git commit -m "test(#127): update all existing tests for tenancyId SPI signatures (Refs #127)"
```

---

## Task 10: Tenancy Isolation Integration Test

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/TenancyIsolationIT.java`

- [ ] **Step 1: Write the tenancy isolation test**

```java
package io.casehub.ledger.service;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.PlainLedgerEntry;
import io.casehub.ledger.runtime.repository.CrossTenantLedgerEntryRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class TenancyIsolationIT {

    static final String TENANT_A = "tenant-a";
    static final String TENANT_B = "tenant-b";

    @Inject
    LedgerEntryRepository repo;

    @Inject
    CrossTenantLedgerEntryRepository crossTenantRepo;

    @Test
    @Transactional
    void tenant_a_entries_invisible_to_tenant_b() {
        final UUID subjectId = UUID.randomUUID();
        final PlainLedgerEntry entry = new PlainLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = subjectId;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = "actor-1";

        repo.save(entry, TENANT_A);

        // Tenant A sees its own entry
        assertEquals(1, repo.findBySubjectId(subjectId, TENANT_A).size());

        // Tenant B sees nothing
        assertEquals(0, repo.findBySubjectId(subjectId, TENANT_B).size());

        // Cross-tenant sees everything
        assertTrue(crossTenantRepo.listAll().stream()
                .anyMatch(e -> e.id.equals(entry.id)));
    }

    @Test
    @Transactional
    void findEntryById_wrong_tenant_returns_empty() {
        final PlainLedgerEntry entry = new PlainLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = UUID.randomUUID();
        entry.entryType = LedgerEntryType.COMMAND;
        entry.actorId = "actor-1";

        repo.save(entry, TENANT_A);

        assertTrue(repo.findEntryById(entry.id, TENANT_A).isPresent());
        assertTrue(repo.findEntryById(entry.id, TENANT_B).isEmpty());
    }

    @Test
    @Transactional
    void attestation_save_rejects_cross_tenant_entry() {
        final PlainLedgerEntry entry = new PlainLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = UUID.randomUUID();
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = "actor-1";

        repo.save(entry, TENANT_A);

        final LedgerAttestation attestation = new LedgerAttestation();
        attestation.id = UUID.randomUUID();
        attestation.ledgerEntryId = entry.id;
        attestation.attestorId = "attestor-1";
        attestation.verdict = AttestationVerdict.SOUND;
        attestation.capabilityTag = "*";

        // Saving attestation under wrong tenant should fail
        assertThrows(IllegalArgumentException.class,
                () -> repo.saveAttestation(attestation, TENANT_B));
    }

    @Test
    @Transactional
    void cross_tenant_findAllEvents_sees_all_tenants() {
        final PlainLedgerEntry entryA = new PlainLedgerEntry();
        entryA.id = UUID.randomUUID();
        entryA.subjectId = UUID.randomUUID();
        entryA.entryType = LedgerEntryType.EVENT;
        entryA.actorId = "actor-a";
        repo.save(entryA, TENANT_A);

        final PlainLedgerEntry entryB = new PlainLedgerEntry();
        entryB.id = UUID.randomUUID();
        entryB.subjectId = UUID.randomUUID();
        entryB.entryType = LedgerEntryType.EVENT;
        entryB.actorId = "actor-b";
        repo.save(entryB, TENANT_B);

        final var allEvents = crossTenantRepo.findAllEvents();
        assertTrue(allEvents.stream().anyMatch(e -> e.id.equals(entryA.id)));
        assertTrue(allEvents.stream().anyMatch(e -> e.id.equals(entryB.id)));
    }
}
```

- [ ] **Step 2: Run the isolation test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TenancyIsolationIT`
Expected: All 4 tests PASS.

- [ ] **Step 3: Commit**

```
git add runtime/src/test/java/io/casehub/ledger/service/TenancyIsolationIT.java
git commit -m "test(#127): tenancy isolation integration test — cross-tenant invisibility, wrong-tenant rejection (Refs #127)"
```

---

## Task 11: Run Full Test Suite and Fix Remaining Issues

- [ ] **Step 1: Run complete test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: ALL tests pass across api, runtime, persistence-memory modules.

- [ ] **Step 2: Fix any remaining compilation or test failures**

These will be mechanical — missed call sites, test helper methods that need the tenancyId parameter. Fix each, re-run.

- [ ] **Step 3: Final commit**

```
git add -A
git commit -m "fix(#127): remaining test fixes for multi-tenancy SPI changes (Refs #127)"
```

---

## Task 12: Update CLAUDE.md and Documentation

**Files:**
- Modify: `CLAUDE.md` — update project structure section with new files
- Modify: `docs/DESIGN.md` — add tenancy section if appropriate

- [ ] **Step 1: Update CLAUDE.md project structure**

Add the new files to the project structure tree:
- `runtime/.../qualifier/CrossTenant.java`
- `runtime/.../qualifier/LedgerSystem.java`
- `runtime/.../service/identity/LedgerSystemCurrentPrincipal.java`
- `runtime/.../service/identity/CrossTenantProducer.java`
- `runtime/.../repository/CrossTenantLedgerEntryRepository.java`
- `runtime/.../repository/CrossTenantReactiveLedgerEntryRepository.java`
- `runtime/.../repository/jpa/JpaCrossTenantLedgerEntryRepository.java`
- `persistence-memory/.../InMemoryCrossTenantLedgerEntryRepository.java`
- `persistence-memory/.../InMemoryCrossTenantReactiveLedgerEntryRepository.java`

Update the `LedgerEntry` description to mention `tenancyId`.
Update repository descriptions to note tenant-scoped vs cross-tenant split.

- [ ] **Step 2: Commit**

```
git add CLAUDE.md
git commit -m "docs(#127): update CLAUDE.md — tenancy infrastructure, SPI split, cross-tenant repos (Refs #127)"
```
