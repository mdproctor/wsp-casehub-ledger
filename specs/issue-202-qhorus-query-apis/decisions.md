## D1: No outcome field on LedgerEntry

**Choice:** Keep the entry/attestation separation — aggregate queries JOIN entries with attestations and group by `AttestationVerdict`
**Alternatives:**
- Add `outcome` field to `LedgerEntry` — simpler queries but conflates decision with assessment, breaks multi-attestor scoring model
**Rationale:** The ledger separates "what happened" (LedgerEntry) from "how it was judged" (LedgerAttestation). OutcomeRecord already writes both in one TX. An outcome field would make it impossible for the same entry to receive different verdicts from different attestors.
**Trade-offs:** Aggregate queries require a JOIN instead of a simple GROUP BY on a single table. Unit tests must demonstrate qhorus derivation patterns (fulfillment rate, routing success, per-channel quality) to prove the model is sufficient.
**Sources:** OutcomeRecord.java, LedgerEntry.java, LedgerAttestation verdict model
**Exploration:** deep-analysis
**Status:** captured

## D2: Single SPI — no LedgerQueryRepository split

**Choice:** Add streaming and aggregate methods to the existing `LedgerEntryRepository` interface
**Alternatives:**
- Split into `LedgerQueryRepository` for read-path additions — cleaner interface segregation but doubles CDI wiring and test stubbing
**Rationale:** The repository already mixes reads and writes, and the in-memory implementation backs all methods from one store. Streaming and aggregate methods are new query shapes over the same entity graph, not a different responsibility.
**Trade-offs:** Interface grows to ~20+ methods. Acceptable for a repository SPI.
**Sources:** LedgerEntryRepository.java (15 existing methods), InMemoryLedgerEntryRepository.java
**Exploration:** quick
**Status:** captured

## D3: Both streaming and cursor-based pagination

**Choice:** Option C — add `Stream<LedgerEntry>` methods for in-process traversal AND cursor-based pagination for paged external access
**Alternatives:**
- Stream only (Option A) — simpler but forces fetch loops for REST/GraphQL consumers
- Cursor only (Option B) — no resource lifecycle concerns but unnatural for in-process batch traversal
**Rationale:** Three methods total (2 stream + 1 cursor) is light. Streaming fits qhorus formal verification (in-process batch). Cursor fits future REST/GraphQL paging. Both are additive, non-overlapping use cases. In-memory cursor implementation is a trivial subList().
**Trade-offs:** Stream methods require callers to manage resource lifecycle (try-with-resources, transactional scope). SPI Javadoc must document this clearly.
**Sources:** Issue #202, JPA `getResultStream()` API
**Exploration:** quick
**Status:** captured

## D4: Simple maps and records for aggregate return types

**Choice:** Concrete return types — `Map<AttestationVerdict, Long>` for verdict counts, `AttestationSummary` record for the full summary
**Alternatives:**
- Generic `AggregateResult` container — more extensible but over-engineered for three concrete queries
**Rationale:** Three specific methods with clear return types. `AttestationSummary` record lives in `api/model/`. No abstraction tax for a known, bounded set of aggregations.
**Trade-offs:** Adding a new aggregation dimension later requires a new method + return type rather than fitting into a generic framework. Acceptable given YAGNI.
**Sources:** Issue #201, AttestationVerdict enum
**Exploration:** quick
**Status:** captured
