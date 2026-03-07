# Agent Harness Design

## Purpose

Build `agent-harness` — a standalone repository containing reference docs and Claude Code skills that, when installed into any project, create a self-contained harness enabling autonomous agent-driven development.

The harness shifts the engineering model: humans steer (decisions, plans, final merge), agents execute (code, tests, review, cleanup).

Inspired by OpenAI's harness engineering approach: the discipline of building scaffolding around AI agents — not the agent itself, but the constraints, feedback loops, and verification systems that let agents produce reliable work.

## Core Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Framework support | Agnostic | Works for any language/framework without pre-built templates |
| Installation | Claude Code skill → self-contained copy | No external dependency after setup |
| Architecture | Agent-Native | Reference docs, not templates. Agent reasons and adapts. |
| Bootstrap modes | Greenfield + Existing | Different flows for new vs established codebases |
| Convention handling | Opinionated defaults | Agent proposes sensible defaults for detected stack, user confirms/overrides |
| Docs structure | Hybrid | Start shallow (5-8 files), auto-split when docs exceed 200 lines |
| Verification trigger | Pre-commit gate | Lint + test + enforcement run together at commit time. No per-edit checks. |
| Retry limit | 3 | Retries before escalating to human. Conservative to catch fundamental issues early. |
| Review agent | CodeRabbit | External reviewer. Agent reads comments via `gh`, addresses them, loops up to 3 cycles. |
| Entropy management | Post-feature | Built into /implement-feature as final step, not a separate trigger. |
| Greenfield output | Config + docs + scaffold | Ready to start coding, but no first implementation plan. |
| Human intervention | Plan approval, escalations, final merge | Everything else is autonomous. |

## Repo Structure

```
agent-harness/
├── README.md                          # What this is, how to use it
├── harness.schema.yaml                # Schema definition for harness.yaml
│
├── references/                        # "What good looks like" — agent reads and adapts
│   ├── agents-md-reference.md         # Reference AGENTS.md (table-of-contents style, ~100 lines)
│   ├── testing-reference.md            # Reference testing guide
│   ├── conventions-reference.md       # Reference coding conventions doc
│   ├── architecture-reference.md      # Reference architecture doc
│   └── doc-structure-reference.md     # How to organize docs/ directory
│
├── skills/                            # Claude Code commands (copied into .claude/commands/)
│   ├── install-harness.md             # Entry point — detects greenfield vs existing, delegates
│   ├── bootstrap-greenfield.md        # Greenfield project brainstorm + scaffold
│   ├── bootstrap-existing.md          # Existing project analysis + adaptation
│   ├── implement-feature.md           # Orchestration: plan → code → test → review → merge-ready
│   ├── create-plan.md                 # Implementation plan generator
│   ├── create-pr.md                   # PR creation with standards gate
│   ├── create-tests.md               # Test generator (tier-aware)
│   ├── create-qa.md                   # QA checklist generator
│   ├── deslop.md                      # Code cleanup (already framework-agnostic)
│   ├── garden.md                      # Entropy scan + cleanup
│   └── doc-split.md                   # Auto-split docs exceeding line threshold
│
├── enforcement/                       # Architecture enforcement framework
│   ├── enforcement-reference.md       # How to write project-specific rules
│   └── common-rules/                  # Language-agnostic rule patterns
│       ├── no-secrets-in-code.md
│       ├── import-direction.md
│       └── file-size-limits.md
│
└── hooks/                             # Claude Code hook configurations
    └── hooks-reference.md             # Reference for verification hooks setup
```

### Design principles

- `references/` are not templates. They are examples of well-structured docs that the agent reads, understands, and adapts to the target project.
- `skills/` are framework-agnostic. They read `harness.yaml` for project-specific commands rather than hardcoding any tool.
- `enforcement/` describes what to enforce. The agent generates the actual checks (scripts, tests, CI steps) for the target project's language.
- After installation, the target project is self-contained. The harness repo is not needed at runtime.

## harness.yaml — Central Config

Generated during bootstrap. Every skill reads this for project-specific values.

