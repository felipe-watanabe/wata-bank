# notification-service

## Purpose

Consumes domain events asynchronously and reacts to business state changes. Demonstrates idempotent consumers, deduplication, transient/terminal failure separation, and operational handling of background processing. This service has no public REST API — it is a pure consumer.

## Core Responsibilities

- Consume payment lifecycle events (from payment-orchestrator)
- Consume account events if needed (from customer-account-service / ledger-service)
- Trigger notification actions (email, push, in-app — simulated via MailHog in dev)
- Prevent duplicate side effects via processed message tracking
- Distinguish transient and terminal failures — route to dead-letter queue (DLQ) when non-retryable
- Support reprocessing for retryable events

## Tech Stack

- Java 21
- Spring Boot 3.5.11
- Kafka / RabbitMQ (consumer)
- PostgreSQL (processed message tracking, notification persistence) or Redis (deduplication cache)
- MailHog / fake SMTP (email simulation in dev)
- Dead-letter queue (DLQ)
- Structured logging
- Testcontainers

## Package Structure

```
com.watabank.notification/
├── NotificationServiceApplication.java
├── domain/
│   ├── Notification.java
│   ├── NotificationAttempt.java
│   ├── ProcessedMessage.java
│   ├── NotificationTemplate.java
│   └── DeliveryFailure.java
├── application/
│   ├── PaymentEventConsumer.java
│   └── NotificationDispatcher.java
├── infrastructure/
│   ├── persistence/
│   │   ├── NotificationRepository.java
│   │   ├── ProcessedMessageRepository.java
│   │   └── NotificationAttemptRepository.java
│   └── messaging/
│       └── KafkaConsumerConfig.java
└── web/
    └── (none — consumer only)
```

## Building & Running

```bash
./gradlew :services:notification-service:build
./gradlew :services:notification-service:test
./gradlew :services:notification-service:bootRun
```

## Consumers

See [api-contract.md](docs/api-contract.md) for consumed event contracts.

## Domain Model

See [domain-model.md](docs/domain-model.md) for entity relationships and business rules.

## Key Flows

See [flow-diagram.md](docs/flow-diagram.md) for idempotent consumption and DLQ routing.
