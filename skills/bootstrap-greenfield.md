---
name: bootstrap-greenfield
description: Bootstrap a new greenfield project with full agent harness configuration, documentation, project scaffolding, and enforcement rules.
---

# Bootstrap Greenfield Project

You are setting up a brand-new project from scratch with the full agent harness. The harness repository path (`HARNESS_PATH`) was provided by the `/install-harness` skill. If it was not provided, ask the user for it before proceeding.

Follow every step in order. Do not skip steps. Do not batch questions -- ask them ONE AT A TIME.

---

## Step 1: Read the harness schema

Read `HARNESS_PATH/harness.schema.yaml` to understand the full configuration structure. You will use this schema to generate `harness.yaml` in Step 3.

Also read these reference documents -- you will adapt them in later steps:
- `HARNESS_PATH/references/agents-md-reference.md`
- `HARNESS_PATH/references/architecture-reference.md`
- `HARNESS_PATH/references/conventions-reference.md`
- `HARNESS_PATH/references/testing-reference.md`
- `HARNESS_PATH/references/doc-structure-reference.md`
- `HARNESS_PATH/enforcement/enforcement-reference.md`
- `HARNESS_PATH/hooks/hooks-reference.md`

---

## Step 2: Gather project information

Ask the following questions ONE AT A TIME. Wait for each answer before asking the next. Use clear, specific prompts.

### Question 1: Project purpose

> What does this project do? Describe its purpose, who the users are, and the core problem it solves. (A few sentences is fine.)

### Question 2: Language and framework

> What language and framework will this project use?
>
> Common choices:
> - Python + FastAPI
> - Python + Django
> - TypeScript + Next.js
> - TypeScript + Express
> - Dart + Flutter
> - Go + Gin
> - Go + standard library
> - Rust + Axum
> - Java + Spring Boot
> - Other (please specify)

### Question 3: Target platforms

> What platforms will this project target? Select all that apply:
> - Mobile app (Android/iOS)
> - Web API (REST or GraphQL backend)
> - Web frontend (browser-based UI)
> - CLI tool (command-line interface)
> - Background worker (job processing, queues)
> - Other (please specify)

### Question 4: Backend and database

> What backend or database will this project use?
>
> Common choices:
> - PostgreSQL
> - PostgreSQL via Supabase
> - MySQL
> - SQLite
> - MongoDB
> - Firebase / Firestore
> - Redis
> - DynamoDB
> - None (no database needed)
> - Other (please specify)

### Question 5: Deployment target

> Where will this project be deployed?
>
> Common choices:
> - Docker (self-hosted or cloud)
> - Vercel
> - Fly.io
> - AWS Lambda
> - Google Cloud Run
> - Kubernetes
> - App stores (mobile)
> - None yet / undecided
> - Other (please specify)

### Question 6: Core domain entities

> What are the core domain entities or concepts in this project? List the main "nouns" -- the things the system manages, creates, or processes. For each, give a one-line description.
>
> Example: "User -- a registered person who can log in and create orders", "Order -- a purchase with line items and payment status"

---

## Step 3: Generate harness.yaml

Using the answers from Step 2 and the schema from Step 1, generate a `harness.yaml` file in the project root.

### Rules for generating harness.yaml

