# Go Web Single-App Architecture

Canonical name: `go-web-sa-arch`

Use this reference for repositories shaped like a Go module root. If a monorepo stores the service in a subdirectory, treat that actual Go module directory as the root for this structure.

```text
myproject/
├── api/
│   ├── v1/
│   └── router.go
├── cmd/
│   └── server/
├── config/
├── database/
├── deploy/
├── migrations/
├── pkg/
│   ├── logger/
│   └── response/
├── internal/
│   ├── service/
│   ├── dao/
│   ├── model/
│   ├── dto/
│   ├── middleware/
│   ├── utils/
│   └── common/
├── scripts/
├── go.mod
├── Makefile
└── README.md
```

This reference describes a single-application web service architecture with clear layer boundaries. A secondary command or migration entry may exist, but the HTTP API, service layer, DAO layer, and support packages still belong to one application boundary.

## Directory Responsibilities

- `api/v1/`: HTTP adapter layer. Parse Gin inputs, call services, and write unified responses.
- `api/router.go`: Route assembly entry. Register health checks, versioned groups, middleware, and route groups.
- `cmd/server/`: Web process entry. Load config, initialize logger/database, run migrations, register routes, start HTTP, and shut down gracefully.
- `config/`: Configuration structs, Viper loading, required file handling, normalization, validation, and resolved DSN helpers.
- `database/`: GORM setup, driver selection, DB holder, close logic, GORM logger adapters, and database-specific setup helpers.
- `deploy/`: Deployment-facing assets such as Docker, Kubernetes, Helm, or CI deployment templates.
- `migrations/`: SQL migrations, embedded migration files, or migration runner helpers.
- `pkg/response/`: Shared HTTP response envelope helpers.
- `pkg/logger/`: Logging setup and package-level logging helpers.
- `internal/service/`: Use-case orchestration layer for normalization, validation, pagination, external calls, DTO mapping, side effects, and error propagation.
- `internal/dao/`: Data access layer for CRUD, conditional queries, counts, transactions, and not-found mapping.
- `internal/model/`: Persistence entities and storage-facing helpers.
- `internal/dto/`: Request payloads, query structs, response projections, pagination wrappers, and external API contracts.
- `internal/middleware/`: Cross-cutting HTTP concerns such as session checks, CORS, logging, tracing, rate limiting, and recovery.
- `internal/utils/`: Narrow reusable helpers such as pagination defaults and default values.
- `internal/common/`: Low-coupling constants and reusable domain errors.
- `scripts/`: Build, release, operations, and developer support scripts.

## Dependency Direction

Preferred direction:

```text
cmd/server -> api -> api/v1 -> service -> dao -> database
                                      -> model
                                      -> utils/common
api/v1 -> dto, middleware, response, common
middleware -> dto, response, common, focused lookup helpers
service -> dto, model, dao, common, utils, focused pkg/internal helpers
dao -> database, dto query structs, model, common
config/database/pkg are infrastructure dependencies
```

Keep the direction mostly one-way. Handler depends on service, not DAO. DAO should not import handler code. Model should not depend on Gin. DTO should not control database behavior. Middleware may call focused lookup helpers when it is an edge concern, but it should not pull service business flows into every request.

## Coding Style

### Handler / Controller

Keep handlers thin while leaving protocol-specific work at the edge.

- receive `*gin.Context`
- parse IDs with `strconv.ParseInt`
- bind query/body DTOs with `ShouldBindQuery` or `ShouldBindJSON`
- use common protocol errors such as `common.ErrInvalidParam` and `common.ErrInvalidRequestBody`
- call one service function with `c.Request.Context()`
- map session or permission failures to `401`; map most validation/service failures to `400` unless nearby code is more specific
- return success through `response.Success`

Preferred shape:

```go
func UpdateX(c *gin.Context) {
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		response.Error(c, http.StatusBadRequest, common.ErrInvalidParam.Error())
		return
	}
	var req dto.XRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		response.Error(c, http.StatusBadRequest, common.ErrInvalidRequestBody.Error())
		return
	}
	data, err := service.UpdateX(c.Request.Context(), id, req)
	if err != nil {
		response.Error(c, http.StatusBadRequest, err.Error())
		return
	}
	response.Success(c, data)
}
```

Do not add business branching in handlers unless it is purely protocol-specific.

### Service

Service owns use-case execution.

- accept `context.Context` plus DTO/plain values
- trim and normalize requests with private `normalizeXRequest` helpers
- validate business rules with private `validateXRequest` helpers
- normalize pagination through `utils.NormalizePage`
- call DAO functions with context and plain values
- build model structs from DTO payloads
- convert model structs into response DTOs with `toXResponse` helpers
- coordinate external clients, credential/session helpers, audit logs, and other multi-step flows
- return plain `error` values unless the repository already uses app-error wrappers

Prefer explicit flow over abstraction-heavy service frameworks. Service should express business steps clearly and should not hide key rules inside middleware or DAO.

### DAO

DAO exposes direct, readable persistence helpers.

- `CreateX`
- `UpdateX`
- `DeleteX`
- `GetXByID`
- `GetXByName`
- `ListX`
- `CountX`
- `ReplaceXChildren`

DAO rules:

- accept `context.Context` as the first parameter
- use `database.DB.WithContext(ctx)` or the repository's DB holder
- keep optional filters readable with a `tx` variable
- count before applying order/offset/limit for paged lists
- return model pointers, slices, totals, or scalar counts
- convert `gorm.ErrRecordNotFound` to `common.ErrNotFound`
- use `db.Transaction` for replace-all or multi-step persistence operations
- never accept `*gin.Context`

DAO should answer storage questions, not transport questions. If a result is only for API presentation, prefer shaping it in DTO or service unless the repository already uses DAO projections consistently.

### Model

Use models for persistence-facing entities and storage mapping.

- keep structs plain and explicit
- include GORM tags and JSON tags when the local code exposes model values directly
- keep table helpers such as `TableName()` and `TableComment()` near the model
- allow small model methods for storage-owned behavior, such as decoding JSON column lists or building a DSN
- avoid adding transport-only concerns into model types

### DTO

Use DTOs for transport boundaries and interface contracts.

- request payloads carry `json` tags
- query structs carry `form` tags
- response DTOs carry frontend-facing `json` names
- `PageResponse[T]` and `NewPageResponse` own paginated response shape
- validation should remain explicit in service helpers unless the local code already relies on binding tags

DTO is the right place for request payloads, pagination parameters, filtering fields, response projections, preview responses, and connection-check responses.

### Middleware

Keep middleware focused on HTTP edge concerns.

- parse bearer tokens in small helpers
- call session/token helpers and current-user lookups as needed
- write failures with `response.Error`, then `c.Abort()` and `return`
- store typed profiles in Gin context and expose typed accessors
- keep CORS/header behavior in middleware, not in service

Do not leak `*gin.Context` from middleware into DAO or service logic.

### Config, Logger, and Database Infrastructure

Keep infrastructure initialization explicit and close to the process entry.

- `cmd/server` should call `config.Load`, `logger.Init`, `database.Init`, migrations, route assembly, and HTTP startup in a readable order
- `config` owns Viper setup, config file discovery, required field validation, normalization, and derived values such as origins or DSNs
- `pkg/logger` owns Zap construction, global or package-level accessors, `Sync`, and any small logging helpers
- `internal/middleware` owns HTTP request logging middleware, usually backed by `pkg/logger`
- `database` owns GORM setup and any adapter that implements `gorm.io/gorm/logger.Interface`
- GORM SQL logs should be configured in `database.Init`, not emitted manually from DAO functions
- DAO functions should keep using `database.DB.WithContext(ctx)` and should not import Zap directly
- prefer explicit configuration keys such as `LOG_LEVEL` and `GORM_LOG_LEVEL`, validated in `config`, over scattered hard-coded logging behavior

