# ADR-002: Three-Layer RBAC Strategy

## Status

Accepted

## Date

2026-04-17

## Context

The platform stores highly sensitive compensation data (salaries, bonuses, equity grants). A single unauthorized data exposure could cause significant legal, financial, and employee-relations damage. Regulatory requirements (SOX) mandate strict access controls and audit trails.

Single-layer authorization (e.g., checking permissions only at the API gateway) is insufficient because:

- A bug in a single auth layer exposes all data.
- Internal services may be accessed in unexpected ways (e.g., a misconfigured network rule, a compromised internal service).
- Compliance auditors expect defense-in-depth controls for sensitive financial data.

We need a strategy where no single point of failure can expose compensation data to unauthorized users.

## Decision

We will enforce RBAC at three independent layers:

### Layer 1: NestJS Guard (Gateway)

- Auth0 JWT validation and RBAC role extraction.
- `AuthGuard` verifies the token is valid and extracts user identity.
- `RbacGuard` checks the user's role against the required role for the endpoint.
- Computes the user's authorized subtree IDs (the set of org nodes they can access).
- Signs internal headers (`X-User-Id`, `X-User-Role`, `X-Subtree-Ids`) with HMAC-SHA256 before forwarding to the .NET API.

### Layer 2: .NET Policy (Service)

- Middleware verifies the HMAC signature on internal headers. Requests with invalid or missing signatures are rejected with 403.
- Authorization policies validate that the requested resource falls within the user's authorized subtree IDs.
- Domain-level checks enforce business rules (e.g., HR Admin can modify compensation, Manager can only view).

### Layer 3: PostgreSQL RLS (Data)

- Row-level security policies on all tables containing sensitive data.
- The application sets `app.current_user_id` via `SET LOCAL` at the start of each transaction.
- RLS policies filter rows so that queries only return data the current user is authorized to see.
- Even if both application layers are bypassed, the database itself refuses to return unauthorized rows.

## Consequences

### Positive

- **Defense-in-depth** -- three independent layers mean an attacker must compromise all three to access unauthorized data.
- **Compliance-friendly** -- SOX auditors can verify each layer independently. The database-level enforcement is particularly compelling because it is declarative and auditable.
- **Fail-secure** -- if any layer fails or is misconfigured, the other layers still protect the data. RLS is the ultimate safety net.
- **Internal header signing** prevents request spoofing between services. Even if an attacker gains network access to the .NET API, they cannot forge valid HMAC-signed headers.

### Negative

- **Increased complexity** -- three authorization implementations must be kept in sync. A role change requires updates in NestJS guards, .NET policies, and PostgreSQL RLS policies.
- **Performance overhead** -- RLS adds a filter predicate to every query, which has a measurable (though typically small) cost. Setting `app.current_user_id` on every transaction adds a round-trip.
- **Testing burden** -- authorization must be tested at all three layers, including edge cases where layers might disagree.
- **Debugging difficulty** -- when a user cannot access expected data, the issue could be in any of three layers, making diagnosis slower.

## Alternatives Considered

### Single-Layer NestJS Authorization Only

Enforce all RBAC in NestJS guards and trust the .NET API and database as internal, authorized components.

- **Rejected because:** A single bug in NestJS guard logic would expose all compensation data. The .NET API has no independent verification -- it trusts whatever NestJS sends. If the internal network is compromised, the .NET API serves any request without authorization checks.

### API Gateway Authorization Only (e.g., AWS API Gateway + Lambda Authorizer)

Use an infrastructure-level API gateway to handle all auth, with backend services operating without authorization logic.

- **Rejected because:** API gateway authorization is coarse-grained -- it can validate tokens and check roles but cannot enforce row-level access (e.g., "this manager can only see their subtree"). The subtree-scoping logic is domain-specific and must live in the application. Also creates a single point of failure.
