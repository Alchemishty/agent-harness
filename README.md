# Agent Harness

A reusable, framework-agnostic harness for autonomous agent-driven development with Claude Code — built on context engineering principles so every project ships with best-in-class memory management, attention-aware design, and self-improving agent workflows.

## Why Agent Harness

Most AI coding setups throw an agent at a codebase with a system prompt and hope for the best. Agent Harness takes a different approach: it builds the **scaffolding** that makes agents produce reliable work over long sessions, across multiple features, and through review cycles.

**Humans steer** — decisions, plans, final merge.
**Agents execute** — code, tests, review responses, cleanup, and learning.

### What makes it different

**Framework-agnostic.** One blueprint works with Python, TypeScript, Dart, Go, Rust, Java — any language, any framework. Configuration drives everything; nothing is hardcoded.

**Self-healing verification.** Every commit passes through a pre-commit gate (lint + test + enforcement). When it fails, the agent reads the error, fixes the issue, and retries — no human attention needed until it's stuck.

**Context-aware by design.** The harness applies research-backed context engineering principles throughout: attention-favored information placement, progressive disclosure, observation masking, compression triggers, and cross-session memory. Agents don't just work — they work *well* over long sessions and *learn* across sessions.

**End-to-end automation.** A single `/implement-feature` command takes a feature from plan through TDD implementation, code review, cleanup, and merge-ready PR — with built-in quality gates at every phase.

## Quick Start

