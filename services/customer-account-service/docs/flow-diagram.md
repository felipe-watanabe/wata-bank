# Customer Account Service — Flow Diagram

## Account Lifecycle State Machine

```
                    ┌──────────┐
                    │  PENDING  │
                    └─────┬─────┘
                          │ POST /activate
                          │ (onboarding complete)
                          ▼
                    ┌──────────┐
            ┌──────►│  ACTIVE  │◄──────┐
            │       └────┬─────┘       │
            │            │             │
            │  POST      │   POST      │ POST
            │  /block    │   /block    │ /unblock
            │            │             │
            │            ▼             │
            │       ┌──────────┐       │
            └───────│ BLOCKED  │───────┘
                    └────┬─────┘
                         │ POST /close
                         │ (from any state)
                         ▼
                    ┌──────────┐
                    │  CLOSED  │ (terminal)
                    └──────────┘
```

## Key Flows

### 1. Customer and Account Creation

```
POST /customers
  → validate input (firstName, lastName, email)
  → check email uniqueness
  → persist Customer entity
  → return 201 with Customer DTO

POST /customers/{customerId}/accounts
  → validate customer exists
  → validate input (currency)
  → generate account number (if not provided)
  → persist Account entity with status = PENDING
  → persist AccountStatusHistory (null → PENDING)
  → return 201 with Account DTO
```

### 2. Account Activation

```
POST /accounts/{accountId}/activate
  → validate account exists
  → validate current status is PENDING
  → validate onboarding requirements met (e.g., KYC checks — simplified for MVP)
  → update Account.status = ACTIVE
  → persist AccountStatusHistory (PENDING → ACTIVE)
  → return 200 with updated Account DTO
```

### 3. Account Block / Unblock

```
POST /accounts/{accountId}/block
  → validate account exists
  → validate current status is ACTIVE
  → accept reason (REGULATORY, FRAUD, CUSTOMER_REQUEST, OTHER)
  → update Account.status = BLOCKED
  → persist AccountBlockReason entity
  → persist AccountStatusHistory (ACTIVE → BLOCKED)
  → return 200 with updated Account DTO

POST /accounts/{accountId}/unblock
  → validate account exists
  → validate current status is BLOCKED
  → update Account.status = ACTIVE
  → update AccountBlockReason.unblockedAt = now
  → persist AccountStatusHistory (BLOCKED → ACTIVE)
  → return 200 with updated Account DTO
```

### 4. Payment Validation (Internal)

```
GET /internal/accounts/{accountId}/validate
  → called by payment-orchestrator during payment intent processing
  → validate account exists
  → check account.status == ACTIVE
  → if BLOCKED: return valid=false with block reason
  → return valid=true
```

### 5. Account Closure

```
POST /accounts/{accountId}/close
  → validate account exists
  → validate current status is not CLOSED
  → update Account.status = CLOSED
  → persist AccountStatusHistory (current → CLOSED)
  → CLOSED is terminal — cannot be reopened
  → return 200 with updated Account DTO
```
