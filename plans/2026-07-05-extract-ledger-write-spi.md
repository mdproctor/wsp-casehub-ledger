# Extract Ledger Write SPI to ledger-api — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract `LedgerEntryRepository` to `api/spi/`, establish two-tier model hierarchy (`@MappedSuperclass` in api, `@Entity` in runtime), add `LedgerAppender` SPI for api-tier event recording, and clean up dead api model duplicates.

**Architecture:** Two-tier split following the existing `LedgerAttestation` pattern. Api types are `@MappedSuperclass` with `@Column` annotations (persistence-agnostic contract). Runtime JPA types extend them and add `@Entity`, `@NamedQuery`, `@EntityListeners`. Repository SPI moves to api tier. New `LedgerAppender` provides value-type write path for consumers that don't create entity subclasses.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (Hibernate 6), H2 + PostgreSQL, Mutiny (reactive)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
- No deployed instances — breaking changes are free
- All commits reference `Refs #168`
- Flyway migrations are rewritten in-place (no incremental scripts)
- Use IntelliJ MCP for all renames, moves, and reference searches — never bash grep
- Follow `ledger-subclass-repo-readonly` protocol: saves go through `LedgerEntryRepository`
- Follow `ledger-entry-named-query` protocol: all JPQL via `@NamedQuery`
- `@Entity(name = "LedgerEntry")` on `JpaLedgerEntry` preserves JPQL entity name

---

### Task 1: Supplement hierarchy — unify parallel copies into two-tier pattern

The supplement classes are parallel copies with zero inheritance. This must be
fixed first because the LedgerEntry two-tier split (Task 2) depends on
`attach()` accepting api supplement types.

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/model/supplement/LedgerSupplement.java`
- Modify: `api/src/main/java/io/casehub/ledger/api/model/supplement/ComplianceSupplement.java`
- Modify: `api/src/main/java/io/casehub/ledger/api/model/supplement/ProvenanceSupplement.java`
- Keep: `api/src/main/java/io/casehub/ledger/api/model/supplement/LedgerSupplementSerializer.java` (becomes canonical)
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/JpaComplianceSupplement.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/JpaProvenanceSupplement.java`
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplement.java`
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ComplianceSupplement.java`
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ProvenanceSupplement.java`
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplementSerializer.java`
- Modify: `runtime/src/main/resources/db/ledger/migration/V1002__ledger_supplement.sql`
- Modify: all runtime files importing `runtime.model.supplement.*`

**Interfaces:**
- Produces: `api.model.supplement.LedgerSupplement` as `@MappedSuperclass` — the type used by `LedgerEntry.attach()` and type-safe accessors
- Produces: `api.model.supplement.ComplianceSupplement` / `ProvenanceSupplement` as `@MappedSuperclass`
- Produces: `runtime.model.supplement.JpaComplianceSupplement` / `JpaProvenanceSupplement` as `@Entity`

**Steps:**

- [ ] **Step 1: Rewrite api `LedgerSupplement` as `@MappedSuperclass`**

  Add `@MappedSuperclass`, `@Id` on `id`, `@Column` on `supplementType`. Mark `ledgerEntry` as `@Transient` (in-memory back-reference for api-tier use). Remove any plain-field-only style. Keep the `id` and `supplementType` fields.

- [ ] **Step 2: Rewrite api `ComplianceSupplement` as `@MappedSuperclass`**

  Add `@MappedSuperclass`. Add `@Column` on all compliance fields (`planRef`, `rationale`, `evidence`, `detail`, `decisionContext`, `algorithmRef`, `confidenceScore`, `contestationUri`, `humanOverrideAvailable`). Extends api `LedgerSupplement`.

- [ ] **Step 3: Rewrite api `ProvenanceSupplement` as `@MappedSuperclass`**

  Same pattern. Add `@Column` on `sourceEntityId`, `sourceEntityType`, `sourceEntitySystem`, `agentConfigHash`. Extends api `LedgerSupplement`.

