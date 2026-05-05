---
layout: post
title: "Extracting a Shared Audit Ledger for the Quarkus AI Ecosystem"
date: 2026-04-15
type: day-zero
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, jpa, flyway, audit]
---

The conversation that led to casehub-ledger started with a different question: could Qhorus borrow some of Tarkus's audit patterns? Tarkus already had hash chains, decision context snapshots, and peer attestations built. Qhorus needed something similar for EU AI Act compliance.

Midway through the analysis I stopped. Both Qhorus and Tarkus need audit ledgers. CaseHub probably will too. The right answer is extraction, not copying.

The first design problem was making `LedgerEntry` genuinely domain-agnostic. Tarkus's version has `workItemId`, `commandType`, `eventType` baked in — all WorkItem-specific. The solution is JPA JOINED inheritance with an abstract base: a `ledger_entry` table holding common fields, and domain-specific subclass tables joining on `id`. Tarkus adds `WorkItemLedgerEntry` with its lifecycle fields. Qhorus adds `AgentMessageLedgerEntry` with tool telemetry. Neither touches the other.

The key abstraction was renaming `workItemId` to `subjectId` — a generic aggregate identifier. Sequence numbering and hash chaining both scope per `subjectId`. Each consuming domain sets it to whatever their aggregate is.

The hash chain canonical form needed simplifying. Previously it included `commandType` and `eventType` — now those live in subclasses. The new form uses only base fields: `subjectId|seqNum|entryType|actorId|actorRole|planRef|occurredAt`. This covers what tamper detection needs without coupling the chain to domain labels.

One decision that mattered for CDI: marking `JpaLedgerEntryRepository` as `@Alternative`. Without it, when a consuming extension provides its own typed repository, CDI sees two beans implementing `LedgerEntryRepository` and fails. `@Alternative` means the base implementation yields automatically when a domain-specific one is present.

Flyway migration numbering was a real gotcha. The subclass join table has `FOREIGN KEY ... REFERENCES ledger_entry (id)`. If you number it V5, it runs before the base schema migration V1000 creates `ledger_entry`, and startup fails with `Table "LEDGER_ENTRY" not found`. Flyway merges all classpath migrations globally and sorts by version number — there's no concept of "extension migrations run first." We discovered this live when building the order-processing example. The rule is straightforward once you know it: domain tables live in V1–V999, subclass join tables in V1002+.

Beyond the extension itself: we migrated `casehub-work` to depend on it, replacing 14 classes with `WorkItemLedgerEntry` and a typed repository. The 69 Tarkus ledger tests pass unchanged in behaviour, just with different imports.

The extension ships with a runnable example at `examples/order-processing/` — `mvn test` gives you a real Quarkus app tracking an order lifecycle with full audit trail, hash chain verification, and peer attestations via REST.
