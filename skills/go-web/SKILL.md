---
name: go-web
description: Apply pragmatic Go web service conventions for Gin/GORM-style repositories that follow either (1) a single-application layered layout at the Go module root, with api, cmd, config, database, deploy, migrations, pkg, scripts, and internal service/dao/model/dto packages, or (2) a multi-entry or multi-app layout with per-app cmd entries and internal/app app subtrees. Use this when Codex needs to inspect a Go service, classify its structure, then apply matching layer responsibilities, naming, response, error, context, DAO, model, DTO, middleware, and file-placement guidance.
metadata:
  author: wu-clan
  version: 2026-04-30
---

# Go Coding Style Guide

Use this skill to apply a practical Go coding style and layer discipline for service-oriented repositories. The goal is to keep the code explicit, layered, readable, and easy to extend without introducing unnecessary abstractions or framework-heavy indirection.

## Workflow

1. inspect the repository root, Go module root, and major service directories before proposing changes
2. classify the repository as Structure A or Structure B
3. explain the evidence for the classification using concrete paths
4. load only the matching structure reference
5. read `references/style-baseline.md` as the style baseline
6. place new code into the existing repository boundaries while keeping the same Gin, GORM, response, error, and context discipline

## Structure Selection

Choose Structure A when most of these signals are present:

- root-level `api/` exists
- `cmd/server` or `cmd/<single-entry>` exists
- `internal/service`, `internal/dao`, `internal/model`, `internal/dto` exist at the same level
- routing is centralized in `api/router.go`
- support directories such as `config`, `database`, `deploy`, `migrations`, `pkg/response`, `pkg/logger`, `scripts`, `internal/middleware`, `internal/common`, or `internal/utils` sit beside the layers
- business modules are represented by parallel files such as `<module>_service.go`, `<module>_dao.go`, `<module>.go`

Choose Structure B when most of these signals are present:

- `cmd/` contains multiple application entries such as `cmd/admin`, `cmd/client`
- `internal/app/<app>/` exists
- each app owns its own `api`, `service`, `dao`, `model`, and `dto`
- shared code lives outside app folders, usually in `internal/common`, `internal/middleware`, `pkg`, `config`, `database`

## Tie-Break Rules

If the repository is mixed or incomplete, use these rules in order:

1. follow the dominant existing layout, not the idealized one
2. prefer the location already used by adjacent features
3. do not migrate Structure A into Structure B, or the reverse, unless the user explicitly asks for a re-architecture
4. when only one new module is needed, keep the change local and consistent with neighboring files

## Expected Output Shape

When giving implementation guidance, answer in this order:

1. structure classification
2. target files or target layer
3. coding style constraints from the baseline
4. dependency direction
5. implementation notes specific to the chosen structure
6. only then code-level advice

## Reference Map

- Structure A: read `references/structure-a.md`
- Structure B: read `references/structure-b.md`
- Style baseline: read `references/style-baseline.md`

## General Go Rules

Apply these rules regardless of structure:

- keep package boundaries obvious and narrow
- prefer small, purpose-built files over large mixed-responsibility files
- keep handler/controller logic thin
- let handlers own protocol work such as Gin binding, route-param parsing, and HTTP status selection unless nearby code does otherwise
- put orchestration in service layer, not in router or DAO
- have services accept `context.Context` plus DTO/plain values and return plain `error` unless the repository already uses an application error wrapper
- keep data access in DAO or repository-like functions
- pass `ctx` into DAO functions and call `database.DB.WithContext(ctx)` or the local DB holder
- keep transport structs in DTO when they differ from persistence models
- centralize response writing through the existing response helper
- centralize reusable domain errors in the existing common errors package
- avoid introducing cross-layer imports that invert dependencies
- follow existing naming in the repository before inventing a new style
- prefer explicit, repetitive clarity over clever abstractions when the codebase already favors it

## Do Not Do This

- do not create a third hybrid structure unless the repository already uses it
- do not move many files just to satisfy a textbook architecture
- do not put SQL or GORM query logic in handlers
- do not put HTTP framework types deep inside DAO code
- do not introduce app-specific error wrappers, repository interfaces, or dependency-injection frameworks when the existing code returns plain errors and uses package-level infra holders
- do not duplicate DTO and model types without a clear transport or persistence boundary
- do not introduce broad `utils` dumping grounds when a focused package would be clearer
- do not copy product-specific package names, domain models, or API semantics into unrelated repositories
