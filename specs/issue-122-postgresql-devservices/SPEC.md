# PostgreSQL Integration Tests via Testcontainers

**Issue:** #122
**Date:** 2026-06-07
**Status:** Approved

## Problem

The test suite uses H2 in `MODE=PostgreSQL`, which emulates PostgreSQL syntax but does not replicate row-locking semantics, concurrent writer behaviour, or subtle SQL dialect differences. As native SQL enters the codebase (`LedgerSequenceAllocator` uses `MERGE INTO` for atomic sequence allocation), the gap between "tests pass on H2" and "correct on PostgreSQL" widens. Concurrent-write correctness (#100) can only be meaningfully tested against real PostgreSQL.

## Approach

Add PostgreSQL as an **additive** test layer alongside the existing H2 suite. Tests that exercise native SQL or whose correctness depends on database-specific semantics opt in via a shared `PostgreSQLTestProfile` base class. H2 tests remain untouched. Coverage, not replacement.

A `PostgreSQLTestResource` (implementing `QuarkusTestResourceLifecycleManager`) starts a Testcontainers `PostgreSQLContainer` and injects the JDBC URL, username, and password. Test resource config has the highest ordinal in the Quarkus config system — it cleanly overrides the H2 URL from `application.properties` without needing to "unset" properties.

### Why not Quarkus DevServices?

DevServices auto-activates when no explicit JDBC URL is configured. But `application.properties` has an unqualified `quarkus.datasource.jdbc.url=jdbc:h2:mem:...` that applies to all profiles, and `QuarkusTestProfile.getConfigOverrides()` cannot unset properties — it can only override them with new values. Setting the URL to empty string still counts as "configured" and prevents DevServices auto-activation. Explicit `QuarkusTestResourceLifecycleManager` is the established pattern in this project (`ScimWireMockResource`) and avoids config override gymnastics.

### What this validates beyond native SQL

Every PostgreSQL test forces Flyway to run all V1000–V1008 migrations against real PostgreSQL. This validates database-specific features the H2 compatibility mode emulates but does not fully guarantee:

- `NULLS NOT DISTINCT` (V1001) — PostgreSQL 15+ feature; H2 emulates the syntax but PostgreSQL is the ground truth
- `BYTEA` columns (V1005) — PostgreSQL binary type, mapped differently in H2
- All `CHECK` constraints across V1005, V1006, V1008
- `DOUBLE PRECISION` default values (V1000, V1001, V1002)
- Index creation against the real PostgreSQL query planner

## Design

### PostgreSQLTestResource

Manages the Testcontainers PostgreSQL lifecycle. Returns config that overrides the H2 datasource:

```java
package io.casehub.ledger.test;

import org.testcontainers.postgresql.PostgreSQLContainer;

public class PostgreSQLTestResource implements QuarkusTestResourceLifecycleManager {

    private PostgreSQLContainer<?> container;

    @Override
    public Map<String, String> start() {
        container = new PostgreSQLContainer<>("postgres:17-alpine");
        container.start();
        return Map.of(
            "quarkus.datasource.db-kind", "postgresql",
            "quarkus.datasource.jdbc.url", container.getJdbcUrl(),
            "quarkus.datasource.username", container.getUsername(),
            "quarkus.datasource.password", container.getPassword()
        );
    }

    @Override
    public void stop() {
        if (container != null) {
            container.stop();
        }
    }
}
```

**Container field**: instance, not static — consistent with project convention (`ScimWireMockResource` uses `private WireMockServer server`). Quarkus creates one lifecycle manager instance per augmentation; `static` is unnecessary.

**PostgreSQL 17**: current stable release. The codebase targets PostgreSQL 15+ (the minimum for `MERGE` support). Using 17 validates forward compatibility. The `-alpine` variant minimises image size for CI.

**`db-kind` override**: setting `db-kind=postgresql` also changes the Hibernate dialect from H2 to PostgreSQL. Since `quarkus.hibernate-orm.database.generation=none` (Flyway manages schema), the dialect change only affects JPQL→SQL query generation, not DDL. This is safe — any JPQL dialect differences between H2 and PostgreSQL would surface here, which is additional coverage.

Test resource config has ordinal ~`Integer.MAX_VALUE`, overriding both `application.properties` (250) and `getConfigOverrides()` (500). The H2 URL is cleanly replaced regardless of which named profile is active.

Location: `runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestResource.java`

### PostgreSQLTestProfile base class

Registers the test resource via `testResources()`:

```java
package io.casehub.ledger.test;

public abstract class PostgreSQLTestProfile implements QuarkusTestProfile {
    @Override
    public List<TestResourceEntry> testResources() {
        return List.of(new TestResourceEntry(PostgreSQLTestResource.class));
    }
}
```

Subclasses override `getConfigProfile()` to activate their named profile for feature flags (trust-score.enabled, scheduler.enabled, etc.). The datasource swap is inherited from the test resource — no `getConfigOverrides()` needed.

Location: `runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestProfile.java`

### Test variants via subclass inheritance

Each PostgreSQL variant is a new test class that extends the original H2 test and overrides only the profile. All `@Test` methods are inherited — zero test logic duplication, with one exception noted below.

```java
@QuarkusTest
@TestProfile(JpaSequenceNumberPgIT.Profile.class)
class JpaSequenceNumberPgIT extends JpaSequenceNumberIT {
    public static class Profile extends PostgreSQLTestProfile {
        @Override
        public String getConfigProfile() {
            return "sequence-number-test";
        }
    }
}
```

### Parent test fix: case-insensitive constraint assertion

`JpaSequenceNumberIT.uniqueConstraintPreventsDuplicateSequenceNumber()` (line 137) asserts:
```java
.hasMessageContaining("IDX_LEDGER_ENTRY_SUBJECT_SEQ");
```

H2 uppercases all unquoted identifiers, so it reports `IDX_LEDGER_ENTRY_SUBJECT_SEQ`. PostgreSQL preserves case — it reports `idx_ledger_entry_subject_seq`. This assertion will fail when inherited by the PostgreSQL subclass.

**Fix**: change the parent test to use a case-insensitive assertion:
```java
.hasMessageMatching("(?i).*idx_ledger_entry_subject_seq.*");
```

This makes the parent test portable across both databases. Since there are no deployed users, fixing the parent is the right design — no backward compatibility concern.

### Test selection criteria

Tests are selected for PostgreSQL variants when they meet any criterion:

1. **Direct native SQL**: the test or the production code it exercises issues `createNativeQuery()` with database-specific syntax
2. **Complex JPQL aggregation**: the test exercises `GROUP BY`/`HAVING` queries with mixed aggregate types where dialect differences surface (e.g. `COUNT` returns `bigint` on PostgreSQL vs `long` on H2)
3. **Database-specific semantics**: the test's correctness depends on behaviour that differs between H2 and PostgreSQL (row locking, constraint enforcement timing, etc.)

Simple JPQL-only tests (single-entity CRUD, named queries without aggregation) are excluded: Hibernate generates database-portable SQL from these, so running them on PostgreSQL adds no coverage beyond the Flyway migration validation (which the selected tests already provide).

### Tests to port

| Original | PostgreSQL variant | Reason |
|----------|-------------------|--------|
| `JpaSequenceNumberIT` | `JpaSequenceNumberPgIT` | Validates that the SQL-standard `MERGE INTO ... AS t USING (SELECT CAST(?1 AS UUID) AS sid) AS s ON ...` syntax in `LedgerSequenceAllocator` works correctly on PostgreSQL 15+. This is the headline validation — the `MERGE` statement is the primary native SQL in production code. |
| `LedgerHealthJobIT` | `LedgerHealthJobPgIT` | Exercises complex JPQL aggregation (`GROUP BY`/`HAVING` with mixed numeric types — `COUNT` returns `bigint`, `MIN`/`MAX` return `integer` on PostgreSQL) against real PostgreSQL dialect; test setup uses native SQL `UPDATE` to create sequence gaps. |

**Excluded**: `ActorTrustScoreRepositoryIT` uses JPQL (named queries via `em.createNamedQuery()`), not native SQL. The `NULLS NOT DISTINCT` unique constraint (V1001) is validated by Flyway migration success, which both selected tests already exercise. Adding a third variant would be diminishing returns.

### Dependencies

Add to `runtime/pom.xml`:

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers-postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

Testcontainers 2.0 renamed the artifact from `postgresql` to `testcontainers-postgresql` and the package from `org.testcontainers.containers` to `org.testcontainers.postgresql`. The Quarkus BOM (3.32.2) manages the version (2.0.3) — no explicit version tag needed.

`quarkus-jdbc-postgresql` is already `<optional>true</optional>` in the runtime pom — optional dependencies are on the compile and test classpaths of the declaring module, so the PostgreSQL JDBC driver is already available for tests.

### Test isolation

- Tests with different profile subclasses get separate Quarkus augmentations and separate Testcontainers instances. Same isolation model as the existing H2 pattern (separate DB URLs per profile).
- Tests within the same profile share a container. `@Transactional` test methods roll back automatically.
- Each augmentation starts with a fresh Flyway migration against an empty database.

### CI

No CI configuration exists in this repo. PostgreSQL tests run as part of `mvn test` — Surefire picks up `*PgIT.java` classes automatically. Docker must be running (hard prerequisite; Testcontainers error message is clear on failure).

## File inventory

**New files (4):**

| File | Purpose |
|------|---------|
| `runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestResource.java` | Testcontainers lifecycle manager |
| `runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestProfile.java` | Abstract base — registers test resource |
| `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberPgIT.java` | Extends JpaSequenceNumberIT |
| `runtime/src/test/java/io/casehub/ledger/service/LedgerHealthJobPgIT.java` | Extends LedgerHealthJobIT |

**Modified files (2):**

| File | Change |
|------|--------|
| `runtime/pom.xml` | Add `org.testcontainers:testcontainers-postgresql` test scope |
| `runtime/src/test/java/.../JpaSequenceNumberIT.java` | Case-insensitive constraint name assertion (line 137) |

## Out of scope

- Replacing H2 tests with PostgreSQL — additive only
- Concurrent-write tests (#100) — separate issue, will use this infrastructure when implemented
- CI matrix configuration — no CI exists yet; tests run in standard suite
- Porting JPQL-only tests — no benefit beyond Flyway migration validation already provided by selected tests
