# payment-orchestrator

## Purpose

Coordinates payment intents, state transitions, and the interaction between accounts and ledger. Demonstrates distributed coordination, eventual consistency, and failure handling through a simplified saga-like approach. The orchestrator does not hold money — it coordinates the debit and credit instructions across services.

## Core Responsibilities

- Create payment intents with explicit lifecycle states
- Validate payment execution preconditions (account status via customer-account-service)
- Coordinate payment state transitions across services
- Integrate with ledger-service for financial posting
- Publish payment lifecycle events (via outbox)
- Protect against duplicate payment requests (idempotency)
- Handle retry and compensation for transient failures

## Tech Stack

- Java 21
- Spring Boot 3.5.11
- PostgreSQL 17.10
- Kafka / RabbitMQ (event publishing)
- Resilience4j (retry, circuit breaker)
- Outbox pattern (event publishing)
- Scheduler/worker for reprocessing
- Testcontainers
- OpenAPI/Swagger

## Package Structure

```
com.watabank.payment/
├── PaymentOrchestratorApplication.java
├── domain/
│   ├── PaymentIntent.java
│   ├── PaymentAttempt.java
│   ├── PaymentStatus.java
│   ├── PaymentStatusHistory.java
│   ├── PaymentIdempotencyKey.java
│   └── PaymentEvent.java
├── application/
│   ├── PaymentIntentService.java
│   ├── PaymentSaga.java
│   └── OutboxPublisher.java
├── infrastructure/
│   ├── persistence/
│   │   ├── PaymentIntentRepository.java
│   │   ├── PaymentAttemptRepository.java
│   │   └── OutboxEventRepository.java
│   └── client/
│       ├── AccountServiceClient.java
│       └── LedgerServiceClient.java
└── web/
    ├── PaymentController.java
    └── dto/
```

## Building & Running

```bash
./gradlew :services:payment-orchestrator:build
./gradlew :services:payment-orchestrator:test
./gradlew :services:payment-orchestrator:bootRun
```

## API

See [api-contract.md](docs/api-contract.md) for full endpoint documentation.

## Domain Model

See [domain-model.md](docs/domain-model.md) for entity relationships and business rules.

## Key Flows

See [flow-diagram.md](docs/flow-diagram.md) for saga orchestration and compensation flows.
