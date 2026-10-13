---
layout: post
title: "The Filter That Counts Right"
date: 2026-09-25
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [mcp, tree-navigation, casehub-activate, filtering, registry]
series: issue-351-transactional-vthread-fix
---

# The Filter That Counts Right

`casehub_activate` registers a domain's operations as individual MCP tools. Once activated, the domain's operations are standalone tools — showing them again in `casehub_model()` navigation and `casehub_search()` results is noise. The question was where to put the filter.

The obvious answer: post-filter the tree nodes in `CaseHubMcpTools` after calling `registry.listChildren()`. Quick to wire, minimal change surface.

The problem is intermediate nodes. The prefix tree groups domains by name — a path like `clinical` aggregates `clinical/trials`, `clinical/patients`, and `clinical/records` into one node showing `children: 3, totalOperations: 50`. If `clinical/trials` is activated, post-filtering removes it from the node list, but the parent's counts are already baked in. The LLM sees `children: 3` but only two children. Navigate deeper and the numbers don't add up.

I filtered at the source instead. `listChildren(String prefix, Set<String> excludedDomains)` skips excluded domains before the grouping step — they never enter the `LinkedHashMap`, never contribute to child counts, never produce empty intermediate nodes. The original single-arg method delegates with `Set.of()`. Same pattern on `search()`.

Direct navigation still works. If an LLM explicitly calls `casehub_model("test")` on an activated domain, it gets the domain detail — the filter only applies to `listChildren` and `search`, not `getDomain`. The filtering reduces noise, not access.
