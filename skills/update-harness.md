---
name: update-harness
description: Update an existing harness installation with latest skills, references, and conventions from the harness repository without touching project-specific config or docs.
---

# Update Harness

Pull the latest harness skills and conventions into a project that was previously set up with `/install-harness`. This updates harness-owned files (skills, agents) while preserving project-owned files (config, docs, enforcement, memory).

**Core principle:** Skills are harness-owned — replace wholesale. Config and docs are project-owned — merge carefully or skip.

---

## Step 1: Locate the harness repository

Use the same detection logic as `/install-harness`:

1. Check if a sibling directory `../agent-harness/` exists relative to the current working directory.
2. Check if `~/agent-harness/` exists.
3. Check if the environment variable `AGENT_HARNESS_PATH` is set and points to a valid directory.

If any are detected, present the path and ask the user to confirm or provide an alternative.

Store the confirmed path as `HARNESS_PATH`.

---

## Step 2: Verify both sides

### 2a: Verify the harness repository

Check that `HARNESS_PATH` contains:
- `HARNESS_PATH/skills/` directory
- `HARNESS_PATH/references/` directory
- `HARNESS_PATH/enforcement/` directory
- `HARNESS_PATH/hooks/` directory
- `HARNESS_PATH/harness.schema.yaml` file

If anything is missing, report and stop.

### 2b: Verify the project has an existing installation

Check that the current project has:
- `harness.yaml` in the project root
- `.claude/commands/` directory with at least one skill file
- `AGENTS.md` in the project root

If these are missing, this project was not set up with the harness. Tell the user:

> This project does not appear to have an existing harness installation. Run `/install-harness` instead.

Stop. Do not proceed.

---

## Step 3: Diff skills

Compare skills in `HARNESS_PATH/skills/` against `.claude/commands/` and `.claude/agents/`.

Exclude bootstrap-only skills that are never copied to projects:
- `install-harness.md`
- `bootstrap-greenfield.md`
- `bootstrap-existing.md`
- `update-harness.md`

For each remaining skill in the harness:

1. **New** — exists in harness but not in the project
2. **Modified** — exists in both, but content differs (compare file contents)
3. **Unchanged** — exists in both with identical content

Special handling for `project-structure-validator.md`: compare against `.claude/agents/` instead of `.claude/commands/`.

Present the diff summary:

```
## Harness Update Summary

### New skills (will be added):
- retrospective.md — Post-implementation and post-review learning capture

### Modified skills (will be updated):
- implement-feature.md — [briefly describe what changed, e.g., "added context engineering: plan re-reading, observation masking, decision probes"]
- garden.md — [brief description]
- create-plan.md — [brief description]

### Unchanged skills:
- create-pr.md
- create-tests.md
- deslop.md
- doc-split.md
- create-qa.md

### Project-owned files (will NOT be touched):
- harness.yaml
- AGENTS.md
- docs/*
- enforcement/*
- memory/*

Proceed with update? (Yes / Show diffs first / Cancel)
```

If the user asks to see diffs, show the diff for each modified file (use a compact format — old vs new section headers, not full file contents).

---

## Step 4: Copy updated skills

After user confirmation:

### 4a: Copy new and modified skills

For each **new** and **modified** skill (excluding bootstrap-only and update-harness):
- Copy from `HARNESS_PATH/skills/<name>.md` to `.claude/commands/<name>.md`
- Exception: `project-structure-validator.md` goes to `.claude/agents/`

### 4b: Verify copies

For each copied file, read it back and verify it is non-empty and matches the source.

### 4c: Report

```
Updated skills:
- [added] retrospective.md → .claude/commands/
- [updated] implement-feature.md → .claude/commands/
- [updated] garden.md → .claude/commands/
- [updated] create-plan.md → .claude/commands/
- [unchanged] 5 skills (skipped)
```

---

## Step 5: Add new schema fields to harness.yaml

Read the current `harness.yaml` and `HARNESS_PATH/harness.schema.yaml`.

For each field defined in the schema under the `docs` section, check if it exists in the project's `harness.yaml`. If a field is **missing**, add it with its default value.

