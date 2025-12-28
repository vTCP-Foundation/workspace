# 16-04 - Implement runObservingGetClaimStatusStage() Method

# Links
- [PRD](../../prd/vtcpd/16-payment-transaction-observing-states.md)
- [Previous task: 16-01-enum-stages-constant](16-01-enum-stages-constant.md)

# Description

This task implements the `runObservingGetClaimStatusStage()` method in `BaseExchangePaymentTransaction`. This method handles the second observing state (`Observing_GetClaimStatus`) where the transaction polls the observer for claim status until finalization.

## Observer RPC Response Fields (GetClaimStatus)

From `observer/README.md`:
- `result.state` (string): `"observing"`, `"approved"`, `"rejected"`, `"not found"`
- `result.votes` (array): populated only when `state == "approved"`
- `result.signature` (string): `"REJECTED_BY_TIMEOUT"` when `state == "rejected"`, empty otherwise

## Observer Guarantees

For existing claims, the state eventually becomes `"approved"` or `"rejected"`. `"not found"` means the claim is missing and must be re-submitted.

## State Machine Behavior

**On first call (no response yet)**:
1. Create `GetClaimStatusRpcRequest` with transaction UUID and block number
2. Send request via `sendRpcRequest()`
3. Return `resultWaitForRpcResponse(RpcMethod::GetClaimStatus, kObservingCheckPeriodMilliseconds)`

**On response received**:
- `"not found"`: Transition back to `Observing_AcceptClaim`
- `"observing"`: Return `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)` to retry
- `"approved"`:
  1. Extract votes from response
  2. Call existing vote validation logic
  3. If validation succeeds: finalize via `approve()`
  4. If validation fails: log error, retry
- `"rejected"`:
  1. Update transaction state to `PaymentObservingState::RejectedByObserving`
  2. Call `rollBack()` to release reservations
  3. Return `resultDone()`
- Network error / timeout / parse error:
  1. Log the error
  2. Return `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)` to retry

# Requirements and DOD

## Requirements

1. **Method Declaration**
   - Add `TransactionResult::SharedConst runObservingGetClaimStatusStage()` declaration to `BaseExchangePaymentTransaction.h`
   - Method should be protected

2. **RPC Request Creation**
   - Create `GetClaimStatusRpcRequest` with:
     - `mTransactionUUID`
     - `mMaximalClaimingBlockNumber`

3. **Response Handling by State**
   - `"not found"` → transition to `Observing_AcceptClaim`
   - `"observing"` → sleep and retry
   - `"approved"` → extract votes, validate, finalize
   - `"rejected"` → update state, rollback, done
   - Error → log and retry

4. **Vote Extraction and Validation**
   - Extract `mParticipantsSignatures` from response votes
   - Reuse existing validation logic from `processParticipantsVotesMessage()` or similar
   - Call `approve()` if validation succeeds

5. **Logging**
   - Log entry to stage
   - Log claim status from response
   - Log vote validation results
   - Log errors with context

## Definition of Done

- [ ] Method declaration added to header
- [ ] Method implementation handles all status scenarios from PRD
- [ ] RPC request created with correct parameters
- [ ] "not found" transitions back to AcceptClaim
- [ ] "observing" results in retry after delay
- [ ] "approved" extracts votes, validates, and calls approve()
- [ ] "rejected" updates state, calls rollBack(), returns done
- [ ] Error cases result in retry after delay
- [ ] Appropriate logging added
- [ ] Code compiles without warnings

# Implementation Plan

## Step 1: Add Method Declaration

File: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.h`

Add in protected section:

```cpp
protected:
    // ... existing methods ...
    TransactionResult::SharedConst runObservingGetClaimStatusStage();
```

## Step 2: Review Existing Infrastructure

Before implementation, review:
- `GetClaimStatusRpcRequest` in `src/core/network/rpc/requests/GetClaimStatusRpcRequest.h`
- `GetClaimStatusRpcResponse` in `src/core/network/rpc/responses/GetClaimStatusRpcResponse.h`
- How votes are structured in the response
- Existing `processParticipantsVotesMessage()` for vote validation logic
- How `approve()` is called after successful validation

## Step 3: Implement Method

File: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.cpp`

