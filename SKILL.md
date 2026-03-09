---
name: agent-harness
description: A framework-agnostic harness for autonomous agent-driven development with Claude Code. Use when setting up new projects, implementing features with TDD, managing code quality, or maintaining codebases. Provides workflow automation, enforcement rules, context engineering, and cross-session memory.
---

# Agent Harness

A complete scaffolding system for autonomous agent-driven development. Humans steer (decisions, plans, merge). Agents execute (code, tests, review, cleanup, learning).

## When to Activate

Activate these skills when:
- Setting up a new project or installing the harness into an existing codebase
- Planning and implementing features with TDD
- Creating pull requests with quality gates
- Cleaning up code and managing documentation
- Running post-implementation or post-review retrospectives
- Updating an existing harness installation to the latest version

## Skill Bundles

### Bootstrap
Skills for installing and configuring the harness in any project.

- **install-harness** — Entry point: detects greenfield vs existing, copies skills, delegates to bootstrap
- **bootstrap-greenfield** — Interactive setup for new projects: questions, scaffold, docs, enforcement
- **bootstrap-existing** — Deep-scan existing codebases and adapt without restructuring
- **update-harness** — Pull latest skills into already-installed projects

### Workflow
End-to-end development automation from plan to merge-ready PR.

- **implement-feature** — Full orchestrator: plan, TDD implementation, verification, review, completion
- **create-plan** — Structured implementation plans from feature descriptions
- **create-tests** — Tier-aware test generation (unit/integration/e2e)
- **create-pr** — PR creation with blocking standards gate
- **create-qa** — QA checklist generation from diff analysis
- **retrospective** — Post-implementation and post-review learning capture to memory

### Maintenance
Codebase health and cleanup.

- **deslop** — Code cleanup: comments, naming, dead code, simplification
- **garden** — Entropy scan: file size, doc freshness, test gaps, scratch cleanup, memory freshness
- **doc-split** — Auto-split documents exceeding size threshold
- **project-structure-validator** — Architecture compliance validation agent

## Key Concepts

### Configuration-Driven
`harness.yaml` is the single source of truth. Every command, threshold, and rule lives here. Skills read from it at runtime — nothing is hardcoded.

### Context Engineering
The harness applies research-backed principles: cross-session memory (`memory/`), observation masking (`scratch/`), plan re-reading, compression triggers, attention-favored placement, phase isolation, and retrospective learning.

### Self-Healing Verification
Every commit passes a pre-commit gate (lint + test + enforcement). Failures are automatically diagnosed and fixed by the agent, up to a configurable retry limit.

## References

- [Context Management Guide](references/context-management-reference.md)
- [Architecture Reference](references/architecture-reference.md)
- [Conventions Reference](references/conventions-reference.md)
- [Testing Reference](references/testing-reference.md)
- [Doc Structure Reference](references/doc-structure-reference.md)
- [AGENTS.md Reference](references/agents-md-reference.md)
