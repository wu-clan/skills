# Style Baseline

Use this reference as the style source for the Go skill. It captures a pragmatic three-layer Gin + GORM service style with thin API handlers, service-owned business flow, direct DAO helpers, simple response envelopes, and plain errors.

## What Is Stable and Reusable

The most reusable parts of this baseline are:

- thin Gin handlers that parse protocol inputs and delegate quickly
- services that accept `context.Context` plus DTO/plain values
- service-owned normalization, validation, orchestration, DTO mapping, and audit-like side effects
- direct DAO helpers using `database.DB.WithContext(ctx)`
- DAO functions that may accept DTO query structs
- `internal/common` errors, constants, and pagination helpers instead of scattered literals
- explicit DTO request/query/response structs and generic `PageResponse[T]`
- simple `pkg/response` envelope helpers instead of per-handler response JSON
- package-level infrastructure holders for database, config, and logger when that is the local pattern
- Viper-owned config loading in `config`, following the project policy for file-only or environment-aware config, with normalization and validation in one place
- Zap-owned process and request logging through `pkg/logger` and HTTP middleware
- GORM setup and SQL logging adapters in `database`, keeping DAO functions focused on queries
- explicit, pragmatic code over repository frameworks or dependency-injection layers

## Naming Style

### File Naming

Follow these patterns when the repository layout is compatible:

- `api/router.go`
- `api/v1/<module>.go`
- `internal/service/<module>_service.go`
- `internal/dao/<module>_dao.go`
- `internal/model/<module>.go`
- `internal/dto/<module>.go`
- `internal/dto/request.go`
- `internal/dto/response.go`
- `pkg/response/response.go`

The style favors predictable module-parallel naming for business features, with focused support packages for middleware, common, logger, config, database, deploy, migrations, scripts, and shared response helpers.

### Function Naming

Use direct names that describe the action clearly.

Examples from the baseline:

- `GetXList`
- `CreateX`
- `UpdateX`
- `DeleteX`
- `PreviewX`
- `RefreshX`
- `ValidateX`
- `ListX`
- `GetXByID`
- `GetXByName`
- `ReplaceXChildren`
- `toXResponse`

Prefer straightforward verbs such as `Get`, `List`, `Create`, `Update`, `Delete`, `Preview`, `Refresh`, `Validate`, `Normalize`, `Resolve`, `Count`, `Replace`, and `toXResponse`.

## Layer Style

### Handler Style

Handlers are the presentation layer. They are thin, but they do own HTTP/Gin protocol concerns.

Observed pattern:

- accept `*gin.Context`
- parse path parameters with `strconv.ParseInt`
- bind query/body payloads with `ShouldBindQuery` or `ShouldBindJSON`
- use `common.ErrInvalidParam` or `common.ErrInvalidRequestBody` for common protocol failures
- extract current-user or session values with the middleware accessor and pass DTO or plain values into service; do not pass `*gin.Context`
- pass `c.Request.Context()` to service functions
- choose the HTTP status at the edge, commonly `400` for validation/service errors and `401` for session or permission failures
- return success through `response.Success`

Preferred shape:

```go
func GetXList(c *gin.Context) {
	var req dto.XRequest
	if err := c.ShouldBindQuery(&req); err != nil {
		response.Error(c, http.StatusBadRequest, common.ErrInvalidRequestBody.Error())
		return
	}
	data, err := service.ListX(c.Request.Context(), req)
	if err != nil {
		response.Error(c, http.StatusBadRequest, err.Error())
		return
	}
	response.Success(c, data)
}
```

For route IDs:

```go
id, err := strconv.ParseInt(c.Param("id"), 10, 64)
if err != nil {
	response.Error(c, http.StatusBadRequest, common.ErrInvalidParam.Error())
	return
}
```

Do not move GORM, SQL construction, or business rules into handlers.

### Service Style

Service is the business layer and the main use-case layer.

Observed responsibilities:

- trim and normalize request values through small `normalizeXRequest` helpers
- validate business constraints through `validateXRequest` helpers that return `common.Err...` values
- normalize pagination through `common.NormalizePage`
- fetch and persist through DAO functions
- call focused domain/infrastructure helpers such as external clients, credential utilities, and session helpers
- build model values from DTOs
- map model values into response DTOs through `toXResponse` helpers
- coordinate multi-step flows and best-effort side effects, such as audit logging
- return plain `error` values, not framework-specific app errors, unless the local repository already uses such a wrapper

