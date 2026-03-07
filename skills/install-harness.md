---
name: install-harness
description: Install the agent harness into a target project. Detects project type (greenfield vs existing), copies skills, and delegates to the appropriate bootstrap command.
---

# Install Harness

You are installing the agent harness framework into the current project. Follow these steps exactly in order.

---

## Step 1: Locate the harness repository

Ask the user where the agent-harness repository is located. Before asking, attempt auto-detection:

1. Check if a sibling directory `../agent-harness/` exists relative to the current working directory.
2. Check if `~/agent-harness/` exists.
3. Check if the environment variable `AGENT_HARNESS_PATH` is set and points to a valid directory.

If any of these exist, present the detected path and ask the user to confirm or provide an alternative:

> I detected the agent-harness repository at `<detected-path>`. Is this correct, or should I use a different path?

If none are detected, ask:

> Where is the agent-harness repository located? Please provide the absolute path.

Store the confirmed path as `HARNESS_PATH` for the rest of this skill.

---

## Step 2: Verify harness repository structure

Verify that `HARNESS_PATH` contains the expected structure. Check for the existence of ALL of the following:

- `HARNESS_PATH/references/` directory
- `HARNESS_PATH/skills/` directory
- `HARNESS_PATH/enforcement/` directory
- `HARNESS_PATH/hooks/` directory
- `HARNESS_PATH/harness.schema.yaml` file
- `HARNESS_PATH/references/agents-md-reference.md` file
- `HARNESS_PATH/references/architecture-reference.md` file
- `HARNESS_PATH/references/conventions-reference.md` file
- `HARNESS_PATH/references/testing-reference.md` file
- `HARNESS_PATH/references/doc-structure-reference.md` file

If any are missing, report exactly which files/directories are absent and stop. Tell the user:

> The harness repository at `<path>` is incomplete. Missing: `<list>`. Please ensure the harness repo is fully set up before running this command.

Do NOT proceed if verification fails.

---

## Step 3: Detect project type (greenfield vs existing)

Analyze the current working directory (the target project) to determine whether it is a greenfield or existing project.

### 3a: Count source files

Count files in the project, excluding these directories:
- `.git/`
- `node_modules/`
- `.dart_tool/`
- `build/`
- `dist/`
- `.next/`
- `target/`
- `__pycache__/`
- `.venv/`
- `venv/`
- `.idea/`
- `.vscode/`

Count only files with common source extensions: `.dart`, `.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`, `.rs`, `.java`, `.kt`, `.swift`, `.c`, `.cpp`, `.h`, `.rb`, `.ex`, `.exs`.

### 3b: Check for package manager configs

Look for any of these files in the project root:
- `pubspec.yaml` (Dart/Flutter)
- `package.json` (Node.js/TypeScript)
- `pyproject.toml` or `setup.py` or `requirements.txt` (Python)
- `go.mod` (Go)
- `Cargo.toml` (Rust)
- `pom.xml` or `build.gradle` or `build.gradle.kts` (Java/Kotlin)
- `Gemfile` (Ruby)
- `mix.exs` (Elixir)

### 3c: Classify

Apply these rules:
- If fewer than 5 source files AND no package manager config found: classify as **greenfield**.
- Otherwise: classify as **existing**.

### 3d: Confirm with user

Present your findings and ask the user to confirm:

> I scanned the project and found **<N> source files** and **<detected configs or "no package manager configs">**.
>
> This looks like a **<greenfield|existing>** project. Is that correct?

If the user disagrees, use their classification instead.

---

## Step 4: Check for existing harness installation

Before copying anything, check if the project already has harness artifacts:

- `.claude/commands/` directory with skill files
- `harness.yaml` in the project root
- `AGENTS.md` in the project root
- `docs/architecture.md`, `docs/conventions.md`, `docs/domain.md`, `docs/testing.md`

If any exist, warn the user:

> This project already has some harness artifacts: `<list>`. Installing will overwrite these. Do you want to proceed? (Overwrite all / Keep existing and only add missing / Cancel)

Respect the user's choice. If they say "Keep existing and only add missing", skip files that already exist during all subsequent steps.

---

## Step 5: Copy skills into target project

Create the `.claude/commands/` directory in the target project if it does not exist.

Copy `.md` files from `HARNESS_PATH/skills/` into the target project, with these exceptions -- do NOT copy:
- `install-harness.md` (this file -- it stays in the harness repo only)
- `bootstrap-greenfield.md` (bootstrap-only, stays in harness repo)
- `bootstrap-existing.md` (bootstrap-only, stays in harness repo)

**Special handling for `project-structure-validator.md`:**
- This file goes to `.claude/agents/project-structure-validator.md` (NOT `.claude/commands/`)
- Create the `.claude/agents/` directory if it does not exist

All other skill files go to `.claude/commands/`.

For each file copied, verify it was written correctly by reading it back and checking it is non-empty.

Report what was copied:

> Copied the following skills to `.claude/commands/`:
> - `<filename1>`
> - `<filename2>`
> - ...
>
> Copied to `.claude/agents/`:
> - `project-structure-validator.md`

---

## Step 6: Delegate to bootstrap

Based on the project type determined in Step 3:

- If **greenfield**: Invoke `/bootstrap-greenfield` and pass `HARNESS_PATH` as context. Tell the user:

  > Project classified as greenfield. Starting greenfield bootstrap...

- If **existing**: Invoke `/bootstrap-existing` and pass `HARNESS_PATH` as context. Tell the user:

  > Project classified as existing. Starting existing project bootstrap...

The bootstrap skill will handle generating `harness.yaml`, `AGENTS.md`, `docs/`, `enforcement/`, and all other project-specific artifacts.

---

## Error Recovery

If any step fails:

1. Report exactly what failed and why.
2. Do NOT attempt to continue to subsequent steps.
3. Do NOT leave the project in a partially configured state -- if skills were copied but bootstrap failed, tell the user what was already done and what remains.
4. Suggest the user fix the issue and re-run `/install-harness`.
