# Service Map

## Service Dependency Graph

```
┌─────────────────────────────────────────────────────────────┐
│                        External World                       │
│  (API clients, external transaction feeds, notification    │
│   delivery channels — MailHog/SMTP dev, real in prod)      │
└────────────────────┬───────────────────┬──────────┬─────────┘
                     │                   │          │
                     ▼                   │          ▼
         ┌───────────────────┐           │   ┌───────────────────┐
         │ customer-account  │           │   │ reconciliation    │
         │ service           │           │   │ service           │
         │ (REST API)        │           │   │ (REST API)        │
         └────────┬──────────┘           │   └────────┬──────────┘
                  │                      │            │
                  │ sync (REST)          │            │ sync (REST)
                  ▼                      │            ▼
         ┌───────────────────┐           │   ┌───────────────────┐
         │ payment           │◄──────────┘   │ ledger            │
         │ orchestrator      │               │ service           │
         │ (REST API)        │               │ (REST API)        │
         └──┬───────────┬────┘               └───────────────────┘
            │           │
            │ sync      │ async (domain events via
            │ (REST)    │ Kafka/RabbitMQ + outbox)
            ▼           ▼
         ┌───────────────────┐    async    ┌───────────────────┐
         │ ledger            │◄────────────│ notification      │
         │ service           │  consumes   │ service           │
         │ (REST API)        │  events     │ (consumer only)   │
         └───────────────────┘             └───────────────────┘
```

## Service Details

### customer-account-service

| Aspect | Detail |
| --- | --- |
| **Role** | Foundational domain service for customer and account management |
| **Depends On** | PostgreSQL |
| **Exposes** | REST API |
| **Consumes** | Nothing (source of truth) |
| **Key Entities** | Customer, Account, AccountStatusHistory, AccountLimit, AccountBlockReason |

### ledger-service

| Aspect | Detail |
| --- | --- |
| **Role** | Financial core — immutable double-entry bookkeeping |
| **Depends On** | PostgreSQL, Kafka/RabbitMQ |
| **Exposes** | REST API, domain events (via outbox) |
| **Consumes** | Nothing (receives posting requests synchronously) |
| **Key Entities** | LedgerEntry, LedgerLine, AccountReference, IdempotencyKey, OutboxEvent |

### payment-orchestrator

| Aspect | Detail |
| --- | --- |
| **Role** | Coordinates payment intents across account and ledger services |
| **Depends On** | customer-account-service (REST), ledger-service (REST), PostgreSQL, Kafka/RabbitMQ |
| **Exposes** | REST API, domain events (via outbox) |
| **Consumes** | Nothing (drives orchestration) |
| **Key Entities** | PaymentIntent, PaymentAttempt, PaymentStatusHistory, PaymentIdempotencyKey, PaymentEvent |

### notification-service

| Aspect | Detail |
| --- | --- |
| **Role** | Asynchronous reaction to business state changes |
| **Depends On** | Kafka/RabbitMQ, PostgreSQL/Redis (deduplication) |
| **Exposes** | Nothing (consumer only — no public API) |
| **Consumes** | Payment lifecycle events, account events |
| **Key Entities** | Notification, NotificationAttempt, ProcessedMessage, NotificationTemplate, DeliveryFailure |

### reconciliation-service

| Aspect | Detail |
| --- | --- |
| **Role** | Batch comparison of external records against internal data |
| **Depends On** | payment-orchestrator (REST), ledger-service (REST), PostgreSQL |
| **Exposes** | REST API |
| **Consumes** | External transaction files/feeds (import) |
| **Key Entities** | ReconciliationJob, ExternalTransaction, MatchResult, Divergence, ExceptionCase |

## Communication Patterns

### Synchronous (REST)
- **payment-orchestrator → customer-account-service**: Validate account status and limits before payment
- **payment-orchestrator → ledger-service**: Post balanced financial entries for settlement
- **reconciliation-service → payment-orchestrator / ledger-service**: Query payment and ledger records for matching

### Asynchronous (Events via Kafka/RabbitMQ + Outbox)
- **ledger-service → Kafka**: Publish financial events (e.g., posting completed)
- **payment-orchestrator → Kafka**: Publish payment lifecycle events (e.g., payment settled, payment failed)
- **notification-service → Kafka**: Consume payment and account events, trigger notifications

## Infrastructure Dependencies

| Component | Purpose | Used By |
| --- | --- | --- |
| PostgreSQL 17.10 | Primary data store | All services that persist state |
| Kafka / RabbitMQ | Event broker | ledger-service, payment-orchestrator, notification-service |
| MailHog / SMTP | Email notification delivery (dev) | notification-service |
| Redis | Deduplication cache (optional) | notification-service |
