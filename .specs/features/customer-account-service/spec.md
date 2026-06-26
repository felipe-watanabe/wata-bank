# Customer Account Service — Full Implementation

## Problem Statement

The `customer-account-service` has complete documentation (domain model, API contract, flow diagrams) but zero implementation code. No Java sources, no build files, no database migrations, no tests. This feature implements the full service — the foundational domain service of the mini banking ecosystem that manages customers, accounts, status transitions, limits, and block reasons. It is the source of truth that `payment-orchestrator` and `ledger-service` will depend on.

## Goals

- [ ] Implement all 5 domain entities with JPA persistence and relation mappings
- [ ] Implement 13 REST endpoints (6 customer/account lifecycle + 1 internal validation + 4 limit CRUD + 2 customer/account detail)
- [ ] Implement full account state machine with auditable status history (PENDING → ACTIVE ⇄ BLOCKED, any → CLOSED)
- [ ] Create Flyway database migrations for all tables
- [ ] Create root + service Gradle build configuration (Java 21, Spring Boot 3.5.11)
- [ ] Implement idempotency keys on all write endpoints
- [ ] Achieve 80%+ test coverage on domain and application layers
- [ ] Expose health, metrics, and structured logging with correlation ID propagation

## Out of Scope

| Feature | Reason |
| --- | --- |
| Authentication/authorization | Deferred — MVP uses open internal endpoints |
| Rate limiting | Not needed for portfolio demonstration |
| Account balance tracking | Owned by ledger-service; this service only stores metadata |
| Customer update/delete endpoints | Not specified in API contract; deferred |
| Notification or event publishing | Outbox/events are for ledger-service and payment-orchestrator per AD-006 |
| Email verification flows | Simplified for MVP — email uniqueness enforced at creation |
| Kafka/RabbitMQ integration | This service is purely synchronous REST |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Build system scope | Full root + service Gradle builds (settings.gradle.kts at root, shared conventions in root build.gradle.kts, service-level build.gradle.kts) | No build files exist; root build enables future services | y |
| Account number generation | Numeric 10-digit sequential (zero-padded, e.g. 0000000001) | Simple, deterministic, bank-like format | y |
| Block reason concurrency | Multiple concurrent block reasons allowed; unblock resolves ALL active blocks | Accounts may be blocked for multiple independent reasons (e.g., FRAUD + REGULATORY) | y |
| Activation onboarding | Customer must have firstName, lastName, email present (guaranteed by creation) | Lightweight identity completeness check | y |
| Idempotency keys | Full idempotency via `X-Idempotency-Key` header on all POST/PUT endpoints | Protects against duplicate customer/account creation and duplicate status transitions | y |
| Internal endpoint auth | No authentication between services (trusted network) | Simplest for MVP; can add later | y |
| Account limits scope | Full CRUD endpoints for AccountLimit (create, read, update, delete per account) | Provides complete limit management surface | y |
| Unblock behavior | Unblock resolves ALL active block reasons simultaneously (sets unblockedAt on all) | Simplest concurrency model; specific-block unblock can be added later | y |
| Default account limits | Accounts get three default limits on creation: TRANSACTION=10,000.00, DAILY=50,000.00, MONTHLY=200,000.00 in the account's currency | Sensible defaults for portfolio demo; limits can be updated/deleted via CRUD endpoints | y |
| PENDING → CLOSED transition | Allowed — accounts can be closed from any state including PENDING | Spec says "any state → CLOSED (terminal)" | y |

**Open questions:** none — all resolved or logged above.

---

## User Stories

### P1.1: Customer Creation & Retrieval ⭐ MVP

**User Story**: As a bank operator, I want to create customers and retrieve their details so that accounts can be opened under verified customer identities.

**Why P1**: Customers are the root entity — nothing else can be created without them.

**Acceptance Criteria**:

1. WHEN a valid POST to `/api/v1/customers` with `firstName`, `lastName`, `email` THEN system SHALL create a Customer with a new UUID, persist it, and return `201` with the Customer DTO including `id`, `firstName`, `lastName`, `email`, `createdAt`.
2. WHEN creating a customer with an email that already exists THEN system SHALL return `409 Conflict`.
3. WHEN creating a customer with a missing `firstName`, `lastName`, or `email` THEN system SHALL return `400 Bad Request` with validation error details.
4. WHEN creating a customer with an invalid email format THEN system SHALL return `400 Bad Request`.
5. WHEN a GET to `/api/v1/customers/{customerId}` for an existing customer THEN system SHALL return `200` with the Customer DTO and a list of account summaries (id, accountNumber, status, currency) for all accounts owned by that customer.
6. WHEN a GET to `/api/v1/customers/{customerId}` for a non-existent customer THEN system SHALL return `404 Not Found`.
7. WHEN a POST includes an `X-Idempotency-Key` header and the same key is used again THEN system SHALL return `200` with the previously created Customer (no duplicate creation).