- [ ] **Step 4: Create `JpaComplianceSupplement`**

  In `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/`. Extends `api.model.supplement.ComplianceSupplement`. Adds `@Entity`, `@Table(name = "ledger_supplement_compliance")`. Adds `@ManyToOne @JoinColumn(name = "ledger_entry_id")` for the JPA relationship. Adds `@PrePersist` for id assignment (`if (id == null) id = UUID.randomUUID()`).

- [ ] **Step 5: Create `JpaProvenanceSupplement`**

  Same pattern as Step 4. `@Table(name = "ledger_supplement_provenance")`.

- [ ] **Step 6: Delete runtime supplement classes**

  Delete `runtime.model.supplement.LedgerSupplement`, `ComplianceSupplement`, `ProvenanceSupplement`, `LedgerSupplementSerializer`. The api versions are now canonical.

- [ ] **Step 7: Update V1002 migration**

  Rewrite `V1002__ledger_supplement.sql`: drop the JOINED `ledger_supplement` base table. Create two self-contained tables: `ledger_supplement_compliance` (carries base columns `id`, `ledger_entry_id`, `supplement_type` plus all compliance-specific columns) and `ledger_supplement_provenance` (same base columns plus provenance-specific columns).

- [ ] **Step 8: Update all runtime imports**

  Use IntelliJ `ide_find_references` to find every import of `runtime.model.supplement.ComplianceSupplement` and `runtime.model.supplement.ProvenanceSupplement`. Update to use `api.model.supplement.*` for type references and `runtime.model.supplement.Jpa*` for entity instantiation. Key files: `LedgerComplianceReportService`, `ProvenanceCaptureEnricher`, `LedgerProvSerializer`, `LedgerProvExportService`, `DecisionRecord`. Also update `LedgerSupplementSerializer` imports — delete the runtime copy, use api's.

- [ ] **Step 9: Update test files**

  Update supplement imports in all test files: `LedgerSupplementIT`, `LedgerSupplementSerializerTest`, `CanonicalBytesTest`, `PlainEntityTest`, `LedgerProvExportServiceIT`, `LedgerProvSerializerTest`, `LedgerPrivacyWiringIT`, `LedgerComplianceReportServiceIT`.

- [ ] **Step 10: Build and run all tests**

  Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test`
  Expected: all tests pass

- [ ] **Step 11: Commit**

  ```
  feat(api): unify supplement hierarchy into two-tier pattern

  Api supplements become @MappedSuperclass with @Column. Runtime JPA
  supplements extend them as @Entity. JOINED inheritance eliminated —
  each supplement type is an independent entity with self-contained table.
  Runtime LedgerSupplementSerializer deleted (api copy is canonical).

  Refs #168
  ```

---

### Task 2: LedgerEntry two-tier split — `@MappedSuperclass` in api, `JpaLedgerEntry` in runtime

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/jpa/JpaLedgerEntry.java`
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/PlainLedgerEntry.java` (reparent)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/KeyRotationEntry.java` (reparent)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ErasureReceiptLedgerEntry.java` (reparent)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorIdentityBindingEntry.java` (reparent)
- Modify: `deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java` (DotName update)
- Modify: ~23 runtime service files (import updates)
- Modify: ~33 test files (import updates + test subclass reparenting)

**Interfaces:**
- Consumes: Task 1 supplements (api supplement types for `attach()`)
- Produces: `api.model.LedgerEntry` as `@MappedSuperclass` — the SPI type for Task 3
- Produces: `runtime.model.jpa.JpaLedgerEntry` — the entity class for subclasses

**Steps:**

