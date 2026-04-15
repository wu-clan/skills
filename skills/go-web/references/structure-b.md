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
│   │   │   ├── api/
│   │   │   │   ├── v1/
│   │   │   │   └── router.go
│   │   │   ├── service/
│   │   │   ├── dao/
│   │   │   ├── model/
│   │   │   └── dto/
│   ├── middleware/
│   └── common/
├── scripts/
├── go.mod
└── README.md
```

This reference describes a multi-application repository where multiple delivery surfaces share infrastructure, but each app keeps its own API, service, DAO, model, and DTO boundaries.

## Core Principle

Separate by application boundary first, by technical layer second. In other words, decide which app owns the feature before deciding which layer inside that app owns the code.

## Directory Responsibilities

- `cmd/admin/`: Admin process entry. Assemble admin dependencies and start the admin-facing service.
- `cmd/client/`: Client process entry. Assemble client dependencies and start the client-facing service.
- `config/`: Configuration structs, loading logic, and environment-specific configuration mapping.
- `database/`: Infrastructure connection setup such as database, cache, or message-broker clients.
- `deploy/`: Deployment-facing assets.
- `migrations/`: Database schema versioning and migration assets.
- `pkg/`: Cross-app or cross-project reusable utilities.
- `internal/app/admin/api/v1/`: Admin HTTP adapter layer.
- `internal/app/admin/api/router.go`: Admin route assembly entry.
- `internal/app/admin/service/`: Admin use-case orchestration layer.
- `internal/app/admin/dao/`: Admin data access layer.
- `internal/app/admin/model/`: Admin persistence entities or internal projections.
- `internal/app/admin/dto/`: Admin request and response contracts.
- `internal/app/client/api/v1/`: Client HTTP adapter layer.
- `internal/app/client/api/router.go`: Client route assembly entry.
- `internal/app/client/service/`: Client use-case orchestration layer.
- `internal/app/client/dao/`: Client data access layer.
- `internal/app/client/model/`: Client persistence entities or internal projections.
- `internal/app/client/dto/`: Client request and response contracts.
- `internal/middleware/`: Cross-cutting middleware reusable across apps.
- `internal/common/`: Low-coupling shared definitions such as constants, error codes, enums, and lightweight shared types.
- `scripts/`: Build, release, operations, and developer support scripts.

## Dependency Direction

Preferred direction inside one app:

```text
cmd/<app> -> internal/app/<app>/api -> service -> dao
                                      -> dto/model
shared infra: config, database, pkg, internal/common, internal/middleware
```

Cross-app imports should be rare. Prefer extracting truly shared logic into a neutral shared package rather than importing one app's private service package from another. Shared infra can be common; business ownership should stay app-local unless there is a very strong reason to merge.

## Coding Style

### App-First Organization

Before writing code, decide which app owns the feature. This is the first question, not a later refactor step.

- admin-only feature -> `internal/app/admin/...`
- client-only feature -> `internal/app/client/...`
- truly shared concern -> shared package outside app folders

Do not put app-specific request DTOs, handlers, or service rules in shared locations.

### Handler Style

Reuse the thin-handler pattern inside each app.

- handler receives protocol context
- handler delegates almost immediately to service
- handler writes unified success/error response
- app-specific protocol exceptions stay at the edge

The handler layer should explain how the app speaks to the outside world, not how the business works internally.

### Service Style

Reuse the same service style inside each app.

- service binds or validates request data if that is the local repository convention
- service owns paging normalization and business validation
- service calls DAO with plain values
- service maps domain and persistence errors into app-level errors
- service coordinates multiple DAO calls or shared helpers

Service is still the main business layer even in a multi-app repository; app split does not weaken layer discipline.

### DAO Style

Reuse the same DAO style inside each app.

- keep functions direct and intention-revealing
- keep query logic readable
- return model slices, projections, totals, or single models explicitly
- avoid HTTP types and app response wrappers in DAO

DAO belongs to the app when query shape or ownership is app-specific. Only move DAO code to shared packages when multiple apps truly need the same persistence behavior.

### Shared Code Discipline

Move code into shared packages only when all of these are true:

- at least two apps need it
- the behavior is conceptually the same
- moving it will not force unrelated dependencies into both apps
- naming remains clear outside the original app context

Good shared candidates in this style:

- common error values
- pagination defaults
- bind helpers if multiple apps share the same transport stack
- logging and response utilities

Bad shared candidates are app-specific handlers, app-specific DTOs, and business rules that only one app understands.

## Naming Guidance

Apply the same naming flavor inside each app subtree.

- files remain `<module>_service.go`, `<module>_dao.go`
- handler names remain direct and action-oriented
- DTO names remain explicit and request-shape-oriented
- helper names remain narrow and practical, not overly abstract

## Error Handling Style

Keep the same error-handling pattern, but scoped per app.

- common reusable errors may live in `internal/common`
- app services should convert infra/domain errors into app-level response errors
- handlers should not implement business error mapping themselves
- record-not-found handling should stay near service logic

Handlers serialize errors. App services interpret them.

## File Placement Rules

When adding a feature:

1. identify the owning app first
2. create or extend files only inside that app subtree
3. add routes in that app's router
4. keep request/response DTOs under that app
5. keep handler/service/dao separation strict
6. move to shared packages only after duplication proves it is worth it

If a feature is only used by admin, keep it in admin even if client may someday need something similar. Duplicate a small amount first; extract later when the shared shape is real.

## Standard API Addition Flow

For Structure B, add a new interface in this order:

1. identify the owning app before writing any code
2. add request or response DTOs under `internal/app/<app>/dto/` if needed
3. add or extend `internal/app/<app>/service/<module>_service.go`
4. add or extend `internal/app/<app>/dao/<module>_dao.go`
5. add or extend `internal/app/<app>/model/` only if the app needs new persistence or projection structs
6. add or extend `internal/app/<app>/api/<module>.go`
7. register the route in that app's router
8. extract shared helpers only if a second app genuinely needs the same behavior

Preferred ownership by concern:

- app-specific request binding -> app DTO plus app service
- app-specific orchestration -> app service
- app-specific query behavior -> app DAO
- shared error values or helper defaults -> `internal/common` or another narrow shared package
- final response serialization -> app handler

## Common Anti-Patterns

Avoid these:

- putting all handlers for every app into one global `api/`
- importing one app's private service package from another app
- moving app DTOs into shared packages prematurely
- creating huge `common` or `utils` packages that hide ownership
- coupling unrelated app startup logic inside one `cmd` entry
- weakening layer boundaries just because the repo has multiple apps
