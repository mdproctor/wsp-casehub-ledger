# OutcomeRecord & AuditRecord Metadata Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #172 — feat: OutcomeRecord supplementary data — carry routing rationale into audit trail
**Issue group:** #172

**Goal:** Add a `@Nullable String metadata` field to `LedgerEntry`, `OutcomeRecord`, and `AuditRecord` so consumers can carry freeform JSON audit context (routing rationale, candidate lists, decision explanations) through the write path into the immutable ledger.

**Architecture:** New `metadata` column on `ledger_entry` (V1011 migration). Both write-path value types (`OutcomeRecord`, `AuditRecord`) get a `metadata` component + `withMetadata(String)` wither. Save services copy metadata to the entry. `canonicalBytes()` includes metadata as a positional field (10th base field, always present — empty string when null). Size-limited at 64KB (configurable via `casehub.ledger.metadata.max-size`). REST, archiver, and PROV export updated.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, H2 (tests), Flyway

## Global Constraints

- API module must remain pure Java — no Jackson dependency
- `metadata` is `String` (consumer-provided JSON), not `Map`
- `metadata` is positional in `canonicalBytes()` — always present, empty string when null
- No PII in metadata (contract, not enforced) — GDPR erasure doesn't scan field contents
- Pre-release — modify V1000 is forbidden; use new V1011 migration
- Enrichers MUST NOT overwrite `metadata`
- Size limit: 64KB default, configurable, `IllegalArgumentException` on exceed

---

### Task 1: API module — LedgerEntry metadata field + canonicalBytes()

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/CanonicalBytesTest.java`

**Interfaces:**
- Produces: `LedgerEntry.metadata` field (public String, `@Column(name = "metadata", columnDefinition = "TEXT")`)
- Produces: Updated `canonicalBytes()` with metadata as 10th positional field

- [ ] **Step 1: Write failing tests for metadata in canonicalBytes()**

Add to `CanonicalBytesTest.java`:

```java
@Test
void metadataNull_rendersEmptyStringInPositionalSlot() {
    final TestEntry entry = new TestEntry();
    entry.subjectId = UUID.randomUUID();
    entry.sequenceNumber = 1;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = "actor-1";
    entry.actorRole = "Tester";
    entry.occurredAt = Instant.parse("2026-07-12T10:00:00Z");
    entry.tenancyId = "default";
    entry.actorType = ActorType.AGENT;
    entry.metadata = null;

    final byte[] canonical = entry.canonicalBytes();
    final String canonicalStr = new String(canonical, StandardCharsets.UTF_8);

    // 10 base fields = 9 pipes when no optional appended
    final long pipeCount = canonicalStr.chars().filter(ch -> ch == '|').count();
    assertEquals(9, pipeCount, "Should have 9 pipes (10 base fields) when no supplements");
}

@Test
void metadataPresent_appearsInCanonicalBytes() {
    final TestEntry entry = new TestEntry();
    entry.subjectId = UUID.randomUUID();
    entry.sequenceNumber = 1;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = "actor-1";
    entry.actorRole = "Tester";
    entry.occurredAt = Instant.parse("2026-07-12T10:00:00Z");
    entry.tenancyId = "default";
    entry.actorType = ActorType.AGENT;
    entry.metadata = "{\"rationale\":\"test\"}";

    final byte[] canonical = entry.canonicalBytes();
    final String canonicalStr = new String(canonical, StandardCharsets.UTF_8);

    assertTrue(canonicalStr.contains("{\"rationale\":\"test\"}"),
            "metadata should appear in canonical bytes");
}

