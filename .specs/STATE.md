# STATE

## Decisions

### AD-001
- **Decision**: Mono-repo with `platform/` and `services/` top-level directories
- **Reason**: LLM-assisted scaffolding benefits from a single consistent pass; local orchestration and shared documentation are easier in one repository
- **Trade-off**: Service-level CI independence is deferred; can adopt multi-repo later if needed
- **Scope**: Entire project — repository structure, build orchestration, shared documentation
- **Date**: 2026-06-25
- **Status**: active

### AD-002
- **Decision**: Java 21 + Spring Boot 3.5.11 + PostgreSQL 17.10 as the baseline stack for all services
- **Reason**: Provides consistency across services; simplifies local development and shared platform standards
- **Trade-off**: Locks all services to the same runtime and framework version; no polyglot flexibility
- **Scope**: All services under `services/`
- **Date**: 2026-06-25
- **Status**: active

### AD-003
- **Decision**: OpenAPI for public API documentation, Flyway for database migrations, Testcontainers for integration testing
- **Reason**: Industry-standard tools that align with portfolio-quality codebases; Testcontainers ensures realistic integration tests
- **Trade-off**: Adds dependency management overhead per service
- **Scope**: All services under `services/`
- **Date**: 2026-06-25
- **Status**: active

### AD-004
- **Decision**: Platform layer (`platform/`) is a non-domain engineering foundation — no business endpoints, no business database
- **Reason**: Keeps cross-cutting concerns centralized without creating a fake domain service; follows platform engineering patterns
- **Trade-off**: Standards are advisory, not enforced by a shared library — each service must consciously adopt them
- **Scope**: `platform/` directory
- **Date**: 2026-06-25
- **Status**: active

### AD-005
- **Decision**: Each service owns its own domain, persistence, API, event contracts, and implementation details; shared code is minimal, shared standards are strong
- **Reason**: Service autonomy preserves bounded contexts; strong shared standards ensure ecosystem consistency
- **Trade-off**: Duplication of boilerplate (health checks, metrics, logging config) across services
- **Scope**: All services under `services/`
- **Date**: 2026-06-25
- **Status**: active

### AD-006
- **Decision**: Outbox pattern for reliable event publishing in ledger-service and payment-orchestrator
- **Reason**: Transactional consistency between business state and event publishing is critical for financial correctness
- **Trade-off**: Additional outbox table and publisher complexity per service
- **Scope**: `services/ledger-service`, `services/payment-orchestrator`
- **Date**: 2026-06-25
- **Status**: active

### AD-007
- **Decision**: Docker Compose for local orchestration of all infrastructure dependencies (PostgreSQL, Kafka/RabbitMQ, Mailhog)
- **Reason**: Simplifies local development setup; services can be started independently while sharing infrastructure
- **Trade-off**: Docker Compose network complexity grows with service count
- **Scope**: `platform/local-dev/docker-compose/`
- **Date**: 2026-06-25
- **Status**: active

## Handoff

- **Feature**: customer-account-service — full service implementation
- **Phase / Task**: Specify + Design + Tasks complete — awaiting user confirmation to begin Execute
- **Completed**: spec.md (9 stories, 9 requirements, 60+ ACs), design.md (14 components, 4 layers, 6 migrations, error strategy), tasks.md (12 tasks, 6 phases, test coverage matrix, co-location validated)
- **In-progress** (file:line): none — all planning artifacts written
- **Next step**: User confirms tasks; agent enters Execute phase (implement T1 → T12 per tasks.md)
- **Blockers**: none
- **Uncommitted files**: .specs/features/customer-account-service/spec.md, design.md, tasks.md; .specs/STATE.md (handoff updated)
- **Branch**: main
