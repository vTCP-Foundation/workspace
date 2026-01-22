# 18-06 - Other Audit Transactions Update

# Links
- [PRD-18: Audit Mechanism Based on Finalized Transactions](../../prd/vtcpd/18-audit-mechanism-finalized-transactions.md)
- [Previous task: 18-04-audit-source-transaction](18-04-audit-source-transaction.md)
- [Previous task: 18-05-audit-target-transaction](18-05-audit-target-transaction.md)

# Description

This task applies the same audit logic changes implemented in `AuditSourceTransaction` (Task 18-04) to other transactions that perform audits as the initiator:
- `SetOutgoingTrustLineTransaction`
- `CloseIncomingTrustLineTransaction`

These transactions initiate audits when modifying trust line parameters and must use the same finalized-transaction-based audit mechanism.

The implementation should reuse or refactor common logic from `AuditSourceTransaction` where possible to avoid code duplication.

# Requirements and DOD

## Requirements

### SetOutgoingTrustLineTransaction
1. Add block number retrieval step before audit
2. If block number retrieval fails: log error, return `resultDone()`, add TODO
3. Select only finalized receipts with `auditNumber = 0`
4. Build sorted transaction UUID list
5. Deduplicate the sorted transaction UUID list (no duplicates)
6. Compute transaction list hash for signature
7. Include transaction list in AuditMessage
8. Handle `Audit_OK`: update receipt audit numbers, preserve excluded amounts, and do both atomically in a single storage transaction
9. Handle `Audit_UpdateTransactionsList`: validate UUIDs, retry (max 1), or Conflict
10. Verify observing is still possible for each UUID in `Audit_UpdateTransactionsList`; if any is not observable, set TL to Conflict
11. Handle `Audit_Invalid`: set TL to Conflict
12. Implement retry counter (max 1 retry)

### CloseIncomingTrustLineTransaction
13. Same requirements as SetOutgoingTrustLineTransaction (points 1-12)

### Code Reuse
14. Consider extracting common audit logic to `BaseTrustLineTransaction` or a helper class
15. If extraction is complex, acceptable to duplicate with clear documentation
16. Ensure consistent behavior across all audit-initiating transactions

## Definition of Done

- [ ] `SetOutgoingTrustLineTransaction` updated with new audit logic
  - [ ] Block number retrieval implemented
  - [ ] Finalized receipt selection implemented
  - [ ] Transaction list in AuditMessage
  - [ ] All response handlers implemented
  - [ ] Retry logic implemented
  - [ ] Observing window check for update list implemented
- [ ] Transaction list deduplication implemented for SetOutgoing/CloseIncoming
- [ ] Receipt updates and trust line state update are atomic on success
- [ ] `CloseIncomingTrustLineTransaction` updated with new audit logic
  - [ ] Block number retrieval implemented
  - [ ] Finalized receipt selection implemented
  - [ ] Transaction list in AuditMessage
  - [ ] All response handlers implemented
  - [ ] Retry logic implemented
  - [ ] Observing window check for update list implemented
- [ ] Common logic refactored if practical
- [ ] Code compiles without errors
- [ ] Behavior consistent with AuditSourceTransaction

# Implementation Plan

## Step 1: Analyze Code Reuse Opportunity

1. Review `AuditSourceTransaction` implementation from Task 18-04
2. Review `SetOutgoingTrustLineTransaction` current audit flow
3. Review `CloseIncomingTrustLineTransaction` current audit flow
4. Identify common code that can be extracted to:
   - `BaseTrustLineTransaction` (if it exists and is appropriate)
   - A new helper class/functions
   - Or document why duplication is acceptable

## Step 2: Analyze SetOutgoingTrustLineTransaction

1. Read `src/core/transactions/transactions/trust_lines/SetOutgoingTrustLineTransaction.h`
2. Read `src/core/transactions/transactions/trust_lines/SetOutgoingTrustLineTransaction.cpp`
3. Identify where audit is initiated
4. Identify current receipt handling
5. Identify response handling

## Step 3: Analyze CloseIncomingTrustLineTransaction

1. Read `src/core/transactions/transactions/trust_lines/CloseIncomingTrustLineTransaction.h`
2. Read `src/core/transactions/transactions/trust_lines/CloseIncomingTrustLineTransaction.cpp`
3. Identify where audit is initiated
4. Identify current receipt handling
5. Identify response handling

## Step 4: Implement Common Helper (if practical)

