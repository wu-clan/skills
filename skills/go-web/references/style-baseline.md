# Style Baseline

Use this reference as the style source for the Go skill. Treat it as a collection of coding habits that can be transplanted into service-oriented Go repositories.

## What Is Stable and Reusable

The most reusable parts of this baseline are:

- thin handler functions
- service-owned binding, validation, orchestration, and error mapping
- direct DAO query helpers with readable names
- explicit DTO request structs
- centralized pagination and bind helpers
- unified response wrapper and application error type
- parallel file naming by module across layers
- explicit, pragmatic code over abstraction-heavy patterns

## Naming Style

### File Naming

Follow these patterns when the repository layout is compatible:

- `api/v1/talk.go`
- `internal/service/talk_service.go`
- `internal/dao/talk_dao.go`
- `internal/model/talk.go`
- `internal/dto/request.go`
- `internal/dto/response.go`

The style favors predictable module-parallel naming over deep folder nesting.

### Function Naming

Use direct names that describe the action clearly.

Examples from the baseline:

- `GetTalkList`
- `GetTalkDetail`
- `GetBlogList`
- `StartSync`
- `GetBlogByBlogID`
- `ListBlogs`
- `BatchUpsertBlogs`
- `normalizePage`
- `bindQuery`
- `bindJSON`

Prefer straightforward verbs such as `Get`, `List`, `Start`, `BatchUpsert`, `Validate`, `Normalize`, `Bind`.

## Layer Style

### Handler Style

Handlers are intentionally thin.

Observed pattern:

- accept `*gin.Context`
- call one service function
- if error, return unified error response
- if success, return unified success response

This keeps protocol code shallow and easy to scan.

### Service Style

Service is the main use-case layer.

Observed responsibilities:

- bind request DTOs
- perform business-level validation beyond binding tags
- normalize pagination
- call DAO functions
- aggregate data across entities when needed
- translate infra errors into app errors with status codes

The style favors explicit request-to-response flow instead of hidden middleware magic.

### DAO Style

DAO functions are small and direct.

Observed responsibilities:

- single-record lookup
- list with count and pagination
- batch upsert
- lightweight summary projections

The code style favors readable query chains and purpose-specific DAO functions rather than a generic repository framework.

## Error Handling Style

The baseline uses a small, practical error strategy.

- reusable error values live in `internal/common/error.go`
- application-layer wrapping lives in `pkg/response/response.go`
- service functions return `*response.AppError`
- handlers only serialize the app error
- `gorm.ErrRecordNotFound` is mapped in service
- invalid parameters become `400`
- missing resources become `404`
- unexpected failures become `500`

This style keeps transport mapping consistent while leaving lower layers simple.

## DTO and Validation Style

The baseline keeps request DTOs explicit and close to transport.

Examples:

- `QueryByQQRequest`
- `QueryByTalkIDRequest`
- `QueryByTargetRequest`
- `SyncRequest`

Patterns to reuse:

- binding tags on DTOs
- service helpers for `bindQuery` and `bindJSON`
- explicit follow-up validation in service when binding tags are not enough
- centralized pagination normalization helper

## Standard API Addition Flow

When adding a new interface, follow this order unless the repository already uses a different nearby pattern:

1. define the request and response shape first
2. decide which module owns the interface
3. add or extend DTOs if transport shape differs from model shape
4. add or extend service logic for validation, orchestration, and error mapping
5. add or extend DAO logic for storage access
6. add or extend model definitions only when persistence shape changes
7. add the handler last, keep it thin, and register the route at the edge

Use this checklist to decide the landing point of each change:

- request binding fields -> `dto`
- transport-specific response wrapper or projection -> `dto`
- pagination normalization, business validation, aggregation -> `service`
- query construction, list/count lookup, upsert -> `dao`
- persistence schema fields -> `model`
- HTTP entry and response serialization -> `api`
- route exposure -> router file nearest to the API edge

If an interface only reads existing data, avoid changing model definitions. If an interface only reshapes existing data, prefer DTO or service changes over model changes.

## Response Style

The baseline favors one response helper package instead of custom response logic per handler.

Patterns to reuse:

- `response.Success(c, data)`
- `response.Error(c, err.Code, err.Error())`
- one response struct
- one application error type

This reduces noise in handlers and keeps HTTP output consistent.

## Comment Style

Comments are sparse but useful.

Observed patterns:

- exported types and functions often have short comments
- comments explain intent, not obvious syntax
- short section comments appear above query blocks when they improve scanning

Do not add comments to narrate trivial assignments. Add comments when they clarify purpose or domain meaning.

## Dependency Style

The baseline keeps dependency flow easy to follow.

- handlers depend on service and response helpers
- services depend on DTO, DAO, model, common errors, and response app errors
- DAO depends on DB holder and models
- external integrations should stay in their own packages instead of leaking across layers

Prefer this kind of visible dependency flow over hidden indirection.

## Practical Rules to Reuse in Other Repositories

When porting this style into another Go repository:

1. keep handlers thin
2. put business orchestration in service
3. keep DAO explicit and query-focused
4. keep request DTOs explicit
5. centralize bind and pagination helpers when repetition appears
6. centralize response writing
7. use direct names, not abstract framework names
8. follow existing neighboring patterns before forcing a wider refactor

## What Not To Copy Blindly

Do not copy these blindly into unrelated repositories:

- product-specific package names
- domain-specific request shapes
- status semantics tied to one product flow
- domain model fields that belong only to one project

Copy the coding discipline, naming flavor, and layer boundaries. Do not copy the business domain.