- [ ] **Step 1: Rewrite api `LedgerEntry` as `@MappedSuperclass`**

  Add `@MappedSuperclass`. Add `@Id` on `id`, `@Column` on all persistent fields (copy exact column names and nullability from runtime version). Include agent signing fields (`agentSignature`, `agentPublicKey`, `agentKeyRef`), DID binding (`actorDid`), and `digest`. Keep `supplements` as `@Transient` (api-tier helper methods). Keep `supplementJson` with `@Column`. Keep `canonicalBytes()` (final), `domainContentBytes()` (protected). Keep `attach()`, `refreshSupplementJson()`, `compliance()`, `provenance()`. Add eager `id = UUID.randomUUID()` in a no-arg constructor or field initializer. Do NOT add: `@Entity`, `@Inheritance`, `@Table`, `@NamedQuery`, `@EntityListeners`, `@PrePersist`, `pendingIdentityStatus`.

- [ ] **Step 2: Create `JpaLedgerEntry`**

  In `runtime/src/main/java/io/casehub/ledger/runtime/model/jpa/`. Extends `api.model.LedgerEntry`. Adds:
  - `@Entity(name = "LedgerEntry")` — preserves JPQL entity name
  - `@Inheritance(strategy = InheritanceType.JOINED)`
  - `@DiscriminatorColumn(name = "dtype", discriminatorType = DiscriminatorType.STRING)`
  - `@Table(name = "ledger_entry")`
  - `@EntityListeners({LedgerTraceListener.class, LedgerIdentityEnforcementListener.class})`
  - All 4 `@NamedQuery` declarations (copied from old runtime LedgerEntry)
  - `@Transient public IdentityBindingStatus pendingIdentityStatus`
  - `@PrePersist void prePersist()` — defensive `if (id == null)` + `if (occurredAt == null)`
  - Per-type supplement JPA fields: `@OneToMany List<JpaComplianceSupplement> complianceSupplements`, `@OneToMany List<JpaProvenanceSupplement> provenanceSupplements`
  - Override `attach()` to synchronise transient list + JPA lists
  - `@PostLoad` to populate transient `supplements` list from JPA lists

- [ ] **Step 3: Delete runtime `LedgerEntry.java`**

  Delete `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`.

- [ ] **Step 4: Reparent internal subclasses**

  Change `extends LedgerEntry` → `extends JpaLedgerEntry` and update imports in:
  - `PlainLedgerEntry`
  - `KeyRotationEntry`
  - `ErasureReceiptLedgerEntry`
  - `ActorIdentityBindingEntry`

- [ ] **Step 5: Update `LedgerProcessor` DotName**

  In `deployment/LedgerProcessor.java`, change:
  ```java
  static final DotName LEDGER_ENTRY = DotName.createSimple(
      "io.casehub.ledger.runtime.model.LedgerEntry");
  ```
  To:
  ```java
  static final DotName LEDGER_ENTRY = DotName.createSimple(
      "io.casehub.ledger.api.model.LedgerEntry");
  ```

- [ ] **Step 6: Update runtime service imports**

  Use IntelliJ `ide_find_references` for `runtime.model.LedgerEntry` across runtime `src/main/java`. Update each import to `api.model.LedgerEntry` OR `runtime.model.jpa.JpaLedgerEntry` depending on usage:
  - Services that work with the DATA MODEL (fields, canonicalBytes): → `api.model.LedgerEntry`
  - Services that need JPA entity specifics (EntityManager, persist): → `runtime.model.jpa.JpaLedgerEntry`
  - Key files: `LedgerMerkleTree`, `AgentCryptographicVerifier`, `AgentEntrySigner`, `LedgerEntryArchiver`, `LedgerRetentionJob`, `LedgerVerificationService`, `LedgerComplianceReportService`, `LedgerProvSerializer`, `LedgerProvExportService`, `AgentSignatureVerificationService`, `ReactiveAgentSignatureVerificationService`, `RetentionEligibilityChecker`, `LedgerEntryEnricher`, `LedgerEnricherPipeline`, `LedgerTraceListener`, `TraceIdEnricher`, `OutcomeRecordSaveService`, `TrustScoreComputer`, `TrustScoreCalculator`, `ComputedTrustScoreSource`, `PerActorTrustComputer`, `IncrementalTrustUpdateObserver`, `TrustScoreJob`

