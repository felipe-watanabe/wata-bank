# Project Foundation Specification

## Problem Statement

The mini banking ecosystem codebase needs its foundational documentation scaffold. The architecture and service boundaries are defined in `local-docs/mini-banking-specs-by-service.md`, but the project has no AGENTS.md for AI-assisted development, no platform architecture docs, no per-service domain documentation, and no `.specs/` structure for spec-driven development. Without this foundation, code generation tools and developer onboarding lack the reference point needed to build the 5 services correctly.

## Goals

- [ ] Create AGENTS.md with build/lint/test commands, coding conventions, and project structure that AI coding agents can follow
- [ ] Generate all platform documentation files (architecture overview, service map, data flow, ADRs)
- [ ] Generate per-service documentation (README.md, domain-model.md, api-contract.md, flow-diagram.md) for all 5 services
- [ ] Establish `.specs/STATE.md` with initial architecture decisions (AD-001 through AD-007)

## Out of Scope

| Feature | Reason |
| --- | --- |
| Service source code (Java, build files, Dockerfiles) | Documentation scaffold only; code comes in subsequent features |
| CI/CD workflow files | Platform CI templates are documentation placeholders; actual workflows come with service implementation |
| Docker Compose content | Local-dev boilerplate is scaffolded; working compose comes with service implementation |
| Observability config (Prometheus, Grafana, OTel) | Config placeholders only; real config comes with service implementation |

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| AGENTS.md format follows OpenCode convention | Standard agent guide: commands, conventions, project structure | Per user selection: "Standard agent guide" over full architectural reference | y |
| Per-service docs are descriptive (not code-generative) | Domain-model, API-contract, and flow-diagram describe intent not implementation | Implementation details come with service code; docs set the boundary | y |
| Source spec is authoritative for all content | All generated docs derive from mini-banking-specs-by-service.md | Single source of truth; no new domain decisions invented | y |

**Open questions:** none — all resolved or logged above.

## User Stories

### P1: AI coding agents can navigate and build the project ⭐ MVP

**User Story**: As a developer using AI coding tools, I want an AGENTS.md file that tells AI agents how to build, test, lint, and follow coding conventions so that generated code is consistent and correct.

**Why P1**: AGENTS.md is the entry point for all AI-assisted development; without it, every coding session starts blind.

**Acceptance Criteria**:

1. WHEN an AI agent reads AGENTS.md THEN it SHALL discover build, test, lint, and format commands for the project
2. WHEN an AI agent reads AGENTS.md THEN it SHALL understand the Java/Spring Boot coding conventions
3. WHEN an AI agent reads AGENTS.md THEN it SHALL understand the mono-repo structure and service boundaries
4. WHEN an AI agent reads AGENTS.md THEN it SHALL know how to run services locally

**Independent Test**: An AI agent reading AGENTS.md can correctly answer "How do I build service X?" and "What conventions should I follow?"

### P1: Platform documentation exists for the ecosystem ⭐ MVP

**User Story**: As a developer, I want platform architecture docs (system overview, service map, data flow, ADRs) so that I understand how services relate and what decisions shaped the architecture.

**Why P1**: Architecture docs are the shared understanding baseline for all service teams.

**Acceptance Criteria**:

1. WHEN a developer reads `platform/docs/architecture/system-overview.md` THEN they SHALL understand the ecosystem scope and component responsibilities
2. WHEN a developer reads `platform/docs/architecture/service-map.md` THEN they SHALL understand service-to-service dependencies and communication patterns
3. WHEN a developer reads `platform/docs/architecture/data-flow.md` THEN they SHALL understand how data moves between services
4. WHEN a developer lists ADRs THEN they SHALL find ADR-001 (repo structure), ADR-002 (observability baseline), ADR-003 (idempotency strategy)

**Independent Test**: A new developer can answer "How does a payment flow through the system?" by reading the platform docs.

### P1: Each service has its domain documentation ⭐ MVP

**User Story**: As a developer, I want per-service documentation (README, domain model, API contract, flow diagram) so that I understand the responsibilities and boundaries of each service before writing code.

**Why P1**: Service-level docs are the contract between specification and implementation.

**Acceptance Criteria**:

1. WHEN a developer reads any service README THEN they SHALL understand the service's purpose, core responsibilities, and tech stack
2. WHEN a developer reads any domain-model.md THEN they SHALL understand the suggested domain entities and their relationships
3. WHEN a developer reads any api-contract.md THEN they SHALL understand the endpoints the service SHALL expose
4. WHEN a developer reads any flow-diagram.md THEN they SHALL understand the key workflow sequences in text form

**Independent Test**: A developer assigned to implement a service can start coding from the service docs alone.

## Edge Cases

- All services must document their auth/model requirements consistently
- The notification service has no external API — its flow diagram must document consumer behavior
- The reconciliation service has batch/job semantics — its API contract must account for scheduled operations

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| --- | --- | --- | --- |
| FDN-01 | P1: AGENTS.md exists with commands and conventions | Execute | Verified |
| FDN-02 | P1: Platform system-overview.md | Execute | Verified |
| FDN-03 | P1: Platform service-map.md | Execute | Verified |
| FDN-04 | P1: Platform data-flow.md | Execute | Verified |
| FDN-05 | P1: Platform ADR-001 (repo structure) | Execute | Verified |
| FDN-06 | P1: Platform ADR-002 (observability baseline) | Execute | Verified |
| FDN-07 | P1: Platform ADR-003 (idempotency strategy) | Execute | Verified |
| FDN-08 | P1: customer-account-service docs (README + 3 docs) | Execute | Verified |
| FDN-09 | P1: ledger-service docs (README + 3 docs) | Execute | Verified |
| FDN-10 | P1: payment-orchestrator docs (README + 3 docs) | Execute | Verified |
| FDN-11 | P1: notification-service docs (README + 3 docs) | Execute | Verified |
| FDN-12 | P1: reconciliation-service docs (README + 3 docs) | Execute | Verified |
| FDN-13 | P1: .specs/STATE.md with initial ADRs | Execute | Verified |

**Coverage:** 13 total, 13 mapped to tasks, 0 unmapped

## Success Criteria

- [ ] AGENTS.md is readable by AI coding tools and answers all common "how to" questions
- [ ] Every service directory under `services/` has README.md, docs/domain-model.md, docs/api-contract.md, docs/flow-diagram.md
- [ ] Platform docs cover system overview, service map, data flow, and 3 ADRs
- [ ] .specs/STATE.md records all initial architecture decisions
