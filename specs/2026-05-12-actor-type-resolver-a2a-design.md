# ActorTypeResolver — A2A Protocol Role Mappings
**Issue:** casehubio/ledger#75  
**Date:** 2026-05-12  
**Scope:** `ActorTypeResolver` only — two files changed, no schema or type changes

---

## Problem

`ActorTypeResolver.resolve("agent")` returns `ActorType.HUMAN` via the catch-all. Every A2A message with `role: "agent"` is silently misclassified as a human actor in the ledger. This corrupts EigenTrust scoring and attestation weights for A2A-sourced interactions.

`resolve("user")` coincidentally returns `HUMAN` (also via catch-all), which is correct — but the intent is invisible and fragile. Any future change to the catch-all logic could silently break the mapping.

---

## Design

### Rule insertion — Option A (after persona rule, before catch-all)

Two explicit equality checks inserted between the versioned-persona regex block and the `return ActorType.HUMAN` catch-all:

```java
// A2A protocol role: "user" → the human/initiating party in an A2A conversation
if (actorId.equals("user")) {
    return ActorType.HUMAN;
}
// A2A protocol role: "agent" → the AI agent responding in an A2A conversation
if (actorId.equals("agent")) {
    return ActorType.AGENT;
}
return ActorType.HUMAN;
```

**Why after the persona rule:** creates a clean two-cluster structure — general identity patterns (rules 1–4) then protocol-specific named strings (rules 5–6) then catch-all. Future protocol-specific string additions have a clear home. Matches the ordering in the issue spec.

**Why not before the persona rule:** `"agent:worker-1"` (Qhorus-internal namespaced ID) and `"agent"` (raw A2A role string) are semantically distinct. Grouping them would obscure that distinction.

### Javadoc update

The class Javadoc enumeration of rules gains two new entries:

```
5. A2A protocol role "user" → HUMAN (human/initiating party in an A2A conversation)
6. A2A protocol role "agent" → AGENT (AI agent responding in an A2A conversation)
7. Everything else → HUMAN (conservative default)
```

The existing rules 1–4 are renumbered 1–4; the catch-all becomes rule 7.

---

## Tests

Two new test methods added to `ActorTypeResolverTest`, in the existing comment-delimited sections:

| Method | Section | Assertion |
|--------|---------|-----------|
| `a2aRole_agent_isAgent()` | `// ── AGENT` | `resolve("agent")` → `AGENT` |
| `a2aRole_user_isHuman()` | `// ── HUMAN` | `resolve("user")` → `HUMAN` |

All nine existing test cases pass unchanged.

---

## Files Changed

| File | Change |
|------|--------|
| `api/src/main/java/io/casehub/ledger/api/model/ActorTypeResolver.java` | Two `if` blocks + Javadoc update |
| `runtime/src/test/java/io/casehub/ledger/model/ActorTypeResolverTest.java` | Two new test methods |

---

## Out of Scope

- No changes to `ActorType` enum (three values unchanged)
- No schema changes
- No Flyway migrations
- No downstream propagation (this is a bug fix to existing logic, not a new SPI method)