@Test
void mutatingMetadata_changesHash() throws Exception {
    final TestEntry entry = new TestEntry();
    entry.subjectId = UUID.randomUUID();
    entry.sequenceNumber = 1;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = "actor-1";
    entry.actorRole = "Tester";
    entry.occurredAt = Instant.parse("2026-07-12T10:00:00Z");
    entry.tenancyId = "default";
    entry.actorType = ActorType.AGENT;
    entry.metadata = "{\"v\":1}";

    final String hashBefore = sha256hex(entry.canonicalBytes());
    entry.metadata = "{\"v\":2}";
    final String hashAfter = sha256hex(entry.canonicalBytes());

    assertNotEquals(hashBefore, hashAfter, "Changing metadata should change the hash");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime -Dtest="CanonicalBytesTest" -DfailIfNoTests=false`
Expected: compilation error — `metadata` field does not exist on `LedgerEntry`

- [ ] **Step 3: Add metadata field to LedgerEntry**

Use `ide_insert_member` to add the field after `actorDid` (line 181):

```java
/**
 * Consumer-provided freeform JSON context for this entry.
 *
 * <p>Carries domain-specific audit data (routing rationale, candidate lists,
 * decision explanations) that is opaque to the ledger. Stored verbatim,
 * included in {@link #canonicalBytes()} for tamper evidence, returned on reads.
 *
 * <p><strong>Contract:</strong> Must be valid JSON. Must NOT contain personally
 * identifiable information (PII) — the GDPR Art.17 erasure mechanism severs
 * the token-identity mapping but does not scan or modify field contents.
 *
 * @see io.casehub.ledger.api.model.OutcomeRecord#withMetadata(String)
 * @see io.casehub.ledger.api.model.AuditRecord#withMetadata(String)
 */
@Column(name = "metadata", columnDefinition = "TEXT")
public String metadata;
```

- [ ] **Step 4: Update canonicalBytes() to include metadata as positional field 10**

Use `ide_replace_member` on `canonicalBytes()`. The updated method adds `metadata` (always present, empty string when null) as the 10th pipe-delimited base field, after `causedByEntryId`:

```java
public final byte[] canonicalBytes() {
    final List<String> parts = new ArrayList<>(10);

    parts.add(subjectId != null ? subjectId.toString() : "");
    parts.add(String.valueOf(sequenceNumber));
    parts.add(entryType != null ? entryType.name() : "");
    parts.add(actorId != null ? actorId : "");
    parts.add(actorRole != null ? actorRole : "");
    parts.add(occurredAt != null
            ? occurredAt.truncatedTo(ChronoUnit.MILLIS).toString()
            : "");
    parts.add(tenancyId != null ? tenancyId : "");
    parts.add(actorType != null ? actorType.name() : "");
    parts.add(causedByEntryId != null ? causedByEntryId.toString() : "");
    parts.add(metadata != null ? metadata : "");

    final StringBuilder canonical = new StringBuilder(String.join("|", parts));

    if (supplementJson != null && !supplementJson.isEmpty()) {
        canonical.append("|").append(supplementJson);
    }

    final byte[] domainBytes = domainContentBytes();
    if (domainBytes.length > 0) {
        canonical.append("|");
        canonical.append(new String(domainBytes, StandardCharsets.UTF_8));
    }

    return canonical.toString().getBytes(StandardCharsets.UTF_8);
}
```

- [ ] **Step 5: Fix existing CanonicalBytesTest assertions**

The `excludesSupplementJson_whenAbsent` test asserts 8 pipes (9 fields). Update to 9 pipes (10 fields):

```java
assertEquals(9, pipeCount, "Should have exactly 9 pipes (10 base fields) when no supplements");
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime -Dtest="CanonicalBytesTest" -DfailIfNoTests=false`
Expected: all pass

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java runtime/src/test/java/io/casehub/ledger/service/CanonicalBytesTest.java
git commit -m "feat(api): add metadata field to LedgerEntry with positional canonicalBytes()

Refs #172"
```

---

### Task 2: API module — OutcomeRecord and AuditRecord metadata component

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/model/OutcomeRecord.java`
- Modify: `api/src/main/java/io/casehub/ledger/api/model/AuditRecord.java`
- Modify: `api/src/test/java/io/casehub/ledger/api/model/OutcomeRecordTest.java`
- Modify: `api/src/test/java/io/casehub/ledger/api/model/AuditRecordTest.java`

**Interfaces:**
- Consumes: Nothing (pure value types)
- Produces: `OutcomeRecord.metadata()` accessor, `OutcomeRecord.withMetadata(String)`
- Produces: `AuditRecord.metadata()` accessor, `AuditRecord.withMetadata(String)`

- [ ] **Step 1: Write failing tests for OutcomeRecord.withMetadata**

Add to `OutcomeRecordTest.java`:

```java
@Test
void withMetadata_setsValue() {
    OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7)
            .withMetadata("{\"key\":\"value\"}");
    assertThat(r.metadata()).isEqualTo("{\"key\":\"value\"}");
}

