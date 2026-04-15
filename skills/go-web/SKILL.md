---
name: go-web
description: Apply Go coding style and architecture conventions for repositories that follow either (1) a single-application layered layout with `api`, `cmd`, and `internal/{service,dao,model,dto}`, or (2) a multi-entry / multi-app layout with `cmd/<app>` and `internal/app/<app>/...`. Use this when Codex needs to inspect a Go codebase, classify which of the two structures it follows, then apply the matching coding style, layer responsibilities, naming rules, error-handling patterns, and file placement guidance.
---

# Go Coding Style Guide

Use this skill to apply a practical Go coding style and layer discipline for service-oriented repositories. The goal is to keep the code explicit, layered, readable, and easy to extend without introducing unnecessary abstractions.

## Workflow

1. inspect the repository root and major directories before proposing changes
2. classify the repository as Structure A or Structure B
3. explain the evidence for the classification using concrete paths
4. load only the matching structure reference
5. read `references/style-baseline.md` as the style baseline
6. place new code into the existing repository boundaries while keeping the same coding discipline

## Structure Selection

Choose Structure A when most of these signals are present:

- root-level `api/` exists
- `cmd/server` or `cmd/<single-entry>` exists
- `internal/service`, `internal/dao`, `internal/model`, `internal/dto` exist at the same level
- routing is centralized in `api/router.go`
- business modules are represented by parallel files such as `talk_service.go`, `talk_dao.go`, `talk.go`

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
- put orchestration in service layer, not in router or DAO
- keep data access in DAO or repository-like functions
- keep transport structs in DTO when they differ from persistence models
- avoid introducing cross-layer imports that invert dependencies
- follow existing naming in the repository before inventing a new style
- prefer explicit, repetitive clarity over clever abstractions when the codebase already favors it

## Do Not Do This

- do not create a third hybrid structure unless the repository already uses it
- do not move many files just to satisfy a textbook architecture
- do not put SQL or GORM query logic in handlers
- do not put HTTP framework types deep inside DAO code
- do not duplicate DTO and model types without a clear transport or persistence boundary
- do not introduce broad `utils` dumping grounds when a focused package would be clearer
- do not copy product-specific package names, domain models, or API semantics into unrelated repositories
