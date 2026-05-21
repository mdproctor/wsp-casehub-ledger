---
layout: post
title: "The step that kept skipping"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [ledger]
tags: [tooling, work-end, publish-blog, skill-improvement]
---

After closing qhorus#179, the blog entry I'd just written was sitting on workspace main and not on mdproctor.github.io. The fix was three lines — copy, commit, push. But the question was: why did that keep happening?

The `work-end` skill has a Step 8g that's supposed to invoke `publish-blog` automatically when `BLOG_COUNT > 0`. It's marked "mandatory" and "not optional". It had clearly not run.

I read the skill exhaustively and found six separate bypass vectors:

1. `publish-blog` was positioned as a separate step (8g) *after* issue close (8f). Once the issue is closed, there's cognitive pressure to wrap up — 8g gets absorbed into the general "done" feeling.

2. The close plan showed the blog entry in the *artifact routing* section ("blog → workspace main"), which made it look handled. Publishing and promoting are different operations, but they look similar in a summary.

3. The "Publish blog" line in the plan template said "omit if BLOG_COUNT=0". So the line could simply be absent with no visual anomaly.

4. The step-path description (step-by-step mode) called publish-blog an "offer" in Phase 3. Contradicted the mandatory language in 8g.

5. No pre-execution acknowledgement when BLOG_COUNT > 0 — no forced confirmation before running "all".

6. Step 8h's self-check fires too late: it asks Claude to notice its own omission in a report Claude just generated.

The fix was structural: fold publish-blog into 8a, where workspace main is already active and the blog entries have just been promoted. Running it there — immediately after `git push`, before switching back to the epic branch — means there's no separate step to forget. The blog goes straight from "just landed on main" to "published". The plan template always shows the Publish blog line now (never omitted, says "skipped (no entries)" when empty), and a warning fires before execution if BLOG_COUNT > 0.

After fixing the skill, I ran the cross-branch audit. Not just current checkouts — all branches of all workspace repos, scanned with `git for-each-ref + git ls-tree`. Found 30 entries that had never reached mdproctor.github.io: 27 from the rolling `work-end` skip bug and 3 from open epic branches that hadn't gone through close at all.

The audit script is three dozen lines of Python. The fix to the skill is about 50 lines of changes. Together they resolve a gap that had been silently accumulating since mid-May.

The IntelliJ VFS revert was a separate annoyance — during the squash rebase, IntelliJ detected the repository state change and fired a VFS refresh that overwrote working-tree edits with its cached pre-rebase state. No error, no notification, build still passed. `git diff HEAD` catches it, but only if you remember to check.
