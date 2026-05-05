---
layout: post
title: "Forgiveness Was a Patch"
date: 2026-04-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, trust-scoring, bayesian, privacy, gdpr, pseudonymisation]
---

The session started by verifying that two open issues (#19 and #20) were already closed — the work had landed in the previous session but the handover said they were still open. Five minutes of checking, then done.

What followed was more interesting.

## Forgiveness Was a Patch

`TrustScoreComputer` had a `ForgivenessParams` mechanism — a two-parameter softener that raised effective scores for AI agents that failed transiently. The idea was sound: pure EigenTrust punishes an agent permanently for a network timeout. Forgiveness corrected that.

But `ForgivenessParams` was bolted on top of a coarse model. The real problem was that the model classified each decision as 1.0, 0.5, or 0.0 — clean, mixed, or negative — then averaged those scores weighted by age. One positive attestation scored the same as a hundred. The model had no concept of uncertainty.

The Bayesian Beta approach eliminates the patch by fixing the underlying model. Each attestation contributes a recency-weighted increment to α (positive verdicts) or β (negative verdicts). Score = α/(α+β). Prior is Beta(1,1), which gives 0.5 with no history — maximum uncertainty. Unattested decisions contribute nothing.

An actor with one positive attestation scores 2/3. One with a hundred scores 101/102. The model knows the difference. Old negative attestations fade naturally via recency weighting on β — no explicit forgiveness mechanism needed, because the principled model already does what the patch was trying to do.

We ripped out `ForgivenessParams` entirely and replaced the algorithm. 18 unit tests, 5 integration tests. Issue #28 closed.

## The Hardest Axiom

After the trust scoring work, I turned to Axiom 7 in the auditability assessment: privacy compatibility. Six of eight axioms addressed — this was the remaining gap.

The fundamental tension: GDPR Article 17 gives data subjects the right to erasure. An immutable audit ledger cannot delete entries without breaking the Merkle hash chain. These two requirements are genuinely incompatible at first glance.

The resolution is pseudonymisation. Store tokens instead of raw actor identities. Back them with a detachable mapping. When someone requests erasure, delete the mapping row — ledger entries retain the token but the link to the real person is gone. The hash chain is intact; the personal data link isn't.

Every organisation worldwide has different legal and operational requirements for this. I didn't want to bake in one strategy. The design uses two CDI SPIs: `ActorIdentityProvider` (tokenise/resolve/erase) and `DecisionContextSanitiser` (sanitise decisionContext JSON). Default implementations are pass-through — existing consumers see zero behaviour change. Organisations that need tokenisation activate the built-in `InternalActorIdentityProvider` via config, or supply their own bean.

The wiring runs through `LedgerPrivacyProducer`, which uses `@Produces @DefaultBean @ApplicationScoped` — a Quarkus Arc pattern I hadn't used on producer methods before. It turns out a consumer-supplied `ActorIdentityProvider` bean silently replaces the producer's output without any configuration change. Clean.

`LedgerErasureService` handles GDPR Art.17 requests: find the token for the actor, count affected entries, sever the mapping, return an `ErasureResult`.

## What Claude Caught

During the integration tests for `InternalActorIdentityProvider`, one assertion failed: `resolve()` returned the actor identity after `erase()` had run. The DB row was deleted but `em.find()` returned it anyway.

Claude caught the cause immediately — bulk JPQL DML (`executeUpdate()`) bypasses Hibernate's L1 persistence context. The row was gone from the database; the cache didn't know. Fix: `em.clear()` after `executeUpdate()`. Standard JPA, easy to miss.

159 tests passing. Issues #28 and #29 closed. Axiom 7 addressed.
