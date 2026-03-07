# Architecture

<!-- ADAPT: Replace this introduction with a 2-3 sentence summary of what your system does and who it serves. -->

This document defines the system architecture, layer responsibilities, dependency rules, and data flow patterns. Layers described here are enforced mechanically via `enforcement/` rules, not just documented -- violations fail CI.

---

## System Overview

<!-- ADAPT: Replace this diagram with your actual system topology. Show external services, databases, entry points, and the boundaries between them. -->

```
┌──────────────────────────────────────────────────────┐
│                  External Services                    │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐ │
│  │  Auth       │  │  Database  │  │  Object Store  │ │
│  │  Provider   │  │  (SQL/Doc) │  │  (Files/Blobs) │ │
│  └─────┬──────┘  └─────┬──────┘  └───────┬────────┘ │
└────────┼───────────────┼──────────────────┼──────────┘
         │               │                  │
    ┌────┴───────────────┴──────────────────┴────┐
    │            Infrastructure Layer             │
    │     (SDK clients, HTTP adapters, config)    │
    └────┬───────────────────────────────────┬───┘
         │                                   │
┌────────┴──────────┐          ┌─────────────┴──────────┐
│   Entry Point A    │          │   Entry Point B         │
│   (API / Mobile)   │          │   (CLI / Admin Panel)   │
└────────────────────┘          └─────────────────────────┘
```

---

## Layer Definitions

The codebase is organized into layers with strict dependency rules. Each layer may only depend on layers below it -- never sideways or upward.

```
  ┌─────────────────────────┐
  │   UI / API / CLI        │  ← Entry points, request handlers, screens
  ├─────────────────────────┤
  │   Services              │  ← Business logic, orchestration, use cases
  ├─────────────────────────┤
  │   Repositories          │  ← Data access, external service wrappers
  ├─────────────────────────┤
  │   Models                │  ← Data structures, enums, value objects
  └─────────────────────────┘
```

### Models (bottom layer)

Pure data structures with no dependencies on any other layer. Models define the shape of data flowing through the system.

Responsibilities:
- Data classes and value objects
- Enums and constants
- Serialization logic (fromJson/toJson, fromMap/toMap)
- Validation rules intrinsic to the data itself

Dependency rule: **Models depend on nothing** (only the language standard library).

<!-- ADAPT: List your actual model files and what they represent. -->

### Repositories

Thin wrappers around data sources. A repository translates between external representations (database rows, API responses, file contents) and domain models.

Responsibilities:
- CRUD operations against databases
- HTTP calls to external APIs
- File system reads/writes
- Cache reads/writes

Dependency rule: **Repositories depend on Models only.**

<!-- ADAPT: List your repositories and what data source each wraps. -->

### Services

Business logic and orchestration. A service coordinates one or more repositories to fulfill a use case. Services contain the "why" of the application -- the rules, calculations, and workflows.

Responsibilities:
- Business rule enforcement
- Multi-repository coordination (e.g., save record then upload file)
- Computation and transformation logic
- Background jobs and sync workflows

Dependency rule: **Services depend on Models and Repositories only.**

<!-- ADAPT: List your services and what use cases each handles. -->

### UI / API / CLI (top layer)

The entry point layer. This is where user input arrives and responses leave. It should be thin -- delegate logic to services immediately.

Responsibilities:
- HTTP route handlers / controller methods
- Screen widgets and UI components
- CLI command definitions and argument parsing
- Request validation and response formatting
- State management (providers, stores, reducers)

Dependency rule: **UI/API/CLI depends on Services and Models. Never import Repositories directly.**

<!-- ADAPT: List your entry points and what each serves. -->

---

## Dependency Rules Summary

| Layer | May depend on | Must NOT depend on |
|-------|---------------|---------------------|
| Models | Standard library only | Repositories, Services, UI/API |
| Repositories | Models | Services, UI/API |
| Services | Models, Repositories | UI/API |
| UI / API / CLI | Models, Services | Repositories (go through Services) |

