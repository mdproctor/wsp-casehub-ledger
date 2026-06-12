# System Tokenisation Exemption Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Exempt non-human actors from GDPR pseudonymisation by adding `ActorType` to the `ActorIdentityProvider.tokenise()` SPI and implementing HUMAN-only tokenisation.

**Architecture:** The tokenisation policy moves from 4 scattered repository call sites into the `ActorIdentityProvider` SPI. `InternalActorIdentityProvider.tokenise(rawActorId, actorType)` tokenises when `actorType == HUMAN || actorType == null`; all other types return the raw string unchanged. Repository call sites simplify — they pass `actorType` and let the SPI decide.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM, H2 (test), JUnit 5, AssertJ

**Spec:** `specs/issue-130-system-tokenisation-exempt/2026-06-12-system-tokenisation-exempt-design.md`

---

### Task 1: Change SPI signature and update implementations

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/ActorIdentityProvider.java:30`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/InternalActorIdentityProvider.java:33-50`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/privacy/PassThroughActorIdentityProvider.java:8-11`
- Test: `runtime/src/test/java/io/casehub/ledger/privacy/PassThroughPrivacyTest.java`

- [ ] **Step 1: Write failing unit tests for PassThroughActorIdentityProvider**

Add three new tests to `PassThroughPrivacyTest.java`. These test the new signature — they will not compile until the SPI is updated.

```java
@Test
void tokenise_withHumanType_returnsRawActorId_unchanged() {
    assertThat(provider.tokenise("alice@example.com", ActorType.HUMAN)).isEqualTo("alice@example.com");
}

@Test
void tokenise_withSystemType_returnsRawActorId_unchanged() {
    assertThat(provider.tokenise("system:health-check", ActorType.SYSTEM)).isEqualTo("system:health-check");
}

@Test
void tokenise_withNullType_returnsRawActorId_unchanged() {
    assertThat(provider.tokenise("alice@example.com", null)).isEqualTo("alice@example.com");
}
```

Add import: `import io.casehub.platform.api.identity.ActorType;`

Update the two existing `tokenise` tests to use the new 2-arg signature with `ActorType.HUMAN`:

```java
@Test
void tokenise_returnsRawActorId_unchanged() {
    assertThat(provider.tokenise("alice@example.com", ActorType.HUMAN)).isEqualTo("alice@example.com");
}

