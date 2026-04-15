# Structure B: Multi-Entry / Multi-App Go Service

Use this reference for repositories shaped like:

```text
cmd/
  admin/
  client/
internal/
  app/
    admin/
      api/
      service/
      dao/
      model/
      dto/
    client/
      api/
      service/
      dao/
      model/
      dto/
  common/
  middleware/
config/
database/
pkg/
```

This reference explains how to apply the same coding discipline to a multi-entry or multi-app repository.

## Core Principle

Separate by application boundary first, by technical layer second. Then apply the same thin-handler, service-first, explicit-DAO style inside each app.

## Directory Responsibilities

- `cmd/<app>`: process entry for one application surface
- `internal/app/<app>/api/`: protocol adapter for that app only
- `internal/app/<app>/service/`: use cases for that app only
- `internal/app/<app>/dao/`: storage access scoped to that app's needs
- `internal/app/<app>/model/`: app-local model or projection types when needed
- `internal/app/<app>/dto/`: app-local request/response contracts
- `internal/common/`: shared errors, constants, enums, small common helpers
- `internal/middleware/`: cross-app interceptors
- `config/`, `database/`, `pkg/`: shared infrastructure

## Dependency Direction

Preferred direction inside one app:

```text
cmd/<app> -> internal/app/<app>/api -> service -> dao
                                      -> dto/model
shared infra: config, database, pkg, internal/common, internal/middleware
```

Cross-app imports should be rare. Prefer extracting truly shared logic into a neutral shared package rather than importing one app's service package from another.

## Coding Style

### App-First Organization

Before writing code, decide which app owns the feature.

- admin-only feature -> `internal/app/admin/...`
- client-only feature -> `internal/app/client/...`
- truly shared concern -> shared package outside app folders

Do not put app-specific request DTOs or handlers in shared locations.

### Handler Style

Reuse the thin-handler pattern inside each app.

- handler receives protocol context
- handler delegates almost immediately to service
- handler writes unified success/error response
- app-specific protocol exceptions stay at the edge

### Service Style

Reuse the same service style inside each app.

- service binds or validates request data if that is the local repository convention
- service owns paging normalization and business validation
- service calls DAO with plain values
- service maps domain and persistence errors into app-level errors
- service coordinates multiple DAO calls or shared helpers

### DAO Style

Reuse the same DAO style inside each app.

- keep functions direct and intention-revealing
- keep query logic readable
- return model slices, projections, totals, or single models explicitly
- avoid HTTP types and app response wrappers in DAO

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

## File Placement Rules

When adding a feature:

1. identify the owning app first
2. create or extend files only inside that app subtree
3. add routes in that app's router
4. keep request/response DTOs under that app
5. keep handler/service/dao separation strict
6. move to shared packages only after duplication proves it is worth it

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