- [ ] **Step 7: Update test files**

  Update imports and reparent test subclasses (`TestLedgerEntry`, `ConcreteEntry`, etc.) across all 33 test files. Test subclasses that are `@Entity` extend `JpaLedgerEntry`. Non-entity test helpers extend `api.model.LedgerEntry`.

- [ ] **Step 8: Update persistence-memory module**

  `InMemoryLedgerEntryRepository` and related classes: update `runtime.model.LedgerEntry` imports to `api.model.LedgerEntry`. These work with the data model, not JPA specifics.

- [ ] **Step 9: Build and run all tests**

  Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test`
  Expected: all tests pass

- [ ] **Step 10: Commit**

  ```
  feat(api): two-tier LedgerEntry — @MappedSuperclass in api, JpaLedgerEntry in runtime

  api.model.LedgerEntry becomes @MappedSuperclass with all persistent
  fields, canonicalBytes(), supplement helpers. JpaLedgerEntry extends it
  with @Entity, @Inheritance(JOINED), @NamedQuery, @EntityListeners.
  All subclasses reparented. LedgerProcessor DotName updated.

  Refs #168
  ```

---

### Task 3: Move `LedgerEntryRepository` and `ReactiveLedgerEntryRepository` to `api/spi/`

**Files:**
- Move: `runtime/.../repository/LedgerEntryRepository.java` → `api/.../spi/LedgerEntryRepository.java`
- Move: `runtime/.../repository/ReactiveLedgerEntryRepository.java` → `api/.../spi/ReactiveLedgerEntryRepository.java`
- Modify: `runtime/.../repository/NoOpLedgerEntryRepository.java` (import update)
- Modify: `runtime/.../repository/jpa/JpaLedgerEntryRepository.java` (import update)
- Modify: `persistence-memory/.../InMemoryLedgerEntryRepository.java` (import update)
- Modify: `persistence-memory/.../InMemoryReactiveLedgerEntryRepository.java` (import update)
- Modify: all runtime services injecting `LedgerEntryRepository` (import update)
- Modify: ~51 test files (import update)

**Interfaces:**
- Consumes: Task 2 api.model.LedgerEntry (for method signatures)
- Produces: `api.spi.LedgerEntryRepository` — consumer-facing write/read SPI
- Produces: `api.spi.ReactiveLedgerEntryRepository` — reactive counterpart

**Steps:**

- [ ] **Step 1: Move `LedgerEntryRepository` interface to api**

  Use IntelliJ `ide_move_file` to move `runtime/src/main/java/io/casehub/ledger/runtime/repository/LedgerEntryRepository.java` to `api/src/main/java/io/casehub/ledger/api/spi/`. Update the package declaration. Update imports to reference `api.model.LedgerEntry` and `api.model.LedgerAttestation` (should already be correct after Task 2, but verify).

- [ ] **Step 2: Move `ReactiveLedgerEntryRepository` to api**

  Same as Step 1. Move to `api/src/main/java/io/casehub/ledger/api/spi/`.

- [ ] **Step 3: Update all implementations and consumers**

  Use IntelliJ `ide_find_references` for the old import path `runtime.repository.LedgerEntryRepository`. Update every file:
  - Runtime implementations: `NoOpLedgerEntryRepository`, `JpaLedgerEntryRepository`, `CrossTenantLedgerEntryRepository` (if it references the base interface)
  - Persistence-memory: `InMemoryLedgerEntryRepository`, `InMemoryReactiveLedgerEntryRepository`, `InMemoryCrossTenantLedgerEntryRepository`, `InMemoryCrossTenantReactiveLedgerEntryRepository`
  - Runtime services: `OutcomeRecordSaveService`, `LedgerRetentionJob`, `LedgerHealthJob`, `KeyRotationService`, `ReactiveKeyRotationService`, all enrichers, trust services
  - All ~51 test files

- [ ] **Step 4: Build and run all tests**

  Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test`
  Expected: all tests pass

