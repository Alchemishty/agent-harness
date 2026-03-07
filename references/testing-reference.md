# Testing Guide

<!-- ADAPT: Replace language-specific examples with the languages and test frameworks your project uses. The principles and structure apply universally. -->

This document defines the testing strategy, conventions, and expectations for the codebase. Every behavioral change requires a corresponding test update. Tests are not optional -- they are part of the definition of done.

---

## Test Tiers

Tests are organized into three tiers. Each tier has a different scope, speed, and purpose. Most tests should be unit tests. Integration and end-to-end tests are used sparingly for high-value verification.

```
        /\
       /  \        End-to-End (E2E)
      / few \      Full system, real dependencies, slow
     /--------\
    /          \   Integration
   /  moderate  \  Multiple units together, some real dependencies
  /--------------\
 /                \ Unit
/     many         \ Single function/class, no external dependencies, fast
────────────────────
```

### Unit Tests

Test a single function, method, or class in isolation. All dependencies are mocked or stubbed. These run in milliseconds.

**What to test here:**
- Pure functions (calculations, transformations, formatting)
- Model serialization and deserialization (fromJson/toJson round-trips)
- Validation logic
- State management (provider/store transitions given inputs)
- Edge cases: empty inputs, null values, boundary values, error paths
- Business rule enforcement

**What NOT to test here:**
- Database queries (use integration tests)
- HTTP requests to real services (use integration tests)
- UI rendering and interaction (use widget/component tests or E2E)

### Integration Tests

Test multiple units working together, often with a real database, file system, or test server. These run in seconds.

**What to test here:**
- Repository methods against a real (test) database
- Service orchestration across multiple repositories
- API route handlers with a test HTTP client
- Serialization round-trips through an actual data store
- Sync workflows (write locally, sync remotely, verify consistency)

**What NOT to test here:**
- Individual business rules (use unit tests)
- Full user flows across multiple screens (use E2E)

### End-to-End (E2E) Tests

Test complete user flows through the real application. These interact with the system as a user would -- clicking buttons, filling forms, navigating between screens.

**What to test here:**
- Critical user journeys (login, complete a core workflow, logout)
- Cross-screen navigation and data passing
- Flows that span multiple services or subsystems

**What NOT to test here:**
- Every edge case (use unit tests)
- Internal implementation details
- Anything that can be verified faster at a lower tier

---

## Decision Matrix: Which Tier?

| What you are testing | Tier | Why |
|---------------------|------|-----|
| A pure function that computes a value | Unit | No dependencies, fast, deterministic |
| A model's fromJson with various inputs | Unit | Pure data transformation |
| Validation rules on a form field | Unit | Pure logic, no UI needed |
| State transitions in a provider/store | Unit | Mock the dependencies, test the logic |
| A database query returns correct results | Integration | Needs a real (test) database |
| An API endpoint returns correct status and body | Integration | Needs HTTP transport |
| Upload file then verify it is retrievable | Integration | Needs real storage (or test double) |
| User logs in, creates a record, sees it in a list | E2E | Full journey across multiple screens/pages |
| Error message appears when network is unavailable | E2E | Requires simulating system conditions |

---

## File Location Convention

Test files mirror the source tree. A source file at a given path has its test file at the corresponding path under the test directory.

### Pattern

```
Source:  src/core/models/user.ext
Test:    tests/unit/core/models/test_user.ext    (or user_test.ext, user.test.ext)

Source:  src/features/auth/auth_service.ext
Test:    tests/unit/features/auth/test_auth_service.ext

Source:  src/repositories/order_repo.ext
Test:    tests/integration/repositories/test_order_repo.ext
```

### Language-Specific Naming

| Language | Test file naming | Location |
|----------|-----------------|----------|
| Python | `test_<module>.py` | `tests/unit/...`, `tests/integration/...` |
| TypeScript | `<module>.test.ts` | `__tests__/...` or mirrored `tests/...` |
| Dart | `<module>_test.dart` | `test/unit/...`, `test/widget/...`, `integration_test/...` |
| Go | `<module>_test.go` | Same directory as source (Go convention) |

### Directory Structure

<!-- ADAPT: Replace with your actual test directory layout. -->

```
tests/                          (or test/)
├── helpers/
│   ├── mocks.ext               # Shared mock/stub definitions
│   ├── factories.ext           # Test data factories (seeds, builders)
│   └── fixtures.ext            # Shared setup/teardown utilities
├── unit/
│   ├── core/
│   │   └── models/
│   │       └── test_user.ext
│   └── features/
│       └── auth/
│           └── test_auth_service.ext
├── integration/
│   └── repositories/
│       └── test_order_repo.ext
└── e2e/
    └── test_user_journey.ext
```

---

## Test Helper Patterns

### Factories / Seeds

Create reusable functions that produce valid test data with sensible defaults. Override only the fields relevant to each test.

