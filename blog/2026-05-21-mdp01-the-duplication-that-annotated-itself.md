---
layout: post
title: "The duplication that annotated itself"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [ledger]
tags: [quarkus, refactoring, jpa, architecture, testing]
---

The comment was right there in `ReactiveLedgerVerificationService`:

```java
* The cryptographic verification logic ({@link #verifyCryptographic}) is duplicated
* from {@link LedgerVerificationService} pending decomposition at casehubio/ledger#93.
```

I left that during the #92 reactive tier separation knowing it was wrong and
deferring it. This session was #93.

The fix sounds obvious: extract the duplicated `verifyCryptographic` method into a
shared utility, then pull the signature verification pipeline out of
`LedgerVerificationService` into its own bean. The extraction pattern already exists
— `LedgerMerkleTree` is a package-private `final` static class, no CDI, pure Java,
called by anything in the same package that needs Merkle operations. We did the same
for Ed25519: `AgentCryptographicVerifier`, same shape, same access rules.

`LedgerVerificationService` is now three methods: `treeRoot`, `inclusionProof`,
`verify`. Merkle operations, nothing else. The two callers —
`LedgerComplianceReportService` and `LedgerRetentionJob` — never called
`verifyAgentSignature` anyway. The old bean was just carrying the method because
no one had given it a proper home yet.

The result landed with a symmetry I wasn't expecting. The blocking/reactive
pairing is now clean:

- `AgentSignatureVerificationService` ↔ `ReactiveAgentSignatureVerificationService`
- `KeyRotationService` ↔ `ReactiveKeyRotationService`

`LedgerVerificationService` has no reactive counterpart, and it shouldn't — Merkle
verification requires reading the full ordered subject history to recompute the
frontier. You can't do that reactively without sequential consistency guarantees.
The architecture just tells you.

One test isolation gotcha worth noting. If you seed entries in a `@QuarkusTest`
without mocking `AgentKeyProvider`, your tests pass because no signing key is
configured in `application.properties`. `AgentSignatureEnricher` runs at
`@PrePersist` and silently does nothing. The moment someone adds a test signing key
for an unrelated test — the entries you expected unsigned are now signed, and
SUSPECT-path tests start failing with wrong keyRefs and no clear error. Claude
caught this in the code quality review of the reactive IT; the fix is a
`@BeforeEach` that mocks the provider to return empty for all actors and overrides
only the ones that need a real key.

The comment that pointed at itself is gone. 456 tests, clean build.