These rules are enforced by lint rules or import restriction checks in `enforcement/`. A pull request that violates a dependency rule will not pass CI.

---

## Data Flow

<!-- ADAPT: Replace this with your primary data flow. Show the path from user action to persistence and back. -->

### Write Path (user creates data)

```
User action
  → UI/API layer validates input format
    → Service applies business rules and computes derived values
      → Repository persists to database / uploads to storage
        → Service returns result
          → UI/API layer formats and returns response
```

### Read Path (user retrieves data)

```
User request
  → UI/API layer parses parameters
    → Service applies authorization and filtering logic
      → Repository queries database / fetches from storage
        → Service transforms and enriches data
          → UI/API layer formats and returns response
```

### Background Sync Path (offline-capable systems)

```
Data written to local store (offline)
  → Connectivity change detected
    → Sync service queries unsynced records
      → For each record: upload assets → insert remote record → mark synced
        → Failures are retried on next connectivity event
```

---

## External Service Integrations

<!-- ADAPT: Replace this table with your actual external services. -->

| Service | Purpose | Layer that owns the integration |
|---------|---------|----------------------------------|
| Auth Provider | Authentication, session management | Repository (auth_repository) |
| Primary Database | Persistent data storage | Repository (data_repository) |
| Object Storage | File/blob storage (images, PDFs) | Repository (storage_repository) |
| Notification Service | Push/local notifications | Service (notification_service) |

### Integration Rules

1. **External service calls live in Repositories only.** No raw HTTP calls or SDK usage in Services or UI.
2. **Credentials are never hardcoded.** Pass via environment variables, build-time defines, or secret managers.
3. **Wrap SDKs in thin adapters.** This isolates the codebase from SDK version changes and makes testing possible.
4. **Handle connectivity failures gracefully.** Every external call must have a timeout and a failure path.

---

## Entry Points

<!-- ADAPT: Replace with your actual entry points. -->

| Entry point | File | Target platform | Purpose |
|-------------|------|-----------------|---------|
| Main app | `lib/main.dart` or `src/index.ts` | Mobile / Server | Primary user-facing application |
| Admin panel | `lib/main_admin.dart` or `src/admin.ts` | Web | Administrative interface |
| CLI tool | `bin/cli.dart` or `src/cli.ts` | Terminal | Developer/operator tooling |
| Worker | `src/worker.ts` | Server | Background job processing |

Each entry point bootstraps its own dependency graph (database connections, service instances, router configuration) and should share core/ code but never import from another entry point.

---

## Mapping Layers to Project Types

The four-layer model adapts to any project type. The layers stay the same; only the top layer changes.

### Web API

```
Routes / Controllers  ← top layer (handles HTTP requests, returns JSON)
Services              ← business logic
Repositories          ← database queries, external API calls
Models                ← request/response schemas, domain entities
```

### Mobile App

```
Screens / Widgets     ← top layer (UI rendering, user interaction)
Services / Providers  ← business logic, state management
Repositories          ← local DB, remote API, file storage
Models                ← data classes, enums
```

### CLI Tool

```
Commands / Handlers   ← top layer (argument parsing, output formatting)
Services              ← business logic, orchestration
Repositories          ← file system, API calls, config files
Models                ← data structures, option types
```

### Shared Library

```
Public API surface    ← top layer (exported functions and classes)
Internal logic        ← business logic (unexported)
Adapters              ← external integrations (unexported)
Types                 ← data structures (exported)
```

In every case, the dependency direction flows downward. The top layer is the thinnest. The bottom layers are the most reusable and testable.

---

## Enforcement

Layer boundaries are not suggestions. They are enforced by:

1. **Import restriction rules** in `enforcement/` that fail CI on violations
2. **Code review automation** that flags cross-layer imports
3. **Directory structure** that makes violations visually obvious

<!-- ADAPT: Reference your specific enforcement mechanism (lint rules, custom scripts, architectural test files). -->

If you need to break a layer boundary, that is a signal to refactor -- extract a shared model, introduce a service method, or create a new repository.
