# Mini Banking Ecosystem

A portfolio-grade mini banking ecosystem demonstrating senior-level Java backend engineering, fintech domain modeling, distributed consistency, and operational maturity. Implemented as a monorepo with five business microservices and a cross-cutting platform layer.

## Architecture

The ecosystem consists of five autonomous services, each owning its own bounded context:

| Service | Role | Primary Tech |
| --- | --- | --- |
| [customer-account-service](services/customer-account-service/) | Customer and account CRUD, status management | Spring Boot, PostgreSQL |
| [ledger-service](services/ledger-service/) | Immutable double-entry bookkeeping | Spring Boot, PostgreSQL, Kafka |
| [payment-orchestrator](services/payment-orchestrator/) | Payment intent coordination, saga | Spring Boot, PostgreSQL, Kafka |
| [notification-service](services/notification-service/) | Async event consumption, notifications | Spring Boot, Kafka, MailHog |
| [reconciliation-service](services/reconciliation-service/) | Batch reconciliation, matching | Spring Boot, PostgreSQL |

The [platform](platform/) layer centralizes CI/CD, observability, security baselines, and local development tooling.

## Tech Stack

- Java 21, Spring Boot 3.5.11
- Gradle (Kotlin DSL)
- PostgreSQL 17.10 with Flyway migrations
- Kafka or RabbitMQ for domain events
- Testcontainers for integration testing
- OpenAPI/Swagger for API documentation

## Quick Start

### Prerequisites

- Java 21 (Adoptium or similar)
- Docker and Docker Compose

### Local Development

```bash
# 1. Start infrastructure (PostgreSQL, Kafka, MailHog)
docker compose -f platform/local-dev/docker-compose/docker-compose.infrastructure.yml up -d

# 2. Build all services
./gradlew build

# 3. Run a specific service
./gradlew :services:customer-account-service:bootRun
```

### Observability in Dev

- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000` (admin/admin)
- Service health: `http://localhost:<port>/actuator/health`

## Project Structure

```
├── AGENTS.md                    # AI coding agent guide
├── README.md                    # This file
├── .specs/                      # Spec-driven development artifacts
├── local-docs/                  # Reference documents
├── platform/                    # Cross-cutting engineering foundation
│   ├── ci/                      # CI/CD workflow templates
│   ├── observability/           # Prometheus, Grafana, OTel configs
│   ├── security/                # Auth baselines, security headers
│   ├── local-dev/               # Docker Compose, startup scripts
│   └── docs/                    # Platform documentation
│       ├── architecture/        # System overview, service map, data flow
│       └── adrs/                # Architecture Decision Records
└── services/                    # Business microservices
    ├── customer-account-service/
    ├── ledger-service/
    ├── payment-orchestrator/
    ├── notification-service/
    └── reconciliation-service/
```

## Documentation

- [AGENTS.md](AGENTS.md) — AI agent guide with build commands and coding conventions
- [System Overview](platform/docs/architecture/system-overview.md) — Ecosystem scope and design principles
- [Service Map](platform/docs/architecture/service-map.md) — Service dependencies and communication patterns
- [Data Flow](platform/docs/architecture/data-flow.md) — End-to-end scenarios (payment, reconciliation, notifications)
- [ADRs](platform/docs/adrs/) — Architecture Decision Records
- Per-service docs: [README + domain model + API contract + flow diagram](services/) for each service
