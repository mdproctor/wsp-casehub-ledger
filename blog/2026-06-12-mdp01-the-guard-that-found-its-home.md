---
layout: post
title: "The guard that found its home"
date: 2026-06-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [privacy, tokenisation, spi, gdpr]
---

## The guard that found its home

The issue was simple: `ActorIdentityProvider.tokenise()` runs on every `save()` and `saveAttestation()`. It pseudonymises the actor identity — replaces the raw string with a UUID token backed by the `ActorIdentity` table. GDPR Art.17 compliance: sever the mapping, the actor becomes permanently anonymous.

The problem: it tokenises everything. SYSTEM actors, AGENT actors, humans — all treated alike. SYSTEM actors are not natural persons. Neither are agents. Their actorIds are configuration strings like `"system:health-check"` or `"claude:tarkus-reviewer@v1"`. Tokenising them creates entries in the identity table that serve no privacy purpose, breaks client-side identity comparison after read-back, and forces consumers through the `tokeniseForQuery` path for non-privacy reasons.

The first spec proposed the obvious fix: add `&& entry.actorType != ActorType.SYSTEM` to the four repository call sites. It was correct, local, and wrong.

The review caught it. The tokenisation decision is a policy concern. `ActorIdentityProvider` is the abstraction that owns pseudonymisation — "should this actor be tokenised?" belongs inside the policy implementation, not scattered across every caller. casehub-work already has its own `JpaWorkItemLedgerEntryRepository.save()` — no tokenisation call today, but if it adds one, the call-site approach requires it to independently rediscover the exemption rule. Every future repository implementation carries the same risk.

The fix: change the SPI signature. `tokenise(String rawActorId)` becomes `tokenise(String rawActorId, ActorType actorType)`. `InternalActorIdentityProvider` tokenises when `actorType == HUMAN` or null. Everything else returns unchanged.

Positive selection matters here. The initial spec exempted only SYSTEM — negative exclusion. But the principled rule is: tokenise HUMAN actors, not "tokenise everything except X." If a fourth `ActorType` appears in the future, positive selection defaults to not-tokenised. Negative exclusion silently tokenises non-person actors. The default should require opting in, not opting out.

One edge case the review missed: `LedgerEntry.actor_type` is nullable in the schema. `LedgerAttestation.attestor_type` is NOT NULL. When `actorType` is null, the safe default is to tokenise — could be human, better to pseudonymise unnecessarily than miss PII.

The repository call sites got simpler. The old three-line null-guarded conditional collapsed to a single assignment — the SPI handles null internally. Four call sites, same pattern, less code.
