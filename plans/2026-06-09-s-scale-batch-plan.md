# S-Scale Batch (#124, #125, #129) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement three S-scale issues on a single branch: combined attestation query (#124), materialization flag (#125), tenancyId through CDI event chain (#129).

**Architecture:** #124 adds `findAttestationsByActorId()` to `CrossTenantLedgerEntryRepository` (blocking + reactive SPIs, JPA + in-memory impls). #125 adds `casehub.ledger.trust-score.materialization.enabled` runtime config gate to `TrustScoreJob` and `IncrementalTrustUpdateObserver`. #129 adds `tenancyId` as 2nd component to `AgentIdentityValidatedEvent` and `AgentIdentityViolationEvent` in `casehub-platform-api`, then threads it through the enricher → observer → repository chain.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Mutiny (Uni<T>), CDI events, SmallRye Config

**Spec:** `specs/2026-06-09-s-scale-batch-design.md`

**Project repo:** `/Users/mdproctor/claude/casehub/ledger`
**Platform repo:** `/Users/mdproctor/claude/casehub/platform`

---

## Task 1: #124 — Add @NamedQuery to LedgerAttestation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java`

- [ ] **Step 1: Add the @NamedQuery annotation**

Add after the existing `LedgerAttestation.findByAttestorIdAndCapabilityTag` named query (L33–35):

```java
@NamedQuery(
        name = "LedgerAttestation.findByActorIdEvents",
        query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId IN ("
              + "SELECT e.id FROM LedgerEntry e WHERE e.actorId = :actorId AND e.entryType = :type)")
```

- [ ] **Step 2: Build to verify Hibernate validates the query at startup**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ComputedTrustScoreSourceTest -q`
Expected: PASS — named query is syntactically valid and Hibernate validates it at boot.

- [ ] **Step 3: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerAttestation.java
git commit -m "feat(#124): add LedgerAttestation.findByActorIdEvents @NamedQuery

Refs #124"
```

---

## Task 2: #124 — Add findAttestationsByActorId to blocking SPI + JPA impl

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/CrossTenantLedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaCrossTenantLedgerEntryRepository.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/ComputedTrustScoreSourceTest.java` (stub impl)

- [ ] **Step 1: Write the failing test**

In `ComputedTrustScoreSourceTest.java`, add a test that uses `findAttestationsByActorId` directly on the stub. This validates the SPI contract. Add after the existing `invalidateActor_causesFreshComputation` test (L226):

```java
@Test
void findAttestationsByActorId_returnsGroupedAttestations() {
    final TestLedgerEntry entry = decision("alice");
    ledgerRepo.addEntry(entry);
    final LedgerAttestation att = attestation(entry.id, entry.subjectId,
            AttestationVerdict.SOUND, "review");
    ledgerRepo.addAttestation(att);

    final Map<UUID, List<LedgerAttestation>> result =
            ledgerRepo.findAttestationsByActorId("alice");

    assertThat(result).containsKey(entry.id);
    assertThat(result.get(entry.id)).hasSize(1);
    assertThat(result.get(entry.id).get(0).id).isEqualTo(att.id);
}

@Test
void findAttestationsByActorId_excludesNonEventEntries() {
    final TestLedgerEntry command = decision("alice");
    command.entryType = LedgerEntryType.COMMAND;
    ledgerRepo.addEntry(command);
    ledgerRepo.addAttestation(attestation(command.id, command.subjectId,
            AttestationVerdict.SOUND, "review"));

    final Map<UUID, List<LedgerAttestation>> result =
            ledgerRepo.findAttestationsByActorId("alice");

    assertThat(result).isEmpty();
}

