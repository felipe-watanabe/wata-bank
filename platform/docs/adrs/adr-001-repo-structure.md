# ADR-001: Mono-Repo Structure

- **Status**: Accepted
- **Date**: 2026-06-25

## Context

The mini banking ecosystem consists of five business services and one cross-cutting platform layer. We need to decide on a repository strategy: mono-repo vs. multi-repo.

## Decision

Use a mono-repo with `platform/` and `services/` top-level directories.

## Rationale

1. **LLM-assisted scaffolding**: An LLM can generate consistent structure, shared documentation, and local orchestration in a single pass.
2. **Local development**: A single repository simplifies Docker Compose orchestration — all services can be started with one command.
3. **Shared standards**: Platform standards (CI, observability, security) are versioned alongside the code they govern.
4. **Refactoring agility**: Cross-service changes can be made and tested atomically.
5. **Small team**: With a single developer or small team, mono-repo overhead (merge conflicts, build times) is minimal.

## Consequences

- **Positive**: Simplified CI/CD — one pipeline can build and test all services. Shared Gradle build orchestration.
- **Positive**: Platform docs, scripts, and infrastructure config live alongside service code.
- **Negative**: Service-level CI independence is deferred. If a team grows, repository boundaries may need to shift.
- **Negative**: Monorepo build tooling (Gradle multi-project) adds complexity for service-specific dependency versioning.

## Alternatives Considered

- **Multi-repo**: Rejected for now due to higher upfront scaffolding cost and reduced local development simplicity. Can be adopted later if team size warrants it.
