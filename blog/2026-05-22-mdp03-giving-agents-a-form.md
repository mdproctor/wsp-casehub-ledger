---
layout: post
title: "Giving Agents a Form"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [ledger, eidos]
tags: [agent-identity, research, design]
---

I've been assuming the agent model was complete. You have an actorId —
`"claude:reviewer@v1"` — you have trust scores, attestations, bilateral
signing. Who the agent is: solved. What kind of thing it is: not even asked.

We spent today asking it.

The research sweep covered six domains: LLM frameworks, game AI, academic MAS,
robotics, the A2A ecosystem, and current personality science applied to LLMs.
The short version is that nobody has built what I was imagining.

LLM frameworks like CrewAI and MetaGPT express role as free-form English in a
system prompt. There is no schema, no vocabulary, no discovery mechanism. A2A
Agent Cards describe what an agent *can do* — skills, input/output types — but
say nothing about what *kind* of thing it is. The closest thing is LDP
(arXiv:2603.08852, March 2026), which adds a `reasoning_profile` field.
Freeform string. Designer-assigned quality hints. And critically: LDP found that
unverified quality hints actively degrade routing quality below the
no-metadata baseline. Which is the empirical case for the attestation layer
already in casehub-ledger.

MAST (NeurIPS 2025) annotated 1,642 multi-agent execution traces. 36.9% of
failures were inter-agent misalignment. The single largest failure mode
(13.2%) was reasoning-action mismatch — an agent that says it will do X and
does Y. That's exactly what peer attestation catches, and exactly what you
can't catch from a name string alone.

The behavioral axis needed grounding. I'd been thinking in D&D alignment terms
— law/chaos, good/evil — but there's no empirical validation for that applied
to AI agents. Social Value Orientation (SVO) is the right framework:
Altruist → Prosocial → Individualist → Competitive. Validated in behavioral
economics, multi-agent RL, and now LLMs directly. Observable from allocation
and cooperation decisions. The attestation layer doesn't just store trust scores
— it validates or challenges the self-declared disposition over time.

Two design decisions sharpened under pressure.

The first: I wanted slot values as a closed enum in the platform API. I was
wrong. If a game app uses "mayor" and "polecat" and a clinical app uses
"approver" and "adjudicator", a platform enum forecloses both. The right answer
is open String with a CDI-based vocabulary registry — `Instance<Vocabulary>`
beans discovered at startup. The platform defines structure; domains define
vocabulary.

The second: I was routing `AgentDescriptor` into `casehub-platform-api`
because "other repos reference it." That's not the right test. The right test
is: would multiple peer repos independently define this type without a shared
home, creating duplication? For `ActorType` — yes, ledger, work, and qhorus
all need it and shouldn't depend on each other. For `AgentDescriptor` — no,
those repos can just depend on `casehub-eidos-api`. We added this to the
protocols.

The name took some back-and-forth. Archetype was the instinct — it covers
structured identity, party roles, the game AI heritage. Claude flagged the
Maven collision: `mvn archetype:generate` is a daily command for every Java
developer. Eidos (εἶδος) — Platonic Form, the essential template from which
instances are derived — is more precise anyway. The `AgentDescriptor` IS an
eidos in this sense: define the form, instantiate running agents from it.

The repo is bootstrapped. `casehub-eidos` on casehubio and mdproctor,
workspace on `wsp-casehub-eidos`, Phase 1 waiting.
