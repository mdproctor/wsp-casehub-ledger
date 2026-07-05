# 0017 — Two-tier JPA model hierarchy for persistence-agnostic api types

Date: 2026-07-05
Status: Accepted

## Context and Problem Statement

The ledger's api module had shadow copies of entity classes (`LedgerEntry`, `LedgerAttestation`, supplements) with zero inheritance relationship to the runtime `@Entity` classes. Consumers used only the runtime types — the api copies were dead code. The runtime `LedgerEntry` bundled data model fields, tamper-evidence logic (`canonicalBytes()`), and JPA entity mapping (`@Entity`, `@NamedQuery`, `@EntityListeners`) on a single class, tying all consumers to JPA.

## Decision Drivers

* Consumers (blocks, engine) need the write SPI (`LedgerEntryRepository`) at the api tier without depending on ledger runtime
* The platform supports H2, PostgreSQL, MongoDB, and Redis — persistence must be behind the SPI, not baked into model types
* PLATFORM.MD mandates "consumer-facing SPI interfaces go in `api/spi/`" and "JPA entities must not co-locate with domain SPIs"
* `@MappedSuperclass` with `@Column` is already the established pattern (`api.model.LedgerAttestation`)

## Considered Options

* **Option A** — New `LedgerAppender` SPI with value type only, keep entity in runtime
* **Option B** — Two-tier hierarchy: `@MappedSuperclass` in api, `@Entity` in runtime
* **Option C** — Move `@Entity` to api module
* **Option D** — Three-tier: plain abstract class → `VerifiedLedgerEntry` → `JpaLedgerEntry`

## Decision Outcome

Chosen option: **Option B** — two-tier hierarchy, because it fixes the root cause (parallel type trees), follows the existing `LedgerAttestation` pattern, and keeps api types persistence-agnostic while preserving full JPA mapping capability in runtime.

### Positive Consequences

* `LedgerEntryRepository` moves to `api/spi/` — consumers depend on api, not runtime
* Api types work with any persistence backend (JPA, MongoDB, in-memory)
* `canonicalBytes()` and signing fields are on the api type — any backend maintaining tamper evidence can use them
* Supplement JOINED inheritance eliminated — each supplement type is an independent entity, enabling `instanceof` type safety in api-tier helpers

### Negative Consequences / Tradeoffs

* Consumer subclasses must extend `JpaLedgerEntry` (not `LedgerEntry`) for JOINED inheritance — a breaking change
* `@Entity(name = "LedgerEntry")` needed on `JpaLedgerEntry` to preserve JPQL entity name
* `LedgerAttestation` type boundary creates friction (unchecked casts in in-memory impl) — a known gap to clean up

## Pros and Cons of the Options

### Option A — LedgerAppender with value type only

* ✅ No model changes — minimal blast radius
* ✅ Clean Tier 1 SPI
* ❌ Papers over the dual-type flaw — two parallel class trees remain
* ❌ Consumers needing entity subclass writes still depend on runtime

### Option B — Two-tier `@MappedSuperclass` / `@Entity`

* ✅ Fixes the root cause — one type hierarchy
* ✅ Follows existing `LedgerAttestation` pattern
* ✅ Api types are persistence-agnostic
* ✅ `LedgerEntryRepository` moves cleanly to api/spi
* ❌ Breaking change: every consumer import changes
* ❌ `@MappedSuperclass` has subtle Hibernate bytecode enhancement constraints (see GE-20260705-7c0e86)

### Option C — Move `@Entity` to api

* ✅ Simplest end state — one class, one location
* ❌ Forces datasource configuration on any Quarkus app depending on `casehub-ledger-api`
* ❌ Violates PLATFORM.MD tier separation rule

### Option D — Three-tier with `VerifiedLedgerEntry`

* ✅ Cleanest domain separation (data model → verification → JPA)
* ❌ JPA cannot map fields from a non-`@MappedSuperclass` ancestor — unimplementable without `@MappedSuperclass` on the middle tier, which collapses it back to Option B

## Links

* [#168](https://github.com/casehubio/ledger/issues/168) — Extract ledger write SPI to ledger-api
* [#174](https://github.com/casehubio/ledger/issues/174) — Platform persistence unification
* GE-20260705-7c0e86 — Hibernate bytecode enhancement strips @Transient from @MappedSuperclass
* Design review: `specs/issue-168-extract-ledger-write-spi/2026-07-05-extract-ledger-write-spi-design.md`
