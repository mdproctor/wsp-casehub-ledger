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
