# OutcomeRecord & AuditRecord Metadata

**Issue:** casehubio/ledger#172
**Date:** 2026-07-12
**Status:** Approved
**Depends on:** casehubio/engine#650 (AgentAssignment rationale field) — soft
dependency; metadata is generic, not routing-specific, so this spec can be
implemented independently. engine#650 is needed for the routing-rationale
consumer use case, not for the metadata infrastructure itself.
**Consumer:** casehubio/blocks#30 (LLM and CBR routing strategies producing
rationale that needs audit persistence)

## Problem

Two write-path value types (`OutcomeRecord` for `OutcomeRecorder`, `AuditRecord`
for `LedgerAppender`) have no freeform field for domain-specific audit context.
When a consumer records a routing decision, the rationale, candidate list, and
LLM decision explanation have nowhere to go.

The existing supplement system (`ComplianceSupplement`, `ProvenanceSupplement`)
is for typed cross-cutting concerns with fixed schemas and their own JPA tables.
Domain-specific audit context doesn't fit there.

### Rejected alternatives

Issue #172 considered three options:

1. **`@Nullable String supplementaryData` on OutcomeRecord** — simple but
   naming collides with the existing supplement system.
2. **`@Nullable Map<String, Object> metadata` on OutcomeRecord** — structured
   but requires Jackson in the API module, which is currently pure Java with
   no serialisation dependency.
3. **Use `LedgerEnricherPipeline` to attach rationale at persist time** —
   rejected because enrichers run on the `LedgerEntry` after `buildEntry()`,
   without access to the consumer-provided routing context that motivated the
   write. Metadata must be explicit in the write-path contract, not injected
   via a side-channel that can't see the original `OutcomeRecord`.

This spec chose `@Nullable String metadata` (a variant of option 1 with clearer
naming) applied to both `OutcomeRecord` and `AuditRecord` for consistency across
write paths. Issue #172 originally scoped to OutcomeRecord only; AuditRecord is
included because both are write-path value types and the same gap exists on both.

## Design

Add a `@Nullable String metadata` field to `LedgerEntry`, `OutcomeRecord`, and
`AuditRecord`. Consumer provides JSON; ledger stores verbatim, hashes for tamper
evidence, returns on reads.

### Three data channels on LedgerEntry

| Channel | Purpose | Schema | Owner |
|---------|---------|--------|-------|
| Core fields | Universal, always-relevant | Typed, fixed | Ledger |
| Supplements | Optional cross-cutting concerns (compliance, provenance) | Typed, fixed | Ledger |
| Metadata | Domain-specific audit context | Freeform JSON | Consumer |

### Data model

New field on `LedgerEntry` (`@MappedSuperclass` in api module):

```java
@Column(name = "metadata", columnDefinition = "TEXT")
public String metadata;
```

New migration V1011 (`ledger_entry` table), consistent with the V1002 pattern
for `supplement_json`. V1010 is already taken by `erasure_receipt_entry`.

```sql
-- V1011 — Add metadata column for consumer-provided audit context
ALTER TABLE ledger_entry ADD COLUMN metadata TEXT;
```

`FlywayLocationContractTest` assertion updated from 11 to 12 expected
migrations (V1000–V1011).

**Why `String` not `Map<String, Object>`:** The API module is pure Java with no
Jackson dependency. `String` is consistent with `supplementJson`. Consumer
controls the JSON structure.

**Why on `LedgerEntry` not `PlainLedgerEntry`:** All persistent fields live on
`LedgerEntry` per the two-tier design. Universal availability means domain
subclasses (`WorkItemLedgerEntry`, `MessageLedgerEntry`) inherit it automatically.

### Write-path value types

`OutcomeRecord` — new component after `attestorType`:

```java
public record OutcomeRecord(
        String actorId, UUID subjectId, AttestationVerdict verdict,
        double confidence, String capabilityTag, ActorType actorType,
        String actorRole, Instant occurredAt, String attestorId,
        ActorType attestorType, String metadata) { ... }
```

`AuditRecord` — new component after `causedByEntryId`:

```java
public record AuditRecord(
        UUID subjectId, String actorId, ActorType actorType,
        String actorRole, LedgerEntryType entryType, Instant occurredAt,
        UUID causedByEntryId, String metadata) { ... }
```

**Factory methods** (`of`, `ofGlobal`, `event`) pass `null` for metadata.
**`withMetadata(String)`** on both — throws NPE on null, consistent with all
existing with-methods.

### Save paths

- `OutcomeRecordSaveService.buildEntry()` — `entry.metadata = record.metadata()`
- `DefaultLedgerAppender.append()` — `entry.metadata = record.metadata()`
- Reactive paths are bridges to blocking — no changes needed.

### Tamper evidence — canonicalBytes()

Include `metadata` as a positional field after `causedByEntryId`:

```
subjectId|seqNum|entryType|actorId|actorRole|occurredAt|tenancyId|actorType|causedByEntryId|metadata[|supplementJson][|domainContent]
```

`metadata` is always present in the canonical form — empty string when null.
This makes the field positional (10 base fields instead of 9), eliminating
any ambiguity between metadata and the subsequent optional fields.

