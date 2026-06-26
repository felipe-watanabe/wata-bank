# Payment Orchestrator — API Contract

**Base Path**: `/api/v1`

All endpoints require `X-Correlation-ID` header. Responses use standard HTTP status codes and JSON bodies.

## Payment Intents

### Create Payment Intent

```
POST /api/v1/payments
```

**Request Body**:
```json
{
  "debitAccountId": "uuid (required)",
  "creditAccountId": "uuid (required)",
  "amount": "decimal (required, positive)",
  "currency": "string (required, ISO 4217)",
  "reference": "string (optional, external reference)",
  "idempotencyKey": "string (required, UUID format)"
}
```

**Responses**:
- `202 Accepted` — Payment intent created, processing started
  ```json
  {
    "paymentIntentId": "uuid",
    "status": "PENDING",
    "debitAccountId": "uuid",
    "creditAccountId": "uuid",
    "amount": "decimal",
    "currency": "string",
    "createdAt": "ISO-8601"
  }
  ```
- `200 OK` — Idempotent response (same key, returns original result)
- `400 Bad Request` — Validation error (invalid accounts, amount <= 0, missing fields)
- `409 Conflict` — Debit and credit accounts are the same

### Get Payment Intent

```
GET /api/v1/payments/{paymentIntentId}
```

**Responses**:
- `200 OK` — Full payment intent with current status and attempt history
  ```json
  {
    "paymentIntentId": "uuid",
    "status": "SETTLED | REJECTED | PENDING | ...",
    "debitAccountId": "uuid",
    "creditAccountId": "uuid",
    "amount": "decimal",
    "currency": "string",
    "createdAt": "ISO-8601",
    "updatedAt": "ISO-8601",
    "attempts": [
      {
        "attemptNumber": 1,
        "status": "SETTLED",
        "ledgerEntryId": "uuid",
        "attemptedAt": "ISO-8601",
        "completedAt": "ISO-8601"
      }
    ],
    "statusHistory": [
      {
        "previousStatus": "PENDING",
        "newStatus": "VALIDATING",
        "changedAt": "ISO-8601"
      }
    ]
  }
  ```
- `404 Not Found` — Payment intent does not exist

### List Payment Intents

```
GET /api/v1/payments?status={status}&page={page}&size={size}
```

**Query Parameters**:
- `status` (optional) — Filter by PaymentStatus
- `page` (default: 0)
- `size` (default: 20, max: 100)

**Responses**:
- `200 OK` — Paginated list of payment intent summaries

## Payment Lifecycle Events

The orchestrator publishes events to Kafka/RabbitMQ via the outbox pattern on state transitions:

| Transition | Event Type | Payload |
| --- | --- | --- |
| Intent created | `PAYMENT_CREATED` | `{ paymentIntentId, debitAccountId, creditAccountId, amount }` |
| Payment settled | `PAYMENT_SETTLED` | `{ paymentIntentId, ledgerEntryId, amount }` |
| Payment rejected | `PAYMENT_REJECTED` | `{ paymentIntentId, reason }` |
| Payment failed | `PAYMENT_FAILED` | `{ paymentIntentId, reason, attemptNumber }` |

Consumers (e.g., notification-service) subscribe to these event types.

## Error Handling

| Scenario | HTTP Status | PaymentIntent Status | Behavior |
| --- | --- | --- | --- |
| Blocked debit account | 200 OK (poll) | REJECTED | Reject immediately |
| Blocked credit account | 200 OK (poll) | REJECTED | Reject immediately |
| Ledger service timeout | 200 OK (poll) | FAILED | Retry with exponential backoff (max 3 attempts) |
| Ledger service unavailable | 200 OK (poll) | FAILED | Retry with circuit breaker |
| Idempotency key collision | 200 OK | — | Return original response |
