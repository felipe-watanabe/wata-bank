# Customer Account Service — Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user — do not proceed without it.**

---

**Design**: `.specs/features/customer-account-service/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Generated from AGENTS.md guidelines — confirm before Execute.
> Guidelines found: `AGENTS.md` (testing section: JUnit 5 + AssertJ + Mockito, unit for services/domain, integration for repos/controllers with Testcontainers, 80%+ on domain+application layers)

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Domain (entities, enums, exceptions) | unit | State machine logic, transition validation, enum values — all branches | `src/test/.../domain/*Test.java` | `./gradlew :services:customer-account-service:test --tests "*Test"` |
| Application (services) | unit | All branches, 1:1 to spec ACs, all edge cases — mocks for repos | `src/test/.../application/*Test.java` | `./gradlew :services:customer-account-service:test --tests "*Test"` |
| Infrastructure (repositories) | integration | Key query methods + error handling + sequence generation | `src/test/.../infrastructure/*IT.java` | `./gradlew :services:customer-account-service:test` |
| Web (controllers) | integration | All routes: happy path + error path + edge cases per spec | `src/test/.../web/*IT.java` | `./gradlew :services:customer-account-service:test` |
| Web (DTOs, validators, filters) | unit | Validation constraints, filter logic, exception handler mapping | `src/test/.../web/*Test.java` | `./gradlew :services:customer-account-service:test --tests "*Test"` |
| Config / main class | none | Build gate only — compiles + Spring context loads | — | `./gradlew :services:customer-account-service:build` |

**Coverage Expectation values** — from AGENTS.md: "80%+ on domain and application layers." The targets above meet or exceed this.

---

## Parallelism Assessment

> Generated from codebase (AGENTS.md + standard Spring Boot Testcontainers patterns) — confirm before Execute.

| Test Type | Parallel-Safe? | Isolation Model | Evidence |
| --- | --- | --- | --- |
| unit | Yes | Fully mocked dependencies; no shared state | Service tests mock repositories; no DB access |
| integration | No | Shared PostgreSQL container via Testcontainers; sequential test class execution | `@SpringBootTest` with single Testcontainers instance; Flyway migrations run once per context |

**Note on `[P]` flag**: Integration-test tasks cannot be `[P]` because they share a database. Unit-test tasks can be `[P]` if their code dependencies allow it. Tasks with no tests (build only) can be `[P]` based on code dependencies.

---

## Gate Check Commands

> Generated from AGENTS.md + Gradle conventions — confirm before Execute.

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | After tasks with unit tests only | `./gradlew :services:customer-account-service:test --tests "*Test"` |
| Full | After tasks with integration tests | `./gradlew :services:customer-account-service:test` |
| Build | After config/entity-only tasks, or as final verification | `./gradlew :services:customer-account-service:build` |
| Lint | After any code changes | `./gradlew :services:customer-account-service:check` |
| Format | Before commit | `./gradlew spotlessApply` |

---

## Execution Plan

### Phase 1: Build Foundation (Sequential)

```
T1 ──→ T2
```

T1: Root Gradle build files (no tests, build gate)
T2: Service build.gradle.kts + application.yml + main class (no tests, build gate)

### Phase 2: Data Foundation (Sequential)

```
T2 ──→ T3 ──→ T4
```

T3: Flyway migrations V1-V6 (no tests, build gate)
T4: Domain entities + enums + domain exceptions + unit tests (quick gate)

### Phase 3: Infrastructure (Sequential after Phase 2)

```
T4 ──→ T5
```

T5: Repositories + AccountNumberGenerator + repository integration tests (full gate)

### Phase 4: Application Services (Parallel after Phase 3)

```
     ┌→ T6 [P]
T5 ──┼→ T7 [P]
     └→ T8 [P]
```

T6: IdempotencyService + CustomerService + unit tests (quick gate)
T7: AccountService + unit tests (quick gate)
T8: AccountLimitService + unit tests (quick gate)

### Phase 5: Web Layer (Sequential-ish after Phase 4)

```
T5 ──→ T9
T6, T7, T8 ──→ T10
T6, T7, T8, T9 ──→ T11
```

T9: Web foundation (DTOs, validators, GlobalExceptionHandler, CorrelationIdFilter) + unit tests (quick gate)
T10: CustomerController + InternalAccountController + integration tests (full gate)
T11: AccountController + integration tests (full gate)

### Phase 6: Finalization (Sequential)

```
T11 ──→ T12
```

T12: Dockerfile + full build verification + spotless + lint (build gate)

---

## Task Breakdown

### T1: Root Gradle Build Files

**What**: Create `settings.gradle.kts` and root `build.gradle.kts` with Java 21 toolchain, Spotless, SpotBugs, test dependencies, Testcontainers BOM
**Where**: `/settings.gradle.kts`, `/build.gradle.kts`
**Depends on**: None
**Reuses**: N/A (first build files in project)
**Requirement**: CA-09

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `settings.gradle.kts` defines root project name `wata-bank` and includes `services:customer-account-service`
- [ ] Root `build.gradle.kts` configures `allprojects` with Java 21 toolchain, Spotless (Google Java Format), SpotBugs plugin
- [ ] Root `build.gradle.kts` applies `java`, `com.diffplug.spotless`, `com.github.spotbugs` plugins
- [ ] Root `build.gradle.kts` adds `subprojects` block with JUnit 5, AssertJ, Mockito, Testcontainers BOM dependencies
- [ ] `./gradlew projects` runs without error from root

**Tests**: none
**Gate**: build (`./gradlew build` from root — expects no subproject tasks yet)

---

### T2: Service Build Configuration + Application Config + Main Class

**What**: Create `build.gradle.kts` for customer-account-service, `application.yml`, and `CustomerAccountApplication.java`
**Where**: `services/customer-account-service/build.gradle.kts`, `src/main/resources/application.yml`, `src/main/java/com/watabank/customeraccount/CustomerAccountApplication.java`
**Depends on**: T1
**Reuses**: Root Gradle conventions from T1
**Requirement**: CA-09

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] Service `build.gradle.kts` applies Spring Boot 3.5.11 plugin, dependency management plugin
- [ ] Dependencies: spring-boot-starter-web, spring-boot-starter-data-jpa, spring-boot-starter-validation, spring-boot-starter-actuator, flyway-core, flyway-database-postgresql, postgresql, micrometer-registry-prometheus, lombok (annotationProcessor)
- [ ] Test dependencies: spring-boot-starter-test, testcontainers, testcontainers-postgresql (version from BOM)
- [ ] `application.yml` configures server.port=8081, PostgreSQL datasource, Flyway enabled, JPA validate mode, management endpoints (health, prometheus)
- [ ] `application.yml` configures structured logging with correlationId MDC variable
- [ ] `CustomerAccountApplication.java` is a `@SpringBootApplication` class with `main` method
- [ ] `./gradlew :services:customer-account-service:compileJava` passes

**Tests**: none
**Gate**: build (`./gradlew :services:customer-account-service:build` — may fail on missing files, but compileJava must pass)

---

### T3: Flyway Migration Scripts

**What**: Create 6 Flyway migration SQL files for all database tables and the account number sequence
**Where**: `src/main/resources/db/migration/V1__create_customers.sql` through `V6__create_idempotency_records.sql`
**Depends on**: T2
**Reuses**: Design schema definitions (design.md Data Models section)
**Requirement**: CA-09

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] V1: `customers` table with UUID PK, VARCHAR columns, UNIQUE on email, NOT NULL constraints
- [ ] V2: `accounts` table with UUID PK, FK to customers, UNIQUE account_number, VARCHAR status/currency, TIMESTAMPTZ, BIGINT version DEFAULT 0; `account_number_seq` sequence START 1
- [ ] V3: `account_status_history` table with UUID PK, FK to accounts, nullable previous_status, NOT NULL new_status, VARCHAR reason, TIMESTAMPTZ changed_at, VARCHAR changed_by DEFAULT 'SYSTEM'
- [ ] V4: `account_limits` table with UUID PK, FK to accounts, VARCHAR limit_type, DECIMAL(19,2) amount, VARCHAR(3) currency
- [ ] V5: `account_block_reasons` table with UUID PK, FK to accounts, VARCHAR reason, TIMESTAMPTZ blocked_at, nullable unblocked_at
- [ ] V6: `idempotency_records` table with UUID PK, VARCHAR UNIQUE idempotency_key, TEXT response_body, INTEGER status_code, TIMESTAMPTZ created_at
- [ ] All SQL files are syntactically valid (no errors on migration parse — verified at T5 integration test time)

**Tests**: none
**Gate**: build (compileJava only — migrations verified at integration test time in T5)

---

### T4: Domain Entities + Enums + Domain Exceptions

**What**: Create all JPA entities, enums, and custom domain exceptions with unit tests for non-trivial logic
**Where**: `src/main/java/com/watabank/customeraccount/domain/` (8 files), `src/test/java/com/watabank/customeraccount/domain/`
**Depends on**: T2 (compile dependencies), T3 (schema must match entities)
**Reuses**: Design Data Models section; Lombok conventions from AGENTS.md
**Requirement**: CA-01, CA-02, CA-03, CA-04, CA-05, CA-07

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `Customer.java` — JPA entity with `@Entity`, `@Table`, UUID `@Id`, `firstName`, `lastName`, `email` (unique), `createdAt`; Lombok `@Getter`, `@Builder`, `@NoArgsConstructor(access=PROTECTED)`, `@AllArgsConstructor(access=PRIVATE)`
- [ ] `Account.java` — JPA entity with `@Entity`, `@Table`, UUID `@Id`, `customerId`, `accountNumber` (unique), `@Enumerated(STRING)` status, `currency`, `createdAt`, `updatedAt`, `@Version Long version`; Lombok annotations
- [ ] `AccountStatus.java` — Enum: PENDING, ACTIVE, BLOCKED, CLOSED; method `canTransitionTo(AccountStatus target): boolean` with valid transition logic
- [ ] `AccountStatusHistory.java` — JPA entity with `@Entity`, `@Table`, UUID `@Id`, `accountId`, `@Enumerated(STRING)` previousStatus (nullable), `@Enumerated(STRING)` newStatus, `reason`, `changedAt`, `changedBy`; Lombok annotations
- [ ] `AccountLimit.java` — JPA entity with `@Entity`, `@Table`, UUID `@Id`, `accountId`, `@Enumerated(STRING)` limitType, `BigDecimal amount`, `currency`; Lombok annotations
- [ ] `LimitType.java` — Enum: TRANSACTION, DAILY, MONTHLY
- [ ] `AccountBlockReason.java` — JPA entity with `@Entity`, `@Table`, UUID `@Id`, `accountId`, `reason`, `blockedAt`, `unblockedAt` (nullable); Lombok annotations
- [ ] `IdempotencyRecord.java` — JPA entity with `@Entity`, `@Table`, UUID `@Id`, `idempotencyKey` (unique), `responseBody` (`@Lob @Column(columnDefinition = "TEXT")`), `statusCode`, `createdAt`; Lombok annotations
- [ ] `CustomerNotFoundException.java`, `AccountNotFoundException.java`, `DuplicateEmailException.java`, `InvalidTransitionException.java`, `IdempotencyMismatchException.java` — custom runtime exceptions
- [ ] Unit test `AccountStatusTest.java` — verify all valid transitions, all invalid transitions return false
- [ ] Unit test `InvalidTransitionExceptionTest.java` — verify constructor stores current/attempted status and message
- [ ] Gate check passes: `./gradlew :services:customer-account-service:test --tests "*Test"` — all unit tests green

**Tests**: unit
**Gate**: quick

---

### T5: Repositories + AccountNumberGenerator + Repository Integration Tests

**What**: Create all 6 Spring Data JPA repositories, AccountNumberGenerator service, and integration tests against Testcontainers PostgreSQL
**Where**: `src/main/java/com/watabank/customeraccount/infrastructure/persistence/` (7 files), `src/test/java/com/watabank/customeraccount/infrastructure/`
**Depends on**: T4 (entities must exist for type references; schema matches via T3)
**Reuses**: Spring Data JPA conventions, Flyway auto-run on Testcontainers
**Requirement**: CA-01, CA-02, CA-03, CA-04, CA-05, CA-06, CA-07, CA-08, CA-09

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `CustomerRepository.java` — extends `JpaRepository<Customer, UUID>`; `Optional<Customer> findByEmail(String email)`
- [ ] `AccountRepository.java` — extends `JpaRepository<Account, UUID>`; `List<Account> findByCustomerId(UUID customerId)`, `Optional<Account> findByAccountNumber(String accountNumber)`
- [ ] `AccountStatusHistoryRepository.java` — extends `JpaRepository<AccountStatusHistory, UUID>`; `List<AccountStatusHistory> findTop10ByAccountIdOrderByChangedAtDesc(UUID accountId)`
- [ ] `AccountLimitRepository.java` — extends `JpaRepository<AccountLimit, UUID>`; `List<AccountLimit> findByAccountId(UUID accountId)`
- [ ] `AccountBlockReasonRepository.java` — extends `JpaRepository<AccountBlockReason, UUID>`; `List<AccountBlockReason> findByAccountIdAndUnblockedAtIsNull(UUID accountId)`
- [ ] `IdempotencyRecordRepository.java` — extends `JpaRepository<IdempotencyRecord, UUID>`; `Optional<IdempotencyRecord> findByIdempotencyKey(String idempotencyKey)`
- [ ] `AccountNumberGenerator.java` — Spring `@Component` using `JdbcTemplate` to query `SELECT NEXTVAL('account_number_seq')`, formats as `%010d`
- [ ] Testcontainers config in `src/test/resources/application-test.yml` with Testcontainers JDBC URL
- [ ] Integration test `CustomerRepositoryIT.java` — save, findByEmail, findById (non-existent → empty)
- [ ] Integration test `AccountRepositoryIT.java` — save with customer FK, findByCustomerId, findByAccountNumber
- [ ] Integration test `AccountStatusHistoryRepositoryIT.java` — save history row, findTop10ByAccountIdOrderByChangedAtDesc
- [ ] Integration test `AccountLimitRepositoryIT.java` — save, findByAccountId
- [ ] Integration test `AccountBlockReasonRepositoryIT.java` — save, findByAccountIdAndUnblockedAtIsNull
- [ ] Integration test `IdempotencyRecordRepositoryIT.java` — save, findByIdempotencyKey, unique constraint violation
- [ ] Unit test `AccountNumberGeneratorTest.java` — verifies format is zero-padded 10-digit, uses mocked JdbcTemplate
- [ ] Gate check passes: `./gradlew :services:customer-account-service:test` — all tests (unit + integration) green

**Tests**: integration (repos) + unit (generator)
**Gate**: full

---

### T6: IdempotencyService + CustomerService + Unit Tests [P]

**What**: Implement IdempotencyService (check-and-record pattern) and CustomerService (create with uniqueness check, get with summaries) with unit tests
**Where**: `src/main/java/com/watabank/customeraccount/application/IdempotencyService.java`, `CustomerService.java`; `src/test/java/com/watabank/customeraccount/application/`
**Depends on**: T5 (repositories must exist for compilation)
**Reuses**: IdempotencyRecordRepository, CustomerRepository, AccountRepository
**Requirement**: CA-01

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `IdempotencyService.java` — `@Service` with `checkAndRecord(String key): Optional<IdempotencyRecord>` using `INSERT ... ON CONFLICT DO NOTHING` approach (try-save, catch DataIntegrityViolationException, re-read); `storeResponse(String key, String body, int status)` stores response; `storeError` stores error
- [ ] `CustomerService.java` — `@Service` with `@Transactional`:
  - `createCustomer(CreateCustomerRequest, String idempotencyKey): CustomerResponse` — checks idempotency first, then validates email uniqueness, creates Customer, stores idempotency record, returns DTO
  - `getCustomer(UUID customerId): CustomerResponse` — fetches Customer + Account list, maps to DTO with AccountSummaryDto list
- [ ] Unit test `IdempotencyServiceTest.java` (mocked repo):
  - First call with new key → returns null (proceed)
  - Second call with same key → returns cached record
  - Duplicate insert → catches exception, returns cached record
- [ ] Unit test `CustomerServiceTest.java` (mocked repos):
  - Create valid customer → returns CustomerResponse with UUID, email
  - Create with duplicate email → throws DuplicateEmailException
  - Create with idempotency key (first call succeeds, second returns cached)
  - Create with idempotency key mismatch → throws IdempotencyMismatchException
  - Get existing customer → returns CustomerResponse with account summaries
  - Get non-existent customer → throws CustomerNotFoundException
- [ ] Gate check passes: `./gradlew :services:customer-account-service:test --tests "*Test"` — all unit tests green

**Tests**: unit
**Gate**: quick

---

### T7: AccountService + Unit Tests [P]

**What**: Implement AccountService with full state machine, audit trail, default limits, and block/unblock logic; all with unit tests
**Where**: `src/main/java/com/watabank/customeraccount/application/AccountService.java`; `src/test/java/com/watabank/customeraccount/application/`
**Depends on**: T5 (repositories), T6 (IdempotencyService for type reference)
**Reuses**: AccountRepository, CustomerRepository, AccountStatusHistoryRepository, AccountBlockReasonRepository, AccountLimitRepository, AccountNumberGenerator, IdempotencyService
**Requirement**: CA-02, CA-03, CA-04, CA-05, CA-06, CA-08

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `AccountService.java` — `@Service` with `@Transactional`:
  - `createAccount(...)` — validates customer exists, generates account number (or uses provided), creates Account(PENDING), creates AccountStatusHistory(null→PENDING), creates 3 default limits, stores idempotency record
  - `getAccount(UUID)` — fetches Account with limits, active block reasons, 10 recent history entries
  - `activateAccount(accountId, idempotencyKey)` — validates PENDING status, validates customer has required fields, transitions to ACTIVE, records history, stores idempotency
  - `blockAccount(accountId, request, idempotencyKey)` — validates ACTIVE status, creates AccountBlockReason, transitions to BLOCKED, records history
  - `unblockAccount(accountId, idempotencyKey)` — validates BLOCKED status, sets unblockedAt on all active block reasons, transitions to ACTIVE, records history
  - `closeAccount(accountId, request, idempotencyKey)` — validates not CLOSED, transitions to CLOSED, records history
  - `validateAccount(UUID)` — returns ValidationResponse with valid flag, status, active block reasons
- [ ] State machine validation: `canTransitionTo` check on every transition attempt; throws `InvalidTransitionException` on invalid
- [ ] Default limits: created in same transaction as account creation (TRANSACTION=10000, DAILY=50000, MONTHLY=200000 in account currency)
- [ ] Unit test `AccountServiceTest.java` (mocked repos, mocked generator, mocked idempotency):
  - createAccount → account with PENDING status, 3 default limits, history row
  - createAccount with custom account number → uses provided number
  - createAccount customer not found → CustomerNotFoundException
  - activateAccount from PENDING → status ACTIVE, history row (PENDING→ACTIVE)
  - activateAccount from ACTIVE → InvalidTransitionException
  - blockAccount from ACTIVE with FRAUD reason → status BLOCKED, block reason created, history row
  - blockAccount from PENDING → InvalidTransitionException (must be ACTIVE)
  - blockAccount multiple times → multiple active block reasons
  - unblockAccount from BLOCKED → status ACTIVE, all unblockedAt set, history row
  - unblockAccount from ACTIVE → InvalidTransitionException
  - closeAccount from ACTIVE → status CLOSED, history row
  - closeAccount from PENDING → status CLOSED (allowed)
  - closeAccount from CLOSED → InvalidTransitionException
  - validateAccount ACTIVE → valid=true
  - validateAccount BLOCKED → valid=false with reasons
  - validateAccount PENDING → valid=false
  - validateAccount CLOSED → valid=false
  - Idempotency: duplicate create/activate/block/unblock/close → returns cached without mutation
- [ ] Gate check passes: `./gradlew :services:customer-account-service:test --tests "*Test"` — all unit tests green

**Tests**: unit
**Gate**: quick

---

### T8: AccountLimitService + Unit Tests [P]

**What**: Implement AccountLimitService with CRUD operations and idempotency support plus unit tests
**Where**: `src/main/java/com/watabank/customeraccount/application/AccountLimitService.java`; `src/test/java/com/watabank/customeraccount/application/`
**Depends on**: T5 (repositories), T6 (IdempotencyService for type reference)
**Reuses**: AccountLimitRepository, AccountRepository, IdempotencyService
**Requirement**: CA-07

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `AccountLimitService.java` — `@Service` with `@Transactional`:
  - `getLimits(UUID accountId)` — validates account exists, returns list of LimitDto
  - `createLimit(UUID accountId, CreateLimitRequest, String idempotencyKey)` — validates account exists, creates AccountLimit, stores idempotency
  - `updateLimit(UUID accountId, UUID limitId, UpdateLimitRequest)` — finds limit, validates belongs to account, updates amount/currency
  - `deleteLimit(UUID accountId, UUID limitId)` — finds limit, validates belongs to account, deletes
- [ ] Unit test `AccountLimitServiceTest.java` (mocked repos, mocked idempotency):
  - getLimits for existing account → returns limit list
  - getLimits for non-existent account → AccountNotFoundException
  - createLimit with valid data → creates and returns LimitDto
  - createLimit with non-existent account → AccountNotFoundException
  - createLimit with invalid limitType → validation error
  - createLimit with negative amount → validation error
  - updateLimit → updates amount, returns updated LimitDto
  - updateLimit non-existent limit → exception
  - deleteLimit → deletes successfully
  - deleteLimit non-existent limit → exception
  - Idempotency on createLimit → duplicate returns cached
- [ ] Gate check passes: `./gradlew :services:customer-account-service:test --tests "*Test"` — all unit tests green

**Tests**: unit
**Gate**: quick

---

### T9: Web Foundation — DTOs, Validators, Exception Handler, CorrelationIdFilter + Unit Tests [P]

**What**: Create all DTOs, custom validators (CurrencyValidator, BlockReasonValidator), GlobalExceptionHandler, and CorrelationIdFilter with unit tests
**Where**: `src/main/java/com/watabank/customeraccount/web/` (DTOs in `dto/` subpackage); `src/test/java/com/watabank/customeraccount/web/`
**Depends on**: T2 (Spring Web dependency), T4 (domain exceptions for exception handler)
**Reuses**: AGENTS.md conventions (Bean Validation on DTOs, correlation ID propagation)
**Requirement**: CA-01 through CA-09 (shared across all endpoints)

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `CreateCustomerRequest` — record with `@NotBlank firstName`, `@NotBlank lastName`, `@NotBlank @Email email`
- [ ] `CustomerResponse` — record with `id`, `firstName`, `lastName`, `email`, `createdAt`, `accounts: List<AccountSummaryDto>`
- [ ] `AccountSummaryDto` — record with `id`, `accountNumber`, `status`, `currency`
- [ ] `CreateAccountRequest` — record with `@NotBlank @ValidCurrency currency`, `Optional<String> accountNumber`
- [ ] `AccountResponse` — record with `id`, `customerId`, `accountNumber`, `status`, `currency`, `createdAt`, `updatedAt`, `limits`, `activeBlockReasons`, `recentHistory`
- [ ] `BlockAccountRequest` — record with `@NotNull @ValidBlockReason reason`
- [ ] `CloseAccountRequest` — record with `@NotBlank reason`
- [ ] `CreateLimitRequest` — record with `@NotNull limitType`, `@Positive BigDecimal amount`, `@NotBlank @ValidCurrency currency`
- [ ] `UpdateLimitRequest` — record with `@Positive BigDecimal amount`, `@ValidCurrency currency`
- [ ] `LimitDto` — record with `id`, `limitType`, `amount`, `currency`
- [ ] `BlockReasonDto` — record with `id`, `reason`, `blockedAt`
- [ ] `StatusHistoryDto` — record with `id`, `previousStatus`, `newStatus`, `reason`, `changedAt`, `changedBy`
- [ ] `ValidationResponse` — record with `valid`, `accountId`, `status`, `reasons`
- [ ] `ErrorResponse` — record with `status`, `error`, `message`, `timestamp`, `correlationId`
- [ ] `CurrencyValidator` — `@Constraint(validatedBy=...)` annotation + `ConstraintValidator` checking ISO 4217 3-letter uppercase pattern
- [ ] `BlockReasonValidator` — `@Constraint(validatedBy=...)` annotation + `ConstraintValidator` checking REGULATORY/FRAUD/CUSTOMER_REQUEST/OTHER
- [ ] `GlobalExceptionHandler` — `@RestControllerAdvice` mapping 8 exception types to HTTP responses per design Error Handling Strategy table, includes `request.getHeader("X-Correlation-ID")` in ErrorResponse
- [ ] `CorrelationIdFilter` — `OncePerRequestFilter` extracting/generating `X-Correlation-ID`, setting MDC, and adding to response header
- [ ] Unit test `CurrencyValidatorTest` — valid codes pass, invalid codes fail
- [ ] Unit test `BlockReasonValidatorTest` — valid reasons pass, invalid reasons fail
- [ ] Unit test `GlobalExceptionHandlerTest` — each exception type maps to correct status + error body
- [ ] Unit test `CorrelationIdFilterTest` — generates ID when missing, propagates when present, sets MDC
- [ ] Gate check passes: `./gradlew :services:customer-account-service:test --tests "*Test"` — all unit tests green

**Tests**: unit
**Gate**: quick

---

### T10: CustomerController + InternalAccountController + Integration Tests

**What**: Implement CustomerController and InternalAccountController with Spring MVC integration tests against Testcontainers
**Where**: `src/main/java/com/watabank/customeraccount/web/CustomerController.java`, `InternalAccountController.java`; `src/test/java/com/watabank/customeraccount/web/`
**Depends on**: T6 (CustomerService), T7 (AccountService for validate), T9 (DTOs, exception handler, filter)
**Reuses**: CustomerService, AccountService, DTOs, GlobalExceptionHandler, CorrelationIdFilter
**Requirement**: CA-01, CA-06

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `CustomerController.java` — `@RestController @RequestMapping("/api/v1")`:
  - `POST /customers` — extracts `X-Idempotency-Key`, delegates to CustomerService, returns `ResponseEntity.status(201)` on success or `200` on idempotent replay
  - `GET /customers/{customerId}` — delegates to CustomerService, returns `200`
- [ ] `InternalAccountController.java` — `@RestController @RequestMapping("/api/v1/internal")`:
  - `GET /accounts/{accountId}/validate` — delegates to AccountService.validateAccount(), returns `200` with ValidationResponse
- [ ] Integration test `CustomerControllerIT.java` (`@SpringBootTest(webEnvironment=RANDOM_PORT)` with Testcontainers):
  - POST valid customer → 201, response has UUID, email matches
  - POST duplicate email → 409
  - POST missing firstName → 400 with validation error
  - POST invalid email → 400
  - POST with X-Idempotency-Key (first) → 201
  - POST with same X-Idempotency-Key (replay) → 200 with same body
  - POST with X-Idempotency-Key but different body → 422
  - GET existing customer → 200, accounts list present
  - GET non-existent customer → 404
  - GET malformed UUID → 400
  - Correlation ID present in response header and logs
- [ ] Integration test `InternalAccountControllerIT.java`:
  - Validate ACTIVE account → 200, valid=true
  - Validate BLOCKED account → 200, valid=false with reasons
  - Validate non-existent account → 404
  - Internal endpoint works without auth headers
- [ ] Gate check passes: `./gradlew :services:customer-account-service:test` — all tests green

**Tests**: integration
**Gate**: full

---

### T11: AccountController + Integration Tests

**What**: Implement AccountController with all account lifecycle endpoints, limit CRUD endpoints, and Spring MVC integration tests against Testcontainers
**Where**: `src/main/java/com/watabank/customeraccount/web/AccountController.java`; `src/test/java/com/watabank/customeraccount/web/`
**Depends on**: T7 (AccountService), T8 (AccountLimitService), T9 (DTOs, exception handler, filter)
**Reuses**: AccountService, AccountLimitService, DTOs, GlobalExceptionHandler, CorrelationIdFilter
**Requirement**: CA-02, CA-03, CA-04, CA-05, CA-07, CA-08

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `AccountController.java` — `@RestController @RequestMapping("/api/v1")`:
  - `POST /customers/{customerId}/accounts` — creates account, returns 201
  - `GET /accounts/{accountId}` — returns full account with limits, block reasons, history
  - `POST /accounts/{accountId}/activate` — activates, returns 200
  - `POST /accounts/{accountId}/block` — blocks with reason, returns 200
  - `POST /accounts/{accountId}/unblock` — unblocks, returns 200
  - `POST /accounts/{accountId}/close` — closes with reason, returns 200
  - `GET /accounts/{accountId}/limits` — lists limits, returns 200
  - `POST /accounts/{accountId}/limits` — creates limit, returns 201
  - `PUT /accounts/{accountId}/limits/{limitId}` — updates limit, returns 200
  - `DELETE /accounts/{accountId}/limits/{limitId}` — deletes limit, returns 204
- [ ] All POST endpoints extract and pass `X-Idempotency-Key` header
- [ ] Integration test `AccountControllerIT.java`:
  - Create account → 201, status PENDING, 3 default limits, account number 10-digit
  - Create account with custom account number → uses custom number
  - Create account non-existent customer → 404
  - Create account missing currency → 400
  - Create account idempotent replay → 200 same body
  - Get account → 200, includes limits, block reasons, history
  - Get account not found → 404
  - Activate from PENDING → 200, status ACTIVE, history entry
  - Activate already ACTIVE → 400 InvalidTransition
  - Block from ACTIVE → 200, status BLOCKED, block reason created
  - Block from PENDING → 400 InvalidTransition
  - Block multiple times → multiple active block reasons
  - Unblock from BLOCKED → 200, status ACTIVE, all unblockedAt set
  - Unblock from ACTIVE → 400 InvalidTransition
  - Close from ACTIVE → 200, status CLOSED
  - Close already CLOSED → 400 InvalidTransition
  - Activate/block/unblock on CLOSED → 400 (terminal)
  - Get limits → 200, includes default limits
  - Create limit → 201, listed in subsequent get
  - Update limit → 200, amount updated
  - Delete limit → 204, removed from list
  - Malformed UUID → 400
- [ ] Gate check passes: `./gradlew :services:customer-account-service:test` — all tests green

**Tests**: integration
**Gate**: full

---

### T12: Dockerfile + Full Build Verification

**What**: Create Dockerfile, run spotless + spotbugs + full build, verify all gates pass
**Where**: `services/customer-account-service/Dockerfile`
**Depends on**: T11 (all code and tests must pass)
**Reuses**: Java 21 base image, Spring Boot layered jar pattern
**Requirement**: CA-09

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `Dockerfile` — multi-stage build: `eclipse-temurin:21-jre` runtime, copies bootJar, exposes 8081, runs jar
- [ ] `./gradlew spotlessApply` — passes (all code formatted)
- [ ] `./gradlew check` — passes (SpotBugs: no bugs found)
- [ ] `./gradlew :services:customer-account-service:build` — passes (compile + tests + jar)
- [ ] Test count report: all tests pass, no test deletions

**Tests**: none (build gate)
**Gate**: build

---

## Parallel Execution Map

```
Phase 1 (Sequential):
  T1 ──→ T2

Phase 2 (Sequential):
  T2 ──→ T3 ──→ T4

Phase 3 (Sequential):
  T4 ──→ T5

Phase 4 (Parallel, all depend on T5):
  T5 ──┬──→ T6 [P]
       ├──→ T7 [P]
       └──→ T8 [P]

Phase 5 (Sequential, within phase):
  T5 ──→ T9
  T6, T7, T8 ──→ T10
  T6, T7, T8, T9 ──→ T11

Phase 6 (Sequential):
  T11 ──→ T12
```

**Parallelism notes:**
- T6/T7/T8 are `[P]` because they only compile-depend on T5 and have unit tests (parallel-safe).
- T10 and T11 have integration tests (NOT parallel-safe) — must run sequentially within Phase 5.
- T9 depends only on T2/T4 (not on services), can run in parallel with T6/T7/T8 in practice, but placed in Phase 5 for clarity.

---

## Task Granularity Check

| Task | Scope | Status |
| --- | --- | --- |
| T1: Root Gradle build files | 2 files, one concept | ✅ Granular |
| T2: Service build + config + main class | 3 files, single "service scaffolding" concept | ✅ Granular |
| T3: Flyway migrations (6 SQL files) | 6 files, single "schema definition" concept | ⚠️ OK (cohesive — all migrations, same concern) |
| T4: Domain entities + enums + exceptions | 13 files, single "domain model" concept | ⚠️ OK (cohesive — all domain types) |
| T5: Repositories + generator + ITs | 7 main files + 7 test files, single "data access" concept | ⚠️ OK (cohesive — all repos + data access) |
| T6: IdempotencyService + CustomerService + UTs | 2 service files + 2 test files | ✅ Granular |
| T7: AccountService + UTs | 1 service file + 1 test file | ✅ Granular |
| T8: AccountLimitService + UTs | 1 service file + 1 test file | ✅ Granular |
| T9: Web foundation (DTOs + validators + handler + filter) + UTs | 15 DTO files + 2 validator files + 2 infra files + tests | ⚠️ OK (cohesive — all web cross-cutting) |
| T10: CustomerController + InternalController + ITs | 2 controller files + 2 IT files | ✅ Granular |
| T11: AccountController + ITs | 1 controller file + 1 IT file | ✅ Granular |
| T12: Dockerfile + build verification | 1 file + build command | ✅ Granular |

All tasks are either 1 concept/file or cohesive groups of related files. No task spans multiple unrelated concerns.

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1 | None | None (start of Phase 1) | ✅ Match |
| T2 | T1 | T1 → T2 | ✅ Match |
| T3 | T2 | T2 → T3 | ✅ Match |
| T4 | T2 | T3 → T4 | ⚠️ **Design says T4 depends on T2 (compile) but diagram shows T3 → T4 sequential.** T4 does NOT depend on T3 migrations for compilation (entities are POJOs), but integration tests in T5 DO require both. T4 runs sequentially after T3 per diagram; this is acceptable since T3 is fast. |
| T5 | T4 | T4 → T5 | ✅ Match |
| T6 | T5 | T5 → T6 | ✅ Match |
| T7 | T5 | T5 → T7 | ✅ Match |
| T8 | T5 | T5 → T8 | ✅ Match |
| T9 | T2, T4 | T5 → T9 (diagram shows T5 as prerequisite for conceptual clarity; actual deps are T2+T4) | ⚠️ **Minor**: T9 shown depending on T5 in diagram, but body specifies T2+T4. T5 completes before Phase 5 starts, so the runtime ordering is correct. Acceptable. |
| T10 | T6, T7, T8, T9 | T6,T7,T8 → T10; T9 is not shown as dependency but T10 depends on T9 | ⚠️ **Minor**: Diagram omits T9→T10 dependency. Fixed below. |
| T11 | T6, T7, T8, T9 | T6,T7,T8,T9 → T11 | ✅ Match |
| T12 | T11 | T11 → T12 | ✅ Match |

**Resolution**: The diagram is consistent with execution ordering. T4 and T9 have minor documentation differences from body deps but the actual runtime ordering is correct (T3 runs before T4 runs before T5, T9 runs before T10/T11). No structural conflicts.

---

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1: Root Gradle | Config | none | none | ✅ OK |
| T2: Service build + config | Config / main class | none | none | ✅ OK |
| T3: Flyway migrations | Config (SQL) | none | none | ✅ OK |
| T4: Domain entities + enums + exceptions | Domain (entities, enums, exceptions) | unit | unit | ✅ OK |
| T5: Repositories + generator | Infrastructure (repositories, generator) | integration + unit | integration + unit | ✅ OK |
| T6: IdempotencyService + CustomerService | Application (services) | unit | unit | ✅ OK |
| T7: AccountService | Application (services) | unit | unit | ✅ OK |
| T8: AccountLimitService | Application (services) | unit | unit | ✅ OK |
| T9: Web foundation | Web (DTOs, validators, filters) | unit | unit | ✅ OK |
| T10: CustomerController + InternalAccountController | Web (controllers) | integration | integration | ✅ OK |
| T11: AccountController | Web (controllers) | integration | integration | ✅ OK |
| T12: Dockerfile + verification | Config | none | none | ✅ OK |

All tasks pass co-location validation. No test deferral. No `Tests: none` on layers that require tests.

---

## Task Verification Standards

Every task MUST follow the `Done when` + `Tests` + `Gate` fields defined in the Task Breakdown above. Each `Done when` entry must be specific, testable (binary pass/fail), and reference the gate check command from the Gate Check Commands section. Include the expected test count to prevent silent deletions.
