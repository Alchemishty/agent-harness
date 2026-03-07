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

Hooks are configured in `.claude/settings.json` (project-level settings, committed to the repo). The relevant section is `hooks.PreToolCall`.

### Structure

```json
{
  "hooks": {
    "PreToolCall": [
      {
        "matcher": "Bash",
        "pattern": "git commit",
        "command": "bash enforcement/run-all.sh && ruff check src/ && pytest tests/ -x"
      }
    ]
  }
}
```

### Fields

| Field | Purpose |
|-------|---------|
| `matcher` | Which tool to intercept. `"Bash"` matches the Bash tool that runs shell commands. |
| `pattern` | Regex or substring to match within the tool call arguments. `"git commit"` matches any bash command containing `git commit`. |
| `command` | Shell command to execute before the tool call proceeds. If this command exits non-zero, the tool call is blocked. |

### How the command is built from harness.yaml

The bootstrap skill reads `verification.checks` from `harness.yaml` and chains the commands together with `&&`:

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

Becomes:

```json
{
  "command": "bash enforcement/run-all.sh && ruff check src/ && pytest tests/ -x"
}
```

The `&&` chaining means checks run in order and stop at the first failure. This is intentional -- the agent gets the earliest error first, which is typically the cheapest to fix (enforcement rules before lint, lint before tests).

### Conceptual example with placeholders

The bootstrap skills generate the actual config with real commands from `harness.yaml`. The conceptual pattern is:

```json
{
  "hooks": {
    "PreToolCall": [
      {
        "matcher": "Bash",
        "pattern": "git commit",
        "command": "<enforcement_command> && <analyze_command> && <test_command>"
      }
    ]
  }
}
```

Where `<enforcement_command>`, `<analyze_command>`, and `<test_command>` are replaced with the actual commands from `harness.yaml → verification.checks` and `harness.yaml → commands`.

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

### Example generated config for a Python project

```json
{
  "hooks": {
    "PreToolCall": [
      {
        "matcher": "Bash",
        "pattern": "git commit",
        "command": "bash enforcement/run-all.sh && ruff check src/ && pytest tests/ -x"
      }
    ]
  }
}
```

### Example generated config for a Flutter project

```json
{
  "hooks": {
    "PreToolCall": [
      {
        "matcher": "Bash",
        "pattern": "git commit",
        "command": "bash enforcement/run-all.sh && dart analyze lib/ && flutter test"
      }
    ]
  }
}
```

### Example generated config for a Go project

```json
{
  "hooks": {
    "PreToolCall": [
      {
        "matcher": "Bash",
        "pattern": "git commit",
        "command": "bash enforcement/run-all.sh && golangci-lint run && go test ./..."
      }
    ]
  }
}
```

## Important Notes

- **Hooks are project-level, not user-level.** They live in `.claude/settings.json` which is committed to the repo. Every agent session on this repo gets the same verification gate.
- **Hooks do not replace CI.** CI is the final safety net. Hooks catch issues early during agent development; CI catches anything that slipped through.
- **The pattern match is substring-based.** `"git commit"` matches `git commit -m "message"`, `git commit --amend`, etc. This is intentional -- all commit variants should pass verification.
- **Keep hook commands fast.** The agent waits for the hook to complete before proceeding. If tests are slow, consider running only unit tests in the hook (`commands.test_unit`) and leaving full integration tests for CI.
- **Do not put interactive commands in hooks.** Hooks run non-interactively. Commands that prompt for input will hang.
