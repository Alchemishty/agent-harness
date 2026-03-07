# Agent Harness Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a reusable, framework-agnostic agent harness that can be installed into any project via a Claude Code skill, enabling autonomous agent-driven development with human intervention limited to decisions, planning, and final merge.

**Architecture:** Agent-Native approach — reference docs serve as examples of "what good looks like," Claude Code skills read `harness.yaml` for project-specific config, and the agent adapts everything to the target project through reasoning rather than string substitution. The harness is self-contained after installation.

**Tech Stack:** Claude Code (skills, hooks, agents), shell scripts (enforcement), YAML (config schema), Markdown (reference docs, skills)

**Design Doc:** `docs/plans/2026-03-07-agent-harness-design.md` (in LiveStock Pricer repo — copy to this repo in Task 1)

---

## Task 1: Repository Foundation

**Files:**
- Create: `README.md`
- Create: `.gitignore`
- Create: `harness.schema.yaml`
- Copy: `docs/plans/2026-03-07-agent-harness-design.md` (from LiveStock Pricer)

**Step 1: Create README.md**

```markdown
# Agent Harness

A reusable, framework-agnostic harness for autonomous agent-driven development with Claude Code.

## What This Is

A collection of reference docs, Claude Code skills, and enforcement frameworks that — when installed into any project — create a self-contained system where:

- **Humans** steer: decisions, plans, final merge
- **Agents** execute: code, tests, review responses, cleanup

## Quick Start

1. Open Claude Code in your project directory
2. Run: `/install-harness` (point it to this repo's location)
3. The skill will detect whether your project is greenfield or existing and guide you through setup

## What Gets Installed

After running `/install-harness`, your project will contain:

- `harness.yaml` — central config (commands, stack, enforcement rules)
- `AGENTS.md` — table-of-contents pointing to docs/
- `docs/` — architecture, conventions, domain, testing docs
- `.claude/commands/` — all skills adapted to your project
- `enforcement/` — executable architecture rules
- Verification hooks — pre-commit gate (lint + test + enforcement)

## Skills

| Skill | Purpose |
|---|---|
| `/install-harness` | Entry point — detects mode, delegates |
| `/implement-feature` | Orchestrator: plan → code → test → review → merge-ready |
| `/create-plan` | Implementation plan generator |
| `/create-tests` | Tier-aware test generator |
| `/create-pr` | PR creation with standards gate |
| `/create-qa` | QA checklist from diff |
| `/deslop` | Code cleanup |
| `/garden` | Entropy scan + cleanup |
| `/doc-split` | Auto-split large docs |

## Philosophy

Inspired by OpenAI's harness engineering: the discipline of building scaffolding around AI agents — constraints, feedback loops, and verification systems that let agents produce reliable work.
```

**Step 2: Create .gitignore**

```
.DS_Store
*.swp
*.swo
*~
.idea/
.vscode/
```

**Step 3: Create harness.schema.yaml**

This defines the full schema for `harness.yaml` with descriptions and defaults. The bootstrap skills reference this when generating a project's config.

