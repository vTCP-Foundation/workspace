# 12-07 - RPC Method: rejectTransaction Implementation

# Links
- [PRD](../../../prd/vtcpd/12-observer-ambiguous-transaction-handling.md)
- [Previous task: 12-06](12-06-rpc-get-participants-signatures.md)
- [Observer RPC Protocol](../../../../observer/README.md)

# Description
Implement the rejectTransaction method to retrieve rejection signature from the observer via RPCService.GetRejectionSignature RPC endpoint. This method checks if a rejection signature is available and, if present, invokes the mRejectTransactionSignal and marks the claim as Done. If signature is not yet available, the claim remains in RejectedByObserving status for retry.

This enables payment transactions to receive rejection notifications for proper rollback.

# Requirements and DOD

## Functional Requirements
1. Replace stub implementation of `rejectTransaction` in `src/core/observing/ObservingHandler.cpp`
2. Method signature (already declared):
   ```cpp
   void ObservingHandler::rejectTransaction(ObservingPaymentClaim::Shared claim)
   ```
3. Implementation requirements:
   - Add info log: "Getting rejection signature: {transactionUUID}"
   - Check if mObservers is empty: throw NotFoundError("No observers configured")
   - Use first observer: `auto firstObserver = mObservers[0]`
   - Construct JSON-RPC request:
     - method: "RPCService.GetRejectionSignature"
     - params: array with single object containing:
       - transaction_uuid (string)
       - max_claim_block_number (uint64)
     - id: 1
   - Send request via TCP (same pattern as previous methods)
   - Parse response and extract result.state and result.signature
   - Check if signature field is non-empty:
     - If **non-empty**:
       - Log info: "Received rejection signature from observer for {transactionUUID}"
       - Invoke signal: `mRejectTransactionSignal(transactionUUID, maxBlockNumber)`
       - Update claim status: `claim->setStatus(ObservingPaymentClaim::Done)`
     - If **empty**:
       - Log debug: "No rejection signature available yet for {transactionUUID}"
       - Do NOT change status (remains RejectedByObserving)
       - Do NOT invoke signal
   - Close socket
   - On exception: log error and re-throw
4. Signal signature (already exists in ObservingHandler.h):
   ```cpp
   signals::signal<void(
       const TransactionUUID&,
       BlockNumber)> mRejectTransactionSignal;
   ```

## Definition of Done
- [ ] rejectTransaction stub replaced with full implementation
- [ ] Empty observers check with NotFoundError
- [ ] First observer selected correctly
- [ ] JSON-RPC request constructed per observer/README.md spec
- [ ] TCP connection and communication implemented
- [ ] Response parsed correctly
- [ ] RPC error handling implemented
- [ ] State and signature fields extracted from result
- [ ] Non-empty signature triggers signal invocation
- [ ] Non-empty signature updates claim status to Done
- [ ] Empty signature logged as debug
- [ ] Empty signature keeps status as RejectedByObserving (no change)
- [ ] Empty signature does NOT invoke signal
- [ ] mRejectTransactionSignal invoked with correct parameters (when signature present)
- [ ] Network errors throw exceptions
- [ ] Code compiles without errors or warnings
- [ ] Method integrates with processPaymentClaims timer loop

# Implementation Plan

## Step 1: Locate rejectTransaction Stub
1. Open `src/core/observing/ObservingHandler.cpp`
2. Find rejectTransaction stub (after getParticipantsSignatures implementation)
3. Remove TODO comment

## Step 2: Add Initial Checks and Logging
1. Add info log with transactionUUID
2. Add empty observers check
3. Get first observer

## Step 3: Construct JSON-RPC Request
1. Create request JSON:
   ```cpp
   json request = {
       {"method", "RPCService.GetRejectionSignature"},
       {"params", json::array({
           {
               {"transaction_uuid", boost::uuids::to_string(claim->transactionUUID())},
               {"max_claim_block_number", claim->maxBlockNumberForClaiming()}
           }
       })},
       {"id", 1}
   };
   ```

