# 15-04 - CompletedPaymentsObserverMonitoringTransaction

# Links
- [PRD](../../prd/vtcpd/15-completed-payments-observer-monitoring.md)
- [Task 01 - GetClaimStatusesRpcRequest](15-01-get-claim-statuses-rpc-request-signing-fields.md)

# Description
Create the main transaction class `CompletedPaymentsObserverMonitoringTransaction` that monitors the Observer for claim requests on completed payment transactions and automatically submits vote signatures for transactions being observed.

This transaction implements a 5-state state machine:
1. GetCurrentBlock - fetch current block number from Observer
2. GetTransactionsForMonitoring - retrieve completed transactions from database
3. GetClaimStatuses - query Observer for claim statuses (with signed request)
4. ProcessClaimStatuses - submit votes for "observing" claims (with signed requests)
5. WaitForVotesSubmissionResponses - collect all RPC responses

The transaction includes private methods for serializing data before signing, using the same byte order as `BaseExchangePaymentTransaction::getSerializedReceipt`.

# Requirements and DOD

## Functional Requirements

### Class Structure
1. Create class `CompletedPaymentsObserverMonitoringTransaction` inheriting from `BaseTransaction`
2. Add new `TransactionType::Payments_CompletedPaymentsObserverMonitoring` enum value
3. Constructor takes: `StorageHandler*`, `Keystore*`, `Logger&`
4. Define constant: `kMaxTransactionsPerCycle = 100`

### State Machine
5. Implement 5 states as described in PRD
6. Use async RPC via `outgoingRpcRequestSignal` and `resultWaitForRpcResponse()`

### Signing Serialization Methods
7. Implement `serializeGetClaimStatusesForSigning()`:
   - Parameters: `const vector<pair<TransactionUUID, BlockNumber>>& claims`, `const PublicKey::Shared& publicKey`
   - Returns: `pair<BytesShared, size_t>`
   - Serialization order: claims count (uint32), claims array (sorted by UUID ascending), public key
   - Sort claims by `transactionUUID` lexicographically ascending before serialization

8. Implement `serializeSubmitClaimVotesForSigning()`:
   - Parameters: `const TransactionUUID& transactionUUID`, `BlockNumber maxClaimBlockNumber`, `const map<PaymentNodeID, Signature::Shared>& votes`, `const PublicKey::Shared& publicKey`
   - Returns: `pair<BytesShared, size_t>`
   - Serialization order: transactionUUID (16 bytes), blockNumber (8 bytes), votes count (uint32), votes (sorted by PaymentNodeID ascending), public key

9. Use same byte order as `BaseExchangePaymentTransaction::getSerializedReceipt`

### Context Management
10. Store in transaction context:
    - Current block number (from State 1)
    - Transactions list (from State 2)
    - Own public key (fetched once in State 2, reused across states)
    - Pending RPC responses map (for State 5)

### Error Handling
11. On Observer unavailable: log and finish transaction
12. On GetClaimStatuses signing error: log and finish transaction
13. On SubmitClaimVotes signing error: log, skip claim, continue with next
14. On RPC timeout: log and mark failed

### Logging
15. Log all significant events:
    - Current block number received
    - Number of transactions for monitoring
    - Claim statuses received
    - Each vote submission (success/failure)
    - Transaction completion

## Definition of Done
- [ ] Header file created: `CompletedPaymentsObserverMonitoringTransaction.h`
- [ ] Implementation file created: `CompletedPaymentsObserverMonitoringTransaction.cpp`
- [ ] TransactionType enum updated with new value
- [ ] CMakeLists.txt updated
- [ ] All 5 states implemented
- [ ] Both serialization methods implemented with correct byte order
- [ ] Sorting implemented (UUID for claims, PaymentNodeID for votes)
- [ ] Error handling implemented per requirements
- [ ] Logging implemented for all significant events
- [ ] Code compiles without warnings

# Implementation Plan

## Step 1: Add TransactionType enum value
File: `src/core/transactions/transactions/base/TransactionType.h`
Add: `Payments_CompletedPaymentsObserverMonitoring`

## Step 2: Create header file
Create `src/core/transactions/transactions/regular/payments/CompletedPaymentsObserverMonitoringTransaction.h`:

```cpp
#ifndef VTCPD_COMPLETEDPAYMENTSOBSERVERMONITORINGTRANSACTION_H
#define VTCPD_COMPLETEDPAYMENTSOBSERVERMONITORINGTRANSACTION_H

#include "../../base/BaseTransaction.h"
#include "../../../../io/storage/interfaces/StorageHandler.h"
#include "../../../../crypto/keychain.h"

class CompletedPaymentsObserverMonitoringTransaction : public BaseTransaction
{
public:
    typedef shared_ptr<CompletedPaymentsObserverMonitoringTransaction> Shared;

public:
    CompletedPaymentsObserverMonitoringTransaction(
        StorageHandler *storageHandler,
        crypto::Keystore *keystore,
        Logger &log);

    TransactionResult::SharedConst run() override;
    const string logHeader() const override;

protected:
    // State handlers
    TransactionResult::SharedConst runGetCurrentBlockStage();
    TransactionResult::SharedConst runGetTransactionsForMonitoringStage();
    TransactionResult::SharedConst runGetClaimStatusesStage();
    TransactionResult::SharedConst runProcessClaimStatusesStage();
    TransactionResult::SharedConst runWaitForVotesSubmissionResponsesStage();

    // Signing serialization methods
    pair<BytesShared, size_t> serializeGetClaimStatusesForSigning(
        const vector<pair<TransactionUUID, BlockNumber>>& claims,
        const sphincs::PublicKey::Shared& publicKey);

    pair<BytesShared, size_t> serializeSubmitClaimVotesForSigning(
        const TransactionUUID& transactionUUID,
        BlockNumber maxClaimBlockNumber,
        const map<PaymentNodeID, sphincs::Signature::Shared>& votes,
        const sphincs::PublicKey::Shared& publicKey);

protected:
    enum Stages {
        GetCurrentBlock = 1,
        GetTransactionsForMonitoring,
        GetClaimStatuses,
        ProcessClaimStatuses,
        WaitForVotesSubmissionResponses
    };

    static const uint32_t kMaxTransactionsPerCycle = 100;

protected:
    StorageHandler *mStorageHandler;
    crypto::Keystore *mKeystore;

    // Context
    BlockNumber mCurrentBlockNumber;
    vector<pair<TransactionUUID, BlockNumber>> mTransactionsForMonitoring;
    sphincs::PublicKey::Shared mOwnPublicKey;
    map<TransactionUUID, bool> mPendingResponses; // UUID -> received flag
};

#endif
```

