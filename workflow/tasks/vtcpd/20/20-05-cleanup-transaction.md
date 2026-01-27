# 20-05 - CleanupHistoricalCryptoDataTransaction Implementation

# Links
- [PRD-20: Cleanup Historical Crypto Data](../../prd/vtcpd/20-cleanup-historical-crypto-data.md)
- [Task 20-01: Handler Interfaces and SQLite Implementation](20-01-handler-interfaces-sqlite.md)
- [Task 20-03: Signal Infrastructure](20-03-signal-infrastructure.md)
- [Task 20-04: Signal Emission in Audit Transactions](20-04-signal-emission.md)

# Description
Create the new `CleanupHistoricalCryptoDataTransaction` that implements the complete logic for cleaning up obsolete historical crypto data (audits, debt receipts, and related payment records) from the database.

This is the core transaction of PRD-20. It is triggered by `historyCryptoDataCleanupSignal` after successful audit completion and removes data that is no longer needed for finalization through observing.

# Requirements and DOD

## Requirements

### Transaction Creation
1. Create new transaction class `CleanupHistoricalCryptoDataTransaction`
2. Add new transaction type `CleanupHistoricalCryptoDataType = 1203` to `BaseTransaction::TransactionType` enum
3. Transaction receives `ContractorID` and `SerializedEquivalent` as constructor parameters
4. Define constant `RETENTION_OFFSET = 2` in the transaction

### Core Logic
5. Get `TrustLineID` from `TrustLinesManager` using ContractorID
6. Get `current_audit_number` from `AuditHandler.getActualAuditNumber(TrustLineID)`
7. Calculate `threshold = current_audit_number - RETENTION_OFFSET`
8. If `threshold <= 0`, log the fact and return `resultDone()` without cleanup
9. Retrieve current block number via `sendRpcRequest(make_shared<GetBlockNumberRpcRequest>(currentTransactionUUID()))`
10. If RPC fails, log error and return `resultDone()`
11. Get candidate audits using `AuditHandler.auditsLessEqualThanAuditNumber(TrustLineID, threshold)`
12. Process audits sequentially from lowest to highest audit number

### Per-Audit Processing
13. For each candidate audit:
    - Get incoming receipts with matching audit_number (excluding audit_number = 0)
    - Get outgoing receipts with matching audit_number (excluding audit_number = 0)
    - Collect unique transaction UUIDs from all receipts
    - If no UUIDs (empty receipts), delete the audit and continue to next
    - For each UUID:
      - Try to get `effective_claiming_block_number` from `PaymentTransactionsHandler`
      - If `NotFoundError`, log warning and continue (missing record does not block cleanup)
      - If `effective_claiming_block_number >= current_block_number`, STOP entire cleanup (finalization through observing still possible)
    - If all UUIDs have `effective_claiming_block_number < current_block_number`, proceed with deletion

### Atomic Deletion
14. Deletion must be atomic within single DB transaction:
    - Delete incoming_payment_receipts by (TrustLineID, audit_number)
    - Delete outgoing_payment_receipts by (TrustLineID, audit_number)
    - Delete audit by (TrustLineID, audit_number)
    - For each deleted transaction UUID:
      - Check if UUID exists in any `incoming_payment_receipts` (any TrustLine)
      - Check if UUID exists in any `outgoing_payment_receipts` (any TrustLine)
      - If not found anywhere, delete from `payment_transactions` and `payment_participants_votes`
15. On error, rollback DB transaction and return `resultDone()`

### TransactionsManager Integration
16. Update `launchCleanupHistoricalCryptoDataTransaction` stub from Task 20-03 with full implementation

## Definition of Done
- [ ] `CleanupHistoricalCryptoDataTransaction.h` created
- [ ] `CleanupHistoricalCryptoDataTransaction.cpp` created
- [ ] Transaction type added to `BaseTransaction::TransactionType` enum
- [ ] `TransactionsManager::launchCleanupHistoricalCryptoDataTransaction` fully implemented
- [ ] Code compiles without errors
- [ ] Transaction follows established patterns from other trust line transactions

# Implementation Plan

## Step 1: Add Transaction Type to BaseTransaction
**File**: `src/core/transactions/transactions/base/BaseTransaction.h`

Add to `TransactionType` enum (around line 154, in General section):
```cpp
// General
NoEquivalentType = 1200,
PongReactionType = 1201,
RemoveOutdatedCryptoDataType = 1202,
CleanupHistoricalCryptoDataType = 1203  // NEW
```

## Step 2: Create Transaction Header
**File**: `src/core/transactions/transactions/trust_lines/CleanupHistoricalCryptoDataTransaction.h`

