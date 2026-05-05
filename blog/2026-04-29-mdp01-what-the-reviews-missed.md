---
layout: post
title: "What the Reviews Missed"
date: 2026-04-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [trust, agentic, code-review, quarkus]
---

We shipped a lot today. The prerequisites I'd been deferring are done — the
`LedgerEntryEnricher` pipeline (#67), the `ActorTrustScore` discriminator model
(#68), trust decay acceleration with a proper `DecayFunction` SPI (#55),
`TrustGateService` (#54), and `ActorTypeResolver` propagated across all four
consumer repos (#53). The push went out at the end of the session with 28 new
commits.

The technical work was mostly straightforward. What was interesting was
everything that surfaced around it.

## The sentinel that wasn't needed

When I was designing the `actor_trust_score` discriminator model, the
`scope_key` column needs to be nullable — `NULL` for GLOBAL rows, a capability
tag for CAPABILITY rows. Enforcing uniqueness on a nullable column in SQL is a
known headache: most databases treat `NULL != NULL` in unique constraints, so
two GLOBAL rows for the same actor would silently bypass the constraint.

My first instinct was a sentinel value — empty string `''` for GLOBAL scope
keys. It works, it's portable, it's been the standard workaround for years.

But before writing the SQL, I checked what H2 version Quarkus 3.32.2 pulls.
The answer: H2 2.4.240. PostgreSQL 15 introduced `UNIQUE NULLS NOT DISTINCT`
in 2022. H2 added support for it too, with a bug that was fixed in H2 2.3.230
(July 2024). The Quarkus BOM is already past that threshold.

So the migration ended up with:

```sql
CONSTRAINT uq_actor_trust_score_key UNIQUE NULLS NOT DISTINCT (actor_id, score_type, scope_key)
```

No sentinel. NULL means "no scope" everywhere it appears, not just in the
application code. The sentinel approach would have been a lie that survived in
the schema indefinitely; the NULLS NOT DISTINCT approach says what it means.

The lesson isn't complicated: check the library version before reaching for a
workaround. The workaround might have been obsolete for two years.

## Decay as signal

The confidence field on `LedgerAttestation` has been stored since the beginning
but contributed nothing to the trust computation — every attestation added its
full recency weight regardless of stated confidence. We wired it in early in
the session: `weight = recencyWeight × clamp(confidence, 0, 1)`.

The decay issue is more interesting. Trust decay was symmetric — a FLAGGED
attestation from yesterday decayed at exactly the same rate as a SOUND one from
yesterday. Only time distinguished them.

That's wrong for a security context. A code reviewer who approves a PR that
ships a vulnerability should carry that signal longer than the passage of time
would suggest. A sustained run of SOUND attestations should be required to
recover trust, not just waiting out the half-life.

The fix is a valence multiplier — FLAGGED and CHALLENGED attestations decay at
`flaggedPersistenceMultiplier × recencyWeight`, defaulting to 0.5. Half the
decay rate means twice the persistence. A failure three months ago hits about
as hard as a failure six weeks ago did before.

We extracted the decay logic into a `DecayFunction` SPI at the same time,
rather than adding another magic constant to `TrustScoreComputer`. The SPI
means alternative decay strategies — linear, step, no-decay for test suites —
can plug in as `@Alternative` CDI beans without touching the algorithm. It
costs almost nothing extra when you're already inside the file.

## What got dismissed

The more unsettling discovery was process, not code.

In a long agentic session, the controller (Claude) processes volumes of review
output the user never reads. Code reviewers flag findings as Critical,
Important, or Suggestion. The controller decides what to fix and what to skip.

During this session two findings from the code reviewer were labeled Important
and then dismissed with reasoning like "minor, not blocking":

- casehub-qhorus had test fixtures using `"agent-a"` as actor IDs throughout.
  After introducing `ActorTypeResolver`, these now resolve to HUMAN, not AGENT.
  Only the one test that explicitly asserted `actorType == AGENT` was updated.
  The rest — representing agent actors with an ID that no longer classifies as
  an agent — were left.

- claudony was calling `ActorTypeResolver.resolve(event.actorId())` on the raw
  (potentially null) event value, while the adjacent line coalesced it to
  `"system"` before storing. Both produce SYSTEM when null, but they derive
  from different sources. Any future field added to the same block would have
  to remember to coalesce independently.

Both were real correctness issues. I caught them — but only because the
dismissal appeared in passing text I happened to read.

The pattern is structural: the user assumes Important findings were handled; the
controller assumes they're minor enough to skip; no one is wrong in isolation.
The asymmetry creates invisible quality debt that only surfaces by accident.

The fix I've put in memory is blunt: Important findings reach the user or they
get fixed. The controller doesn't get to make that call unilaterally.

## The audit that found more

After shipping the `ActorTypeResolver` changes across four consumer repos, we
ran a Step 6 sweep — parallel subagents reading the recently-changed files and
running tests. It surfaced 8 pre-existing test failures in casehub-work
(`TrustScoreComputerTest` expects `score == 1.0` for unattested decisions; the
Bayesian Beta model correctly returns `0.5`), and two production code issues in
claudony we hadn't touched.

None of those were visible from casehub-ledger's perspective. The lesson is
that Step 6 means more than grepping for the class you renamed — it means
actually running the consumer tests, reading the consumer code, and confirming
nothing pre-existing is already on fire.

We tracked all of it in issue #72 because those repos are mid-session with
other Claude instances and can't be patched now.
