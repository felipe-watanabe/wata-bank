# Data Flow

## Key Scenarios

### 1. Account Creation and Activation

```
Client → customer-account-service: POST /customers (create customer)
  → customer-account-service persists Customer
  → returns customerId

Client → customer-account-service: POST /accounts (create account for customer)
  → customer-account-service persists Account (status: PENDING)
  → returns accountId

Client → customer-account-service: POST /accounts/{id}/activate (onboarding complete)
  → customer-account-service validates onboarding requirements
  → transitions Account status: PENDING → ACTIVE
  → persists AccountStatusHistory entry
  → returns updated account
```

**Data flow**: All synchronous, single-service. No events emitted for account activation (simplified model).

### 2. Payment Execution (Happy Path)

```
Client → payment-orchestrator: POST /payments (create payment intent)
  → payment-orchestrator persists PaymentIntent (status: PENDING)
  → payment-orchestrator publishes PaymentCreated event (outbox)
  → returns paymentIntentId

payment-orchestrator → customer-account-service: GET /accounts/{debitAccountId}
  → validates account exists and is ACTIVE
  → validates account is not blocked
  → validates sufficient limit if applicable

payment-orchestrator → customer-account-service: GET /accounts/{creditAccountId}
  → validates account exists and is ACTIVE

payment-orchestrator → ledger-service: POST /postings
  (with balanced debit/credit lines + idempotency key)
  → ledger-service validates balance (debits = credits)
  → ledger-service persists LedgerEntry + LedgerLines (immutable)
  → ledger-service persists OutboxEvent in same transaction
  → returns posting success

payment-orchestrator: transitions PaymentIntent status PENDING → SETTLED
  → publishes PaymentSettled event (outbox)
  → returns payment status to client

ledger-service outbox publisher: reads OutboxEvent → publishes to Kafka

notification-service: consumes PaymentSettled event from Kafka
  → checks deduplication (ProcessedMessage)
  → creates Notification
  → simulates delivery (MailHog in dev)
  → persists NotificationAttempt
```

**Key ACID boundaries**:
- Account validation is a read — no transactional boundary needed between services
- Ledger posting + OutboxEvent persistence is a single PostgreSQL transaction
- Idempotency key prevents duplicate financial effects on retry

### 3. Payment Execution (Blocked Account — Rejection)

```
Client → payment-orchestrator: POST /payments (create payment intent)
  → persists PaymentIntent (status: PENDING)

payment-orchestrator → customer-account-service: GET /accounts/{debitAccountId}
  → account is BLOCKED

payment-orchestrator: transitions PaymentIntent status PENDING → REJECTED
  → captures rejection reason (account blocked)
  → publishes PaymentRejected event (outbox)
  → returns rejection to client
```

### 4. Reconciliation Workflow

```
Operator → reconciliation-service: POST /reconciliation/jobs (upload external transactions CSV/JSON)
  → reconciliation-service creates ReconciliationJob (status: IN_PROGRESS)
  → parses and persists ExternalTransaction records linked to job

reconciliation-service: For each ExternalTransaction:
  → calls payment-orchestrator: GET /payments?reference={txRef}
    → if found: creates MatchResult(status: MATCHED, paymentId)
    → if not found: creates Divergence(status: MISSING_IN_PAYMENT)
  → calls ledger-service: GET /postings?reference={txRef}
    → cross-validates against payment match
    → if mismatch: creates Divergence(status: DIVERGENT)

reconciliation-service: finalizes ReconciliationJob (status: COMPLETED)
  → stores MatchResult, Divergence, ExceptionCase records
  → returns job summary to operator
```

**Key principle**: Reconciliation is read-only — it does not mutate financial truth. Divergences are flagged for operational review and controlled follow-up.

### 5. Notification Flow (Idempotent Consumer)

```
Kafka → notification-service: PaymentSettled event (eventId: XYZ)
  → notification-service checks ProcessedMessage table for eventId XYZ
  → XYZ not found → process event:
    → constructs Notification from template
    → attempts delivery (MailHog SMTP)
    → persists Notification + NotificationAttempt
    → inserts ProcessedMessage (eventId XYZ) in same transaction
    → acknowledges Kafka offset

Kafka → notification-service: PaymentSettled event (eventId: XYZ) [redelivery]
  → notification-service checks ProcessedMessage table for eventId XYZ
  → XYZ found → discard event (duplicate)
  → acknowledges Kafka offset (no side effects)
```

**Key guarantee**: The same event never produces the same notification side effect twice.