- [ ] **Step 5: Commit**

  ```
  feat(api): move LedgerEntryRepository and ReactiveLedgerEntryRepository to api/spi

  Consumer-facing repository SPI now in api tier. Method signatures
  reference api.model types. All implementations and consumers updated.

  Refs #168
  ```

---

### Task 4: Add `LedgerAppender`, `ReactiveLedgerAppender`, and `AuditRecord`

**Files:**
- Create: `api/src/main/java/io/casehub/ledger/api/model/AuditRecord.java`
- Create: `api/src/main/java/io/casehub/ledger/api/spi/LedgerAppender.java`
- Create: `api/src/main/java/io/casehub/ledger/api/spi/ReactiveLedgerAppender.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/DefaultLedgerAppender.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/DefaultReactiveLedgerAppender.java`
- Test: `api/src/test/java/io/casehub/ledger/api/model/AuditRecordTest.java`
- Test: `runtime/src/test/java/io/casehub/ledger/service/LedgerAppenderIT.java`

**Interfaces:**
- Consumes: Task 3 `api.spi.LedgerEntryRepository`
- Produces: `api.spi.LedgerAppender` — value-type write SPI for api-tier consumers
- Produces: `api.model.AuditRecord` — input value type

**Steps:**

- [ ] **Step 1: Write `AuditRecord` test**

  Test canonical constructor validation (ATTESTATION rejected), factory method, with-methods.

  ```java
  // api/src/test/java/io/casehub/ledger/api/model/AuditRecordTest.java
  @Test void rejectsAttestationType() {
      assertThrows(IllegalArgumentException.class,
          () -> new AuditRecord(UUID.randomUUID(), "actor", ActorType.AGENT,
              null, LedgerEntryType.ATTESTATION, null, null));
  }

  @Test void eventFactory_setsDefaults() {
      var r = AuditRecord.event("actor-1", UUID.randomUUID());
      assertEquals(LedgerEntryType.EVENT, r.entryType());
      assertEquals(ActorType.AGENT, r.actorType());
      assertNull(r.actorRole());
  }

  @Test void withMethods_returnNewInstance() {
      var r = AuditRecord.event("a", UUID.randomUUID());
      var r2 = r.withActorRole("reviewer");
      assertNotSame(r, r2);
      assertEquals("reviewer", r2.actorRole());
      assertNull(r.actorRole());
  }
  ```

- [ ] **Step 2: Run test to verify it fails**

  Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AuditRecordTest`
  Expected: FAIL — `AuditRecord` does not exist

- [ ] **Step 3: Implement `AuditRecord`**

  ```java
  package io.casehub.ledger.api.model;

  public record AuditRecord(
      UUID subjectId, String actorId, ActorType actorType,
      String actorRole, LedgerEntryType entryType,
      Instant occurredAt, UUID causedByEntryId
  ) {
      public AuditRecord {
          Objects.requireNonNull(actorId, "actorId required");
          Objects.requireNonNull(subjectId, "subjectId required");
          if (entryType == LedgerEntryType.ATTESTATION) {
              throw new IllegalArgumentException(
                  "AuditRecord does not support ATTESTATION — use OutcomeRecorder");
          }
          if (actorType == null) actorType = ActorType.AGENT;
          if (entryType == null) entryType = LedgerEntryType.EVENT;
      }
      public static AuditRecord event(String actorId, UUID subjectId) {
          return new AuditRecord(subjectId, actorId, ActorType.AGENT,
              null, LedgerEntryType.EVENT, null, null);
      }
      public AuditRecord withActorRole(String role) {
          return new AuditRecord(subjectId, actorId, actorType,
              Objects.requireNonNull(role), entryType, occurredAt, causedByEntryId);
      }
      public AuditRecord withCausedBy(UUID entryId) {
          return new AuditRecord(subjectId, actorId, actorType,
              actorRole, entryType, occurredAt, Objects.requireNonNull(entryId));
      }
      public AuditRecord withOccurredAt(Instant ts) {
          return new AuditRecord(subjectId, actorId, actorType,
              actorRole, entryType, Objects.requireNonNull(ts), causedByEntryId);
      }
  }
  ```

- [ ] **Step 4: Run test to verify it passes**

  Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=AuditRecordTest`
  Expected: PASS

