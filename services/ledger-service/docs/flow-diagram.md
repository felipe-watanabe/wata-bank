# Ledger Service — Flow Diagram

## Posting Workflow (Happy Path)

```
Client → POST /postings (idempotencyKey, lines[])
  │
  ├─ BEGIN TRANSACTION
  │
  ├─ Check idempotency: SELECT * FROM idempotency_keys WHERE key = ?
  │   ├─ Found → return stored response (200 OK, idempotent)
  │   └─ Not found → continue
  │
  ├─ Validate balance: SUM(debit amounts) == SUM(credit amounts)
  │   └─ Unbalanced → ROLLBACK, return 400 Bad Request
  │
  ├─ Validate lines: at least one DEBIT and one CREDIT
  │   └─ Invalid → ROLLBACK, return 400 Bad Request
  │
  ├─ Insert LedgerEntry (id, date, description)
  │
  ├─ For each line:
  │   └─ Insert LedgerLine (entryId, accountRef, amount, lineType)
  │
  ├─ Insert IdempotencyKey (key, serialized response)
  │
  ├─ Insert OutboxEvent (aggregateId=entryId, eventType="POSTING_COMPLETED", payload)
  │
  ├─ COMMIT TRANSACTION
  │
  └─ Return 201 Created (entryId, lines[])
```

## Outbox Publisher (Async)

```
Background scheduler (every N seconds):
  │
  ├─ SELECT * FROM outbox_events WHERE published_at IS NULL ORDER BY created_at
  │
  └─ For each event:
      ├─ Publish to Kafka/RabbitMQ
      │   ├─ Success → UPDATE published_at = NOW()
      │   └─ Failure → retry on next cycle (at-least-once delivery)
      │
      └─ Continued...
```

**Delivery guarantees**: At-least-once. Consumers (notification-service) are idempotent — duplicate events are detected and discarded.

## Balance Derivation

```
GET /accounts/{accountId}/balance
  │
  ├─ SELECT
  │     SUM(CASE WHEN line_type = 'CREDIT' THEN amount ELSE 0 END) -
  │     SUM(CASE WHEN line_type = 'DEBIT'  THEN amount ELSE 0 END) AS balance
  │   FROM ledger_lines ll
  │   JOIN ledger_entries le ON ll.ledger_entry_id = le.id
  │   WHERE ll.account_id = ?
  │     [AND le.entry_date <= ? (when asOf is provided)]
  │
  └─ Return result (zero if no postings exist)
```

**Key principle**: Balance is never stored — it is a projection over immutable ledger lines.

## Concurrency: Idempotency Key Protection

```
Thread A: POST /postings (idempotencyKey=XYZ)       Thread B: POST /postings (idempotencyKey=XYZ)
  │                                                     │
  ├─ BEGIN TX                                           ├─ BEGIN TX
  ├─ CHECK key XYZ: not found                           ├─ CHECK key XYZ:
  │  (proceeds)                                         │  (BLOCKED — waits for Thread A's TX)
  ├─ INSERT LedgerEntry                                 │
  ├─ INSERT IdempotencyKey XYZ                          │
  ├─ INSERT OutboxEvent                                 │
  ├─ COMMIT                                             ├─ (unblocked) CHECK key XYZ: found
  └─ Return 201                                         ├─ COMMIT
                                                        └─ Return 200 (stored response)
```

**Protection**: The unique constraint on `idempotency_key` combined with SERIALIZABLE or READ COMMITTED isolation ensures only one transaction succeeds.
