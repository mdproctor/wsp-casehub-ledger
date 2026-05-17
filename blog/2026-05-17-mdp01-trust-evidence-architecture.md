---
layout: post
title: "The architecture of trust evidence"
date: 2026-05-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [trust, signing, cdi, reactive]
---

Two design decisions from this week's ledger work keep rattling around, and I want to write them down while they're fresh.

The first is about key identity. When we shipped bilateral entry signing, each `LedgerEntry` got the agent's Ed25519 public key bytes stored alongside the signature. Self-contained, verifiable in perpetuity. But there was no way to say "this entry was signed by key generation X" — just raw bytes with no identity. That gap matters for the compromise case: if an agent's key is leaked, you need to know *which entries are now suspect*.

The obvious answer is to configure a key ID per actor. We rejected it. Two deployments that both use the same agent persona but configure different key names would produce incompatible audit trails. The better answer is to *derive* the key ID from the key itself: `Base64URL(SHA-256(publicKey.getEncoded()))`. Any party with the public key can verify the keyRef independently, with no operator coordination, no configuration, and zero chance of mismatch. This is what Sigstore does, and the reasoning holds: identity should be a property of the thing, not a label you attach to it.

The second decision is about what *SUSPECT* means. When a key is reported compromised, entries signed by that key during the compromised window become suspect. The question was: should the verification result be `INVALID`? It shouldn't. The signature is cryptographically valid — the entry *was* produced by that actor at that time. The content is intact. What's in question is whether the actor was operating with integrity. `INVALID` says "tamper detected." `SUSPECT` says "genuine, but the signing authority is in question." That distinction matters in a compliance audit — they're different verdicts.

The second story this week is smaller but felt important. I wanted a CDI event to fire when signature verification returns SUSPECT, so consumers could do real-time alerting without polling. We built `AgentSignatureSuspectEvent` — same pattern as `LedgerGapDetected`: plain record, `Event<T>` injection, no delivery coupling. Consumers choose `@Observes` or `@ObservesAsync` independently.

The interesting part came when we added the reactive twin. `verifyAgentSignature()` was blocking; there was no reactive version. Claude pointed out the asymmetry before I did, and I realised it wasn't an oversight — it was a pattern we'd never formalised. The ledger has always supported both stacks (`LedgerEntryRepository` blocking, `ReactiveLedgerEntryRepository` for Mutiny consumers), but there was no written rule saying new service methods had to maintain that parity.

We added the rule. A new method on any ledger service must ship both blocking and reactive variants unless one is demonstrably unsuitable. The `fireAsync()` call for the reactive path hit a type wrinkle: `event.fireAsync()` returns `CompletionStage<Event<T>>`, not `CompletionStage<Void>`. You need `.replaceWith()` to extract the value you actually want. Subtle enough that it's worth noting.

The other thing that surfaced: at epic close, the design journal was empty. We'd committed directly via subagents without going through the journal-writing hook, and the tooling silently skipped the merge. That's the wrong default. The skill now surfaces a `[W]rite / [S]kip` prompt instead — losing design reasoning permanently is a worse outcome than interrupting the close flow. We wrote the retrospective entry, merged it, and fixed the tooling before finishing.
