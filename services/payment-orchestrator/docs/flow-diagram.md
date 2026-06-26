# Payment Orchestrator — Flow Diagram

## Payment Lifecycle State Machine

```
                    ┌──────────┐
                    │  PENDING │
                    └────┬─────┘
                         │
                         ▼
                    ┌────────────┐
              ┌────►│ VALIDATING │
              │     └──┬──────┬──┘
              │        │      │
              │   valid│      │ invalid (blocked, nonexistent)
              │        │      │
              │        ▼      ▼
              │   ┌──────────┐   ┌──────────┐
              │   │ DEBITING │   │ REJECTED │ (terminal)
              │   └────┬─────┘   └──────────┘
              │        │
              │  ledger│success
              │        │
              │        ▼
              │   ┌──────────┐
              │   │ SETTLED  │ (terminal success)
              │   └──────────┘
              │
              │  transient failure + retryable:
              │   ┌──────────┐
              └───│ FAILED   │──► retry (max 3, with backoff)
                  └──────────┘
                  │
                  │ max retries exhausted:
                  ▼
              ┌──────────┐
              │ REJECTED │ (terminal — manual review required)
              └──────────┘
```

## Happy Path: Payment Settlement

```
1. Client → POST /payments (debitAccountId, creditAccountId, amount, idempotencyKey)
   ├─ Check PaymentIdempotencyKey: if duplicate → return stored response
   ├─ Create PaymentIntent (status = PENDING)
   ├─ Persist PaymentStatusHistory (null → PENDING)
   ├─ Persist OutboxEvent (PAYMENT_CREATED)
   ├─ Return 202 Accepted
   └─ (Async saga begins)

2. Saga Step 1: VALIDATING
   ├─ Transition status: PENDING → VALIDATING
   ├─ Persist PaymentStatusHistory
   ├─ Call customer-account-service: GET /internal/accounts/{debit}/validate
   │   ├─ valid=true → continue
   │   └─ valid=false → reject (see rejection flow)
   ├─ Call customer-account-service: GET /internal/accounts/{credit}/validate
   │   ├─ valid=true → continue
   │   └─ valid=false → reject (see rejection flow)
   └─ Both accounts valid → proceed to DEBITING

3. Saga Step 2: DEBITING
   ├─ Transition status: VALIDATING → DEBITING
   ├─ Persist PaymentStatusHistory
   ├─ Construct balanced posting:
   │   ├─ DEBIT: debitAccountId, amount
   │   └─ CREDIT: creditAccountId, amount
   ├─ Call ledger-service: POST /postings (lines, idempotencyKey=paymentIntentId)
   │   ├─ 201 Created → proceed to SETTLED
   │   └─ Timeout / 5xx → FAILED (retryable)
   └─ Create PaymentAttempt (attemptNumber, status, ledgerEntryId)

4. Saga Step 3: SETTLED
   ├─ Transition status: DEBITING → SETTLED
   ├─ Persist PaymentStatusHistory
   ├─ Persist OutboxEvent (PAYMENT_SETTLED)
   └─ Done — terminal success state
```

## Rejection Flow (Blocked/Invalid Account)

```
Saga Step 1 (VALIDATING):
   ├─ Account validation returns valid=false (or 404)
   ├─ Transition status: VALIDATING → REJECTED
   ├─ Persist PaymentStatusHistory
   ├─ Create PaymentAttempt (status=REJECTED, failureReason="Account blocked: fraud")
   ├─ Persist OutboxEvent (PAYMENT_REJECTED)
   └─ Done — terminal failure state
```

## Retry Flow (Transient Failure)

```
Saga Step 2 (DEBITING):
   ├─ Ledger service call fails with timeout / 503
   ├─ Transition status: VALIDATING → FAILED
   ├─ Persist PaymentStatusHistory
   ├─ Create PaymentAttempt (status=FAILED, failureReason="Ledger timeout")
   ├─ Check attemptNumber < 3
   │   ├─ Yes → schedule retry with exponential backoff
   │   │   └─ Retries: 1s, 2s, 4s delays
   │   └─ No → max retries exhausted
   │       ├─ Transition status: FAILED → REJECTED
   │       └─ Persist OutboxEvent (PAYMENT_REJECTED)
   └─ Persist OutboxEvent (PAYMENT_FAILED)
```

## Compensation Flow (Future Enhancement)

```
For reversals/refunds: create a new PaymentIntent reversing debit/credit:
   ├─ debitAccountId = original creditAccountId
   ├─ creditAccountId = original debitAccountId
   ├─ amount = original amount
   └─ reference = "Reversal of {original.paymentIntentId}"
```

## Idempotency Guarantee

```
1. Client includes idempotencyKey in POST /payments
2. Orchestrator checks-and-inserts PaymentIdempotencyKey within a single transaction
3. If key exists:
   - If original PaymentIntent settled: return 200 with original result
   - If original PaymentIntent still processing: return 202 with current status
4. If key does not exist: create new PaymentIntent and insert key atomically
```

**Protection**: Even if two requests arrive simultaneously, the unique constraint on `idempotencyKey` ensures exactly one PaymentIntent is created.
