# Context Management Guide

This reference teaches agents how to manage their context window effectively during long-running development sessions. Apply these principles to every task — they prevent the quality degradation that occurs as conversations grow.

## Why Context Management Matters

Language models have a finite context window (the "attention budget"). As context fills with tool output, code, and conversation history, three things happen:

1. **Information in the middle gets 10-40% lower recall** (U-shaped attention curve — beginning and end are favored)
2. **Quality degrades around 70-80% context utilization** — responses become less accurate
3. **Accumulated noise crowds out signal** — verbose tool output dilutes important instructions

Context management is about keeping the signal-to-noise ratio high throughout a session.

## The Memory Directory

```
memory/
  patterns.md       # Confirmed patterns and conventions
  fixes.md          # Solutions to recurring problems
  preferences.md    # User workflow and tool preferences
  domain.md         # Domain knowledge accumulated over sessions
```

### What to Write

- **Confirmed patterns** — coding patterns validated across multiple sessions (not guesses from one observation)
- **Recurring fixes** — solutions to problems that come up repeatedly (e.g., "When pytest fails with X, the fix is Y")
- **User preferences** — workflow preferences the user has stated explicitly (e.g., "always use bun instead of npm")
- **Domain knowledge** — business rules, entity relationships, and edge cases discovered during implementation

### What NOT to Write

- Session-specific state (current task, in-progress work, temporary context)
- Speculative conclusions from reading a single file
- Anything already documented in `docs/` — don't duplicate
- Information that might be incomplete — verify before writing

### How to Use

At session start, read all files in `memory/` to load cross-session context. This is cheap (a few hundred tokens) and gives immediate benefit.

When writing new memories, check for existing entries first. Update rather than duplicate.

### Temporal Memory: Invalidate, Don't Discard

When a memory entry turns out to be wrong or outdated, **do not delete it**. Instead, mark it as superseded with a date and pointer to what replaced it:

```markdown
## ~~Use pattern X for database queries~~ [SUPERSEDED 2026-03-10]

Superseded by: "Use pattern Y for database queries" (below)
Reason: Pattern X caused N+1 query issues discovered during user-profile feature.
```

Why preserve history:
- Future agents can understand *why* something changed, not just what the current state is
- If a superseded pattern resurfaces in old code, the agent knows the context
- Prevents re-learning lessons that were already learned and then forgotten

When reading memory at session start, **skip entries marked `[SUPERSEDED]`** — they exist for historical reference, not for active use. Only load current (unmarked) entries into working context.

## The Scratch Pad

```
scratch/
  last-test-run.log       # Full test output
  last-lint-run.log        # Full lint output
  last-enforcement.log     # Full enforcement output
  build-output.log         # Build output
```

### The Pattern

When running commands that produce verbose output (tests, linting, builds):

1. **Capture** — Write full output to a file in `scratch/`
2. **Summarize** — Keep only a compact summary in conversation context (pass/fail, error count, first 3 errors)
3. **Reference** — If you need details later, re-read the specific scratch file

This prevents 500+ lines of test output from consuming context window space that's needed for reasoning.

### Example

Instead of keeping the full pytest output in context:
```
PASSED: 47 tests
FAILED: 2 tests
  - test_user_create (AssertionError: expected 201 got 400)
  - test_user_delete (PermissionError: not authorized)
Full output: scratch/last-test-run.log
```

### Lifecycle

- `scratch/` is listed in `.gitignore` — it is never committed
- Files are overwritten on each run (not accumulated)
- The `/garden` skill cleans `scratch/` during post-feature cleanup

## Context Budget Guidelines

### For Skills

When loading context at the start of a task:
- Read `AGENTS.md` first (small, high-signal)
- Read only the **relevant sections** of docs files, not entire files — if you need the naming conventions from `conventions.md`, read that section, not the whole file
- Read `memory/` files (small, high-signal)
- Load `harness.yaml` (required, compact)

### For Implementation Loops

