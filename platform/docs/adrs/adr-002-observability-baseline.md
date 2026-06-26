# ADR-002: Observability Baseline

- **Status**: Accepted
- **Date**: 2026-06-25

## Context

All services in the ecosystem must be observable for operational maturity. We need a baseline that every service adheres to without creating a shared library.

## Decision

Every service MUST implement the following observability baseline:

1. **Health endpoint** (`/actuator/health`): Spring Boot Actuator with a liveness and readiness probe.
2. **Metrics endpoint** (`/actuator/prometheus`): Micrometer metrics exposed in Prometheus format.
3. **Structured logging**: JSON-formatted logs with at minimum: timestamp, level, service name, correlation ID, and message.
4. **Correlation ID propagation**: Accept `X-Correlation-ID` header on inbound requests; forward it on all outbound HTTP calls and include it in published domain event headers.

## Rationale

1. **Consistency**: Every service is debugged the same way — grep for correlation ID across all service logs.
2. **No shared library**: Each service configures its own Actuator and logging. Standards are enforced through convention and PR review, not through a forced dependency.
3. **Portfolio quality**: Health checks, metrics, and structured logs are table stakes for production-grade microservices.

## Consequences

- **Positive**: Prometheus + Grafana can scrape all services uniformly. Log aggregation (ELK/Loki) can parse a consistent format.
- **Positive**: Correlation IDs enable distributed tracing even without a dedicated tracing backend.
- **Negative**: Each service must duplicate Actuator and logging configuration. Template conventions mitigate this.

## Alternatives Considered

- **Shared observability library**: Rejected — introduces coupling and versioning overhead. Standards + conventions achieve the same result with less maintenance burden.
- **OpenTelemetry auto-instrumentation**: Deferred as a future enhancement. The baseline uses Spring Boot's built-in capabilities first.
