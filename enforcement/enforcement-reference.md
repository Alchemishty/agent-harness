# Enforcement Reference

How to write project-specific enforcement rules. Bootstrap skills read this guide when generating the `enforcement/` directory for a target project.

## Overview

Enforcement rules are executable scripts that validate architectural constraints. They run at two points in the development workflow:

1. **Pre-commit gate** -- triggered by Claude Code verification hooks when the agent attempts `git commit`
2. **CI** -- dedicated job that blocks PR merges on failure

Rules are the mechanism that prevents architectural drift. They encode decisions from `docs/conventions.md` and `docs/architecture.md` as automated checks that catch violations before they reach the codebase.

## Rule Structure

Each rule is a standalone executable script in the `enforcement/` directory. Scripts can be written in shell, Python, or the project's primary language.

```
enforcement/
├── run-all.sh                    # Orchestrator — runs every rule, exits non-zero if ANY fails
├── check-no-secrets.sh           # Universal rule
├── check-import-direction.sh     # Structural rule (project-specific)
├── check-file-size.sh            # Universal rule
└── check-no-raw-sql-handlers.sh  # Custom rule (added over time)
```

Naming convention: prefix all rule scripts with `check-` so they are easily identifiable and won't collide with `run-all.sh` or other utility scripts.

### Exit codes

- **Exit 0** -- rule passes, no violations found
- **Exit non-zero** -- rule fails, violations detected

### Error output format

Every rule MUST produce output in this format on failure. The format is designed so that agents can parse it and self-fix without human intervention:

```
ERROR: [rule-name]
  [file:line] [what's wrong]
  FIX: [exactly what to do to resolve this]
  REFERENCE: [docs/conventions.md#section if applicable]
```

Example:

```
ERROR: import-direction
  src/api/routes.py:14 imports from src/models/user.py directly
  FIX: Import through the services layer. Use 'from src.services.user_service import UserService' instead.
  REFERENCE: docs/architecture.md#layer-dependencies
```

Multiple violations in a single rule should each follow this format, one after another:

```
ERROR: no-secrets-in-code
  src/config.py:23 matches pattern: API_KEY = "sk-..."
  FIX: Move to environment variable. Never commit secrets. Use os.environ['API_KEY'] instead.
  REFERENCE: docs/conventions.md#environment-variables

ERROR: no-secrets-in-code
  lib/services/auth.dart:7 matches pattern: TOKEN = "ghp_..."
  FIX: Move to environment variable. Never commit secrets. Pass via --dart-define at build time.
  REFERENCE: docs/conventions.md#environment-variables
```

The `FIX:` line is critical. Without it, the agent cannot self-correct and will waste retries.

## run-all.sh

The orchestrator script runs every rule and aggregates results. It exits non-zero if ANY rule fails.

### Template

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
FAILED=0

# Run all check-*.sh scripts in this directory
for script in "$SCRIPT_DIR"/check-*.sh; do
  [ -f "$script" ] || continue
  echo "Running: $(basename "$script")"
  if ! bash "$script"; then
    FAILED=1
  fi
  echo ""
done

if [ "$FAILED" -ne 0 ]; then
  echo "ENFORCEMENT FAILED: one or more rules violated."
  echo "Fix the issues above and retry."
  exit 1
fi

echo "All enforcement rules passed."
exit 0
```

### Key properties

- Uses a glob pattern (`check-*.sh`) so adding a new rule only requires creating a new `check-*.sh` script. No need to edit `run-all.sh`.
- `set -euo pipefail` is used for the script itself, but individual rules are called with `if ! bash ...` so that a failing rule does not short-circuit the others.
- All rules run, even if early ones fail. This gives the agent the full picture on the first attempt.
- The final exit code reflects whether any rule failed.
- The `check-` prefix naming convention ensures only enforcement rule scripts run, not utility scripts or `run-all.sh` itself.

## When Rules Run

### Pre-commit gate (verification hooks)

The agent's Claude Code hooks intercept `git commit` and run verification checks before allowing the commit. One of those checks is:

```yaml
# harness.yaml
verification:
  checks:
    - name: "enforcement"
      command: "bash enforcement/run-all.sh"
```

On failure, the commit is blocked, the error output is returned to the agent, and the agent reads it and fixes the violations. This loops up to `verification.max_retries` (default 3) before escalating to a human.

### CI

A CI job runs the same `bash enforcement/run-all.sh` command. This catches anything that slipped through local checks (e.g., if hooks were bypassed).

## How to Add a New Rule

1. **Create the script** in `enforcement/`. Name it descriptively (e.g., `no-circular-imports.sh`).
2. **Implement the check.** Read config from `harness.yaml` if needed. Exit 0 on pass, non-zero on fail. Follow the error output format above.
3. **Add to run-all.sh.** Add a `run_rule "your-rule.sh"` line in the appropriate section.
4. **Update harness.yaml** if the rule reads any new config values from `enforcement:`.
5. **Document the rule** in `docs/conventions.md` so developers understand what is being enforced and why.

### Checklist for a well-written rule

- [ ] Exits 0 when no violations are found (including when there are no relevant files)
- [ ] Exits non-zero when violations are found
- [ ] Every violation includes `ERROR:`, the file and line, `FIX:`, and `REFERENCE:`
- [ ] Respects `.gitignore` exclusions (does not scan build artifacts, dependencies, generated code)
- [ ] Reads project-specific config from `harness.yaml` rather than hardcoding values
- [ ] Runs in under 10 seconds for a typical project (enforcement should be fast)
- [ ] Works on both macOS and Linux (CI environments)

## Rule Categories

### Universal rules

Same across all projects. Provided as common-rules in the harness:

- `no-secrets-in-code` -- regex scan for hardcoded secrets
- `file-size-limits` -- flag files exceeding line count threshold

### Structural rules

Generated per-project during bootstrap based on the architecture:

- `import-direction` -- enforce layer dependency direction
- `file-location` -- ensure files are in the correct directories per layer rules
- `barrel-completeness` -- ensure barrel/index files export all public modules

### Custom rules

Added over time as the project evolves. When a pattern of drift is discovered (e.g., during `/garden`), encode it as a new rule so it never recurs.

Examples:
- `no-raw-sql-in-handlers` -- business logic belongs in the service layer
- `no-platform-imports-in-core` -- platform-specific code stays out of shared modules
- `test-file-naming` -- test files must follow naming convention
