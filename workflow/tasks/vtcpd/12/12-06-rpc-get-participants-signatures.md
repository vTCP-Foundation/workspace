# 12-06 - RPC Method: getParticipantsSignatures Implementation

# Links
- [PRD](../../../prd/vtcpd/12-observer-ambiguous-transaction-handling.md)
- [Previous task: 12-05](12-05-rpc-check-transaction.md)
- [Observer RPC Protocol](../../../../observer/README.md)

# Description
Implement the getParticipantsSignatures method to retrieve participant vote signatures from the observer via RPCService.GetClaimVotes RPC endpoint. This method converts base64 signature strings to sphincs::Signature::Shared objects, constructs a map of signatures by PaymentNodeID, invokes the mParticipantsVotesSignal, and marks the claim as Done.

This enables payment transactions to receive the signatures needed for finalization.

# Requirements and DOD

## Functional Requirements
1. Replace stub implementation of `getParticipantsSignatures` in `src/core/observing/ObservingHandler.cpp`
2. Method signature (already declared):
   ```cpp
   void ObservingHandler::getParticipantsSignatures(ObservingPaymentClaim::Shared claim)
   ```
3. Implementation requirements:
   - Add info log: "Getting participants signatures: {transactionUUID}"
   - Check if mObservers is empty: throw NotFoundError("No observers configured")
   - Use first observer: `auto firstObserver = mObservers[0]`
   - Construct JSON-RPC request:
     - method: "RPCService.GetClaimVotes"
     - params: array with single object containing:
       - transaction_uuid (string)
       - max_claim_block_number (uint64)
     - id: 1
   - Send request via TCP (same pattern as previous methods)
   - Parse response and extract result.votes array
   - For each vote in array:
     - Extract index (PaymentNodeID)
     - Extract signature (string, base64)
     - Convert signature: `auto signature = make_shared<sphincs::Signature>(signatureBase64)`
     - Validate signature: `if (!signature->isValid()) { log warning, skip }`
     - Add to map: `signaturesMap[nodeId] = signature`
   - Log info: "Retrieved X participant signatures from observer"
   - Invoke signal: `mParticipantsVotesSignal(transactionUUID, maxBlockNumber, signaturesMap)`
   - Update claim status: `claim->setStatus(ObservingPaymentClaim::Done)`
   - Close socket
   - On exception: log error and re-throw
4. Signal signature (already exists in ObservingHandler.h):
   ```cpp
   signals::signal<void(
       const TransactionUUID&,
       BlockNumber,
       map<PaymentNodeID, sphincs::Signature::Shared>)> mParticipantsVotesSignal;
   ```

## Definition of Done
- [ ] getParticipantsSignatures stub replaced with full implementation
- [ ] Empty observers check with NotFoundError
- [ ] First observer selected correctly
- [ ] JSON-RPC request constructed per observer/README.md spec
- [ ] TCP connection and communication implemented
- [ ] Response parsed correctly
- [ ] RPC error handling implemented
- [ ] Votes array extracted from result
- [ ] Each vote parsed for index and signature
- [ ] Signatures converted from base64 to sphincs::Signature::Shared
- [ ] Invalid signatures logged and skipped
- [ ] Valid signatures added to map by PaymentNodeID
- [ ] Info log with signature count
- [ ] mParticipantsVotesSignal invoked with correct parameters
- [ ] Claim status updated to Done after signal invocation
- [ ] Network errors throw exceptions
- [ ] Empty votes array handled correctly
- [ ] Code compiles without errors or warnings
- [ ] Method integrates with processPaymentClaims timer loop

# Implementation Plan

## Step 1: Locate getParticipantsSignatures Stub
1. Open `src/core/observing/ObservingHandler.cpp`
2. Find getParticipantsSignatures stub (after checkTransaction implementation)
3. Remove TODO comment

## Step 2: Add Initial Checks and Logging
1. Add info log with transactionUUID
2. Add empty observers check
3. Get first observer

