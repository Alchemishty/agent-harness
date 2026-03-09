# Context Engineering Upgrades Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Integrate context engineering best practices from the Agent-Skills-for-Context-Engineering research into agent-harness, giving every harness-managed project built-in memory management, context budget awareness, and degradation prevention.

**Architecture:** Six targeted changes across schema, references, skills, and bootstrap. All changes are additive — no existing behavior is broken. The harness gains three new conventions (memory/, scratch/, context management) and three skill improvements (plan re-reading, attention-favored placement, phase isolation guidance).

**Tech Stack:** Markdown, YAML, Bash

---

## Assumptions
- All changes go into the existing agent-harness repo at `/Volumes/Samsung T7/agent-harness/`
- No external dependencies are introduced
- The changes are conventions and documentation — no executable code beyond updating existing bash patterns
- The harness.schema.yaml additions are backward-compatible (new fields have defaults)

## Open Questions
- None — scope was confirmed by user from the analysis

## Context and Orientation
- **Files to read before starting:** All files have been read in this session
- **Patterns to follow:** Existing harness conventions — YAML schema with defaults, markdown skills with frontmatter, reference docs with `<!-- ADAPT: -->` markers
- **Related decisions:** This is a new capability addition, no prior decisions conflict

## Steps

### Task 1: Add memory/ and scratch/ to harness.schema.yaml

**Files:**
- Modify: `harness.schema.yaml:103-131` (docs section)

**Step 1: Add memory and scratch fields to the docs schema section**

Add two new fields after the `decisions` field (line 127) and before `max_doc_lines` (line 128):

```yaml
    memory:
      type: string
      default: "memory/"
      description: "Directory for cross-session agent memory. Agents write learned patterns, failure fixes, and user preferences here. Persists across sessions."
    scratch:
      type: string
      default: "scratch/"
      description: "Directory for temporary working files (tool output captures, intermediate results). Auto-cleaned between features."
```

**Step 2: Verify the schema file is valid YAML**

Run: `python3 -c "import yaml; yaml.safe_load(open('/Volumes/Samsung T7/agent-harness/harness.schema.yaml'))"`
Expected: No output (valid YAML)

**Step 3: Commit**

```bash
git add harness.schema.yaml
git commit -m "feat: add memory/ and scratch/ directory conventions to schema"
```

---

### Task 2: Create context-management-reference.md

**Files:**
- Create: `references/context-management-reference.md`

**Step 1: Write the context management reference document**

This is a new reference document that bootstrap skills will use to educate agents about context management. Content covers:

1. **Context Budget Awareness** — guidelines for how agents should manage their context window
2. **Memory Directory Convention** — what goes in memory/, file organization, what to write and not write
3. **Scratch Pad Pattern** — how to use scratch/ for capturing verbose tool output
4. **Attention-Favored Placement** — U-shaped attention curve, where to put critical info
5. **Plan Re-Reading** — why agents should re-read plans between implementation steps
6. **Phase Isolation** — when and why to use sub-agents for long-running tasks

Reference the research: information in the middle of context gets 10-40% lower recall, degradation begins ~70-80% context utilization, filesystem-based memory scored 74% on LoCoMo (beating specialized memory tools).

**Step 2: Commit**

```bash
git add references/context-management-reference.md
git commit -m "feat: add context management reference document"
```

---

### Task 3: Update doc-structure-reference.md with memory/ and scratch/

**Files:**
- Modify: `references/doc-structure-reference.md`

**Step 1: Add memory/ and scratch/ sections to the Standard Documents section**

After the `decisions/` section (around line 124), add documentation for the two new directories:

- `memory/` — Cross-session agent memory. Describe the convention: agents write learned patterns here, organized by topic (e.g., `memory/debugging.md`, `memory/patterns.md`, `memory/preferences.md`). Include what to write (confirmed patterns, recurring fixes, user preferences) and what NOT to write (session-specific state, speculative conclusions).
- `scratch/` — Temporary working files. Describe the convention: agents write verbose tool output here (test results, lint output, build logs) and reference them compactly in conversation. Auto-cleaned between features. Not committed to git.

**Step 2: Add scratch/ to the .gitignore note**

Add a note that `scratch/` should be in `.gitignore` since it contains temporary files.

**Step 3: Update the Standard Documents tree diagram**

Add `memory/` and `scratch/` to the directory tree shown on lines 13-23.

**Step 4: Commit**

```bash
git add references/doc-structure-reference.md
git commit -m "feat: add memory/ and scratch/ to doc structure reference"
```

---

### Task 4: Update AGENTS.md reference for attention-favored placement

**Files:**
- Modify: `references/agents-md-reference.md`

**Step 1: Add memory/ and scratch/ to Quick Reference table**

Add rows for memory directory and scratch directory to the Quick Reference table.

**Step 2: Restructure Agent Rules for attention-favored placement**

The U-shaped attention curve means beginning and end of a section get best recall. Reorder the 10 Agent Rules so the most critical rules are at positions 1-3 and 8-10, with less critical rules in the middle (4-7).

