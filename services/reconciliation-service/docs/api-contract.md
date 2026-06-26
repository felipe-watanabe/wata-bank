# Reconciliation Service — API Contract

**Base Path**: `/api/v1`

All endpoints require `X-Correlation-ID` header. Responses use standard HTTP status codes and JSON bodies.

## Reconciliation Jobs

### Import External Transactions

```
POST /api/v1/reconciliation/jobs
```

Uploads a CSV or JSON file of external transactions and kicks off a reconciliation job.

**Request**: `multipart/form-data`
- `file` (required): CSV or JSON file containing external transactions
- `sourceSystem` (required): String identifier for the external system (e.g., "SWIFT", "ACH", "CARD_NETWORK")

**CSV Format**:
```csv
externalReference,amount,currency,transactionDate,sourceAccount,destinationAccount
EXT-001,100.00,USD,2026-01-15T10:30:00Z,ACC-A123,ACC-B456
EXT-002,250.50,USD,2026-01-15T11:00:00Z,ACC-C789,ACC-D012
```

**JSON Format** (alternative):
```json
{
  "transactions": [
    {
      "externalReference": "EXT-001",
      "amount": 100.00,
      "currency": "USD",
      "transactionDate": "2026-01-15T10:30:00Z",
      "sourceAccount": "ACC-A123",
      "destinationAccount": "ACC-B456"
    }
  ]
}
```

**Responses**:
- `202 Accepted` — Job created, reconciliation started
  ```json
  {
    "jobId": "uuid",
    "status": "IN_PROGRESS",
    "totalExternalTransactions": 150,
    "startedAt": "ISO-8601"
  }
  ```
- `400 Bad Request` — Invalid file format or missing required fields

### Get Reconciliation Job

```
GET /api/v1/reconciliation/jobs/{jobId}
```

**Responses**:
- `200 OK` — Full job details with summary counts
  ```json
  {
    "jobId": "uuid",
    "status": "COMPLETED",
    "startedAt": "ISO-8601",
    "completedAt": "ISO-8601",
    "totalExternalTransactions": 150,
    "matchedCount": 145,
    "missingCount": 3,
    "divergentCount": 2
  }
  ```
- `404 Not Found` — Job does not exist

### List Reconciliation Jobs

```
GET /api/v1/reconciliation/jobs?page={page}&size={size}
```

**Responses**:
- `200 OK` — Paginated list of job summaries

## Matching Results

### Get Match Results for Job

```
GET /api/v1/reconciliation/jobs/{jobId}/matches?page={page}&size={size}
```

**Responses**:
- `200 OK` — Paginated list of MatchResult records
  ```json
  {
    "content": [
      {
        "matchId": "uuid",
        "externalReference": "EXT-001",
        "matchType": "BOTH",
        "paymentIntentId": "uuid",
        "ledgerEntryId": "uuid",
        "matchedAt": "ISO-8601"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 145
  }
  ```

## Divergences

### Get Divergences for Job

```
GET /api/v1/reconciliation/jobs/{jobId}/divergences?type={type}&resolved={resolved}&page={page}&size={size}
```

**Query Parameters**:
- `type` (optional) — Filter by DivergenceType (MISSING_IN_PAYMENT, MISSING_IN_LEDGER, AMOUNT_MISMATCH, ACCOUNT_MISMATCH)
- `resolved` (optional) — Filter by resolved status (true/false)

**Responses**:
- `200 OK` — Paginated list of Divergence records
  ```json
  {
    "content": [
      {
        "divergenceId": "uuid",
        "externalReference": "EXT-050",
        "divergenceType": "AMOUNT_MISMATCH",
        "externalAmount": 150.00,
        "internalAmount": 100.00,
        "detail": "External amount 150.00 differs from payment amount 100.00",
        "detectedAt": "ISO-8601",
        "resolved": false
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 5
  }
  ```

## Scheduled Reconciliation

A background job runs on a configurable schedule (cron expression). When triggered:

```
1. Check for new external transaction files in the configured import directory
2. For each file:
   ├─ Create ReconciliationJob (IN_PROGRESS)
   ├─ Parse and persist ExternalTransaction records
   ├─ For each ExternalTransaction:
   │   ├─ Match against payment-orchestrator (by reference)
   │   ├─ Match against ledger-service (by reference)
   │   └─ Classify as MATCHED, MISSING, or DIVERGENT
   └─ Update ReconciliationJob (COMPLETED, with summary counts)
```
