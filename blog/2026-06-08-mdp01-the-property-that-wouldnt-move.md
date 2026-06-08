---
layout: post
title: "The Property That Wouldn't Move"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, testcontainers, postgresql, testing]
---

## The Property That Wouldn't Move

H2 in `MODE=PostgreSQL` has been the test database since the ledger's inception. It emulates PostgreSQL syntax well enough for JPQL, but as native SQL crept in — `LedgerSequenceAllocator` uses a SQL-standard `MERGE INTO` for atomic per-subject sequence allocation — the gap between "tests pass" and "correct on PostgreSQL" widened. I wanted to add real PostgreSQL as an additive test layer without touching the existing H2 suite.

The design looked clean: a `QuarkusTestResourceLifecycleManager` starts a Testcontainers PostgreSQL container and returns the JDBC URL, username, password, and `db-kind=postgresql`. Test resource config has the highest ordinal in the Quarkus config system — it overrides everything. Test variants extend the existing H2 test classes, inheriting all test methods. Zero duplication.

Except `quarkus.datasource.db-kind` is a build-time property. It gets baked into the application during augmentation, before the test resource's `start()` method ever runs. The highest config ordinal in the world doesn't matter if the property was already consumed and discarded.

The error message is clear enough once it fires: "Build time property cannot be changed at runtime: quarkus.datasource.db-kind is set to 'postgresql' but it is build time fixed to 'h2'." But the path to triggering it is not obvious at all. Nothing in the `QuarkusTestResourceLifecycleManager` API signals that its config map has a class of properties it simply cannot influence. The ordinal model suggests a total ordering — it doesn't advertise that build-time properties exist in a separate universe.

The fix is a clean split. `db-kind=postgresql` goes in `PostgreSQLTestProfile.getConfigOverrides()`, which participates in augmentation and triggers re-augmentation when the value differs from the default. Runtime properties — the JDBC URL, credentials — stay in the test resource where they belong. Build-time config in the profile; runtime config in the resource. Each property in the layer that can actually influence it.

A second surprise: the case-insensitive assertion. The parent test checked for a unique constraint violation by name — `hasMessageContaining("IDX_LEDGER_ENTRY_SUBJECT_SEQ")`. H2 uppercases all unquoted identifiers; PostgreSQL preserves lowercase. The fix was `hasMessageMatching("(?is).*idx_ledger_entry_subject_seq.*")` — the `s` flag (DOTALL) was necessary because H2 wraps its error messages across multiple lines, and without it `.` won't match newlines. A one-line fix, but it took a test failure to surface.

The result is four new files: a test resource, an abstract profile base, and two test variants that inherit every test method from their H2 parents. The pattern extends trivially — any future test that needs real PostgreSQL adds a 17-line class extending its H2 counterpart.
