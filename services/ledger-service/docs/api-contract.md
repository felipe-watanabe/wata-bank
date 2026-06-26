# Ledger Service — API Contract

**Base Path**: `/api/v1`

All endpoints require `X-Correlation-ID` header. Responses use standard HTTP status codes and JSON bodies.

## Postings

### Create Posting

```
POST /api/v1/postings
```

**Request Body**:
```json
{
  "idempotencyKey": "string (required, UUID format, client-generated)",
  "description": "string (required, e.g. 'Payment settlement for intent abc-123')",
  "lines": [
    {
      "accountId": "uuid (required)",
      "accountNumber": "string",
      "amount": "decimal (required, positive)",
      "lineType": "DEBIT | CREDIT (required)",
      "description": "string (optional)"
    }
  ]
}
```

**Validation Rules**:
- At least one DEBIT and one CREDIT line
- SUM(debit amounts) == SUM(credit amounts) — posting must be balanced
- All amounts must be positive
- `idempotencyKey` is required and must be unique per request

**Responses**:
- `201 Created` — Posting executed successfully
  ```json
  {
    "entryId": "uuid",
    "entryDate": "ISO-8601",
    "description": "string",
    "idempotencyKey": "string",
    "lines": [
      {
        "lineId": "uuid",
        "accountId": "uuid",
        "amount": "decimal",
        "lineType": "DEBIT | CREDIT",
        "description": "string"
      }
    ]
  }
  ```
- `200 OK` — Idempotent response (same idempotency key, returns original result)
- `400 Bad Request` — Unbalanced posting, invalid line types, missing required fields
- `409 Conflict` — Processing conflict (retry with same idempotency key)

### Get Posting

```
GET /api/v1/postings/{entryId}
```

**Responses**:
- `200 OK` — Full LedgerEntry with all LedgerLines
- `404 Not Found` — Entry does not exist

### List Postings (with pagination)

```
GET /api/v1/postings?accountId={accountId}&page={page}&size={size}
```

**Query Parameters**:
- `accountId` (optional) — Filter postings involving this account
- `page` (default: 0)
- `size` (default: 20, max: 100)

**Responses**:
- `200 OK` — Paginated list of LedgerEntry summaries
  ```json
  {
    "content": [
      {
        "entryId": "uuid",
        "entryDate": "ISO-8601",
        "description": "string"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 150
  }
  ```

## Balances

### Get Account Balance

```
GET /api/v1/accounts/{accountId}/balance
```

Balance is derived from all LedgerLines referencing this account: SUM(CREDIT amounts) - SUM(DEBIT amounts).

**Responses**:
- `200 OK`
  ```json
  {
    "accountId": "uuid",
    "balance": "decimal",
    "currency": "string",
    "asOf": "ISO-8601 (timestamp of the calculated balance)"
  }
  ```
- `404 Not Found` — No postings exist for this account (returns zero balance)

### Get Account Balance at Point in Time

```
GET /api/v1/accounts/{accountId}/balance?asOf=2026-01-01T00:00:00Z
```

**Query Parameters**:
- `asOf` (required) — ISO-8601 timestamp; balance is calculated from all postings up to this point

**Responses**:
- `200 OK` — Same format as current balance, with `asOf` reflecting the requested timestamp

## Idempotency

All POST endpoints are idempotent via the `idempotencyKey`:
- First request with key XYZ: executes posting, stores key + response, returns `201 Created`
- Subsequent request with key XYZ: returns stored response, returns `200 OK` (no duplicate posting)
- The check and insert are in the same database transaction — no race condition window
