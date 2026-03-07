---
name: bootstrap-existing
description: Install the agent harness into an existing codebase. Scans the project to understand its structure, then generates configuration and documentation that describes the project as it is -- without restructuring code.
---

# Bootstrap Existing Project

You are installing the agent harness into an existing codebase. The harness repository path (`HARNESS_PATH`) was provided by the `/install-harness` skill. If it was not provided, ask the user for it before proceeding.

**Critical rule: Do NOT restructure existing code, rename files, or move directories.** The harness adapts to the project, not the other way around. Document what exists, note improvement opportunities, but never change the source code during bootstrap.

Follow every step in order.

---

## Step 1: Read reference materials

Read these files from the harness repository:
- `HARNESS_PATH/harness.schema.yaml`
- `HARNESS_PATH/references/agents-md-reference.md`
- `HARNESS_PATH/references/architecture-reference.md`
- `HARNESS_PATH/references/conventions-reference.md`
- `HARNESS_PATH/references/testing-reference.md`
- `HARNESS_PATH/references/doc-structure-reference.md`
- `HARNESS_PATH/enforcement/enforcement-reference.md`
- `HARNESS_PATH/hooks/hooks-reference.md`

---

## Step 2: Deep-scan the project

Perform a thorough scan of the existing project. Gather all of the following information before presenting anything to the user.

### 2a: Package and dependency files

Read all package manager configs found in the project root:
- `pubspec.yaml` (Dart/Flutter) -- extract name, dependencies, dev_dependencies, SDK constraints
- `package.json` (Node.js) -- extract name, scripts, dependencies, devDependencies
- `pyproject.toml` / `setup.py` / `requirements.txt` (Python) -- extract project name, dependencies, tool configs
- `go.mod` (Go) -- extract module path, Go version, dependencies
- `Cargo.toml` (Rust) -- extract package name, edition, dependencies
- `pom.xml` / `build.gradle` / `build.gradle.kts` (Java/Kotlin) -- extract group, artifact, dependencies

From these files, determine:
- **Primary language**
- **Primary framework** (from dependency names)
- **Database/ORM** (from dependency names -- look for sqlalchemy, prisma, drift, gorm, diesel, pg, mongoose, etc.)
- **Test framework** (from dev dependencies -- pytest, jest, vitest, flutter_test, etc.)
- **Linter** (from dev dependencies or config files -- ruff, eslint, dart analyze, golangci-lint, clippy)

### 2b: Directory structure

Run `tree` (or equivalent) on the project, excluding:
- `.git/`, `node_modules/`, `.dart_tool/`, `build/`, `dist/`, `.next/`, `target/`, `__pycache__/`, `.venv/`, `venv/`

Limit depth to 4 levels. Capture the output.

From the directory structure, identify:
- **Source root** (src/, lib/, internal/, app/)
- **Apparent architectural layers** (models/, services/, routes/, features/, handlers/, repositories/, etc.)
- **Test directories** (tests/, test/, __tests__/, integration_test/, spec/)
- **Configuration directories** (.github/, .circleci/, supabase/, prisma/, etc.)
- **Documentation** (README.md, docs/, CONTRIBUTING.md, etc.)

### 2c: Existing CI configuration

Check for CI config files:
- `.github/workflows/*.yml`
- `.circleci/config.yml`
- `.gitlab-ci.yml`
- `Jenkinsfile`
- `.travis.yml`

If found, read them and extract: what steps run (lint, test, build, deploy), what triggers them, what language version is used.

### 2d: Git history analysis

Run `git log --oneline -20` to see recent commit messages. Identify:
- **Commit message style** (conventional commits? imperative? lowercase? capitalized?)
- **Any patterns** (prefixes like feat:, fix:, chore:)

Run `git branch -a --list` to see branch naming patterns.

### 2e: Existing documentation

Read any existing documentation files:
- `README.md`
- `AGENTS.md` (if present)
- `CLAUDE.md` (if present)
- `CONTRIBUTING.md`
- `docs/*.md` (any existing docs)

Note what is already documented and what is missing.

### 2f: Test file inventory