```yaml
version: 1

project:
  name: "my-project"
  description: "Short description"
  type: "greenfield"  # or "existing"

stack:
  language: "python"
  framework: "fastapi"
  runtime: "3.12"
  backend: "postgresql"
  orm: "sqlalchemy"
  test_framework: "pytest"
  linter: "ruff"
  formatter: "ruff format"

commands:
  install: "uv sync"
  analyze: "ruff check src/"
  format: "ruff format src/"
  test: "pytest tests/"
  test_unit: "pytest tests/unit/"
  test_integration: "pytest tests/integration/"
  build: "docker build -t my-project ."
  migrate: "alembic upgrade head"

platforms:
  - name: "api"
    entry: "src/main.py"
    targets: ["docker"]
    deploy: "fly deploy"

docs:
  root: "docs/"
  architecture: "docs/architecture.md"
  conventions: "docs/conventions.md"
  domain: "docs/domain.md"
  testing: "docs/testing.md"
  plans_active: "docs/plans/active/"
  plans_completed: "docs/plans/completed/"
  decisions: "docs/decisions/"
  max_doc_lines: 200

enforcement:
  import_direction:
    enabled: true
    layers: ["models", "repositories", "services", "api"]
    rule: "left-to-right only"
  file_size_limit: 300
  no_secrets_pattern: "(API_KEY|SECRET|PASSWORD|TOKEN)\\s*="
  custom_rules: "enforcement/"

review:
  provider: "coderabbit"
  auto_address_comments: true
  max_review_cycles: 3

verification:
  max_retries: 3
  on_fail: "fix"
  checks:
    - name: "lint"
      command: "ruff check src/"
    - name: "test"
      command: "pytest tests/ -x"
    - name: "enforcement"
      command: "bash enforcement/run-all.sh"

gardening:
  enabled: true
  trigger: "post-feature"
  checks:
    - "unused imports and exports"
    - "files exceeding file_size_limit"
    - "docs matching actual code structure"
    - "test coverage gaps"
```

## Bootstrap Flows

### Greenfield (`/bootstrap-greenfield`)

Triggered when `/install-harness` runs on an empty or near-empty repo.

1. Agent asks questions one at a time:
   - What does this project do? (purpose, users, core domain)
   - What language/framework?
   - What platforms? (mobile, web, API, CLI)
   - What backend/database?
   - What's the deployment target?
   - What are the core domain entities?
2. Agent generates:
   - `harness.yaml` (filled from conversation)
   - `AGENTS.md` (table-of-contents, ~100 lines)
   - `docs/` directory (architecture, conventions, domain, testing)
   - Project scaffold (directory structure, entry points, CI config, .gitignore, empty test dirs)
   - `.claude/` directory (all skills adapted, hooks configured, structure validator)
   - `enforcement/` directory (architecture rules as executable checks)
3. Agent presents summary for approval
4. Agent commits: `chore: bootstrap project with agent harness`

### Existing Project (`/bootstrap-existing`)

Triggered when `/install-harness` runs on a repo with existing code.

1. Agent scans:
   - Package files (pubspec.yaml, package.json, pyproject.toml, go.mod, Cargo.toml)
   - Directory structure
   - Existing tests, CI configs, docs
   - Git history
2. Agent presents findings as a summary
3. Agent asks targeted questions:
   - Confirm detected tools and conventions
   - Suggest architectural improvements (with options)
   - Identify gaps (missing tests, no CI, no docs)
4. Agent generates same outputs as greenfield, but adapted to existing reality
   - Documents the codebase as-is, does not restructure existing code
   - Improvements come later through normal plan → implement → review cycle
5. Agent presents summary for approval
6. Agent commits: `chore: install agent harness`

## Skill System

### Skills overview

| Skill | Purpose | Reads from harness.yaml |
|---|---|---|
| `/install-harness` | Entry point, detects mode, delegates | — |
| `/bootstrap-greenfield` | Greenfield brainstorm + scaffold | Generates it |
| `/bootstrap-existing` | Existing project scan + adapt | Generates it |
| `/implement-feature` | Orchestrator: plan → code → test → PR | commands, verification, review, gardening |
| `/create-plan` | Implementation plan generator | commands, docs paths |
| `/create-tests` | Tier-aware test generator | commands.test, stack.test_framework |
| `/create-pr` | PR with standards gate | commands.analyze, commands.test, review |
| `/create-qa` | QA checklist from diff | docs.domain (for categories) |
| `/deslop` | Code cleanup | — (framework-agnostic) |
| `/garden` | Entropy scan + cleanup | enforcement, docs, gardening.checks |
| `/doc-split` | Auto-split large docs | docs.max_doc_lines |

### `/implement-feature` — The Orchestrator

This is the primary autonomous workflow:

1. If no plan exists, invoke `/create-plan` first
2. Read plan, break into steps
3. For each step:
   a. Write/update tests
   b. Implement code
   c. Verification loop (pre-commit gate):
      - Run all `verification.checks` (lint, test, enforcement)
      - If fail → read error → fix → retry (up to `max_retries: 3`)
      - If still failing → STOP, escalate to human
   d. Mark step done in plan file
4. Run enforcement checks
5. Run `/deslop`
6. Run `/garden`
7. Run `/create-qa`
8. Push branch, open PR via `gh`
9. Wait for CodeRabbit review
10. Read review comments via `gh api`
11. Address comments → push → loop (up to `review.max_review_cycles: 3`)
12. If approved → notify human for final merge decision
13. If max cycles reached → escalate with summary

