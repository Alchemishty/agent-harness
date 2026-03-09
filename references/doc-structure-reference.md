# docs/ Directory Convention

Every project bootstrapped by the harness follows this documentation structure. The goal is progressive disclosure: an agent reads AGENTS.md first (the table of contents), then drills into specific docs only when it needs depth.

## Why Progressive Disclosure

Large AGENTS.md files cause two problems. Agents waste context window on sections irrelevant to their current task, and maintainers stop updating a monolith because every edit risks breaking something else. Splitting into focused documents solves both: agents load only what they need, and each doc has a clear owner and scope.

AGENTS.md stays short (~100 lines) and acts purely as a table of contents with quick-reference summaries. When an agent needs detail, it follows a link into docs/.

## Standard Documents

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
memory/
scratch/           (in .gitignore — not committed)
```

### architecture.md

System-level view of the project. Covers:

- High-level component diagram (ASCII or Mermaid)
- Data flow between components
- External service dependencies
- Entry points and build targets
- Directory-to-responsibility mapping

Update this file whenever you add a new service, change the database schema, or restructure modules.

### conventions.md

The coding standards document. Covers:

- Import ordering and style
- Naming conventions (files, variables, classes, database columns)
- Error handling patterns
- State management patterns
- File and directory naming rules
- Formatting and linter configuration

This is the single source of truth for "how we write code here." Rule 3 in the Agent Rules points here. When an agent is unsure how to structure something, conventions.md is the authority.

### domain.md

Business domain knowledge that does not belong in code comments. Covers:

- Glossary of domain terms with precise definitions
- Business rules and invariants
- Entity relationships from the domain perspective (not the database perspective)
- Edge cases and known constraints
- User roles and their permissions

This file bridges the gap between what the product owner knows and what the codebase expresses.

### testing.md

Testing strategy and practical guidance. Covers:

- Test runner commands and configuration
- Directory layout for tests (unit, integration, e2e)
- Mocking and fixture conventions
- Coverage targets and how to measure them
- How to write a test for each layer of the architecture
- CI gate requirements

## plans/ Directory

Plans track multi-step work that spans more than one commit.

### plans/active/

Contains in-progress plans. Each plan is a markdown file named descriptively:

```
plans/active/add-goat-pricing.md
plans/active/migrate-to-riverpod-v3.md
```

A plan file contains:

- **Goal**: one sentence describing the outcome
- **Steps**: numbered checklist of tasks (use `- [ ]` / `- [x]`)
- **Context**: why this work is happening, links to issues or discussions
- **Constraints**: anything the implementer must not break

When all steps are checked off, move the file to plans/completed/ with the completion date prepended to the filename.

### plans/completed/

Archive of finished plans. Filename format: `YYYY-MM-DD-original-name.md`.

```
plans/completed/2026-02-15-add-goat-pricing.md
plans/completed/2026-03-01-migrate-to-riverpod-v3.md
```

Never delete completed plans. They serve as a record of what was done and why.

## decisions/ Directory

Numbered decision records capture non-obvious choices. Filename format:

```
decisions/001-use-drift-for-local-db.md
decisions/002-manual-riverpod-no-codegen.md
decisions/003-snake-case-json-keys.md
```

Each decision record contains:

- **Status**: accepted, superseded, or deprecated
- **Context**: the situation that prompted a decision
- **Decision**: what was chosen
- **Alternatives considered**: what was rejected and why
- **Consequences**: known tradeoffs of this choice

Number decisions sequentially. Never reuse a number. If a decision is reversed, mark the original as superseded and create a new record that references it.

## memory/ Directory

Cross-session agent memory. Agents write learned patterns, recurring fixes, user preferences, and accumulated domain knowledge here. Files persist across sessions and are read at session startup to bootstrap context.

Organize by topic, not chronologically:

```
memory/
  patterns.md       # Confirmed coding patterns and conventions
  fixes.md          # Solutions to recurring problems
  preferences.md    # User workflow and tool preferences
  domain.md         # Domain knowledge accumulated over sessions
```

**What to write:** Patterns confirmed across multiple sessions, solutions to recurring problems, explicit user preferences, verified domain knowledge.

**What NOT to write:** Session-specific state, speculative conclusions, anything already in docs/ (don't duplicate), incomplete information.

When an entry becomes outdated, mark it `[SUPERSEDED]` with a date and reason rather than deleting — this preserves history so future agents understand why something changed. Check for existing entries before writing new ones.

See [references/context-management-reference.md](../references/context-management-reference.md) for the full context management guide.

## scratch/ Directory

Temporary working directory for verbose tool output (test results, lint output, build logs, enforcement checks). Agents write full output here and keep only compact summaries in conversation context. This prevents context window bloat during long sessions.

```
scratch/
  last-test-run.log
  last-lint-run.log
  last-enforcement.log
```

**Lifecycle:** Files are overwritten on each run, never accumulated. The `/garden` skill cleans scratch/ during post-feature cleanup. The directory is listed in `.gitignore` and never committed.

## Auto-Split Behavior

The harness monitors document size via the `docs.max_doc_lines` setting in harness.yaml. When a document exceeds this threshold during a bootstrap or update operation, the harness splits it into sub-documents and replaces the original with a short index file that links to the parts.

For example, if conventions.md grows past the threshold:

```
docs/
  conventions.md            # Now just an index with links
  conventions/
    imports.md
    naming.md
    error-handling.md
    state-management.md
```

The index file (conventions.md) retains its original path so that all existing links in AGENTS.md continue to work. Agents follow the sub-links from the index when they need a specific section.

The default threshold is 200 lines, but each project can override it in harness.yaml:

```yaml
docs:
  max_doc_lines: 200
```

## Naming Conventions

- All doc filenames use **lowercase kebab-case**: `architecture.md`, `error-handling.md`
- Plan filenames are descriptive and action-oriented: `add-user-export.md`, not `user-stuff.md`
- Decision filenames are zero-padded and descriptive: `001-use-postgres.md`
- Sub-document directories match their parent file name: `conventions.md` splits into `conventions/`

## How AGENTS.md Stays Current

AGENTS.md is the entry point. It must always reflect the current state of docs/. When you:

- **Add a new doc**: add a row to the Quick Reference table in AGENTS.md
- **Rename a doc**: update the corresponding link in AGENTS.md
- **Remove a doc**: remove its row from AGENTS.md
- **Split a doc**: no change needed (the original path still resolves to the index)

The harness bootstrap skill generates AGENTS.md from a template (see references/agents-md-reference.md) and populates the Quick Reference table based on which docs/ files exist. After bootstrap, maintaining the table is the responsibility of the agent making changes.