@Test
void withMetadata_preservesOtherFields() {
    OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7)
            .withActorRole("reviewer")
            .withMetadata("{\"k\":1}");
    assertThat(r.actorId()).isEqualTo("actor");
    assertThat(r.actorRole()).isEqualTo("reviewer");
    assertThat(r.capabilityTag()).isEqualTo("strategy");
    assertThat(r.confidence()).isEqualTo(0.7);
    assertThat(r.metadata()).isEqualTo("{\"k\":1}");
}

@Test
void withMetadata_null_throwsNPE() {
    OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
    assertThatThrownBy(() -> r.withMetadata(null))
            .isInstanceOf(NullPointerException.class);
}

@Test
void factoryMethods_metadataIsNull() {
    OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
    assertThat(r.metadata()).isNull();

    OutcomeRecord g = OutcomeRecord.ofGlobal("actor", subjectId, SOUND, 0.7);
    assertThat(g.metadata()).isNull();
}
```

- [ ] **Step 2: Write failing tests for AuditRecord.withMetadata**

Add to `AuditRecordTest.java`:

```java
@Test
void withMetadata_setsValue() {
    AuditRecord r = AuditRecord.event("actor", subjectId)
            .withMetadata("{\"trigger\":\"sla\"}");
    assertThat(r.metadata()).isEqualTo("{\"trigger\":\"sla\"}");
}

@Test
void withMetadata_preservesOtherFields() {
    AuditRecord r = AuditRecord.event("actor", subjectId)
            .withActorRole("orchestrator")
            .withMetadata("{\"k\":1}");
    assertThat(r.actorId()).isEqualTo("actor");
    assertThat(r.actorRole()).isEqualTo("orchestrator");
    assertThat(r.metadata()).isEqualTo("{\"k\":1}");
}

@Test
void withMetadata_null_throwsNPE() {
    AuditRecord r = AuditRecord.event("actor", subjectId);
    assertThatThrownBy(() -> r.withMetadata(null))
            .isInstanceOf(NullPointerException.class);
}