@Test
void tokenise_nullSafe_returnsNull() {
    assertThat(provider.tokenise(null, ActorType.HUMAN)).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest='PassThroughPrivacyTest' -DfailIfNoTests=false`

Expected: Compilation error — `tokenise(String, ActorType)` does not exist.

- [ ] **Step 3: Update ActorIdentityProvider SPI signature**

In `ActorIdentityProvider.java`, replace the `tokenise` method:

```java
/**
 * Returns a token to store in place of {@code rawActorId} on write.
 * Creates a new mapping if one does not yet exist.
 * Called on every {@code save()} and {@code saveAttestation()}.
 *
 * <p>Only {@link ActorType#HUMAN} actors (and null actorType as a safe default)
 * are tokenised. Non-human actors (SYSTEM, AGENT) are returned unchanged —
 * they are not natural persons and have no GDPR pseudonymisation obligation.
 *
 * <p>
 * <strong>Reactive constraint:</strong> implementations must be non-blocking. In reactive
 * persistence paths this method is called synchronously on the Vert.x event loop without
 * a worker-pool hop. A blocking implementation (e.g. one backed by a JPA query) will
 * stall the event loop. The built-in implementations are non-blocking. Refs #106.
 *
 * @param rawActorId the real actor identity; may be {@code null}
 * @param actorType the type of actor; {@code null} is treated as potentially human (tokenised)
 * @return token to store, or {@code null} if input is {@code null}
 */
String tokenise(String rawActorId, ActorType actorType);
```

Add import: `import io.casehub.platform.api.identity.ActorType;`

- [ ] **Step 4: Update PassThroughActorIdentityProvider**

Replace the `tokenise` method:

```java
@Override
public String tokenise(final String rawActorId, final ActorType actorType) {
    return rawActorId;
}
```

Add import: `import io.casehub.platform.api.identity.ActorType;`

- [ ] **Step 5: Update InternalActorIdentityProvider**

Replace the `tokenise` method:

```java
@Override
public String tokenise(final String rawActorId, final ActorType actorType) {
    if (rawActorId == null) {
        return null;
    }
    if (actorType != null && actorType != ActorType.HUMAN) {
        return rawActorId;
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
```

Add import: `import io.casehub.platform.api.identity.ActorType;`

- [ ] **Step 6: Run PassThroughPrivacyTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest='PassThroughPrivacyTest' -DfailIfNoTests=false`

Expected: All 8 tests PASS (5 existing + 3 new). Note: the project will not compile fully yet — repository call sites still use the old signature.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/privacy/ActorIdentityProvider.java \
       runtime/src/main/java/io/casehub/ledger/runtime/privacy/InternalActorIdentityProvider.java \
       runtime/src/main/java/io/casehub/ledger/runtime/privacy/PassThroughActorIdentityProvider.java \
       runtime/src/test/java/io/casehub/ledger/privacy/PassThroughPrivacyTest.java
git commit -m "feat(#130): add ActorType to ActorIdentityProvider.tokenise() — HUMAN-only tokenisation

Refs #130"
```

---

### Task 2: Update InternalActorIdentityProviderIT

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/privacy/InternalActorIdentityProviderIT.java`

- [ ] **Step 1: Update existing tests to use new signature**

All existing calls to `provider.tokenise("...")` must become `provider.tokenise("...", ActorType.HUMAN)`. Update these methods:

- `tokenise_createsToken_differentFromRawActorId`: `provider.tokenise("alice@example.com", ActorType.HUMAN)`
- `tokenise_sameActorId_returnsSameToken`: both calls add `, ActorType.HUMAN`
- `tokenise_differentActorIds_returnsDifferentTokens`: both calls add `, ActorType.HUMAN`
- `tokenise_null_returnsNull`: `provider.tokenise(null, ActorType.HUMAN)`
- `tokeniseForQuery_existingActor_returnsToken`: the `tokenise` call on line 81 adds `, ActorType.HUMAN`
- `resolve_existingToken_returnsRealIdentity`: the `tokenise` call on line 100 adds `, ActorType.HUMAN`
- `erase_severed_resolveReturnsEmpty`: the `tokenise` call on line 126 adds `, ActorType.HUMAN`
- `tokeniseForQuery_afterErase_returnsRawActorId`: the `tokenise` call on line 147 adds `, ActorType.HUMAN`

- [ ] **Step 2: Add SYSTEM exemption test**

```java
@Test
@Transactional
void tokenise_systemActor_returnsRawActorId() {
    final String rawActorId = "system:health-check";
    final String result = provider.tokenise(rawActorId, ActorType.SYSTEM);
    assertThat(result).isEqualTo(rawActorId);
}
```

- [ ] **Step 3: Add AGENT exemption test**

```java
@Test
@Transactional
void tokenise_agentActor_returnsRawActorId() {
    final String rawActorId = "claude:tarkus-reviewer@v1";
    final String result = provider.tokenise(rawActorId, ActorType.AGENT);
    assertThat(result).isEqualTo(rawActorId);
}
```

- [ ] **Step 4: Add null actorType safe-default test**

```java
@Test
@Transactional
void tokenise_nullActorType_tokenises() {
    final String rawActorId = "unknown-" + java.util.UUID.randomUUID();
    final String result = provider.tokenise(rawActorId, null);
    assertThat(result).isNotNull().isNotEqualTo(rawActorId);
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest='InternalActorIdentityProviderIT' -DfailIfNoTests=false`

Expected: All tests PASS (12 existing updated + 3 new = 15 tests). Note: full project still won't compile — repository call sites not yet updated.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/privacy/InternalActorIdentityProviderIT.java
git commit -m "test(#130): update InternalActorIdentityProviderIT for ActorType-aware tokenisation

Refs #130"
```

---

### Task 3: Update repository call sites

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java:117-118,236-237`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java:98-99,191-192`

- [ ] **Step 1: Update JpaLedgerEntryRepository.save()**

Replace lines 117-119:
```java
if (entry.actorId != null) {
    entry.actorId = actorIdentityProvider.tokenise(entry.actorId);
}
```

With:
```java
entry.actorId = actorIdentityProvider.tokenise(entry.actorId, entry.actorType);
```

- [ ] **Step 2: Update JpaLedgerEntryRepository.saveAttestation()**

Replace lines 236-238:
```java
if (attestation.attestorId != null) {
    attestation.attestorId = actorIdentityProvider.tokenise(attestation.attestorId);
}
```

With:
```java
attestation.attestorId = actorIdentityProvider.tokenise(
        attestation.attestorId, attestation.attestorType);
```

- [ ] **Step 3: Update InMemoryLedgerEntryRepository.save()**

Replace lines 98-100:
```java
if (entry.actorId != null) {
    entry.actorId = actorIdentityProvider.tokenise(entry.actorId);
}
```

With:
```java
entry.actorId = actorIdentityProvider.tokenise(entry.actorId, entry.actorType);
```

- [ ] **Step 4: Update InMemoryLedgerEntryRepository.saveAttestation()**

Replace lines 191-193:
```java
if (attestation.attestorId != null) {
    attestation.attestorId = actorIdentityProvider.tokenise(attestation.attestorId);
}
```

With:
```java
attestation.attestorId = actorIdentityProvider.tokenise(
        attestation.attestorId, attestation.attestorType);
```

- [ ] **Step 5: Verify full project compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime,persistence-memory`

Expected: BUILD SUCCESS — all call sites now use the new signature.

- [ ] **Step 6: Run existing tests as regression check**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest='LedgerPrivacyWiringIT' -DfailIfNoTests=false`

Expected: Some existing tests will FAIL — `LedgerPrivacyWiringIT.entry()` fixture uses `ActorType.AGENT`, and AGENT actors are no longer tokenised. `save_actorIdStoredAsToken_notRawIdentity` and `saveAttestation_attestorIdStoredAsToken` will fail. This is expected — the fixture is updated in Task 4. Proceed to commit.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java \
       persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java
git commit -m "refactor(#130): pass ActorType to tokenise() at all repository call sites

Refs #130"
```

---

### Task 4: Update JPA integration tests — privacy wiring

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/privacy/LedgerPrivacyWiringIT.java`

- [ ] **Step 1: Fix existing fixture — entry() uses ActorType.AGENT**

The `entry()` helper on line 181 sets `e.actorType = ActorType.AGENT`. After Task 3, AGENT actors are no longer tokenised. Update the helper to default to `ActorType.HUMAN` and add an overload:

```java
private TestEntry entry(final String actorId) {
    return entry(actorId, ActorType.HUMAN);
}

private TestEntry entry(final String actorId, final ActorType actorType) {
    final TestEntry e = new TestEntry();
    e.subjectId = UUID.randomUUID();
    e.sequenceNumber = 1;
    e.entryType = LedgerEntryType.EVENT;
    e.actorId = actorId;
    e.actorType = actorType;
    e.actorRole = "Classifier";
    e.occurredAt = Instant.now();
    return e;
}
```

- [ ] **Step 2: Run existing tests to verify regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest='LedgerPrivacyWiringIT' -DfailIfNoTests=false`

Expected: All 6 existing tests PASS (fixture now uses HUMAN, so tokenisation still applies).

- [ ] **Step 3: Add SYSTEM entry save test**

```java
@Test
@Transactional
void save_systemActorId_storedRaw_notTokenised() {
    final String rawActorId = "system:health-check";
    final TestEntry entry = entry(rawActorId, ActorType.SYSTEM);
    repo.save(entry, DEFAULT_TENANT_ID);

    final LedgerEntry stored = em.find(LedgerEntry.class, entry.id);
    assertThat(stored.actorId).isEqualTo(rawActorId);
}
```

- [ ] **Step 4: Add SYSTEM attestation save test**

```java
@Test
@Transactional
void saveAttestation_systemAttestor_storedRaw_notTokenised() {
    final String rawAttestorId = "system:compliance-engine";
    final TestEntry entry = entry("human-actor-" + UUID.randomUUID());
    repo.save(entry, DEFAULT_TENANT_ID);

    final LedgerAttestation att = new LedgerAttestation();
    att.id = UUID.randomUUID();
    att.ledgerEntryId = entry.id;
    att.subjectId = entry.subjectId;
    att.attestorId = rawAttestorId;
    att.attestorType = ActorType.SYSTEM;
    att.verdict = AttestationVerdict.SOUND;
    att.confidence = 1.0;
    att.occurredAt = Instant.now();
    repo.saveAttestation(att, DEFAULT_TENANT_ID);

    final LedgerAttestation stored = em.find(LedgerAttestation.class, att.id);
    assertThat(stored.attestorId).isEqualTo(rawAttestorId);
}
```

- [ ] **Step 5: Add findByActorId round-trip for SYSTEM**

```java
@Test
@Transactional
void findByActorId_systemActor_roundTrip_worksWithoutTokenisation() {
    final String rawActorId = "system:scheduler";
    final Instant from = Instant.now().minus(1, ChronoUnit.HOURS);
    final Instant to = Instant.now().plus(1, ChronoUnit.HOURS);

    repo.save(entry(rawActorId, ActorType.SYSTEM), DEFAULT_TENANT_ID);

    final List<LedgerEntry> results = repo.findByActorId(rawActorId, from, to, DEFAULT_TENANT_ID);
    assertThat(results).hasSize(1);
    assertThat(results.get(0).actorId).isEqualTo(rawActorId);
}
```

- [ ] **Step 6: Run all privacy wiring tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest='LedgerPrivacyWiringIT' -DfailIfNoTests=false`

Expected: All 9 tests PASS (6 existing + 3 new).

- [ ] **Step 7: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/privacy/LedgerPrivacyWiringIT.java
git commit -m "test(#130): add SYSTEM/AGENT tokenisation exemption tests to LedgerPrivacyWiringIT

Refs #130"
```

---

### Task 5: Update erasure and OutcomeRecorder tests

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/privacy/LedgerErasureServiceIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderDefaultAttestorIT.java`

- [ ] **Step 1: Add erasure no-op test for SYSTEM actor**

In `LedgerErasureServiceIT.java`, add a test and update the `saveEntry` helper to accept an `ActorType` parameter:

Update the existing `saveEntry` helper and add an overload:

```java
private void saveEntry(final String actorId) {
    saveEntry(actorId, ActorType.AGENT);
}

private void saveEntry(final String actorId, final ActorType actorType) {
    final TestEntry entry = new TestEntry();
    entry.subjectId = UUID.randomUUID();
    entry.sequenceNumber = 1;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = actorId;
    entry.actorType = actorType;
    entry.actorRole = "Classifier";
    entry.occurredAt = Instant.now();
    repo.save(entry, DEFAULT_TENANT_ID);
}
```

Then add:

```java
@Test
@Transactional
void erase_systemActor_neverTokenised_mappingNotFound() {
    saveEntry("system:health-check", ActorType.SYSTEM);

    final ErasureResult result = erasureService.erase("system:health-check");

    assertThat(result.mappingFound()).isFalse();
    assertThat(result.affectedEntryCount()).isZero();
}
```

- [ ] **Step 2: Run erasure tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest='LedgerErasureServiceIT' -DfailIfNoTests=false`

Expected: 4 existing + 1 new = 5 tests. The existing tests use `ActorType.AGENT` in the helper — under the pseudonymisation profile, AGENT actors are no longer tokenised, so `erase_knownActor_mappingFound_correctCount` will fail. The default overload above already fixes this (`ActorType.HUMAN`). Verify all 5 PASS.

- [ ] **Step 3: Add OutcomeRecorder SYSTEM attestor test**

The existing `OutcomeRecorderDefaultAttestorIT` runs under the `outcome-recorder-test` profile which does NOT have tokenisation enabled. To test the SYSTEM exemption under tokenisation, add a new test to `LedgerPrivacyWiringIT.java` (which uses the pseudonymisation profile):

In `LedgerPrivacyWiringIT.java`, add:

```java
@Inject
io.casehub.ledger.api.spi.OutcomeRecorder outcomeRecorder;
```

And the test:

```java
@Test
@Transactional
void outcomeRecorder_systemAttestor_storedRaw_underTokenisation() {
    final UUID subjectId = UUID.randomUUID();
    final var record = io.casehub.ledger.api.model.OutcomeRecord.of(
            "human-actor-" + UUID.randomUUID(), subjectId, "code-review",
            AttestationVerdict.SOUND, 0.9)
            .withAttestor("system:compliance-engine", ActorType.SYSTEM);

    outcomeRecorder.record(record);

    final var entries = repo.findBySubjectId(subjectId, DEFAULT_TENANT_ID);
    assertThat(entries).hasSize(1);
    final var attestations = repo.findAttestationsByEntryId(entries.get(0).id, DEFAULT_TENANT_ID);
    assertThat(attestations).hasSize(1);
    assertThat(attestations.get(0).attestorId).isEqualTo("system:compliance-engine");
}
```

Add import: `import io.casehub.ledger.api.model.OutcomeRecord;` (if not already present via the fully qualified usage above).

- [ ] **Step 4: Run all affected tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest='LedgerErasureServiceIT,LedgerPrivacyWiringIT,OutcomeRecorderDefaultAttestorIT' -DfailIfNoTests=false`

Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/privacy/LedgerErasureServiceIT.java \
       runtime/src/test/java/io/casehub/ledger/privacy/LedgerPrivacyWiringIT.java
git commit -m "test(#130): add erasure no-op and OutcomeRecorder SYSTEM attestor tests

Refs #130"
```

---

### Task 6: Update in-memory test helpers and run full test suite

**Files:**
- Modify: `persistence-memory/src/test/java/io/casehub/ledger/memory/LedgerAttestationAccessor.java:22-33`
- Modify: `persistence-memory/src/test/java/io/casehub/ledger/memory/MemoryTestEntry.java`

- [ ] **Step 1: Add withActorType to MemoryTestEntry**

Add a fluent mutator after the existing `withActorRole` method:

```java
/** Override actorType. */
public MemoryTestEntry withActorType(final ActorType newType) {
    this.actorType = newType;
    return this;
}
```

- [ ] **Step 2: Add actorType parameter to LedgerAttestationAccessor.create()**

Add an overload that accepts `ActorType`:

```java
/** Create a test attestation with explicit attestorType. */
public static LedgerAttestation create(final UUID entryId, final UUID subjectId,
        final String attestorId, final String capabilityTag, final ActorType attestorType) {
    final LedgerAttestationAccessor a = new LedgerAttestationAccessor();
    a.ledgerEntryId = entryId;
    a.subjectId = subjectId;
    a.attestorId = attestorId;
    a.attestorType = attestorType;
    a.verdict = AttestationVerdict.SOUND;
    a.confidence = 1.0;
    a.capabilityTag = capabilityTag;
    return a;
}
```

- [ ] **Step 3: Run full test suite (excluding Docker-only tests)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest='!JpaSequenceNumberPgIT,!LedgerHealthJobPgIT'`

Expected: BUILD SUCCESS — all tests pass. Any remaining compilation or assertion failures need investigation here.

- [ ] **Step 4: Commit**

```bash
git add persistence-memory/src/test/java/io/casehub/ledger/memory/MemoryTestEntry.java \
       persistence-memory/src/test/java/io/casehub/ledger/memory/LedgerAttestationAccessor.java
git commit -m "refactor(#130): add actorType helpers to MemoryTestEntry and LedgerAttestationAccessor

Refs #130"
```

---

### Task 7: Update CLAUDE.md and DESIGN.md

**Files:**
- Modify: `CLAUDE.md`
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Update CLAUDE.md**

In the `ActorIdentityProvider` entry in the project structure tree, update the javadoc description to reflect the ActorType-aware tokenisation.

In the `Key Design Decisions` section, find the privacy/pseudonymisation references and add a note:

Find `ActorIdentityProvider.java` in the project structure and update:
```
│           ├── ActorIdentityProvider.java   — SPI: tokenise/resolve/erase actor identities; tokenise takes ActorType — only HUMAN actors are pseudonymised
```

- [ ] **Step 2: Update docs/DESIGN.md**

In the Implementation Tracker or Privacy section, note that tokenisation is now ActorType-aware.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md docs/DESIGN.md
git commit -m "docs(#130): update CLAUDE.md and DESIGN.md for ActorType-aware tokenisation

Refs #130"
```
