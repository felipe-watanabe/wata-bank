# Notification Service — API Contract (Consumer Contracts)

This service has **no public REST API**. It is a pure consumer. This document defines the event contracts it consumes from Kafka/RabbitMQ.

## Consumed Events

All events are consumed from a Kafka topic or RabbitMQ exchange. Each event includes a `X-Correlation-ID` header for distributed tracing.

### Event: PAYMENT_SETTLED

Published by `payment-orchestrator` when a payment reaches the SETTLED state.

```json
{
  "eventId": "uuid",
  "eventType": "PAYMENT_SETTLED",
  "timestamp": "ISO-8601",
  "correlationId": "string",
  "payload": {
    "paymentIntentId": "uuid",
    "ledgerEntryId": "uuid",
    "debitAccountId": "uuid",
    "creditAccountId": "uuid",
    "amount": "decimal",
    "currency": "string"
  }
}
```

**Notification Action**: Send "payment settled" email to both debit and credit account holders. Two Notification records created — one per recipient.

### Event: PAYMENT_REJECTED

Published by `payment-orchestrator` when a payment is rejected.

```json
{
  "eventId": "uuid",
  "eventType": "PAYMENT_REJECTED",
  "timestamp": "ISO-8601",
  "correlationId": "string",
  "payload": {
    "paymentIntentId": "uuid",
    "debitAccountId": "uuid",
    "creditAccountId": "uuid",
    "amount": "decimal",
    "reason": "string"
  }
}
```

**Notification Action**: Send "payment rejected" email to the debit account holder with the rejection reason.

### Event: PAYMENT_FAILED

Published by `payment-orchestrator` when a payment fails with a retryable error.

```json
{
  "eventId": "uuid",
  "eventType": "PAYMENT_FAILED",
  "timestamp": "ISO-8601",
  "correlationId": "string",
  "payload": {
    "paymentIntentId": "uuid",
    "debitAccountId": "uuid",
    "creditAccountId": "uuid",
    "amount": "decimal",
    "reason": "string",
    "attemptNumber": 1
  }
}
```

**Notification Action**: No immediate user notification — internal operational alert for monitoring systems.

## Deduplication Contract

**Each `eventId` is processed at most once.** Consumers check the `ProcessedMessage` table before processing:

```
1. Receive event with eventId = abc-123
2. BEGIN TRANSACTION
3. SELECT FROM processed_messages WHERE event_id = 'abc-123'
4. If EXISTS → discard event (NOP), COMMIT, acknowledge offset
5. If NOT EXISTS → process, create Notification, insert ProcessedMessage, COMMIT, acknowledge offset
```

**Redelivery**: If the same event is redelivered by the broker (at-least-once), the ProcessedMessage check guarantees idempotency.

## Retry & DLQ Behavior

| Failure Type | Behavior |
| --- | --- |
| Transient (SMTP timeout, connection refused) | Retry with exponential backoff (1s, 2s, 4s) — max 3 attempts. On final failure, route to DLQ. |
| Terminal (invalid email format, missing template) | No retry. Route directly to DLQ. Persist DeliveryFailure record. |
| Duplicate event | Discard silently. NOP. |
| Kafka consumer poll failure | Kafka handles offset management — redelivers up to `max.poll.records` |