@Test
void eventFactory_metadataIsNull() {
    AuditRecord r = AuditRecord.event("actor", subjectId);
    assertThat(r.metadata()).isNull();
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="OutcomeRecordTest,AuditRecordTest"`
Expected: compilation error — `metadata` component does not exist

- [ ] **Step 4: Add metadata to OutcomeRecord**

Use `ide_edit_member` on `OutcomeRecord` record declaration. Add `String metadata` as the 11th component. Update the compact constructor (no validation — metadata is nullable). Update factory methods to pass `null`. Add `withMetadata(String)`:

```java
public record OutcomeRecord(
        String actorId,
        UUID subjectId,
        AttestationVerdict verdict,
        double confidence,
        String capabilityTag,
        ActorType actorType,
        String actorRole,
        Instant occurredAt,
        String attestorId,
        ActorType attestorType,
        String metadata
) {
    public OutcomeRecord {
        Objects.requireNonNull(actorId,       "actorId required");
        Objects.requireNonNull(subjectId,     "subjectId required");
        Objects.requireNonNull(verdict,       "verdict required");
        Objects.requireNonNull(capabilityTag, "capabilityTag required — use CapabilityTag.GLOBAL if intentional");
        if (confidence <= 0.0 || confidence > 1.0) {
            throw new IllegalArgumentException(
                    "confidence must be in (0.0, 1.0] — got " + confidence
                            + ". Recommended: 0.1 (tick), 0.7 (game), 1.0 (session).");
        }
        if ((attestorId == null) != (attestorType == null)) {
            throw new IllegalArgumentException(
                    "attestorId and attestorType must be provided together or both left null.");
        }
        if (actorType == null) {
            actorType = ActorType.AGENT;
        }
    }

    public static OutcomeRecord of(final String actorId, final UUID subjectId,
            final String capabilityTag, final AttestationVerdict verdict, final double confidence) {
        return new OutcomeRecord(actorId, subjectId, verdict, confidence,
                capabilityTag, ActorType.AGENT, null, null, null, null, null);
    }

    public static OutcomeRecord ofGlobal(final String actorId, final UUID subjectId,
            final AttestationVerdict verdict, final double confidence) {
        return new OutcomeRecord(actorId, subjectId, verdict, confidence,
                CapabilityTag.GLOBAL, ActorType.AGENT, null, null, null, null, null);
    }

    public OutcomeRecord withActorRole(final String role) {
        Objects.requireNonNull(role, "role");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, role, occurredAt, attestorId, attestorType, metadata);
    }

    public OutcomeRecord withActorType(final ActorType t) {
        Objects.requireNonNull(t, "actorType — use ActorType.AGENT to set the default explicitly");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                t, actorRole, occurredAt, attestorId, attestorType, metadata);
    }

    public OutcomeRecord withOccurredAt(final Instant ts) {
        Objects.requireNonNull(ts, "occurredAt");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, actorRole, ts, attestorId, attestorType, metadata);
    }

    public OutcomeRecord withAttestor(final String id, final ActorType t) {
        Objects.requireNonNull(id, "attestorId");
        Objects.requireNonNull(t,  "attestorType");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, actorRole, occurredAt, id, t, metadata);
    }

    /** @throws NullPointerException if m is null */
    public OutcomeRecord withMetadata(final String m) {
        Objects.requireNonNull(m, "metadata");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, actorRole, occurredAt, attestorId, attestorType, m);
    }
}
```

- [ ] **Step 5: Add metadata to AuditRecord**

Use `ide_edit_member` on `AuditRecord` record declaration. Add `String metadata` as the 8th component. Update factory and with-methods:

```java
public record AuditRecord(
        UUID subjectId,
        String actorId,
        ActorType actorType,
        String actorRole,
        LedgerEntryType entryType,
        Instant occurredAt,
        UUID causedByEntryId,
        String metadata
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

    public static AuditRecord event(final String actorId, final UUID subjectId) {
        return new AuditRecord(subjectId, actorId, ActorType.AGENT,
                null, LedgerEntryType.EVENT, null, null, null);
    }

    public AuditRecord withActorRole(final String role) {
        return new AuditRecord(subjectId, actorId, actorType,
                Objects.requireNonNull(role, "role"), entryType, occurredAt, causedByEntryId, metadata);
    }

    public AuditRecord withCausedBy(final UUID entryId) {
        return new AuditRecord(subjectId, actorId, actorType,
                actorRole, entryType, occurredAt, Objects.requireNonNull(entryId, "entryId"), metadata);
    }

    public AuditRecord withOccurredAt(final Instant ts) {
        return new AuditRecord(subjectId, actorId, actorType,
                actorRole, entryType, Objects.requireNonNull(ts, "ts"), causedByEntryId, metadata);
    }

    /** @throws NullPointerException if m is null */
    public AuditRecord withMetadata(final String m) {
        return new AuditRecord(subjectId, actorId, actorType,
                actorRole, entryType, occurredAt, causedByEntryId, Objects.requireNonNull(m, "metadata"));
    }
}
```

- [ ] **Step 6: Fix existing test constructor calls**

All existing `new OutcomeRecord(...)` calls in `OutcomeRecordTest` need `, null` appended for the metadata parameter. All existing `new AuditRecord(...)` calls in `AuditRecordTest` need `, null` appended. Use `ide_search_text` to find all constructor call sites across the project and fix each one.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="OutcomeRecordTest,AuditRecordTest"`
Expected: all pass

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/model/OutcomeRecord.java api/src/main/java/io/casehub/ledger/api/model/AuditRecord.java api/src/test/java/io/casehub/ledger/api/model/OutcomeRecordTest.java api/src/test/java/io/casehub/ledger/api/model/AuditRecordTest.java
git commit -m "feat(api): add metadata component to OutcomeRecord and AuditRecord

Refs #172"
```

---

### Task 3: Runtime — save paths, config, migration, enricher contract

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/OutcomeRecordSaveService.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/DefaultLedgerAppender.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryEnricher.java`
- Create: `runtime/src/main/resources/db/ledger/migration/V1011__ledger_entry_metadata.sql`
- Modify: `runtime/src/test/java/io/casehub/ledger/FlywayLocationContractTest.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerAppenderIT.java`