Search for test files using language-appropriate patterns:
- `*_test.dart`, `*_test.go`, `test_*.py`, `*.test.ts`, `*.test.js`, `*.spec.ts`, `*.spec.js`

Count how many test files exist. Note which source directories have corresponding test coverage and which do not.

### 2g: Linter and formatter config

Check for existing linter/formatter configuration:
- `.eslintrc.*`, `eslint.config.*`
- `ruff.toml`, `pyproject.toml [tool.ruff]`
- `analysis_options.yaml`
- `.prettierrc.*`
- `.golangci.yml`
- `rustfmt.toml`, `.clippy.toml`

---

## Step 3: Present findings

Present a structured summary of everything discovered:

> ## Project Scan Results
>
> **Language:** <detected>
> **Framework:** <detected>
> **Database/ORM:** <detected or "none detected">
> **Test framework:** <detected or "none detected">
> **Linter:** <detected or "none detected">
>
> ### Directory structure:
> ```
> <tree output, annotated with layer guesses>
> ```
>
> ### Existing CI: <description or "none found">
> ### Existing tests: <N test files found, covering X directories>
> ### Existing docs: <list or "none found">
> ### Commit style: <detected pattern>
>
> ### Identified gaps:
> - <list gaps: missing tests, no CI, no docs, no linting, no .gitignore, etc.>

---

## Step 4: Propose defaults and confirm decisions

For each major configuration area, propose a default based on what was detected and ask the user to confirm or override. Ask these ONE AT A TIME.

### Decision 1: Commands

> Based on the project scan, I propose these commands for `harness.yaml`:
>
> - **install:** `<detected>`
> - **analyze:** `<detected>`
> - **format:** `<detected>`
> - **test:** `<detected>`
> - **build:** `<detected or "none">`
> - **codegen:** `<detected or "none">`
> - **migrate:** `<detected or "none">`
>
> Are these correct? Adjust any that are wrong.

### Decision 2: Architectural layers

> Looking at the directory structure, I see these apparent layers:
>
> <list detected layers with their directories>
>
> I propose mapping these to the import direction enforcement as:
> `<layer1>` -> `<layer2>` -> `<layer3>` -> `<layer4>` (imports allowed left-to-right only)
>
> Does this match your architecture? Should I adjust the layers or their order?

If the architecture does not clearly map to layers, say so and ask:

> The directory structure does not clearly map to strict architectural layers. Should I:
> - **A)** Define layers based on what I see and enable import direction enforcement
> - **B)** Skip import direction enforcement for now (you can enable it later)

### Decision 3: CI

If CI already exists:

> I found an existing CI configuration at `<path>`. I will NOT overwrite it. Should I:
> - **A)** Leave CI as-is
> - **B)** Add a separate harness verification workflow alongside the existing one

If no CI exists:

> No CI configuration was found. Should I generate `.github/workflows/ci.yml` with analyze, test, and build jobs?

### Decision 4: Documentation scope

> I will generate the following documentation describing the project AS IT IS:
> - `AGENTS.md` -- agent entry point (links to docs/)
> - `docs/architecture.md` -- current system architecture
> - `docs/conventions.md` -- coding patterns observed in the codebase
> - `docs/domain.md` -- domain concepts (I will need your input on this)
> - `docs/testing.md` -- current testing approach
> - `docs/decisions/001-initial-setup.md` -- recording the harness installation
>
> Any existing README.md or docs will NOT be modified or overwritten.
>
> Does this sound right?

### Decision 5: Domain entities

> What are the core domain entities or concepts in this project? List the main "nouns" with one-line descriptions. I will use these for `docs/domain.md`.
>
> If you are unsure, I can infer some from the model files I found:
> <list any model class names detected from scanning model directories>

---

## Step 5: Generate harness.yaml

Using the confirmed decisions and the schema from `HARNESS_PATH/harness.schema.yaml`, generate `harness.yaml` in the project root.

Rules:
1. Set `project.type` to `existing`.
2. Fill in ALL required fields.
3. Use the exact commands confirmed by the user in Decision 1.
4. Use the architectural layers confirmed in Decision 2 for `enforcement.import_direction.layers`.
5. Set `enforcement.file_size_limit` to `300` (default).
6. Include verification checks matching the available tooling (only include checks that will actually pass).
7. Set `docs.max_doc_lines` to `200`.

