---
title: "The Search That Skips the Catalog"
date: 2026-09-23
author: mdp
entry_type: note
subtype: diary
series: issue-295-unified-api-generation
projects:
  - casehubio/ledger
tags: [mcp-domain, mcp-tools, discovery, api-design]
---

# The Search That Skips the Catalog

At 142 domains and 607 operations, `casehub_model` stopped being useful. An LLM
that needs to find "create work item" has to first browse 142 domain summaries,
pick one, then scan its operations. Two round trips before it can act. For a
human, that's a page of JSON to skim. For an LLM spending tokens on tool calls,
it's worse — the catalog exceeds the practical threshold for reliable selection
at around 30 items per tier.

The fix is obvious once you see it: let the LLM search instead of browse.
`casehub_search("work item")` returns the matching operations directly — domain
name, operation name, parameters, return type. No browsing. No two-tier drill-down.
Case-insensitive substring matching across operation names, summaries, parameter
names, and return types. The implementation is six methods and a record: the
registry already has all the metadata at startup, so search is an in-memory scan
with no indexing overhead.

The second problem is what happens when searching doesn't help — when the LLM
needs to explore what's available rather than find something specific. 142 domains
in a flat list is noise. The app-level grouping restructures the catalog into
three tiers: app (16 entries) → domain (~9 per app) → operation. Each tier stays
under the 30-item threshold.

The design choice that matters is how `app` gets its value. Every `@McpDomain`
class gets a new `app()` annotation attribute. When it's empty — which it is for
all 142 existing domains — the domain name becomes the app name. This means the
feature is backward-compatible by default: no consumer needs to change anything,
and existing `casehub_model()` calls return a structurally different but
informationally equivalent response. The three-tier hierarchy only kicks in when
consumers explicitly group their domains under a shared app name. Sixteen repos
need one annotation change each.

What this means in practice: an LLM that used to spend three tool calls to find
and execute an operation — browse domains, browse operations, execute — can now
do it in two: search, execute. For exploration, the three-tier catalog keeps
every selection under 30 items, which is where LLM selection reliability stays
high. The evaluation that prompted this work showed these two changes would have
the highest impact on real-world LLM usability across the platform.
