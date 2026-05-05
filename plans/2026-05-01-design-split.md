# DESIGN.md Split Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split `docs/DESIGN.md` into stable-structure and capabilities files before Group B adds more trust content.

**Architecture:** Move four algorithm-heavy sections (Merkle MMR, PROV-DM, Agent Identity, Agent Mesh) verbatim into a new `docs/DESIGN-capabilities.md`. Add a Further Reading block to `DESIGN.md` linking to the new file. No content is rewritten.

**Tech Stack:** Markdown files only — no code changes.

---

## File Map

| Action | File | Change |
|---|---|---|
| Modify | `docs/DESIGN.md` | Remove lines 164–381 (4 sections); insert Further Reading block after line 35 |
| Create | `docs/DESIGN-capabilities.md` | New file: header + 4 moved sections |

---

### Task 1: Create DESIGN-capabilities.md

**Files:**
- Create: `docs/DESIGN-capabilities.md`

- [ ] **Step 1: Create the file with this exact content**

```markdown
# CaseHub Ledger — Capabilities Design

> Part of the casehub-ledger design documentation. See [`DESIGN.md`](DESIGN.md) for
> entity model, architecture, SPI contracts, and configuration.

```

Then append the four sections extracted verbatim from `docs/DESIGN.md`:
- `## Merkle Mountain Range` (currently line 164) through the line before `## W3C PROV-DM JSON-LD Export`
- `## W3C PROV-DM JSON-LD Export` (currently line 180) through the line before `## Agent Identity Model`
- `## Agent Identity Model` (currently line 241) through the line before `## Agent Mesh Topology`
- `## Agent Mesh Topology` (currently line 341) through the line before `## Configuration` (line 381)

The simplest way to do this:

```bash
# Write header
cat > docs/DESIGN-capabilities.md << 'EOF'
# CaseHub Ledger — Capabilities Design

> Part of the casehub-ledger design documentation. See [`DESIGN.md`](DESIGN.md) for
> entity model, architecture, SPI contracts, and configuration.

EOF

# Append the four sections (lines 164–381 of current DESIGN.md)
sed -n '164,381p' docs/DESIGN.md >> docs/DESIGN-capabilities.md
```

- [ ] **Step 2: Verify the new file has all four section headers**

```bash
grep "^## " docs/DESIGN-capabilities.md
```

Expected output (exactly):
```
## Merkle Mountain Range
## W3C PROV-DM JSON-LD Export
## Agent Identity Model
## Agent Mesh Topology
```

- [ ] **Step 3: Verify line count is reasonable**

```bash
wc -l docs/DESIGN-capabilities.md
```

Expected: approximately 222 lines (218 extracted + 4 header lines).

---

### Task 2: Update DESIGN.md — remove the four sections

**Files:**
- Modify: `docs/DESIGN.md`

- [ ] **Step 1: Delete lines 164–381 (the four sections now in DESIGN-capabilities.md)**

```bash
sed -i '' '164,381d' docs/DESIGN.md
```

- [ ] **Step 2: Verify the four sections are gone and Configuration follows Key Design Decisions**

```bash
grep "^## " docs/DESIGN.md
```

Expected output:
```
## Purpose
## Ecosystem Context
## Architecture
## Supplements
## Key Design Decisions
## Configuration
## What Is Deliberately Out of Scope
## Roadmap
## Implementation Tracker
```

- [ ] **Step 3: Verify line count dropped by roughly 218**

```bash
wc -l docs/DESIGN.md
```

Expected: approximately 285 lines.

---

### Task 3: Update DESIGN.md — insert Further Reading block

**Files:**
- Modify: `docs/DESIGN.md`

The Ecosystem Context section ends at line 35 (the `---` separator before `## Architecture`). After the deletion in Task 2, that separator is still at the same relative position — verify with:

```bash
grep -n "^## Architecture" docs/DESIGN.md
```

Note the line number returned (call it L). The `---` separator immediately before it is at L-1. Insert the Further Reading block before that `---`.

- [ ] **Step 1: Find the line number of `## Architecture`**

```bash
grep -n "^## Architecture" docs/DESIGN.md
```

Note the line number. The `---` separator sits at that line minus 1.

- [ ] **Step 2: Insert the Further Reading block**

Using the line number from Step 1 (example: if `## Architecture` is at line 20, insert before line 19):

```bash
# Replace the --- separator before ## Architecture with the Further Reading block + separator
# Find the exact separator line:
ARCH_LINE=$(grep -n "^## Architecture" docs/DESIGN.md | cut -d: -f1)
SEP_LINE=$((ARCH_LINE - 1))

# Insert the Further Reading block before the separator
sed -i '' "${SEP_LINE}i\\
\\
## Further Reading\\
\\
| Document | What it covers |\\
|---|---|\\
| [\`DESIGN-capabilities.md\`](DESIGN-capabilities.md) | Merkle Mountain Range, W3C PROV-DM JSON-LD export, agent identity model, agent mesh topology |\\
" docs/DESIGN.md
```

- [ ] **Step 3: Verify the Further Reading section is in place**

```bash
grep -n "Further Reading\|DESIGN-capabilities" docs/DESIGN.md
```

Expected: two lines — one for the heading, one for the table row — both before the `## Architecture` line.

- [ ] **Step 4: Verify final section order in DESIGN.md**

```bash
grep "^## " docs/DESIGN.md
```

Expected:
```
## Purpose
## Ecosystem Context
## Further Reading
## Architecture
## Supplements
## Key Design Decisions
## Configuration
## What Is Deliberately Out of Scope
## Roadmap
## Implementation Tracker
```

---

### Task 4: Final verification and commit

**Files:**
- No new changes — verification only, then commit.

- [ ] **Step 1: Check DESIGN-capabilities.md opens cleanly (spot-check)**

```bash
head -10 docs/DESIGN-capabilities.md
tail -5 docs/DESIGN-capabilities.md
```

Expected head:
```
# CaseHub Ledger — Capabilities Design

> Part of the casehub-ledger design documentation. See [`DESIGN.md`](DESIGN.md) for
> entity model, architecture, SPI contracts, and configuration.
```

Expected tail: last few lines of the Agent Mesh Topology section (ends before `## Configuration`).

- [ ] **Step 2: Check no stale references remain in DESIGN.md**

```bash
grep -n "Merkle\|PROV-DM\|Agent Identity\|Agent Mesh\|EigenTrust\|tlog-checkpoint\|actorId format" docs/DESIGN.md
```

Expected: zero matches (these terms only appear in DESIGN-capabilities.md now). If any appear, they are likely legitimate cross-references in the Roadmap or Tracker — review manually and leave them if they refer to the capability rather than define it.

- [ ] **Step 3: Commit**

```bash
git add docs/DESIGN.md docs/DESIGN-capabilities.md
git commit -m "docs: split DESIGN.md into core and capabilities files"
```
