# Customer Account Service — API Contract

**Base Path**: `/api/v1`

All endpoints require `X-Correlation-ID` header. Responses use standard HTTP status codes and JSON bodies.

## Customers

### Create Customer

```
POST /api/v1/customers
```

**Request Body**:
```json
{
  "firstName": "string (required)",
  "lastName": "string (required)",
  "email": "string (required, valid email format)"
}
```

**Responses**:
- `201 Created` — Customer created successfully
  ```json
  {
    "id": "uuid",
    "firstName": "string",
    "lastName": "string",
    "email": "string",
    "createdAt": "ISO-8601"
  }
  ```
- `400 Bad Request` — Validation error (invalid email, missing required field)
- `409 Conflict` — Email already exists

### Get Customer

```
GET /api/v1/customers/{customerId}
```

**Responses**:
- `200 OK` — Customer details with list of account summaries
- `404 Not Found` — Customer does not exist

## Accounts

### Create Account

```
POST /api/v1/customers/{customerId}/accounts
```

**Request Body**:
```json
{
  "currency": "string (required, ISO 4217, e.g. USD)",
  "accountNumber": "string (optional, auto-generated if not provided)"
}
```

**Responses**:
- `201 Created` — Account created with status PENDING
  ```json
  {
    "id": "uuid",
    "customerId": "uuid",
    "accountNumber": "string",
    "status": "PENDING",
    "currency": "string",
    "createdAt": "ISO-8601"
  }
  ```
- `400 Bad Request` — Validation error
- `404 Not Found` — Customer does not exist

### Get Account

```
GET /api/v1/accounts/{accountId}
```

**Responses**:
- `200 OK` — Full account details including current limits and recent status history
- `404 Not Found` — Account does not exist

### Activate Account

```
POST /api/v1/accounts/{accountId}/activate
```

**Responses**:
- `200 OK` — Account status transitioned PENDING → ACTIVE
  ```json
  {
    "id": "uuid",
    "status": "ACTIVE",
    "updatedAt": "ISO-8601"
  }
  ```
- `400 Bad Request` — Account is already ACTIVE, CLOSED, or onboarding requirements not met
- `404 Not Found` — Account does not exist

### Block Account

```
POST /api/v1/accounts/{accountId}/block
```

**Request Body**:
```json
{
  "reason": "string (required, REGULATORY | FRAUD | CUSTOMER_REQUEST | OTHER)"
}
```

**Responses**:
- `200 OK` — Account status transitioned ACTIVE → BLOCKED
- `400 Bad Request` — Account is already BLOCKED or cannot be blocked from current status
- `404 Not Found` — Account does not exist

### Unblock Account

```
POST /api/v1/accounts/{accountId}/unblock
```

**Responses**:
- `200 OK` — Account status transitioned BLOCKED → ACTIVE, block reason cleared
- `400 Bad Request` — Account is not BLOCKED
- `404 Not Found` — Account does not exist

### Close Account

```
POST /api/v1/accounts/{accountId}/close
```

**Request Body**:
```json
{
  "reason": "string (required)"
}
```

**Responses**:
- `200 OK` — Account status transitioned to CLOSED (terminal state)
- `400 Bad Request` — Account is already CLOSED
- `404 Not Found` — Account does not exist

## Internal Endpoints (service-to-service)

### Validate Account for Payment

```
GET /api/v1/internal/accounts/{accountId}/validate
```

Returns whether an account is eligible for payment execution (ACTIVE, not BLOCKED). Used by payment-orchestrator.

**Responses**:
- `200 OK` — `{ "valid": true, "accountId": "uuid", "status": "ACTIVE" }`
- `200 OK` — `{ "valid": false, "accountId": "uuid", "status": "BLOCKED", "reason": "..." }`
- `404 Not Found` — Account does not exist
