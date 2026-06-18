---
layout: post
title: "The Race That Wasn't"
date: 2026-06-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [concurrency, h2, postgresql, sequence-allocation]
---

The issue title left nothing ambiguous: "ON CONFLICT DO NOTHING masks concurrent data race instead of fixing it." The analysis was thorough — traced the commit history back to the change, named the specific pattern, proposed three alternative fixes.

The core claim: two threads racing to insert the same `(subject_id, tenancy_id)` row for the first time would both proceed, DO NOTHING lets them both in, and the second thread reads a stale sequence number. Silent corruption in an immutable audit ledger.

I brought Claude in to trace the full execution path before accepting or challenging it.

The critical question is whether H2 in MODE=PostgreSQL blocks TX2 at the INSERT step. If TX1 holds an index-entry lock on the new row and TX2 blocks until TX1 commits — then DO NOTHING correctly handles the expected post-commit conflict, and the subsequent UPDATE row lock serialises the Merkle frontier update. If H2 uses purely optimistic MVCC and both inserts proceed without blocking — then the DO NOTHING is genuinely masking something.

The concurrent test answers it. Five threads fire simultaneously at the same new subject pair. All five return distinct, contiguous sequence numbers. If H2 used purely optimistic MVCC, two threads would compute the same `next_seq - 1` and the assertion would fail. It doesn't. H2 MODE=PostgreSQL does block TX2 at the INSERT. The implementation was correct.

But the investigation surfaced something real.

The DO NOTHING + UPDATE pattern works, but it's two statements where one would do. We tried replacing it with a single `INSERT … ON CONFLICT (subject_id, tenancy_id) DO UPDATE SET next_seq = next_seq + 1` — the canonical PostgreSQL upsert. On real PostgreSQL, it works. H2 2.4.240 in MODE=PostgreSQL rejects it: `SQLGrammarException [42000-240]`. H2 accepts the no-column-target `DO NOTHING` form but rejects the named-target `DO UPDATE` form, even in PG mode.

That's not documented anywhere. The H2 changelog mentions `ON CONFLICT DO NOTHING` support in MODE=PostgreSQL. `DO UPDATE` being excluded is only discoverable by trying it.

The result: a three-way dialect enum. Real PostgreSQL gets the single-statement upsert — DO UPDATE acquires the row lock that holds the Merkle Serialization Invariant without a second statement. H2 in MODE=PostgreSQL keeps the two-statement path. H2 standard gets a full MERGE with both WHEN MATCHED UPDATE and WHEN NOT MATCHED INSERT, making it a single-statement upsert too, where before it was a partial MERGE followed by a separate UPDATE.

The diagnosis was wrong on the race. But being challenged to prove it forced the kind of careful execution trace that wouldn't have happened otherwise — and that's where the H2 limitation surfaced.

The concurrent test isn't just a regression guard. In this case it was the only reliable proof. The blocking behaviour of H2's index-entry lock in MODE=PostgreSQL isn't documented; without a passing 5-thread concurrent test, the analysis would have been inconclusive.