Option A - Helper class:
```cpp
class AuditHelper {
public:
    static vector<TransactionUUID> selectFinalizedTransactions(...);
    static BytesShared computeTransactionListHash(...);
    static bool validateUpdateTransactionsList(...);
    // etc.
};
```

Option B - Base class methods:
```cpp
// In BaseTrustLineTransaction
protected:
    ResultCode runBlockNumberRetrievalStep();
    vector<TransactionUUID> selectFinalizedReceipts();
    void handleAuditOK(...);
    void handleAuditUpdateTransactionsList(...);
    void handleAuditInvalid();
```

Option C - Duplicate with documentation (if complexity doesn't justify extraction)

## Step 5: Update SetOutgoingTrustLineTransaction

1. Add member variables (same as AuditSourceTransaction):
   - `BlockNumber mCurrentBlockNumber`
   - `vector<TransactionUUID> mOriginalTransactionList`
   - `vector<TransactionUUID> mCurrentTransactionList`
   - `uint8_t mAuditRetryCount`

2. Add block number retrieval step before audit stage

3. Modify audit stage:
   - Select finalized receipts
   - Build sorted transaction list
   - Deduplicate the sorted list
   - Compute hash
   - Construct AuditMessage with transaction list

4. Update response handling:
   - `Audit_OK`: update receipts, preserve amounts
   - `Audit_UpdateTransactionsList`: validate, retry or conflict
   - `Audit_Invalid`: set conflict

## Step 6: Update CloseIncomingTrustLineTransaction

1. Same modifications as SetOutgoingTrustLineTransaction
2. Add member variables
3. Add block number retrieval step
4. Modify audit stage
5. Update response handling

## Step 7: Verification

1. Compile the project
2. Verify no regressions
3. Verify consistent behavior with AuditSourceTransaction
4. Review for code duplication opportunities

## Files to Modify

| File | Changes |
|------|---------|
| `src/core/transactions/transactions/trust_lines/SetOutgoingTrustLineTransaction.h` | Add members, method declarations |
| `src/core/transactions/transactions/trust_lines/SetOutgoingTrustLineTransaction.cpp` | Implement new audit logic |
| `src/core/transactions/transactions/trust_lines/CloseIncomingTrustLineTransaction.h` | Add members, method declarations |
| `src/core/transactions/transactions/trust_lines/CloseIncomingTrustLineTransaction.cpp` | Implement new audit logic |
| `src/core/transactions/transactions/trust_lines/base/BaseTrustLineTransaction.h` | (Optional) Add helper methods |
| `src/core/transactions/transactions/trust_lines/base/BaseTrustLineTransaction.cpp` | (Optional) Implement helpers |

# Test Plan

**Complexity Level:** Moderate

## Functional Validation

### SetOutgoingTrustLineTransaction
- Verify block number retrieval before audit
- Verify finalized receipt selection
- Verify transaction list included in AuditMessage
- Verify `Audit_OK` handling
- Verify `Audit_UpdateTransactionsList` handling with retry
- Verify observing window check for update list
- Verify `Audit_Invalid` handling
- Verify retry limit enforcement

### CloseIncomingTrustLineTransaction
- Same validations as SetOutgoingTrustLineTransaction

### Consistency
- Verify behavior matches AuditSourceTransaction for same scenarios

## Integration Validation
- Verify integration with receipt handlers
- Verify integration with TrustLinesManager
- Verify proper database persistence
- Verify trust line parameter changes still work correctly after audit

# Verification and Validation

## Architecture integrity
- Consistent audit behavior across all initiating transactions
- Code reuse where practical
- No violation of existing transaction patterns

## Security
- Same security properties as AuditSourceTransaction
- Transaction list validation in all transactions

## Performance
- No additional overhead compared to AuditSourceTransaction
- Efficient reuse of common logic

## Scalability
- Same scalability as AuditSourceTransaction

## Reliability
- Consistent error handling across transactions
- Proper conflict detection

## Maintainability
- Common logic extracted where practical
- Clear documentation of any duplication
- Consistent patterns across transactions

## Cost
- N/A (no infrastructure changes)

## Compliance
- N/A (internal protocol implementation)

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify AuditSourceTransaction or AuditTargetTransaction (done in previous tasks)
- Do not modify message classes (done in Task 18-01)
- Ensure trust line parameter modification logic (SetOutgoing/CloseIncoming) is not affected
- If code extraction is too complex, document the reason and proceed with controlled duplication

# Task ID Tracking
- Task ID: 18-06
