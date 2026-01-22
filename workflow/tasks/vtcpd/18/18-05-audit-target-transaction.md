# 18-05 - AuditTargetTransaction Implementation

# Links
- [PRD-18: Audit Mechanism Based on Finalized Transactions](../../prd/vtcpd/18-audit-mechanism-finalized-transactions.md)
- [Previous task: 18-01-audit-messages-extension](18-01-audit-messages-extension.md)
- [Previous task: 18-02-receipt-handlers-and-zero-audit-number](18-02-receipt-handlers-and-zero-audit-number.md)
- [Previous task: 18-03-trust-lines-manager-receipts-preservation](18-03-trust-lines-manager-receipts-preservation.md)
- [Previous task: 18-04-audit-source-transaction](18-04-audit-source-transaction.md)

# Description

This task implements the full audit logic on the contractor side in `AuditTargetTransaction`. The transaction must:
1. Retrieve current block number to determine observing possibility for various scenarios
2. Select own finalized receipts with `auditNumber = 0`
3. Compare initiator's transaction list with own receipts
4. Handle desynchronization scenarios according to PRD rules
5. Respond appropriately and update local state

This is a complex task that implements the contractor's verification and reconciliation logic.

# Requirements and DOD

## Requirements

### Block Number Retrieval
1. Add step to retrieve current block number via `sendRpcRequest(make_shared<GetBlockNumberRpcRequest>(currentTransactionUUID()))`
2. If block number retrieval fails:
   - Log the error
   - Return `resultDone()`
   - Add TODO comment in code to revisit this case

### Receipt Selection
3. Select own finalized receipts via `getFinalizedReceiptsWithZeroAuditNumber()`
4. Build local transaction UUID list from selected receipts
5. Sort local list lexicographically by raw bytes
6. Deduplicate the sorted local transaction list (no duplicates)

### Transaction List Comparison
6. Compare initiator's transaction list (from `AuditMessage`) with own list
7. Identify discrepancies:
   - Transactions in initiator's list but not in own list
   - Transactions in initiator's list that are not finalized locally (but have receipts)
   - Transactions in own list but not in initiator's list

### Desynchronization Handling