```yaml
# harness.schema.yaml — defines the structure of harness.yaml
# Bootstrap skills read this to know what config to generate.
# Comments serve as documentation for the agent.

schema:
  version:
    type: integer
    default: 1
    description: "Schema version for forward compatibility"

  project:
    name:
      type: string
      required: true
      description: "Project name, used in commit messages and PR titles"
    description:
      type: string
      required: true
      description: "One-line project description"
    type:
      type: string
      enum: ["greenfield", "existing"]
      description: "How this project was bootstrapped"

  stack:
    language:
      type: string
      required: true
      description: "Primary language (python, dart, typescript, go, rust, etc.)"
    framework:
      type: string
      description: "Primary framework (fastapi, flutter, nextjs, gin, etc.)"
    runtime:
      type: string
      description: "Runtime version (3.12, 18.x, 1.21, etc.)"
    backend:
      type: string
      description: "Backend/database (postgresql, supabase, firebase, mongodb, etc.)"
    orm:
      type: string
      description: "ORM or database layer (sqlalchemy, drift, prisma, gorm, etc.)"
    test_framework:
      type: string
      required: true
      description: "Test framework (pytest, flutter_test, jest, go test, etc.)"
    linter:
      type: string
      required: true
      description: "Linter tool name (ruff, dart, eslint, golangci-lint, clippy, etc.)"
    formatter:
      type: string
      description: "Formatter command (ruff format, dart format, prettier, gofmt, etc.)"

  commands:
    install:
      type: string
      required: true
      description: "Install dependencies command"
    analyze:
      type: string
      required: true
      description: "Static analysis / lint command"
    format:
      type: string
      description: "Code formatting command"
    test:
      type: string
      required: true
      description: "Run all tests"
    test_unit:
      type: string
      description: "Run unit tests only"
    test_integration:
      type: string
      description: "Run integration tests only"
    build:
      type: string
      description: "Build command"
    migrate:
      type: string
      description: "Database migration command"
    codegen:
      type: string
      description: "Code generation command (if applicable)"

  platforms:
    type: array
    items:
      name:
        type: string
        description: "Platform name (api, web, mobile, cli)"
      entry:
        type: string
        description: "Entry point file path"
      targets:
        type: array
        description: "Build targets (docker, android, ios, web, binary)"
      deploy:
        type: string
        description: "Deploy command"

  docs:
    root:
      type: string
      default: "docs/"
    architecture:
      type: string
      default: "docs/architecture.md"
    conventions:
      type: string
      default: "docs/conventions.md"
    domain:
      type: string
      default: "docs/domain.md"
    testing:
      type: string
      default: "docs/testing.md"
    plans_active:
      type: string
      default: "docs/plans/active/"
    plans_completed:
      type: string
      default: "docs/plans/completed/"
    decisions:
      type: string
      default: "docs/decisions/"
    max_doc_lines:
      type: integer
      default: 200
      description: "Auto-split threshold. Docs exceeding this trigger /doc-split."

  enforcement:
    import_direction:
      enabled:
        type: boolean
        default: false
      layers:
        type: array
        description: "Ordered list of architectural layers (left-to-right dependency)"
      rule:
        type: string
        default: "left-to-right only"
    file_size_limit:
      type: integer
      default: 300
      description: "Max lines per source file before /garden flags it"
    no_secrets_pattern:
      type: string
      default: "(API_KEY|SECRET|PASSWORD|TOKEN)\\s*="
    custom_rules:
      type: string
      default: "enforcement/"
      description: "Directory containing project-specific rule scripts"

  review:
    provider:
      type: string
      default: "coderabbit"
      description: "External review agent (coderabbit, greptile, or none)"
    auto_address_comments:
      type: boolean
      default: true
    max_review_cycles:
      type: integer
      default: 3

  verification:
    max_retries:
      type: integer
      default: 3
    on_fail:
      type: string
      enum: ["fix", "escalate"]
      default: "fix"
    checks:
      type: array
      description: "Verification checks run at pre-commit gate"
      items:
        name:
          type: string
        command:
          type: string

  gardening:
    enabled:
      type: boolean
      default: true
    trigger:
      type: string
      default: "post-feature"
    checks:
      type: array
      default:
        - "unused imports and exports"
        - "files exceeding file_size_limit"
        - "docs matching actual code structure"
        - "test coverage gaps"
```

**Step 4: Copy the design doc**

```bash
cp "/Volumes/Samsung T7/Livestock-Pricing-App/docs/plans/2026-03-07-agent-harness-design.md" \
   "/Volumes/Samsung T7/agent-harness/docs/plans/"
```

**Step 5: Commit**

```bash
git add -A
git commit -m "chore: initialize agent-harness repo with foundation files"
```

---

## Task 2: Reference Docs — AGENTS.md and Doc Structure

**Files:**
- Create: `references/agents-md-reference.md`
- Create: `references/doc-structure-reference.md`

**Step 1: Write agents-md-reference.md**

This is the reference example of a well-structured AGENTS.md. The bootstrap skills read this and generate a project-specific version. It demonstrates the table-of-contents pattern (~100 lines, pointers to docs/).

Write it as a generic template with commentary explaining what each section does and why it's structured that way. Include `<!-- ADAPT: ... -->` comments where the agent should fill in project-specific content.

Key sections:
- Project name and one-line description
- Quick Reference (links to docs/ files)
- Commands (from harness.yaml)
- Agent Rules (8-10 universal rules that apply to every project)

The agent rules should be universal:
1. Read before modifying
2. Run verification before committing
3. Follow docs/conventions.md
4. Update docs when architecture changes
5. Check harness.yaml for commands — never hardcode tool invocations
6. Escalate to human when stuck after 3 retries
7. Write tests before implementation
8. Keep commits atomic — one logical change per commit
9. Record non-obvious decisions in docs/decisions/
10. Never commit secrets or credentials

