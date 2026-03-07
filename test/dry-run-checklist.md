# Agent Harness Dry-Run Checklist

Manual verification checklist for testing the harness on fresh projects.

## Test 1: Greenfield Project

### Setup
1. Create an empty directory and initialize git:
   ```bash
   mkdir /tmp/test-greenfield && cd /tmp/test-greenfield && git init
   ```
2. Open Claude Code in the directory
3. Run `/install-harness`

### Verify Installation Flow
- [ ] Skill asks for harness repo location (or auto-detects)
- [ ] Skill correctly detects greenfield (empty repo)
- [ ] Skill asks user to confirm greenfield classification
- [ ] Delegates to bootstrap-greenfield

### Verify Bootstrap Conversation
- [ ] Asks project purpose (open-ended)
- [ ] Asks language/framework (multiple choice)
- [ ] Asks platforms (multiSelect)
- [ ] Asks backend/database
- [ ] Asks deployment target
- [ ] Asks core domain entities
- [ ] Questions come one at a time (not all at once)

### Verify Generated Files
Pick a test stack: **Python + FastAPI + PostgreSQL**

#### Config
- [ ] `harness.yaml` exists at project root
- [ ] `harness.yaml` has correct stack (language: python, framework: fastapi)
- [ ] `harness.yaml` commands section has real commands (not placeholders)
- [ ] `harness.yaml` verification.checks references the correct lint/test commands

#### Documentation
- [ ] `AGENTS.md` exists at project root
- [ ] `AGENTS.md` is ~100 lines (not a monolith)
- [ ] `AGENTS.md` has Quick Reference linking to docs/ files
- [ ] `AGENTS.md` has all 10 Agent Rules
- [ ] `docs/architecture.md` exists and describes the project
- [ ] `docs/conventions.md` exists with language-appropriate patterns
- [ ] `docs/domain.md` exists with entities from conversation
- [ ] `docs/testing.md` exists with framework-appropriate patterns
- [ ] `docs/plans/active/` directory exists
- [ ] `docs/plans/completed/` directory exists
- [ ] `docs/decisions/001-initial-setup.md` exists

#### Skills
- [ ] `.claude/commands/implement-feature.md` exists
- [ ] `.claude/commands/create-plan.md` exists
- [ ] `.claude/commands/create-tests.md` exists
- [ ] `.claude/commands/create-pr.md` exists
- [ ] `.claude/commands/create-qa.md` exists
- [ ] `.claude/commands/deslop.md` exists
- [ ] `.claude/commands/garden.md` exists
- [ ] `.claude/commands/doc-split.md` exists
- [ ] `.claude/agents/project-structure-validator.md` exists
- [ ] Skills do NOT include install-harness, bootstrap-greenfield, bootstrap-existing

#### Enforcement
- [ ] `enforcement/run-all.sh` exists and is executable
- [ ] `enforcement/` has at least no-secrets check
- [ ] If import direction was configured: import-direction check exists
- [ ] File size limit check exists

#### Project Scaffold
- [ ] Directory structure matches agreed architecture
- [ ] At least one entry point file exists (even if minimal)
- [ ] `.github/workflows/ci.yml` exists with analyze + test jobs
- [ ] `.gitignore` exists and is appropriate for the language
- [ ] Test directories exist (mirroring source layout)

#### Verification Hooks
- [ ] `.claude/settings.json` has hook configuration (or instructions to set it up)

#### Git
- [ ] Everything committed with message "chore: bootstrap project with agent harness"
- [ ] Clean git status after bootstrap

---

## Test 2: Implement Feature (on greenfield project)

### Setup
Continue from Test 1 (greenfield Python + FastAPI project).

### Run
```
/implement-feature "add a health check endpoint at GET /health that returns {"status": "ok"}"
```

### Verify Workflow
- [ ] Plan generated in `docs/plans/active/`
- [ ] Plan presented for approval before implementation
- [ ] After approval: tests written first (TDD)
- [ ] Implementation follows the plan steps
- [ ] Pre-commit gate runs on commit attempt
- [ ] If lint fails: agent reads error, fixes, retries
- [ ] If test fails: agent reads error, fixes, retries
- [ ] All commits pass verification
- [ ] `/deslop` runs after implementation
- [ ] `/garden` runs after deslop
- [ ] `/create-qa` generates `qa/QA-<branch>.md`
- [ ] PR opened via `gh pr create`
- [ ] Agent waits for CodeRabbit review (or skips if not configured)

### Verify Output
- [ ] Health check endpoint works
- [ ] Tests pass
- [ ] Lint passes
- [ ] PR has meaningful title and body
- [ ] QA checklist has change-specific items

---

## Test 3: Existing Project

### Setup
1. Clone a small open source project:
   ```bash
   git clone https://github.com/tiangolo/fastapi /tmp/test-existing
   cd /tmp/test-existing
   ```
   (Or any small project with source code, tests, and a package manager config)
2. Open Claude Code in the directory
3. Run `/install-harness`

### Verify Installation Flow
- [ ] Skill correctly detects existing project (has source code)
- [ ] Skill asks user to confirm classification
- [ ] Delegates to bootstrap-existing

### Verify Scan
- [ ] Agent presents detected: language, framework, test tool, linter
- [ ] Agent shows directory structure summary
- [ ] Agent identifies existing tests and CI
- [ ] Agent identifies gaps (if any)

### Verify Generated Files
- [ ] `harness.yaml` reflects the ACTUAL project (not an ideal)
- [ ] `AGENTS.md` describes the project as-is
- [ ] `docs/architecture.md` reflects actual structure
- [ ] `docs/conventions.md` reflects patterns already in the codebase
- [ ] Existing source code is UNTOUCHED (no restructuring)
- [ ] Existing CI is UNTOUCHED (not overwritten)
- [ ] Enforcement rules may flag pre-existing issues (that's expected)

---

## Test 4: Edge Cases

- [ ] Running `/install-harness` twice warns about existing installation
- [ ] Harness works with a TypeScript + Next.js project (different stack)
- [ ] Harness works with a Go project (different stack)
- [ ] `/garden` correctly identifies oversized files
- [ ] `/doc-split` correctly splits a doc exceeding max_doc_lines
- [ ] `/create-tests` generates tests appropriate for the project's test framework
- [ ] Enforcement rules produce error messages with FIX: remediation
