# ActorTypeResolver A2A Role Mappings Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix `ActorTypeResolver.resolve("agent")` returning `HUMAN` (wrong) and make `resolve("user")` → `HUMAN` explicit rather than relying on the catch-all.

**Architecture:** Insert two explicit equality checks (rules 5 and 6) between the versioned-persona regex block and the `return ActorType.HUMAN` catch-all. Update class Javadoc to list all 7 rules. Add two tests (TDD order: tests first).

**Tech Stack:** Java 21, JUnit 5, AssertJ. No Quarkus context needed — `ActorTypeResolver` is a pure static utility.

---

## Files

| Action | Path |
|--------|------|
| Modify (tests) | `runtime/src/test/java/io/casehub/ledger/model/ActorTypeResolverTest.java` |
| Modify (impl) | `api/src/main/java/io/casehub/ledger/api/model/ActorTypeResolver.java` |

---

### Task 1: Write failing tests

**Files:**
- Modify: `runtime/src/test/java/io/casehub/ledger/model/ActorTypeResolverTest.java`

- [ ] **Step 1: Add the two new test methods**

Add `a2aRole_agent_isAgent()` in the `// ── AGENT` section and `a2aRole_user_isHuman()` in the `// ── HUMAN` section. Final file:

```java
package io.casehub.ledger.model;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.ActorType;
import io.casehub.ledger.api.model.ActorTypeResolver;

class ActorTypeResolverTest {

    // ── AGENT ─────────────────────────────────────────────────────────────────

    @Test
    void versionedPersona_simpleVersion_isAgent() {
        assertThat(ActorTypeResolver.resolve("claude:analyst@v1")).isEqualTo(ActorType.AGENT);
    }

    @Test
    void versionedPersona_semver_isAgent() {
        assertThat(ActorTypeResolver.resolve("claude:analyst@v1.2.3")).isEqualTo(ActorType.AGENT);
    }

    @Test
    void agentPrefix_isAgent() {
        assertThat(ActorTypeResolver.resolve("agent:worker-1")).isEqualTo(ActorType.AGENT);
    }

    @Test
    void a2aRole_agent_isAgent() {
        assertThat(ActorTypeResolver.resolve("agent")).isEqualTo(ActorType.AGENT);
    }

    // ── SYSTEM ────────────────────────────────────────────────────────────────

    @Test
    void exactSystem_isSystem() {
        assertThat(ActorTypeResolver.resolve("system")).isEqualTo(ActorType.SYSTEM);
    }

    @Test
    void systemColon_isSystem() {
        assertThat(ActorTypeResolver.resolve("system:scheduler")).isEqualTo(ActorType.SYSTEM);
    }

    @Test
    void nullActorId_isSystem() {
        assertThat(ActorTypeResolver.resolve(null)).isEqualTo(ActorType.SYSTEM);
    }

    @Test
    void blankActorId_isSystem() {
        assertThat(ActorTypeResolver.resolve("")).isEqualTo(ActorType.SYSTEM);
    }

    // ── HUMAN ─────────────────────────────────────────────────────────────────

    @Test
    void plainUsername_isHuman() {
        assertThat(ActorTypeResolver.resolve("alice")).isEqualTo(ActorType.HUMAN);
    }

    @Test
    void emailAddress_isHuman() {
        assertThat(ActorTypeResolver.resolve("alice@example.com")).isEqualTo(ActorType.HUMAN);
    }

    @Test
    void a2aRole_user_isHuman() {
        assertThat(ActorTypeResolver.resolve("user")).isEqualTo(ActorType.HUMAN);
    }
}
```

- [ ] **Step 2: Run the tests — confirm exactly one failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ActorTypeResolverTest
```

Expected: 10 tests run, 1 failure — `a2aRole_agent_isAgent` fails with `expected: AGENT but was: HUMAN`. The `a2aRole_user_isHuman` test passes (catch-all returns HUMAN, which is coincidentally correct).

---

### Task 2: Implement the fix

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/model/ActorTypeResolver.java`

- [ ] **Step 1: Update ActorTypeResolver — Javadoc and two new rules**

Replace the entire file:

```java
package io.casehub.ledger.api.model;

/**
 * Canonical utility for deriving an {@link ActorType} from an actor ID string.
 *
 * <p>
 * The resolution rules, in priority order:
 * <ol>
 * <li>{@code null} or blank → {@link ActorType#SYSTEM} (safe default)</li>
 * <li>{@code "system"} or {@code "system:*"} → {@link ActorType#SYSTEM}</li>
 * <li>{@code "agent:*"} → {@link ActorType#AGENT}</li>
 * <li>Versioned persona format {@code word:word@version} (e.g. {@code "claude:analyst@v1"}) → {@link ActorType#AGENT}</li>
 * <li>A2A protocol role {@code "user"} → {@link ActorType#HUMAN} (human/initiating party in an A2A conversation)</li>
 * <li>A2A protocol role {@code "agent"} → {@link ActorType#AGENT} (AI agent responding in an A2A conversation)</li>
 * <li>Everything else → {@link ActorType#HUMAN} (conservative default)</li>
 * </ol>
 *
 * <p>
 * All consumers that derive {@link ActorType} from an actor ID string must use this class
 * to ensure consistent classification across the casehubio ecosystem.
 */
public final class ActorTypeResolver {

    private ActorTypeResolver() {
    }

    /**
     * Resolves the {@link ActorType} for the given actor ID.
     *
     * @param actorId the actor identifier, may be {@code null}
     * @return the resolved {@link ActorType}, never {@code null}
     */
    public static ActorType resolve(String actorId) {
        if (actorId == null || actorId.isBlank()) {
            return ActorType.SYSTEM;
        }
        if (actorId.equals("system") || actorId.startsWith("system:")) {
            return ActorType.SYSTEM;
        }
        if (actorId.startsWith("agent:")) {
            return ActorType.AGENT;
        }
        // Versioned persona format: word:word@word — e.g. "claude:analyst@v1"
        if (actorId.matches("[\\w-]+:[\\w-]+@[\\w.]+")) {
            return ActorType.AGENT;
        }
        // A2A protocol role: "user" → the human/initiating party in an A2A conversation
        if (actorId.equals("user")) {
            return ActorType.HUMAN;
        }
        // A2A protocol role: "agent" → the AI agent responding in an A2A conversation
        if (actorId.equals("agent")) {
            return ActorType.AGENT;
        }
        return ActorType.HUMAN;
    }
}
```

- [ ] **Step 2: Run the full test class — confirm all 11 pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ActorTypeResolverTest
```

Expected: 11 tests, 0 failures.

- [ ] **Step 3: Run the full ledger build to confirm nothing else broke**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/ledger/pom.xml
```

Expected: BUILD SUCCESS, 0 failures.

---

### Task 3: Commit

- [ ] **Step 1: Stage and commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  api/src/main/java/io/casehub/ledger/api/model/ActorTypeResolver.java \
  runtime/src/test/java/io/casehub/ledger/model/ActorTypeResolverTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "fix: extend ActorTypeResolver with explicit A2A protocol role mappings

'agent' → AGENT (was silently HUMAN via catch-all — bug).
'user' → HUMAN (now explicit, was coincidentally correct via catch-all).

Closes #75"
```
