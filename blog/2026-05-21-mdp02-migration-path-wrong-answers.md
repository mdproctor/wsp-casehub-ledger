---
layout: post
title: "The migration path with three wrong answers"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [ledger]
tags: [quarkus, flyway, extension, architecture]
---

A migration path move sounds like infrastructure housekeeping. SQL files, a directory rename, update some properties. Done in twenty minutes.

The interesting part was the decision about what the extension *should not* do.

The problem: `casehub-ledger` ships Flyway migrations at `classpath:db/migration` — the same default Quarkus uses for everything. When a consumer has both `casehub-ledger` and `casehub-work` on the classpath, and a second datasource needs to scan `classpath:db/migration` to pick up ledger's base tables, it also picks up `casehub-work`'s V1–V27 migrations. Flyway fails with `Found more than one migration with version 1`. It's been blocking Flyway re-enablement in the consumer repos for a while.

Moving to `classpath:db/ledger/migration` is straightforward. The question was how the extension should tell consumers about the new path.

**First candidate: ship a `runtime/application.properties` with the location.** The extension becomes self-configuring — add the dependency, migrations run. The problem is that `quarkus.flyway.locations` in an extension JAR replaces Quarkus's built-in default of `db/migration`. Any consumer who hasn't explicitly set their own locations now scans only `classpath:db/ledger/migration`. Their domain migrations, sitting in `db/migration`, silently don't run.

**Second candidate: give ledger its own named datasource.** Cleanly isolated — named datasources get their own Flyway config, no interference. But ledger must share the consumer's datasource to participate in the consumer's transactions. Audit integrity requires that a `LedgerEntry` and the domain entity change it records commit atomically. A named datasource means a separate transaction context. You'd need XA coordination to maintain that guarantee, which Quarkus doesn't wire by default. The named datasource is architecturally incompatible with what the ledger is for.

Explicit consumer configuration is the honest answer. Consumers add `classpath:db/ledger/migration` alongside whatever they already have. We added a `@BuildStep` in `LedgerProcessor` that checks whether any configured Flyway datasource includes the path and logs a warning at augmentation time if not — misconfiguration surfaces at build time rather than when Hibernate can't find the `ledger_entry` table.

One detail worth keeping: a void `@BuildStep` with no producer dependencies is a graph orphan. Quarkus may elide it. You anchor it with `@Produce(ArtifactResultBuildItem.class)` — from `io.quarkus.deployment.pkg.builditem`, not `io.quarkus.deployment.builditem`, which doesn't exist and the compiler error won't tell you where to look.

Claude caught a real mistake in the code review. I'd updated all nine example apps from `classpath:db/migration` to `classpath:db/ledger/migration`. Every example has its own domain migrations in `db/migration` — subclass join tables, domain entity schemas. By replacing instead of extending the locations list, I'd broken all of them. The correct config is `classpath:db/migration,classpath:db/ledger/migration`. What DESIGN.md documented was right; the examples contradicted it.

The follow-on for `casehub-qhorus` is less straightforward than it looked. Qhorus already has `V1003__agent_message_ledger_entry.sql` in its own migration path. Ledger has `V1003__ledger_entry_archive.sql`. Adding both paths to the qhorus Flyway config will immediately fail with a duplicate version error. That needs renumbering before the paths can be safely combined.