8. **Initiator has unknown finalized transactions** (in initiator's list, contractor has no receipt):
   - Set trust line to Conflict
   - Respond with `Audit_Invalid`

9. **Initiator's transactions not finalized locally** (contractor has receipt but transaction not committed):
   - Check if observing is still possible: `currentBlockNumber <= effectiveClaimingBlockNumber`
   - If observing possible: respond with `Audit_UpdateTransactionsList` containing these transaction UUIDs
   - If observing not possible: set trust line to Conflict, respond with `Audit_Invalid`

10. **Contractor has extra finalized transactions** (in own list but not in initiator's list):
    - Check if observing is still possible for each extra transaction
    - If observing possible for all: exclude them from own calculation, continue with audit verification
    - If observing not possible for any: set trust line to Conflict, respond with `Audit_Invalid`

### Audit Verification (when lists match or after reconciliation)
11. Verify proposed audit data (balance, amounts) matches local calculation
12. If verification passes:
   - Sign audit response
   - Update receipt audit numbers
   - Update trust line state with preserved excluded amounts
   - Perform receipt updates and trust line state update atomically in a single storage transaction
   - Respond with `Audit_OK`
13. If verification fails:
    - Set trust line to Conflict
    - Respond with `Audit_Invalid`

### Observing Check Logic
14. For each transaction, retrieve `effectiveClaimingBlockNumber` from `payment_transactions` table
15. Observing is possible if `currentBlockNumber <= effectiveClaimingBlockNumber`
16. Observing is not possible if `currentBlockNumber > effectiveClaimingBlockNumber`

## Definition of Done

- [ ] Block number retrieval step implemented
- [ ] Block number failure handling implemented (log + resultDone + TODO)
- [ ] Own finalized receipt selection implemented
- [ ] Transaction list comparison logic implemented
- [ ] Local transaction list deduplicated
- [ ] "Initiator has unknown transactions" scenario handled → Conflict + Audit_Invalid
- [ ] "Initiator's transactions not finalized locally" scenario handled
  - [ ] Observing possible → Audit_UpdateTransactionsList
  - [ ] Observing not possible → Conflict + Audit_Invalid
- [ ] "Contractor has extra transactions" scenario handled
  - [ ] Observing possible → exclude and continue
  - [ ] Observing not possible → Conflict + Audit_Invalid
- [ ] Lists match scenario → verify and respond OK
- [ ] Audit verification logic implemented
- [ ] Receipt audit number updates on success
- [ ] Trust line state updates with preserved amounts
- [ ] Receipt updates and trust line state update are atomic on success
- [ ] Code compiles without errors
- [ ] State machine flow matches PRD diagram

# Implementation Plan

## Step 1: Analyze Existing Code

1. Read `src/core/transactions/transactions/trust_lines/AuditTargetTransaction.h`
2. Read `src/core/transactions/transactions/trust_lines/AuditTargetTransaction.cpp`
3. Understand current state machine and message handling
4. Identify where audit verification currently happens
5. Understand how responses are sent

## Step 2: Add Member Variables

1. Add `BlockNumber mCurrentBlockNumber` for retrieved block number
2. Add `vector<TransactionUUID> mOwnFinalizedTransactions` for own receipt UUIDs
3. Add `vector<TransactionUUID> mInitiatorTransactions` from received AuditMessage
4. Add `vector<TransactionUUID> mExcludedTransactions` for excluded own transactions

## Step 3: Implement Block Number Retrieval

1. Add step for block number retrieval after receiving AuditMessage
2. Send `GetBlockNumberRpcRequest`
3. Handle response:
   - On success: store block number, proceed
   - On failure: log error, return `resultDone()`, add TODO

## Step 4: Implement Receipt Selection

1. After block number retrieval, select own finalized receipts:
   ```cpp
   auto outgoingReceipts = mStorageHandler->outgoingPaymentReceiptHandler()->
       getFinalizedReceiptsWithZeroAuditNumber(mTrustLineID);
   auto incomingReceipts = mStorageHandler->incomingPaymentReceiptHandler()->
       getFinalizedReceiptsWithZeroAuditNumber(mTrustLineID);
   ```
2. Extract and sort transaction UUIDs
3. Deduplicate the sorted list
4. Store in `mOwnFinalizedTransactions`

## Step 5: Implement Transaction List Comparison

1. Extract initiator's list from AuditMessage: `mInitiatorTransactions = auditMessage->transactionUUIDs()`
2. Implement comparison logic to identify:
   - `unknownToContractor`: in initiator's list, no local receipt
   - `notFinalizedLocally`: in initiator's list, local receipt exists but not finalized
   - `extraOnContractor`: in own list, not in initiator's list

## Step 6: Implement "Unknown Transactions" Handler

```cpp
if (!unknownToContractor.empty()) {
    // Check if these are truly unknown (no receipt at all)
    for (const auto& uuid : unknownToContractor) {
        if (!hasAnyReceiptForTransaction(uuid)) {
            setTrustLineToConflict();
            sendAuditResponse(ConfirmationMessage::Audit_Invalid);
            return resultDone();
        }
    }
}
```

## Step 7: Implement "Not Finalized Locally" Handler

```cpp
if (!notFinalizedLocally.empty()) {
    vector<TransactionUUID> cannotObserve;
    vector<TransactionUUID> canObserve;

    for (const auto& uuid : notFinalizedLocally) {
        auto effectiveBlock = getEffectiveClaimingBlockNumber(uuid);
        if (mCurrentBlockNumber <= effectiveBlock) {
            canObserve.push_back(uuid);
        } else {
            cannotObserve.push_back(uuid);
        }
    }

    if (!cannotObserve.empty()) {
        setTrustLineToConflict();
        sendAuditResponse(ConfirmationMessage::Audit_Invalid);
        return resultDone();
    }

    if (!canObserve.empty()) {
        sendAuditResponse(ConfirmationMessage::Audit_UpdateTransactionsList, canObserve);
        return resultDone();
    }
}
```

## Step 8: Implement "Extra Transactions" Handler

```cpp
if (!extraOnContractor.empty()) {
    for (const auto& uuid : extraOnContractor) {
        auto effectiveBlock = getEffectiveClaimingBlockNumber(uuid);
        if (mCurrentBlockNumber > effectiveBlock) {
            setTrustLineToConflict();
            sendAuditResponse(ConfirmationMessage::Audit_Invalid);
            return resultDone();
        }
    }
    // All extras can still be observed - exclude them
    mExcludedTransactions = extraOnContractor;
}
```

## Step 9: Implement Audit Verification

1. Calculate expected balance based on matched transactions
2. Compare with initiator's proposed values
3. Verify signature payload includes correct transaction list hash
4. If match: proceed to success
5. If mismatch: Conflict + Audit_Invalid

## Step 10: Implement Success Handler

```cpp
// Update receipt audit numbers
mStorageHandler->outgoingPaymentReceiptHandler()->
    updateAuditNumberByTransactionUUIDs(mTrustLineID, newAuditNumber, matchedTransactions);
mStorageHandler->incomingPaymentReceiptHandler()->
    updateAuditNumberByTransactionUUIDs(mTrustLineID, newAuditNumber, matchedTransactions);

// Update trust line with preserved excluded amounts
mTrustLines->updateTrustLineTotalReceiptsAmounts(
    mContractorID,
    includedIncoming, includedOutgoing,
    excludedIncoming, excludedOutgoing);

// Send success response
sendAuditResponse(ConfirmationMessage::Audit_OK);
```

## Step 11: Implement Helper Methods

1. `hasAnyReceiptForTransaction(TransactionUUID uuid)` - check if any receipt exists
2. `getEffectiveClaimingBlockNumber(TransactionUUID uuid)` - get from payment_transactions
3. `setTrustLineToConflict()` - set state and persist
4. `sendAuditResponse(status, optionalTransactionList)` - send AuditResponseMessage

## Files to Modify

| File | Changes |
|------|---------|
| `src/core/transactions/transactions/trust_lines/AuditTargetTransaction.h` | Add member variables, method declarations |
| `src/core/transactions/transactions/trust_lines/AuditTargetTransaction.cpp` | Implement all new logic |

# Test Plan

**Complexity Level:** Complex

## Functional Validation
- Verify block number is retrieved after receiving AuditMessage
- Verify block number failure results in `resultDone()` with log entry
- Verify own finalized receipts are correctly selected
- Verify transaction list comparison identifies all discrepancy types
- Verify "unknown transactions" scenario results in Conflict + Audit_Invalid
- Verify "not finalized locally + observing possible" results in Audit_UpdateTransactionsList
- Verify "not finalized locally + observing not possible" results in Conflict + Audit_Invalid
- Verify "extra transactions + observing possible" excludes and continues
- Verify "extra transactions + observing not possible" results in Conflict + Audit_Invalid
- Verify successful audit updates receipt numbers and trust line state
- Verify excluded amounts are preserved correctly

## Integration Validation
- Verify integration with receipt handlers (Task 18-02)
- Verify integration with TrustLinesManager (Task 18-03)
- Verify integration with AuditResponseMessage (Task 18-01)
- Verify proper database persistence

## Error Handling Validation
- Verify graceful handling of block number retrieval failure
- Verify graceful handling of database errors
- Verify all paths result in proper response to initiator

# Verification and Validation

## Architecture integrity
- State machine modifications follow existing patterns
- Response handling integrates with existing message framework
- Proper separation of verification and state update logic

## Security
- Validates initiator's transaction list against local data
- Prevents accepting unknown transactions
- Proper conflict detection prevents data corruption

## Performance
- Block number retrieval is async
- Efficient set operations for list comparison
- Minimized database queries

## Scalability
- Handles variable transaction list sizes
- Efficient comparison algorithms

## Reliability
- All desynchronization scenarios handled
- Proper conflict state for unresolvable situations
- Atomic database updates

## Maintainability
- Clear separation of comparison and handling logic
- Well-documented decision points
- Helper methods for reusable operations

## Cost
- N/A (no infrastructure changes)

## Compliance
- N/A (internal protocol implementation)

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify AuditSourceTransaction in this task (done in Task 18-04)
- Do not modify message classes (done in Task 18-01)
- Do not modify receipt handlers (done in Task 18-02)
- Do not modify TrustLinesManager (done in Task 18-03)
- Ensure all decision points are logged for debugging
- Transaction must always send a response to initiator (no hanging)
