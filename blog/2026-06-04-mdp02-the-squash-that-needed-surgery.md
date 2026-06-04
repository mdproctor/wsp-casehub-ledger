---
layout: post
title: "The Squash That Needed Surgery"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [git, history-rewrite, squash, rebase, arc42, documentation]
excerpt: "A simple history squash revealed a hidden merge commit, the non-obvious patch-id detection behind 'previously applied', and a documentation audit that should have happened before the write."
---

The request was simple: squash all entries since the last squash. The same operation that's been running since April. The first failure was expected — a merge commit mid-history that `git rebase -i` can't pick directly. We aborted and decided to linearise it first.

Linearising a merge commit is simpler than it sounds when the merge was a fast-forward. If `tree(feature-tip) == tree(merge-commit)` — and you can verify this with `git diff` — then cherry-picking all post-merge commits onto the feature branch tip produces zero conflicts. The tree state is identical. We confirmed this, created the working branch, cherry-picked 210 commits, and got a linear history that was tree-identical to the original.

Then the squash hit its second wall.

## What "previously applied" actually means

The rebase printed hundreds of "skipped previously applied commit" warnings and then conflicted on the first remaining commit. The symptom looks like the rebase is being smart. It isn't.

Git computes a patch-id — a SHA of the diff — for every commit being rebased. It then checks whether the same patch-id already appears anywhere reachable from the upstream. Our backup branch is the pre-squash state, so it contains the original commits. The squashed commits on the working branch have identical content — same diffs, different SHAs. Git declares them "already applied" and skips them. The first unskipped commit then tries to apply to a base that already contains its context, and the conflict cascade begins.

Two changes are required together. First, use `git merge-base` to find the true common ancestor of the backup and the squashed branch — not the backup tip, which contains the wrong base for this purpose. Second, pass `--reapply-cherry-picks` to disable the patch-id detection entirely. Either one alone fails: using the merge-base without the flag still skips commits; using the flag without the correct base still conflicts.

```bash
MB=$(git merge-base backup/pre-squash-main-20260531 main)
GIT_SEQUENCE_EDITOR="cp /tmp/rebase_todo.txt" \
  git rebase -i --reapply-cherry-picks "$MB"
```

After that, 343 commits became 257. The history is linear. Both remotes accepted the force-push. CI passed first time.

## The review that should have come first

While the git work was in progress, I ran the exhaustive source review of ARC42STORIES.MD that should have been the starting point when I wrote it. A separate Claude read all 40 workspace blogs, `docs/DESIGN.md`, `docs/DESIGN-capabilities.md`, all 17 ADRs, the design specs, and the integration guide — the material I'd been too selective about.

The audit came back with 11 factual errors and 15 significant omissions. Among the errors: wrong field names on `LedgerMerkleFrontier` (`height`/`hashValue` instead of `level`/`hash`), the wrong module path for `OutcomeRecord` (it's in `api/`, not `runtime/service/`), TrustScoreJob described as three-pass when it runs four, and the minimum Flyway version for consumer tables quoted as V1002 when the integration guide says V1004. All avoidable with the right sources.

The omissions tell the same story from a different angle: CAPABILITY_DIMENSION scoring never explained, the reactive tier separation pattern never documented, the `@ProvenanceCapture` interceptor not mentioned, four ADRs missing from §10.

I'd read 16 of 40 blogs and assumed CLAUDE.md covered the rest. It doesn't. CLAUDE.md is a navigation aid, not a design specification. The protocol is written now.