Current most critical rules (move to edges):
- "Never commit secrets" → Rule 1 (top)
- "Run verification before committing" → Rule 2
- "Write tests before implementation" → Rule 3
- "Escalate when stuck after 3 retries" → Rule 8
- "Check harness.yaml for commands" → Rule 9
- "Read before modifying" → Rule 10 (bottom)

Less critical (middle positions 4-7):
- "Follow docs/conventions.md"
- "Update docs when architecture changes"
- "Keep commits atomic"
- "Record non-obvious decisions"

**Step 3: Add a Context Management section**

After the Testing section and before Agent Rules, add a brief "Context Management" section that points to `memory/` and mentions the scratch pad convention. Keep it to 3-4 lines.

**Step 4: Commit**

```bash
git add references/agents-md-reference.md
git commit -m "feat: restructure AGENTS.md reference for attention-favored placement"
```

---

### Task 5: Update implement-feature.md with context engineering improvements

**Files:**
- Modify: `skills/implement-feature.md`

**Step 1: Add plan re-reading to Phase 2 loop**

In Phase 2 (Implementation Loop), between steps 2.1 and 2.2, add a new step "2.1b — Re-anchor context" that instructs the agent to re-read the plan file at the start of each step iteration. This combats context degradation during long sessions by placing the plan back at a recent, attention-favored position.

**Step 2: Add scratch pad pattern to Phase 2.6 (pre-commit gate)**

In step 2.6, when the pre-commit gate runs verification checks, add guidance to write verbose output to `scratch/last-verification.log` and only keep the summary + first 3 errors in conversation context. This prevents test/lint output from bloating the context window.

**Step 3: Add memory write-back to Phase 5 (Completion)**

In Phase 5, after moving the plan to completed, add a step to write any learned patterns to `memory/`. Specifically: if the agent encountered and fixed recurring issues during implementation, it should write them to `memory/patterns.md` or `memory/fixes.md` so future sessions benefit.

**Step 4: Add Phase Isolation guidance**

Add a note after Phase 2's description explaining that for plans with 7+ steps, the agent SHOULD consider using sub-agents for each phase to get fresh context windows. Reference the principle: "Sub-agents exist primarily to isolate context, not to anthropomorphize role division."

**Step 5: Add memory/ reading to Setup**

In the Setup section (line 19-36), add `memory/` to the list of things to read at startup: "If `memory/` exists, read its files to load cross-session learned patterns."

**Step 6: Commit**

```bash
git add skills/implement-feature.md
git commit -m "feat: add context engineering improvements to implement-feature"
```

---

### Task 6: Update bootstrap skills to create memory/ and scratch/

**Files:**
- Modify: `skills/bootstrap-greenfield.md`
- Modify: `skills/bootstrap-existing.md`

**Step 1: Update bootstrap-greenfield.md**

In Step 5 (Generate docs/ directory), add `memory/` to the directory structure to create. Add a `.gitkeep` in it.

In Step 6c (.gitignore), add `scratch/` to the .gitignore template.

In Step 4 (Generate AGENTS.md), add memory/ to the Quick Reference table generation instructions.

In Step 9 (Present summary), add `memory/` and `scratch/` to the summary listing.

Also add `HARNESS_PATH/references/context-management-reference.md` to Step 1 (Read the harness schema) reference list.

**Step 2: Update bootstrap-existing.md**

Mirror the same additions:
- Add `memory/` to docs directory creation in Step 7
- Add `scratch/` to .gitignore guidance
- Add `memory/` to AGENTS.md generation in Step 6
- Add the context management reference to Step 1 reads
- Add to Step 11 summary

**Step 3: Commit**

```bash
git add skills/bootstrap-greenfield.md skills/bootstrap-existing.md
git commit -m "feat: update bootstrap skills to create memory/ and scratch/"
```

---

### Task 7: Update garden.md to manage scratch/ cleanup

**Files:**
- Modify: `skills/garden.md`

**Step 1: Add scratch/ cleanup check**

Add a new check "6. Scratch Cleanup" that:
- Checks if `scratch/` directory exists and has files
- If running as part of post-feature cleanup: delete all files in scratch/
- Report: "Cleaned N files from scratch/"

**Step 2: Add memory/ freshness check**

Add a new check "7. Memory Freshness" that:
- If `memory/` exists, check that files in it are not stale (referenced paths still exist)
- Report any stale memory entries as "Needs Attention"

**Step 3: Commit**

```bash
git add skills/garden.md
git commit -m "feat: add scratch cleanup and memory freshness to garden"
```

---

## Validation
- All modified YAML files parse correctly: `python3 -c "import yaml; yaml.safe_load(open('harness.schema.yaml'))"`
- All markdown files are well-formed (no broken links within the repo)
- No existing skill behavior is broken — all changes are additive
- The new reference document is self-contained and referenced by bootstrap skills

## Decision Log
(Filled during implementation)
