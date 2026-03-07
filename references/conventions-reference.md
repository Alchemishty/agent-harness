# Coding Conventions

<!-- ADAPT: Replace language-specific examples with the languages your project actually uses. Keep the principles; swap the syntax. -->

This document defines coding conventions enforced across the codebase. These are not style preferences -- they are rules that keep the codebase consistent, reviewable, and maintainable. Violations are caught by linters, enforcement rules, or code review.

---

## Import Ordering and Style

Imports are grouped and ordered consistently. Groups are separated by a blank line. Within each group, imports are sorted alphabetically.

### Group Order

1. **Standard library / language built-ins**
2. **External packages / third-party dependencies**
3. **Internal project imports** (package or relative)

### Python

```python
# 1. Standard library
import os
from datetime import datetime
from pathlib import Path

# 2. Third-party
import httpx
from pydantic import BaseModel
from sqlalchemy import Column, String

# 3. Internal
from myproject.core.models import User
from myproject.core.services import AuthService
```

### TypeScript

```typescript
// 1. Node built-ins
import fs from 'node:fs';
import path from 'node:path';

// 2. Third-party
import express from 'express';
import { z } from 'zod';

// 3. Internal
import { UserModel } from '@/core/models/user';
import { AuthService } from '@/core/services/auth';
```

### Dart

```dart
// 1. Dart SDK
import 'dart:async';
import 'dart:io';

// 2. Flutter / packages
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// 3. Internal (always use package imports, not relative)
import 'package:myapp/core/models/user.dart';
import 'package:myapp/core/services/auth_service.dart';
```

### Go

```go
import (
    // 1. Standard library
    "context"
    "fmt"
    "net/http"

    // 2. Third-party
    "github.com/gin-gonic/gin"
    "github.com/jackc/pgx/v5"

    // 3. Internal
    "myproject/internal/models"
    "myproject/internal/services"
)
```

### Rules

- Never mix groups. A blank line separates each group.
- Remove unused imports. Unused imports fail CI in every language.
- Prefer explicit imports over wildcards (`from x import *`, `import *`).
- Use absolute/package imports for cross-module references. Relative imports are acceptable only within the same feature directory.

---

## Naming Conventions

<!-- ADAPT: Adjust casing rules to match your language. The categories remain the same. -->

| Thing | Convention | Examples |
|-------|-----------|----------|
| Files | `snake_case` (Python, Dart, Go) or `kebab-case` (TypeScript) | `user_model.py`, `auth-service.ts` |
| Classes / Types | `PascalCase` | `UserModel`, `AuthService`, `AnimalType` |
| Functions / Methods | `camelCase` (TS, Dart) or `snake_case` (Python, Go) | `calculatePrice`, `get_user_by_id` |
| Variables | `camelCase` (TS, Dart) or `snake_case` (Python, Go) | `totalPrice`, `user_count` |
| Constants | `SCREAMING_SNAKE_CASE` or language-idiomatic | `MAX_RETRIES`, `defaultTimeout` |
| Private members | Language-idiomatic prefix | `_internalState` (Dart), `__private` (Python), unexported (Go) |
| Enums | `PascalCase` type, `camelCase` or `UPPER_CASE` values | `AnimalType.cow`, `Status.PENDING` |
| Database columns | `snake_case` | `created_at`, `animal_type`, `user_id` |
| JSON keys | `snake_case` | `{ "first_name": "...", "created_at": "..." }` |
| URL paths | `kebab-case` | `/api/v1/user-profiles` |
| Environment variables | `SCREAMING_SNAKE_CASE` | `DATABASE_URL`, `API_SECRET_KEY` |

### Boolean Naming

Booleans should read as yes/no questions:

- `isLoading`, `hasPermission`, `canEdit`, `shouldRetry`
- Not: `loading`, `permission`, `edit`, `retry`

### Collection Naming

Collections use plural nouns. Single items use singular:

- `users` (list), `user` (single item)
- `pendingItems` (list), `currentItem` (single item)

---

## Error Handling

### Principles

1. **Never swallow errors.** Every `catch` block must either re-raise, log, or surface the error to the user. An empty `catch` block is always a bug.
2. **Fail fast at boundaries.** Validate input at the system boundary (API handler, form submission, CLI argument parsing). Do not pass invalid data deeper into the system hoping something downstream will catch it.
3. **Use typed errors where the language supports it.** Distinguish between "not found", "unauthorized", "validation failed", and "unexpected". Callers need this to decide what to do.
4. **Log context, not just the message.** Include the operation being attempted, relevant IDs, and the original error. "Failed to upload photo for calculation abc-123: connection timeout" is useful. "Error" is not.
5. **Clean up on failure.** If a multi-step operation fails midway, clean up partial state (delete uploaded file, roll back transaction, reset UI loading state).

### Pattern: UI / Stateful Context

After any async operation in a UI context, check that the component is still alive before touching state or showing messages:

```
await someAsyncWork();
if component is no longer mounted/alive:
    return early
update UI state
```

