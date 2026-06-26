# Ledger Service — Domain Model

## Entities

### LedgerEntry

Represents a single balanced financial posting. Immutable once created.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `entryDate` | Instant | When the posting was recorded |
| `description` | String | Human-readable description of the posting |
| `idempotencyKey` | String | Unique key preventing duplicate posting execution |
| `createdAt` | Instant | System timestamp |

### LedgerLine

A single debit or credit within a LedgerEntry. The sum of all credits must equal the sum of all debits within one entry.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `ledgerEntryId` | UUID | Reference to parent LedgerEntry |
| `accountReference` | AccountReference | Which account this line debits or credits |
| `amount` | BigDecimal | Monetary amount (always positive) |
| `lineType` | LineType | DEBIT or CREDIT |
| `description` | String | Line-level description |

### LineType (Enum)

| Value | Description |
| --- | --- |
| `DEBIT` | Decreases liability / increases asset |
| `CREDIT` | Increases liability / decreases asset |

### AccountReference

A value object linking a ledger line to an external account identity.

| Field | Type | Description |
| --- | --- | --- |
| `accountId` | UUID | Reference to Account in customer-account-service |
| `accountNumber` | String | Human-readable account number (denormalized for display) |

### IdempotencyKey

Tracks processed idempotency keys to prevent duplicate postings.

| Field | Type | Description |
| --- | --- | --- |
| `key` | String | Unique idempotency key from the client |
| `response` | String | JSON-serialized response from the original posting |
| `createdAt` | Instant | When the key was first processed |

### OutboxEvent

Reliably publishes domain events in the same transaction as the posting.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Immutable unique identifier |
| `aggregateId` | UUID | ID of the aggregate that produced the event (LedgerEntry ID) |
| `eventType` | String | Discriminator (e.g., "POSTING_COMPLETED") |
| `payload` | String | JSON-serialized event data |
| `createdAt` | Instant | When the event was recorded |
| `publishedAt` | Instant | When the event was sent to the broker (null if pending) |

## Relationships

```
LedgerEntry (1) ────────── (2+) LedgerLine    (at least one DEBIT + one CREDIT)
LedgerEntry (1) ────────── (0..1) OutboxEvent  (optional, on creation)
IdempotencyKey           standalone table        (unique constraint on key)
```

## Business Rules

1. Every posting MUST be balanced: SUM(debit amounts) = SUM(credit amounts).
2. LedgerEntry and LedgerLine records are IMMUTABLE — never updated or deleted after creation.
3. The same idempotency key cannot produce duplicate financial effects. Check-and-set in a single transaction.
4. Account balances are DERIVED from ledger lines (SUM credits - SUM debits per account), never stored as mutable state.
5. OutboxEvent persistence MUST be part of the same transactional boundary as the LedgerEntry creation.
6. Each posting SHALL have at least one debit line and one credit line.