**Independent Test**: Start service, POST a customer, GET it back, verify 409 on duplicate email, verify 400 on bad input.

---

### P1.2: Account Creation ⭐ MVP

**User Story**: As a bank operator, I want to create accounts for customers so that customers can hold financial accounts in different currencies.

**Why P1**: Accounts are the core entity the entire ecosystem revolves around.

**Acceptance Criteria**:

1. WHEN a valid POST to `/api/v1/customers/{customerId}/accounts` with `currency` (ISO 4217) THEN system SHALL create an Account with status `PENDING`, auto-generate a 10-digit account number, persist the AccountStatusHistory row (null → PENDING), and return `201` with the Account DTO.
2. WHEN an optional `accountNumber` is provided in the request THEN system SHALL use it instead of auto-generating.
3. WHEN the customer does not exist THEN system SHALL return `404 Not Found`.
4. WHEN `currency` is missing or not a valid ISO 4217 code THEN system SHALL return `400 Bad Request`.
5. WHEN an account is created THEN system SHALL automatically create three default AccountLimit rows (TRANSACTION=10,000.00, DAILY=50,000.00, MONTHLY=200,000.00) in the account's currency.
6. WHEN account creation request includes an `X-Idempotency-Key` and the same key is used again THEN system SHALL return `200` with the previously created Account (no duplicate creation).

**Independent Test**: Create a customer, create an account under it, verify status is PENDING, verify account number is 10-digit, verify three default limits exist, verify 404 on unknown customer.

---

### P1.3: Account Lifecycle — Activate ⭐ MVP

**User Story**: As a bank operator, I want to activate a pending account so that it becomes ready for payment transactions.

**Why P1**: Accounts must be activated before any payment processing can occur.

**Acceptance Criteria**:

1. WHEN a POST to `/api/v1/accounts/{accountId}/activate` for a PENDING account with a valid customer (firstName, lastName, email present) THEN system SHALL transition the account to `ACTIVE`, persist an AccountStatusHistory row (PENDING → ACTIVE), and return `200` with the updated Account DTO.
2. WHEN the account is not in PENDING status (e.g., already ACTIVE, BLOCKED, or CLOSED) THEN system SHALL return `400 Bad Request`.
3. WHEN the account does not exist THEN system SHALL return `404 Not Found`.
4. WHEN the owning customer is missing required fields (firstName, lastName, email — not possible through normal API, edge case from data corruption) THEN system SHALL return `400 Bad Request` with a specific error message.
5. WHEN the activate request includes an `X-Idempotency-Key` THEN duplicate requests with the same key SHALL return `200` idempotently (no double transition).

**Independent Test**: Create customer + account, POST activate, verify status is ACTIVE, verify history row exists, verify 400 on already-active account.

---

### P1.4: Account Lifecycle — Block & Unblock ⭐ MVP

**User Story**: As a risk officer, I want to block and unblock accounts with auditable reasons so that suspicious or regulated accounts can be temporarily suspended.

**Why P1**: Blocking is a core compliance operation; the entire payment ecosystem depends on it.

**Acceptance Criteria**:

1. WHEN a POST to `/api/v1/accounts/{accountId}/block` with a valid `reason` (REGULATORY, FRAUD, CUSTOMER_REQUEST, or OTHER) for an ACTIVE account THEN system SHALL transition the account to `BLOCKED`, create an AccountBlockReason row (with blockedAt set, unblockedAt null), persist an AccountStatusHistory row (ACTIVE → BLOCKED), and return `200`.
2. WHEN the account is not ACTIVE THEN system SHALL return `400 Bad Request`.
3. WHEN the `reason` is missing or not a valid enum value THEN system SHALL return `400 Bad Request`.
4. WHEN a POST to `/api/v1/accounts/{accountId}/unblock` for a BLOCKED account THEN system SHALL transition the account to `ACTIVE`, set `unblockedAt` on ALL active AccountBlockReason rows, and persist an AccountStatusHistory row (BLOCKED → ACTIVE).
5. WHEN the account is not BLOCKED THEN system SHALL return `400 Bad Request`.
6. WHEN an account is BLOCKED with multiple active block reasons, unblocking SHALL resolve all of them (set unblockedAt on every active row).
7. WHEN block/unblock includes an `X-Idempotency-Key` THEN duplicate requests SHALL return `200` idempotently.

