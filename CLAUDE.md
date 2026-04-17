# CLAUDE.md

## Project Overview

Employee Budget Allocation Platform — a Workday-like enterprise application for managing organizational hierarchy, employee compensation, and budget allocation.

**Architecture:** Monorepo with three services:

- **React SPA** (`apps/web`) — Single-page application frontend
- **NestJS BFF** (`apps/bff`) — Backend-for-frontend, the ONLY public entry point
- **.NET 8 API** (`apps/api`) — Internal API, never exposed publicly

**Infrastructure:** PostgreSQL 16 (with `ltree` extension for hierarchy), ElastiCache Redis for caching.

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
- React: functional components only, React Query for data fetching
- Import order: node modules → external packages → internal modules → relative imports
- File naming: kebab-case for files, PascalCase for components/classes

### C# (.NET)

- .NET 8, C# 12
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

## File Structure Quick Reference

```
apps/web/src/
  components/        # Shared UI components
  features/          # Feature modules (hierarchy/, compensation/, budget/)
  hooks/             # Custom React hooks
  services/          # API client layer
  store/             # State management
  types/             # TypeScript types

apps/bff/src/
  modules/           # NestJS modules (hierarchy/, employee/, budget/, auth/)
  guards/            # AuthGuard, RbacGuard
  interceptors/      # Logging, caching, error handling
  pipes/             # Validation pipes
  config/            # Configuration module

apps/api/
  src/
    Domain/          # Entities, value objects, domain events
    Application/     # CQRS commands, queries, handlers, validators
    Infrastructure/  # EF Core, repositories, SNS publisher, Redis
    Api/             # Controllers, middleware, health checks
```

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
