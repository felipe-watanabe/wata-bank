# Payment Orchestrator — Domain Model

## Entities

### PaymentIntent

Represents a request to move funds between two accounts. Owns the payment lifecycle.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `debitAccountId` | UUID | Source account (funds withdrawn) |
| `creditAccountId` | UUID | Destination account (funds deposited) |
| `amount` | BigDecimal | Payment amount (always positive) |
| `currency` | String | ISO 4217 currency code |
| `status` | PaymentStatus | Current lifecycle state |
| `reference` | String | External reference (e.g., transaction ID from client) |
| `idempotencyKey` | String | Unique key preventing duplicate payment execution |
| `createdAt` | Instant | Creation timestamp |
| `updatedAt` | Instant | Last status change timestamp |

### PaymentStatus (Enum)

| Value | Description |
| --- | --- |
| `PENDING` | Payment intent created — awaiting validation and execution |
| `VALIDATING` | Account validation in progress |
| `DEBITING` | Ledger posting in progress (debit source, credit destination) |
| `SETTLED` | Payment completed successfully — terminal success state |
| `REJECTED` | Payment rejected (blocked account, insufficient limits) — terminal failure state |
| `FAILED` | Payment failed with retryable error — may be retried |
| `COMPENSATING` | Compensation in progress (reversal of a settled payment) |

### PaymentAttempt

Records each execution attempt of a payment intent.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `paymentIntentId` | UUID | Reference to PaymentIntent |
| `attemptNumber` | int | Sequential attempt number (1, 2, 3...) |
| `status` | PaymentStatus | Status reached by this attempt |
| `ledgerEntryId` | UUID | Reference to the LedgerEntry created by this attempt (null if not yet posted) |
| `failureReason` | String | Description of failure (null if successful) |
| `attemptedAt` | Instant | When this attempt was started |
| `completedAt` | Instant | When this attempt concluded |

### PaymentStatusHistory

Audit trail of all status transitions.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `paymentIntentId` | UUID | Reference to PaymentIntent |
| `previousStatus` | PaymentStatus | Status before transition |
| `newStatus` | PaymentStatus | Status after transition |
| `changedAt` | Instant | When the transition occurred |

### PaymentIdempotencyKey

Prevents duplicate payment execution.

| Field | Type | Description |
| --- | --- | --- |
| `key` | String | Unique idempotency key from client |
| `paymentIntentId` | UUID | Reference to the PaymentIntent created/returned |
| `response` | String | JSON-serialized response |
| `createdAt` | Instant | When first processed |

### PaymentEvent

Outbox event published on payment state changes.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `paymentIntentId` | UUID | Reference to PaymentIntent |
| `eventType` | String | Discriminator (e.g., "PAYMENT_CREATED", "PAYMENT_SETTLED", "PAYMENT_REJECTED") |
| `payload` | String | JSON-serialized event data |
| `createdAt` | Instant | When recorded |
| `publishedAt` | Instant | When published (null if pending) |

## Relationships

```
PaymentIntent (1) ────────── (N) PaymentAttempt
PaymentIntent (1) ────────── (N) PaymentStatusHistory
PaymentIntent (1) ────────── (0..1) PaymentIdempotencyKey
PaymentIntent (1) ────────── (N) PaymentEvent
```

## Business Rules

1. The same payment request MUST NOT settle more than once (idempotency key check).
2. Blocked or invalid accounts cannot proceed to settlement — payment is REJECTED.
3. Failed transitions must be captured explicitly in PaymentStatusHistory.
4. Transient failures (network timeout, service unavailable) are RETRYABLE with backoff.
5. Lifecycle states must be explicit and auditable — no implicit state.
6. A payment in a terminal state (SETTLED, REJECTED) cannot be retried.
7. Compensation (reversal) is a separate payment intent with its own lifecycle.