The pre-existing conditional append of `supplementJson` and `domainContent`
is safe because those two are structurally distinct — `supplementJson` is
always a JSON object (`{"COMPLIANCE":{...}}`), while `domainContentBytes()`
produces pipe-delimited typed fields. No realistic collision exists between
them. Metadata, being consumer-provided JSON, could easily be confused with
`supplementJson` if both were conditionally appended — hence positional.

Pre-release: no production hashes exist. All hashes can be recomputed.

### Size constraint

Metadata is consumer-provided with no structural bound (unlike `supplementJson`
which is system-generated from typed supplements). A configurable maximum size
prevents unbounded growth in `canonicalBytes()` hashing, `LedgerEntryArchiver`
output, and archive storage.

Default: 65,536 bytes (64 KB). Configurable via
`casehub.ledger.metadata.max-size`. Validated in the write path
(`OutcomeRecordSaveService.buildEntry()` and `DefaultLedgerAppender.append()`)
before setting `entry.metadata`. Throws `IllegalArgumentException` if exceeded.

### Enricher contract

`LedgerEntryEnricher` Javadoc lists fields enrichers MUST NOT overwrite:
`subjectId`, `sequenceNumber`, `tenancyId`, `occurredAt`. Add `metadata` to
this list. Metadata flows from the consumer's record to `entry.metadata` in
`buildEntry()`/`append()` BEFORE enrichment runs. An enricher overwriting
`metadata` would silently discard consumer-provided data.

### Downstream consumers

**Updated:**

- `LedgerEntryArchiver.toJson()` — include `metadata` in archive JSON
- `LedgerEntryArchiveRecord` Javadoc — update "all core fields,
  `supplementJson`" description to include `metadata`
- `LedgerProvSerializer` — add `ledger:metadata` property on PROV entity
- `LedgerEntryResponse` DTO — add `metadata` component
- `LedgerDtoMapper.toResponse()` — map `entry.metadata`

**Not changed:**

- `DecisionRecord` / `LedgerComplianceReportService` — compliance-focused;
  metadata is domain context, not GDPR Art.22 automated-decision data
- `LedgerAttestation` — already has `evidence` for attestor reasoning
- In-memory module — stores whole entry objects
- Testing module — NoOp repos don't touch entry fields

**LedgerEntryResponse includes `metadata` but not `supplementJson`:**
This asymmetry is intentional. Metadata is consumer-owned data that consumers
want to read back via the REST API. Supplements (`ComplianceSupplement`,
`ProvenanceSupplement`) are ledger-internal cross-cutting concerns — they have
dedicated query paths if consumers need them and are not part of the
consumer-facing entry representation.

### GDPR and personal data

Metadata is for domain-specific audit context (routing rationale, candidate
lists, decision explanations). Consumers MUST NOT include personally
identifiable information (PII) in metadata. The ledger's GDPR Art.17 erasure
mechanism (`LedgerErasureService`) severs the token→identity mapping but does
not scan or modify entry field contents. PII embedded in metadata would survive
erasure.

This constraint is documented in `LedgerEntry.metadata` Javadoc and in the
`OutcomeRecord`/`AuditRecord` `withMetadata()` Javadoc. Enforcement is by
contract — consistent with how the ledger handles all consumer-provided string
fields (`actorRole`, `capabilityTag`).

### Consumer usage

```java
// Routing decision with rationale
OutcomeRecord.of(actorId, subjectId, "routing", SOUND, 0.8)
    .withMetadata("{\"rationale\":\"Highest capability score\","
                + "\"candidates\":[\"agent-a\",\"agent-b\"]}");

// Audit event with context
AuditRecord.event(actorId, subjectId)
    .withMetadata("{\"trigger\":\"sla-breach\",\"threshold\":\"PT4H\"}");
```

## Tests

### API module (pure JUnit 5)

- `OutcomeRecordTest` — `withMetadata` sets value, preserves other fields,
  null throws NPE, factory methods default to null
- `AuditRecordTest` — same pattern
- Existing constructor calls updated with `, null` parameter

### Runtime module (QuarkusTest)

- `OutcomeRecorderIT` — metadata on `OutcomeRecord` flows to persisted entry
- `ReactiveOutcomeRecorderIT` — same through reactive bridge
- `LedgerAppenderIT` — metadata on `AuditRecord` flows to persisted entry
  via `DefaultLedgerAppender.append()`
- Canonical bytes test — verify metadata included in hash, verify null
  metadata renders as empty string in positional slot
- `LedgerEntryArchiver` test — metadata appears in archive JSON
- `LedgerProvSerializerTest` — `ledger:metadata` property present on PROV
  entity when metadata is set; omitted when metadata is null
- Size limit tests (both `OutcomeRecordSaveService` and
  `DefaultLedgerAppender` paths):
  - metadata within limit is persisted
  - metadata exceeding configured max throws `IllegalArgumentException`
  - metadata exactly at the limit succeeds

## Not in scope

- **LedgerAttestation metadata** — attestations already have `evidence`. A
  separate concern if needed later.
- **JSON validation** — no runtime validation that metadata is valid JSON.
  Documented as "should be valid JSON" in Javadoc. Same as `supplementJson`.
- **Field-level GDPR erasure** — metadata is constrained to not contain PII
  (see §GDPR above). If a future use case requires PII in metadata,
  field-level erasure must be designed then (casehubio/ledger#178).