```
function createTestUser(overrides):
    return User(
        id: overrides.id ?? "test-user-123",
        name: overrides.name ?? "Test User",
        email: overrides.email ?? "test@example.com",
        createdAt: overrides.createdAt ?? now(),
    )
```

Benefits:
- Tests are not coupled to the full constructor signature.
- When a model gains a new required field, update the factory in one place.
- Each test shows only the fields that matter for that test case.

### Fixtures

Shared setup and teardown that multiple tests need (database connections, test servers, temporary directories).

Rules:
- Fixtures set up before and tear down after each test (or test group).
- Fixtures must not leak state between tests. Each test starts clean.
- Prefer per-test setup over shared mutable state.

### Mocks and Stubs

Use mocks for dependencies that the unit under test calls but that you do not want to exercise (databases, HTTP clients, file systems).

Rules:
- Mock at the boundary (repository interface, service interface), not deep internals.
- Prefer simple stubs (return a canned value) over complex mock verification.
- If a mock requires more than 10 lines of setup, the unit under test may have too many dependencies -- consider refactoring.

---

## Good Test Principles

### Test Behavior, Not Implementation

Test what the code does, not how it does it. If you refactor internals without changing behavior, tests should still pass.

Bad: "verify that `_internalHelper` was called with argument X"
Good: "verify that the output is Y given input X"

### One Concept Per Test

Each test verifies exactly one behavior. If a test has multiple unrelated assertions, split it into separate tests.

Bad: "test user creation" that checks validation, database insertion, and email sending.
Good: Three tests -- "rejects invalid email", "persists user to database", "sends welcome email".

### Descriptive Names

Test names should read as a sentence describing the expected behavior. Someone reading only the test name should understand what is being verified.

Bad: `test1`, `testCalculation`, `it works`
Good:
- `calculates_zero_price_when_all_inputs_are_minimum`
- `returns_error_when_user_not_found`
- `syncs_unsynced_records_on_connectivity_change`
- `rejects_empty_email_with_validation_error`

### Arrange-Act-Assert (AAA)

Every test has three distinct phases, separated by blank lines:

```
// Arrange: set up inputs, dependencies, and expected state
input = createTestInput(price: 100, quantity: 2)
service = createService(mockRepo)

// Act: perform the single operation being tested
result = service.calculateTotal(input)

// Assert: verify the outcome
assertEqual(result, 200)
```

Do not mix phases. Do not assert during arrangement. Do not act during assertion.

### Test Edge Cases

Every test suite should include tests for boundary conditions:

| Edge case | Example |
|-----------|---------|
| Empty input | Empty string, empty list, empty map |
| Null / missing values | Optional fields omitted, nullable fields set to null |
| Boundary values | 0, -1, MAX_INT, first/last element |
| Error paths | Network failure, invalid input, unauthorized access |
| Duplicate operations | Creating the same record twice, double-submitting a form |
| Concurrent operations | Two syncs running at once (if applicable) |

---

## When to Update Tests

**Rule: any behavioral change requires a test update.** If the behavior did not change, tests should still pass without modification.

| Change type | Test action required |
|-------------|---------------------|
| New feature | Add new test file(s) covering the feature |
| Bug fix | Add a test that reproduces the bug (fails before fix, passes after) |
| Refactor (no behavior change) | Tests must pass without modification. If they break, the refactor changed behavior. |
| New field on a model | Update factory/seed helpers. Add serialization round-trip test for the new field. |
| Changed validation rules | Update existing validation tests. Add tests for new boundary cases. |
| Deleted feature | Delete corresponding test files. |
| Changed API contract | Update integration tests for the new contract. |

---

## Multi-Language Test Structure Examples

### Python (pytest)

```python
# tests/unit/core/models/test_user.py

import pytest
from myproject.core.models.user import User


class TestUserFromJson:
    def test_parses_valid_json(self):
        # Arrange
        data = {"id": "123", "name": "Alice", "created_at": "2025-01-01T00:00:00Z"}

        # Act
        user = User.from_json(data)

        # Assert
        assert user.id == "123"
        assert user.name == "Alice"

    def test_returns_default_name_when_missing(self):
        data = {"id": "123", "created_at": "2025-01-01T00:00:00Z"}

        user = User.from_json(data)

        assert user.name == "Unknown"

    def test_raises_on_missing_id(self):
        data = {"name": "Alice", "created_at": "2025-01-01T00:00:00Z"}

        with pytest.raises(KeyError):
            User.from_json(data)


class TestUserCalculateDiscount:
    def test_returns_zero_discount_for_new_user(self):
        user = create_test_user(account_age_days=0)

        discount = user.calculate_discount()

        assert discount == 0.0

    def test_returns_ten_percent_for_veteran_user(self):
        user = create_test_user(account_age_days=365)

        discount = user.calculate_discount()

        assert discount == 0.10
```

