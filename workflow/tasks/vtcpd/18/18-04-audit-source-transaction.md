# 18-04 - AuditSourceTransaction Implementation

# Links
- [PRD-18: Audit Mechanism Based on Finalized Transactions](../../prd/vtcpd/18-audit-mechanism-finalized-transactions.md)
- [Previous task: 18-01-audit-messages-extension](18-01-audit-messages-extension.md)
- [Previous task: 18-02-receipt-handlers-and-zero-audit-number](18-02-receipt-handlers-and-zero-audit-number.md)
- [Previous task: 18-03-trust-lines-manager-receipts-preservation](18-03-trust-lines-manager-receipts-preservation.md)

# Description

This task implements the full audit logic on the initiator side in `AuditSourceTransaction`. The transaction must:
1. Retrieve current block number to determine observing possibility
2. Select only finalized receipts with `auditNumber = 0`
3. Build and send transaction list with the audit message
4. Handle various response scenarios including retry logic
5. Update receipt audit numbers upon successful completion

This is a complex task that significantly modifies the audit flow state machine.

# Requirements and DOD

## Requirements

### Block Number Retrieval
1. Add new step to retrieve current block number via `sendRpcRequest(make_shared<GetBlockNumberRpcRequest>(currentTransactionUUID()))`
2. If block number retrieval fails:
   - Log the error
   - Return `resultDone()`
   - Add TODO comment in code to revisit this case

### Receipt Selection
3. Replace current receipt selection with call to `getFinalizedReceiptsWithZeroAuditNumber()`
4. Build list of transaction UUIDs from selected receipts
5. Sort transaction UUIDs lexicographically by raw bytes
6. Deduplicate the sorted transaction UUID list (no duplicates)
7. Compute hash of transaction list for signature payload

### Audit Message Construction
7. Include sorted transaction UUID list in `AuditMessage`
8. Include transaction list hash in audit signature payload (after balance, before `equivalentRegistryAddress`)

### Response Handling
9. **On `Audit_OK`**:
   - Update receipt audit numbers via `updateAuditNumberByTransactionUUIDs()`
   - Update trust line balance and preserve excluded receipt amounts
   - Perform receipt updates and trust line state update atomically in a single storage transaction
   - Complete transaction successfully

10. **On `Audit_UpdateTransactionsList`**:
    - Validate that all UUIDs in response exist in original sent list
    - If validation fails (unknown UUIDs): set TL to Conflict, finish transaction
    - If first retry: exclude listed transactions, recalculate balance, retry audit
    - If second `Audit_UpdateTransactionsList`: set TL to Conflict, finish transaction
    - Verify observing is still possible for each listed transaction; if any is not observable, set TL to Conflict and finish

11. **On `Audit_Invalid`**:
    - Set trust line state to Conflict
    - Finish transaction

### State Machine Updates
12. Add retry counter member variable (max 1 retry allowed)
13. Add member to store original transaction list for validation
14. Update state machine flow per PRD diagram

## Definition of Done

- [ ] Block number retrieval step implemented
- [ ] Block number failure handling implemented (log + resultDone + TODO)
- [ ] Finalized receipt selection implemented
- [ ] Transaction list building and sorting implemented
- [ ] Transaction list deduplication implemented
- [ ] Transaction list hash computation implemented
- [ ] AuditMessage sent with transaction list
- [ ] `Audit_OK` response handling with receipt audit number updates
- [ ] Receipt updates and trust line state update are atomic on success
- [ ] `Audit_UpdateTransactionsList` response handling with validation
- [ ] `Audit_UpdateTransactionsList` retry logic (max 1 retry)
- [ ] `Audit_UpdateTransactionsList` observing window check implemented
- [ ] `Audit_Invalid` response handling (set Conflict)
- [ ] Retry counter implemented
- [ ] Original transaction list stored for validation
- [ ] Code compiles without errors
- [ ] State machine flow matches PRD diagram

# Implementation Plan

## Step 1: Analyze Existing Code

1. Read `src/core/transactions/transactions/trust_lines/AuditSourceTransaction.h`
2. Read `src/core/transactions/transactions/trust_lines/AuditSourceTransaction.cpp`
3. Understand current state machine and step flow
4. Identify where receipt selection currently happens
5. Identify where AuditMessage is constructed and sent
6. Identify response handling logic

## Step 2: Add Member Variables

1. Add `BlockNumber mCurrentBlockNumber` to store retrieved block number
2. Add `vector<TransactionUUID> mOriginalTransactionList` to store sent list for validation
3. Add `uint8_t mAuditRetryCount = 0` for retry tracking
4. Add `vector<TransactionUUID> mCurrentTransactionList` for current attempt

## Step 3: Implement Block Number Retrieval Step

1. Add new step/stage for block number retrieval
2. Send `GetBlockNumberRpcRequest`:
   ```cpp
   sendRpcRequest(make_shared<GetBlockNumberRpcRequest>(currentTransactionUUID()));
   ```
3. Handle response:
   - On success: store block number, proceed to next step
   - On failure: log error, return `resultDone()`, add TODO comment

## Step 4: Modify Receipt Selection

1. Replace current receipt loading with:
   ```cpp
   auto outgoingReceipts = mStorageHandler->outgoingPaymentReceiptHandler()->
       getFinalizedReceiptsWithZeroAuditNumber(mTrustLineID);
   auto incomingReceipts = mStorageHandler->incomingPaymentReceiptHandler()->
       getFinalizedReceiptsWithZeroAuditNumber(mTrustLineID);
   ```
2. Extract transaction UUIDs from receipts
3. Sort UUIDs using lexicographic comparison by raw bytes
4. Deduplicate the sorted list
5. Store in `mOriginalTransactionList` and `mCurrentTransactionList`

## Step 5: Calculate Balance and Amounts

