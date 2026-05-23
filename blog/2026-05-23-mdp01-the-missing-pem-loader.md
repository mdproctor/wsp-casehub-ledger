---
layout: post
title: "The Missing PEM Loader"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: log
projects: [ledger]
tags: [pqc, archunit, platform-api, evaluation]
---

The smallest task — evaluate whether `LedgerTraceIdProvider` should move to `casehub-platform-api` — looked like a clear yes. Multiple peer repos use the interface. It's named after the ledger but the concern is cross-cutting. The platform-api-scope rule says a type belongs there when multiple repos need it and can't share it through a single domain `*-api` module. All the criteria were met.

Then the question: what's the point of moving interfaces to platform if the consuming repo already has `casehub-ledger-runtime` as a compile dependency?

That's the whole thing. Moving `LedgerTraceIdProvider` reduces one import statement and changes nothing about what's on the classpath. The coupling was never what it looked like. Closed won't-do.

---

Claude and I tightened the existing parity check next — it verified reactive methods return `Uni<T>` but not what T is. `Uni<Void>` where `Uni<KeyRotationEntry>` was expected would pass silently. We added `checkUniTypeArg()` to extract the type argument erasure and compare it against the blocking return type.

Claude flagged a dead branch during review. The first draft guarded against `typeArgs.isEmpty()` after a successful `instanceof JavaParameterizedType` check — the natural defensive move. But ArchUnit's builder has `checkArgument(typeArguments.size() > 0)` baked in; no `JavaParameterizedType` can have an empty argument list. Raw `Uni` is represented as `JavaClass`, not `JavaParameterizedType`. The guard was unreachable and misleading.

---

The post-quantum work was the most substantial. Three places in ledger hardcoded `"Ed25519"` as a string literal — `AgentSignatureEnricher`, `AgentCryptographicVerifier`, and `LedgerMerklePublisher`. We replaced all three: `privateKey.getAlgorithm()` for signing, trial-load through a supported-algorithm list for verification. ADR 0013 documents the migration path. The timing trigger: adopt ML-DSA once BouncyCastle ≥ 1.79 ships in the Quarkus BOM, at which point ML-DSA support activates without further code changes.

The ADR claimed operators would need no code changes to adopt ML-DSA — generate new keys, record a rotation, done.

Claude caught `LedgerPemUtil`. It reads PEM files for the per-actorId signing keys and the Merkle checkpoint private key, and it also hardcoded `"Ed25519"`. An operator who configured an ML-DSA PEM file after adopting BouncyCastle would get an `InvalidKeySpecException` at key-load time, before any signing code was reached. The ADR's core claim was false without this fix.

We applied the same trial-load approach to `LedgerPemUtil`. Four files changed instead of three. The claim became true.

---

The evaluation on `LedgerTraceIdProvider` would have generated a platform migration and changes across three repos. Claude's review on the signing work caught a gap that would have made the ADR's key promise false on first use.

The code that doesn't ship and the sentence that gets corrected are both doing real work.