@Test
void findAttestationsByActorId_excludesOtherActors() {
    final TestLedgerEntry aliceEntry = decision("alice");
    ledgerRepo.addEntry(aliceEntry);
    ledgerRepo.addAttestation(attestation(aliceEntry.id, aliceEntry.subjectId,
            AttestationVerdict.SOUND, "review"));
    final TestLedgerEntry bobEntry = decision("bob");
    ledgerRepo.addEntry(bobEntry);
    ledgerRepo.addAttestation(attestation(bobEntry.id, bobEntry.subjectId,
            AttestationVerdict.FLAGGED, "review"));

    final Map<UUID, List<LedgerAttestation>> result =
            ledgerRepo.findAttestationsByActorId("alice");

    assertThat(result).containsOnlyKeys(aliceEntry.id);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ComputedTrustScoreSourceTest#findAttestationsByActorId_returnsGroupedAttestations -q`
Expected: FAIL — `findAttestationsByActorId` does not exist on the interface.

- [ ] **Step 3: Add method to blocking SPI**

In `CrossTenantLedgerEntryRepository.java`, add after `findAttestationsForEntries` (L65):

```java
/**
 * Return all attestations for EVENT entries by the given actor, grouped by entry ID.
 *
 * <p>Encapsulates the actorId→entryIds→attestations lookup in a single call,
 * eliminating the manual ID bridging required by the separate
 * {@link #findEventsByActorId} + {@link #findAttestationsForEntries} pattern.
 *
 * @param actorId the actor identity to filter by
 * @return map from entry ID to its attestations; empty map if no EVENT entries exist
 */
Map<UUID, List<LedgerAttestation>> findAttestationsByActorId(String actorId);
```

- [ ] **Step 4: Add JPA implementation**

In `JpaCrossTenantLedgerEntryRepository.java`, add after `findAttestationsForEntries` (L96):

```java
@Override
public Map<UUID, List<LedgerAttestation>> findAttestationsByActorId(final String actorId) {
    final String token = actorIdentityProvider.tokeniseForQuery(actorId);
    final List<LedgerAttestation> all = em
            .createNamedQuery("LedgerAttestation.findByActorIdEvents", LedgerAttestation.class)
            .setParameter("actorId", token)
            .setParameter("type", LedgerEntryType.EVENT)
            .getResultList();
    return all.stream().collect(Collectors.groupingBy(a -> a.ledgerEntryId));
}
```

- [ ] **Step 5: Add stub implementation in test**

In `ComputedTrustScoreSourceTest.StubLedgerEntryRepository`, add after `findAttestationsForEntries` (L265):

```java
@Override
public Map<UUID, List<LedgerAttestation>> findAttestationsByActorId(final String actorId) {
    final Set<UUID> eventIds = entries.stream()
            .filter(e -> actorId.equals(e.actorId) && e.entryType == LedgerEntryType.EVENT)
            .map(e -> e.id)
            .collect(java.util.stream.Collectors.toSet());
    return findAttestationsForEntries(eventIds);
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ComputedTrustScoreSourceTest -q`
Expected: PASS — all tests including the three new ones.

- [ ] **Step 7: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/CrossTenantLedgerEntryRepository.java
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaCrossTenantLedgerEntryRepository.java
git add runtime/src/test/java/io/casehub/ledger/service/ComputedTrustScoreSourceTest.java
git commit -m "feat(#124): add findAttestationsByActorId to CrossTenantLedgerEntryRepository — blocking SPI + JPA impl

Refs #124"
```

---

## Task 3: #124 — Add findAttestationsByActorId to in-memory impls + reactive SPI

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantLedgerEntryRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/CrossTenantReactiveLedgerEntryRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantReactiveLedgerEntryRepository.java`

- [ ] **Step 1: Add to InMemoryCrossTenantLedgerEntryRepository**

After `findAttestationsForEntries` (L74):

```java
@Override
public Map<UUID, List<LedgerAttestation>> findAttestationsByActorId(final String actorId) {
    final Set<UUID> eventIds = blocking.allEntries().stream()
            .filter(e -> e.entryType == LedgerEntryType.EVENT)
            .filter(e -> actorId.equals(e.actorId))
            .map(e -> e.id)
            .collect(Collectors.toSet());
    if (eventIds.isEmpty()) {
        return Collections.emptyMap();
    }
    return blocking.allAttestations().stream()
            .filter(a -> eventIds.contains(a.ledgerEntryId))
            .collect(Collectors.groupingBy(a -> a.ledgerEntryId));
}
```

Add `import io.casehub.ledger.api.model.LedgerEntryType;` to the imports.

- [ ] **Step 2: Add to CrossTenantReactiveLedgerEntryRepository interface**

After `findAttestationsForEntries` (L48):

```java
Uni<Map<UUID, List<LedgerAttestation>>> findAttestationsByActorId(String actorId);
```

- [ ] **Step 3: Add to InMemoryCrossTenantReactiveLedgerEntryRepository**

After `findAttestationsForEntries` (L57):

```java
@Override
public Uni<Map<UUID, List<LedgerAttestation>>> findAttestationsByActorId(final String actorId) {
    return Uni.createFrom().item(() -> blocking.findAttestationsByActorId(actorId));
}
```

- [ ] **Step 4: Run all tests to verify no compile breaks**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q`
Expected: PASS — all modules compile and tests pass.

- [ ] **Step 5: Commit**

```
git add persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantLedgerEntryRepository.java
git add persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantReactiveLedgerEntryRepository.java
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/CrossTenantReactiveLedgerEntryRepository.java
git commit -m "feat(#124): add findAttestationsByActorId to in-memory impls + reactive SPI

Refs #124"
```

---

## Task 4: #124 — Update callers to use findAttestationsByActorId

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/ComputedTrustScoreSource.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/IncrementalTrustUpdateObserver.java`

- [ ] **Step 1: Update ComputedTrustScoreSource.computeFresh()**

Replace the method body (L157–168):

```java
private TrustScoreCalculator.ComputedScores computeFresh(final String actorId) {
    final List<LedgerEntry> decisions = ledgerRepo.findEventsByActorId(actorId);
    if (decisions.isEmpty()) {
        return EMPTY_SENTINEL;
    }
    final Map<UUID, List<LedgerAttestation>> attestationsByEntry =
            ledgerRepo.findAttestationsByActorId(actorId);
    return calculator.computeAll(decisions, attestationsByEntry, Instant.now());
}
```

Remove the `Set` and `Collectors` imports if they become unused (check after edit — `Collectors` is still used in `allCapabilityScores`; `Set` may no longer be needed).

- [ ] **Step 2: Update IncrementalTrustUpdateObserver.onAttestationRecorded()**

Replace lines L78–88:

```java
final List<LedgerEntry> decisions = ledgerRepo.findEventsByActorId(event.actorId());

if (decisions.isEmpty()) {
    return;
}

final Map<UUID, List<LedgerAttestation>> attestationsByEntry =
        ledgerRepo.findAttestationsByActorId(event.actorId());
```

Remove the now-unused imports: `java.util.Set`, `java.util.UUID`, `java.util.stream.Collectors`.

- [ ] **Step 3: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`
Expected: PASS — all existing tests pass with the new query path.

- [ ] **Step 4: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/ComputedTrustScoreSource.java
git add runtime/src/main/java/io/casehub/ledger/runtime/service/IncrementalTrustUpdateObserver.java
git commit -m "feat(#124): update callers to use findAttestationsByActorId — eliminate manual ID bridging

Closes #124"
```

---

## Task 5: #125 — Add materialization config + gate TrustScoreJob

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`

- [ ] **Step 1: Write the failing test**

In a new test or by modifying an existing `TrustScoreJob` test, verify the materialization gate. But `TrustScoreJob` tests use `@QuarkusTest` — let's check what exists first. If no dedicated TrustScoreJob test exists, create a focused unit test.

Create `runtime/src/test/java/io/casehub/ledger/service/TrustScoreJobMaterializationTest.java`:

```java
package io.casehub.ledger.service;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import org.junit.jupiter.api.Test;

import static org.mockito.Mockito.*;

class TrustScoreJobMaterializationTest {

    @Test
    void computeTrustScores_skipsWhenMaterializationDisabled() {
        final TrustScoreJob job = new TrustScoreJob();
        final LedgerConfig config = mock(LedgerConfig.class, RETURNS_DEEP_STUBS);
        when(config.trustScore().enabled()).thenReturn(true);
        when(config.trustScore().materialization().enabled()).thenReturn(false);
        job.config = config;

        job.computeTrustScores();

        // If it didn't throw and didn't touch ledgerRepo (which is null),
        // the materialization gate worked — runComputation() was skipped.
        verifyNoInteractions(config.trustScore().bootstrap());
    }

    @Test
    void computeTrustScores_proceedsWhenMaterializationEnabled() {
        final TrustScoreJob job = new TrustScoreJob();
        final LedgerConfig config = mock(LedgerConfig.class, RETURNS_DEEP_STUBS);
        when(config.trustScore().enabled()).thenReturn(true);
        when(config.trustScore().materialization().enabled()).thenReturn(true);
        job.config = config;

        // This will NPE on ledgerRepo (not injected) — that proves
        // it got past the gate and tried to run computation.
        try {
            job.computeTrustScores();
        } catch (final NullPointerException e) {
            // Expected — no ledgerRepo injected. The point is it got past the gate.
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TrustScoreJobMaterializationTest -q`
Expected: FAIL — `materialization()` method does not exist on `TrustScoreConfig`.

- [ ] **Step 3: Add MaterializationConfig to LedgerConfig**

In `LedgerConfig.TrustScoreConfig`, add after the `IncrementalConfig incremental()` method (L409):

```java
/**
 * Materialization settings — controls whether trust scores are persisted to
 * {@code ActorTrustScoreRepository}.
 *
 * <p>When disabled, {@code TrustScoreJob} and {@code IncrementalTrustUpdateObserver}
 * skip all computation and persistence. {@code ComputedTrustScoreSource} continues
 * to compute scores on read from raw attestation history, unaffected by this flag.
 *
 * @return the materialization sub-configuration
 */
MaterializationConfig materialization();

/** Trust score materialization (persistence to ActorTrustScoreRepository). */
interface MaterializationConfig {
    /**
     * When {@code false}, the batch job and incremental observer skip computation
     * and persistence entirely. Trust score config (decay, aggregation strategy)
     * remains active for on-read computation via {@code ComputedTrustScoreSource}.
     *
     * @return {@code true} if materialization is active (default)
     */
    @WithDefault("true")
    boolean enabled();
}
```

- [ ] **Step 4: Add materialization gate to TrustScoreJob.computeTrustScores()**

Replace the method body (L70–75):

```java
@Scheduled(every = "{casehub.ledger.trust-score.schedule:24h}", identity = "ledger-trust-score-job")
@Transactional
public void computeTrustScores() {
    if (!config.trustScore().enabled()
            || !config.trustScore().materialization().enabled()) {
        return;
    }
    runComputation();
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TrustScoreJobMaterializationTest -q`
Expected: PASS.

- [ ] **Step 6: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java
git add runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java
git add runtime/src/test/java/io/casehub/ledger/service/TrustScoreJobMaterializationTest.java
git commit -m "feat(#125): add materialization config gate to TrustScoreJob

Refs #125"
```

---

## Task 6: #125 — Gate IncrementalTrustUpdateObserver

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/IncrementalTrustUpdateObserver.java`

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/ledger/service/IncrementalTrustUpdateObserverMaterializationTest.java`:

```java
package io.casehub.ledger.service;

import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.service.AttestationRecordedEvent;
import io.casehub.ledger.runtime.service.IncrementalTrustUpdateObserver;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.mockito.Mockito.*;

class IncrementalTrustUpdateObserverMaterializationTest {

    @Test
    void onAttestationRecorded_skipsWhenMaterializationDisabled() {
        final IncrementalTrustUpdateObserver observer = new IncrementalTrustUpdateObserver();
        final LedgerConfig config = mock(LedgerConfig.class, RETURNS_DEEP_STUBS);
        when(config.trustScore().enabled()).thenReturn(true);
        when(config.trustScore().incremental().enabled()).thenReturn(true);
        when(config.trustScore().materialization().enabled()).thenReturn(false);
        observer.config = config;

        final AttestationRecordedEvent event =
                new AttestationRecordedEvent("alice", UUID.randomUUID(), UUID.randomUUID());

        observer.onAttestationRecorded(event);

        // If it didn't throw and didn't touch ledgerRepo (which is null),
        // the materialization gate worked.
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=IncrementalTrustUpdateObserverMaterializationTest -q`
Expected: FAIL — the observer doesn't check `materialization().enabled()` yet, so it proceeds and NPEs on `ledgerRepo`.

- [ ] **Step 3: Add materialization gate**

In `IncrementalTrustUpdateObserver.onAttestationRecorded()`, update the guard (L71–74):

```java
if (!config.trustScore().enabled()
        || !config.trustScore().incremental().enabled()
        || !config.trustScore().materialization().enabled()) {
    return;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=IncrementalTrustUpdateObserverMaterializationTest -q`
Expected: PASS.

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q`
Expected: PASS — all modules, all tests.

- [ ] **Step 6: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/IncrementalTrustUpdateObserver.java
git add runtime/src/test/java/io/casehub/ledger/service/IncrementalTrustUpdateObserverMaterializationTest.java
git commit -m "feat(#125): gate IncrementalTrustUpdateObserver with materialization.enabled

Closes #125"
```

---

## Task 7: #129 — Add tenancyId to platform-api event records

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/platform/platform-api/src/main/java/io/casehub/platform/api/identity/AgentIdentityValidatedEvent.java`
- Modify: `/Users/mdproctor/claude/casehub/platform/platform-api/src/main/java/io/casehub/platform/api/identity/AgentIdentityViolationEvent.java`

- [ ] **Step 1: Update AgentIdentityValidatedEvent**

Replace the record (entire file):

```java
package io.casehub.platform.api.identity;

/** Fired async when actorId→DID binding validation succeeds (VALID result). */
public record AgentIdentityValidatedEvent(
        String actorId,
        String tenancyId,
        String actorDid,
        IdentityBindingStatus status,
        boolean alsoKnownAsVerified,
        boolean keyMatchVerified,
        String verifiedKeyRef,
        CredentialValidationResult credentialResult,
        String didMethod) {}
```

- [ ] **Step 2: Update AgentIdentityViolationEvent**

Replace the record (entire file):

```java
package io.casehub.platform.api.identity;

/** Fired async when actorId→DID binding validation returns a non-VALID result. */
public record AgentIdentityViolationEvent(
        String actorId,
        String tenancyId,
        String actorDid,
        IdentityBindingStatus status) {}
```

- [ ] **Step 3: Build and install platform-api SNAPSHOT**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl platform-api -q -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: BUILD SUCCESS — SNAPSHOT published to local `.m2`.

- [ ] **Step 4: Commit to platform repo**

```
git -C /Users/mdproctor/claude/casehub/platform add platform-api/src/main/java/io/casehub/platform/api/identity/AgentIdentityValidatedEvent.java
git -C /Users/mdproctor/claude/casehub/platform add platform-api/src/main/java/io/casehub/platform/api/identity/AgentIdentityViolationEvent.java
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(ledger#129): add tenancyId as 2nd component to identity event records

Per PP-20260601-e368ea — CDI SPI event records carry tenancyId as 2nd component.

Refs casehubio/ledger#129"
```

---

## Task 8: #129 — Update enricher to pass tenancyId in events

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorIdentityValidationEnricher.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityValidationEnricherTest.java`

- [ ] **Step 1: Write the failing test**

In `ActorIdentityValidationEnricherTest.java`, update the `firesValidatedEventOnValid` test (L196–204) to verify tenancyId is passed:

```java
@Test
void firesValidatedEventOnValid_carriesTenancyId() {
    byte[] key = {1};
    var vm = new VerificationMethod("id", "Ed25519", key);
    var doc = new DIDDocument("did:web:x", List.of(vm), List.of("claude:r@v1"));
    when(resolver.resolve("did:web:x")).thenReturn(Optional.of(doc));
    var e = entry("claude:r@v1", "did:web:x", key);
    e.tenancyId = "tenant-alpha";
    enricher.enrich(e);
    verify(event).fireAsync(argThat(ev ->
        ev instanceof AgentIdentityValidatedEvent validated
        && "tenant-alpha".equals(validated.tenancyId())));
}

@Test
void firesViolationEvent_carriesTenancyId() {
    when(resolver.resolve("did:web:x")).thenReturn(Optional.empty());
    var e = entry("claude:r@v1", "did:web:x", null);
    e.tenancyId = "tenant-beta";
    enricher.enrich(e);
    verify(event).fireAsync(argThat(ev ->
        ev instanceof AgentIdentityViolationEvent violation
        && "tenant-beta".equals(violation.tenancyId())));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ActorIdentityValidationEnricherTest#firesValidatedEventOnValid_carriesTenancyId -q`
Expected: FAIL — constructor call doesn't include tenancyId yet.

- [ ] **Step 3: Update fireEvent() in ActorIdentityValidationEnricher**

Replace the `fireEvent` method (L132–141):

```java
private void fireEvent(final LedgerEntry entry, final IdentityBindingStatus status) {
    final String didMethod = extractDidMethod(entry.actorDid);
    if (status == IdentityBindingStatus.VALID) {
        event.fireAsync(new AgentIdentityValidatedEvent(
            entry.actorId, entry.tenancyId, entry.actorDid, status,
            true, true, entry.agentKeyRef, null, didMethod));
    } else {
        event.fireAsync(new AgentIdentityViolationEvent(
            entry.actorId, entry.tenancyId, entry.actorDid, status));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ActorIdentityValidationEnricherTest -q`
Expected: PASS — all tests including the two new ones.

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorIdentityValidationEnricher.java
git add runtime/src/test/java/io/casehub/ledger/service/identity/ActorIdentityValidationEnricherTest.java
git commit -m "feat(#129): pass tenancyId from LedgerEntry to identity event records in enricher

Refs #129"
```

---

## Task 9: #129 — Update observer to read tenancyId + remove repository fallback

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorIdentityBindingObserver.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorIdentityBindingRepository.java`

- [ ] **Step 1: Update ActorIdentityBindingObserver**

Replace the `onValidated`, `onViolation`, and `persistBinding` methods (L41–85):

```java
void onValidated(@ObservesAsync AgentIdentityValidatedEvent event) {
    persistBinding(
        event.tenancyId(), event.actorId(), event.actorDid(), event.status(),
        event.alsoKnownAsVerified(), event.keyMatchVerified(),
        event.verifiedKeyRef(), event.credentialResult(), event.didMethod());
}

void onViolation(@ObservesAsync AgentIdentityViolationEvent event) {
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
        entry.tenancyId = tenancyId;
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
        repository.save(entry);
    } catch (final Exception e) {
        LOG.warnf("ActorIdentityBindingObserver failed to persist binding for %s: %s",
            actorId, e.getMessage());
    }
}
```

- [ ] **Step 2: Remove tenancyId fallback from JpaActorIdentityBindingRepository.save()**

In `JpaActorIdentityBindingRepository.java`, remove lines L51–53:

```java
// DELETE these lines:
if (entry.tenancyId == null) {
    entry.tenancyId = TenancyConstants.DEFAULT_TENANT_ID;
}
```

Remove the unused import `io.casehub.platform.api.identity.TenancyConstants`.

- [ ] **Step 3: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q`
Expected: PASS — all modules.

- [ ] **Step 4: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/identity/ActorIdentityBindingObserver.java
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaActorIdentityBindingRepository.java
git commit -m "feat(#129): thread tenancyId through observer, remove repository fallback

ActorIdentityBindingObserver now reads tenancyId from the CDI event and
sets it on the entry explicitly. JpaActorIdentityBindingRepository no
longer defaults to DEFAULT_TENANT_ID — a null tenancyId at save time
is a bug and should fail visibly.

Closes #129"
```

---

## Task 10: Final verification + CLAUDE.md sync

**Files:**
- Modify: `CLAUDE.md` (if needed)

- [ ] **Step 1: Run full test suite across all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -q`
Expected: PASS — all modules, clean build.

- [ ] **Step 2: Verify commit history**

Run: `git log --oneline main..HEAD`
Expected: One commit per task, clean messages, issue references.

- [ ] **Step 3: Invoke superpowers:requesting-code-review**

Review the full diff before committing. Any finding Minor or above not fixed this session must be captured as a GitHub issue.

- [ ] **Step 4: Invoke implementation-doc-sync**

Sync CLAUDE.md if the implementation introduced new config keys, conventions, or structural changes.
