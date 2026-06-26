# Customer Account Service — Domain Model

## Entities

### Customer

Central entity representing a customer in the system.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `firstName` | String | Customer first name |
| `lastName` | String | Customer last name |
| `email` | String | Unique email address |
| `createdAt` | Instant | Creation timestamp |

### Account

A financial account owned by a customer.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `customerId` | UUID | Reference to owning Customer |
| `accountNumber` | String | Human-readable account number, externally referenceable |
| `status` | AccountStatus | Current state (PENDING, ACTIVE, BLOCKED, CLOSED) |
| `currency` | String | ISO 4217 currency code (e.g., USD) |
| `createdAt` | Instant | Creation timestamp |
| `updatedAt` | Instant | Last modification timestamp |

### AccountStatus (Enum)

| Value | Description |
| --- | --- |
| `PENDING` | Account created but not yet activated — onboarding incomplete |
| `ACTIVE` | Account is fully operational — can participate in payments |
| `BLOCKED` | Account is temporarily suspended — cannot be used for payment execution |
| `CLOSED` | Account is permanently closed — terminal state, cannot be reopened |

### AccountStatusHistory

Audit trail of all status transitions for an account.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `accountId` | UUID | Reference to Account |
| `previousStatus` | AccountStatus | Status before transition (null for first transition) |
| `newStatus` | AccountStatus | Status after transition |
| `reason` | String | Human-readable reason for transition |
| `changedAt` | Instant | When the transition occurred |
| `changedBy` | String | Entity that triggered the change (user ID or system) |

### AccountLimit

Per-account limit configuration (e.g., transaction limit, daily limit).

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `accountId` | UUID | Reference to Account |
| `limitType` | LimitType | Category of limit (TRANSACTION, DAILY, MONTHLY) |
| `amount` | BigDecimal | Maximum allowed amount |
| `currency` | String | ISO 4217 currency code |

### AccountBlockReason

Reason for blocking an account, stored for auditability.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `accountId` | UUID | Reference to Account |
| `reason` | String | Reason for the block (REGULATORY, FRAUD, CUSTOMER_REQUEST, OTHER) |
| `blockedAt` | Instant | When the block was applied |
| `unblockedAt` | Instant | When the block was lifted (null if still blocked) |

## Relationships

```
Customer (1) ──────────── (N) Account
Account  (1) ──────────── (N) AccountStatusHistory
Account  (1) ──────────── (N) AccountLimit
Account  (1) ──────────── (N) AccountBlockReason
```

## Business Rules

1. An account cannot become ACTIVE unless minimum onboarding requirements are met.
2. Blocked accounts (status == BLOCKED) cannot be used for payment execution.
3. Account status transitions must be validated and auditable via AccountStatusHistory.
4. Customer and account identifiers (UUIDs, account numbers) must be stable and externally referenceable.
5. Account status transitions follow the valid state machine (see flow-diagram.md).