1. Fill in ALL required fields from the schema. Do not leave required fields empty.
2. Choose opinionated defaults for the detected stack:

   **Python + FastAPI:**
   - linter: `ruff check src/`
   - formatter: `ruff format src/`
   - test_framework: `pytest`
   - install: `uv sync` (prefer uv) or `pip install -e ".[dev]"`
   - analyze: `ruff check src/`
   - test: `pytest`

   **TypeScript + Next.js / Express:**
   - linter: `eslint .`
   - formatter: `prettier --write .`
   - test_framework: `vitest` (prefer vitest) or `jest`
   - install: `npm install`
   - analyze: `eslint .`
   - test: `npm test`

   **Dart + Flutter:**
   - linter: `dart analyze lib/`
   - formatter: `dart format lib/`
   - test_framework: `flutter_test`
   - install: `flutter pub get`
   - analyze: `dart analyze lib/`
   - test: `flutter test`

   **Go:**
   - linter: `golangci-lint run`
   - formatter: `gofmt -w .`
   - test_framework: `go test`
   - install: `go mod download`
   - analyze: `golangci-lint run`
   - test: `go test ./...`

   **Rust:**
   - linter: `cargo clippy`
   - formatter: `cargo fmt`
   - test_framework: `cargo test`
   - install: `cargo build`
   - analyze: `cargo clippy -- -D warnings`
   - test: `cargo test`

3. Set `project.type` to `greenfield`.
4. Set `enforcement.import_direction.enabled` to `true` with layers ordered bottom-to-top: `[models, repositories, services, ui]` (adapt layer names to the language conventions).
5. Set verification checks to include lint, test, and enforcement.
6. Set `docs.max_doc_lines` to `200`.
7. Include platform entries based on the user's answers (entry points, build targets, deploy commands).

Write the file to `./harness.yaml` in the project root.

---

## Step 4: Generate AGENTS.md

Read `HARNESS_PATH/references/agents-md-reference.md` as your template.

Generate `AGENTS.md` in the project root by adapting the template:

1. Replace the project name placeholder with the actual project name.
2. Write a 2-3 sentence description based on the user's answer to Question 1.
3. Fill in the Quick Reference table pointing to:
   - `docs/architecture.md`
   - `docs/conventions.md`
   - `docs/domain.md`
   - `docs/testing.md`
   - `docs/plans/active/`
   - `docs/decisions/`
4. Fill in the Tech Stack table from the user's language, framework, database, and other answers.
5. Fill in the Commands section mirroring what was written to `harness.yaml`.
6. Write a brief Project Structure section showing the planned directory layout (you will create this in Step 6).
7. Write a brief Architecture Overview pointing to `docs/architecture.md`.
8. List 3-5 key conventions pointing to `docs/conventions.md`.
9. List 3-5 core domain terms from the user's answer to Question 6, pointing to `docs/domain.md`.
10. Write a brief testing summary pointing to `docs/testing.md`.
11. Include the Agent Rules section exactly as shown in the reference (all 10 rules).

Keep AGENTS.md under 120 lines. It is a table of contents, not the full documentation.

---

## Step 5: Generate docs/ directory

Create the following directory structure:

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

### 5a: architecture.md

Read `HARNESS_PATH/references/architecture-reference.md` as your template.

Adapt it for this project:
- Replace the system overview diagram with one showing the actual components from the user's answers (database, auth, storage, entry points).
- Fill in the layer definitions appropriate to the language/framework.
- Fill in the dependency rules table.
- Write data flow diagrams for the primary write and read paths based on the project's purpose.
- List external service integrations from the user's answers.
- List entry points based on the platforms chosen.

Remove all `<!-- ADAPT: -->` comment markers. The output must be clean markdown with no template instructions.

### 5b: conventions.md

Read `HARNESS_PATH/references/conventions-reference.md` as your template.

Adapt it for this project:
- Include only the import ordering section for the chosen language (remove other language examples).
- Fill in naming conventions appropriate to the language.
- Fill in error handling patterns appropriate to the framework.
- Include serialization patterns if the project has a database or API.
- Include barrel/index file conventions appropriate to the language.
- Fill in formatting and linting tool names from `harness.yaml`.

Remove all `<!-- ADAPT: -->` comment markers.

### 5c: domain.md

Generate a domain document from the user's answer to Question 6 (core domain entities). Include:

- A glossary section with each entity name and its definition.
- An entity relationships section describing how entities relate to each other (inferred from context; ask the user if relationships are unclear).
- A business rules section listing any rules or invariants that can be inferred from the project description. If none are obvious, add placeholder headers with a note: "To be filled in as business rules are defined."
- A user roles section if roles were mentioned or can be inferred.

