# Reconciliation Service — Domain Model

## Entities

### ReconciliationJob

Represents a single reconciliation run against a set of external transactions.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `startedAt` | Instant | When the job started |
| `completedAt` | Instant | When the job finished (null if in progress) |
| `status` | JobStatus | IN_PROGRESS, COMPLETED, FAILED |
| `totalExternalTransactions` | int | Number of external records processed |
| `matchedCount` | int | Number of matched records |
| `missingCount` | int | Number of records missing in internal systems |
| `divergentCount` | int | Number of records with data mismatches |

### JobStatus (Enum)

| Value | Description |
| --- | --- |
| `IN_PROGRESS` | Job is actively processing |
| `COMPLETED` | Job finished successfully (may include mismatches) |
| `FAILED` | Job encountered a non-recoverable error |

### ExternalTransaction

An imported transaction from an external feed, linked to a reconciliation job.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `jobId` | UUID | Reference to parent ReconciliationJob |
| `externalReference` | String | External transaction identifier |
| `amount` | BigDecimal | Transaction amount |
| `currency` | String | ISO 4217 currency code |
| `transactionDate` | Instant | When the external transaction occurred |
| `sourceAccount` | String | External source account identifier |
| `destinationAccount` | String | External destination account identifier |
| `rawData` | String | Original record as JSON (for audit) |

### MatchResult

Records a successful match between an external transaction and an internal record.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `externalTransactionId` | UUID | Reference to ExternalTransaction |
| `matchType` | MatchType | PAYMENT, LEDGER, or BOTH |
| `paymentIntentId` | UUID | Matched payment intent (null if not matched) |
| `ledgerEntryId` | UUID | Matched ledger entry (null if not matched) |
| `matchedAt` | Instant | When the match was identified |

### MatchType (Enum)

| Value | Description |
| --- | --- |
| `PAYMENT` | External transaction matched a payment intent |
| `LEDGER` | External transaction matched a ledger entry |
| `BOTH` | External transaction matched both payment and ledger records |
| `NONE` | No match found (routed to Divergence) |

### Divergence

Records a mismatch between external and internal data.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `externalTransactionId` | UUID | Reference to ExternalTransaction |
| `divergenceType` | DivergenceType | MISSING_IN_PAYMENT, MISSING_IN_LEDGER, AMOUNT_MISMATCH, ACCOUNT_MISMATCH |
| `internalReference` | String | Internal record reference (if a match was found but data differed) |
| `detail` | String | Human-readable description of the discrepancy |
| `detectedAt` | Instant | When the divergence was identified |
| `resolved` | boolean | Whether the divergence has been addressed |

### DivergenceType (Enum)

| Value | Description |
| --- | --- |
| `MISSING_IN_PAYMENT` | External transaction has no corresponding payment intent |
| `MISSING_IN_LEDGER` | External transaction has no corresponding ledger entry |
| `AMOUNT_MISMATCH` | Amounts differ between external and internal records |
| `ACCOUNT_MISMATCH` | Account identifiers differ between external and internal records |

### ExceptionCase

Manually reviewed exceptions — escalated divergences that require human intervention.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `divergenceId` | UUID | Reference to related Divergence |
| `status` | ExceptionStatus | OPEN, UNDER_REVIEW, RESOLVED |
| `assignedTo` | String | User responsible for resolution |
| `notes` | String | Review notes |
| `createdAt` | Instant | When the case was created |
| `resolvedAt` | Instant | When the case was resolved |

## Relationships

```
ReconciliationJob (1) ────── (N) ExternalTransaction
ExternalTransaction (1) ──── (0..1) MatchResult
ExternalTransaction (1) ──── (0..1) Divergence
Divergence (1) ───────────── (0..1) ExceptionCase
```

## Business Rules

1. Imported records MUST be traceable to a reconciliation job (jobId on ExternalTransaction).
2. Matching rules MUST be deterministic and explicit — same input always produces the same result.
3. Missing and divergent cases MUST be preserved for review — never silently discarded.
4. Reconciliation MUST NOT mutate financial truth directly. Divergences are flags for manual review or controlled follow-up.
5. Each reconciliation job SHALL report summary counts: total, matched, missing, divergent.
6. External reference matching is case-insensitive and leading/trailing whitespace is trimmed.