1. Sum incoming receipt amounts → `includedIncoming`
2. Sum outgoing receipt amounts → `includedOutgoing`
3. Calculate new balance: `currentBalance + includedIncoming - includedOutgoing`
4. Store calculated values for later use

## Step 6: Compute Transaction List Hash

1. Use hash function from Task 18-01
2. Hash the sorted transaction list
3. Include hash in signature payload

## Step 7: Modify AuditMessage Construction

1. Pass `mCurrentTransactionList` to AuditMessage constructor
2. Update signature computation to include transaction list hash

## Step 8: Implement Response Processing

### Audit_OK Handler:
```cpp
// Update receipt audit numbers and trust line state atomically
auto ioTransaction = mStorageHandler->beginTransaction();
mStorageHandler->outgoingPaymentReceiptHandler()->
    updateAuditNumberByTransactionUUIDs(mTrustLineID, newAuditNumber, mCurrentTransactionList);
mStorageHandler->incomingPaymentReceiptHandler()->
    updateAuditNumberByTransactionUUIDs(mTrustLineID, newAuditNumber, mCurrentTransactionList);
mTrustLines->updateTrustLineTotalReceiptsAmounts(
    mContractorID,
    includedIncoming, includedOutgoing,
    excludedIncoming, excludedOutgoing);
ioTransaction->commit();
```

### Audit_UpdateTransactionsList Handler:
```cpp
auto& responseList = responseMessage->transactionUUIDs();

// Validate all UUIDs are in original list
for (const auto& uuid : responseList) {
    if (find(mOriginalTransactionList.begin(), mOriginalTransactionList.end(), uuid)
        == mOriginalTransactionList.end()) {
        // Unknown UUID - set conflict and finish
        setTrustLineToConflict();
        return resultDone();
    }
}

// Check retry count
if (mAuditRetryCount >= 1) {
    setTrustLineToConflict();
    return resultDone();
}

mAuditRetryCount++;

// Ensure observing is still possible for each listed transaction
auto ioTx = mStorageHandler->beginTransaction();
for (const auto& uuid : responseList) {
    auto effectiveBlock = ioTx->paymentTransactionsHandler()->effectiveClaimingBlockNumber(uuid);
    if (mCurrentBlockNumber > effectiveBlock) {
        setTrustLineToConflict();
        return resultDone();
    }
}

// Exclude listed transactions
excludeTransactions(responseList);

// Recalculate balance and retry
return retryAudit();
```

### Audit_Invalid Handler:
```cpp
setTrustLineToConflict();
return resultDone();
```

## Step 9: Implement Helper Methods

1. `excludeTransactions(const vector<TransactionUUID>& toExclude)`:
   - Remove UUIDs from `mCurrentTransactionList`
   - Recalculate amounts
   - Update balance calculation

2. `setTrustLineToConflict()`:
   - Set trust line state to `TrustLine::Conflict`
   - Persist to database

3. `retryAudit()`:
   - Rebuild AuditMessage with reduced transaction list
   - Send message
   - Return to response waiting state

## Step 10: Update State Machine

1. Modify step enumeration if needed
2. Update `run()` method to include new steps
3. Ensure proper state transitions

## Files to Modify

| File | Changes |
|------|---------|
| `src/core/transactions/transactions/trust_lines/AuditSourceTransaction.h` | Add member variables, method declarations |
| `src/core/transactions/transactions/trust_lines/AuditSourceTransaction.cpp` | Implement all new logic |

# Test Plan

**Complexity Level:** Complex

## Functional Validation
- Verify block number is retrieved before receipt selection
- Verify block number failure results in `resultDone()` with log entry
- Verify only finalized receipts with `auditNumber = 0` are selected
- Verify transaction list is sorted correctly
- Verify transaction list hash is included in signature
- Verify `Audit_OK` updates receipt audit numbers
- Verify `Audit_OK` preserves excluded receipt amounts
- Verify `Audit_UpdateTransactionsList` validates UUIDs against original list
- Verify `Audit_UpdateTransactionsList` rejects transactions with expired observing window
- Verify unknown UUIDs in response cause Conflict
- Verify retry limit (max 1) is enforced
- Verify `Audit_Invalid` sets trust line to Conflict

## Integration Validation
- Verify integration with receipt handlers (Task 18-02)
- Verify integration with TrustLinesManager (Task 18-03)
- Verify integration with AuditMessage (Task 18-01)
- Verify proper database persistence of all changes

## Error Handling Validation
- Verify graceful handling of block number retrieval failure
- Verify graceful handling of empty receipt list
- Verify graceful handling of database errors

# Verification and Validation

## Architecture integrity
- State machine modifications follow existing patterns
- New steps integrate with existing transaction framework
- No violation of single responsibility principle

## Security
- Transaction list hash prevents tampering
- UUID validation prevents injection attacks via `Audit_UpdateTransactionsList`
- Signature includes all relevant data

## Performance
- Block number retrieval is async (non-blocking)
- Receipt selection query is optimized (from Task 18-02)
- No unnecessary database operations

## Scalability
- Handles variable number of receipts per audit
- Transaction list size limited by practical network constraints

## Reliability
- Retry mechanism provides fault tolerance
- Conflict detection prevents silent data corruption
- Atomic updates ensure consistency

## Maintainability
- Clear step separation in state machine
- Well-documented response handling logic
- Helper methods for reusable logic

## Cost
- N/A (no infrastructure changes)

## Compliance
- N/A (internal protocol implementation)

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify AuditTargetTransaction in this task (separate task)
- Do not modify message classes (already done in Task 18-01)
- Do not modify receipt handlers (already done in Task 18-02)
- Do not modify TrustLinesManager (already done in Task 18-03)
- Ensure all state transitions are logged for debugging
