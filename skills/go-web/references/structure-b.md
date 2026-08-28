# Go Web Multi-App Architecture

Canonical name: `go-web-ma-arch`

Use this reference for repositories shaped like:

```text
myproject/
├── cmd/
│   ├── admin/
│   └── client/
├── config/
├── database/
├── deploy/
├── migrations/
├── pkg/
│   ├── logger/
│   └── response/
├── internal/
│   ├── app/
│   │   ├── admin/
│   │   │   ├── api/
│   │   │   │   ├── v1/
│   │   │   │   └── router.go
│   │   │   ├── service/
│   │   │   ├── dao/
│   │   │   ├── model/
│   │   │   └── dto/
│   │   └── client/
│   │       ├── api/
│   │       │   ├── v1/
│   │       │   └── router.go
│   │       ├── service/
│   │       ├── dao/
│   │       ├── model/
│   │       └── dto/
│   ├── middleware/
│   └── common/
├── scripts/
├── go.mod
└── README.md
```

This reference describes a multi-application repository where multiple delivery surfaces share infrastructure, but each app keeps its own three layers: API, service, and DAO, plus that app's model and DTO.

## Core Principle

Separate by application boundary first, by technical layer second. Decide which app owns the feature before deciding which layer inside that app owns the code. Inside an app, keep the same three-layer rules as Structure A.

## Directory Responsibilities

- `cmd/<app>/`: Process entry for one app. Assemble shared infrastructure and start that app's service.
- `config/`: Shared configuration structs, loading, required file handling, validation, and resolved DSN helpers.
- `database/`: Shared GORM setup, driver selection, DB holder, close logic, and GORM logger adapters.
- `deploy/`: Deployment-facing assets such as Docker, Kubernetes, Helm, or CI deployment templates.
- `migrations/`: Shared schema migrations or app-specific migration assets when the repository uses them.
- `pkg/response/`: Shared HTTP response envelope helpers.
- `pkg/logger/`: Shared logging setup and package-level logging helpers.
- `internal/app/<app>/api/v1/`: App-specific presentation layer.
- `internal/app/<app>/api/router.go`: App-specific route assembly entry.
- `internal/app/<app>/service/`: App-specific business layer.
- `internal/app/<app>/dao/`: App-specific data layer.
- `internal/app/<app>/model/`: App-specific persistence entities or projections.
- `internal/app/<app>/dto/`: App-specific request and response contracts.
- `internal/middleware/`: Cross-cutting middleware reusable across apps.
- `internal/common/`: Low-coupling constants, reusable domain errors, and pagination defaults.
- `scripts/`: Build, release, operations, and developer support scripts.

## Dependency Direction

Preferred direction inside one app:

```text
cmd/<app> -> internal/app/<app>/api -> api/v1 -> service -> dao -> database
                                                -> dto/model
                                                -> common
infra: config, database, pkg, internal/common, internal/middleware
```

Cross-app imports should be rare. Prefer extracting truly shared behavior into a neutral shared package rather than importing one app's private service package from another. Shared infrastructure can be common; business ownership should stay app-local unless there is a strong reason to merge.

Each app service must not import Gin or GORM. Each app DAO may accept that app's query DTOs.

## Coding Style

### App-First Organization

Before writing code, decide which app owns the feature.

- admin-only feature -> `internal/app/admin/...`
- client-only feature -> `internal/app/client/...`
- truly shared edge concern -> `internal/middleware` or another narrow shared package
- truly shared infrastructure -> `config`, `database`, `pkg`, or `internal/common`

Do not put app-specific request DTOs, handlers, or service rules in shared locations.

### Handler Style

Reuse the thin Gin handler pattern inside each app.

- receive `*gin.Context`
- parse path parameters and bind query/body DTOs at the edge
- call one app service function with `c.Request.Context()`
- return success through the shared response helper
- return failures through `response.Error`
- keep app-specific protocol exceptions at the edge

The handler layer should explain how the app speaks to the outside world, not how the business works internally.

### Service Style

Reuse the same plain-error service style inside each app.

- accept `context.Context` plus app DTO/plain values
- normalize payloads with small helpers
- validate business constraints explicitly
- normalize pagination centrally
- call app DAO functions with context and plain values or query DTOs
- coordinate common helpers, external clients, session checks, audit, or encryption as needed
- map model values to app response DTOs
- return plain `error` values unless the repository already uses app-error wrappers

Service is still the main business layer in a multi-app repository; app split does not weaken layer discipline. App services must not import Gin or GORM.

### DAO Style

Reuse the same direct DAO style inside each app.

- accept `context.Context` as the first parameter
- use the shared DB holder with `WithContext(ctx)` unless the repository injects DB handles already
- keep query logic readable
- return model slices, projections, totals, single models, or scalar counts explicitly
- map `gorm.ErrRecordNotFound` to shared or app-local not-found errors
- own transactions for multi-step persistence
- avoid HTTP types and response wrappers in DAO