**Rules:**
- **Only add missing fields** — never modify existing values
- **Preserve YAML formatting** — add new fields in the same style as existing ones
- **Preserve comments** — do not strip existing comments

Likely additions for projects installed before context engineering:
- `docs.memory` (default: `"memory/"`)
- `docs.scratch` (default: `"scratch/"`)

Present what will be added:

```
The following fields are missing from harness.yaml and will be added with defaults:
- docs.memory: "memory/"
- docs.scratch: "scratch/"

No existing values will be changed. Proceed? (Yes / Skip / Cancel)
```

If the user confirms, add the fields. If no fields are missing, report:

```
harness.yaml is up to date — no new fields needed.
```

---

## Step 6: Create new directories and update .gitignore

### 6a: memory/ directory

If `memory/` does not exist:
- Create `memory/` directory
- Place a `.gitkeep` inside it
- Report: "Created memory/ directory for cross-session agent memory"

### 6b: scratch/ in .gitignore

Read `.gitignore`. If `scratch/` is not listed:
- Append `scratch/` to `.gitignore` (add a blank line before it if the file doesn't end with one)
- Report: "Added scratch/ to .gitignore"

If `.gitignore` does not exist, create one with just `scratch/`.

---

## Step 7: Report AGENTS.md suggested updates

Read the project's `AGENTS.md` and `HARNESS_PATH/references/agents-md-reference.md`.

Check for differences that the user may want to adopt:

### 7a: Quick Reference table

Check if these rows exist in the Quick Reference table:
- `memory/` row
- Context management reference row

If missing, suggest adding them.

### 7b: Context Management section

Check if AGENTS.md has a `## Context Management` section. If not, suggest adding one.

### 7c: Agent Rules ordering

Check if Agent Rules follow attention-favored placement (rule 1 should be about secrets, not "read before modifying"). If the old ordering is detected, suggest reordering.

**Do NOT auto-edit AGENTS.md.** Present all suggestions and let the user choose:

```
## Suggested AGENTS.md Updates

Your AGENTS.md could benefit from these updates:

1. Add "Agent memory" and "Context management" rows to Quick Reference table
2. Add a "Context Management" section (3-4 lines about memory/ and scratch/)
3. Reorder Agent Rules for attention-favored placement (critical rules at edges)

Should I apply these? (All / Pick which ones / Skip)
```

If the user approves, make the changes. If they skip, move on.

---

## Step 8: Commit

Stage all changed files and present a summary:

```
## Update Complete

### Changes made:
- [N] skills updated in .claude/commands/
- [N] new skills added to .claude/commands/
- [N] fields added to harness.yaml
- memory/ directory created
- scratch/ added to .gitignore
- AGENTS.md updated (if applicable)

### Not touched:
- docs/* (project-owned)
- enforcement/* (project-owned)
- memory/* contents (project-owned)
- .claude/hooks/* (project-specific)

Commit these changes?
```

If the user confirms, commit with message:

```
chore: update agent harness to latest version

- Updated [N] skills, added [N] new skills
- Added memory/ and scratch/ conventions
- [any other changes made]
```

Do NOT push. Tell the user:

> Harness update committed locally. Run `git push` when ready.

---

## Rules

1. **Never overwrite harness.yaml values.** Only add missing fields with defaults.
2. **Never overwrite docs/.** These are project-owned. If the harness reference templates changed, the project docs are still valid — they were written for this specific project.
3. **Never overwrite enforcement/ scripts.** These may have project-specific customizations (language-specific patterns, custom thresholds).
4. **Never overwrite memory/.** This is accumulated cross-session learning — overwriting it destroys knowledge.
5. **Never overwrite .claude/hooks/.** Hook scripts contain project-specific commands.
6. **Always show what will change before changing it.** The user must confirm before any writes happen.
7. **Skills are safe to replace.** They are generic harness files, not project-customized. Replacing them is the whole point of the update.
8. **If in doubt, skip and report.** It is always safer to skip an update and let the user decide than to overwrite something project-specific.