## Step 3: Implement state machine
In `CompletedPaymentsObserverMonitoringTransaction.cpp`:

### State 1: GetCurrentBlock
- Send `GetBlockNumberRpcRequest`
- Call `resultWaitForRpcResponse()`
- On response: extract block number, move to State 2
- On error: log and return `resultDone()`

### State 2: GetTransactionsForMonitoring
- Get IOTransaction from StorageHandler
- Fetch public key: `ioTransaction->paymentKeysHandler()->getOwnPublicKey()`
- On failure: log and return `resultDone()`
- Store public key in `mOwnPublicKey`
- Call `transactionsForObserverMonitoring(mCurrentBlockNumber, kMaxTransactionsPerCycle)`
- If empty: log and return `resultDone()`
- Store in `mTransactionsForMonitoring`
- Move to State 3

### State 3: GetClaimStatuses
- Sort `mTransactionsForMonitoring` by transactionUUID
- Call `serializeGetClaimStatusesForSigning()`
- Sign with `mKeystore->signPaymentTransaction()`
- On signing error: log and return `resultDone()`
- Create `GetClaimStatusesRpcRequest` with claims, public key, signature
- Send request
- Call `resultWaitForRpcResponse()`
- On response: parse statuses
- If empty statuses: log and return `resultDone()`
- Move to State 4

### State 4: ProcessClaimStatuses
- For each status with state == "observing":
  - Get votes from `paymentParticipantsVotesHandler->participantsSignatures()`
  - Sort votes by PaymentNodeID
  - Call `serializeSubmitClaimVotesForSigning()`
  - Sign with keystore
  - On signing error: log and skip this claim
  - Create `SubmitClaimVotesRpcRequest`
  - Send request
  - Add to `mPendingResponses` with received=false
- For other statuses ("accepted", "rejected", "expired"): log and skip
- If no pending responses: return `resultDone()`
- Move to State 5

### State 5: WaitForVotesSubmissionResponses
- Check for RPC responses
- For each response: log result, mark as received in `mPendingResponses`
- If all received: log summary and return `resultDone()`
- Otherwise: return `resultWaitForRpcResponse()`

## Step 4: Implement serialization methods
Follow byte order from `BaseExchangePaymentTransaction::getSerializedReceipt`:
- Use `memcpy` for copying data
- Ensure consistent endianness

## Step 5: Update CMakeLists.txt
Add `CompletedPaymentsObserverMonitoringTransaction.cpp` to payments CMakeLists.txt

# Test Plan

**Complexity**: Complex

Unit tests will be implemented in Task 06 (Unit Tests). This task focuses on implementation only.

Expected test coverage (to be implemented in Task 06):
- Constructor stores dependencies correctly
- Transaction type is correct
- kMaxTransactionsPerCycle constant equals 100
- serializeGetClaimStatusesForSigning: correct byte order, claims sorted by UUID
- serializeSubmitClaimVotesForSigning: correct byte order, votes sorted by PaymentNodeID
- Empty transactions list finishes immediately
- Empty claim statuses finishes transaction

# Verification and Validation

## Architecture integrity
- Inherits from BaseTransaction (correct transaction hierarchy)
- Uses async RPC pattern from PRD-14
- State machine pattern consistent with other payment transactions
- Serialization follows existing patterns (BaseExchangePaymentTransaction)

## Security
- Uses existing Keystore for signing (proven security)
- Public key obtained from trusted PaymentKeysHandler
- Data serialization follows canonical order (prevents signing ambiguity)
- No raw memory leaks (uses BytesShared)

## Performance
- Async RPC prevents blocking
- Maximum 100 transactions per cycle (bounded work)
- Single public key fetch per transaction
- Efficient map-based pending response tracking

## Scalability
- Configurable transaction limit via constant
- Batch processing of claims
- Parallel RPC requests for vote submissions

## Reliability
- Graceful error handling at each state
- Continue processing on individual claim failures
- Complete response collection before finishing
- Next cycle retries failed operations

## Maintainability
- Clear state machine structure
- Separate methods for each state
- Dedicated serialization methods
- Comprehensive logging

## Cost
- N/A

## Compliance
- Follows Observer RPC protocol
- Compatible with existing payment transaction patterns

# Restrictions
- Do not implement RPC infrastructure (use existing from PRD-14)
- Do not modify existing transaction types
- Serialization must match byte order in BaseExchangePaymentTransaction::getSerializedReceipt
- Commit changes only after code compiles without warnings