### 5d: testing.md

Read `HARNESS_PATH/references/testing-reference.md` as your template.

Adapt it for this project:
- Include only the test structure examples for the chosen language (remove other languages).
- Fill in test runner commands from `harness.yaml`.
- Fill in the directory structure showing where unit, integration, and e2e tests will live.
- Fill in mock/fixture conventions appropriate to the language and framework.
- Set coverage targets: suggest 80% line coverage as a starting goal.

Remove all `<!-- ADAPT: -->` comment markers.

### 5e: decisions/001-initial-setup.md

Create a decision record documenting the bootstrap choices:

```markdown
# 001: Initial Project Setup

**Status:** accepted

**Context:** This project was bootstrapped as a greenfield project using the agent harness. The following technology choices were made during setup.

**Decision:**
- Language: <language>
- Framework: <framework>
- Database: <database>
- Deployment: <deployment>
- Linter: <linter>
- Test framework: <test framework>
- Architecture: four-layer (models, repositories, services, UI/API)

**Alternatives considered:** None -- these were selected during initial project setup based on project requirements.

**Consequences:**
- All future code must follow the conventions established in docs/conventions.md.
- Architectural layer boundaries are enforced via import direction rules.
- These choices can be revisited by creating a new decision record that supersedes this one.
```

### 5f: Create empty plan directories

Create `docs/plans/active/` and `docs/plans/completed/` as empty directories. Place a `.gitkeep` file in each so Git tracks them.

---

## Step 6: Scaffold project structure

Create the directory layout and minimal starter files appropriate to the chosen language and framework.

### 6a: Directory layout

Create directories matching the four-layer architecture. Use language-idiomatic naming:

**Python + FastAPI:**
```
src/
  models/
  repositories/
  services/
  api/
    routes/
tests/
  unit/
  integration/
  helpers/
```

**TypeScript + Next.js:**
```
src/
  models/
  repositories/
  services/
  app/          (Next.js app router)
tests/
  unit/
  integration/
  helpers/
```

**TypeScript + Express:**
```
src/
  models/
  repositories/
  services/
  routes/
tests/
  unit/
  integration/
  helpers/
```

**Dart + Flutter:**
```
lib/
  core/
    models/
    services/
  features/
  local/
test/
  unit/
  widget/
  helpers/
integration_test/
```

**Go:**
```
cmd/
  <appname>/
internal/
  models/
  repositories/
  services/
  handlers/
tests/
  integration/
  helpers/
```

**Rust:**
```
src/
  models/
  repositories/
  services/
  handlers/
tests/
  integration/
  helpers/
```

Adapt these templates if the user's answers suggest a different structure. The key rule: **every layer gets its own directory, and test directories mirror source directories**.

### 6b: Minimal entry point files

Create the minimum files needed to run the project:

- A main entry point that starts a server, prints "Hello, world!", or returns a health check response -- whatever is idiomatic for the framework.
- If the framework needs a config file (e.g., `next.config.js`, `analysis_options.yaml`), create it with sensible defaults.

Do NOT generate boilerplate beyond what is needed to run `build` and `test` successfully. The user will add real code later.

### 6c: .gitignore

Generate a `.gitignore` appropriate for the chosen language. Include at minimum:
- Build output directories
- Dependency directories (node_modules, .dart_tool, target, __pycache__, .venv)
- IDE files (.idea, .vscode -- except .vscode/settings.json if useful)
- OS files (.DS_Store, Thumbs.db)
- Environment files (.env, .env.local)
- Coverage output

### 6d: CI workflow

Generate `.github/workflows/ci.yml` with three jobs:

1. **analyze** -- Run the linter/static analysis command from `harness.yaml`.
2. **test** -- Run the test command from `harness.yaml`. Depends on analyze.
3. **build** -- Run the build command from `harness.yaml` (if one was defined). Depends on test.

