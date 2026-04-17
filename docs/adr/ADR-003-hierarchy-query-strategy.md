# ADR-003: Hierarchy Query Strategy (ltree + CQRS)

## Status

Accepted

## Date

2026-04-17

## Context

The platform must store and query an organizational hierarchy of 5,000+ employees. Key query patterns:

- **Subtree queries**: "Show me all employees under VP of Engineering" (could be 500+ nodes).
- **Compensation rollup**: "What is the total compensation for the VP of Engineering's org?" (sum across all descendants).
- **Ancestry path**: "Show the reporting chain from this IC to the CEO."
- **Tree rendering**: Render an interactive org chart with expand/collapse for the entire hierarchy.

The non-functional requirement is that tree queries must return in < 500ms for 5,000 nodes.

Recursive CTEs can answer these queries but are computationally expensive at scale -- each query traverses the tree at runtime. For a read-heavy application (reads outnumber writes by ~100:1), this is wasteful.

## Decision

We will use CQRS (Command Query Responsibility Segregation) with two different strategies for reads and writes:

### Write Side (Commands)

- The normalized `employees` table stores each employee with a `manager_id` foreign key (standard adjacency list).
- When the org structure changes (hire, transfer, termination), a recursive CTE computes the affected subtree and updates the materialized view.
- All org structure changes publish domain events via the transactional outbox pattern.

### Read Side (Queries)

- A **materialized view** stores each employee with a pre-computed `ltree` path column (e.g., `ceo.vp_eng.dir_platform.manager_1.employee_42`).
- The `ltree` column has a GiST index, enabling sub-millisecond subtree queries using the `<@` (descendant of) operator.
- Compensation rollups are pre-aggregated in the materialized view, so fetching the total cost of a subtree is a single indexed lookup rather than a recursive aggregation.
- The materialized view is refreshed asynchronously when domain events are processed (via SQS consumer).

### PostgreSQL ltree Extension

- Native PostgreSQL extension purpose-built for hierarchical data.
- Supports operators like `<@` (is descendant), `@>` (is ancestor), `~` (matches lquery pattern).
- GiST index on ltree columns provides O(log n) subtree queries regardless of tree depth.

## Consequences

### Positive

- **Sub-millisecond read performance** -- subtree queries hit the GiST-indexed materialized view. Benchmarks show < 1ms for 5,000-node subtree queries.
- **Pre-aggregated rollups** eliminate expensive runtime aggregation. Total compensation for any subtree is a single row lookup.
- **Separation of concerns** -- write-side logic (validating transfers, maintaining referential integrity) is decoupled from read-side optimization.
- **ltree is native PostgreSQL** -- no external dependencies, well-tested, and supported by the PostgreSQL community for 20+ years.

### Negative

- **Eventual consistency** -- after an org change, the materialized view is stale until the event is processed and the view is refreshed. The delay is typically < 5 seconds but is not zero.
- **Materialized view refresh cost** -- refreshing the entire view is O(n) on the employee count. For 5,000 employees this is fast (< 1 second), but for 50,000+ it may require incremental refresh strategies.
- **Dual model complexity** -- developers must understand that writes go to normalized tables and reads come from materialized views. Bugs can arise if the two get out of sync.
- **ltree path maintenance** -- when an employee transfers (changes manager), their entire subtree's ltree paths must be recomputed. This is an O(k) operation where k is the subtree size.

## Alternatives Considered

### Recursive CTE Only (No Materialized View)

Use recursive CTEs for all queries against the normalized adjacency list.

- **Rejected because:** Recursive CTEs recompute the tree traversal on every query. At 5,000+ nodes, subtree queries take 50-200ms depending on tree depth and shape. Compensation rollups require joining with compensation tables during recursion, pushing response times above the 500ms threshold under concurrent load.

### Closure Table

Store all ancestor-descendant pairs in a separate table, enabling O(1) subtree queries via simple joins.

- **Rejected because:** The closure table for 5,000 employees contains ~25 million rows (assuming average depth of 10). Insertions and transfers require updating all affected pairs, which is expensive and error-prone. The storage overhead is significant compared to ltree's compact path representation.

### Nested Sets (Modified Preorder Tree Traversal)

Assign left/right values to each node, enabling subtree queries via range comparisons.

- **Rejected because:** Nested sets require renumbering large portions of the tree on every insert or move operation. For an org structure with frequent transfers and new hires, the write amplification is unacceptable. Concurrent modifications are difficult to handle safely.
