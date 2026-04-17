# CLAUDE.md

## Project Overview

Employee Budget Allocation Platform — a Workday-like enterprise application for managing organizational hierarchy, employee compensation, and budget allocation.

**Architecture:** Two repos — a TypeScript monorepo (Nx 20) and a separate .NET repo:

**TypeScript Monorepo (Nx 20) — Module Federation micro frontends:**
- **Shell** (`apps/shell`) — Host React app (layout, navigation, auth, routing)
- **MFE Hierarchy** (`apps/mfe-hierarchy`) — Org tree visualization micro frontend
- **MFE Compensation** (`apps/mfe-compensation`) — Compensation management micro frontend
- **MFE Budget** (`apps/mfe-budget`) — Budget allocation & tracking micro frontend
- **MFE Admin** (`apps/mfe-admin`) — HR admin micro frontend (employee CRUD, CSV import)
- **NestJS BFF** (`apps/bff`) — Backend-for-frontend, the ONLY public entry point
- **Shared libs** (`libs/`) — shared-ui, shared-types, shared-auth, shared-state, shared-utils, contracts

**Separate Repo:**
- **.NET 9 API** — Internal API (Clean Architecture), never exposed publicly

**Infrastructure:** PostgreSQL 17 (with `ltree` extension for hierarchy), ElastiCache Redis 7.4 (Valkey compatible) for caching.

**Auth:** Auth0 SSO with 3-layer RBAC enforcement (NestJS guard → .NET policy → PostgreSQL RLS).

## Architecture Rules

- **NestJS BFF is the ONLY entry point.** The .NET API is never exposed publicly. All external traffic routes through BFF.
- **3-layer RBAC enforcement:** NestJS validates auth/RBAC first, .NET re-validates, PostgreSQL RLS acts as the last line of defense.
- **Internal header signing:** NestJS signs internal headers (`X-User-Id`, `X-User-Role`, `X-Subtree-Ids`) with HMAC. The .NET API must verify these signatures before processing any request.
- **CQRS:** Writes go to normalized tables. Reads go to materialized views.
- **Append-only compensation:** Compensation records are append-only. NEVER update or delete compensation rows.
- **Domain events:** All org structure changes must publish domain events via the transactional outbox pattern.

## Code Conventions

### TypeScript (React + NestJS)

- Strict TypeScript (`strict: true`)
- Use `interface` for object shapes, `type` for unions/intersections
- NestJS: use decorators for validation (`class-validator`), guards for auth
- React 19: functional components only, TanStack Query v5 for data fetching
- Import order: node modules → external packages → internal modules → relative imports
- File naming: kebab-case for files, PascalCase for components/classes

### C# (.NET)

- .NET 9, C# 13
- Use MediatR for CQRS command/query dispatch
- FluentValidation for request validation
- Entity Framework Core for data access
- Follow Clean Architecture: Domain → Application → Infrastructure → API layers
- Async/await everywhere, never `.Result` or `.Wait()`
- File naming: PascalCase matching class name

### SQL

- Use EF Core migrations for schema changes
- All queries against hierarchy use the materialized view, never recursive CTEs at runtime
- Always set `app.current_user_id` via `SET` before queries to enable RLS

## Testing Requirements

- **Unit tests:** xUnit (.NET), Jest (TypeScript)
- **Integration tests:** test containers for PostgreSQL and Redis
- **Contract tests:** Pact (NestJS as consumer, .NET as provider)
- **E2E:** Playwright
- **Minimum coverage:** 80% for business logic, 60% overall

## Key Patterns

- **Cache-aside:** Check Redis first, fall through to API on miss, populate cache after.
- **Circuit breaker:** NestJS uses `opossum` for .NET calls. If circuit is open, serve stale cache.
- **Event-driven:** Domain events → transactional outbox → SNS → SQS → consumers.
- **Feature flags:** Split.io SDK in both NestJS and .NET. Gate new features behind flags.

## Micro Frontend Rules

- **No direct imports between MFEs.** MFEs communicate only via the shared event bus (`libs/shared-state`) or shared Auth0 context (`libs/shared-auth`).
- **Shared singletons:** `react`, `react-dom`, `react-router-dom`, `@tanstack/react-query`, `@auth0/auth0-react` are shared via Module Federation — never bundle these per MFE.
- **Independent deployability:** Each MFE builds and deploys independently to its own S3 path. Shell loads MFEs dynamically via `remoteEntry.js`.
- **Fault isolation:** If an MFE fails to load, the shell must render a fallback error boundary — never crash the entire app.

## File Structure Quick Reference