This prevents crashes from updating destroyed components and is mandatory after every `await` in widget/component lifecycle methods.

### Pattern: Service / Business Logic

```
set loading state
clear previous error
try:
    perform operation
    set success state
catch known error types:
    set error state with descriptive message
catch unexpected errors:
    log full error with context
    set error state with generic message
finally:
    clear loading state (always, even on error)
```

### Pattern: Repository / Data Access

```
try:
    perform external call (DB query, HTTP request, file read)
    return result
catch infrastructure errors:
    wrap in domain-specific error type
    include original error as cause
    re-raise
```

Repositories should not catch and suppress errors. They translate infrastructure failures into domain errors and let the caller decide what to do.

---

## Serialization Patterns

### Boundary Validation

All data entering the system from external sources (API requests, database rows, JSON files, environment variables) must be validated at the boundary. Never trust external data shapes.

```
external data (JSON, query params, env vars)
  → parse and validate at boundary (fromJson, schema validation, type coercion)
    → internal model (type-safe, validated)
      → business logic operates on trusted types only
```

### Key Casing

- **External data** (JSON, database columns, API responses): `snake_case`
- **Internal code** (variables, fields, properties): language-idiomatic casing

The serialization layer handles the translation. Business logic never sees snake_case keys.

### Enum Serialization

Store enums as their string name, not numeric index. Numeric indices break when enum values are reordered.

Always parse with a fallback:

```
parse enum from string:
    find matching value by name
    if no match: return safe default (not crash)
```

### Date/Time Serialization

Always store and transmit as ISO 8601 strings in UTC. Convert to local time only at the display layer.

---

## Code Smells

| Smell | Symptom | Fix |
|-------|---------|-----|
| God file | File exceeds ~300 lines or has multiple unrelated responsibilities | Split into focused files. One file = one concept. |
| Missing error handling | `catch` block is empty, or errors are ignored with `_ =` / `void` | Add logging at minimum. Surface to user if actionable. Re-raise if the caller needs to know. |
| Hardcoded configuration | URLs, credentials, feature flags, or magic numbers embedded in source | Extract to environment variables, config files, or named constants. |
| Unused imports | Import statements referencing modules not used in the file | Remove them. Configure linter to flag automatically. |
| Raw external calls in UI | Database queries, HTTP requests, or SDK calls directly in a screen/component/handler | Move to a repository or service. UI layer calls services only. |
| Missing cleanup checks | Async operation completes but the component/context is destroyed before state update | Check mounted/alive state after every await in lifecycle-bound code. |
| Swallowed errors | `catch (e) {}` or `except Exception: pass` with no action taken | At minimum log. Preferably surface to user or re-raise. |
| TODO without tracking | `// TODO: fix this later` with no issue link or owner | Attach an issue number: `// TODO(#142): migrate to new API`. Untracked TODOs rot. |
| Circular dependencies | Module A imports B, B imports A (directly or transitionally) | Extract shared code to a lower layer. Break the cycle by introducing an interface or moving the shared type to models. |
| Leaky abstraction | Caller must understand internal details of a dependency to use it correctly | Redesign the interface. The caller should not need to know about retry logic, connection pooling, or pagination internals. |

---

## Barrel / Index Files

Each module directory may have a barrel file (named `index.ts`, `__init__.py`, `models.dart`, etc.) that re-exports its public API.

### Purpose

- Simplify imports for consumers: one import instead of five.
- Define the public surface of a module explicitly.
- Hide internal implementation files from external consumers.

### Rules

1. **One barrel file per directory.** Never nest barrels (a barrel that re-exports another barrel).
2. **Only export public API.** Internal helpers, private utilities, and implementation details are not exported.
3. **Keep barrels flat.** A barrel file contains only export/re-export statements. No logic, no classes, no functions.
4. **Update barrels when adding files.** If you add a new public file to a module, add its export to the barrel.

### Example Structure

```
core/
  models/
    user.ext
    order.ext
    models.ext        ← barrel: exports user and order
  services/
    auth_service.ext
    sync_service.ext
    services.ext      ← barrel: exports auth_service and sync_service
```

Consumers import from the barrel:

```
import { User, Order } from '@/core/models';       // TypeScript
from myproject.core.models import User, Order       # Python
import 'package:myapp/core/models/models.dart';     // Dart
```

---

## Formatting and Linting

<!-- ADAPT: Replace with your actual formatter and linter configuration. -->

- **Formatter:** Use the language-standard formatter (Prettier, Black, dart format, gofmt). No custom formatting rules.
- **Linter:** Use the language-standard linter with a strict preset. Zero warnings policy -- warnings are treated as errors in CI.
- **Line length:** Follow language convention (80 for Dart/Go, 88 for Python/Black, no enforced limit for TypeScript with Prettier defaults).
- **Trailing commas:** Use where the language/formatter supports them (Dart, TypeScript). They produce cleaner diffs.
- **Format on save:** Recommended for all editors. CI enforces formatting regardless.