Write to `./harness.yaml`.

---

## Step 6: Generate AGENTS.md

Read `HARNESS_PATH/references/agents-md-reference.md` as your template.

Generate `AGENTS.md` by adapting the template to describe the project AS IT EXISTS:

1. Project name and description from the package config or user input.
2. Quick Reference table pointing to `docs/` files.
3. Tech Stack table from detected language, framework, database, and dependencies.
4. Commands section mirroring `harness.yaml` (these must be commands that actually work today).
5. Project Structure section showing the ACTUAL directory layout from the scan (not an idealized version).
6. Architecture Overview summarizing the actual architecture, pointing to `docs/architecture.md`.
7. Key Conventions listing 3-5 patterns ACTUALLY observed in the code, pointing to `docs/conventions.md`.
8. Domain Concepts listing core entities, pointing to `docs/domain.md`.
9. Testing summary describing the ACTUAL testing state, pointing to `docs/testing.md`.
10. Agent Rules section from the reference template (all 10 rules).

Keep AGENTS.md under 120 lines. Do NOT describe aspirational state -- describe reality.

---

## Step 7: Generate docs/ directory

Create the docs directory structure:

```
docs/
  architecture.md
  conventions.md
  domain.md
  testing.md
  plans/
    active/
    completed/
  decisions/
    001-initial-setup.md
```

Place a `.gitkeep` in `docs/plans/active/` and `docs/plans/completed/`.

### 7a: architecture.md

Adapt `HARNESS_PATH/references/architecture-reference.md` to describe the ACTUAL architecture:

- Draw the system overview diagram based on what actually exists (real services, real databases, real entry points).
- Map the actual directory structure to architectural layers. If the mapping is imperfect, document it honestly. For example: "The `utils/` directory contains a mix of service logic and helper functions. Consider splitting into separate layers in the future."
- Document actual dependency patterns. If there are known violations (e.g., UI directly calling the database), note them: "Known deviation: `screens/result_screen.dart` directly accesses Supabase for PDF sharing. This should be refactored through a service."
- Document actual data flows based on the codebase.
- List actual external service integrations.
- List actual entry points.

Remove all `<!-- ADAPT: -->` comment markers.

### 7b: conventions.md

Adapt `HARNESS_PATH/references/conventions-reference.md` to describe patterns ACTUALLY used in the code:

- Examine 3-5 representative source files to identify the import style actually used. Document that style (even if it is inconsistent -- note the inconsistency).
- Document the naming conventions actually in use (check file names, class names, variable names, database columns).
- Document the error handling pattern actually used in the codebase.
- Document serialization patterns if the project has models with fromJson/toJson.
- Document any barrel/index file usage.
- List the actual linter and formatter and their configuration.

Do NOT document aspirational conventions. Document what IS. If something should change, add it as a note: "Note: some files use relative imports while others use package imports. New code should prefer package imports for consistency."

Remove all `<!-- ADAPT: -->` comment markers.

### 7c: domain.md

Generate from the user's answer in Decision 5, supplemented by what you found in model files:

