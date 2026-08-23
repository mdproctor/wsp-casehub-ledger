# Qhorus Query APIs — Streaming, Cursor, and Aggregate Queries

**Issues:** #202 (streaming/cursor), #201 (aggregate queries)
**Branch:** issue-202-qhorus-query-apis
**Date:** 2026-08-23

## Summary

Add six new methods to `LedgerEntryRepository` and one new model record
to `api/model/` to support qhorus formal verification (E7) and compliance
evidence export (E5). No schema changes. No new entities.

## SPI Additions — LedgerEntryRepository

### Streaming queries (#202)

```java
Stream<LedgerEntry> streamBySubjectId(UUID subjectId, String tenancyId);

Stream<LedgerEntry> streamByActorId(String actorId, Instant from, Instant to, String tenancyId);
```

**Ordering:** `streamBySubjectId` returns entries in sequence order
ascending. `streamByActorId` returns entries ordered by `occurredAt`
ascending. Time range `[from, to]` is inclusive on both bounds —
consistent with existing `findByActorId`.

**Resource lifecycle contract:** The returned `Stream` must be consumed
within the caller's transaction scope and closed explicitly via
try-with-resources. Failing to close leaks a database cursor in the JPA
implementation. The in-memory implementation has no resource concern.

**JPA implementation:** `em.createNamedQuery(...).getResultStream()`.
Each query is declared as a `@NamedQuery` on `JpaLedgerEntry` per
protocol PP-20260618-51c673.

**In-memory implementation:** `allEntries().stream().filter(...)` — no
resource lifecycle, no close required.

### Cursor-based pagination (#202)

```java
List<LedgerEntry> findBySubjectIdPaged(UUID subjectId, int afterSequence, int limit, String tenancyId);
```

Returns entries with `sequenceNumber > afterSequence`, ordered ascending,
capped at `limit`. First page: `afterSequence = 0`. Subsequent pages:
pass the last entry's `sequenceNumber`.

**JPA implementation:** `@NamedQuery` with `setFirstResult`/`setMaxResults`
or a WHERE clause on `sequenceNumber > :after` with `setMaxResults`.
The WHERE approach is preferred — it's stable under concurrent inserts
and doesn't require offset tracking.

**In-memory implementation:** filter + sort + subList.

### Aggregate queries (#201)

```java
Map<AttestationVerdict, Long> countByActorAndVerdict(String actorId, Instant from, Instant to, String tenancyId);

Map<AttestationVerdict, Long> countBySubjectAndVerdict(UUID subjectId, Instant from, Instant to, String tenancyId);

AttestationSummary summariseAttestationsByActor(String actorId, Instant from, Instant to, String tenancyId);
```

All three JOIN `LedgerEntry` with `LedgerAttestation` and aggregate by
`AttestationVerdict`. The time range `[from, to]` is inclusive on both
bounds, filtering on `LedgerEntry.occurredAt`.

**Empty-result contract:** When the actor/subject has entries but no
attestations in the time range, verdict count maps are empty (not null)
and `AttestationSummary` returns `totalAttestations = 0` with confidence
fields set to `0.0`.

**No `outcome` field on `LedgerEntry`.** The ledger separates decisions
(entries) from assessments (attestations). Aggregating by verdict on the
attestation is the correct query shape. Unit tests demonstrate qhorus
derivation patterns to prove the model is sufficient.

## New Model — AttestationSummary

```java
// api/src/main/java/io/casehub/ledger/api/model/AttestationSummary.java
public record AttestationSummary(
    Map<AttestationVerdict, Long> verdictCounts,
    long totalAttestations,
    double meanConfidence,
    double minConfidence,
    double maxConfidence
) {}
```

Immutable record in `api/model/`. Captures both verdict distribution and
confidence statistics in a single query result.

## Implementations Required

| Layer | What |
|-------|------|
| `api/` | 6 new methods on `LedgerEntryRepository`, `AttestationSummary` record |
| `runtime/` JPA | `@NamedQuery` declarations on `JpaLedgerEntry`, implementations in `JpaLedgerEntryRepository` |
| `persistence-memory/` | Implementations in `InMemoryLedgerEntryRepository` |
| `runtime/` NoOp | Default returns in `NoOpLedgerEntryRepository` (empty streams, empty maps, empty summary) |
| `testing/` | Default returns in `NoOpLedgerEntryRepository` |

## Qhorus Derivation Tests

Unit tests in the `runtime/` test suite demonstrating how qhorus derives
outcomes without an `outcome` field on `LedgerEntry`:

1. **Fulfillment rate** — write N decisions via `OutcomeRecorder` with
   mixed ENDORSED/CHALLENGED verdicts, then `countByActorAndVerdict` →
   `endorsed / (endorsed + challenged)` = fulfillment rate

2. **Routing success rate** — same pattern scoped to a capability tag,
   filtered by time range

3. **Per-channel quality** — `countBySubjectAndVerdict` per subject (channel),
   compare verdict distributions across channels

4. **Confidence distribution** — `summariseAttestationsByActor` returns
   mean/min/max confidence, demonstrating quality-of-assessment metrics

5. **Streaming traversal** — write entries via `OutcomeRecorder`, stream
   back via `streamBySubjectId`, verify correct order and completeness.
   Verify stream is closeable without side effects.

6. **Cursor pagination** — write N entries, page through with
   `findBySubjectIdPaged` using `afterSequence` from each page's last
   entry, verify all entries are returned exactly once.

7. **Empty results** — actor with entries but no attestations returns
   empty verdict map. Actor with no entries at all returns empty map.
   `AttestationSummary` with zero attestations returns `totalAttestations = 0`.

## Protocol Compliance

- **PP-20260618-51c673:** All new queries use `@NamedQuery` — no inline
  `em.createQuery()`.
- **PP-20260616-05dc6a:** No new per-subject tables. All queries use
  existing `ledger_entry` + `ledger_attestation` with `tenancyId` scoping.

## Reactive Parity

Reactive variants (`Uni<Stream<...>>`, `Uni<Map<...>>`) are deferred.
The qhorus consumers (E5 compliance export, E7 formal verification) are
blocking-path batch operations. Reactive methods can be added later as
the same SPI extension pattern used for existing reactive pairs.

## Not in Scope

- REST/GraphQL exposure of these queries (separate issue)
- `TrustScoreSnapshot` materialization (#200 — already implemented, closed)
- Domain-specific aggregation in consumer subclass repositories

## References

- `api/spi/LedgerEntryRepository.java` — existing SPI (15 methods)
- `api/spi/OutcomeRecorder.java` — write path producing entry + attestation
- `api/model/OutcomeRecord.java` — demonstrates verdict is on attestation, not entry
- `api/model/AttestationVerdict.java` — SOUND / FLAGGED / ENDORSED / CHALLENGED
- `runtime/model/jpa/JpaLedgerEntry.java` — @NamedQuery declarations
- `persistence-memory/InMemoryLedgerEntryRepository.java` — in-memory impl
- `docs/protocols/casehub/ledger-entry-named-query.md` — PP-20260618-51c673
- `docs/protocols/casehub/per-subject-table-tenancy.md` — PP-20260616-05dc6a
- Issue #202 — streaming/cursor queries (qhorus E7)
- Issue #201 — aggregate queries (qhorus E5)