- [ ] **Step 5: Create `LedgerAppender` and `ReactiveLedgerAppender` interfaces**

  ```java
  // api/src/main/java/io/casehub/ledger/api/spi/LedgerAppender.java
  package io.casehub.ledger.api.spi;
  public interface LedgerAppender {
      UUID append(AuditRecord record, String tenancyId);
  }

  // api/src/main/java/io/casehub/ledger/api/spi/ReactiveLedgerAppender.java
  package io.casehub.ledger.api.spi;
  public interface ReactiveLedgerAppender {
      Uni<UUID> append(AuditRecord record, String tenancyId);
  }
  ```

- [ ] **Step 6: Write `LedgerAppenderIT` test**

  ```java
  // runtime/src/test/java/io/casehub/ledger/service/LedgerAppenderIT.java
  @QuarkusTest
  class LedgerAppenderIT {
      @Inject LedgerAppender appender;
      @Inject LedgerEntryRepository repo;

      @Test void append_persistsPlainLedgerEntry() {
          var subjectId = UUID.randomUUID();
          var record = AuditRecord.event("actor-1", subjectId)
              .withActorRole("orchestrator");
          UUID id = appender.append(record, DEFAULT_TENANT_ID);
          assertNotNull(id);
          var entry = repo.findEntryById(id, DEFAULT_TENANT_ID);
          assertTrue(entry.isPresent());
          assertEquals("actor-1", entry.get().actorId);
          assertEquals("orchestrator", entry.get().actorRole);
          assertEquals(LedgerEntryType.EVENT, entry.get().entryType);
      }

      @Test void append_assignsSequenceNumber() {
          var subjectId = UUID.randomUUID();
          appender.append(AuditRecord.event("a", subjectId), DEFAULT_TENANT_ID);
          UUID id2 = appender.append(AuditRecord.event("a", subjectId), DEFAULT_TENANT_ID);
          var entry = repo.findEntryById(id2, DEFAULT_TENANT_ID).orElseThrow();
          assertEquals(2, entry.sequenceNumber);
      }
  }
  ```

- [ ] **Step 7: Implement `DefaultLedgerAppender` and `DefaultReactiveLedgerAppender`**

  ```java
  // runtime/src/main/java/io/casehub/ledger/runtime/service/DefaultLedgerAppender.java
  @DefaultBean @ApplicationScoped
  public class DefaultLedgerAppender implements LedgerAppender {
      @Inject LedgerEntryRepository repo;
      @Override public UUID append(AuditRecord record, String tenancyId) {
          PlainLedgerEntry entry = new PlainLedgerEntry();
          entry.actorId = record.actorId();
          entry.actorType = record.actorType();
          entry.actorRole = record.actorRole();
          entry.subjectId = record.subjectId();
          entry.entryType = record.entryType();
          entry.occurredAt = record.occurredAt();
          entry.causedByEntryId = record.causedByEntryId();
          LedgerEntry saved = repo.save(entry, tenancyId);
          return saved.id;
      }
  }

  // DefaultReactiveLedgerAppender wraps blocking on worker pool
  @DefaultBean @ApplicationScoped
  public class DefaultReactiveLedgerAppender implements ReactiveLedgerAppender {
      @Inject LedgerAppender delegate;
      @Override public Uni<UUID> append(AuditRecord record, String tenancyId) {
          return Uni.createFrom().item(() -> delegate.append(record, tenancyId))
              .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
      }
  }
  ```

