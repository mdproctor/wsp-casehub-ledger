---
layout: post
title: "Teaching the Ledger to Speak W3C"
date: 2026-04-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, prov-dm, w3c, provenance, interoperability]
---

The W3C PROV-DM mapping was obvious before we wrote a line of code. `LedgerEntry` maps to `prov:Entity` — it's the thing with a provenance story. `actorId` maps to `prov:Agent`. Each entry's action maps to `prov:Activity`. The canonical relations follow: `wasGeneratedBy`, `wasAssociatedWith`, `wasDerivedFrom` for the sequential chain. This is the kind of mapping that validates itself.

Two decisions shaped the scope.

First: PROV-JSON or PROV-JSON-LD? JSON-LD adds a single `@context` field to PROV-JSON. Any tool reading PROV-JSON ignores `@context` and works identically. We get RDF store importability for free. That's not a tradeoff — it's a free upgrade. We did JSON-LD.

Second: per-entry export or per-subject? Per-subject. Regulators don't ask about one event in isolation. They ask what happened to this aggregate. A per-subject export gives the complete provenance graph in one call.

Two things in the mapping were non-obvious.

Agent deduplication. The same `actorId` often appears across many entries — a classifier agent that acts on entries 1, 3, and 7 of a subject. A naive implementation emits three identical agent nodes. We deduplicate within each export: same `actorId` across N entries produces exactly one `prov:Agent` node.

`wasDerivedFrom` covers two structurally different relationships. The sequential chain (entry 3 derives from entry 2) and cross-subject causality (`causedByEntryId` — an entry in subject B was caused by an entry in subject A). PROV-DM doesn't distinguish them; both are derivation edges. We emit both as `wasDerivedFrom` with different blank node keys. A downstream consumer gets both relationships without needing to understand the internal distinction.

`ProvenanceSupplement` translated naturally. It already carries `sourceEntityId`, `sourceEntityType`, and `sourceEntitySystem`. We construct a `hadPrimarySource` IRI: `ledger:external/<type>/<system>/<id>`. A consumer can locate the external entity by parsing the IRI components — no side-channel needed.

The code is a pure static utility, `LedgerProvSerializer.toProvJsonLd()`, using Jackson's `ObjectMapper` and `LinkedHashMap` for deterministic key ordering. A CDI bean, `LedgerProvExportService`, fetches entries and initialises lazy supplements within a transaction boundary before delegating to the serialiser. That split keeps the mapping logic independently testable — 13 unit tests run with no Quarkus container.

`docs/prov-dm-mapping.md` documents every field, every supplement, and every IRI convention in one place. The goal was that someone implementing a consumer of the export — or verifying compliance with GDPR Art.22 — could find the answer without reading the source code.
