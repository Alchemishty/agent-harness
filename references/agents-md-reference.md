# <!-- ADAPT: Project Name --> Development Guide

<!-- ADAPT: 2-3 sentence description of what the project is, who uses it, and its core value proposition. -->

## Quick Reference

| Topic | Doc |
|-------|-----|
| Architecture & data flow | [docs/architecture.md](docs/architecture.md) |
| Coding conventions | [docs/conventions.md](docs/conventions.md) |
| Domain concepts & glossary | [docs/domain.md](docs/domain.md) |
| Testing strategy | [docs/testing.md](docs/testing.md) |
| Active plans | [docs/plans/active/](docs/plans/active/) |
| Decision records | [docs/decisions/](docs/decisions/) |

## Tech Stack

<!-- ADAPT: Table of layer → technology pairs used in the project. Example: -->
<!-- | Layer | Tech | -->
<!-- |-------|------| -->
<!-- | Framework | Next.js 14 | -->
<!-- | Database | PostgreSQL via Prisma | -->
<!-- | Auth | NextAuth.js | -->

## Commands

<!-- ADAPT: These should mirror the commands defined in harness.yaml. -->
<!-- The harness reads these from harness.yaml at runtime; keep them in sync. -->

```bash
# <!-- ADAPT: install dependencies command -->
# <!-- ADAPT: run dev server command -->
# <!-- ADAPT: run linter / static analysis command -->
# <!-- ADAPT: run tests command -->
# <!-- ADAPT: build for production command -->
```

## Project Structure

<!-- ADAPT: Tree diagram of the top-level directory layout. -->
<!-- Focus on the directories an agent needs to navigate daily. -->
<!-- Annotate each node with a short purpose comment. -->

```
<!-- ADAPT: project tree here -->
```

## Architecture Overview

<!-- ADAPT: Brief summary (5-10 lines) of how the system fits together. -->
<!-- Point to docs/architecture.md for the full version. -->

See [docs/architecture.md](docs/architecture.md) for diagrams and detailed data flow.

## Key Conventions

<!-- ADAPT: List the 3-5 most critical conventions an agent must know immediately. -->
<!-- Point to docs/conventions.md for the full set. -->

See [docs/conventions.md](docs/conventions.md) for the complete guide.

## Domain Concepts

<!-- ADAPT: List the 3-5 core domain terms with one-line definitions. -->
<!-- Point to docs/domain.md for the full glossary. -->

See [docs/domain.md](docs/domain.md) for the full glossary and business rules.

## Testing

<!-- ADAPT: Summarize the testing approach in 3-5 lines. -->
<!-- Point to docs/testing.md for details on fixtures, mocks, and coverage targets. -->

See [docs/testing.md](docs/testing.md) for the full testing strategy.

## Agent Rules

1. **Read before modifying** — always read a file before editing it
2. **Run verification before committing** — never skip the pre-commit gate
3. **Follow docs/conventions.md** — consistency over preference
4. **Update docs when architecture changes** — keep docs/ in sync with reality
5. **Check harness.yaml for commands** — never hardcode tool invocations
6. **Escalate to human when stuck after 3 retries** — don't spin endlessly
7. **Write tests before implementation** — TDD by default
8. **Keep commits atomic** — one logical change per commit
9. **Record non-obvious decisions in docs/decisions/** — future agents need context
10. **Never commit secrets or credentials** — use environment variables
