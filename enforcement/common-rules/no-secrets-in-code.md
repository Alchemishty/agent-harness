# Rule: no-secrets-in-code

Scan source files for hardcoded secrets and credentials. Prevents accidental commit of API keys, passwords, tokens, and other sensitive values.

## Configuration

Reads from `harness.yaml`:

```yaml
enforcement:
  no_secrets_pattern: "(API_KEY|SECRET|PASSWORD|TOKEN)\\s*="
```

The default pattern catches common secret assignment patterns. Projects should extend this pattern for project-specific secret names (see Extending the Pattern below).

## Behavior

1. Read the `no_secrets_pattern` regex from `harness.yaml`
2. Scan all tracked source files, respecting `.gitignore` exclusions
3. Skip binary files, lock files, and the `harness.yaml` file itself (which contains the pattern definition)
4. For each match, report the file, line number, matched content (truncated), and remediation

## Error Output

```
ERROR: no-secrets-in-code
  src/config.py:23 matches pattern: API_KEY = "sk-proj-abc..."
  FIX: Move to environment variable. Never commit secrets. Use os.environ['API_KEY'] instead.
  REFERENCE: docs/conventions.md#environment-variables

ERROR: no-secrets-in-code
  lib/services/auth.dart:7 matches pattern: TOKEN = "ghp_1234567..."
  FIX: Move to environment variable. Never commit secrets. Pass via --dart-define at build time.
  REFERENCE: docs/conventions.md#environment-variables
```

## Example Implementation

This shell script works for any language. Bootstrap skills should adapt the exclusion list and remediation text to the target project.

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── Read config ──────────────────────────────────────────────────────────────
# Default pattern if harness.yaml is not parseable
DEFAULT_PATTERN='(API_KEY|SECRET|PASSWORD|TOKEN)\s*='

# Try to read pattern from harness.yaml (simple grep-based extraction)
if [ -f "harness.yaml" ]; then
  PATTERN=$(grep 'no_secrets_pattern:' harness.yaml | sed 's/.*no_secrets_pattern:\s*//' | sed 's/^"//' | sed 's/"$//' | sed "s/^'//" | sed "s/'$//")
fi
PATTERN="${PATTERN:-$DEFAULT_PATTERN}"

# ── Files to scan ────────────────────────────────────────────────────────────
# Use git ls-files to respect .gitignore, filter to source files only
FILES=$(git ls-files --cached --others --exclude-standard 2>/dev/null || find . -type f)

# Exclude non-source files
EXCLUDE_PATTERNS=(
  '\.lock$'
  '\.sum$'
  '\.g\.dart$'
  'harness\.yaml$'
  'harness\.schema\.yaml$'
  '\.env\.example$'
  'node_modules/'
  '\.git/'
  'build/'
  'dist/'
  '\.png$'
  '\.jpg$'
  '\.jpeg$'
  '\.gif$'
  '\.ico$'
  '\.woff'
  '\.ttf$'
  '\.eot$'
  '\.pdf$'
  '\.zip$'
  '\.tar'
)

EXCLUDE_REGEX=$(printf '%s|' "${EXCLUDE_PATTERNS[@]}")
EXCLUDE_REGEX="${EXCLUDE_REGEX%|}"  # Remove trailing pipe

FILTERED_FILES=$(echo "$FILES" | grep -Ev "$EXCLUDE_REGEX" || true)

if [ -z "$FILTERED_FILES" ]; then
  echo "no-secrets-in-code: no source files to scan."
  exit 0
fi

# ── Scan ─────────────────────────────────────────────────────────────────────
FAILED=0

while IFS= read -r file; do
  [ -f "$file" ] || continue

  # Use grep with line numbers; -E for extended regex, -n for line numbers
  MATCHES=$(grep -nE "$PATTERN" "$file" 2>/dev/null || true)

  if [ -n "$MATCHES" ]; then
    while IFS= read -r match; do
      LINE_NUM=$(echo "$match" | cut -d: -f1)
      LINE_CONTENT=$(echo "$match" | cut -d: -f2-)

      # Truncate long lines for readability (show first 80 chars)
      TRUNCATED=$(echo "$LINE_CONTENT" | cut -c1-80)
      if [ ${#LINE_CONTENT} -gt 80 ]; then
        TRUNCATED="${TRUNCATED}..."
      fi

      echo "ERROR: no-secrets-in-code"
      echo "  ${file}:${LINE_NUM} matches pattern: ${TRUNCATED}"
      echo "  FIX: Move to environment variable. Never commit secrets."
      echo "  REFERENCE: docs/conventions.md#environment-variables"
      echo ""
      FAILED=1
    done <<< "$MATCHES"
  fi
done <<< "$FILTERED_FILES"

if [ "$FAILED" -ne 0 ]; then
  exit 1
fi

echo "no-secrets-in-code: passed."
exit 0
```

## Extending the Pattern

The default pattern `(API_KEY|SECRET|PASSWORD|TOKEN)\s*=` is intentionally broad. Projects should extend it to catch project-specific secret names.

### Adding project-specific patterns

Update `harness.yaml` with a more comprehensive regex:

```yaml
enforcement:
  # Extended to include project-specific secret names
  no_secrets_pattern: "(API_KEY|SECRET|PASSWORD|TOKEN|SUPABASE_ANON_KEY|DATABASE_URL|STRIPE_KEY|AWS_ACCESS_KEY|PRIVATE_KEY)\\s*="
```

### Common extensions by ecosystem

**Supabase/Firebase projects:**
```
SUPABASE_URL|SUPABASE_ANON_KEY|SUPABASE_SERVICE_ROLE|FIREBASE_API_KEY
```

**AWS projects:**
```
AWS_ACCESS_KEY_ID|AWS_SECRET_ACCESS_KEY|AWS_SESSION_TOKEN
```

**Stripe/payment projects:**
```
STRIPE_SECRET_KEY|STRIPE_WEBHOOK_SECRET|sk_live_|sk_test_
```

**General API integrations:**
```
OPENAI_API_KEY|ANTHROPIC_API_KEY|SENDGRID_API_KEY
```

### Pattern tips

- Use `\s*=` to catch assignments with varying whitespace
- Prefix patterns like `sk_live_` or `ghp_` catch token values directly, not just variable names
- Test your pattern against your codebase before enabling: `grep -rnE "YOUR_PATTERN" .`
- Be careful with overly broad patterns that produce false positives in test fixtures or documentation

## False Positive Handling

Some matches are intentional (e.g., documentation examples, test fixtures with fake values). Handle these by:

1. Using clearly fake values in examples: `API_KEY = "your-api-key-here"` (the rule should still flag these -- better safe)
2. If a file legitimately needs to reference these patterns (e.g., a template or example), add it to the exclusion list in the script
3. Never suppress the rule globally. If false positives are excessive, the pattern needs refinement rather than disabling.