```
apps/shell/src/
  components/        # Layout, navigation, error boundaries
  config/            # Module Federation remote config
  routes/            # Top-level routing, lazy MFE loading

apps/mfe-hierarchy/src/
  components/        # Org tree visualization
  hooks/             # Hierarchy-specific hooks
  services/          # BFF API client for hierarchy

apps/mfe-compensation/src/
  components/        # Compensation views, history tables
  hooks/             # Compensation-specific hooks
  services/          # BFF API client for compensation

apps/mfe-budget/src/
  components/        # Budget allocation, utilization dashboards
  hooks/             # Budget-specific hooks
  services/          # BFF API client for budget

apps/mfe-admin/src/
  components/        # Employee CRUD, CSV import
  hooks/             # Admin-specific hooks
  services/          # BFF API client for admin

libs/shared-ui/     # Shared UI components (design system)
libs/shared-types/  # Shared TypeScript types/interfaces
libs/shared-auth/   # Auth0 context provider, hooks, guards
libs/shared-state/  # Event bus, shared TanStack Query client
libs/shared-utils/  # Common utilities (formatting, validation)
libs/contracts/     # Pact contract tests

apps/bff/src/
  modules/           # NestJS modules (hierarchy/, employee/, budget/, auth/)
  guards/            # AuthGuard, RbacGuard
  interceptors/      # Logging, caching, error handling
  pipes/             # Validation pipes
  config/            # Configuration module
```

**Separate .NET Repo** (`employee_budget_allocation_api`):

```
src/
  Domain/            # Entities, value objects, domain events
  Application/       # CQRS commands, queries, handlers, validators
  Infrastructure/    # EF Core, repositories, SNS publisher, Redis
  Api/               # Controllers, middleware, health checks
tests/
  Api.UnitTests/
  Api.IntegrationTests/
  Api.ArchitectureTests/
```

## Environments

The platform uses a **3-environment promotion strategy** with separate AWS accounts:

| Environment | Purpose | Branch Trigger | AWS Account | URL |
|-------------|---------|---------------|-------------|-----|
| **test** | Dev testing, integration, QA | PR merges to `develop` | Shared dev account | `test.budgetalloc.example.com` |
| **beta** | Pre-production, UAT, canary validation | PR merges to `release/*` or promotion from test | Separate beta account | `beta.budgetalloc.example.com` |
| **prod** | Production | PR merges to `main` or promotion from beta | Dedicated prod account (isolated) | `app.budgetalloc.example.com` |

**Git branching:**
- `develop` → deploys to **test** automatically
- `release/*` → deploys to **beta** automatically
- `main` → deploys to **prod** (requires manual approval gate)
- Feature branches → PR to `develop` (CI only, no deploy)

**Key differences per environment:**

| Config | test | beta | prod |
|--------|------|------|------|
| EKS nodes | 2 (single AZ OK) | 3 (multi-AZ) | 6+ (multi-AZ, 3 AZs) |
| RDS | Single, db.t4g.medium | Multi-AZ, db.r6g.large | Multi-AZ, db.r6g.xlarge + read replica |
| Redis | 1 node, cache.t4g.small | 2-node cluster | 3-node cluster, Multi-AZ |
| Argo Rollouts | Disabled (direct deploy) | Canary 50%→100% | Canary 20%→40%→80%→100% with analysis |
| Split.io | Development env | Staging env | Production env |
| Auth0 | Dev tenant | Staging tenant | Production tenant |
| Seed data | 5000+ employees (seeded) | 1000 employees (subset) | Real data only |
| Log level | DEBUG | INFO | WARN |
| Feature flags | All ON (for testing) | Selective (UAT) | Controlled rollout |

**Docker image tagging:**
- test: `{ecr}/eba-bff:test-{sha}`, `{ecr}/eba-bff:test-latest`
- beta: `{ecr}/eba-bff:beta-{sha}`, `{ecr}/eba-bff:beta-latest`
- prod: `{ecr}/eba-bff:prod-{sha}`, `{ecr}/eba-bff:prod-latest`, `{ecr}/eba-bff:v{semver}`

## Common Commands

- `make dev` — Start all services locally
- `make test` — Run all tests
- `make test-unit` — Unit tests only
- `make test-integration` — Integration tests with test containers
- `make seed` — Generate 5000+ employee seed data
- `make migrate` — Run database migrations
- `make lint` — Lint all projects
- `make build` — Build all projects
- `make docker-build` — Build Docker images

## Important Warnings

- **NEVER** expose .NET API endpoints publicly
- **NEVER** skip RBAC validation in NestJS guards
- **NEVER** delete compensation records (append-only)
- **NEVER** use recursive CTEs for read queries in production (use materialized view)
- **NEVER** commit secrets or `.env` files
- **ALWAYS** publish domain events when modifying org structure or compensation
- **ALWAYS** invalidate Redis cache after org/comp changes
- **ALWAYS** include `correlationId` in all log entries
