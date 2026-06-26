# Project Foundation — Tasks

## Test Coverage Matrix

| Layer | Coverage Expectation | Applies to tasks |
| --- | --- | --- |
| Documentation | Content completeness: every heading in the spec template has content | All tasks |
| Structure | Directory existence: every expected file path exists | All tasks |

## Gate Check Commands

| Gate Level | Command |
| --- | --- |
| Quick | `ls -la [file path]` — verify file exists |
| Build | `ls -la [file path]` — verify file exists |

## Execution Plan

### Phase 1: Top-Level Foundation

#### T1: Create AGENTS.md
- **Req IDs**: FDN-01
- **Files**: `AGENTS.md`
- **Done when**: AGENTS.md exists with build/lint/test commands, coding conventions, project structure overview, and local dev instructions
- **Tests**: Verify file exists and contains required sections: "Build & Test Commands", "Code Style & Conventions", "Project Structure"
- **Gate**: Quick

#### T2: Create platform/system-overview.md
- **Req IDs**: FDN-02
- **Files**: `platform/docs/architecture/system-overview.md`
- **Done when**: Document describes ecosystem scope, all 5 services + platform layer, shared architecture standards
- **Tests**: Verify file exists, contains service summaries for all 5 services, mentions platform layer role
- **Gate**: Quick
- **Depends on**: T1

### Phase 2: Platform Architecture Docs

#### T3: Create platform/service-map.md
- **Req IDs**: FDN-03
- **Files**: `platform/docs/architecture/service-map.md`
- **Done when**: Document maps service-to-service dependencies, communication protocols, sync vs async boundaries
- **Tests**: Verify file exists, identifies ledger-service as dependency of payment-orchestrator, shows notification-service as async consumer
- **Gate**: Quick
- **Depends on**: T2

#### T4: Create platform/data-flow.md
- **Req IDs**: FDN-04
- **Files**: `platform/docs/architecture/data-flow.md`
- **Done when**: Document describes end-to-end data flows for key scenarios (account creation, payment execution, reconciliation)
- **Tests**: Verify file exists, covers payment flow (customer → payment-orchestrator → ledger → notification), covers reconciliation flow
- **Gate**: Quick
- **Depends on**: T3

#### T5: Create platform ADRs (ADR-001, ADR-002, ADR-003)
- **Req IDs**: FDN-05, FDN-06, FDN-07
- **Files**: `platform/docs/adrs/adr-001-repo-structure.md`, `platform/docs/adrs/adr-002-observability-baseline.md`, `platform/docs/adrs/adr-003-idempotency-strategy.md`
- **Done when**: All 3 ADRs exist, each with title, status, context, decision, consequences sections
- **Tests**: Verify all 3 files exist with required sections
- **Gate**: Quick
- **Depends on**: T2

### Phase 3: Service Documentation — Batch 1

#### T6: Create customer-account-service docs
- **Req IDs**: FDN-08
- **Files**: `services/customer-account-service/README.md`, `services/customer-account-service/docs/domain-model.md`, `services/customer-account-service/docs/api-contract.md`, `services/customer-account-service/docs/flow-diagram.md`
- **Done when**: 4 files exist covering purpose, entities (Customer, Account, AccountStatusHistory, AccountLimit, AccountBlockReason), endpoints (create customer, create account, get account, block/unblock), and state transitions
- **Tests**: Verify all 4 files exist, domain-model lists all 5 entities, api-contract lists all 4 endpoint categories
- **Gate**: Quick
- **Depends on**: T1

#### T7: Create ledger-service docs
- **Req IDs**: FDN-09
- **Files**: `services/ledger-service/README.md`, `services/ledger-service/docs/domain-model.md`, `services/ledger-service/docs/api-contract.md`, `services/ledger-service/docs/flow-diagram.md`
- **Done when**: 4 files exist covering double-entry bookkeeping purpose, entities (LedgerEntry, LedgerLine, AccountReference, IdempotencyKey, OutboxEvent), endpoints (posting, balance query, posting history), and idempotency/outbox patterns
- **Tests**: Verify all 4 files exist, domain-model lists all 5 entities, api-contract covers idempotency key usage
- **Gate**: Quick
- **Depends on**: T6

### Phase 4: Service Documentation — Batch 2

#### T8: Create payment-orchestrator docs
- **Req IDs**: FDN-10
- **Files**: `services/payment-orchestrator/README.md`, `services/payment-orchestrator/docs/domain-model.md`, `services/payment-orchestrator/docs/api-contract.md`, `services/payment-orchestrator/docs/flow-diagram.md`
- **Done when**: 4 files exist covering saga-like coordination purpose, entities (PaymentIntent, PaymentAttempt, PaymentStatusHistory, PaymentIdempotencyKey, PaymentEvent), endpoints (payment intent, state machine), and retry/compensation flows
- **Tests**: Verify all 4 files exist, domain-model lists all 5 entities, api-contract covers duplicate protection
- **Gate**: Quick
- **Depends on**: T7

#### T9: Create notification-service docs
- **Req IDs**: FDN-11
- **Files**: `services/notification-service/README.md`, `services/notification-service/docs/domain-model.md`, `services/notification-service/docs/api-contract.md`, `services/notification-service/docs/flow-diagram.md`
- **Done when**: 4 files exist covering async consumer purpose, entities (Notification, NotificationAttempt, ProcessedMessage, NotificationTemplate, DeliveryFailure), consumer contracts, and deduplication/retry/DLQ patterns
- **Tests**: Verify all 4 files exist, domain-model covers deduplication entity, api-contract documents consumer-only nature
- **Gate**: Quick
- **Depends on**: T8

#### T10: Create reconciliation-service docs
- **Req IDs**: FDN-12
- **Files**: `services/reconciliation-service/README.md`, `services/reconciliation-service/docs/domain-model.md`, `services/reconciliation-service/docs/api-contract.md`, `services/reconciliation-service/docs/flow-diagram.md`
- **Done when**: 4 files exist covering batch reconciliation purpose, entities (ReconciliationJob, ExternalTransaction, MatchResult, Divergence, ExceptionCase), endpoints (import, reconciliation job, divergence report), and matching rules
- **Tests**: Verify all 4 files exist, domain-model lists all 5 entities, api-contract covers MATCHED/MISSING/DIVERGENT statuses
- **Gate**: Quick
- **Depends on**: T9
