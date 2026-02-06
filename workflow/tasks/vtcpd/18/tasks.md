# PRD-18 Tasks: Audit Mechanism Based on Finalized Transactions

## PRD Reference
- [PRD-18: Audit Mechanism Based on Finalized Transactions](../../prd/vtcpd/18-audit-mechanism-finalized-transactions.md)

## Task List

| Task ID | Name | Complexity | Status | Dependencies |
|---------|------|------------|--------|--------------|
| [18-01](18-01-audit-messages-extension.md) | Audit Messages Extension | Simple | Pending | None |
| [18-02](18-02-receipt-handlers-and-zero-audit-number.md) | Receipt Handlers and Zero Audit Number | Moderate | Pending | None |
| [18-03](18-03-trust-lines-manager-receipts-preservation.md) | TrustLinesManager Receipts Preservation | Simple | Pending | None |
| [18-04](18-04-audit-source-transaction.md) | AuditSourceTransaction Implementation | Complex | Pending | 18-01, 18-02, 18-03 |
| [18-05](18-05-audit-target-transaction.md) | AuditTargetTransaction Implementation | Complex | Pending | 18-01, 18-02, 18-03 |
| [18-06](18-06-other-audit-transactions.md) | Other Audit Transactions Update | Moderate | Pending | 18-04, 18-05 |
| [18-07](18-07-unit-tests.md) | Unit Tests | Moderate | Pending | 18-01, 18-02, 18-03, 18-08 |
| [18-08](18-08-audit-submit-claim-votes.md) | Audit Completion Claim Votes Submission | Moderate | Pending | 18-04, 18-05, 18-06 |

## Execution Order

### Phase 1: Infrastructure (can be executed in parallel)
- 18-01: Audit Messages Extension
- 18-02: Receipt Handlers and Zero Audit Number
- 18-03: TrustLinesManager Receipts Preservation

### Phase 2: Core Logic (can be executed in parallel after Phase 1)
- 18-04: AuditSourceTransaction Implementation
- 18-05: AuditTargetTransaction Implementation

### Phase 3: Remaining Transactions
- 18-06: Other Audit Transactions Update

### Phase 4: Audit Completion Claim Votes Submission
- 18-08: Audit Completion Claim Votes Submission

### Phase 5: Testing
- 18-07: Unit Tests

## Dependency Diagram

```
Phase 1 (parallel):
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   18-01     │  │   18-02     │  │   18-03     │
│  Messages   │  │  Handlers   │  │  Manager    │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
Phase 2 (parallel):     ▼
       ┌────────────────┴────────────────┐
       │                                 │
┌──────▼──────┐                  ┌───────▼─────┐
│   18-04     │                  │   18-05     │
│   Source    │                  │   Target    │
└──────┬──────┘                  └───────┬─────┘
       │                                 │
       └────────────────┬────────────────┘
                        │
Phase 3:                ▼
                ┌───────┴───────┐
                │    18-06      │
                │    Other      │
                └───────────────┘

                        │
Phase 4:                ▼
                ┌───────────────┐
                │    18-08      │
                │    Votes      │
                └───────────────┘

Phase 5 (can start after Phase 1):
                ┌───────────────┐
                │    18-07      │
                │    Tests      │
                └───────────────┘
```

## Summary

- **Total Tasks**: 8
- **Simple Tasks**: 2 (18-01, 18-03)
- **Moderate Tasks**: 4 (18-02, 18-06, 18-07, 18-08)
- **Complex Tasks**: 2 (18-04, 18-05)

## Notes

- Tasks 18-01, 18-02, 18-03 have no dependencies and can be executed in parallel
- Tasks 18-04 and 18-05 depend on Phase 1 completion and can be executed in parallel
- Task 18-06 depends on both 18-04 and 18-05
- Task 18-08 depends on 18-04, 18-05, and 18-06
- Task 18-07 (unit tests) can start after Phase 1 for testing infrastructure components, but should include 18-08 coverage
