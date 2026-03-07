# Rule: file-size-limits

Flag source files that exceed a configurable line count threshold. Large files indicate modules that have taken on too many responsibilities and should be split.

## Configuration

Reads from `harness.yaml`:

```yaml
enforcement:
  file_size_limit: 300
```

The default limit is 300 lines. This is a pragmatic threshold -- large enough for substantial modules but small enough to discourage god-files.

## Behavior

1. Read `enforcement.file_size_limit` from `harness.yaml`
2. Scan all tracked source files, respecting `.gitignore` exclusions
3. Count lines per file (total lines by default; optionally exclude blank lines and comments)
4. Report files exceeding the limit

## Error Output

```
ERROR: file-size-limits
  src/services/user_service.py:0 has 427 lines (limit: 300)
  FIX: Split into smaller modules. Extract related functions into separate files.
  REFERENCE: docs/conventions.md#file-size

ERROR: file-size-limits
  lib/features/calculation/screens/calculation_form_screen.dart:0 has 385 lines (limit: 300)
  FIX: Split into smaller modules. Extract widget methods into separate widget files.
  REFERENCE: docs/conventions.md#file-size
```

The `:0` line number indicates the violation is file-level, not at a specific line.

## Example Implementation

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── Read config ──────────────────────────────────────────────────────────────
DEFAULT_LIMIT=300

if [ -f "harness.yaml" ]; then
  LIMIT=$(grep 'file_size_limit:' harness.yaml | awk '{print $2}' | tr -d '"' | tr -d "'")
fi
LIMIT="${LIMIT:-$DEFAULT_LIMIT}"

# ── Determine source file extensions ────────────────────────────────────────
# Adapt this list based on stack.language from harness.yaml
# The bootstrap skill generates the appropriate list for the target project
SOURCE_EXTENSIONS=(
  "py"
  "dart"
  "ts"
  "tsx"
  "js"
  "jsx"
  "go"
  "rs"
  "java"
  "kt"
  "swift"
  "rb"
)

# Build grep pattern for file extensions
EXT_PATTERN=$(printf '\.%s$|' "${SOURCE_EXTENSIONS[@]}")
EXT_PATTERN="${EXT_PATTERN%|}"  # Remove trailing pipe

# ── Files to scan ────────────────────────────────────────────────────────────
FILES=$(git ls-files --cached --others --exclude-standard 2>/dev/null || find . -type f)
FILTERED_FILES=$(echo "$FILES" | grep -E "$EXT_PATTERN" || true)

# Exclude generated files, tests (optional), and vendor directories
EXCLUDE_PATTERNS=(
  '\.g\.dart$'
  '\.freezed\.dart$'
  '\.gen\.ts$'
  'node_modules/'
  'vendor/'
  'build/'
  'dist/'
  '\.dart_tool/'
)

EXCLUDE_REGEX=$(printf '%s|' "${EXCLUDE_PATTERNS[@]}")
EXCLUDE_REGEX="${EXCLUDE_REGEX%|}"

FILTERED_FILES=$(echo "$FILTERED_FILES" | grep -Ev "$EXCLUDE_REGEX" || true)

if [ -z "$FILTERED_FILES" ]; then
  echo "file-size-limits: no source files to scan."
  exit 0
fi

# ── Scan ─────────────────────────────────────────────────────────────────────
FAILED=0

while IFS= read -r file; do
  [ -f "$file" ] || continue

  # Count total lines
  LINE_COUNT=$(wc -l < "$file" | tr -d ' ')

  if [ "$LINE_COUNT" -gt "$LIMIT" ]; then
    echo "ERROR: file-size-limits"
    echo "  ${file}:0 has ${LINE_COUNT} lines (limit: ${LIMIT})"
    echo "  FIX: Split into smaller modules. Extract related functions into separate files."
    echo "  REFERENCE: docs/conventions.md#file-size"
    echo ""
    FAILED=1
  fi
done <<< "$FILTERED_FILES"

# ── Result ───────────────────────────────────────────────────────────────────
if [ "$FAILED" -ne 0 ]; then
  exit 1
fi

echo "file-size-limits: passed."
exit 0
```

## Counting Strategy

The default implementation counts total lines including blank lines and comments. This keeps the rule simple and predictable.

Projects that want a more lenient count can modify the script to exclude:

- **Blank lines:** `grep -c '.' "$file"` instead of `wc -l`
- **Comment-only lines:** language-specific regex (e.g., `^\s*#` for Python, `^\s*//` for Dart/TS/Go)
- **Import blocks:** large import sections can inflate line counts without adding complexity

If using a lenient count, document the counting method in `docs/conventions.md` so expectations are clear.

## What to Do When a File Exceeds the Limit

The remediation is always the same: split the file into smaller, focused modules.

Common split strategies by file type:

| File type | Split strategy |
|-----------|---------------|
| Service class | Extract methods into helper modules or sub-services |
| Screen/component | Extract widget methods into separate widget files |
| Router/controller | Split routes into route groups in separate files |
| Model file | Split into one-model-per-file if multiple models exist |
| Test file | Split by test category (unit, edge cases, integration) |
| Utility file | Group functions by domain and split into focused utility modules |

## Excluding Specific Files

Some files legitimately need to be long (e.g., generated code, migration files, configuration). Handle exclusions by:

1. Adding patterns to the `EXCLUDE_PATTERNS` array in the script
2. Generated files should already be excluded by convention (e.g., `*.g.dart`, `*.gen.ts`)
3. Never exclude a file just because "it's hard to split" -- that is exactly the situation this rule is meant to catch

If a specific non-generated file genuinely needs an exception, add a comment in the script explaining why, and document the exception in `docs/conventions.md`.