- [ ] **Step 8: Run tests**

  Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime`
  Expected: all tests pass including new LedgerAppenderIT

- [ ] **Step 9: Commit**

  ```
  feat(api): add LedgerAppender SPI with AuditRecord value type

  New api-tier write path for pure event recording. AuditRecord accepts
  actorId, subjectId, entryType (validates: no ATTESTATION). DefaultLedgerAppender
  creates PlainLedgerEntry and delegates to LedgerEntryRepository.save().

  Refs #168
  ```

---

### Task 5: Dead api model cleanup

**Files:**
- Delete: `api/src/main/java/io/casehub/ledger/api/model/ActorTrustScore.java`
- Delete: `api/src/main/java/io/casehub/ledger/api/model/LedgerMerkleFrontier.java`
- Delete: `api/src/main/java/io/casehub/ledger/api/model/ActorIdentity.java`
- Delete: `api/src/main/java/io/casehub/ledger/api/model/LedgerEntryArchiveRecord.java`
- Create: `api/src/main/java/io/casehub/ledger/api/model/ScoreType.java` (extracted enum)
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/ActorTrustScore.java` (ScoreType import)

**Interfaces:**
- Produces: `api.model.ScoreType` standalone enum (used by engine's `WorkerDecisionEventCaptureTest`)

**Steps:**

- [ ] **Step 1: Extract `ScoreType` enum**

  Create `api/src/main/java/io/casehub/ledger/api/model/ScoreType.java` with values `GLOBAL`, `CAPABILITY`, `DIMENSION`, `CAPABILITY_DIMENSION`. Update `runtime.model.ActorTrustScore` to import from `api.model.ScoreType` instead of its inner enum. Update any references to `ActorTrustScore.ScoreType` across the codebase (engine test has 1 reference).

- [ ] **Step 2: Delete dead api model classes**

  Delete `api.model.ActorTrustScore`, `api.model.LedgerMerkleFrontier`, `api.model.ActorIdentity`, `api.model.LedgerEntryArchiveRecord`. Use IntelliJ `ide_find_references` to verify zero remaining references before each deletion.

- [ ] **Step 3: Build and run all tests**

  Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test`
  Expected: all tests pass

- [ ] **Step 4: Commit**

  ```
  chore(api): remove dead model duplicates, extract ScoreType enum

  Delete unused api copies of ActorTrustScore, LedgerMerkleFrontier,
  ActorIdentity, LedgerEntryArchiveRecord. ScoreType enum extracted
  to api.model for cross-module use.

  Refs #168
  ```

---

### Task 6: Documentation updates

**Files:**
- Modify: `ARC42STORIES.MD` — §5 Key Files, §5.3.1 save pipeline, Layer Taxonomy L1
- Modify: `CLAUDE.md` — project structure, package references

**Interfaces:**
- Consumes: all prior tasks (final class locations)

**Steps:**

- [ ] **Step 1: Update ARC42STORIES.MD**

  Update §5 Key Files: `LedgerEntry.java` location (now api module), `LedgerEntryRepository.java` (now api/spi). Update §5.3.1 save pipeline description with two-tier hierarchy. Update Layer Taxonomy L1 context with new repository location.

- [ ] **Step 2: Update CLAUDE.md**

  Update project structure tree to reflect: api model hierarchy, JpaLedgerEntry location, repository SPI in api/spi, new LedgerAppender/AuditRecord, supplement two-tier pattern.

- [ ] **Step 3: Commit**

  ```
  docs: update ARC42STORIES.MD and CLAUDE.md for #168 two-tier model

  Refs #168
  ```

---

## Task Dependency Graph

```
Task 1 (supplements) → Task 2 (LedgerEntry split) → Task 3 (repo move) → Task 4 (LedgerAppender)
                                                                        → Task 5 (dead model cleanup)
                                                                        → Task 6 (docs)
```

Tasks 4, 5, and 6 can run in parallel after Task 3 completes.