Preferred shape:

```go
func CreateX(ctx context.Context, req dto.XRequest) (*dto.XResponse, error) {
	normalizeXRequest(&req)
	if err := validateXRequest(req); err != nil {
		return nil, err
	}
	item := &model.X{Name: req.Name}
	if err := dao.CreateX(ctx, item); err != nil {
		return nil, err
	}
	resp := toXResponse(item)
	return &resp, nil
}
```

When a use case needs the current user, accept a plain identifier or DTO field from the handler, not a Gin or GORM type.

Keep service flow explicit. Favor small private helpers in the same file over generic service frameworks. Service must not import Gin or GORM and must not accept `*gorm.DB`.

### DAO Style

DAO is the data layer. Functions are small, direct, and context-aware.

Observed responsibilities:

- create, update, delete, and get records
- list with count, filtering, ordering, offset, and limit
- run small transactions for replace-style or multi-step persistence operations
- return model pointers, model slices, totals, or simple scalar counts
- map `gorm.ErrRecordNotFound` to `common.ErrNotFound`
- accept DTO query structs when the list or filter shape already lives in `internal/dto`

Preferred shape:

```go
func GetXByID(ctx context.Context, id int64) (*model.X, error) {
	var item model.X
	if err := database.DB.WithContext(ctx).First(&item, id).Error; err != nil {
		if errors.Is(err, gorm.ErrRecordNotFound) {
			return nil, common.ErrNotFound
		}
		return nil, err
	}
	return &item, nil
}
```

For paged lists, build a query variable, apply optional filters, count first, then apply order/offset/limit and load results.

DAO should not accept `*gin.Context`, should not serialize responses, and should not know HTTP status codes. Transactions belong here, using `database.DB.WithContext(ctx).Transaction`, not in the service layer.

### Model Style

Models are persistence-facing GORM structs.

Patterns to reuse:

- explicit fields with `gorm` and `json` tags
- `ID int64`, `CreatedAt time.Time`, and `UpdatedAt time.Time` when persisted
- `TableName()` when the table name differs from GORM defaults
- `TableComment()` if migrations or schema tools use model comments
- small entity helpers only when behavior belongs to the entity, such as decoding stored JSON column lists or building a DSN

Do not add transport-only fields to models when a DTO response shape is clearer.

### DTO and Pagination Style

DTOs define transport contracts and, when useful, DAO list/filter arguments.

Patterns to reuse:

- payload structs use `json` tags
- list query structs use `form` tags
- response structs use frontend-facing `json` names
- pagination is represented by `PageResponse[T]`
- `NewPageResponse(list, total, page, pageSize)` constructs paged responses
- request validation is mostly explicit in service helpers, not hidden in tags
- pagination defaults live in `internal/common`, for example `common.NormalizePage`

Use DTOs for request payloads, query filters, response projections, connection-check results, preview results, and generic record maps.

## Error Handling Style

The baseline uses plain errors plus a unified response helper.

- reusable domain errors live in `internal/common/error.go`
- low-level lookup miss errors are converted to `common.ErrNotFound` in DAO or near the storage boundary
- services return `error` directly
- handlers decide HTTP status and call `response.Error(c, status, err.Error())`
- session or permission handlers and middleware return `401`
- most business validation and service errors return `400` in the current API style
- infrastructure functions may use `fmt.Errorf` with clear messages, and wrap with `%w` when preserving causes matters

Do not introduce app-specific error wrappers unless the existing repository already has that pattern.

## Response Style

The baseline favors one small response package.

Patterns to reuse:

- `response.Success(c, data)` writes HTTP 200 with `Envelope{Code: 0, Message: "ok", Data: data}`
- `response.Error(c, code, message)` writes the same HTTP status as `Envelope.Code`
- handlers do not hand-build envelopes
- small success maps like `gin.H{"ok": true}` or `gin.H{"deleted": true}` are acceptable for command endpoints when nearby code uses them

## Middleware and Session Style

Middleware stays at the HTTP edge but may call focused session or lookup helpers when the repository already does this.

Patterns to reuse:

- parse bearer credentials from request headers with a small helper
- return early with `response.Error`, `c.Abort()`, and `return`
- store the current user profile in Gin context under a package-local key
- expose a typed accessor such as `CurrentAdminUserProfile(c)`
- keep CORS and other cross-cutting protocol logic in `internal/middleware`

The handler reads that accessor and passes plain values or DTOs into service. Do not push middleware-specific Gin context values deep into service or DAO layers.

## Config, Database, Logger, and Startup Style

Startup code is explicit and linear.

Patterns to reuse:

- `cmd/server/main.go` loads config, initializes logger, initializes database, runs migrations, registers routes, starts HTTP server, and handles graceful shutdown
- `config` owns Viper loading, required file handling, normalization, validation, and resolved DSN helpers
- `database` owns `Init`, `Open`, `Close`, DB globals, driver switching, and GORM log level parsing
- `pkg/logger` wraps zap/lumberjack and exposes direct package functions
- migrations use embedded SQL files and focused helper functions

Infrastructure placement rules:


Config loading policy:

- follow the repository's existing config policy before adding defaults or environment overrides
- if a project requires explicit config files, do not call `AutomaticEnv` and do not add `SetDefault`; return clear errors for missing files or required keys
- if a project already supports environment overrides, keep that behavior centralized in `config` and document the precedence

- use Viper in `config` only; business layers should receive normalized config values or use already-initialized infrastructure
- initialize Zap in the process entry through `pkg/logger`, then use narrow helpers such as `logger.L()` and `logger.Sync()`
- put HTTP request logging in middleware, not in handlers
- put GORM logger adapters in `database`, implementing `gorm.io/gorm/logger.Interface`
- pass log level and GORM log level from config into logger/database initialization
- keep DAO functions free of logging side effects unless the repository already has an explicit audit pattern

Follow the current package-level infrastructure style instead of introducing a container or constructor graph unless the repository already has one.

## External Resource Style

External resource code belongs in focused packages, not in handlers.

Patterns to reuse:

- keep external DB/client connection and SQL construction in a focused internal package
- use `context.WithTimeout` with shared constants for external calls
- quote and validate identifiers before constructing SQL
- keep schema introspection helpers beside external execution helpers
- return plain maps or DTOs only at the service boundary

## Comment Style

Comments are sparse but useful.

- use comments for route groups, embedded file directives, table meaning, or non-obvious domain constraints
- avoid comments that narrate obvious assignments
- keep comments short and close to the relevant code

## Test Style

Tests use the standard library style unless the repository already has test helpers.

- use `testing`
- use `t.Fatalf` / `t.Fatal`
- use `t.TempDir()` for temporary files
- keep test names behavior-oriented, such as `TestMigrateSkipsSQLite`
- avoid adding assertion frameworks just for a few checks

## Dependency Style

The baseline keeps dependency flow easy to follow.

- `cmd` depends on config, database, migrations, logger, and API registration
- `api/v1` depends on DTO, middleware when needed, service, common errors, and response helpers
- `service` depends on DTO, DAO, model, common, and focused infrastructure packages
- `dao` depends on `database`, `model`, query DTOs, and common errors
- `model` stays independent of Gin and handlers
- `dto` stays independent of database and GORM
- `pkg/response` depends on Gin but not on business packages

Prefer this visible dependency flow over hidden indirection. Service must not depend on Gin or GORM.

## Practical Rules to Reuse in Other Repositories

When porting this style into another Go repository:

1. inspect adjacent files first
2. keep handlers thin but responsible for protocol parsing
3. pass `context.Context` from handler to service to DAO
4. put business normalization, validation, orchestration, and DTO mapping in service
5. keep DAO explicit, query-focused, and `WithContext` aware
6. centralize common errors, constants, pagination, and response writing
7. return plain errors unless the repository already has app-error wrappers
8. use direct names and local helpers before adding abstractions
9. follow existing neighboring patterns before forcing a wider refactor

## What Not To Copy Blindly

Do not copy these blindly into unrelated repositories:

- product-specific package names
- domain-specific request shapes
- status semantics tied to one product flow
- database table fields that belong only to one project
- package-level globals if the target repository already uses explicit dependency injection

Copy the coding discipline, naming flavor, response shape, and layer boundaries. Do not copy the business domain.