**Step 2: Write doc-structure-reference.md**

This explains the docs/ directory convention:
- Why shallow-first (progressive disclosure)
- What each standard doc covers
- When and how docs get split (max_doc_lines threshold)
- How plans/ and decisions/ directories work
- Naming conventions for files

**Step 3: Commit**

```bash
git add references/
git commit -m "docs: add AGENTS.md and doc structure references"
```

---

## Task 3: Reference Docs — Architecture, Conventions, Testing

**Files:**
- Create: `references/architecture-reference.md`
- Create: `references/conventions-reference.md`
- Create: `references/testing-reference.md`

**Step 1: Write architecture-reference.md**

Reference example of a good architecture doc. Shows:
- System overview diagram (ASCII)
- Layer definitions with dependency rules
- Data flow description
- External service integrations
- Entry points

All with `<!-- ADAPT: ... -->` comments. The agent reads this and generates a project-specific version during bootstrap.

**Step 2: Write conventions-reference.md**

Reference example of coding conventions. Covers:
- Import ordering and style
- Naming conventions (files, classes, functions, variables)
- Error handling patterns
- Serialization patterns
- Code smells table (smell → symptom → fix)

Framework-agnostic but with examples showing how different languages handle each convention.

**Step 3: Write testing-reference.md**

Reference example of a testing guide. Covers:
- Test tier definitions (unit, integration, e2e) — framework-agnostic
- File location mirroring convention
- Test helper patterns (factories, fixtures, mocks)
- What to test at each tier
- Deciding which tier (decision matrix)
- Good test principles (behavior not implementation, one concept per test, descriptive names)

**Step 4: Commit**

```bash
git add references/
git commit -m "docs: add architecture, conventions, and testing references"
```

---

## Task 4: Core Skills — install-harness

**Files:**
- Create: `skills/install-harness.md`

**Step 1: Write install-harness.md**

This is the entry point skill. When a user runs `/install-harness` in any project, it:

1. Asks where the agent-harness repo is located (or auto-detects if sibling directory)
2. Reads the harness repo's references/, skills/, enforcement/, hooks/
3. Detects if the target project is greenfield or existing:
   - Greenfield: <5 files, no package manager config, or user confirms
   - Existing: has source code, package files, git history
4. Delegates to `/bootstrap-greenfield` or `/bootstrap-existing`

The skill should:
- Be clear about what it will create in the target project
- List all files that will be added
- Ask for confirmation before writing anything
- Reference the harness.schema.yaml for config generation

**Step 2: Commit**

```bash
git add skills/install-harness.md
git commit -m "feat: add install-harness entry point skill"
```

---

## Task 5: Core Skills — bootstrap-greenfield

**Files:**
- Create: `skills/bootstrap-greenfield.md`

**Step 1: Write bootstrap-greenfield.md**

The greenfield bootstrap skill. Guides a conversation to set up a brand new project:

1. Ask questions one at a time:
   - What does this project do?
   - What language/framework?
   - What platforms?
   - What backend/database?
   - What deployment target?
   - What are the core domain entities?
2. Generate harness.yaml from answers (reading harness.schema.yaml for structure)
3. Generate AGENTS.md (reading agents-md-reference.md, adapting to project)
4. Generate docs/ directory:
   - architecture.md (from architecture-reference.md)
   - conventions.md (from conventions-reference.md)
   - domain.md (from conversation about entities)
   - testing.md (from testing-reference.md)
   - plans/active/, plans/completed/, decisions/
5. Scaffold project structure:
   - Directory layout matching architecture
   - Minimal entry points
   - CI config (.github/workflows/ci.yml)
   - .gitignore
   - Empty test directories mirroring source layout
6. Copy and adapt all skills into .claude/commands/
7. Generate enforcement rules from agreed architecture
8. Configure verification hooks
9. Present summary for approval
10. Commit everything

The skill must reference the harness repo path (passed from install-harness) to read reference docs.

**Step 2: Commit**

```bash
git add skills/bootstrap-greenfield.md
git commit -m "feat: add bootstrap-greenfield skill"
```

---

## Task 6: Core Skills — bootstrap-existing

**Files:**
- Create: `skills/bootstrap-existing.md`

**Step 1: Write bootstrap-existing.md**

The existing project bootstrap skill:

1. Scan the project:
   - Package files (pubspec.yaml, package.json, pyproject.toml, go.mod, Cargo.toml, etc.)
   - Directory structure (tree)
   - Existing tests, CI configs, docs
   - Git log (recent commit patterns)
   - .gitignore contents