### TypeScript (Jest / Vitest)

```typescript
// tests/unit/core/models/user.test.ts

import { describe, it, expect } from 'vitest';
import { User } from '@/core/models/user';
import { createTestUser } from '../../helpers/factories';

describe('User.fromJson', () => {
  it('parses valid JSON into a User instance', () => {
    const data = { id: '123', name: 'Alice', created_at: '2025-01-01T00:00:00Z' };

    const user = User.fromJson(data);

    expect(user.id).toBe('123');
    expect(user.name).toBe('Alice');
  });

  it('throws on missing required id field', () => {
    const data = { name: 'Alice', created_at: '2025-01-01T00:00:00Z' };

    expect(() => User.fromJson(data)).toThrow('id is required');
  });
});

describe('User.calculateDiscount', () => {
  it('returns zero for a new user', () => {
    const user = createTestUser({ accountAgeDays: 0 });

    const discount = user.calculateDiscount();

    expect(discount).toBe(0);
  });

  it('returns 10% for a user older than one year', () => {
    const user = createTestUser({ accountAgeDays: 365 });

    const discount = user.calculateDiscount();

    expect(discount).toBe(0.10);
  });
});
```

### Dart (flutter_test)

```dart
// test/unit/core/models/user_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:myapp/core/models/user.dart';
import '../../helpers/seed.dart';

void main() {
  group('User.fromJson', () {
    test('parses valid JSON into a User instance', () {
      // Arrange
      final data = {'id': '123', 'name': 'Alice', 'created_at': '2025-01-01T00:00:00Z'};

      // Act
      final user = User.fromJson(data);

      // Assert
      expect(user.id, '123');
      expect(user.name, 'Alice');
    });

    test('uses default name when name is missing', () {
      final data = {'id': '123', 'created_at': '2025-01-01T00:00:00Z'};

      final user = User.fromJson(data);

      expect(user.name, 'Unknown');
    });
  });

  group('User.calculateDiscount', () {
    test('returns zero discount for a new user', () {
      final user = seedUser(accountAgeDays: 0);

      final discount = user.calculateDiscount();

      expect(discount, 0.0);
    });

    test('returns 10% discount for veteran user', () {
      final user = seedUser(accountAgeDays: 365);

      final discount = user.calculateDiscount();

      expect(discount, 0.10);
    });
  });
}
```

### Go (testing)

```go
// internal/models/user_test.go

package models_test

import (
    "testing"

    "myproject/internal/models"
    "myproject/internal/testhelpers"
)

func TestUserFromJSON_ValidInput(t *testing.T) {
    // Arrange
    data := map[string]any{
        "id":         "123",
        "name":       "Alice",
        "created_at": "2025-01-01T00:00:00Z",
    }

    // Act
    user, err := models.UserFromJSON(data)

    // Assert
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.ID != "123" {
        t.Errorf("expected ID '123', got %q", user.ID)
    }
    if user.Name != "Alice" {
        t.Errorf("expected Name 'Alice', got %q", user.Name)
    }
}

func TestUserFromJSON_MissingID(t *testing.T) {
    data := map[string]any{
        "name":       "Alice",
        "created_at": "2025-01-01T00:00:00Z",
    }

    _, err := models.UserFromJSON(data)

    if err == nil {
        t.Fatal("expected error for missing ID, got nil")
    }
}

func TestCalculateDiscount_NewUser(t *testing.T) {
    user := testhelpers.CreateTestUser(t, testhelpers.WithAccountAgeDays(0))

    discount := user.CalculateDiscount()

    if discount != 0.0 {
        t.Errorf("expected 0.0 discount, got %f", discount)
    }
}

func TestCalculateDiscount_VeteranUser(t *testing.T) {
    user := testhelpers.CreateTestUser(t, testhelpers.WithAccountAgeDays(365))

    discount := user.CalculateDiscount()

    if discount != 0.10 {
        t.Errorf("expected 0.10 discount, got %f", discount)
    }
}
```

---

## Running Tests

<!-- ADAPT: Replace with your actual test commands. -->

```bash
# Run all unit tests
pytest tests/unit/                      # Python
npm test                                # TypeScript
flutter test test/unit/                 # Dart
go test ./internal/...                  # Go

# Run a specific test file
pytest tests/unit/core/models/test_user.py
npx vitest run tests/unit/core/models/user.test.ts
flutter test test/unit/core/models/user_test.dart
go test ./internal/models/ -run TestUserFromJSON

# Run integration tests
pytest tests/integration/
flutter test integration_test/
go test -tags=integration ./...

# Run with coverage
pytest --cov=myproject tests/
npx vitest run --coverage
flutter test --coverage
go test -cover ./...
```

Tests must pass before any commit reaches the main branch. Failing tests block CI.
