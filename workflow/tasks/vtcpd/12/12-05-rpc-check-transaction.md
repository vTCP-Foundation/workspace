# 12-05 - RPC Method: checkTransaction Implementation

# Links
- [PRD](../../../prd/vtcpd/12-observer-ambiguous-transaction-handling.md)
- [Previous task: 12-04](12-04-rpc-send-claim.md)
- [Observer RPC Protocol](../../../../observer/README.md)

# Description
Implement the checkTransaction method to query observer for claim status via RPCService.GetClaimStatus RPC endpoint. This method checks the current state of a previously submitted claim and updates the claim's status accordingly: "not found" → NoInfo, "observing" → unchanged, "approved" → ParticipantsVotesPresent, "rejected" → RejectedByObserving.

This enables the state machine to progress claims through the observer protocol automatically.

# Requirements and DOD

## Functional Requirements
1. Replace stub implementation of `checkTransaction` in `src/core/observing/ObservingHandler.cpp`
2. Method signature (already declared):
   ```cpp
   void ObservingHandler::checkTransaction(ObservingPaymentClaim::Shared claim)
   ```
3. Implementation requirements:
   - Add debug log: "Checking transaction status: {transactionUUID}"
   - Check if mObservers is empty: throw NotFoundError("No observers configured")
   - Use first observer: `auto firstObserver = mObservers[0]`
   - Construct JSON-RPC request:
     - method: "RPCService.GetClaimStatus"
     - params: array with single object containing:
       - transaction_uuid (string)
       - max_claim_block_number (uint64)
     - id: 1
   - Send request via TCP (same pattern as sendClaim)
   - Parse response and extract result.state
   - Update claim status based on state value:
     - "not found" → `claim->setStatus(ObservingPaymentClaim::NoInfo)`, log info
     - "observing" → no status change, log debug (remains Observing)
     - "approved" → `claim->setStatus(ObservingPaymentClaim::ParticipantsVotesPresent)`, log info
     - "rejected" → `claim->setStatus(ObservingPaymentClaim::RejectedByObserving)`, log info
     - unknown state → log warning, no status change
   - Close socket
   - On exception: log error and re-throw
4. Use same includes as sendClaim (already present)

## Definition of Done
- [ ] checkTransaction stub replaced with full implementation
- [ ] Empty observers check with NotFoundError
- [ ] First observer selected correctly
- [ ] JSON-RPC request constructed per observer/README.md spec
- [ ] TCP connection and communication implemented
- [ ] Response parsed correctly
- [ ] RPC error handling implemented
- [ ] "not found" state updates to NoInfo with logging
- [ ] "observing" state keeps status unchanged with debug log
- [ ] "approved" state updates to ParticipantsVotesPresent with logging
- [ ] "rejected" state updates to RejectedByObserving with logging
- [ ] Unknown state logged as warning without status change
- [ ] Network errors throw exceptions
- [ ] Debug and info logging added
- [ ] Code compiles without errors or warnings
- [ ] Method integrates with processPaymentClaims timer loop

# Implementation Plan

## Step 1: Locate checkTransaction Stub
1. Open `src/core/observing/ObservingHandler.cpp`
2. Find checkTransaction stub (after sendClaim implementation)
3. Remove TODO comment

## Step 2: Add Initial Checks and Logging
1. Add debug log with transactionUUID
2. Add empty observers check (same as sendClaim)
3. Get first observer

## Step 3: Construct JSON-RPC Request
1. Create request JSON:
   ```cpp
   json request = {
       {"method", "RPCService.GetClaimStatus"},
       {"params", json::array({
           {
               {"transaction_uuid", boost::uuids::to_string(claim->transactionUUID())},
               {"max_claim_block_number", claim->maxBlockNumberForClaiming()}
           }
       })},
       {"id", 1}
   };
   ```

## Step 4: Send Request via TCP
1. Wrap in try-catch
2. Create resolver, resolve address
3. Create socket, connect
4. Write request + newline
5. Read response until newline
6. Parse response JSON

## Step 5: Process Response State
1. Check for RPC error (same as sendClaim)
2. Validate result and state fields:
   ```cpp
   if (!response.contains("result") || !response["result"].contains("state")) {
       throw runtime_error("Invalid RPC response: missing state");
   }
   ```
3. Extract state string:
   ```cpp
   string state = response["result"]["state"].get<string>();
   ```
4. Implement state machine:
   ```cpp
   if (state == "not found") {
       claim->setStatus(ObservingPaymentClaim::NoInfo);
       info() << "Claim not found on observer, will retry: " << claim->transactionUUID();
   } else if (state == "observing") {
   #ifdef DEBUG_LOG_OBSEVING_HANDLER
       debug() << "Claim still observing: " << claim->transactionUUID();
   #endif
   } else if (state == "approved") {
       claim->setStatus(ObservingPaymentClaim::ParticipantsVotesPresent);
       info() << "Claim approved by observer: " << claim->transactionUUID();
   } else if (state == "rejected") {
       claim->setStatus(ObservingPaymentClaim::RejectedByObserving);
       info() << "Claim rejected by observer: " << claim->transactionUUID();
   } else {
       warning() << "Unknown claim state from observer: " << state;
   }
   ```

## Step 6: Add Exception Handling
1. Catch std::exception
2. Log error with observer address and error message
3. Re-throw for timer handler

## Step 7: Test Implementation
1. Build project in debug mode
2. Verify compilation succeeds
3. Test with mock responses (if observer available)
4. Verify status transitions occur correctly

# Test Plan

**Test Type**: Moderate task - network RPC integration with state machine

**Test Scope**: JSON construction, TCP communication, response parsing, status transitions, logging.

**Validation Approach**:
- Code review to verify state machine logic
- Manual testing with observer returning different states
- Debug logs confirm state transitions
- Exception handling tested via network failures

**Tests to be implemented in 12-08b**:
- JSON-RPC request constructed correctly
- "not found" response updates status to NoInfo
- "observing" response keeps status unchanged
- "approved" response updates status to ParticipantsVotesPresent
- "rejected" response updates status to RejectedByObserving
- Unknown state logged as warning, no status change
- Network error throws exception
- Empty mObservers throws NotFoundError
- RPC error in response throws exception
- Missing state field throws exception
- First observer selected correctly
- Socket closed after operation

# Verification and Validation

## Architecture integrity
- Follows same RPC pattern as sendClaim
- State machine logic matches PRD specification exactly
- Integrates with processPaymentClaims timer loop
- Maintains single-threaded execution

## Security
- No validation of observer state responses (per PRD)
- Uses first observer only (no quorum)
- Network communication unencrypted

## Performance
- Synchronous RPC call per claim in Observing status
- Minimal JSON parsing overhead
- Network latency impacts processing cycle
- State transitions in-memory (fast)

## Scalability
- Scales with number of Observing claims
- Network calls can accumulate if many claims stuck in Observing
- Timer period (60s) limits check frequency

## Reliability
- Exception handling prevents timer crash
- Status transitions atomic (single assignment)
- "not found" enables retry via NoInfo loop
- Unknown states logged without crashing

## Maintainability
- Clear state machine logic
- Comprehensive logging for all transitions
- Error messages aid debugging
- Consistent with sendClaim implementation

## Cost
- No cost validation required

## Compliance
- Implements observer RPC protocol exactly
- Follows PRD state machine specification
- Uses project-standard error handling

# Restrictions
- Commit changes only after successful compilation
- Do not add caching of state responses
- Do not add batching (one claim per request)
- Do not validate state string values beyond switch logic
- Keep synchronous (no async operations)
- Do not add artificial delays or backoff
