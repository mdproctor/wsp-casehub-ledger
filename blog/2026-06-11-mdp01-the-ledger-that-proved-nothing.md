---
layout: post
title: "The ledger that proved nothing happened"
date: 2026-06-11
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [merkle, tamper-evidence, content-integrity, pipeline]
---

## The ledger that proved nothing happened

The Merkle chain was hashing the envelope and leaving the letter unsigned.

Six fields went into the leaf hash — `subjectId`, `seqNum`, `entryType`,
`actorId`, `actorRole`, `occurredAt`. That proves an entry existed at
position N, by actor Y, at time T. It does not prove what the entry said.
A DBA could change `caseStatus = REJECTED` to `APPROVED` and the chain
wouldn't notice. The agent signature covered the same six fields — the
bilateral signing infrastructure proved the agent signed something, not
what it signed.

The DESIGN.md rationale called this intentional: "domain labels do not
participate in tamper detection." I'd written that myself, and it was
wrong. It treated an implementation shortcut as a security principle. RFC
9162, Git, blockchain — every serious append-only log hashes content. We
were the outlier.

### The pipeline problem underneath

The fix wasn't just expanding the canonical form. The JPA save path
computed the digest *before* `em.persist()`, which triggered `@PrePersist`,
which ran the enricher pipeline. So even if we'd added `supplementJson`
to the hash months ago, provenance supplements set by
`ProvenanceCaptureEnricher` would have been excluded — the digest was
already locked in.

The in-memory path had it right by accident. It ran enrichers explicitly
before computing the digest. Two implementations of the same repository,
producing hashes with different content coverage, and no test caught it
because each path was verified in isolation.

### Signing is not enrichment

`AgentSignatureEnricher` sat in the enricher pipeline at `@Priority(20)`,
running before `ProvenanceCaptureEnricher` at 30 — except that
`ProvenanceCaptureEnricher` never actually had `@Priority(30)`. The
annotation was in the javadoc but never on the class. Both it and
`TraceIdEnricher` sorted to `Integer.MAX_VALUE`.

But the deeper problem wasn't ordering. The signature is not content
enrichment — it's a seal. It has a hard dependency on the digest being
final. Treating it as "enricher with a priority" obscured the
architectural distinction between adding data and closing the envelope.

We extracted it into `AgentEntrySigner` — a direct call in the save
pipeline, not a CDI-discovered enricher. The pipeline now reads linearly:
`prepareKey → enrich → hash → sign → persist`. The `prepareKey` phase
exists because `ActorIdentityValidationEnricher` needs the agent's public
key to check DID bindings, but the actual signature has to come after
hashing. Two-phase signing: key material first, cryptographic seal last.

### What the hash covers now

```
subjectId|seqNum|entryType|actorId|actorRole|occurredAt
  |tenancyId|actorType|causedByEntryId|supplementJson|domainContent
```

`canonicalBytes()` moved from `LedgerMerkleTree` (a static utility) to
`LedgerEntry` (a `public final` instance method). The structural encoding
is sealed — subclasses can't change it. They contribute domain-specific
bytes through `domainContentBytes()`, a protected override point. A
build-time guard in `LedgerProcessor` fails deployment if an `@Entity`
subclass has persistent fields but doesn't override it.

`KeyRotationEntry` and `ActorIdentityBindingEntry` already have their
overrides. Engine and qhorus need theirs — mechanical, one method each.

The chain now proves what entries contain, not just that they exist.