## Step 3: Construct JSON-RPC Request
1. Create request JSON:
   ```cpp
   json request = {
       {"method", "RPCService.GetClaimVotes"},
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
5. Validate result.votes exists:
   ```cpp
   if (!response.contains("result") || !response["result"].contains("votes")) {
       throw runtime_error("Invalid RPC response: missing votes");
   }
   ```

## Step 5: Parse Votes Array
1. Get votes array: `auto votesArray = response["result"]["votes"];`
2. Create signatures map: `map<PaymentNodeID, sphincs::Signature::Shared> signaturesMap;`
3. Iterate through votes:
   ```cpp
   for (const auto& vote : votesArray) {
       PaymentNodeID nodeId = vote["index"].get<PaymentNodeID>();
       string signatureBase64 = vote["signature"].get<string>();

       // Parse signature from base64
       auto signature = make_shared<sphincs::Signature>(signatureBase64);
       if (!signature->isValid()) {
           warning() << "Invalid signature from observer for node " << nodeId;
           continue;
       }

       signaturesMap[nodeId] = signature;
   }
   ```

## Step 6: Invoke Signal and Update Status
1. Log signature count:
   ```cpp
   info() << "Retrieved " << signaturesMap.size()
          << " participant signatures from observer";
   ```
2. Invoke signal:
   ```cpp
   mParticipantsVotesSignal(
       claim->transactionUUID(),
       claim->maxBlockNumberForClaiming(),
       signaturesMap);
   ```
3. Update claim status:
   ```cpp
   claim->setStatus(ObservingPaymentClaim::Done);
   ```

## Step 7: Add Exception Handling
1. Catch std::exception
2. Log error with observer address and error message
3. Re-throw for timer handler

## Step 8: Test Implementation
1. Build project in debug mode
2. Verify compilation succeeds
3. Test signal invocation (can connect test handler)
4. Verify status updated to Done

# Test Plan

**Test Type**: Moderate task - network RPC with signature parsing and signal emission

**Test Scope**: JSON construction, TCP communication, response parsing, signature conversion, signal invocation, status update.

**Validation Approach**:
- Code review to verify signature parsing logic
- Manual testing with observer returning votes
- Signal connection test to verify parameters passed correctly
- Invalid signature handling verified via logs

**Tests to be implemented in 12-08b**:
- JSON-RPC request constructed correctly
- Votes array parsed successfully
- Each vote converted to map entry
- Signatures converted from base64
- Invalid signatures logged and skipped
- Valid signatures added to map
- Empty votes array handled without error
- mParticipantsVotesSignal invoked with correct parameters
- Claim status updated to Done after signal
- Network error throws exception
- Empty mObservers throws NotFoundError
- RPC error throws exception
- Missing votes field throws exception
- Socket closed after operation

# Verification and Validation

## Architecture integrity
- Follows same RPC pattern as previous methods
- Signal mechanism enables loose coupling with payment transactions
- Signature parsing uses existing sphincs infrastructure
- Done status enables automatic cleanup

## Security
- No validation of signature validity beyond sphincs::Signature::isValid() (per PRD)
- Invalid signatures skipped (logged warning)
- No verification that signatures correspond to participant keys
- Trust observer response (per PRD scope)

## Performance
- Signature parsing per vote (linear with participant count)
- Signal invocation synchronous
- Map construction linear with valid signatures
- Network call single request per claim

## Scalability
- Handles multiple signatures per claim
- Map size proportional to participant count
- Signal emission handles arbitrary map size

## Reliability
- Invalid signature parsing doesn't crash (isValid() check)
- Continue processing even if some signatures invalid
- Status update ensures claim cleaned up
- Exception handling prevents timer crash

## Maintainability
- Clear signature parsing logic
- Warning logs for invalid signatures
- Info log confirms successful retrieval
- Consistent with previous RPC implementations

## Cost
- No cost validation required

## Compliance
- Implements observer RPC protocol exactly
- Follows PRD algorithm specification
- Signal signature matches ObservingHandler.h declaration

# Restrictions
- Commit changes only after successful compilation
- Do not validate signatures beyond isValid() check
- Do not verify signature-participant correspondence (per PRD)
- Do not batch requests
- Keep synchronous
- Invoke signal exactly once per successful response
- Update status to Done only after signal invocation