During multi-step implementation (`/implement-feature`):
- **Re-read the plan** at the start of each step — this re-anchors the plan in a recent, attention-favored position
- **Write verbose output to scratch/** — don't let test/lint output accumulate in conversation (see Observation Masking below)
- **Write context checkpoints** from step 4 onward — `scratch/session-state.md` (see Compression Triggers below)
- **For plans with 7+ steps**, consider using sub-agents for phases — each gets a fresh context window

### Attention-Favored Placement

The U-shaped attention curve means models attend best to the **beginning** and **end** of their context. Apply this to:

- **AGENTS.md** — most critical rules at positions 1-3 and 8-10, less critical in the middle
- **Skill instructions** — put critical rules at the END of the skill (closest to where the agent generates its response)
- **Plan files** — put the current step's details last in the prompt

## Observation Masking

The scratch pad pattern applies beyond just the verification gate. **Any tool output over ~50 lines should be captured to file and summarized.** This includes:

- `git diff` output when exploring what changed
- Large file reads during planning or analysis
- Build and migration output
- Dependency install output
- Any API response or log dump

### The Rule

When a command or file read produces verbose output:

1. If the output is **under 50 lines** — keep it in context, it's fine
2. If the output is **50-200 lines** — keep a summary (key findings, errors, structure) and note where the full output can be re-read
3. If the output is **over 200 lines** — write to `scratch/`, keep only a 5-10 line summary in context

### During Planning (`/create-plan`)

Planning reads many files (architecture.md, conventions.md, domain.md, decisions/, source files). For each file:
- Extract only the **relevant sections**, not the entire file
- If you need to scan a large source file to understand its structure, capture the full read to scratch and keep a summary (class names, method signatures, key patterns)
- For `docs/decisions/` — read titles and status lines first, then drill into only the relevant decisions

### During Implementation (`/implement-feature`)

- Step verification output → `scratch/last-verification.log`
- Test output → `scratch/last-test-run.log`
- Large git diffs → `scratch/last-diff.log`
- Build output → `scratch/build-output.log`

### The Payoff

A typical implementation session without masking consumes 40-60% of context on tool output alone. With masking, that drops to 10-15%, leaving the remaining budget for reasoning, plan adherence, and code quality.

## Compression Triggers and Context Recovery

When a session runs long (many implementation steps, multiple retry cycles, extensive exploration), context accumulates to the point where quality degrades. Instead of hoping for the best, use explicit checkpoints.

### When to Trigger

- **After completing step 4+** of a multi-step plan — the session is getting long
- **After any retry cycle** on the verification gate — retries add significant context
- **Before switching phases** (Implementation → Cleanup → PR) — good natural checkpoint
- **When you notice yourself losing track** of earlier decisions or plan details

### The Checkpoint Protocol

Write a structured state summary to `scratch/session-state.md`:

```markdown
# Session State Checkpoint

## Current Position
- Plan: <plan file path>
- Current step: <N> of <total>
- Branch: <branch name>

## Completed Steps
- Step 1: <one-line summary> [DONE]
- Step 2: <one-line summary> [DONE]
- ...

## Files Modified This Session
- <path> — <what was changed>
- <path> — <what was changed>

## Decisions Made
- <decision and rationale>
- <decision and rationale>

## Open Issues
- <any unresolved problems or questions>

## Next Action
- <what to do next>
```

### How to Use the Checkpoint

1. **For yourself** — Re-read `scratch/session-state.md` to re-orient. This is cheaper than re-reading all the accumulated conversation context.
2. **For sub-agents** — Pass the checkpoint file when spawning a sub-agent. It gets the full picture in ~200 tokens instead of inheriting a bloated conversation.
3. **For recovery** — If the session is terminated and restarted, the checkpoint file survives in `scratch/` (it's local, not committed, but persists on disk).

### State Summary vs Memory

| | `scratch/session-state.md` | `memory/*.md` |
|---|---|---|
| **Scope** | This session only | Cross-session |
| **Lifetime** | Deleted by `/garden` | Persists permanently |
| **Content** | Current progress, files touched, decisions | Confirmed patterns, fixes, preferences |
| **Purpose** | Context recovery within a session | Learning across sessions |

Don't confuse them. Session state is temporary and specific. Memory is durable and general.

## Plan Re-Reading

During long implementation sessions, the original plan drifts toward the middle of context (the lowest-recall zone). Combat this with explicit re-reading:

1. Before starting each implementation step, re-read the relevant step from the plan file
2. This places the plan requirements at a recent, high-attention position
3. Cost: ~100-200 tokens per re-read. Benefit: prevents drift from plan intent

This is the principle of "manipulating attention through recitation" — deliberately placing information where it will be recalled.

## Phase Isolation

For long-running tasks (7+ implementation steps), context accumulates to the point of degradation. The solution is sub-agents:

- Each sub-agent gets a **fresh context window** focused on one phase
- Pass only the relevant state: plan file path, branch name, `harness.yaml` path, `memory/` path
- The sub-agent loads only what it needs and works without inherited noise

**Key insight:** Sub-agents exist primarily to **isolate context**, not to divide roles.

### When to Isolate

| Steps in plan | Recommendation |
|---------------|----------------|
| 1-3 | No isolation needed |
| 4-6 | Optional — use judgment |
| 7+ | Strongly consider phase isolation |

### What to Pass to Sub-Agents

- Path to the plan file (with which steps to execute)
- Path to `harness.yaml`
- Path to `memory/`
- Branch name
- Any decisions made in prior phases (from the plan's Decision Log)

### The Telephone Game Problem

When a supervisor agent paraphrases sub-agent results, fidelity drops. Research found **50% performance improvement** by implementing direct pass-through instead of supervisor summarization.

**The fix:** Sub-agents write their results to files, not just to conversation. The orchestrator reads the files rather than relying on its own paraphrased summary.

| Instead of | Do this |
|------------|---------|
| Sub-agent returns text summary → orchestrator paraphrases to user | Sub-agent writes to `scratch/phase-N-results.md` → orchestrator reads file |
| Sub-agent makes decisions → orchestrator re-explains them | Sub-agent writes decisions to plan's Decision Log → orchestrator reads the log |
| Sub-agent finds issues → orchestrator relays from memory | Sub-agent writes to `scratch/issues-found.md` → orchestrator reads file |

This ensures the orchestrator works from the **original output**, not a lossy telephone-game version. It also means if the orchestrator's context is compressed, the sub-agent's full results are still on disk.
