# ADR-004: Event-Driven Architecture with Transactional Outbox

## Status

Accepted

## Date

2026-04-17

## Context

When the org structure or compensation data changes, several downstream operations must occur:

1. **Materialized view refresh** -- the read-side ltree view must be updated to reflect the new hierarchy or compensation values.
2. **Cache invalidation** -- Redis caches for affected subtrees must be invalidated so stale data is not served.
3. **Audit logging** -- changes must be recorded in the audit trail for SOX compliance.
4. **Budget recalculation** -- department budget utilization must be recomputed when compensation changes.

These operations must be reliable -- if an employee transfer is committed to the database but the cache is not invalidated, managers will see stale data. If the materialized view is not refreshed, read queries will return incorrect rollup totals.

The naive approach -- performing all downstream operations synchronously within the same request -- is fragile. If SNS is unavailable or Redis times out, the primary write operation fails even though the database commit succeeded.

## Decision

We will use the **transactional outbox pattern** with SNS/SQS fan-out:

### Transactional Outbox

1. When a domain event occurs (e.g., `EmployeeTransferred`, `CompensationUpdated`), the event is written to an `outbox_events` table in the **same database transaction** as the domain change.
2. This guarantees that if the domain change is committed, the event is also committed. If the transaction rolls back, the event is also rolled back. No dual-write problem.

### Background Publisher

3. A background worker polls the `outbox_events` table for unpublished events (or uses PostgreSQL `LISTEN/NOTIFY` for lower latency).
4. The worker publishes each event to an **SNS topic** and marks it as published in the outbox.
5. If publishing fails, the event remains in the outbox and is retried on the next poll cycle.

### SQS Fan-Out

6. The SNS topic fans out to multiple **SQS queues**, each serving a different consumer:
   - **Materialized View Refresh Queue** -- triggers recomputation of the ltree view for the affected subtree.
   - **Cache Invalidation Queue** -- invalidates Redis keys for the affected subtree and rollup.
   - **Audit Log Queue** -- writes the event to the audit trail (if not already handled synchronously).
   - **Budget Recalculation Queue** -- recomputes budget utilization for affected departments.

### Domain Events

The following domain events are published:

| Event | Trigger |
|-------|---------|
| `EmployeeHired` | New employee added to the hierarchy |
| `EmployeeTerminated` | Employee removed from the hierarchy |
| `EmployeeTransferred` | Employee moved to a new manager |
| `CompensationUpdated` | New compensation record appended |
| `BudgetAllocated` | Budget assigned or modified for a department |
| `BudgetThresholdBreached` | Utilization crosses a configured alert threshold |

## Consequences

### Positive

- **Atomicity** -- domain changes and events are committed in the same transaction. No scenario where a change is committed but its event is lost.
- **Reliability** -- if SNS is temporarily unavailable, events queue up in the outbox and are published when connectivity is restored. No data loss.
- **Decoupling** -- downstream consumers (cache, view refresh, audit) are independent. Adding a new consumer requires only subscribing a new SQS queue to the SNS topic.
- **At-least-once delivery** -- SQS guarantees at-least-once delivery with visibility timeout. Combined with idempotent consumers, this provides reliable processing.
- **Observability** -- the outbox table provides a complete log of all events, their publication status, and processing latency.

### Negative

- **Eventual consistency** -- downstream operations (view refresh, cache invalidation) are asynchronous. There is a window (typically < 5 seconds) where the read side is stale after a write.
- **Operational complexity** -- the background publisher is a critical component that must be monitored. If it stops, events accumulate in the outbox and the read side becomes increasingly stale.
- **Idempotency requirement** -- because SQS provides at-least-once delivery, all consumers must be idempotent. Processing the same event twice must produce the same result.
- **Outbox table growth** -- the outbox table grows with every event. A retention policy and archival strategy is needed to prevent unbounded growth.
- **Infrastructure cost** -- SNS topics, SQS queues, and the background publisher add infrastructure components and associated costs.

## Alternatives Considered

### Direct SNS Publish (No Outbox)

Publish events directly to SNS after the database commit, without an outbox table.

- **Rejected because:** This creates a dual-write problem. If the database commit succeeds but the SNS publish fails (network timeout, SNS outage), the event is lost. The domain change exists in the database but downstream consumers are never notified. This leads to stale materialized views and caches with no automatic recovery.

### Change Data Capture with Debezium

Use Debezium to capture PostgreSQL WAL changes and stream them to Kafka/SNS.

- **Rejected because:** Debezium adds significant operational complexity (Kafka Connect, Zookeeper/KRaft, connector configuration). The WAL-based approach captures all row-level changes, which is more granular than needed -- we want domain events (e.g., "employee transferred"), not row mutations (e.g., "manager_id column changed from 42 to 57"). Mapping low-level CDC events to domain events adds a translation layer that the outbox pattern avoids entirely.

### Synchronous Updates (No Events)

Perform all downstream operations (view refresh, cache invalidation) synchronously within the same HTTP request.

- **Rejected because:** This couples the write operation to all downstream consumers. If Redis times out during cache invalidation, the entire write request fails -- even though the database commit was valid. It also increases request latency (materialized view refresh adds 100-500ms to every write). As the number of downstream consumers grows, request latency grows linearly.
