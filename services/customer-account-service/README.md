# customer-account-service

## Purpose

The foundational domain service of the mini banking ecosystem. Manages customers, accounts, onboarding/activation status, account state transitions, limits, and account-level restrictions. Serves as the source of truth for customer and account metadata.

## Core Responsibilities

- Create and manage customers
- Create and manage accounts
- Track account status (PENDING → ACTIVE → BLOCKED → CLOSED)
- Block and unblock accounts with auditable reasons
- Expose account data to internal consumers via REST API
- Serve as source of truth for customer and account metadata

## Tech Stack

- Java 21
- Spring Boot 3.5.11
- Spring Data JPA
- PostgreSQL 17.10
- Flyway (database migrations)
- Bean Validation
- OpenAPI/Swagger (API documentation)
- Testcontainers (integration testing)

## Package Structure

```
com.watabank.customeraccount/
├── CustomerAccountApplication.java
├── domain/
│   ├── Customer.java
│   ├── Account.java
│   ├── AccountStatus.java
│   ├── AccountStatusHistory.java
│   ├── AccountLimit.java
│   └── AccountBlockReason.java
├── application/
│   ├── CustomerService.java
│   └── AccountService.java
├── infrastructure/
│   └── persistence/
│       ├── CustomerRepository.java
│       ├── AccountRepository.java
│       └── AccountStatusHistoryRepository.java
└── web/
    ├── CustomerController.java
    ├── AccountController.java
    └── dto/
```

## Building & Running

```bash
./gradlew :services:customer-account-service:build
./gradlew :services:customer-account-service:test
./gradlew :services:customer-account-service:bootRun
```

## API

See [api-contract.md](docs/api-contract.md) for full endpoint documentation.

## Domain Model

See [domain-model.md](docs/domain-model.md) for entity relationships and business rules.

## Key Flows

See [flow-diagram.md](docs/flow-diagram.md) for account lifecycle and state transitions.
