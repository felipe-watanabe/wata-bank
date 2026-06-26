# ADR-003: Idempotency Strategy

- **Status**: Accepted
- **Date**: 2026-06-25

## Context

Financial operations must never be executed twice for the same request. Network retries, client timeouts, and message redelivery can all cause duplicate requests. We need a consistent idempotency strategy across services that handle money movement and write operations.

## Decision

All write operations use a **check-and-set idempotency pattern in a single transaction**:

1. Clients generate a unique idempotency key (UUID) per request and include it in the request body or header.
2. The server, within the same database transaction:
   - Checks for an existing record with the provided idempotency key.
   - If found: returns the previously stored result (idempotent response).
   - If not found: executes the business logic, persists the result, and inserts the idempotency key.
3. For services using the outbox pattern (ledger-service, payment-orchestrator), the idempotency check and outbox event persistence MUST be part of the same transactional boundary as the business operation.

## Rationale

1. **Exactly-once semantics**: The database's ACID guarantees prevent race conditions — two concurrent requests with the same key cannot both succeed.
2. **Simplicity**: No distributed lock or external coordination service needed. The database is the single source of truth for both business state and idempotency tracking.
3. **Outbox consistency**: By co-locating the idempotency check and outbox write in the same transaction, we guarantee that duplicate requests never produce duplicate events.

## Consequences

- **Positive**: Strong guarantee — no duplicate financial effects regardless of network failures or retry storms.
- **Positive**: Clients can safely retry with the same key after a timeout — the server handles it idempotently.
- **Negative**: Every write table needs either an idempotency key column or a separate idempotency tracking table. This adds schema complexity.
- **Negative**: Clients must be taught to generate and reuse idempotency keys correctly. Incorrect key generation (new key per retry) defeats the purpose.

## Services Affected

| Service | Idempotency Mechanism |
| --- | --- |
| ledger-service | `idempotency_key` column on LedgerEntry; unique constraint |
| payment-orchestrator | `PaymentIdempotencyKey` entity; unique constraint on key |
| notification-service | `ProcessedMessage` entity; deduplication by `eventId` |
| customer-account-service | Not financial but may benefit from idempotent customer/account creation to prevent duplicates |

## Alternatives Considered

- **Distributed lock (Redis/ZooKeeper)**: Rejected — adds infrastructure dependency and network round trips. Database check-and-set is simpler and sufficient.
- **Event sourcing**: Rejected as over-engineering for this scale. Double-entry ledger already provides an immutable history.
