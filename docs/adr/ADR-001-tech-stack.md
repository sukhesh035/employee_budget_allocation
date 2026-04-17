# ADR-001: Tech Stack Selection

## Status

Accepted

## Date

2026-04-17

## Context

We need an enterprise-grade technology stack for a platform that manages organizational hierarchy (5,000+ employees), compensation data, and budget allocation. Key constraints:

- The org hierarchy requires complex tree queries with sub-second response times.
- Compensation data is highly sensitive and requires defense-in-depth security.
- The frontend must be a rich, interactive SPA with tree visualization.
- The backend must support CQRS, domain-driven design, and strong typing for financial data.
- The team has existing expertise in TypeScript and C#.

## Decision

We will use a three-tier architecture:

1. **React SPA** (`apps/web`) -- Single-page application for the frontend, using React Query for server state management.
2. **NestJS BFF** (`apps/bff`) -- Backend-for-frontend in TypeScript. This is the only publicly exposed service. It handles authentication (Auth0), RBAC enforcement, caching (Redis), and circuit-breaking before proxying requests to the internal API.
3. **.NET 8 API** (`apps/api`) -- Internal API in C# following Clean Architecture. Handles domain logic, CQRS via MediatR, and data access via Entity Framework Core. Never exposed publicly.
4. **PostgreSQL 16** -- Primary database with the `ltree` extension for hierarchy queries and row-level security for data-layer RBAC enforcement.
5. **ElastiCache Redis** -- Caching layer for hierarchy and compensation rollup queries.

## Consequences

### Positive

- **BFF pattern** isolates the internal API from the public internet, reducing the attack surface for sensitive compensation data.
- **NestJS** provides a structured, modular framework for the gateway layer with first-class TypeScript support, decorators for validation, and a mature middleware ecosystem.
- **.NET 8** offers high performance for compute-heavy operations (compensation rollup), strong typing for financial calculations (decimal precision), and mature libraries for CQRS (MediatR) and validation (FluentValidation).
- **PostgreSQL ltree** provides native, indexed hierarchical queries without the complexity of closure tables or nested sets.
- **PostgreSQL RLS** enables data-layer security enforcement as a last line of defense, independent of application code.
- **Separation of concerns** -- the BFF handles cross-cutting concerns (auth, caching, rate limiting) while the .NET API focuses purely on domain logic.

### Negative

- **Two backend runtimes** (Node.js + .NET) increase operational complexity -- two build pipelines, two sets of dependencies, two deployment targets.
- **Inter-service latency** -- every request passes through NestJS before reaching .NET, adding network hop overhead.
- **Team skill split** -- developers need proficiency in both TypeScript and C#, which narrows the hiring pool.
- **Local development complexity** -- developers must run three services, PostgreSQL, and Redis locally (mitigated by Docker Compose).

## Alternatives Considered

### Next.js Monolith

A single Next.js application with API routes handling all backend logic and Prisma for database access.

- **Rejected because:** API routes lack the structure needed for complex domain logic (CQRS, domain events). Prisma has limited support for PostgreSQL-specific features like ltree and RLS. A monolith makes it harder to enforce the security boundary between public-facing and internal APIs.

### React + Express + PostgreSQL

React frontend with a plain Express.js backend and PostgreSQL.

- **Rejected because:** Express provides minimal structure, which would require building framework-level concerns (DI, validation, guards) from scratch. TypeScript lacks the decimal precision and type safety of C# for financial calculations. No natural separation between gateway and domain logic.

### React + Spring Boot + PostgreSQL

React frontend with a Spring Boot (Java/Kotlin) backend.

- **Rejected because:** The team has stronger C# expertise than Java/Kotlin. Spring Boot is viable but .NET 8 offers comparable performance with a more concise syntax (C# 12 primary constructors, records). The BFF pattern still applies, so we would need a separate gateway layer regardless.