Use the appropriate GitHub Actions setup action for the language (actions/setup-python, actions/setup-node, subosito/flutter-action, actions/setup-go, dtolnay/rust-toolchain).

Set the workflow to trigger on `push` to `main` and on all `pull_request` events.

### 6e: Empty test files

Create at least one placeholder test file in the unit test directory that:
- Imports the test framework
- Contains a single passing test (e.g., "sanity check: 1 + 1 = 2" or "app module is importable")
- Demonstrates the AAA (Arrange-Act-Assert) pattern from docs/testing.md

This ensures `test` command passes immediately after bootstrap.

---

## Step 7: Generate enforcement rules

Create the `enforcement/` directory in the project root with these scripts:

### 7a: run-all.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
FAILED=0

for script in "$SCRIPT_DIR"/*.sh; do
    [ "$(basename "$script")" = "run-all.sh" ] && continue
    echo "Running $(basename "$script")..."
    if ! bash "$script"; then
        FAILED=1
        echo "FAILED: $(basename "$script")"
    fi
done

exit $FAILED
```

Make it executable (`chmod +x`).

### 7b: check-import-direction.sh

Generate a script that checks import direction based on the layers defined in `harness.yaml`. The script should:

1. Define the layer order (e.g., models < repositories < services < ui).
2. For each source file, determine which layer it belongs to based on its directory path.
3. Scan its imports and flag any that import from a higher layer.
4. Exit 0 if no violations, exit 1 if violations found, printing each violation.

The implementation depends on the language:
- **Python:** grep for `from <project>.` and `import <project>.` patterns, check the module path against layers.
- **TypeScript:** grep for `from '@/` or `from '../` patterns, resolve to layer.
- **Dart:** grep for `import 'package:<app>/` patterns, resolve to layer.
- **Go:** grep for `"<module>/internal/` patterns, resolve to layer.

Keep the script simple. It does not need to be a full AST parser -- pattern matching on import lines is sufficient.

Make it executable.

### 7c: check-no-secrets.sh

Generate a script that scans source files for patterns matching hardcoded secrets:

```bash
#!/usr/bin/env bash
set -euo pipefail

PATTERN='(API_KEY|SECRET|PASSWORD|TOKEN|PRIVATE_KEY)\s*='
# Adapt the file extensions to the project language
FILES=$(find . -type f \( -name "*.py" -o -name "*.ts" -o -name "*.dart" -o -name "*.go" \) \
    -not -path "./.git/*" \
    -not -path "./node_modules/*" \
    -not -path "./.venv/*" \
    -not -path "./build/*")

VIOLATIONS=0
while IFS= read -r file; do
    [ -z "$file" ] && continue
    if grep -Pn "$PATTERN" "$file" 2>/dev/null; then
        echo "VIOLATION: Possible hardcoded secret in $file"
        VIOLATIONS=1
    fi
done <<< "$FILES"

if [ "$VIOLATIONS" -eq 1 ]; then
    echo "Hardcoded secrets detected. Use environment variables instead."
    exit 1
fi

echo "No hardcoded secrets found."
exit 0
```

Adapt the file extensions to match only the project's language. Make it executable.

### 7d: check-file-size.sh

Generate a script that flags source files exceeding the `file_size_limit` from `harness.yaml` (default 300 lines):

```bash
#!/usr/bin/env bash
set -euo pipefail

MAX_LINES=300  # From harness.yaml enforcement.file_size_limit
# Adapt extensions to the project language
EXTENSIONS="-name '*.py' -o -name '*.ts' -o -name '*.dart' -o -name '*.go'"

VIOLATIONS=0
while IFS= read -r file; do
    [ -z "$file" ] && continue
    LINES=$(wc -l < "$file")
    if [ "$LINES" -gt "$MAX_LINES" ]; then
        echo "VIOLATION: $file has $LINES lines (max $MAX_LINES)"
        VIOLATIONS=1
    fi
done < <(eval find . -type f \\\( $EXTENSIONS \\\) \
    -not -path "./.git/*" \
    -not -path "./node_modules/*" \
    -not -path "./.venv/*" \
    -not -path "./build/*" \
    -not -path "./target/*" \
    -not -path "./.dart_tool/*")

if [ "$VIOLATIONS" -eq 1 ]; then
    echo "Files exceeding $MAX_LINES lines detected. Consider splitting."
    exit 1
fi

echo "All files within size limit."
exit 0
```

Adapt the file extensions and excluded directories to the project's language. Make it executable.

---

## Step 8: Configure verification hooks

Read `HARNESS_PATH/hooks/hooks-reference.md` for the correct hook configuration format.

Generate `.claude/settings.json` with pre-commit verification hooks and a wrapper script. The format uses `PreToolUse` to intercept Bash tool calls:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/pre-commit-checks.sh"
          }
        ]
      }
    ]
  }
}
```

Also generate `.claude/hooks/pre-commit-checks.sh` — a wrapper script that:
1. Reads JSON context from stdin
2. Extracts the command via `jq -r '.tool_input.command // ""'`
3. Exits 0 early if the command is not `git commit`
4. Changes to the session working directory via `jq -r '.session.cwd // "."'`
5. Chains the actual check commands with `&&` (e.g., `bash enforcement/run-all.sh && ruff check src/ && pytest tests/`)

Replace the check commands with the actual commands from the `harness.yaml` you generated in Step 3.

Chain all verification checks with `&&` so the commit is blocked if ANY check fails. The error output is returned to the agent, which can then read it, fix the issue, and retry.

Ask the user if they want you to create or update `.claude/settings.json` with these hooks now.

If yes, create or merge into the existing file, preserving any existing settings.

---

## Step 9: Present summary for approval

Before committing, present a complete summary of everything generated:

> ## Bootstrap Summary
>
> **Project:** <name>
> **Type:** greenfield
> **Stack:** <language> + <framework> + <database>
>
> ### Files generated:
>
> **Configuration:**
> - `harness.yaml` -- project configuration and commands
> - `.gitignore` -- language-appropriate ignore rules
> - `.github/workflows/ci.yml` -- CI pipeline (analyze, test, build)
>
> **Documentation:**
> - `AGENTS.md` -- agent entry point and quick reference
> - `docs/architecture.md` -- system architecture and layer definitions
> - `docs/conventions.md` -- coding conventions and patterns
> - `docs/domain.md` -- domain glossary and business rules
> - `docs/testing.md` -- testing strategy and conventions
> - `docs/decisions/001-initial-setup.md` -- bootstrap decision record
> - `docs/plans/active/.gitkeep`
> - `docs/plans/completed/.gitkeep`
>
> **Enforcement:**
> - `enforcement/run-all.sh` -- enforcement runner
> - `enforcement/check-import-direction.sh` -- layer boundary enforcement
> - `enforcement/check-no-secrets.sh` -- hardcoded secret detection
> - `enforcement/check-file-size.sh` -- file size limit enforcement
>
> **Source scaffold:**
> - `<list all created source directories and files>`
>
> **Skills copied:**
> - `.claude/commands/<list each skill file>`
>
> Does this look correct? Should I proceed with committing?

Wait for explicit approval before committing.

---

## Step 10: Commit

After the user approves, stage all generated files and commit:

```
chore: bootstrap project with agent harness

- Generated harness.yaml with project configuration
- Created AGENTS.md and docs/ documentation structure
- Scaffolded project directory layout with minimal entry points
- Added enforcement rules (import direction, secrets, file size)
- Added CI workflow (.github/workflows/ci.yml)
- Copied agent skills to .claude/commands/
```

Do NOT push to remote. Tell the user:

> Bootstrap complete. All files have been committed locally. Run `git push` when you are ready to push to the remote repository.
