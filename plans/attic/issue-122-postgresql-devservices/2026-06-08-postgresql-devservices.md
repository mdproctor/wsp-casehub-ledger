# PostgreSQL Integration Tests via Testcontainers — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add PostgreSQL Testcontainers as an additive test layer — validate native SQL and complex JPQL aggregation against real PostgreSQL while keeping existing H2 tests untouched.

**Architecture:** A `PostgreSQLTestResource` (Testcontainers lifecycle manager) starts a PostgreSQL container and injects JDBC config at the highest ordinal, cleanly overriding H2. An abstract `PostgreSQLTestProfile` registers this resource. Test variants extend existing H2 test classes — zero test logic duplication.

**Tech Stack:** Quarkus 3.32.2, Testcontainers 2.0.3 (managed by Quarkus BOM), PostgreSQL 17, JUnit 5

**Spec:** `specs/issue-122-postgresql-devservices/SPEC.md`

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `runtime/pom.xml` | Modify | Add `testcontainers-postgresql` test dependency |
| `runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestResource.java` | Create | Testcontainers lifecycle — starts/stops PostgreSQLContainer, injects JDBC config |
| `runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestProfile.java` | Create | Abstract QuarkusTestProfile — registers PostgreSQLTestResource |
| `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java` | Modify | Fix case-sensitive constraint name assertion (line 137) |
| `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberPgIT.java` | Create | PostgreSQL variant — extends JpaSequenceNumberIT |
| `runtime/src/test/java/io/casehub/ledger/service/LedgerHealthJobPgIT.java` | Create | PostgreSQL variant — extends LedgerHealthJobIT |

---

### Task 1: Add Testcontainers PostgreSQL dependency

**Files:**
- Modify: `runtime/pom.xml:107-112` (after the wiremock dependency, before `</dependencies>`)

- [ ] **Step 1: Add the dependency**

Add immediately before the `</dependencies>` closing tag in `runtime/pom.xml`:

```xml
    <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>testcontainers-postgresql</artifactId>
      <scope>test</scope>
    </dependency>
```

No `<version>` tag — the Quarkus BOM (`io.quarkus.platform:quarkus-bom:3.32.2`) manages Testcontainers 2.0.3.

- [ ] **Step 2: Verify dependency resolves**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn dependency:resolve -pl runtime -q`
Expected: exits 0, no errors. The `testcontainers-postgresql` artifact resolves from Maven Central.

- [ ] **Step 3: Commit**

```
git add runtime/pom.xml
git commit -m "build(#122): add testcontainers-postgresql test dependency

Refs #122"
```

---

### Task 2: Create PostgreSQLTestResource

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestResource.java`

- [ ] **Step 1: Write PostgreSQLTestResource**

```java
package io.casehub.ledger.test;

import java.util.Map;

import org.testcontainers.postgresql.PostgreSQLContainer;

import io.quarkus.test.common.QuarkusTestResourceLifecycleManager;

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
                "quarkus.datasource.password", container.getPassword());
    }

    @Override
    public void stop() {
        if (container != null) {
            container.stop();
        }
    }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q`
Expected: exits 0. The Testcontainers PostgreSQL import resolves; `QuarkusTestResourceLifecycleManager` is on the classpath.

- [ ] **Step 3: Commit**

```
git add runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestResource.java
git commit -m "test(#122): PostgreSQLTestResource — Testcontainers lifecycle manager

Refs #122"
```

---

### Task 3: Create PostgreSQLTestProfile

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestProfile.java`

- [ ] **Step 1: Write PostgreSQLTestProfile**

```java
package io.casehub.ledger.test;

import java.util.List;

import io.quarkus.test.junit.QuarkusTestProfile;

public abstract class PostgreSQLTestProfile implements QuarkusTestProfile {

    @Override
    public List<TestResourceEntry> testResources() {
        return List.of(new TestResourceEntry(PostgreSQLTestResource.class));
    }
}
```

- [ ] **Step 2: Verify it compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q`
Expected: exits 0.

- [ ] **Step 3: Commit**

```
git add runtime/src/test/java/io/casehub/ledger/test/PostgreSQLTestProfile.java
git commit -m "test(#122): PostgreSQLTestProfile — abstract base registering test resource

Refs #122"
```

---

### Task 4: Fix case-sensitive constraint assertion in JpaSequenceNumberIT

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java:137`

- [ ] **Step 1: Change the assertion to case-insensitive**

In `JpaSequenceNumberIT.java`, line 137, replace:

```java
        }).hasMessageContaining("IDX_LEDGER_ENTRY_SUBJECT_SEQ");
```

with:

```java
        }).hasMessageMatching("(?i).*idx_ledger_entry_subject_seq.*");