Preferred startup shape:

```go
cfg, err := config.Load()
if err != nil {
	log.Fatal(err)
}
if err := logger.Init(cfg.LogLevel); err != nil {
	log.Fatal(err)
}
defer logger.Sync()
if err := database.Init(cfg.DatabaseDSN, cfg.GormLogLevel); err != nil {
	logger.L().Fatal("database init failed", zap.Error(err))
}
```

## Naming Guidance

Prefer direct, repeated naming:

- files: `<module>_service.go`, `<module>_dao.go`, `<module>.go`
- handlers: `GetXList`, `CreateX`, `UpdateX`, `DeleteX`, `PreviewX`, `RefreshX`, `ValidateX`
- service names: `ListX`, `CreateX`, `UpdateX`, `GetX`, `ResolveX`
- DAO names: `CreateX`, `UpdateX`, `GetXByID`, `GetXByName`, `ListX`, `CountX`, `ReplaceXChildren`
- request DTOs: `XRequest`
- response DTOs: `XResponse`

Prefer direct names over clever generic names.

## Error Handling Style

Keep error handling small and consistent:

- define reusable domain errors in `internal/common/error.go`
- use plain `error` returns from service and DAO
- map `gorm.ErrRecordNotFound` to `common.ErrNotFound` at the DAO/storage boundary
- use `fmt.Errorf` for clear local infrastructure errors
- wrap with `%w` only when callers need the cause
- serialize errors in handlers through `response.Error`
- do not introduce app-specific error wrappers unless the repository already uses them

Handlers should serialize errors. Services should interpret business rules. DAO should normalize storage misses.

## Response Style

Use the shared response helper:

- success: `response.Success(c, data)`
- error: `response.Error(c, statusCode, message)`
- envelope: `code`, `message`, optional `data`
- command endpoints may return small `gin.H` maps when nearby code does

Do not hand-build JSON envelopes in each handler.

## File Placement Rules

When adding a new resource or module:

1. add or extend DTOs in `internal/dto/` if the transport shape changes
2. add or extend service functions in `internal/service/<module>_service.go`
3. add or extend DAO functions in `internal/dao/<module>_dao.go`
4. add or extend models in `internal/model/` only if persistence changes
5. add handler functions under `api/v1/` or the existing API folder
6. register routes in `api/router.go` or the existing router file
7. add middleware, utils, or focused pkg/internal helpers only when the concern clearly belongs there

If the feature is command-only, prefer `cmd/<entry>/` plus focused package helpers instead of exposing an HTTP route.

## Standard API Addition Flow

For Structure A, add a new interface in this order:

1. inspect adjacent routes, handlers, service, DAO, DTO, and model files
2. decide the module name and reuse an existing module file when the feature belongs to that module
3. define request/query/response DTOs in `internal/dto/` if needed
4. add or extend `internal/service/<module>_service.go`
5. add or extend `internal/dao/<module>_dao.go`
6. add or extend `internal/model/` only if persistence shape changes
7. add or extend `api/v1/<module>.go`
8. register the route in `api/router.go`
9. add tests only where the repository already tests similar helpers or where risk is high

Preferred ownership by concern:

- parse path/query/body -> handler
- validate business input -> service
- normalize pagination/defaults -> service or narrow `internal/utils`
- fetch or persist data -> DAO
- map storage miss to common not-found -> DAO/storage boundary
- map model to API shape -> service/DTO
- write success or error response -> handler
- external DB/client behavior -> a focused internal package

## Common Anti-Patterns

Avoid these:

- handlers calling `database.DB` directly
- service logic copied into multiple handlers
- DAO functions returning HTTP-specific response objects
- DAO functions accepting `*gin.Context`
- introducing an app-error wrapper in a plain-error codebase
- introducing repository interfaces or dependency injection for simple CRUD when the codebase uses direct package helpers
- generic service abstractions that make simple CRUD harder to read
- one giant service file owning unrelated modules
- broad `utils` packages for feature-specific business rules