1. Open Claude Code in your project directory
2. Run: `/install-harness` (point it to this repo's location)
3. The skill will detect whether your project is greenfield or existing and guide you through setup

### Updating an Existing Installation

When the harness repo evolves with new skills or improvements:

1. Open Claude Code in your project directory
2. Run: `/update-harness`
3. The skill diffs your installed skills against the latest, shows what changed, and updates with your confirmation

Skills are replaced wholesale (they're harness-owned). Project-specific config (`harness.yaml`, `docs/`, `enforcement/`, `memory/`) is never overwritten.

## What Gets Installed

After running `/install-harness`, your project will contain:

```
harness.yaml              # Central config: commands, stack, enforcement rules
AGENTS.md                 # Table-of-contents entry point (~100 lines)
docs/
  architecture.md         # System architecture and layer definitions
  conventions.md          # Coding standards (the authority)
  domain.md               # Business domain glossary and rules
  testing.md              # Testing strategy and conventions
  plans/active/           # In-progress implementation plans
  plans/completed/        # Archived completed plans
  decisions/              # Numbered architectural decision records
memory/                   # Cross-session agent memory (patterns, fixes, preferences)
scratch/                  # Temporary working files (in .gitignore)
enforcement/              # Executable architecture rules
.claude/commands/         # All skills adapted to your project
.claude/settings.json     # Verification hooks
.github/workflows/ci.yml  # CI pipeline
```

## Skills

### Workflow Skills (installed into target projects)

| Skill | Purpose |
|---|---|
| `/implement-feature` | Full orchestrator: plan → TDD → verify → review → merge-ready |
| `/create-plan` | Structured implementation plan generator |
| `/create-tests` | Tier-aware test generator (unit/integration/e2e) |
| `/create-pr` | PR creation with blocking standards gate |
| `/create-qa` | QA checklist generated from diff |
| `/retrospective` | Post-implementation and post-review learning capture |
| `/deslop` | Code cleanup (comments, naming, dead code) |
| `/garden` | Entropy scan + cleanup (file size, docs freshness, test gaps, scratch cleanup, memory freshness) |
| `/doc-split` | Auto-split docs exceeding size threshold |
| `project-structure-validator` | Architecture compliance agent (`.claude/agents/`) |

### Bootstrap Skills (stay in this repo)

| Skill | Purpose |
|---|---|
| `/install-harness` | Entry point — detects greenfield vs existing, copies skills |
| `/bootstrap-greenfield` | Interactive setup for new projects |
| `/bootstrap-existing` | Deep-scan and adapt to existing codebases |
| `/update-harness` | Pull latest skills into an already-installed project |

## Architecture

### Configuration-Driven

`harness.yaml` is the single source of truth. Every command, threshold, and rule lives here — skills read from it at runtime. No hardcoded `pytest`, `npm test`, or `flutter test` anywhere. Switch stacks by changing config, not by rewriting skills.

### Four-Layer Enforcement

Projects use a universal layer model (Models → Repositories → Services → UI/API) with import direction enforcement. Layer N can only import from layers 0..N-1. Violations fail at both pre-commit and CI.

### Progressive Disclosure

`AGENTS.md` is a ~100 line table of contents. It links to `docs/` for depth. Docs auto-split when they exceed a configurable threshold. Agents load only what they need — no wasted context on irrelevant sections.

### Pre-Commit Verification Gate

Every commit passes enforcement, linting, and tests — or it's blocked. The agent reads the error output, fixes the issue, and retries (up to a configurable limit). After max retries, it stops and escalates to the human. Non-negotiable quality floor with zero human attention required for routine fixes.

## Context Engineering

The harness applies principles from [context engineering research](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering) to prevent the quality degradation that plagues long agent sessions.

### Memory System

```
memory/
  patterns.md         # Coding patterns confirmed across sessions
  fixes.md            # Recurring error solutions
  preferences.md      # User workflow preferences
  review-lessons.md   # Standards learned from code review
  domain.md           # Accumulated domain knowledge
```

Agents read `memory/` at session start and write back after completing features and review cycles. Knowledge compounds — the 10th feature implementation benefits from everything learned in the first 9.

**Temporal memory:** When entries become outdated, they're marked `[SUPERSEDED]` with a date and reason rather than deleted. This preserves *why* something changed, preventing re-learning forgotten lessons.

### Attention-Favored Placement

The U-shaped attention curve means models recall the beginning and end of context best (middle gets 10-40% lower recall). The harness applies this:

- `AGENTS.md` Agent Rules: most critical rules at positions 1-3 and 8-10
- Skill instructions: critical rules placed at the END (closest to where the agent generates responses)
- Implementation loop: plan re-reading at the start of each step re-anchors requirements in a high-attention position

### Observation Masking

Verbose tool output (tests, linting, builds, large diffs) is captured to `scratch/` files. Only compact summaries stay in conversation context. This drops tool output from 40-60% of context budget to 10-15%, leaving the rest for reasoning and code quality.

### Compression Triggers

From implementation step 4 onward, agents write structured state checkpoints to `scratch/session-state.md`. These serve as cheap re-orientation points (~200 tokens) and bootstrap data for sub-agents, preventing quality degradation in long sessions.

### Phase Isolation

For plans with 7+ steps, agents use sub-agents per phase — each gets a fresh context window. Sub-agents write results to files (avoiding the "telephone game" of lossy paraphrasing), and the orchestrator reads files directly for full-fidelity handoff.

### Decision Probes

After implementation completes, a plan adherence check verifies: every step was completed, all planned files exist, test files pass, conventions were followed, and open questions were resolved. This catches drift before it reaches code review.

### Retrospective Learning

`/retrospective` runs after implementation and after code review cycles. It captures:

- Patterns that worked → `memory/patterns.md`
- Recurring errors and fixes → `memory/fixes.md`
- Reviewer-enforced standards → `memory/review-lessons.md`
- Documentation gaps → proposed edits for user approval

Review feedback is treated as the highest-signal input: reviewers explicitly state team standards that may not be documented anywhere.

## Installation Methods

### As a Claude Code Plugin (recommended)

Install from the marketplace — no cloning needed. All skills are available as slash commands immediately.

The plugin is organized into three bundles:
- **agent-harness-bootstrap** — `/install-harness`, `/bootstrap-greenfield`, `/bootstrap-existing`, `/update-harness`
- **agent-harness-workflow** — `/implement-feature`, `/create-plan`, `/create-tests`, `/create-pr`, `/create-qa`, `/retrospective`
- **agent-harness-maintenance** — `/deslop`, `/garden`, `/doc-split`, `project-structure-validator`

### As a Git Clone (for customization)

Clone the repo and point `/install-harness` to its location. This allows forking and customizing skills for your team.

## Repo Structure

```
agent-harness/
├── .claude-plugin/          # Plugin marketplace definition
│   └── marketplace.json
├── SKILL.md                 # Plugin collection metadata
├── skills/                  # Claude Code skills (each in its own directory)
│   ├── implement-feature/
│   │   └── SKILL.md
│   ├── create-plan/
│   │   └── SKILL.md
│   ├── retrospective/
│   │   └── SKILL.md
│   └── ...                  # 14 skills total
├── references/              # "What good looks like" — templates and guides
│   ├── agents-md-reference.md
│   ├── architecture-reference.md
│   ├── conventions-reference.md
│   ├── testing-reference.md
│   ├── doc-structure-reference.md
│   └── context-management-reference.md
├── enforcement/             # Architecture enforcement framework + common rules
├── hooks/                   # Claude Code hook configuration references
└── test/                    # Dry-run checklists for validating the harness
```

## Philosophy

Inspired by OpenAI's harness engineering and grounded in context engineering research: the discipline of building scaffolding around AI agents — constraints, feedback loops, memory systems, and verification gates that let agents produce reliable work.

The core insight: **the scarce resource is human attention**. Everything in the harness is designed to minimize how much of it you spend. Agents handle routine decisions, fix their own verification failures, learn from code review feedback, and carry knowledge across sessions. Humans make the calls that matter: approving plans, merging PRs, and resolving genuinely ambiguous problems.