**Independent Test**: Create customer + account, activate it, block it (verify BLOCKED + block reason + history), unblock it (verify ACTIVE + history + unblockedAt set), verify 400 on blocking already-blocked account.

---

### P1.5: Account Lifecycle — Close ⭐ MVP

**User Story**: As a bank operator, I want to permanently close accounts so that inactive or terminated accounts cannot be used.

**Why P1**: Closure is a terminal operation needed for account lifecycle completion.

**Acceptance Criteria**:

1. WHEN a POST to `/api/v1/accounts/{accountId}/close` with a required `reason` string for any non-CLOSED account THEN system SHALL transition the account to `CLOSED` (terminal), persist an AccountStatusHistory row (current → CLOSED), and return `200`.
2. WHEN the account is already CLOSED THEN system SHALL return `400 Bad Request`.
3. WHEN the `reason` is missing or blank THEN system SHALL return `400 Bad Request`.
4. WHEN attempting to activate, block, unblock, or close a CLOSED account THEN system SHALL return `400 Bad Request` (terminal — no further transitions allowed).
5. WHEN close includes an `X-Idempotency-Key` THEN duplicate requests SHALL return `200` idempotently.

**Independent Test**: Close an active account, verify CLOSED + history, attempt to block/activate/unblock and verify 400, verify 400 on re-close.

---

### P1.6: Internal Payment Validation ⭐ MVP

**User Story**: As the payment-orchestrator service, I want to validate whether an account is eligible for payment execution so that I can gate payment processing.

**Why P1**: This is the integration point that payment-orchestrator depends on; without it, the payment saga cannot proceed.

**Acceptance Criteria**:

1. WHEN a GET to `/api/v1/internal/accounts/{accountId}/validate` for an ACTIVE account THEN system SHALL return `200` with `{"valid": true, "accountId": "...", "status": "ACTIVE"}`.
2. WHEN the account is BLOCKED THEN system SHALL return `200` with `{"valid": false, "accountId": "...", "status": "BLOCKED"}` and include all active block reasons.
3. WHEN the account is PENDING or CLOSED THEN system SHALL return `200` with `{"valid": false, "accountId": "...", "status": "PENDING" or "CLOSED"}`.
4. WHEN the account does not exist THEN system SHALL return `404 Not Found`.
5. This endpoint SHALL NOT require authentication (internal, trusted network).

**Independent Test**: Create customer + account, activate, call validate (expect valid=true), block it, call validate (expect valid=false with reasons), verify 404 on unknown account.

---

### P2.1: Account Limits CRUD

**User Story**: As a bank operator, I want to set transaction, daily, and monthly limits on accounts so that risk controls can be applied per account.

**Why P2**: Important for production readiness but not required for the core account lifecycle to function.

**Acceptance Criteria**:

1. WHEN a GET to `/api/v1/accounts/{accountId}/limits` THEN system SHALL return `200` with all limits configured for that account (including the three defaults created at account creation time).
2. WHEN a POST to `/api/v1/accounts/{accountId}/limits` with `limitType` (TRANSACTION, DAILY, MONTHLY), `amount` (positive BigDecimal), and `currency` (ISO 4217) THEN system SHALL create an additional AccountLimit or update the existing default limit and return `201`.
3. WHEN a PUT to `/api/v1/accounts/{accountId}/limits/{limitId}` with updated `amount` or `currency` THEN system SHALL update the limit and return `200`.
4. WHEN a DELETE to `/api/v1/accounts/{accountId}/limits/{limitId}` THEN system SHALL delete the limit and return `204`.
5. WHEN creating a limit with an invalid `limitType` or non-positive `amount` THEN system SHALL return `400 Bad Request`.
6. WHEN the account does not exist THEN system SHALL return `404 Not Found`.
7. All limit write endpoints SHALL support `X-Idempotency-Key`.

**Independent Test**: Create account, verify three default limits exist, add a custom limit, get all limits (verify default + custom present), update a limit amount, delete a limit, verify remaining.

---

### P2.2: Account Detail Enrichment

**User Story**: As a bank operator, I want to see the full account state including limits, block reasons, and recent status history when I retrieve account details.