**Interfaces:**
- Consumes: `OutcomeRecord.metadata()`, `AuditRecord.metadata()` from Task 2
- Consumes: `LedgerEntry.metadata` field from Task 1
- Produces: `LedgerConfig.MetadataConfig.maxSize()` config interface
- Produces: Size validation in save paths

- [ ] **Step 1: Write failing integration test — OutcomeRecorder metadata flow**

Add to `OutcomeRecorderIT.java`:

```java
@Test
void record_metadataFlowsToPersistedEntry() {
    final String pluginId = "test-agent-" + UUID.randomUUID();
    final UUID subjectId = UUID.randomUUID();

    recorder.record(OutcomeRecord.of(pluginId, subjectId, "routing", SOUND, 0.8)
            .withMetadata("{\"rationale\":\"highest score\",\"candidates\":[\"a\",\"b\"]}"));

    final var entries = ledgerRepo.findBySubjectId(subjectId, DEFAULT_TENANT_ID);
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).metadata)
            .isEqualTo("{\"rationale\":\"highest score\",\"candidates\":[\"a\",\"b\"]}");
}

@Test
void record_nullMetadata_persistsNull() {
    final String pluginId = "test-agent-" + UUID.randomUUID();
    final UUID subjectId = UUID.randomUUID();

    recorder.record(OutcomeRecord.of(pluginId, subjectId, "strategy", SOUND, 0.7));

    final var entries = ledgerRepo.findBySubjectId(subjectId, DEFAULT_TENANT_ID);
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).metadata).isNull();
}
```

- [ ] **Step 2: Write failing integration test — LedgerAppender metadata flow**

Add to `LedgerAppenderIT.java`:

```java
@Test
void append_metadataFlowsToPersistedEntry() {
    final UUID subjectId = UUID.randomUUID();
    final AuditRecord record = AuditRecord.event("actor-meta", subjectId)
            .withMetadata("{\"trigger\":\"sla-breach\"}");
    final UUID id = appender.append(record, DEFAULT_TENANT_ID);

    final var entry = repo.findEntryById(id, DEFAULT_TENANT_ID).orElseThrow();
    assertThat(entry.metadata).isEqualTo("{\"trigger\":\"sla-breach\"}");
}

@Test
void append_nullMetadata_persistsNull() {
    final UUID subjectId = UUID.randomUUID();
    final UUID id = appender.append(AuditRecord.event("actor-1", subjectId), DEFAULT_TENANT_ID);

    final var entry = repo.findEntryById(id, DEFAULT_TENANT_ID).orElseThrow();
    assertThat(entry.metadata).isNull();
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="OutcomeRecorderIT,LedgerAppenderIT"`
Expected: compilation errors in save services (metadata component not passed through)

- [ ] **Step 4: Create V1011 migration**

Write new file:

```sql
-- V1011 — Add metadata column for consumer-provided audit context
ALTER TABLE ledger_entry ADD COLUMN metadata TEXT;
```

File: `runtime/src/main/resources/db/ledger/migration/V1011__ledger_entry_metadata.sql`

- [ ] **Step 5: Add MetadataConfig to LedgerConfig**

Use `ide_insert_member` to add to `LedgerConfig.java` after `erasureReceipt()` (line 145):

```java
MetadataConfig metadata();

interface MetadataConfig {

    @WithDefault("65536")
    int maxSize();
}
```

- [ ] **Step 6: Update OutcomeRecordSaveService.buildEntry() to copy metadata**

Use `ide_replace_member` on `buildEntry` method:

```java
private PlainLedgerEntry buildEntry(final OutcomeRecord record) {
    final PlainLedgerEntry entry = new PlainLedgerEntry();
    entry.actorId    = record.actorId();
    entry.actorRole  = record.actorRole();
    entry.actorType  = record.actorType();
    entry.subjectId  = record.subjectId();
    entry.entryType  = LedgerEntryType.EVENT;
    entry.occurredAt = record.occurredAt();
    entry.metadata   = record.metadata();
    return entry;
}
```

- [ ] **Step 7: Update OutcomeRecordSaveService.save() to validate metadata size**

Use `ide_replace_member` on `save` method. Add `LedgerConfig` injection to the class and size validation before buildEntry:

```java
@Inject
LedgerConfig config;

@Transactional
void save(final OutcomeRecord record, final AttestorDefaults attestor, final String tenancyId) {
    validateMetadataSize(record.metadata());
    final LedgerEntry entry = buildEntry(record);
    ledgerRepo.save(entry, tenancyId);
    java.util.Objects.requireNonNull(entry.id,
            "LedgerEntryRepository.save() must assign entry.id before returning — "
                    + "custom implementations must honour this contract");

    final LedgerAttestation attestation = buildAttestation(record, entry, attestor);
    ledgerRepo.saveAttestation(attestation, tenancyId);
}

private void validateMetadataSize(final String metadata) {
    if (metadata != null && metadata.length() > config.metadata().maxSize()) {
        throw new IllegalArgumentException(
                "metadata exceeds maximum size of " + config.metadata().maxSize()
                        + " bytes — got " + metadata.length());
    }
}
```

- [ ] **Step 8: Update DefaultLedgerAppender.append() to copy metadata and validate size**

Use `ide_replace_member` on `append` method. Add `LedgerConfig` injection:

```java
@Inject
LedgerConfig config;

@Override
public UUID append(final AuditRecord record, final String tenancyId) {
    if (record.metadata() != null && record.metadata().length() > config.metadata().maxSize()) {
        throw new IllegalArgumentException(
                "metadata exceeds maximum size of " + config.metadata().maxSize()
                        + " bytes — got " + record.metadata().length());
    }
    final PlainLedgerEntry entry = new PlainLedgerEntry();
    entry.actorId = record.actorId();
    entry.actorType = record.actorType();
    entry.actorRole = record.actorRole();
    entry.subjectId = record.subjectId();
    entry.entryType = record.entryType();
    entry.occurredAt = record.occurredAt();
    entry.causedByEntryId = record.causedByEntryId();
    entry.metadata = record.metadata();

    final LedgerEntry saved = repo.save(entry, tenancyId);
    return saved.id;
}
```

- [ ] **Step 9: Update LedgerEntryEnricher Javadoc**

Use `ide_edit_member` on the `LedgerEntryEnricher` interface. Add `metadata` to the must-not-overwrite list in both Javadoc blocks:

In the class Javadoc, change:
```
Do NOT overwrite fields stamped by the save pipeline:
{@code subjectId}, {@code sequenceNumber}, {@code tenancyId},
{@code occurredAt}.
```
to:
```
Do NOT overwrite fields stamped by the save pipeline or
provided by the caller: {@code subjectId}, {@code sequenceNumber},
{@code tenancyId}, {@code occurredAt}, {@code metadata}.
```

Same for the `enrich()` method Javadoc.

- [ ] **Step 10: Update FlywayLocationContractTest**

Use `ide_replace_member` on `ledgerMigrations_atCanonicalPath_allExecuteSuccessfully`. Change expected count from 11 to 12:

```java
assertThat(result.migrationsExecuted)
        .as("expected all 12 ledger base migrations (V1000-V1011)")
        .isEqualTo(12);
```

- [ ] **Step 11: Fix all remaining constructor call sites across the project**

Use `ide_search_text` to find all `new OutcomeRecord(` and `new AuditRecord(` constructor calls in test files across all modules (runtime, persistence-memory, testing, rest). Each needs the `, null` metadata parameter appended.

- [ ] **Step 12: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all pass

- [ ] **Step 13: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/OutcomeRecordSaveService.java runtime/src/main/java/io/casehub/ledger/runtime/service/DefaultLedgerAppender.java runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryEnricher.java runtime/src/main/resources/db/ledger/migration/V1011__ledger_entry_metadata.sql runtime/src/test/java/io/casehub/ledger/FlywayLocationContractTest.java runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderIT.java runtime/src/test/java/io/casehub/ledger/service/LedgerAppenderIT.java
git commit -m "feat(runtime): wire metadata through save paths with size limit

V1011 migration, config key casehub.ledger.metadata.max-size (default 64KB),
enricher contract updated.

Refs #172"
```

---

### Task 4: Runtime — size limit tests

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderIT.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerAppenderIT.java`

**Interfaces:**
- Consumes: Size validation from Task 3