- Glossary with each entity and its definition.
- Entity relationships (inferred from model fields, foreign keys, or the user's description).
- Business rules (inferred from service logic, validation, or the user's description).
- User roles if applicable.

### 7d: testing.md

Adapt `HARNESS_PATH/references/testing-reference.md` to describe the ACTUAL testing state:

- Document the test framework and runner commands that actually work.
- Show the actual test directory structure (not idealized).
- Document actual mock/fixture patterns found in existing test helper files.
- Be honest about coverage: "Currently N test files exist, primarily covering <areas>. Coverage gaps exist in <areas>."
- Set a realistic coverage target based on current state. If coverage is low, suggest 50% as a first milestone rather than 80%.

Include only language-specific examples for the project's language. Remove all `<!-- ADAPT: -->` comment markers.

### 7e: decisions/001-initial-setup.md

```markdown
# 001: Agent Harness Installation

**Status:** accepted

**Context:** The agent harness was installed into this existing <language> + <framework> project to standardize agent workflows, documentation, and code quality enforcement.

**Decision:**
- Installed agent harness with documentation, enforcement rules, and skill commands
- Documented the existing architecture and conventions as-is
- Architectural layers mapped as: <layer list>
- Import direction enforcement: <enabled|disabled> (reason if disabled)
- CI: <kept existing | added new | not added>

**Alternatives considered:**
- Restructuring the codebase to match a canonical layout was rejected to avoid disruption. The harness adapts to the existing structure.

**Consequences:**
- New code should follow docs/conventions.md for consistency.
- Enforcement rules may flag existing code -- these are informational, not blockers, until the team decides to address them.
- Documentation now lives in docs/ and should be updated as the code evolves.
```

---

## Step 8: Generate enforcement rules

Create the `enforcement/` directory with the same scripts as the greenfield bootstrap:

- `run-all.sh` -- runner script
- `check-import-direction.sh` -- layer boundary enforcement (only if enabled in Decision 2)
- `check-no-secrets.sh` -- hardcoded secret detection
- `check-file-size.sh` -- file size limit enforcement

**Important for existing projects:** The enforcement scripts should be accurate but may report existing violations. That is expected. Tell the user:

> Note: Enforcement scripts may flag existing code that predates the harness installation. These are informational. You can address them incrementally or adjust the rules in `harness.yaml` if they are too strict for the current codebase.

Adapt file extensions and excluded directories to match the project's actual language and structure. Make all scripts executable.

---

## Step 9: Optionally generate CI

Based on Decision 3:

- If the user chose to leave CI as-is, skip this step.
- If the user chose to add a new workflow, generate `.github/workflows/harness-verify.yml` (separate from existing CI) with analyze and test jobs.
- If no CI existed and the user approved, generate `.github/workflows/ci.yml` with analyze, test, and build jobs appropriate to the detected language and framework.

---

## Step 10: Configure verification hooks

Read `HARNESS_PATH/hooks/hooks-reference.md` for the correct hook configuration format.

Generate `.claude/settings.json` with pre-commit verification hooks. The format uses `PreToolCall` to intercept `git commit` commands:

```json
{
  "hooks": {
    "PreToolCall": [
      {
        "matcher": "Bash",
        "pattern": "git commit",
        "command": "<analyze command> && <test command> && bash enforcement/run-all.sh"
      }
    ]
  }
}
```

Replace `<analyze command>` and `<test command>` with the actual commands from the `harness.yaml` you generated in Step 5 (e.g., `ruff check src/` and `pytest tests/`).

Chain all checks with `&&` so the commit is blocked if ANY check fails.

Ask if the user wants this created or updated now. If yes, create or merge into the existing file.

---

## Step 11: Present summary for approval

Present a complete summary of everything generated:

> ## Bootstrap Summary
>
> **Project:** <name>
> **Type:** existing
> **Stack:** <language> + <framework> + <database>
>
> ### Files generated (new):
> <list every new file>
>
> ### Files NOT modified:
> - All existing source code (untouched)
> - <existing CI config> (preserved)
> - <existing README.md> (preserved)
> - <any other existing docs> (preserved)
>
> ### Known issues flagged by enforcement:
> - <N files exceed size limit>
> - <N import direction violations>
> - <N possible hardcoded secrets>
>
> (These are informational only. Address them incrementally.)
>
> Does this look correct? Should I proceed with committing?

Wait for explicit approval.

---

## Step 12: Commit

After approval, stage ONLY the new files generated by the harness. Do NOT stage any existing files that were modified during scanning (there should be none, but verify with `git status`).

Commit with message:

```
chore: install agent harness

- Generated harness.yaml with project configuration
- Created AGENTS.md and docs/ documentation structure
- Added enforcement rules (import direction, secrets, file size)
- Copied agent skills to .claude/commands/
```

Do NOT push to remote. Tell the user:

> Harness installation complete. All files have been committed locally. Run `git push` when you are ready to push to the remote repository.
>
> Next steps:
> 1. Review the generated docs in `docs/` and correct any inaccuracies.
> 2. Run `bash enforcement/run-all.sh` to see the current state of enforcement.
> 3. Address any critical enforcement violations when convenient.
> 4. Use `/implement-feature` or `/create-plan` to start your next piece of work.
