# 0016 — PlainLedgerEntry as the domain-agnostic concrete LedgerEntry for OutcomeRecorder

Date: 2026-06-02
Status: Accepted

## Context and Problem Statement

`runtime/model/LedgerEntry` is abstract with `@Inheritance(strategy = JOINED)`. `OutcomeRecordSaveService` (added in issue #114) must instantiate a concrete `LedgerEntry` to persist generic EVENT entries via `OutcomeRecorder`. Because `LedgerEntry` is abstract, a concrete subclass is required. Existing subclasses (`KeyRotationEntry`, `ActorIdentityBindingEntry`) are domain-specific and semantically wrong for generic plugin outcome writes.

## Decision Drivers

* `OutcomeRecorder` is domain-agnostic — it must not assume any particular domain subclass
* The base class `LedgerEntry` must remain abstract (it enforces JOINED inheritance correctly)
* JPA and in-memory deployments must both work without consumer-specific code
* Flyway migrations must be additive and schema-minimal

## Considered Options

* **Option A: `PlainLedgerEntry`** — a minimal `@Entity @DiscriminatorValue("PLAIN")` subclass with no extra columns; join table is a single FK to `ledger_entry`
* **Option B: Consumer-provided subclass via SPI** — `OutcomeRecordSaveService` accepts a `Supplier<LedgerEntry>` allowing consumers to inject their domain subclass
* **Option C: Make `LedgerEntry` non-abstract with default discriminator** — remove `abstract`, add a `@DiscriminatorValue` on the base class itself

## Decision Outcome

Chosen option: **Option A (`PlainLedgerEntry`)**, because it is the narrowest change that solves the problem: the base class stays abstract (correct JPA design), the subclass carries no domain-specific fields, the V1009 migration is minimal (one table, one FK), and both JPA and in-memory paths work without consumer configuration.

### Positive Consequences

* No change to `LedgerEntry` — existing consumers and subclasses are unaffected
* `PlainLedgerEntry` works in in-memory deployments with no JPA overhead (no schema requirement)
* The join table adds zero columns — Hibernate loads `PlainLedgerEntry` entries with no extra JOIN cost beyond the base table

### Negative Consequences / Tradeoffs

* Adds V1009 migration (the spec incorrectly stated "No migrations" — corrected in the implementation plan)
* Any JPA consumer who adds `casehub-ledger` and calls `OutcomeRecorder` will have a `plain_ledger_entry` table that is always empty if they use their own domain subclasses — minor schema noise

## Pros and Cons of the Options

### Option A — `PlainLedgerEntry`

* ✅ No change to base class
* ✅ Minimal schema (one FK table)
* ✅ Domain-agnostic, works for all consumers
* ❌ Adds a migration the spec didn't anticipate

### Option B — Consumer subclass via SPI

* ✅ No new production subclass; consumer controls the type
* ❌ Forces every consumer to implement an extra SPI even for trivial use cases
* ❌ Breaks the "one call" promise of `OutcomeRecorder`

### Option C — Non-abstract `LedgerEntry`

* ✅ No new class; no new migration
* ❌ Changes the base class contract — existing consumers depend on `LedgerEntry` being abstract
* ❌ Hibernate behaviour for non-abstract JOINED-inheritance roots is less tested and may have edge cases with discriminator values

## Links

* casehubio/ledger#114 — OutcomeRecorder implementation
* `runtime/src/main/resources/db/ledger/migration/V1009__plain_ledger_entry.sql`
* `runtime/src/main/java/io/casehub/ledger/runtime/model/PlainLedgerEntry.java`