2. Present findings as a summary:
   - Detected: language, framework, test tool, linter, formatter
   - Found: N test files, CI config (yes/no), existing docs (yes/no)
   - Structure: describe what it sees
3. Propose opinionated defaults based on detected stack
4. Ask targeted questions:
   - Confirm or override detected tools
   - Suggest architectural layer separation (with options)
   - Identify gaps (missing tests, no CI, no docs)
5. Generate all outputs (same as greenfield) but document reality:
   - AGENTS.md describes the codebase as-is
   - Architecture doc reflects actual structure, notes improvements for later
   - Does NOT restructure existing code
6. Present summary for approval
7. Commit

Key difference from greenfield: documents what exists, doesn't impose ideal structure. Improvements come through the normal plan → implement cycle.

**Step 2: Commit**

```bash
git add skills/bootstrap-existing.md
git commit -m "feat: add bootstrap-existing skill"
```

---

## Task 7: Core Skills — implement-feature (The Orchestrator)

**Files:**
- Create: `skills/implement-feature.md`

**Step 1: Write implement-feature.md**

This is the most critical skill. It drives autonomous feature development:

1. Accept input: plan file path OR natural language description
2. If no plan exists, invoke /create-plan first, wait for human approval
3. Read the plan, extract ordered steps
4. For each step:
   a. Write/update tests (read harness.yaml for test command and framework)
   b. Implement code
   c. Stage changes and attempt commit
   d. Pre-commit verification gate fires:
      - Read harness.yaml → verification.checks
      - Run each check (lint, test, enforcement)
      - If any fail: read error output, fix, retry (up to verification.max_retries)
      - If still failing: STOP and escalate to human with error summary
   e. On successful commit, mark step done in plan file
5. After all steps complete:
   a. Run enforcement checks (full pass, not just changed files)
   b. Run /deslop (clean up code)
   c. Run /garden (entropy scan, doc freshness, coverage gaps)
   d. Run /create-qa (generate QA checklist)
6. Push branch, open PR via `gh pr create`
7. CodeRabbit review loop:
   a. Wait, then poll: `gh api repos/{owner}/{repo}/pulls/{pr_number}/comments`
   b. For each actionable comment: fix, commit, push
   c. For informational comments: reply via gh api
   d. Loop up to review.max_review_cycles
   e. If approved or no new actionable comments: notify human for merge
   f. If max cycles reached: escalate with summary
8. After merge (human does this), move plan to docs/plans/completed/

The skill must handle:
- Reading harness.yaml for ALL project-specific commands
- Never hardcoding any tool or command
- Clean escalation with context when stuck
- Updating the plan file as progress is made

**Step 2: Commit**

```bash
git add skills/implement-feature.md
git commit -m "feat: add implement-feature orchestrator skill"
```

---

## Task 8: Core Skills — create-plan

**Files:**
- Create: `skills/create-plan.md`

**Step 1: Write create-plan.md**

Framework-agnostic implementation plan generator:

1. Read harness.yaml for commands and stack info
2. Read docs/architecture.md for structure
3. Read docs/conventions.md for patterns
4. Read docs/domain.md for business context
5. Read docs/decisions/ for past architectural choices
6. Generate plan in docs/plans/active/YYYY-MM-DD-<topic>.md
7. Plan skeleton:
   - Purpose / Big Picture
   - Assumptions
   - Open Questions (resolve before implementing)
   - Context and Orientation (files to read, patterns to follow)
   - Steps (ordered, each with: files to touch, what to do, how to verify)
   - Validation (how to know it's done)
   - Decision Log (updated as work progresses)
8. Present plan for human approval before any implementation

Commands referenced in the plan come from harness.yaml, never hardcoded.

**Step 2: Commit**

```bash
git add skills/create-plan.md
git commit -m "feat: add create-plan skill"
```

---

## Task 9: Core Skills — create-tests, create-pr, create-qa

**Files:**
- Create: `skills/create-tests.md`
- Create: `skills/create-pr.md`
- Create: `skills/create-qa.md`

**Step 1: Write create-tests.md**

Tier-aware test generator:
1. Read harness.yaml for test framework, paths, commands
2. Read docs/testing.md for project test patterns and helpers
3. Inspect changes (git diff)
4. Classify changed files by test tier
5. Write tests following patterns from docs/testing.md
6. Run tests (harness.yaml → commands.test)
7. Verify they pass
8. Report coverage summary

All framework-specific details come from harness.yaml and docs/testing.md, not hardcoded.

**Step 2: Write create-pr.md**

PR creation with standards gate:
1. Run harness.yaml → commands.analyze (blocking gate — stop if fails)
2. Run harness.yaml → commands.test (blocking gate)
3. Run enforcement checks (blocking gate)
4. If all pass: inspect changes, generate PR title + body
5. PR body includes: summary, changes list, QA checklist reference, test coverage
6. Push branch, create PR via `gh pr create`

Standards gate must block — the skill refuses to create the PR if verification fails.

**Step 3: Write create-qa.md**

QA checklist generator:
1. Read git diff for changed files
2. Read docs/domain.md for domain entities and categories
3. Generate qa/QA-<branch-name>.md with:
   - Change-specific items (from the diff, most important)
   - Domain-category items (from docs/domain.md)
4. Checklist format with checkboxes

Domain categories come from docs/domain.md, not hardcoded.

**Step 4: Commit**

```bash
git add skills/create-tests.md skills/create-pr.md skills/create-qa.md
git commit -m "feat: add create-tests, create-pr, create-qa skills"
```

---

## Task 10: Core Skills — deslop, garden, doc-split

**Files:**
- Create: `skills/deslop.md`
- Create: `skills/garden.md`
- Create: `skills/doc-split.md`

**Step 1: Write deslop.md**

Code cleanup skill. This is already framework-agnostic. Port from the LiveStock Pricer version with minimal changes:
- Remove unnecessary comments (obvious, "what" instead of "why", outdated)
- Simplify code (rename for clarity, extract functions, early returns, remove dead code)
- Philosophy: if a comment is needed to understand, the code should be clearer instead

**Step 2: Write garden.md**

Entropy management skill. Reads harness.yaml for thresholds:
1. Check files exceeding enforcement.file_size_limit
2. Check docs/ matches actual code structure
3. Check for unused exports, dead code patterns
4. Check test coverage gaps (source files with no corresponding test)
5. Check docs freshness (do architecture.md sections match actual directories?)
6. For fixable issues: make the fix in a cleanup commit
7. For issues needing human input: report them
8. Trigger /doc-split if any doc exceeds docs.max_doc_lines

**Step 3: Write doc-split.md**

Auto-split skill:
1. Read harness.yaml → docs.max_doc_lines
2. Find docs exceeding threshold
3. For each oversized doc:
   - Analyze content headings
   - Split into sub-docs in a new directory (same name as original file)
   - Update AGENTS.md pointers
   - Delete original file
4. Commit the split

**Step 4: Commit**

```bash
git add skills/deslop.md skills/garden.md skills/doc-split.md
git commit -m "feat: add deslop, garden, doc-split skills"
```

---

## Task 11: Enforcement Framework

**Files:**
- Create: `enforcement/enforcement-reference.md`
- Create: `enforcement/common-rules/no-secrets-in-code.md`
- Create: `enforcement/common-rules/import-direction.md`
- Create: `enforcement/common-rules/file-size-limits.md`

**Step 1: Write enforcement-reference.md**

How to write project-specific enforcement rules:
- Each rule is a script (shell, python, or project language) in enforcement/
- Scripts exit 0 on pass, non-zero on fail
- Error output MUST include remediation instructions (so the agent can self-fix)
- A `run-all.sh` script runs all rules in sequence
- Rules are called by verification hooks and CI

Format for error messages:
```
ERROR: [rule-name]
  [file:line] [what's wrong]
  FIX: [exactly what to do]
  REFERENCE: [link to docs/conventions.md section if applicable]
```

**Step 2: Write no-secrets-in-code.md**

Reference for the no-secrets rule:
- Regex pattern from harness.yaml → enforcement.no_secrets_pattern
- Scans all source files (respects .gitignore)
- Reports file:line with the match
- Remediation: "Move to environment variable, reference via config"

**Step 3: Write import-direction.md**

Reference for import direction enforcement:
- Reads harness.yaml → enforcement.import_direction.layers
- Checks that imports only go left-to-right in the layer list
- Language-specific import parsing (examples for Python, TypeScript, Dart, Go)
- Remediation includes the allowed dependency chain

**Step 4: Write file-size-limits.md**

Reference for file size enforcement:
- Reads harness.yaml → enforcement.file_size_limit
- Counts lines per source file
- Reports files exceeding limit
- Remediation: "Split into smaller modules. See docs/conventions.md for patterns."

**Step 5: Commit**

```bash
git add enforcement/
git commit -m "feat: add enforcement framework and common rules"
```

---

## Task 12: Hooks Reference

**Files:**
- Create: `hooks/hooks-reference.md`

**Step 1: Write hooks-reference.md**

Reference for setting up Claude Code verification hooks:
- How Claude Code hooks work (settings.json configuration)
- Pre-commit gate pattern:
  - Intercept git commit commands
  - Run verification.checks from harness.yaml
  - On failure: return error output to agent context
  - Agent sees errors, fixes, retries automatically
- How the bootstrap skills generate the hooks config
- Example hooks configuration for different project types

**Step 2: Commit**

```bash
git add hooks/
git commit -m "feat: add hooks reference documentation"
```

---

## Task 13: Project Structure Validator Agent

**Files:**
- Create: `skills/project-structure-validator.md` (as a .claude/agents/ file in target projects)

Note: This is stored in skills/ in the harness repo but gets installed to .claude/agents/ in target projects.

**Step 1: Write project-structure-validator.md**

Framework-agnostic structure validator:
1. Run tree command to visualize project
2. Read harness.yaml for expected structure
3. Read docs/architecture.md for layer definitions
4. Compare actual vs expected
5. Report violations with fix instructions
6. Optionally auto-fix (move files, create missing directories, update barrel files)

The validator reads all project-specific info from harness.yaml and docs/, never hardcodes paths.

**Step 2: Commit**

```bash
git add skills/project-structure-validator.md
git commit -m "feat: add project structure validator agent"
```

---

## Task 14: Integration Test — Dry Run

**Files:**
- Create: `test/dry-run-checklist.md`

**Step 1: Write dry-run-checklist.md**

A manual verification checklist for testing the harness on a fresh project:

1. Create empty directory, `git init`
2. Run `/install-harness` pointing to agent-harness repo
3. Verify: detects greenfield, launches bootstrap-greenfield
4. Go through the conversation (pick Python + FastAPI as test stack)
5. Verify all outputs:
   - [ ] harness.yaml exists and is valid
   - [ ] AGENTS.md exists, ~100 lines, has pointers to docs/
   - [ ] docs/ directory has architecture.md, conventions.md, domain.md, testing.md
   - [ ] docs/plans/active/ and docs/plans/completed/ exist
   - [ ] docs/decisions/001-initial-setup.md exists
   - [ ] .claude/commands/ has all skills
   - [ ] enforcement/ has project-specific rules + run-all.sh
   - [ ] .github/workflows/ci.yml exists
   - [ ] Project scaffold directories exist
   - [ ] .gitignore exists
6. Run `/implement-feature` with a simple task ("add a health check endpoint")
7. Verify:
   - [ ] Plan generated and presented for approval
   - [ ] Tests written before implementation
   - [ ] Pre-commit gate catches lint errors
   - [ ] Agent fixes lint errors and retries
   - [ ] All commits pass verification
   - [ ] /deslop runs
   - [ ] /garden runs
   - [ ] /create-qa generates checklist
   - [ ] PR opened via gh
8. Test existing project flow:
   - Clone a small open source project
   - Run `/install-harness`
   - Verify: detects existing, scans correctly, documents reality

**Step 2: Commit**

```bash
git add test/
git commit -m "docs: add dry-run integration test checklist"
```

---

## Task 15: Final Polish and Initial Commit

**Step 1: Review all files**

Read every file in the repo. Check for:
- Consistency between skills (do they all read harness.yaml the same way?)
- Cross-references are valid (skills referencing other skills by correct name)
- No hardcoded language/framework assumptions anywhere
- Error messages include remediation instructions
- README accurately describes what's in the repo

**Step 2: Run any fixes needed**

**Step 3: Final commit**

```bash
git add -A
git commit -m "chore: polish and finalize v1 of agent harness"
```

**Step 4: Create GitHub repo and push**

```bash
gh repo create agent-harness --private --source=. --push
```

---

## Execution Notes

- Tasks 1-3 are foundation (repo, references). Must be done first and in order.
- Tasks 4-6 are the bootstrap skills. Task 4 (install-harness) must come before 5-6 since it delegates to them.
- Tasks 7-10 are the operational skills. Task 7 (implement-feature) is the most complex and critical.
- Task 11-12 are enforcement and hooks. Can be done in parallel with skills.
- Task 13 is the structure validator. Depends on harness.yaml schema (Task 1).
- Task 14 is the dry-run test. Must be last before polish.
- Task 15 is final review and push.
