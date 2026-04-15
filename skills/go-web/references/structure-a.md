# Structure A: Single-Application Layered Go Service

Use this reference for repositories shaped like:

```text
api/
cmd/
config/
database/
internal/
  service/
  dao/
  model/
  dto/
pkg/
```

This reference explains how to apply a practical coding style to a single-application layered Go service.

## Directory Responsibilities

- `cmd/<entry>`: bootstrap the process, load config, init logger, init DB, register routes, start servers, handle shutdown
- `api/`: HTTP routing and handler/controller entry points
- `internal/service/`: business orchestration, validation flow, paging rules, aggregation, error mapping
- `internal/dao/`: persistence and query logic, batch upsert, record lookup, list/count queries
- `internal/model/`: persistence models and schema-facing structs
- `internal/dto/`: request and response transport structs when they differ from models
- `database/`: connection lifecycle and migrations
- `config/`: config loading and typed config objects
- `pkg/`: cross-cutting utilities that are not domain-specific

## Dependency Direction

Preferred direction:

```text
cmd -> api -> service -> dao -> database
                     -> model
api -> dto
service -> dto, model, dao, common
pkg/config/database are infrastructure dependencies
```

Keep the direction mostly one-way. A DAO should not import handler code. A model should not depend on Gin. A DTO should not control database behavior.

## Coding Style

### Handler / Controller

Keep handlers extremely thin.

- receive `*gin.Context`
- call one service function
- translate `*response.AppError` to `response.Error`
- return success through `response.Success`
- avoid business branching unless it is purely protocol-specific

Preferred shape:

```go
func GetX(c *gin.Context) {
	data, err := service.GetX(c)
	if err != nil {
		response.Error(c, err.Code, err.Error())
		return
	}
	response.Success(c, data)
}
```

### Service

Service owns request binding, validation, orchestration, pagination normalization, and error mapping.

- bind request DTO with shared helper when possible
- validate required business constraints explicitly
- normalize page and page size centrally
- call DAO functions with plain values
- convert lower-level errors into app errors with HTTP status codes
- use service for cross-entity composition and aggregation

Prefer explicit flow over abstraction-heavy service frameworks.

### DAO

DAO exposes direct, readable query helpers.

- `GetXByID`
- `ListX`
- `BatchUpsertX`
- `CountX` or summary projections when needed

DAO rules:

- accept plain parameters
- use `database.DB` or the repository's DB holder directly
- keep query chains readable
- return model pointers or slices plus totals when pagination needs them
- never accept `*gin.Context`

### Model

Use models for persistence-facing entities.

- keep structs plain and explicit
- avoid adding transport-only concerns into model types
- add helper methods only when behavior truly belongs to the entity

### DTO

Use DTOs for transport boundaries.

- request DTOs carry binding tags
- response DTOs carry projections or paging wrappers
- when model and response shape are identical enough, follow the local repository pattern instead of forcing a separate response DTO

## Naming Guidance

Prefer direct, repeated naming:

- files: `<module>_service.go`, `<module>_dao.go`
- handler names: `GetXList`, `GetXDetail`, `StartSync`
- DAO names: `GetXByID`, `ListX`, `BatchUpsertX`, `ListXSummaries`
- request DTOs: `QueryByQQRequest`, `QueryByTargetRequest`, `SyncRequest`
- helper names: `normalizePage`, `bindQuery`, `bindJSON`, `validateQQ`

Prefer direct names over clever generic names.

## Error Handling Style

Keep error handling small and consistent:

- define common reusable domain errors in `internal/common`
- use an application error wrapper like `response.AppError`
- map validation errors to `400`
- map not found to `404`
- map unexpected persistence or infra errors to `500`
- use `errors.Is(err, gorm.ErrRecordNotFound)` style checks in service, not in handlers

## File Placement Rules

When adding a new resource or module:

1. add handler functions under `api/v1/` or the existing API folder
2. add service functions in `internal/service/<module>_service.go`
3. add persistence functions in `internal/dao/<module>_dao.go`
4. add or extend models in `internal/model/`
5. add request or response DTOs in `internal/dto/` if transport shape differs
6. register routes in `api/router.go` or the existing router file
7. extract shared bind/paging/validation helpers only after repetition is obvious

## Standard API Addition Flow

For Structure A, add a new interface in this order:

1. decide the module name and reuse an existing module file when the feature belongs to that module
2. add request or response DTOs in `internal/dto/` if needed
3. add or extend `internal/service/<module>_service.go`
4. add or extend `internal/dao/<module>_dao.go`
5. add or extend `internal/model/` only if persistence shape changes
6. add or extend `api/v1/<module>.go`
7. register the route in `api/router.go`

Preferred ownership by concern:

- bind query or JSON -> service helper plus DTO
- validate business input -> service
- fetch or persist data -> DAO
- map storage error to HTTP-friendly app error -> service
- write success or error response -> handler

## Common Anti-Patterns

Avoid these:

- handlers calling `database.DB` directly
- service logic copied into multiple handlers
- DAO functions returning HTTP-specific response objects
- request binding spread across handlers and services inconsistently
- generic repository abstractions that make simple CRUD harder to read
- one giant service file owning unrelated modules
