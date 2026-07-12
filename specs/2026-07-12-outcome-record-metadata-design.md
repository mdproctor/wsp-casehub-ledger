# OutcomeRecord & AuditRecord Metadata

**Issue:** casehubio/ledger#172
**Date:** 2026-07-12
**Status:** Approved

## Problem

Two write-path value types (`OutcomeRecord` for `OutcomeRecorder`, `AuditRecord`
for `LedgerAppender`) have no freeform field for domain-specific audit context.
When a consumer records a routing decision, the rationale, candidate list, and
LLM decision explanation have nowhere to go.

The existing supplement system (`ComplianceSupplement`, `ProvenanceSupplement`)
is for typed cross-cutting concerns with fixed schemas and their own JPA tables.
Domain-specific audit context doesn't fit there.

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

New column in V1000 (`ledger_entry` table):

```sql
metadata TEXT
```

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

Include `metadata` after `causedByEntryId`, before `supplementJson`:

```
base-fields[|metadata][|supplementJson][|domainContent]
```

Appended only when non-null/non-empty. Existing entries without metadata produce
identical canonical bytes.

### Downstream consumers

**Updated:**

- `LedgerEntryArchiver.toJson()` — include `metadata` in archive JSON
- `LedgerProvSerializer` — add `ledger:metadata` property on PROV entity
- `LedgerEntryResponse` DTO — add `metadata` component
- `LedgerDtoMapper.toResponse()` — map `entry.metadata`

**Not changed:**

- `DecisionRecord` / `LedgerComplianceReportService` — compliance-focused;
  metadata is domain context, not GDPR Art.22 data
- `LedgerAttestation` — already has `evidence` for attestor reasoning
- In-memory module — stores whole entry objects
- Testing module — NoOp repos don't touch entry fields

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
- Canonical bytes test — verify metadata included in hash
- `LedgerEntryArchiver` test — metadata appears in archive JSON

## Not in scope

- **LedgerAttestation metadata** — attestations already have `evidence`. A
  separate concern if needed later.
- **JSON validation** — no runtime validation that metadata is valid JSON.
  Documented as "should be valid JSON" in Javadoc. Same as `supplementJson`.
- **Schema migration** — pre-release, modify V1000 in place.