```

This makes the assertion portable across H2 (which uppercases unquoted identifiers) and PostgreSQL (which preserves lowercase).

- [ ] **Step 2: Run the existing H2 test to verify it still passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaSequenceNumberIT -q`
Expected: all 7 tests pass. The case-insensitive regex matches the uppercased H2 identifier.

- [ ] **Step 3: Commit**

```
git add runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberIT.java
git commit -m "fix(#122): case-insensitive constraint assertion for PostgreSQL portability

Refs #122"
```

---

### Task 5: Create JpaSequenceNumberPgIT

This is the headline test — validates that `LedgerSequenceAllocator`'s `MERGE INTO` native SQL works on real PostgreSQL 15+.

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberPgIT.java`

- [ ] **Step 1: Write the PostgreSQL variant test class**

```java
package io.casehub.ledger.runtime.repository.jpa;

import io.casehub.ledger.test.PostgreSQLTestProfile;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

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

All 7 `@Test` methods from `JpaSequenceNumberIT` are inherited. No test logic duplication.

- [ ] **Step 2: Run the PostgreSQL variant**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaSequenceNumberPgIT`

Expected: all 7 tests pass. Docker must be running. First run downloads the `postgres:17-alpine` image (~80 MB). Testcontainers starts a PostgreSQL container, Flyway runs V1000–V1008 + V1999 (test-only), all tests execute against real PostgreSQL.

Watch for:
- `assignsContiguousSequenceNumbers` — validates MERGE INTO works on PostgreSQL
- `uniqueConstraintPreventsDuplicateSequenceNumber` — validates the case-insensitive assertion works on PostgreSQL
- `leafHashCoversCorrectSequenceNumber` — validates the full save pipeline (including MERGE) produces correct digests

- [ ] **Step 3: Run the original H2 test to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=JpaSequenceNumberIT -q`
Expected: all 7 tests pass (unchanged).

- [ ] **Step 4: Commit**

```
git add runtime/src/test/java/io/casehub/ledger/runtime/repository/jpa/JpaSequenceNumberPgIT.java
git commit -m "test(#122): JpaSequenceNumberPgIT — MERGE INTO validation on PostgreSQL

Validates LedgerSequenceAllocator's SQL-standard MERGE syntax against
real PostgreSQL 17. Extends JpaSequenceNumberIT — all 7 tests inherited.

Refs #122"
```

---

### Task 6: Create LedgerHealthJobPgIT

Validates complex JPQL aggregation (`GROUP BY`/`HAVING` with mixed numeric types) against the real PostgreSQL dialect.

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerHealthJobPgIT.java`

- [ ] **Step 1: Write the PostgreSQL variant test class**

```java
package io.casehub.ledger.service;

import io.casehub.ledger.test.PostgreSQLTestProfile;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

@QuarkusTest
@TestProfile(LedgerHealthJobPgIT.Profile.class)
class LedgerHealthJobPgIT extends LedgerHealthJobIT {

    public static class Profile extends PostgreSQLTestProfile {
        @Override
        public String getConfigProfile() {
            return "health-test";
        }
    }
}
```

All `@Test` methods and `@BeforeEach` reset from `LedgerHealthJobIT` are inherited. The static inner CDI beans (`GapEventCapture`, `TestReconciliationSource`) are discovered once by CDI — no conflicts.

- [ ] **Step 2: Run the PostgreSQL variant**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerHealthJobPgIT`

Expected: all tests pass. The `checkSequenceGaps()` JPQL — `SELECT e.subjectId, COUNT(e), MIN(e.sequenceNumber), MAX(e.sequenceNumber) FROM LedgerEntry e GROUP BY e.subjectId HAVING COUNT(e) != MAX(e.sequenceNumber) - MIN(e.sequenceNumber) + 1` — executes against real PostgreSQL with its native type mappings (`COUNT` → `bigint`, `MIN`/`MAX` → `integer`).

- [ ] **Step 3: Run the original H2 test to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=LedgerHealthJobIT -q`
Expected: all tests pass (unchanged).

- [ ] **Step 4: Commit**

```
git add runtime/src/test/java/io/casehub/ledger/service/LedgerHealthJobPgIT.java
git commit -m "test(#122): LedgerHealthJobPgIT — JPQL aggregation on PostgreSQL

Validates GROUP BY/HAVING with mixed numeric types against real
PostgreSQL dialect. Extends LedgerHealthJobIT — all tests inherited.

Refs #122"
```

---

### Task 7: Full test suite verification

- [ ] **Step 1: Run the complete runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`

Expected: all tests pass — both existing H2 tests and both new PostgreSQL variants. This confirms:
- No regressions in existing tests
- Both PostgreSQL tests start containers, run Flyway migrations, and pass
- No CDI conflicts from the new test resource or profile classes

- [ ] **Step 2: Run the full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

Expected: BUILD SUCCESS across all modules (api, runtime, deployment, persistence-memory).
