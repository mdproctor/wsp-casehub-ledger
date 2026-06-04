---
layout: post
title: "Reading the Ledger Backwards"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [architecture, arc42, documentation, quarkus]
excerpt: "Completing ARC42STORIES.MD with no DESIGN.md meant reading 16 blog entries first — the only honest source for why the code is shaped the way it is."
---

`casehub-ledger` has 39 blog entries and no DESIGN.md. When the ARC42STORIES.MD stub sat with §3 through §13 as placeholders, the only honest way to fill them was to read the blogs first. We went through 16 of them before writing a sentence.

That's the right approach. Code inspection tells you what exists; blog entries tell you what shaped it — the papers consulted, the algorithms that didn't work first, the design decisions that landed differently than expected. An architecture document built only from the code misses all of that.

## What the stub got wrong

§1 listed `CaseLedgerEntry`, `WorkerDecisionEntry`, and `MerkleChain` as core capabilities. None of them exist. `CaseLedgerEntry` was the Tarkus-era entity name before the domain subclass model settled on `WorkItemLedgerEntry` and `MessageLedgerEntry`. `WorkerDecisionEntry` was an early design name that became `OutcomeRecord` and `PlainLedgerEntry`. `MerkleChain` split into `LedgerMerkleTree` and `LedgerMerkleFrontier`. A document that opens with three wrong class names actively misleads. The bootstrap stub had drifted quickly.

## The delivery arc

Reading the blogs in sequence, thirteen chapters emerged. The foundation extraction from Tarkus in April, the Merkle chain and PROV-DM work the same week, then the long trust scoring arc through May, bilateral agent signing and DID/VC identity in late May, platform identity extraction and OutcomeRecorder in June. Each chapter had a natural boundary — a cluster of issues that shipped together and a design decision that closed something previously open.

The EigenTrust chapter is a good example of why the blogs matter. The implementation found a convergence bug in the Stanford paper's recommended approach for small actor meshes. That's not in the code — it's in the reasoning that shaped the code. An architecture document that misses it loses the insight for anyone who has to work with the algorithm later.

## The layer taxonomy

The CaseHub profile gives Foundation modules latitude to define their own layer taxonomy. I settled on six: Audit Primitives, Tamper Evidence, Trust Infrastructure, Agent Identity, Compliance Surface, and Outcome Recording. Coarser than the issue structure, finer than the Maven module structure.

I wanted a taxonomy that answers "where does this concern live?" in a way that survives the next year of development. Keeping EigenTrust and Bayesian Beta under a single Trust Infrastructure layer felt right — they're two algorithms serving the same architectural purpose. Splitting them by algorithm would produce a map useful to a theorist but confusing to a developer who needs to add an attestation.

## The dedicated session myth

Both C4 diagram stubs said "add in dedicated session." I had everything needed to write them in Mermaid right now — four Maven modules, the consumer list, the platform identity dependency. The placeholder was bootstrap caution from when the stub was first generated, not a genuine constraint. I wrote both in the same session.

The pattern is worth noting: "dedicated session" in a stub usually means "I didn't have this information yet." Before deferring, check whether the condition still holds.
