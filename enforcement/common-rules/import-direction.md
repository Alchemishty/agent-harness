# Rule: import-direction

Enforce that imports between architectural layers only flow in the allowed direction. Prevents circular dependencies and maintains clean layer separation.

## Configuration

Reads from `harness.yaml`:

```yaml
enforcement:
  import_direction:
    enabled: true
    layers: ["models", "repositories", "services", "api"]
    rule: "left-to-right only"
```

## The Rule

Layers are ordered left-to-right. A module in a given layer may only import from layers to its left (earlier in the list). It must never import from layers to its right (later in the list) or from itself across feature boundaries (depending on project conventions).

Given `layers: ["models", "repositories", "services", "api"]`:

| Module in | Can import from | Cannot import from |
|-----------|----------------|--------------------|
| `models` | (nothing -- leaf layer) | repositories, services, api |
| `repositories` | models | services, api |
| `services` | models, repositories | api |
| `api` | models, repositories, services | (can import everything) |

This enforces the dependency inversion principle: high-level modules depend on low-level modules, never the reverse.

## Behavior

1. Read `enforcement.import_direction.layers` from `harness.yaml`
2. If `enabled` is false, exit 0 immediately
3. For each source file, determine which layer it belongs to based on its path
4. Parse import statements from the file
5. For each import, determine which layer it targets
6. If the target layer appears after the source layer in the ordered list, report a violation

## Error Output

```
ERROR: import-direction
  src/models/user.py:3 imports from src/services/user_service.py
  FIX: models layer cannot import from services layer. Move shared logic to models or create an interface.
  ALLOWED: models → (nothing) | repositories → models | services → models, repositories | api → models, repositories, services
  REFERENCE: docs/architecture.md#layer-dependencies

ERROR: import-direction
  src/repositories/user_repo.py:5 imports from src/api/middleware.py
  FIX: repositories layer cannot import from api layer. Extract the needed functionality into a lower layer.
  ALLOWED: models → (nothing) | repositories → models | services → models, repositories | api → models, repositories, services
  REFERENCE: docs/architecture.md#layer-dependencies
```

The `ALLOWED:` line shows the complete dependency chain so the agent understands the full picture.

## Language-Specific Import Parsing

The enforcement script must parse imports differently depending on the project language. Below are patterns for common languages.

### Python

```python
# Standard imports to detect:
import src.services.user_service          # → target layer: services
from src.models.user import User          # → target layer: models
from src.repositories import user_repo    # → target layer: repositories

# Regex patterns:
# ^import\s+(\S+)
# ^from\s+(\S+)\s+import
```

Layer detection from import path: split on `.` and match against layer names in the path segments.

### TypeScript / JavaScript

```typescript
// Standard imports to detect:
import { User } from '../models/user';            // → target layer: models
import { UserService } from '@/services/user';    // → target layer: services
import * as repos from '../../repositories';      // → target layer: repositories
const api = require('../api/routes');              // → target layer: api

// Regex patterns:
// import\s+.*from\s+['"]([^'"]+)['"]
// require\s*\(\s*['"]([^'"]+)['"]\s*\)
```

Layer detection: extract the path from the import string, then match path segments against layer names.

### Dart

```dart
// Standard imports to detect:
import 'package:my_app/services/user_service.dart';    // → target layer: services
import 'package:my_app/models/user.dart';              // → target layer: models
import '../repositories/user_repo.dart';               // → target layer: repositories

// Regex patterns:
// import\s+['"]package:[^/]+/([^'"]+)['"]
// import\s+['"]\.\.?/([^'"]+)['"]
```

Layer detection: extract the path after `package:app_name/` or the relative path, then match segments against layer names.

### Go

```go
// Standard imports to detect:
import (
    "myapp/services/user"       // → target layer: services
    "myapp/models"              // → target layer: models
    "myapp/repositories/db"     // → target layer: repositories
)

// Regex patterns (inside import blocks):
// "([^"]+)"
// Filter to only project-internal imports (starts with module name from go.mod)
```

Layer detection: read module name from `go.mod`, filter imports to those starting with the module name, then match path segments against layer names.

## Implementation Approach

The enforcement script should:

1. **Determine the project language** from `harness.yaml → stack.language`
2. **Build a layer-to-index map** from the ordered layers list (e.g., `{models: 0, repositories: 1, services: 2, api: 3}`)
3. **Map directories to layers.** For each layer name, determine which directory paths correspond to it. This is typically a direct match (e.g., `src/models/` maps to the `models` layer), but the script should handle project-specific directory structures read from `docs/architecture.md` or `harness.yaml`.
4. **For each source file:**
   - Determine its layer from its file path
   - If the file does not belong to any recognized layer, skip it
   - Parse its import statements using language-appropriate regex
   - For each import, determine the target layer
   - If target layer index > source layer index, it is a violation
5. **Report all violations** in the standard error format

## Example Shell Script Skeleton

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── Read config ──────────────────────────────────────────────────────────────
# Parse layers from harness.yaml (simplified — real implementation may use yq)
LAYERS_RAW=$(grep -A 20 'import_direction:' harness.yaml | grep -A 10 'layers:' | grep '^\s*-' | sed 's/.*- *//' | sed 's/"//g' | sed "s/'//g")

if [ -z "$LAYERS_RAW" ]; then
  echo "import-direction: no layers configured, skipping."
  exit 0
fi

# Check if enabled
ENABLED=$(grep -A 1 'import_direction:' harness.yaml | grep 'enabled:' | awk '{print $2}')
if [ "$ENABLED" = "false" ]; then
  echo "import-direction: disabled, skipping."
  exit 0
fi

# Build ordered array
declare -a LAYERS
INDEX=0
while IFS= read -r layer; do
  layer=$(echo "$layer" | xargs)  # trim whitespace
  [ -n "$layer" ] && LAYERS[$INDEX]="$layer" && ((INDEX++))
done <<< "$LAYERS_RAW"

# Build allowed dependency description
ALLOWED=""
for ((i=0; i<${#LAYERS[@]}; i++)); do
  deps=""
  for ((j=0; j<i; j++)); do
    [ -n "$deps" ] && deps="$deps, "
    deps="$deps${LAYERS[$j]}"
  done
  [ -z "$deps" ] && deps="(nothing)"
  ALLOWED="$ALLOWED${LAYERS[$i]} → $deps | "
done
ALLOWED="${ALLOWED% | }"

FAILED=0

# ── Scan source files ───────────────────────────────────────────────────────
# (Language-specific scanning logic goes here)
# For each file:
#   1. Determine source layer from path
#   2. Parse imports
#   3. Determine target layer for each import
#   4. Compare layer indices
#   5. Report violations

# ── Result ───────────────────────────────────────────────────────────────────
if [ "$FAILED" -ne 0 ]; then
  exit 1
fi

echo "import-direction: passed."
exit 0
```

The bootstrap skill fills in the language-specific scanning section based on `stack.language` from `harness.yaml`.

## Edge Cases

- **Files outside any layer** (e.g., utility modules, shared helpers): skip these by default. If the project wants to enforce rules on them, add them as a layer in the configuration.
- **Third-party imports** (e.g., `import numpy`, `import 'package:flutter/material.dart'`): always allowed. Only enforce direction on project-internal imports.
- **Barrel/index re-exports**: treat these as belonging to the layer of the barrel file, not the layer of the re-exported module.
- **Test files**: typically excluded from import direction enforcement since tests legitimately import from any layer. The script should skip files in test directories.
