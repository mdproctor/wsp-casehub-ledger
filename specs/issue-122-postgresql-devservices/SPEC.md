# PostgreSQL Integration Tests via Testcontainers

**Issue:** #122
**Date:** 2026-06-07
**Status:** Approved

## Problem

The test suite uses H2 in `MODE=PostgreSQL`, which emulates PostgreSQL syntax but does not replicate row-locking semantics, concurrent writer behaviour, or subtle SQL dialect differences. As native SQL enters the codebase (`LedgerSequenceAllocator` uses `MERGE INTO` for atomic sequence allocation), the gap between "tests pass on H2" and "correct on PostgreSQL" widens. Concurrent-write correctness (#100) can only be meaningfully tested against real PostgreSQL.

## Approach

Add PostgreSQL as an **additive** test layer alongside the existing H2 suite. Tests that exercise native SQL opt in via a shared `PostgreSQLTestProfile` base class. H2 tests remain untouched. Coverage, not replacement.

A `PostgreSQLTestResource` (implementing `QuarkusTestResourceLifecycleManager`) starts a Testcontainers `PostgreSQLContainer` and injects the JDBC URL, username, and password. Test resource config has the highest ordinal in the Quarkus config system — it cleanly overrides the H2 URL from `application.properties` without needing to "unset" properties.

### Why not Quarkus DevServices?

DevServices auto-activates when no explicit JDBC URL is configured. But `application.properties` has an unqualified `quarkus.datasource.jdbc.url=jdbc:h2:mem:...` that applies to all profiles, and `QuarkusTestProfile.getConfigOverrides()` cannot unset properties — it can only override them with new values. Setting the URL to empty string still counts as "configured" and prevents DevServices auto-activation. Explicit `QuarkusTestResourceLifecycleManager` is the established pattern in this project (`ScimWireMockResource`) and avoids config override gymnastics.

## Design

### PostgreSQLTestResource

Manages the Testcontainers PostgreSQL lifecycle. Returns config that overrides the H2 datasource:

```java
package io.casehub.ledger.test;

import org.testcontainers.postgresql.PostgreSQLContainer;

public class PostgreSQLTestResource implements QuarkusTestResourceLifecycleManager {

    static PostgreSQLContainer<?> container;

    @Override
    public Map<String, String> start() {
        container = new PostgreSQLContainer<>("postgres:16-alpine");
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
            container = null;
        }
    }
}
```

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

Each PostgreSQL variant is a new test class that extends the original H2 test and overrides only the profile. All `@Test` methods are inherited — zero test logic duplication.

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

### Tests to port

| Original | PostgreSQL variant | Reason |
|----------|-------------------|--------|
| `JpaSequenceNumberIT` | `JpaSequenceNumberPgIT` | Directly exercises `LedgerSequenceAllocator` MERGE INTO native SQL |
| `LedgerHealthJobIT` | `LedgerHealthJobPgIT` | Uses native SQL UPDATE to simulate sequence gaps |

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

**Modified files (1):**

| File | Change |
|------|--------|
| `runtime/pom.xml` | Add `org.testcontainers:testcontainers-postgresql` test scope |

## Out of scope

- Replacing H2 tests with PostgreSQL — additive only
- Concurrent-write tests (#100) — separate issue, will use this infrastructure when implemented
- CI matrix configuration — no CI exists yet; tests run in standard suite
- Porting JPQL-only tests — no benefit from PostgreSQL in this iteration