**Why P2**: Provides operational visibility; the GET account endpoint should return everything needed for debugging.

**Acceptance Criteria**:

1. WHEN a GET to `/api/v1/accounts/{accountId}` THEN system SHALL include the account's current limits in the response.
2. WHEN a GET to `/api/v1/accounts/{accountId}` THEN system SHALL include active block reasons (those with unblockedAt == null).
3. WHEN a GET to `/api/v1/accounts/{accountId}` THEN system SHALL include the 10 most recent status history entries.
4. WHEN a GET to `/api/v1/customers/{customerId}` THEN system SHALL include the status of each account in the summary list.

**Independent Test**: Create account, add limit, block account, GET account — verify all enriched fields present.

---

### P3.1: Build & Infrastructure Foundation

**User Story**: As a developer, I want the service to build with a single Gradle command, run all migration-based tests, and start locally with proper configuration.

**Why P3**: Enables all other stories — the build system, migrations, and app config must exist before any code runs, but they're setup concerns, not customer-facing.

**Acceptance Criteria**:

1. WHEN `./gradlew build` from project root THEN all services SHALL compile and all tests SHALL pass.
2. WHEN `./gradlew :services:customer-account-service:test` THEN unit and integration tests SHALL run and pass.
3. WHEN the service starts, Flyway SHALL execute all migrations against PostgreSQL, creating all required tables.
4. WHEN the service starts, `/actuator/health` SHALL return `UP`.
5. WHEN the service starts, `/actuator/prometheus` SHALL expose metrics.
6. WHEN any request is received, structured logs SHALL include the `X-Correlation-ID` header value.
7. WHEN a request has no `X-Correlation-ID`, the service SHALL generate one and propagate it.

**Independent Test**: `./gradlew build` passes, service starts, migrations run, health endpoint responds, metrics endpoint responds, logs contain correlation IDs.

---

## Edge Cases

- WHEN an account is CLOSED (terminal) THEN all state-changing operations (activate, block, unblock, close) SHALL return `400 Bad Request`
- WHEN an account is PENDING THEN block and unblock operations SHALL return `400 Bad Request` (invalid transition)
- WHEN a request has whitespace-only strings for required fields THEN system SHALL treat them as missing and return `400 Bad Request`
- WHEN a request includes extra/unknown JSON fields THEN system SHALL ignore them (lenient parsing, no error)
- WHEN generating account numbers concurrently THEN system SHALL produce unique numbers without gaps or collisions
- WHEN a database constraint violation occurs (e.g., duplicate email race) THEN system SHALL return appropriate HTTP error (409 for duplicates, 500 for unexpected)
- WHEN a status transition is requested but the account's current status has been changed by a concurrent request THEN appropriate error handling SHALL occur (optimistic locking or atomic check-and-set)
- WHEN a GET request has a malformed UUID in the path THEN system SHALL return `400 Bad Request`
- WHEN idempotency key is reused with a different request body THEN system SHALL return `422 Unprocessable Entity`

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| --- | --- | --- | --- |
| CA-01 | P1.1: Customer Creation & Retrieval | Specify | Pending |
| CA-02 | P1.2: Account Creation | Specify | Pending |
| CA-03 | P1.3: Account Lifecycle — Activate | Specify | Pending |
| CA-04 | P1.4: Account Lifecycle — Block & Unblock | Specify | Pending |
| CA-05 | P1.5: Account Lifecycle — Close | Specify | Pending |
| CA-06 | P1.6: Internal Payment Validation | Specify | Pending |
| CA-07 | P2.1: Account Limits CRUD | Specify | Pending |
| CA-08 | P2.2: Account Detail Enrichment | Specify | Pending |
| CA-09 | P3.1: Build & Infrastructure Foundation | Specify | Pending |

**Coverage:** 9 total, 0 mapped to tasks, 9 unmapped

---

## Success Criteria

- [ ] `./gradlew build` passes with zero failures
- [ ] All 13 endpoints return correct status codes and response bodies per the API contract
- [ ] Account state machine rejects all invalid transitions with 400
- [ ] Idempotency keys prevent duplicate writes on all write endpoints
- [ ] Flyway migrations create all 5 entity tables with correct constraints
- [ ] 80%+ test coverage on domain and application layers
- [ ] Service starts and responds to health/metrics endpoints within 10 seconds
- [ ] All integration tests pass against a real PostgreSQL (Testcontainers)
