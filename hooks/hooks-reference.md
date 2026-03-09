# Hooks Reference

Reference for configuring Claude Code verification hooks. Bootstrap skills read this when generating `.claude/settings.json` for a target project.

## What Are Claude Code Hooks?

Claude Code hooks are shell commands that run in response to agent tool calls. They intercept specific tool invocations and execute arbitrary commands before allowing the tool call to proceed.

Hooks are the mechanism that implements the pre-commit verification gate: when the agent attempts `git commit`, a hook fires that runs all verification checks (lint, test, enforcement). If any check fails, the commit is blocked and the error output is returned to the agent.

## The Pre-Commit Gate Pattern

The core pattern is: intercept `git commit` bash commands, run verification checks before allowing the commit through.

```
Agent works freely (editing files, reading code, running commands)
    ↓
Agent runs `git commit`
    ↓
Hook intercepts the Bash tool call
    ↓
Hook runs: enforcement checks + lint + tests
    ↓
┌─────────────┐     ┌──────────────┐
│ All pass     │     │ Any fail     │
│ → commit     │     │ → blocked    │
│   proceeds   │     │ → error      │
└─────────────┘     │   returned   │
                    │   to agent   │
                    └──────┬───────┘
                           ↓
                    Agent reads error,
                    fixes the issue,
                    tries to commit again
                           ↓
                    (repeats up to max_retries)
```

## Configuration

Hooks are configured in `.claude/settings.json` (project-level settings, committed to the repo). The relevant section is `hooks.PreToolUse`.

### Structure

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

The hook script receives a JSON object on stdin with `tool_input` (containing the command) and `session` (containing cwd). The script is responsible for filtering which commands to act on (e.g., only `git commit`).

### Fields

| Field | Purpose |
|-------|---------|
| `matcher` | Which tool to intercept. `"Bash"` matches the Bash tool that runs shell commands. |
| `hooks` | Array of hook definitions to run when the matcher matches. |
| `hooks[].type` | The hook type. Use `"command"` for shell commands. |
| `hooks[].command` | Path to a script that receives tool call context on stdin. If this command exits non-zero, the tool call is blocked. |

### How the hook script is built from harness.yaml

The bootstrap skill reads `verification.checks` from `harness.yaml` and generates a wrapper shell script (e.g., `.claude/hooks/pre-commit-checks.sh`) that:

1. Reads the JSON context from stdin
2. Extracts the command being run via `jq`
3. Only runs checks when the command matches `git commit`
4. Chains the check commands with `&&`

```yaml
# harness.yaml
verification:
  checks:
    - name: "enforcement"
      command: "bash enforcement/run-all.sh"
    - name: "lint"
      command: "ruff check src/"
    - name: "test"
      command: "pytest tests/ -x"
```

Becomes a script like:

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

if ! echo "$COMMAND" | grep -q 'git commit'; then
  exit 0
fi

CWD=$(echo "$INPUT" | jq -r '.session.cwd // "."')
cd "$CWD" || exit 2

bash enforcement/run-all.sh && ruff check src/ && pytest tests/ -x
```

The `&&` chaining means checks run in order and stop at the first failure. This is intentional -- the agent gets the earliest error first, which is typically the cheapest to fix (enforcement rules before lint, lint before tests).

### Conceptual example with placeholders

The bootstrap skills generate the actual config with a wrapper script from `harness.yaml`. The conceptual pattern is:

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

The wrapper script handles filtering (only `git commit` commands), directory resolution, and chaining the actual check commands from `harness.yaml → verification.checks`.

## What Happens on Hook Failure

When the hook command exits non-zero:

1. **The tool call is blocked.** The `git commit` does not execute.
2. **Error output is returned to the agent.** The agent receives the full stdout and stderr from the failed command.
3. **The agent reads the error and fixes it.** This is automatic behavior -- the agent sees the error, understands it (especially if enforcement rules include `FIX:` remediation lines), and makes corrections.
4. **The agent tries to commit again.** This triggers the hook again, running all checks from scratch.

There is no special configuration needed for the retry behavior. It is a natural consequence of the agent's workflow: see error, fix, retry. The hook fires every time `git commit` is attempted.

## Retry Behavior

The agent will retry up to `verification.max_retries` (default 3) times. This retry limit is enforced by the orchestrating skill (`/implement-feature`), not by the hook itself.

```
Attempt 1: git commit → hook fails (lint error) → agent fixes
Attempt 2: git commit → hook fails (test failure) → agent fixes
Attempt 3: git commit → hook fails (enforcement) → agent fixes
Attempt 4: git commit → hook passes → commit succeeds
```

If all retries are exhausted:

```
Attempt 1: git commit → hook fails → agent fixes
Attempt 2: git commit → hook fails → agent fixes
Attempt 3: git commit → hook fails → agent fixes
(max_retries reached)
→ Agent STOPS and escalates to human with error summary
```

The escalation prevents infinite fix-retry loops when the agent cannot resolve a fundamental issue.

## What the Bootstrap Skill Generates

During bootstrap, the skill:

1. Reads `verification.checks` from `harness.yaml`
2. Chains the check commands with `&&`
3. Writes `.claude/settings.json` with the hook configuration
4. Ensures the `.claude/` directory is committed to the repo (so all developers and agents get the same hooks)

### Example generated config (all languages share the same structure)

The `settings.json` is identical regardless of language — only the wrapper script contents differ:

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

The language-specific check commands go inside the wrapper script. Examples:

**Python project** — script runs:
```bash
bash enforcement/run-all.sh && ruff check src/ && pytest tests/ -x
```

**Flutter project** — script runs:
```bash
bash enforcement/run-all.sh && dart analyze lib/ && flutter test
```

**Go project** — script runs:
```bash
bash enforcement/run-all.sh && golangci-lint run && go test ./...
```

## Important Notes

- **Hooks are project-level, not user-level.** They live in `.claude/settings.json` which is committed to the repo. Every agent session on this repo gets the same verification gate.
- **Hooks do not replace CI.** CI is the final safety net. Hooks catch issues early during agent development; CI catches anything that slipped through.
- **The pattern match is substring-based.** `"git commit"` matches `git commit -m "message"`, `git commit --amend`, etc. This is intentional -- all commit variants should pass verification.
- **Keep hook commands fast.** The agent waits for the hook to complete before proceeding. If tests are slow, consider running only unit tests in the hook (`commands.test_unit`) and leaving full integration tests for CI.
- **Do not put interactive commands in hooks.** Hooks run non-interactively. Commands that prompt for input will hang.
