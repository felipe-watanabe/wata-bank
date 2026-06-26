# reconciliation-service

## Purpose

Compares external transaction records with internal payment and ledger data. Demonstrates operational fintech workflows such as matching, divergence detection, and exception handling. Reconciliation is read-only — it does not mutate financial truth directly.

## Core Responsibilities

- Import external transaction files or feeds (CSV, JSON)
- Compare external data with payment records (payment-orchestrator)
- Compare external data with ledger records (ledger-service)
- Identify MATCHED, MISSING, and DIVERGENT cases
- Expose reconciliation outcomes for operational review
- Persist matching results for audit and reporting

## Tech Stack

- Java 21
- Spring Boot 3.5.11
- Spring Batch or scheduled jobs
- PostgreSQL 17.10
- CSV / JSON parsing
- Scheduler (cron or fixed delay)
- REST API for operational endpoints
- Testcontainers

## Package Structure

```
com.watabank.reconciliation/
├── ReconciliationServiceApplication.java
├── domain/
│   ├── ReconciliationJob.java
│   ├── ExternalTransaction.java
│   ├── MatchResult.java
│   ├── Divergence.java
│   └── ExceptionCase.java
├── application/
│   ├── ReconciliationService.java
│   └── MatchingEngine.java
├── infrastructure/
│   ├── persistence/
│   │   ├── ReconciliationJobRepository.java
│   │   ├── ExternalTransactionRepository.java
│   │   ├── MatchResultRepository.java
│   │   └── DivergenceRepository.java
│   └── client/
│       ├── PaymentOrchestratorClient.java
│       └── LedgerServiceClient.java
└── web/
    ├── ReconciliationController.java
    └── dto/
```

## Building & Running

```bash
./gradlew :services:reconciliation-service:build
./gradlew :services:reconciliation-service:test
./gradlew :services:reconciliation-service:bootRun
```

## API

See [api-contract.md](docs/api-contract.md) for full endpoint documentation.

## Domain Model

See [domain-model.md](docs/domain-model.md) for entity relationships and business rules.

## Key Flows

See [flow-diagram.md](docs/flow-diagram.md) for job lifecycle and matching logic.