### Verification Hook System

Pre-commit gate configuration generated from `harness.yaml → verification.checks`:

```
Agent edits files → works freely, no interruption

Agent runs git commit →
  Pre-commit gate fires:
    1. Run lint (commands.analyze)
    2. Run tests (commands.test)
    3. Run enforcement (enforcement checks)
  If any fail:
    → Agent reads error output
    → Fixes the issue
    → Retries commit (up to 3 times)
  If still failing after 3 retries:
    → Abort commit
    → Escalate to human with error summary
```

## Enforcement Framework

### Rule definition

During bootstrap, agent and user agree on architectural rules. Recorded in:
1. `harness.yaml → enforcement` (declarative)
2. `enforcement/` directory (executable checks)

### Rule categories

**Structural** (generated per-project):
- Import/dependency direction between layers
- File location constraints
- Barrel file / re-export completeness

**Universal** (same across all projects):
- No secrets in code (regex pattern scan)
- File size limits
- No TODO without linked issue

**Custom** (added over time):
- Discovered drift patterns get encoded as new rules
- Agent generates check in `enforcement/`, updates `harness.yaml`

### Execution points

1. Pre-commit hook (part of verification gate)
2. CI (dedicated job, blocks PRs)
3. `/garden` (reports violations that slipped through)

### Enforcement error messages

Custom error messages include remediation instructions so the agent can self-fix:

```
ERROR: enforcement/import-direction.sh
  src/api/routes.py imports from src/models/user.py directly.
  FIX: Import through src/services/ layer instead.
  ALLOWED: src/api/ → src/services/ → src/repositories/ → src/models/
```

## CodeRabbit Review Loop

### Flow

1. Agent opens PR via `gh pr create`
2. CodeRabbit auto-reviews
3. Agent polls for comments: `gh api repos/{owner}/{repo}/pulls/{pr}/comments`
4. For each comment:
   - Actionable (code change needed) → fix, commit, push
   - Informational → reply inline via `gh api`
5. Push triggers CodeRabbit re-review
6. Loop up to `review.max_review_cycles` (3)
7. If approved or no new actionable comments → notify human
8. If max cycles reached → escalate with summary

### Boundaries

- Agent does NOT merge PRs. Human decision.
- Agent does NOT dismiss CodeRabbit's concerns. Escalates disagreements.
- Agent does NOT respond to comments it doesn't understand. Escalates.

### Setup

CodeRabbit must be installed on the GitHub repo separately. Bootstrap checks for its presence and reminds you if missing.

## Docs Structure

### After bootstrap

```
AGENTS.md                         # ~100 lines, table of contents + agent rules
docs/
├── architecture.md               # Layers, dependencies, data flow
├── conventions.md                # Coding style, imports, naming
├── domain.md                     # Core entities, relationships, business rules
├── testing.md                    # Test tiers, helpers, patterns
├── plans/
│   ├── active/                   # Current implementation plans
│   └── completed/                # Done plans (moved after merge)
└── decisions/                    # Decision records
    └── 001-initial-setup.md
```

### AGENTS.md structure

```markdown
# Project Name

Short description.

## Quick Reference
- Architecture: docs/architecture.md
- Conventions: docs/conventions.md
- Domain: docs/domain.md
- Testing: docs/testing.md
- Active plans: docs/plans/active/
- Decisions: docs/decisions/

## Commands
(from harness.yaml, listed for quick context)

## Agent Rules
1. Read before modifying
2. Run verification before committing
3. Follow docs/conventions.md
4. Update docs when architecture changes
5. ...
```

### Auto-split

When any doc exceeds `docs.max_doc_lines` (200), `/doc-split` activates during `/garden`:
- Splits by content headings into sub-docs in a directory
- Updates AGENTS.md pointers
- Deletes the oversized original

### Decision records

Non-obvious architectural choices get recorded in `docs/decisions/`:

```markdown
# 001: Use SQLite instead of Hive for local storage

## Context
Needed offline-first local database for mobile.

## Decision
SQLite via drift. Typed queries, migration support, good agent legibility.

## Alternatives considered
- Hive: faster for simple key-value, but no relational queries
- SharedPreferences: too limited

## Consequences
- Need build_runner for code generation
- Migration files must be maintained
```

## Human Intervention Points

Across the entire `/implement-feature` workflow, humans intervene at:

1. **Plan approval** — approve the plan before implementation starts
2. **Escalations** — verification failures after 3 retries, or unresolved review comments after 3 cycles
3. **Final merge** — decide when to merge the PR

Everything between is autonomous.

## Testing the Harness

Build the harness repo separately, then test on a fresh project (not the LiveStock Pricer). This validates that it truly works framework-agnostic without leaking project-specific assumptions.