DAO belongs to the app when query shape or ownership is app-specific. Only move DAO code to shared packages when multiple apps truly need the same persistence behavior.

### Shared Code Discipline

Move code into shared packages only when all of these are true:

- at least two apps need it now
- the behavior is conceptually the same
- moving it will not force unrelated dependencies into both apps
- naming remains clear outside the original app context

Good shared candidates:

- response envelope helpers
- logger setup
- config and database setup
- common error values and constants
- pagination defaults, error values, and constants in `internal/common`
- session middleware when behavior is identical across apps
- external-resource clients with no app-specific business rules

Bad shared candidates:

- app-specific handlers
- app-specific DTOs
- app-specific service rules
- app-specific query projections
- business rules that only one app understands

### Config, Logger, and Database Infrastructure

Keep shared infrastructure neutral and reusable across app entries.

- `cmd/<app>` should initialize config, Zap logger, database, migrations, app routes, and HTTP startup explicitly
- `config` owns Viper setup, config file discovery, required field validation, normalization, and app-specific derived values
- `pkg/logger` owns Zap construction, shared accessors, `Sync`, and small helper functions
- `internal/middleware` owns HTTP request logging middleware and should depend only on shared logger/response/session helpers
- `database` owns GORM setup and any adapter that implements `gorm.io/gorm/logger.Interface`
- GORM SQL logging should be configured through `database.Init`, not scattered through app DAO functions
- app-specific services and DAOs should not import Viper or initialize Zap

## Naming Guidance

Apply the same naming flavor inside each app subtree.

- files remain `<module>_service.go`, `<module>_dao.go`, and `<module>.go`
- handlers remain direct and action-oriented: `GetXList`, `CreateX`, `UpdateX`, `DeleteX`, `PreviewX`, `RefreshX`, `ValidateX`
- services remain use-case oriented: `ListX`, `CreateX`, `UpdateX`, `GetX`, `ResolveX`
- DAOs remain storage-oriented: `GetXByID`, `GetXByName`, `ListX`, `CountX`, `ReplaceXChildren`
- DTOs remain explicit: `XRequest`, `XResponse`

Prefer local clarity over generic cross-app names.

## Error Handling Style

Keep the same error-handling pattern, scoped per app.

- reusable errors may live in `internal/common`
- app-specific errors may live under that app if they are not shared
- DAOs should normalize storage misses near the storage boundary
- services should return plain errors and interpret business rules
- handlers should choose HTTP status codes and serialize errors
- do not add app-specific error wrappers unless the repository already uses that pattern

Handlers serialize errors. App services interpret rules. App DAOs normalize persistence details.

## File Placement Rules

When adding a feature:

1. identify the owning app first
2. create or extend files only inside that app subtree
3. add request/query/response DTOs under `internal/app/<app>/dto/` if needed
4. add or extend `internal/app/<app>/service/<module>_service.go`
5. add or extend `internal/app/<app>/dao/<module>_dao.go`
6. add or extend `internal/app/<app>/model/` only if persistence or projection structs change
7. add or extend `internal/app/<app>/api/v1/<module>.go`
8. register the route in that app's router
9. extract shared helpers only if a second app genuinely needs the same behavior

If a feature is only used by admin, keep it in admin even if client may someday need something similar. Duplicate a small amount first; extract later when the shared shape is real.

## Standard API Addition Flow

For Structure B, add a new interface in this order:

1. identify the owning app before writing any code
2. inspect adjacent files inside that app
3. add request/query/response DTOs under `internal/app/<app>/dto/` if needed
4. add or extend `internal/app/<app>/service/<module>_service.go`
5. add or extend `internal/app/<app>/dao/<module>_dao.go`
6. add or extend `internal/app/<app>/model/` only if the app needs new persistence or projection structs
7. add or extend `internal/app/<app>/api/v1/<module>.go`
8. register the route in that app's router
9. move to shared packages only after duplication proves it is worth it

Preferred ownership by concern:

- app-specific path/query/body parsing -> app handler
- app-specific request and response shape -> app DTO
- app-specific normalization, validation, orchestration -> app service
- app-specific query behavior -> app DAO
- error values, constants, or pagination defaults -> `internal/common`
- final response serialization -> app handler through shared response helpers

## Common Anti-Patterns

Avoid these:

- putting all handlers for every app into one global `api/`
- importing one app's private service package from another app
- moving app DTOs into shared packages prematurely
- creating a huge `common` package that hides ownership
- adding an `internal/utils` package or a `shared` subdirectory under `common`; put pagination and constants in `internal/common`
- coupling unrelated app startup logic inside one `cmd` entry
- weakening layer boundaries just because the repo has multiple apps
- letting app services import Gin or GORM
- introducing repository interfaces or dependency injection frameworks when the existing code uses direct package helpers
- introducing app-error wrappers in a plain-error codebase
