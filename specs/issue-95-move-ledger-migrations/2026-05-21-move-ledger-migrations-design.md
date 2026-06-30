# Design Spec — Move Ledger Migrations to db/ledger/migration/
**Issue:** casehubio/ledger#95  
**Branch:** issue-95-move-ledger-migrations  
**Date:** 2026-05-21  

---

## Problem

`casehub-ledger` ships Flyway migrations at `classpath:db/migration/V1000–V1007`. This
path is shared with `casehub-work` (`V1–V27`) and any consumer's own domain migrations.

When both ledger and qhorus are on the classpath, the qhorus named datasource needs the
ledger base schema — but scanning `classpath:db/migration` to get it also picks up
`casehub-work` V1–V27, which conflicts with qhorus's own `V1__initial_schema.sql`.

This blocks Flyway re-enablement in `casehub-aml`, `casehub-clinical`, and
`casehub-devtown` tests (tracked: casehubio/aml#26).

## Decision: Explicit consumer configuration

There are three candidate mechanisms:

**Rejected — Runtime `application.properties` from the extension:** Setting
`quarkus.flyway.locations` in an extension JAR would replace Quarkus's built-in
default (`db/migration`). Consumers who rely on that default for their own domain
migrations silently lose them. Worse than the problem being fixed.

**Rejected — Named datasource for ledger:** A named datasource (e.g. `ledger`) would
give ledger its own Flyway namespace cleanly. But ledger must share the consumer's
datasource to participate in the consumer's transactions — that is the core audit
integrity guarantee. Separate datasource = separate transaction context = broken atomic
ledger writes. Architecturally incompatible.

**Chosen — Explicit consumer configuration with build-time validation:** Consumers
configure `classpath:db/ledger/migration` in their Flyway locations. The extension
validates this at build time and fails with an actionable error if it is absent.
The 8 example apps are the living reference implementation.

## Changes

### 1. Migration file relocation

Move all 8 SQL files:
```
runtime/src/main/resources/db/migration/V1000–V1007 (deleted)
→
runtime/src/main/resources/db/ledger/migration/V1000–V1007 (created)
```

File names, contents, and version numbers are unchanged. The `db/migration/` directory
is removed from the ledger runtime module entirely.

### 2. Build-time validation in LedgerProcessor

New `@BuildStep` in `LedgerProcessor` receives `FlywayBuildTimeConfig` and checks
whether any configured datasource includes `db/ledger/migration` in its locations. If
none do, it emits a build-time warning:

```
casehub-ledger is on the classpath but classpath:db/ledger/migration is not configured
in any Flyway datasource locations. Ledger tables will not be created.
Add classpath:db/ledger/migration to quarkus.flyway.locations (or
quarkus.flyway.<named-datasource>.locations if using a named datasource).
```

Warning rather than error: DDL-generation test environments (`database.generation=drop-and-create`)
legitimately omit `quarkus.flyway.locations`. A hard failure would break those builds.
The warning fires at build time — visible before any runtime failure — which is the
important property. Production deployments that ignore it get a Hibernate schema error;
that is an acceptable consequence of ignoring a build warning.

### 3. Test application.properties

```properties
# Before
quarkus.flyway.locations=classpath:db/migration

# After
quarkus.flyway.locations=classpath:db/ledger/migration
```

### 4. Example application.properties (8 files)

Same single-line change in each. Examples use ledger only (no casehub-work), so the
combined path is not needed:

```properties
# Before
quarkus.flyway.locations=classpath:db/migration

# After
quarkus.flyway.locations=classpath:db/ledger/migration
```

A consumer using both casehub-work and casehub-ledger would write:
```properties
quarkus.flyway.locations=classpath:db/migration,classpath:db/ledger/migration
```

### 5. FlywayLocationContractTest

Plain-Java test (no Quarkus) in `runtime/src/test/java`. Boots Flyway directly against
H2, targeting `classpath:db/ledger/migration`, and asserts `migrate()` succeeds. Pins
the canonical path as an explicit, fast-feedback contract independent of
`application.properties`.

```java
Flyway flyway = Flyway.configure()
    .dataSource("jdbc:h2:mem:contract;MODE=PostgreSQL;DB_CLOSE_DELAY=-1", "sa", "")
    .locations("classpath:db/ledger/migration")
    .load();
MigrateResult result = flyway.migrate();
assertThat(result.success).isTrue();
assertThat(result.migrationsExecuted).isEqualTo(8);
```

### 6. Documentation

- **CLAUDE.md** — update migration path references from `db/migration/` to
  `db/ledger/migration/`; document consumer configuration requirement
- **DESIGN.md** — update all `classpath:db/migration` references
- **PLATFORM.md** — update Flyway namespace table to document `classpath:db/ledger/migration`
  as the ledger path and state the explicit consumer configuration requirement

## TDD cycle

1. Update `application.properties` → existing `@QuarkusTest` ITs go **red** (Quarkus
   cannot find migrations at the old path)
2. Write `FlywayLocationContractTest` → **red** (no files at new path yet)
3. Move 8 SQL files to `db/ledger/migration/` → both go **green**
4. Add `@BuildStep` validation → verify it fires on a misconfigured app
5. Update examples and docs → full suite green

## Out of scope

- **casehubio/qhorus#179** — update `quarkus.flyway.qhorus.locations` to add
  `classpath:db/ledger/migration`; to be done this session after ledger#95 commits
- **casehubio/aml#26** — consumer apps add the new path; unblocked by this issue

## Consumer migration

Before: `quarkus.flyway.locations=classpath:db/migration`  
After: `quarkus.flyway.locations=classpath:db/migration,classpath:db/ledger/migration`

The build-time validation in `LedgerProcessor` catches misconfiguration immediately.