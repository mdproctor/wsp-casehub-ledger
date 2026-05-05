---
layout: post
title: "Documentation That Lies"
date: 2026-04-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, documentation, capabilities, privacy, gdpr]
---

The session started with a question I hadn't fully sat with: which of these
features are actually useful in the real world, and which are academic
exercises?

## Honest About EigenTrust

Every extension gets described by its author as if every feature is equally
essential. I didn't want to do that. So I rated each capability on a five-star
scale for enterprise applicability — privacy pseudonymisation, decision context
snapshots, and provenance tracking all landed at ★★★★★ because they're either
legally mandated or the answer to questions every enterprise compliance team asks.
EigenTrust transitivity got ★★☆☆☆. The honest framing: it solves a real problem
that isn't mainstream yet. The original EigenTrust paper was designed for P2P
file-sharing networks where you can't trust a central authority. Enterprise audit
systems don't have that problem — they have a known actor set. The feature exists
because large-scale AI agent meshes *will* have that problem, and building the
foundation correctly now is cheaper than rearchitecting later. But I'm not going
to pretend a procurement team will ask for eigenvector computation in 2026.

That conversation became `docs/CAPABILITIES.md` — nine capabilities, each with
regulatory drivers, applicability rating, and "enable when / skip when" guidance.
The supplement system itself gets its own section, because the architectural
decision to use zero-cost optional field sets is worth explaining to someone
evaluating whether to adopt the extension.

## Documentation That Lies

While we were in the docs, I asked Claude to do a systematic review of every
markdown file in the project — README, DESIGN, AUDITABILITY, RESEARCH,
integration guide, examples, compliance doc, the new CAPABILITIES. Look for
staleness, drift, conflicts, and duplication.

What came back wasn't prose polish. It was real bugs.

`previousHash` was listed in the README's core entity fields table. That field
doesn't exist. The Merkle Mountain Range work replaced the linear hash chain
months ago — entries no longer chain via `previousHash`, they accumulate into
an MMR frontier. The README was describing an entity field that would be absent
from any code a consumer wrote.

`findById` appeared as an `@Override` in both the integration guide and the
examples. The actual SPI method is `findEntryById` — renamed to avoid a
return-type conflict with Panache. Consumer code following those examples would
fail to compile.

`ObservabilitySupplement` showed up in five places as if it still existed. It
was deleted when `correlationId` and `causedByEntryId` moved to core fields on
`LedgerEntry`. The supplement table was gone from source; the docs hadn't caught up.

We fixed all of it. Eight files touched, `PRIVACY.md` added as a new first-class
document covering what consumers must decide before going to production with
human actor identities. The responsibility boundary table at the end of that
document is the clearest statement I've written of where the extension's job ends
and the consumer's begins.

The session ends with a two-session plan ahead: EigenTrust transitivity next,
then the LLM agent mesh epic.
