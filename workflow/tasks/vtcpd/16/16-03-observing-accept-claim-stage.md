# 16-03 - Implement runObservingAcceptClaimStage() Method

# Links
- [PRD](../../prd/vtcpd/16-payment-transaction-observing-states.md)
- [Previous task: 16-01-enum-stages-constant](16-01-enum-stages-constant.md)
- [Previous task: 16-02-set-trust-lines-conflict](16-02-set-trust-lines-conflict.md)

# Description

This task implements the `runObservingAcceptClaimStage()` method in `BaseExchangePaymentTransaction`. This method handles the first observing state (`Observing_AcceptClaim`) where the transaction submits a claim to the observer and processes the response.

## Observer RPC Response Fields (AcceptClaim)

From `observer/README.md`:
- `result.success` (bool)
- `result.message` (string):
  - `"claim accepted successfully"` on success
  - `"claim already exists ..."` when duplicate
  - `"max_claim_block_number (...) must be less than current block number (...)"` when claim window is closed

## State Machine Behavior

**On first call (no response yet)**:
1. Create `AcceptClaimRpcRequest` with transaction data
2. Send request via `sendRpcRequest()`
3. Return `resultWaitForRpcResponse(RpcMethod::AcceptClaim, kObservingCheckPeriodMilliseconds)`

**On response received**:
- `result.success=true`: Transition to `Observing_GetClaimStatus`
- `result.success=false` with "claim already exists": Transition to `Observing_GetClaimStatus`
- `result.success=false` with "must be less than current block number":
  1. Call `setTrustLinesToConflictState()`
  2. Update transaction state to `PaymentObservingState::Conflicted`
  3. Return `resultDone()` (no rollback - reservations remain for manual resolution)
- Network error / timeout / other error:
  1. Log the error
  2. Return `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)` to retry

# Requirements and DOD

## Requirements

1. **Method Declaration**
   - Add `TransactionResult::SharedConst runObservingAcceptClaimStage()` declaration to `BaseExchangePaymentTransaction.h`
   - Method should be protected

2. **RPC Request Creation**
   - Create `AcceptClaimRpcRequest` with:
     - `mTransactionUUID`
     - `mMaximalClaimingBlockNumber`
     - `mParticipantsPublicKeys`
     - `mPublicKey` (own public key)
     - `mSignedTransaction` (own signature)

3. **Response Handling**
   - Check for RPC response in context using `rpcRequestIsValid(RpcMethod::AcceptClaim)`
   - If no response: send request and wait
   - If response received: process according to `result.success` and `result.message`

4. **State Transitions**
   - Success or "claim already exists" → set `mStep = Stages::Observing_GetClaimStatus`, return `runObservingGetClaimStatusStage()`
   - "must be less than current block number" → conflict handling → `resultDone()`
   - Error → retry with `resultAwakeAfterMilliseconds()`

5. **Logging**
   - Log entry to stage
   - Log response details
   - Log errors with context

## Definition of Done

- [ ] Method declaration added to header
- [ ] Method implementation handles all response scenarios from PRD
- [ ] RPC request created with correct parameters
- [ ] State transitions implemented correctly
- [ ] Conflict handling calls `setTrustLinesToConflictState()` and updates state
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
    TransactionResult::SharedConst runObservingAcceptClaimStage();
```

## Step 2: Review Existing RPC Infrastructure

Before implementation, review:
- `AcceptClaimRpcRequest` constructor in `src/core/network/rpc/requests/AcceptClaimRpcRequest.h`
- `AcceptClaimRpcResponse` in `src/core/network/rpc/responses/AcceptClaimRpcResponse.h`
- How `sendRpcRequest()` and `resultWaitForRpcResponse()` work (from PRD-14)
- How to access RPC response from context

## Step 3: Implement Method

File: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.cpp`