```cpp
#ifndef VTCPD_CLEANUPHISTORICALCRYPTODATATRANSACTION_H
#define VTCPD_CLEANUPHISTORICALCRYPTODATATRANSACTION_H

#include "../base/BaseTransaction.h"
#include "../../../contractors/ContractorsManager.h"
#include "../../../trust_lines/manager/TrustLinesManager.h"
#include "../../../io/storage/StorageHandler.h"

class CleanupHistoricalCryptoDataTransaction : public BaseTransaction
{
public:
    typedef shared_ptr<CleanupHistoricalCryptoDataTransaction> Shared;

    CleanupHistoricalCryptoDataTransaction(
        ContractorID contractorID,
        const SerializedEquivalent equivalent,
        ContractorsManager *contractorsManager,
        TrustLinesManager *trustLinesManager,
        StorageHandler *storageHandler,
        Logger &logger);

    TransactionResult::SharedConst run() override;

protected:
    const string logHeader() const override;

private:
    TransactionResult::SharedConst runInitializationStage();
    TransactionResult::SharedConst runBlockNumberRequestStage();
    TransactionResult::SharedConst runCleanupStage();

    bool processAuditCleanup(
        IOTransaction::Shared ioTransaction,
        const TrustLineID trustLineID,
        const AuditNumber auditNumber,
        const BlockNumber currentBlockNumber);

private:
    static const uint8_t RETENTION_OFFSET = 2;

    enum Stages {
        Initialization = 1,
        BlockNumberRequest = 2,
        Cleanup = 3,
    };

    ContractorID mContractorID;
    ContractorsManager *mContractorsManager;
    TrustLinesManager *mTrustLinesManager;
    StorageHandler *mStorageHandler;

    bool mBlockNumberRequestSent;
    BlockNumber mCurrentBlockNumber;
};

#endif //VTCPD_CLEANUPHISTORICALCRYPTODATATRANSACTION_H
```

## Step 3: Create Transaction Implementation
**File**: `src/core/transactions/transactions/trust_lines/CleanupHistoricalCryptoDataTransaction.cpp`

Implement the complete cleanup logic following the algorithm from PRD:

1. **Constructor**: Initialize all members
2. **run()**: Stage switch (Initialization → BlockNumberRequest → Cleanup)
3. **runInitializationStage()**: Validate contractor, get TrustLineID, check threshold
4. **runBlockNumberRequestStage()**: Send RPC request for current block number
5. **runCleanupStage()**: Main cleanup loop over candidate audits
6. **processAuditCleanup()**: Process single audit - check observing window, delete if eligible

Key implementation details:
- Use `receiptsByAuditNumber` to get receipts (exclude audit_number = 0 in logic)
- Use `effectiveClaimingBlockNumber` to check observing window
- Use `isContainsTransaction` to check if UUID is referenced elsewhere
- Use `deleteRecordsByAuditNumber` for batch receipt deletion
- Use `deleteAuditByNumber` for audit deletion
- Use `deleteRecord` for payment_transactions
- Use `deleteRecords` for payment_participants_votes

## Step 4: Update TransactionsManager
**File**: `src/core/transactions/manager/TransactionsManager.h`

Add include:
```cpp
#include "../transactions/trust_lines/CleanupHistoricalCryptoDataTransaction.h"
```

**File**: `src/core/transactions/manager/TransactionsManager.cpp`

Update launch method (replace stub from Task 20-03):
```cpp
void TransactionsManager::launchCleanupHistoricalCryptoDataTransaction(
    ContractorID contractorID,
    const SerializedEquivalent equivalent)
{
    try {
        auto transaction = make_shared<CleanupHistoricalCryptoDataTransaction>(
            contractorID,
            equivalent,
            mContractorsManager,
            mEquivalentsSubsystemsRouter->trustLinesManager(equivalent),
            mStorageHandler,
            mLog);

        prepareAndSchedule(
            transaction,
            false,  // regenerateUUID
            false,  // subsidiaryTransactionSubscribe
            false,  // outgoingMessagesSubscribe
            true);  // rpcRequestSubscribe
    } catch (NotFoundError &e) {
        warning() << "Can't launch CleanupHistoricalCryptoDataTransaction. "
                  << "Equivalent not found. Details: " << e.what();
    }
}
```

## Step 5: Add CMake Entry
Ensure the new .cpp file is added to the appropriate CMakeLists.txt for compilation.

# Test Plan

**Complexity**: Complex

**Validation for this task**:
- Code compiles successfully
- Transaction can be instantiated and run
- Basic scenario: cleanup transaction completes without errors when no audits need cleanup
- Log output shows expected flow through stages

**Manual testing scenarios** (via triggering audit completion):
1. Verify transaction launches after audit completion
2. Verify threshold calculation is correct
3. Verify RPC block number retrieval works
4. Verify cleanup stops when observing is still possible
5. Verify atomic deletion of audit data

# Verification and Validation

## Architecture integrity
- Transaction follows established patterns from other trust line transactions
- Proper separation of stages (Initialization, BlockNumberRequest, Cleanup)
- Uses existing handler interfaces consistently
- No direct database access; all through handlers

## Security
- No cleanup of data where finalization through observing is still possible
- Atomic operations prevent partial cleanup states
- Proper validation of contractor existence

## Performance
- Single RPC call for block number
- Batch deletion where possible
- Sequential processing prevents database lock contention

## Scalability
- Handles large numbers of historical audits
- Processes one audit at a time to limit memory usage

## Reliability
- Proper error handling for all operations
- DB transaction rollback on any failure
- Warning logs for missing payment_transactions records (does not block cleanup)
- No data loss for valid, needed records
- Stops cleanly if observing is still possible

## Maintainability
- Clear stage separation
- Well-documented algorithm in PRD
- Follows existing transaction patterns

## Cost
- N/A

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- This task depends on Task 20-01 (handler methods) and Task 20-03/20-04 (signal infrastructure)
- Do not modify other transactions except TransactionsManager
