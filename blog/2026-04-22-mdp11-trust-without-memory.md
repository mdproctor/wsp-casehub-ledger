---
layout: post
title: "Trust Without Memory"
date: 2026-04-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, trust-scoring, llm-agents, identity]
---

An LLM agent has no memory across sessions. Each time Claude starts, it brings
the same character — same values, same system instructions — but no recollection
of anything it did yesterday. Traditional trust scoring assumes the opposite: a
stable actor accumulating a reputation over time. I needed to reconcile those two things.

The question for `actorId` was whether to use a session ID, a configuration hash,
or something stable. Session ID obviously fails — trust resets on every start.
Configuration hash is tempting: it binds the identity precisely to what the agent
is configured to do. But memory files evolve by design. Every session update changes
the hash. Same actor, different hash — the same failure by another route.

## What the Research Said

I sent Claude off with a targeted brief: hit arxiv, NIST, W3C PROV, the EU AI Act,
and whatever the major multi-agent frameworks were doing. It came back with a synthesis
that changed how I was framing the problem.

W3C PROV-DM has a clear position: agents need stable, dereferenceable URIs — not
session identifiers, not hashes that evolve with state. NIST's AI Agent Standards
Initiative (February 2026) says the same thing through the lens of enterprise
identity: treat agents as non-human principals, give them stable machine identities,
lifecycle-manage them as you would a service account. BAID (arxiv:2512.17538) goes
further — cryptographic binding between an agent's identity and its configuration,
treating the program binary itself as the anchor.

The consensus: stable named identity is the right layer. Sessions are ephemeral.
Configuration hashes are forensic tools. The identity that trust accumulates against
should survive both.

## The Format

We settled on versioned persona names: `"{model-family}:{persona}@{major}"`. A Tarkus
code reviewer running on Claude is `"claude:tarkus-reviewer@v1"`. The session ID goes in
`correlationId` — already there for OTel trace linking. The configuration hash goes in
`ProvenanceSupplement.agentConfigHash` if the consumer wants to detect drift within a
version, but it plays no part in the trust key.

The versioning question had a specific answer I hadn't expected going in: only bump the
major version when the change warrants resetting accumulated trust. Prompt tuning, worked
examples, memory file updates — those don't warrant a reset. A complete role redefinition,
a new decision authority, a significant loosening of behavioural constraints — those do.
The forcing question is "should this agent have to earn its reputation again?" Most changes
don't clear that bar.

## The Clean Break

There's no score inheritance API. When a consumer bumps from `@v1` to `@v2`, v2 starts
at Beta(1,1) = 0.5. I considered building a weighted inheritance mechanism — carry some
fraction of v1's score into v2. The problem is that the weight is a second judgement call
with no objective ground truth. You end up with two decisions: "should we reset?" and
"how much should we carry over?" The second is harder and more arbitrary than the first.
The clean break removes it. Consumers who want to pre-seed trust can write synthetic
attestations — that leaves an explicit, auditable trail rather than an opaque multiplier.

## The Documentation Gap

I noticed the consumer-facing docs hadn't caught up once the design was settled.
`integration-guide.md` — the first place a new consumer looks — had nothing. `actorId`
was described as "who performed this action." No hint that LLM agents need special
treatment. `prov-dm-mapping.md` showed `ledger:actor/agent-007` as an example IRI.

We updated both. The integration guide got its own "Actor identity" section showing
the versioned format, the versioning semantics, and when to use `agentConfigHash`. The
PROV-DM reference now shows `ledger:actor/claude:tarkus-reviewer@v1`.

The lesson: thorough design docs don't substitute for consumer-facing examples. They
answer different questions for different readers.