```cpp
TransactionResult::SharedConst BaseExchangePaymentTransaction::runObservingGetClaimStatusStage()
{
    info() << "runObservingGetClaimStatusStage";

    // Check if we have a response from previous request
    if (!rpcRequestIsValid(RpcMethod::GetClaimStatus)) {
        // No response yet - send request
        info() << "Sending GetClaimStatus RPC request";

        auto request = make_shared<GetClaimStatusRpcRequest>(
            mTransactionUUID,
            mMaximalClaimingBlockNumber,
            mPublicKey,      // own public key for signing
            mKeysStore);     // keystore for signing

        sendRpcRequest(request);

        return resultWaitForRpcResponse(
            RpcMethod::GetClaimStatus,
            kObservingCheckPeriodMilliseconds);
    }

    // Process response
    auto response = popRpcResponse<GetClaimStatusRpcResponse>();
    if (!response) {
        warning() << "Failed to get GetClaimStatus response, retrying";
        return resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds);
    }

    const auto& state = response->state();
    info() << "Claim status: " << state;

    if (state == "not found") {
        info() << "Claim not found on observer, re-submitting";
        mStep = Stages::Observing_AcceptClaim;
        return runObservingAcceptClaimStage();
    }

    if (state == "observing") {
        debug() << "Claim still being observed, will check again";
        return resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds);
    }

    if (state == "approved") {
        info() << "Claim approved by observer, extracting votes";

        // Extract votes from response
        auto votes = response->votes();
        if (votes.empty()) {
            warning() << "Approved response but no votes received, retrying";
            return resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds);
        }

        // Set participants signatures from observer response
        mParticipantsSignatures = votes;

        // Validate signatures using existing logic
        if (!checkSignaturesAppropriate()) {
            warning() << "Signature validation failed, retrying";
            return resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds);
        }

        info() << "Votes validated successfully, approving transaction";
        return approve();
    }

    if (state == "rejected") {
        warning() << "Claim rejected by observer";

        // Update payment transaction state
        auto ioTransaction = mStorageHandler->beginTransaction();
        mStorageHandler->paymentTransactionsHandler(ioTransaction)->updateTransactionState(
            currentTransactionUUID(),
            PaymentObservingState::RejectedByObserving);

        // Release reservations
        rollBack();

        info() << "Transaction rejected by observer, rolled back";
        return resultDone();
    }

    // Unknown state - log and retry
    warning() << "Unknown claim state: " << state << ", retrying";
    return resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds);
}
```

## Step 4: Verify Required Includes

Ensure `BaseExchangePaymentTransaction.cpp` has necessary includes:
- `GetClaimStatusRpcRequest.h`
- `GetClaimStatusRpcResponse.h`

## Step 5: Review GetClaimStatusRpcRequest Constructor

The constructor may require different parameters. Check the actual constructor and adjust the request creation accordingly. Key parameters needed:
- Transaction UUID
- Block number for signature
- Public key for signing
- Keystore for signature creation

## Step 6: Verify Vote Extraction

Review `GetClaimStatusRpcResponse::votes()` return type. It should return `map<PaymentNodeID, sphincs::Signature::Shared>` to match `mParticipantsSignatures`. Adjust extraction logic if needed.

## Step 7: Verify Compilation

```bash
make -j$(nproc)
```

# Test Plan

**Complexity**: Moderate

This method involves RPC communication, vote validation, and multiple state transitions.

## Validation Approach

1. **Compilation Test**: Code must compile without warnings
2. **Code Review**: Verify all status scenarios are handled per PRD
3. **Integration Testing**:
   - Test with mock observer returning "not found"
   - Test with mock observer returning "observing"
   - Test with mock observer returning "approved" with valid votes
   - Test with mock observer returning "approved" with invalid votes
   - Test with mock observer returning "rejected"
   - Test with network timeout

## Key Scenarios to Validate

| Scenario | Expected Behavior |
|----------|------------------|
| First call, no prior response | Send RPC request, return wait |
| Response: "not found" | Transition to AcceptClaim |
| Response: "observing" | Retry after delay |
| Response: "approved", valid votes | Validate votes, call approve() |
| Response: "approved", invalid votes | Log error, retry |
| Response: "approved", empty votes | Log warning, retry |
| Response: "rejected" | Update state, rollBack(), done |
| Response: unknown state | Log warning, retry |
| No response / timeout | Retry after delay |

# Verification and Validation

## Architecture integrity
- Follows async RPC pattern from PRD-14
- Reuses existing vote validation infrastructure
- State machine transitions are explicit and logged

## Security
- Vote signatures are validated using existing `checkSignaturesAppropriate()` method
- No external input trusted without validation

## Performance
- Non-blocking async RPC communication
- 60 second delay between polls balances responsiveness with observer load
- Vote validation is performed once per approved response

## Scalability
- Single RPC request per poll
- Retry mechanism handles temporary failures
- Observer eventually resolves to approved/rejected

## Reliability
- All error conditions result in retry
- Rejected state properly rolls back transaction
- Transaction state updated before completion
- Vote validation failure results in retry (observer data may be temporarily inconsistent)

## Maintainability
- Clear separation of status handling cases
- Reuses existing validation methods
- Extensive logging for debugging

## Cost
- N/A - No cost implications

## Compliance
- Follows existing code style and conventions

# Restrictions
- Commit changes only after successful compilation
- Do not modify existing vote validation logic
- Ensure `runObservingAcceptClaimStage()` exists when this method transitions to it
