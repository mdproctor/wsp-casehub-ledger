---
layout: post
title: "Catching the Docs Before They Hit a Consumer"
date: 2026-04-20
type: correction
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, documentation, api-drift]
---

The Merkle sprint deleted `LedgerHashChain`. That was the right call — the class was replaced by `LedgerMerkleTree` and the write path was folded into `JpaLedgerEntryRepository.save()`. Consumers no longer touch hash computation at all.

The documentation didn't get the memo.

`docs/integration-guide.md` still showed a capture service calling `LedgerHashChain.compute(previousHash, entry)` and a REST endpoint calling `LedgerHashChain.verify(entries)`. `docs/examples.md` had the same pattern, plus tests asserting `entries.get(0).digest == entries.get(1).previousHash` — checking a field that no longer exists on `LedgerEntry`.

A consumer following the guide would have hit compile errors with no obvious path to the fix. That's the worst kind of documentation failure — it doesn't fail silently, but it also doesn't point anywhere useful.

The same sprint that deleted `ObservabilitySupplement` (moving `correlationId` and `causedByEntryId` to `LedgerEntry` core) never cleaned the supplements tables out of `DESIGN.md` or the integration guide. That one was subtler — no compile error, just incorrect API guidance.

We fixed all of it systematically: removed every `LedgerHashChain` reference, removed `previousHash` from the base fields table, removed `ObservabilitySupplement` from the supplement tables, and simplified the capture service to the correct pattern:

```java
// Merkle leaf hash and frontier update handled automatically by save()
ledgerRepo.save(entry);
```

That's what the write path does now. The guide should have said that from the moment the Merkle sprint landed.

The drift is understandable. TDD keeps the implementation correct; it doesn't keep the docs correct. The discipline has to be separate. If documentation had been part of the definition of done for the Merkle sprint — not just tests and a passing build, but also checking the integration guide — the gap wouldn't have opened.