- [ ] **Step 1: Write size limit tests for OutcomeRecorder path**

Add to `OutcomeRecorderIT.java`:

```java
@Test
void record_metadataWithinLimit_persists() {
    final String pluginId = "test-agent-" + UUID.randomUUID();
    final UUID subjectId = UUID.randomUUID();
    final String metadata = "{\"k\":\"" + "x".repeat(100) + "\"}";

    recorder.record(OutcomeRecord.of(pluginId, subjectId, "strategy", SOUND, 0.7)
            .withMetadata(metadata));

    final var entries = ledgerRepo.findBySubjectId(subjectId, DEFAULT_TENANT_ID);
    assertThat(entries).hasSize(1);
    assertThat(entries.get(0).metadata).isEqualTo(metadata);
}

@Test
void record_metadataExceedingLimit_throwsIAE() {
    final String pluginId = "test-agent-" + UUID.randomUUID();
    final UUID subjectId = UUID.randomUUID();
    // Default max-size is 65536; create a string just over that
    final String oversized = "x".repeat(65537);

    assertThatThrownBy(() -> recorder.record(
            OutcomeRecord.of(pluginId, subjectId, "strategy", SOUND, 0.7)
                    .withMetadata(oversized)))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("metadata exceeds maximum size");
}
```

Add the assertThatThrownBy import if not present.

- [ ] **Step 2: Write size limit tests for LedgerAppender path**

Add to `LedgerAppenderIT.java`:

```java
@Test
void append_metadataExceedingLimit_throwsIAE() {
    final UUID subjectId = UUID.randomUUID();
    final String oversized = "x".repeat(65537);

    assertThatThrownBy(() -> appender.append(
            AuditRecord.event("actor-1", subjectId).withMetadata(oversized),
            DEFAULT_TENANT_ID))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("metadata exceeds maximum size");
}
```

Add the assertThatThrownBy import.

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="OutcomeRecorderIT,LedgerAppenderIT"`
Expected: all pass

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderIT.java runtime/src/test/java/io/casehub/ledger/service/LedgerAppenderIT.java
git commit -m "test(runtime): add metadata size limit tests for both write paths

Refs #172"
```

---

### Task 5: Downstream consumers — archiver, PROV, REST

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryArchiver.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvSerializer.java`
- Modify: `rest/src/main/java/io/casehub/ledger/rest/dto/LedgerEntryResponse.java`
- Modify: `rest/src/main/java/io/casehub/ledger/rest/dto/LedgerDtoMapper.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerEntryArchiverTest.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerProvSerializerTest.java`

**Interfaces:**
- Consumes: `LedgerEntry.metadata` field from Task 1

- [ ] **Step 1: Write failing test — archiver includes metadata**

Add to `LedgerEntryArchiverTest.java`:

```java
@Test
void toJson_metadataPresent_included() {
    final TestEntry e = entry("agent-1");
    e.metadata = "{\"rationale\":\"test reason\"}";

    final String json = LedgerEntryArchiver.toJson(e, List.of());

    assertThat(json).contains("\"metadata\":\"{\\\"rationale\\\":\\\"test reason\\\"}\"");
}

@Test
void toJson_metadataNull_omitted() {
    final TestEntry e = entry("agent-1");
    e.metadata = null;

    final String json = LedgerEntryArchiver.toJson(e, List.of());

    assertThat(json).doesNotContain("metadata");
}
```

- [ ] **Step 2: Write failing test — PROV serializer includes metadata**

Add to `LedgerProvSerializerTest.java`:

```java
@Test
void metadataPresent_mappedToEntity() throws Exception {
    final UUID subjectId = UUID.randomUUID();
    final TestEntry e = entry(subjectId, 1, "agent-1");
    e.metadata = "{\"candidates\":[\"a\",\"b\"]}";

    final Map<String, Object> doc = parse(
            LedgerProvSerializer.toProvJsonLd(subjectId, List.of(e)));

    final Map<String, Object> entities = asMap(doc.get("entity"));
    final Map<String, Object> entity = asMap(entities.values().iterator().next());
    assertThat(entity).containsKey("ledger:metadata");
    assertThat(entity.get("ledger:metadata")).isEqualTo("{\"candidates\":[\"a\",\"b\"]}");
}

