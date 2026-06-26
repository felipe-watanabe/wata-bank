# Reconciliation Service — Flow Diagram

## Reconciliation Job Lifecycle

```
Trigger: POST /reconciliation/jobs (file upload)
   or Scheduled job detects new file in import directory
   │
   ▼
┌─────────────────────────────────────────────────────┐
│ 1. CREATE JOB                                       │
│    ├─ Create ReconciliationJob (status = IN_PROGRESS)│
│    ├─ Set totalExternalTransactions = line count     │
│    └─ Return jobId → continue async                 │
│                                                      │
│ 2. PARSE & PERSIST EXTERNAL TRANSACTIONS            │
│    ├─ Parse CSV/JSON file                           │
│    ├─ Validate each record:                          │
│    │   ├─ externalReference: required, unique        │
│    │   ├─ amount: required, positive decimal         │
│    │   ├─ currency: required, ISO 4217 format        │
│    │   └─ transactionDate: required, ISO-8601        │
│    ├─ For each valid record:                        │
│    │   └─ INSERT ExternalTransaction (jobId, data)   │
│    └─ Invalid records → skip + log error             │
│                                                      │
│ 3. MATCH AGAINST INTERNAL SYSTEMS                   │
│    └──► See Matching Flow below                     │
│                                                      │
│ 4. FINALIZE JOB                                     │
│    ├─ Compute summary counts:                       │
│    │   ├─ matchedCount = COUNT(MatchResult)          │
│    │   ├─ missingCount = COUNT(Divergence WHERE      │
│    │   │   type IN (MISSING_IN_PAYMENT,              │
│    │   │            MISSING_IN_LEDGER))               │
│    │   └─ divergentCount = COUNT(Divergence WHERE    │
│    │       type IN (AMOUNT_MISMATCH,                 │
│    │                ACCOUNT_MISMATCH))                │
│    ├─ Update ReconciliationJob:                     │
│    │   ├─ status = COMPLETED (or FAILED)             │
│    │   ├─ completedAt = now()                        │
│    │   └─ summary counts updated                     │
│    └─ DONE                                           │
└─────────────────────────────────────────────────────┘
```

## Matching Flow (Per ExternalTransaction)

```
For each ExternalTransaction (externalRef, amount, sourceAcc, destAcc):
  │
  ├─ STEP A: Match against Payment Orchestrator
  │   ├─ GET payment-orchestrator: /payments?reference={externalRef}
  │   │   ├─ 200 OK (found) → matchedPayment = payment
  │   │   │   ├─ Compare amount:
  │   │   │   │   ├─ amounts match → payment match confirmed
  │   │   │   │   └─ amounts differ → Create Divergence (AMOUNT_MISMATCH)
  │   │   │   └─ Continue to Step B
  │   │   │
  │   │   └─ 200 OK (empty) → no payment match
  │   │       └─ Continue to Step B
  │   │
  │   └─ If no match: → Create Divergence (MISSING_IN_PAYMENT)
  │
  ├─ STEP B: Match against Ledger Service
  │   ├─ GET ledger-service: /postings?reference={externalRef}
  │   │   ├─ 200 OK (found) → matchedLedgerEntry = entry
  │   │   │   ├─ Compare amount:
  │   │   │   │   ├─ amounts match → ledger match confirmed
  │   │   │   │   └─ amounts differ → Create Divergence (AMOUNT_MISMATCH)
  │   │   │   └─ Continue to Step C
  │   │   │
  │   │   └─ 200 OK (empty) → no ledger match
  │   │       └─ Continue to Step C
  │   │
  │   └─ If no match: → Create Divergence (MISSING_IN_LEDGER)
  │
  └─ STEP C: Classify Result
      ├─ Both payment + ledger matched → MatchResult (BOTH)
      ├─ Only payment matched → MatchResult (PAYMENT)
      ├─ Only ledger matched → MatchResult (LEDGER)
      └─ Neither matched → Divergence (already created)
```

## Matching Rules (Deterministic)

| Rule | Description |
| --- | --- |
| Reference matching | External reference → payment orchestration reference OR ledger posting reference (case-insensitive, trimmed) |
| Amount tolerance | Exact match (BigDecimal equals). No rounding tolerance — financial reconciliation is precise. |
| Currency matching | Currency codes must match exactly. Cross-currency reconciliation is out of scope for MVP. |
| Date window | Optional: only consider internal records within ±1 day of external transaction date. |

## Job Scheduling

```
Scheduled job (configurable cron, e.g., daily at 02:00):
  │
  ├─ Scan import directory: /data/reconciliation/imports/
  │
  ├─ For each unprocessed file (*.csv, *.json):
  │   ├─ Acquire distributed lock (optional, for single-execution guarantee)
  │   ├─ Create ReconciliationJob
  │   ├─ Process file → Matching Flow
  │   ├─ Move file to /data/reconciliation/processed/
  │   └─ If failure → Move file to /data/reconciliation/failed/
  │
  └─ Log summary: { jobsCompleted, totalTransactions, matched, missing, divergent }
```

## Report Generation

```
GET /reconciliation/jobs/{jobId}/report
  │
  └─ Generate summary report:
      │
      ├─ Job metadata: jobId, startedAt, completedAt, status
      ├─ Summary: total, matched, missing, divergent
      ├─ Missing section: list of MISSING_IN_PAYMENT + MISSING_IN_LEDGER
      │   └─ Each with externalReference, amount, date
      └─ Divergent section: list of AMOUNT_MISMATCH + ACCOUNT_MISMATCH
          └─ Each with externalReference, externalAmount, internalAmount, detail
```
