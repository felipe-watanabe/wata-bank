# System Overview

## Mini Banking Ecosystem

The mini banking ecosystem is composed of five business microservices and one cross-cutting platform layer. Each service operates as an independent bounded context, owning its own domain, persistence, and API contracts. Services communicate via synchronous REST calls for commands requiring immediate consistency and asynchronous domain events for integration across bounded contexts.

## Ecosystem Scope

### Business Services

| Service | Responsibility |
| --- | --- |
| **customer-account-service** | Customer and account CRUD, status management, account-level restrictions. Source of truth for customer and account metadata. |
| **ledger-service** | Immutable double-entry bookkeeping. Balance derivation from postings. Idempotent financial postings. Outbox event publishing. |
| **payment-orchestrator** | Payment intent coordination. Saga-like orchestration across account validation and ledger posting. State machine with retry and compensation. |
| **notification-service** | Asynchronous event consumption. Idempotent consumer with deduplication. Transient failure retry. DLQ handling. |
| **reconciliation-service** | Batch comparison of external transaction records against internal payment and ledger data. Matching, divergence detection, exception reporting. |

### Platform Layer

The `platform/` directory is a non-domain engineering foundation. It centralizes:

- CI/CD workflow templates
- Observability configuration (Prometheus, Grafana, OpenTelemetry)
- Security baselines and auth guidance
- Local development orchestration (Docker Compose, startup scripts)
- Architecture documentation and Architecture Decision Records (ADRs)

The platform layer does NOT expose business APIs or own business persistence.

## Shared Architecture Standards

All services adhere to:

- **Runtime**: Java 21, Spring Boot 3.5.11
- **Persistence**: PostgreSQL 17.10 with Flyway migrations
- **Testing**: JUnit 5 + AssertJ + Mockito; Testcontainers for integration tests
- **API Documentation**: OpenAPI/Swagger
- **Observability**: Structured JSON logging, health checks (`/actuator/health`), metrics (`/actuator/prometheus`), correlation ID propagation (`X-Correlation-ID`)
- **Messaging**: Kafka or RabbitMQ for domain events; outbox pattern where transactional consistency with business state is required
- **Idempotency**: Check-and-set pattern in a single transaction for all write operations

## Design Principles

1. **Bounded Contexts**: Each service owns its domain. No shared domain models. Shared code is minimal; shared standards are strong.
2. **Event-Driven Integration**: Domain events published via the outbox pattern for reliable inter-service communication.
3. **Transactional Integrity**: Financial operations are ACID within a service. Cross-service coordination uses saga-like patterns with explicit state transitions.
4. **Observability First**: Every service emits structured logs, health checks, and metrics from day one.
5. **Idempotency by Default**: All write operations are idempotent. Duplicate requests produce no duplicate financial effects.