@Test
void metadataNull_omittedFromEntity() throws Exception {
    final UUID subjectId = UUID.randomUUID();
    final TestEntry e = entry(subjectId, 1, "agent-1");
    e.metadata = null;

    final Map<String, Object> doc = parse(
            LedgerProvSerializer.toProvJsonLd(subjectId, List.of(e)));

    final Map<String, Object> entities = asMap(doc.get("entity"));
    final Map<String, Object> entity = asMap(entities.values().iterator().next());
    assertThat(entity).doesNotContainKey("ledger:metadata");
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="LedgerEntryArchiverTest,LedgerProvSerializerTest" -DfailIfNoTests=false`
Expected: tests fail — `metadata` not included in archiver output or PROV entity

- [ ] **Step 4: Update LedgerEntryArchiver.toJson()**

Use `ide_replace_member` on `toJson` method. Add after the `supplementJson` line:

```java
if (entry.metadata != null)
    map.put("metadata", entry.metadata);
```

- [ ] **Step 5: Update LedgerProvSerializer**

Use `ide_replace_member` on `toProvJsonLd`. Add after the provenance supplement block inside the entity building loop:

```java
putIfNotNull(entity, "ledger:metadata", entry.metadata);
```

Place it after the `entry.compliance().ifPresent(...)` block and before `entities.put(entryIri, entity)`.

- [ ] **Step 6: Update LedgerEntryResponse DTO**

Use `ide_edit_member` on `LedgerEntryResponse` record to add `String metadata` after `causedByEntryId`:

```java
public record LedgerEntryResponse(
        UUID id,
        UUID subjectId,
        String tenancyId,
        int sequenceNumber,
        String entryType,
        String actorId,
        String actorType,
        String actorRole,
        Instant occurredAt,
        String digest,
        String traceId,
        UUID causedByEntryId,
        String metadata) {
}
```

- [ ] **Step 7: Update LedgerDtoMapper.toResponse()**

Use `ide_replace_member` on `toResponse(LedgerEntry)` to add `entry.metadata`:

```java
public static LedgerEntryResponse toResponse(final LedgerEntry entry) {
    return new LedgerEntryResponse(
            entry.id,
            entry.subjectId,
            entry.tenancyId,
            entry.sequenceNumber,
            entry.entryType != null ? entry.entryType.name() : null,
            entry.actorId,
            entry.actorType != null ? entry.actorType.name() : null,
            entry.actorRole,
            entry.occurredAt,
            entry.digest,
            entry.traceId,
            entry.causedByEntryId,
            entry.metadata);
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="LedgerEntryArchiverTest,LedgerProvSerializerTest" -DfailIfNoTests=false`
Expected: all pass

- [ ] **Step 9: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 10: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEntryArchiver.java runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvSerializer.java rest/src/main/java/io/casehub/ledger/rest/dto/LedgerEntryResponse.java rest/src/main/java/io/casehub/ledger/rest/dto/LedgerDtoMapper.java runtime/src/test/java/io/casehub/ledger/service/LedgerEntryArchiverTest.java runtime/src/test/java/io/casehub/ledger/service/LedgerProvSerializerTest.java
git commit -m "feat: propagate metadata to archiver, PROV export, and REST DTO

Refs #172"
```

---

### Task 6: CLAUDE.md and ARC42STORIES.MD updates

**Files:**
- Modify: `CLAUDE.md` (project repo)
- Modify: `ARC42STORIES.MD` (project repo, if metadata warrants a mention)

- [ ] **Step 1: Update CLAUDE.md project structure**

Add `metadata` to the `LedgerEntry.java` description line and the `AuditRecord.java` description line in the project structure section. Add `V1011` to the migration list. Add `MetadataConfig` to the `LedgerConfig.java` description.

- [ ] **Step 2: Update ARC42STORIES.MD if applicable**

Check whether the three-channel data model (core fields, supplements, metadata) warrants a mention in the architecture record. If §5 (Building Block View) or §8 (Crosscutting Concepts) covers the data model, add a brief entry about metadata.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md ARC42STORIES.MD
git commit -m "docs: update CLAUDE.md and ARC42STORIES.MD for metadata field

Refs #172"
```
