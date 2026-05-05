---
layout: post
title: "From O(N) to O(log N)"
date: 2026-04-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, merkle, cryptography, verifiability, architecture]
---

The last entry ended with issue #11 on deck. The research matrix rated it ★★ — "right move eventually, not urgent." I wasn't sure I agreed. External verifiability is what separates a tamper-evident log from a genuine audit trail. Without it, a regulator checking an AI system's audit record has to trust the operator's own report. That's not verification.

The brainstorm surfaced three options for the internal Merkle structure.

**Full node storage** (~2N rows per subject): every intermediate tree node persisted. Clean proof generation — just walk the pre-stored nodes. But storage grows proportionally to entry count, and managing node mutations on every append is fiddly.

**Checkpoint windows**: keep the linear chain, periodically compute a Merkle root over a batch. Simple. But entries in the current open window have no proof until the window closes. Not good enough.

**Stored frontier / Merkle Mountain Range**: the RFC 9162 approach. At most log₂(N) rows per subject — one per set bit in N's binary representation. A million entries produces at most 20 frontier rows. Inclusion proof generation fetches O(log N) digests at query time. That's the right tradeoff, and it's what's in the spec. We recorded it as ADR 0002 before writing a line of code.

The next question was external publishing. My initial instinct: expose a REST endpoint returning the current tree root, auditor calls it. I talked myself out of this immediately. A pull endpoint means the auditor still trusts the operator to return the honest root. Push to an external log the operator doesn't control is what makes verification genuinely independent. Both, then: pull for self-verification, push for external auditability.

For the push format, I wanted to know what state of the art actually used before inventing anything. We researched RFC 9162, Sunlight (FiloSottile), and Trillian. The answer converged: Ed25519 asymmetric signing with the c2sp.org tlog-checkpoint text format. HMAC with a shared secret doesn't work — you'd need to distribute and rotate the secret to every receiver, and it gives no non-repudiation. The operator could forge a checkpoint. Ed25519 is native Java since version 15, zero extra dependencies.

The implementation has one correctness subtlety worth documenting. The leaf hash is `SHA-256(0x00 | canonicalBytes)`. The `0x00` byte is RFC 9162 domain separation — it prevents a second-preimage attack where an adversary constructs an internal node value that matches a valid leaf. Internal nodes use `0x01 | left_bytes | right_bytes`. Without the prefix the tree is theoretically forgeable.

During code review, Claude caught that `LedgerVerificationService.inclusionProof()` was computing the tree root twice — once inside the pure static method by rebuilding the frontier from all leaf hashes, and once by reading the stored frontier from the database. The first computation used a dummy UUID for `subjectId` (the static method has no DB access), which would silently return a valid-but-unlabelled root that the service immediately discarded. We fixed it: the static method now returns an empty string for `treeRoot`, and the service fetches the authoritative root from the stored frontier in one DB call.

Axiom 4 (Verifiability) is now ✅. The extension can produce a proof that any party can verify without database access, without trusting the operator, and without knowing the JPA schema.
