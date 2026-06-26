# AGENTS.md — Mini Banking Ecosystem

When you need to look up framework or library documentation, use the `context7` MCP tool.

## Build & Test Commands

### Root-level

```bash
# Build all services
./gradlew build

# Run all tests
./gradlew test

# Run tests for a specific service
./gradlew services:customer-account-service:test
./gradlew services:ledger-service:test

# Lint all services (SpotBugs)
./gradlew check

# Format all services (Spotless)
./gradlew spotlessApply
```

### Per-service (from service directory)

```bash
./gradlew :services:customer-account-service:build
./gradlew :services:customer-account-service:test
./gradlew :services:customer-account-service:bootRun
```

### Docker Compose (infrastructure only)

```bash
# Start infrastructure (PostgreSQL, Kafka, MailHog)
docker compose -f platform/local-dev/docker-compose/docker-compose.infrastructure.yml up -d

# Start all services
docker compose -f platform/local-dev/docker-compose/docker-compose.services.yml up -d
```

## Code Style & Conventions

### Java / Spring Boot

- **Version**: Java 21, Spring Boot 3.5.11
- **Build tool**: Gradle (Kotlin DSL preferred)
- **Package structure**: `com.watabank.<service>.<layer>` — e.g., `com.watabank.customeraccount.domain`, `com.watabank.customeraccount.web`
- **Layered architecture per service**: `domain/` (entities, value objects, domain services), `application/` (use cases, ports), `infrastructure/` (persistence, messaging, external clients), `web/` (REST controllers, DTOs)
- **Naming**: Classes PascalCase, methods/variables camelCase, constants UPPER_SNAKE_CASE
- **Tests**: JUnit 5 + AssertJ + Mockito; test class names end with `Test` or `IT` (integration tests)
- **Lombok**: Use `@RequiredArgsConstructor`, `@Getter`, `@Builder` — avoid `@Data`, `@Setter`
- **Validation**: Bean Validation annotations on DTOs (`@NotBlank`, `@NotNull`, `@Positive`)
- **Database**: PostgreSQL 17.10 with Flyway migrations (`src/main/resources/db/migration/`)
- **API docs**: OpenAPI/Swagger annotations on controllers
- **No comments**: Code should be self-documenting. No Javadoc on trivial methods. No inline comments for "what" — only "why" for non-obvious decisions.

### Domain-Driven Design

- Each service is a bounded context — no shared domain models across services
- Domain events are published via the outbox pattern where transactional consistency is required
- Entity IDs are UUIDs, immutable, assigned at creation
- Use `@Embeddable` for value objects, `@EmbeddedId` for composite keys where appropriate
- Idempotency keys for write operations: check-and-set pattern in a single transaction

### Testing

- **Unit tests**: Service layer, domain logic, validators — fast, no Spring context
- **Integration tests**: Repository layer, REST endpoints, messaging — use Testcontainers for PostgreSQL
- **Concurrency tests**: Ledger and payment services — use parallel test execution with distinct idempotency keys
- **Test naming**: `should<ExpectedBehavior>When<Condition>` or `methodName_givenCondition_expectedResult`
- **Coverage target**: 80%+ on domain and application layers

### Git Commits

- Conventional Commits: `feat(ledger): add double-entry posting endpoint`
- One commit per atomic change — never batch unrelated work
- Scope matches the service name: `customer-account`, `ledger`, `payment`, `notification`, `reconciliation`, `platform`
- Never commit secrets, .env files, or IDE-specific files

## Project Structure

```
/
├── AGENTS.md                         # This file — AI coding agent guide
├── README.md                         # Project overview for humans
├── build.gradle.kts                  # Root build (if mono-repo Gradle)
├── settings.gradle.kts               # Gradle settings with service includes
├── .specs/                           # Spec-driven development artifacts
│   ├── STATE.md                      # Project decisions log (ADRs) + handoff
│   └── features/                     # Feature specifications
│       └── [feature]/
│           ├── spec.md               # Requirements with traceable IDs
│           ├── tasks.md              # Atomic tasks with verification
│           └── validation.md         # Verifier report
├── local-docs/                       # Reference documents (input specs)
├── platform/                         # Cross-cutting engineering foundation
│   ├── ci/                           # CI/CD workflow templates
│   ├── observability/                # Prometheus, Grafana, OTel configs
│   ├── security/                     # Auth baselines, security headers
│   ├── local-dev/                    # Docker Compose, startup scripts
│   │   ├── docker-compose/
│   │   └── scripts/
│   └── docs/                         # Platform documentation
│       ├── architecture/             # System overview, service map, data flow
│       └── adrs/                     # Architecture Decision Records
└── services/                         # Business microservices
    ├── customer-account-service/     # Customers, accounts, status transitions
    ├── ledger-service/               # Double-entry bookkeeping, immutable ledger
    ├── payment-orchestrator/         # Payment intents, saga coordination
    ├── notification-service/         # Async event consumers, notifications
    └── reconciliation-service/       # Batch reconciliation, matching rules
```

### Service Internal Structure

Each service follows this layout:

```
services/<service-name>/
├── README.md
├── build.gradle.kts
├── Dockerfile
├── src/
│   ├── main/
│   │   ├── java/com/watabank/<servicename>/
│   │   │   ├── <ServiceName>Application.java
│   │   │   ├── domain/
│   │   │   ├── application/
│   │   │   ├── infrastructure/
│   │   │   └── web/
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   └── test/
│       └── java/com/watabank/<servicename>/
├── docs/
│   ├── domain-model.md
│   ├── api-contract.md
│   └── flow-diagram.md
```

## Service Map

| Service | Role | Depends On | Exposes |
| --- | --- | --- | --- |
| customer-account-service | Customer and account CRUD, status management | PostgreSQL | REST API |
| ledger-service | Immutable double-entry bookkeeping | PostgreSQL, Kafka/RabbitMQ | REST API + domain events |
| payment-orchestrator | Payment intent coordination, saga | customer-account-service, ledger-service, PostgreSQL, Kafka/RabbitMQ | REST API + domain events |
| notification-service | Async event consumption, notifications | Kafka/RabbitMQ, PostgreSQL/Redis | Consumer only (no API) |
| reconciliation-service | Batch reconciliation, matching | payment-orchestrator, ledger-service, PostgreSQL | REST API |

## Local Development

### Prerequisites

- Java 21 (Adoptium or similar)
- Docker and Docker Compose
- Gradle (wrapper included: `./gradlew`)

### Quick Start

```bash
# 1. Start infrastructure
docker compose -f platform/local-dev/docker-compose/docker-compose.infrastructure.yml up -d

# 2. Wait for dependencies
./platform/local-dev/scripts/wait-for-deps.sh

# 3. Build all services
./gradlew build

# 4. Run a specific service
./gradlew :services:customer-account-service:bootRun

# 5. Smoke test
./platform/local-dev/scripts/smoke-test.sh
```

### Observability in Dev

- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000` (admin/admin)
- Service health: `http://localhost:<port>/actuator/health`
- Service metrics: `http://localhost:<port>/actuator/prometheus`

### Correlation ID

All services propagate a `X-Correlation-ID` header. Include it in all inter-service calls. Structured logs include the correlation ID automatically.

## Shared Standards

Every service MUST implement:
- Health endpoint (`/actuator/health`)
- Metrics endpoint (`/actuator/prometheus`)
- Structured logging (JSON with correlation ID)
- Correlation ID propagation (inbound → outbound)
- Flyway database migrations
- Integration tests with Testcontainers
- OpenAPI documentation if HTTP is exposed
