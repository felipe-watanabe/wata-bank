# ledger-service

## Purpose

The financial core of the mini banking ecosystem. Responsible for immutable double-entry bookkeeping, balance derivation from postings, posting history, and financial integrity guarantees. Every debit must have a corresponding credit — balances are derived, never stored as source of truth.

## Core Responsibilities

- Create balanced financial postings (double-entry)
- Store immutable ledger history
- Derive balances from postings (not stored as mutable state)
- Prevent duplicate posting execution via idempotency keys
- Publish financial events reliably via the outbox pattern
- Expose posting history and balance queries

## Tech Stack

- Java 21
- Spring Boot 3.5.11
- PostgreSQL 17.10
- ACID transactions
- Flyway (database migrations)
- Kafka / RabbitMQ (event publishing)
- Outbox table pattern
- Testcontainers (integration + concurrency testing)
- OpenAPI/Swagger

## Package Structure

```
com.watabank.ledger/
├── LedgerServiceApplication.java
├── domain/
│   ├── LedgerEntry.java
│   ├── LedgerLine.java
│   ├── AccountReference.java
│   ├── IdempotencyKey.java
│   └── OutboxEvent.java
├── application/
│   ├── PostingService.java
│   ├── BalanceService.java
│   └── OutboxPublisher.java
├── infrastructure/
│   └── persistence/
│       ├── LedgerEntryRepository.java
│       ├── LedgerLineRepository.java
│       └── OutboxEventRepository.java
└── web/
    ├── PostingController.java
    └── BalanceController.java
```

## Building & Running

```bash
./gradlew :services:ledger-service:build
./gradlew :services:ledger-service:test
./gradlew :services:ledger-service:bootRun
```

## API

See [api-contract.md](docs/api-contract.md) for full endpoint documentation.

## Domain Model

See [domain-model.md](docs/domain-model.md) for entity relationships and business rules.

## Key Flows

See [flow-diagram.md](docs/flow-diagram.md) for posting workflow and idempotency guarantees.
