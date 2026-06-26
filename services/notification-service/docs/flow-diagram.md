# Notification Service — Flow Diagram

## Idempotent Consumer Flow

```
Kafka Broker → Consumer.poll()
  │
  ├─ Receive event (eventId = abc-123, eventType = "PAYMENT_SETTLED")
  │
  ├─ BEGIN TRANSACTION
  │
  ├─ SELECT FROM processed_messages WHERE event_id = 'abc-123'
  │   ├─ EXISTS → NOP
  │   │   ├─ COMMIT
  │   │   ├─ Acknowledge offset
  │   │   └─ DONE (duplicate discarded)
  │   │
  │   └─ NOT EXISTS → continue processing
  │
  ├─ Determine template from eventType:
  │   ├─ PAYMENT_SETTLED → "payment-settled" template
  │   ├─ PAYMENT_REJECTED → "payment-rejected" template
  │   └─ PAYMENT_FAILED → "payment-failed" template
  │
  ├─ Render notification:
  │   ├─ Look up recipient(s) from event payload
  │   ├─ Render template with event data
  │   └─ Create Notification (status = PENDING)
  │
  ├─ Insert ProcessedMessage (event_id = 'abc-123', processedAt = now)
  │
  ├─ COMMIT TRANSACTION
  │
  ├─ Acknowledge offset
  │
  └─ (Async) Dispatch notification → see Dispatch Flow
```

## Dispatch Flow

```
NotificationDispatcher (background):
  │
  ├─ SELECT FROM notifications WHERE status = 'PENDING' ORDER BY created_at
  │
  └─ For each notification:
      │
      ├─ Create NotificationAttempt (attemptNumber, status = PENDING)
      │
      ├─ Deliver:
      │   ├─ EMAIL → SMTP send (MailHog in dev)
      │   │   ├─ Success → Notification.status = SENT
      │   │   │            NotificationAttempt.status = SENT
      │   │   │
      │   │   └─ Failure →
      │   │       ├─ Determine failure type:
      │   │       │   ├─ Network timeout → TRANSIENT
      │   │       │   └─ Invalid email → TERMINAL
      │   │       │
      │   │       ├─ TRANSIENT + attempt < 3:
      │   │       │   └─ Schedule retry (backoff: 1s, 2s, 4s)
      │   │       │
      │   │       ├─ TRANSIENT + attempt >= 3:
      │   │       │   └─ Route to DLQ
      │   │       │      Create DeliveryFailure (TRANSIENT)
      │   │       │
      │   │       └─ TERMINAL:
      │   │           ├─ Route to DLQ
      │   │           ├─ Create DeliveryFailure (TERMINAL)
      │   │           └─ Notification.status = FAILED
      │   │
      │   └─ Other channels: future (SMS, PUSH, IN_APP)
      │
      └─ Continue next notification
```

## DLQ Handling

```
Dead Letter Queue Consumer (monitoring):
  │
  ├─ Consume failed events from DLQ topic
  │
  ├─ Log structured error: { eventId, failureType, reason, notificationId }
  │
  ├─ Persist DeliveryFailure record for operational review
  │
  └─ Expose operational endpoint: GET /actuator/health (includes DLQ backlog count)
```

## Event Processing Scenarios

### Scenario 1: Payment Settled → Two Notifications

```
PAYMENT_SETTLED event received:
  ├─ eventId = abc-123
  ├─ Recipient 1: debitAccountId (payer)
  │   └─ Notification: "Your payment of $100.00 was sent to account {creditAccountId}"
  ├─ Recipient 2: creditAccountId (payee)
  │   └─ Notification: "You received $100.00 from account {debitAccountId}"
  └─ Both processed in same transaction with a single ProcessedMessage insert
```

### Scenario 2: Duplicate Event → Idempotent NOP

```
PAYMENT_SETTLED event received (redelivery):
  ├─ eventId = abc-123 (same as before)
  ├─ SELECT FROM processed_messages → EXISTS
  └─ NOP — no new notifications, no side effects
```

### Scenario 3: Transient Failure → Retry → Success

```
Attempt 1:
  ├─ SMTP send fails: connection timeout
  ├─ FailureType = TRANSIENT
  └─ Schedule retry in 1s

Attempt 2:
  ├─ SMTP send fails: connection timeout (again)
  ├─ FailureType = TRANSIENT
  └─ Schedule retry in 2s

Attempt 3:
  ├─ SMTP send succeeds
  ├─ Notification.status = SENT
  └─ NotificationAttempt.status = SENT
```

### Scenario 4: Terminal Failure → DLQ

```
Attempt 1:
  ├─ SMTP send fails: "Invalid recipient email address"
  ├─ FailureType = TERMINAL
  ├─ Route to DLQ immediately (no retries)
  ├─ Create DeliveryFailure record
  ├─ Notification.status = FAILED
  └─ Log error for operational review
```
