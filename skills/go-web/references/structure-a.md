# Go Web Single-App Architecture

Canonical name: `go-web-sa-arch`

Use this reference for repositories shaped like:

```text
myproject/
├── api/
│   ├── v1/
│   └── router.go
├── cmd/
│   ├── server/
│   └── worker/
├── config/
├── database/
├── deploy/
├── migrations/
├── pkg/
├── internal/
│   ├── service/
│   ├── dao/
│   ├── model/
│   ├── dto/
│   ├── middleware/
│   ├── worker/
│   └── common/
├── scripts/
├── go.mod
└── README.md
```

This reference describes a single-application Web service architecture with clear layer boundaries. One process can expose the main HTTP service, while another process can run background jobs, but both still belong to the same application boundary.

## Directory Responsibilities

- `api/v1/`: HTTP adapter layer. Keep handlers thin: parse input, call service, return unified responses.
- `api/router.go`: Route assembly entry. Register versioned routes, middleware, and public API boundaries.
- `cmd/server/`: Web process entry. Initialize dependencies and start the main HTTP service.
- `cmd/worker/`: Background process entry. Start workers, schedulers, or queue consumers without coupling them to HTTP routing.
- `config/`: Configuration structs, loading logic, and environment-specific configuration mapping.
- `database/`: Database and infrastructure client initialization, connection management, and transaction entry points.
- `deploy/`: Deployment-facing assets such as Docker, Kubernetes, Helm, or CI deployment templates.
- `migrations/`: Database schema migrations or migration version files.
- `pkg/`: Cross-module or cross-project utilities. Do not place strongly business-specific code here.
- `internal/service/`: Use-case orchestration layer for validation, flow control, pagination normalization, aggregation, and error mapping.
- `internal/dao/`: Data access layer for CRUD, conditional queries, batch writes, and persistence details. Do not put HTTP semantics here.
- `internal/model/`: Persistence entities, usually ORM models or storage-facing structs. Do not use them as API presentation shapes by default.
- `internal/dto/`: Transport contracts for request binding and response projections. Do not encode persistence rules here.
- `internal/middleware/`: Cross-cutting application concerns such as auth, logging, tracing, rate limiting, and recovery.
- `internal/worker/`: Background job handlers and worker execution logic.
- `internal/common/`: Low-coupling shared definitions such as constants, error codes, enums, and lightweight shared types.
- `scripts/`: Build, release, operations, and developer support scripts.

## Dependency Direction

Preferred direction:

```text
cmd -> api -> service -> dao -> database
                     -> model
api -> dto
service -> dto, model, dao, common
pkg/config/database are infrastructure dependencies
```

Keep the direction mostly one-way. Handler depends on service, not DAO. DAO should not import handler code. Model should not depend on Gin. DTO should not control database behavior. Worker code may call service or DAO depending on the repository pattern, but should still avoid reverse dependencies on the HTTP layer.

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

Service owns use-case execution: request validation, orchestration, pagination normalization, aggregation, and error mapping.

- bind or accept validated request DTOs according to the repository convention
- validate required business constraints explicitly
- normalize page and page size centrally
- call DAO functions with plain values
- convert lower-level errors into app errors with HTTP status codes
- use service for cross-entity composition and aggregation

Prefer explicit flow over abstraction-heavy service frameworks. Service should express business steps clearly and should not hide key rules inside middleware or DAO.

### DAO

DAO exposes direct, readable persistence helpers.

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

DAO should answer storage questions, not transport questions. If a result is only for API presentation, prefer shaping it in DTO or service unless the repository already uses DAO projections consistently.

### Model

Use models for persistence-facing entities and storage mapping.

- keep structs plain and explicit
- avoid adding transport-only concerns into model types
- add helper methods only when behavior truly belongs to the entity itself

### DTO

Use DTOs for transport boundaries and interface contracts.

- request DTOs carry binding tags
- response DTOs carry projections or paging wrappers
- when model and response shape are identical enough, follow the local repository pattern instead of forcing a separate response DTO

DTO is the right place for request binding tags, pagination parameters, filtering fields, and response-only projections.

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

Handlers should serialize errors. Services should interpret them.

## File Placement Rules

When adding a new resource or module:

1. add handler functions under `api/v1/` or the existing API folder
2. add service functions in `internal/service/<module>_service.go`
3. add persistence functions in `internal/dao/<module>_dao.go`
4. add or extend models in `internal/model/`
5. add request or response DTOs in `internal/dto/` if transport shape differs
6. register routes in `api/router.go` or the existing router file
7. extract shared bind/paging/validation helpers only after repetition is obvious

If the feature is background-only, prefer `internal/worker/` plus `cmd/worker/` instead of exposing an HTTP entry.

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
