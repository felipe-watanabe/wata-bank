# Notification Service — Domain Model

## Entities

### Notification

The core entity representing a notification triggered by a domain event.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `eventId` | String | ID of the triggering domain event |
| `eventType` | String | Event type discriminator (e.g., "PAYMENT_SETTLED") |
| `recipientAccountId` | UUID | Account that should receive the notification |
| `template` | String | Notification template used (e.g., "payment-settled") |
| `content` | String | Rendered notification content |
| `channel` | NotificationChannel | Delivery channel (EMAIL, SMS, PUSH, IN_APP) |
| `status` | NotificationStatus | PENDING, SENT, FAILED |
| `createdAt` | Instant | Creation timestamp |

### NotificationChannel (Enum)

| Value | Description |
| --- | --- |
| `EMAIL` | Email notification via SMTP (MailHog in dev) |
| `SMS` | SMS notification (future) |
| `PUSH` | Push notification (future) |
| `IN_APP` | In-app notification (future) |

### NotificationStatus (Enum)

| Value | Description |
| --- | --- |
| `PENDING` | Notification created but not yet delivered |
| `SENT` | Successfully delivered |
| `FAILED` | Delivery failed (transient or terminal) |

### NotificationAttempt

Records each delivery attempt for a notification.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `notificationId` | UUID | Reference to Notification |
| `attemptNumber` | int | Sequential attempt number |
| `status` | NotificationStatus | Result of this attempt |
| `failureReason` | String | Description of failure (null if SENT) |
| `attemptedAt` | Instant | When the attempt was made |
| `completedAt` | Instant | When the attempt concluded |

### ProcessedMessage

Tracks which events have already been processed — the deduplication guard.

| Field | Type | Description |
| --- | --- | --- |
| `eventId` | String | Unique event identifier from the broker |
| `processedAt` | Instant | When the event was first processed |

### NotificationTemplate

Templates for rendering notification content from event data.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `name` | String | Template name (e.g., "payment-settled") |
| `subject` | String | Template subject line |
| `body` | String | Template body with placeholders |
| `channel` | NotificationChannel | Delivery channel this template is for |

### DeliveryFailure

Persistent record of delivery failures for operational review.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `notificationId` | UUID | Reference to Notification |
| `eventId` | String | Original event ID |
| `failureType` | FailureType | TRANSIENT or TERMINAL |
| `failureReason` | String | Why delivery failed |
| `failedAt` | Instant | When the failure was recorded |
| `resolved` | boolean | Whether the failure has been addressed |

### FailureType (Enum)

| Value | Description |
| --- | --- |
| `TRANSIENT` | Retryable failure (SMTP timeout, network issue) |
| `TERMINAL` | Non-retryable failure (invalid email, template error) |

## Relationships

```
Notification (1) ────────── (N) NotificationAttempt
Notification (1) ────────── (0..1) DeliveryFailure
ProcessedMessage          standalone deduplication table
NotificationTemplate      standalone reference table
```

## Business Rules

1. The same event MUST NOT generate the same notification side effect twice (ProcessedMessage check).
2. Processed event identity (eventId) MUST be tracked in the same transaction as the notification.
3. Transient failures (timeouts, network) are RETRYABLE with backoff (max 3 attempts).
4. Terminal failures (invalid recipient, template error) are NOT RETRYABLE — routed to DLQ or DeliveryFailure.
5. All processing MUST be observable: structured logs with eventId, notificationId, and correlation ID.
6. Message acknowledgment to broker MUST happen AFTER successful processing (ProcessedMessage insert).