```cpp
TransactionResult::SharedConst BaseExchangePaymentTransaction::runObservingAcceptClaimStage()
{
    info() << "runObservingAcceptClaimStage";

    // Check if we have a response from previous request
    if (!rpcRequestIsValid(RpcMethod::AcceptClaim)) {
        // No response yet - send request
        info() << "Sending AcceptClaim RPC request";

        auto request = make_shared<AcceptClaimRpcRequest>(
            mTransactionUUID,
            mMaximalClaimingBlockNumber,
            mParticipantsPublicKeys,
            mPublicKey,
            mSignedTransaction);

        sendRpcRequest(request);

        return resultWaitForRpcResponse(
            RpcMethod::AcceptClaim,
            kObservingCheckPeriodMilliseconds);
    }

    // Process response
    auto response = popRpcResponse<AcceptClaimRpcResponse>();
    if (!response) {
        warning() << "Failed to get AcceptClaim response, retrying";
        return resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds);
    }

    if (response->success()) {
        info() << "AcceptClaim succeeded, transitioning to GetClaimStatus";
        mStep = Stages::Observing_GetClaimStatus;
        return runObservingGetClaimStatusStage();
    }

    // Handle failure cases based on message
    const auto& message = response->message();

    if (message.find("claim already exists") != string::npos) {
        info() << "Claim already exists, transitioning to GetClaimStatus";
        mStep = Stages::Observing_GetClaimStatus;
        return runObservingGetClaimStatusStage();
    }

    if (message.find("must be less than current block number") != string::npos) {
        warning() << "Claim window expired: " << message;

        // Set trust lines to conflict state
        setTrustLinesToConflictState();

        // Update payment transaction state
        auto ioTransaction = mStorageHandler->beginTransaction();
        mStorageHandler->paymentTransactionsHandler(ioTransaction)->updateTransactionState(
            currentTransactionUUID(),
            PaymentObservingState::Conflicted);

        info() << "Transaction set to Conflicted state";
        return resultDone();
    }

    // Unknown error - log and retry
    warning() << "AcceptClaim failed with message: " << message << ", retrying";
    return resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds);
}
```

## Step 4: Verify Required Includes

Ensure `BaseExchangePaymentTransaction.cpp` has necessary includes:
- `AcceptClaimRpcRequest.h`
- `AcceptClaimRpcResponse.h`

## Step 5: Verify Compilation

```bash
make -j$(nproc)
```

# Test Plan

**Complexity**: Moderate

This method involves RPC communication and multiple state transitions.

## Validation Approach

1. **Compilation Test**: Code must compile without warnings
2. **Code Review**: Verify all response scenarios are handled per PRD
3. **Integration Testing**:
   - Test with mock observer returning success
   - Test with mock observer returning "claim already exists"
   - Test with mock observer returning "must be less than current block number"
   - Test with network timeout

## Key Scenarios to Validate

| Scenario | Expected Behavior |
|----------|------------------|
| First call, no prior response | Send RPC request, return wait |
| Response: success=true | Transition to GetClaimStatus |
| Response: "claim already exists" | Transition to GetClaimStatus |
| Response: "must be less than current block number" | Set conflict, update state, done |
| Response: other error | Log, retry after delay |
| No response / timeout | Retry after delay |

# Verification and Validation

## Architecture integrity
- Follows async RPC pattern from PRD-14
- Uses existing RPC request/response infrastructure
- State machine transitions are explicit and logged

## Security
- No external input validation required beyond RPC response parsing
- Uses established RPC communication patterns

## Performance
- Non-blocking async RPC communication
- 60 second delay between retries balances responsiveness with observer load

## Scalability
- Single RPC request per attempt
- Retry mechanism handles temporary failures

## Reliability
- All error conditions result in retry
- Only permanent failure (expired window) results in termination
- Transaction state updated before completion

## Maintainability
- Clear separation of response handling cases
- Extensive logging for debugging
- Follows existing code patterns

## Cost
- N/A - No cost implications

## Compliance
- Follows existing code style and conventions

# Restrictions
- Commit changes only after successful compilation
- Do not modify existing RPC infrastructure
- Ensure `runObservingGetClaimStatusStage()` is at least declared before this method calls it (can be forward-declared or implemented in same task batch)
