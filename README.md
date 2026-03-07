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

## Repo Structure

```
agent-harness/
├── references/          # "What good looks like" — agent reads and adapts
├── skills/              # Claude Code commands (copied into target .claude/commands/)
├── enforcement/         # Architecture enforcement framework + common rules
├── hooks/               # Claude Code hook configuration references
└── docs/plans/          # Design doc and implementation plan
```

## Philosophy

Inspired by OpenAI's harness engineering: the discipline of building scaffolding around AI agents — constraints, feedback loops, and verification systems that let agents produce reliable work.

The core idea: engineers stop writing code and instead build the environment that lets agents write code reliably. The scarce resource is human attention, so everything is designed to minimize how much of it you spend.