## Step 4: Send Request and Parse Response
1. Wrap in try-catch
2. Send via TCP (same as previous methods)
3. Parse JSON response
4. Check for RPC errors
5. Validate result exists:
   ```cpp
   if (!response.contains("result")) {
       throw runtime_error("Invalid RPC response: missing result");
   }
   ```

## Step 5: Parse State and Signature
1. Get result object: `auto result = response["result"];`
2. Extract fields with defaults:
   ```cpp
   string state = result.value("state", "");
   string signatureStr = result.value("signature", "");
   ```

## Step 6: Process Based on Signature Presence
1. Check signature non-empty:
   ```cpp
   if (!signatureStr.empty()) {
       info() << "Received rejection signature from observer for "
              << claim->transactionUUID();

       // Invoke signal
       mRejectTransactionSignal(
           claim->transactionUUID(),
           claim->maxBlockNumberForClaiming());

       // Update status to Done
       claim->setStatus(ObservingPaymentClaim::Done);
   } else {
       // No signature yet
   #ifdef DEBUG_LOG_OBSEVING_HANDLER
       debug() << "No rejection signature available yet for "
               << claim->transactionUUID();
   #endif
       // Keep status as RejectedByObserving (no change)
   }
   ```

## Step 7: Add Exception Handling
1. Catch std::exception
2. Log error with observer address and error message
3. Re-throw for timer handler

## Step 8: Test Implementation
1. Build project in debug mode
2. Verify compilation succeeds
3. Test with signature present (signal invoked, status Done)
4. Test with signature empty (no signal, status unchanged)
5. Verify retry behavior (next cycle calls method again if status still RejectedByObserving)

# Test Plan

**Test Type**: Moderate task - network RPC with conditional signal emission

**Test Scope**: JSON construction, TCP communication, response parsing, conditional logic, signal invocation, status update.

**Validation Approach**:
- Code review to verify conditional logic
- Manual testing with observer returning signature and without
- Signal connection test to verify parameters
- Retry behavior verified via debug logs

**Tests to be implemented in 12-08b**:
- JSON-RPC request constructed correctly
- Non-empty signature invokes mRejectTransactionSignal
- Non-empty signature updates status to Done
- Empty signature logs debug message
- Empty signature keeps status unchanged (RejectedByObserving)
- Empty signature does NOT invoke signal
- Signal parameters correct (transactionUUID, maxBlockNumber)
- Network error throws exception
- Empty mObservers throws NotFoundError
- RPC error throws exception
- Missing result field throws exception
- Socket closed after operation
- Retry occurs in next cycle if signature still empty

# Verification and Validation

## Architecture integrity
- Follows same RPC pattern as previous methods
- Signal mechanism enables loose coupling
- Conditional logic handles observer delay in signature generation
- Status remains RejectedByObserving enables retry

## Security
- No validation of rejection signature (per PRD - accepted as-is)
- Trust observer decision (per PRD scope)
- Signature passed to payment transaction as opaque string (not parsed)

## Performance
- Single RPC call per claim in RejectedByObserving status
- Minimal JSON parsing
- Signal invocation synchronous
- Retry via timer period (60s) acceptable

## Scalability
- Linear with number of rejected claims
- Network calls accumulate if many rejections
- Timer period limits retry frequency

## Reliability
- Empty signature enables graceful retry
- Status transition only when signature available
- Exception handling prevents timer crash
- Signal invocation only when appropriate

## Maintainability
- Clear conditional logic
- Debug logging for empty signature case
- Info logging for success case
- Consistent with previous RPC implementations

## Cost
- No cost validation required

## Compliance
- Implements observer RPC protocol exactly
- Follows PRD conditional logic specification
- Signal signature matches ObservingHandler.h declaration

# Restrictions
- Commit changes only after successful compilation
- Do not parse or validate rejection signature content
- Do not add timeout for signature availability
- Keep synchronous
- Invoke signal exactly once when signature present
- Update status to Done only when signature present
- Do NOT change status if signature empty
