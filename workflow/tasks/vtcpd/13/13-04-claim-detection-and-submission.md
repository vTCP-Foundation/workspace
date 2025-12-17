# 13-04 - Observer Claim Detection and Signature Submission

# Links
- [PRD 13: Observer Successful Transactions Monitoring](../../../prd/vtcpd/13-observer-successful-transactions-monitoring.md)
- [Previous task: 13-03 Timer Infrastructure and Basic Monitoring Cycle](13-03-timer-infrastructure.md)
- [Observer RPC Protocol](../../../../observer/README.md)

# Description
Complete the successful transactions monitoring feature by implementing observer claim detection and finalized signature submission. This task adds RPC integration to query observers for relevant claims (`RPCService.GetClaimStatuses`) and submit finalized transaction signatures to help resolve those claims (`RPCService.SubmitClaimVotes`).

When a claim is detected with "observing" status, the node retrieves all participant signatures from its database and submits them to the observer, potentially helping to finalize the transaction for all network participants. The monitoring cycle continues polling until transactions age out by block number, ensuring persistent assistance even if observer statuses change.

# Requirements and DOD

## Functional Requirements

### 1. Claim Detection Method
**File:** `src/core/observing/ObservingHandler.cpp`

Implement new private method `checkForRelevantClaims`:

```cpp
/**
 * Checks observer for claims related to node's successful transactions.
 * Queries observer via RPCService.GetClaimStatuses (bulk query).
 * For each claim with status "observing", calls submitFinalizedSignatures.
 *
 * @param transactions Vector of (TransactionUUID, BlockNumber) pairs to check
 */
void checkForRelevantClaims(
    const vector<pair<TransactionUUID, BlockNumber>>& transactions);
```

**Behavior:**
1. Check if `mObservers` is empty → log warning and return
2. Get first observer from `mObservers`
3. Construct bulk RPC request for `RPCService.GetClaimStatuses`:
   - Create JSON array with transaction queries
   - Each query: `{"transaction_uuid": string, "max_claim_block_number": BlockNumber}`
4. Send RPC request to observer (TCP connection)
5. Parse JSON response
6. Check for RPC errors → log and return (don't crash cycle)
7. Extract `result.statuses` array from response
8. If transaction UUID absent in response → treat as "no claims" (skip)
9. For each claim in response:
   - Extract: `transaction_uuid`, `max_claim_block_number`, `state`
   - If `state == "observing"` → call `submitFinalizedSignatures(uuid, blockNumber)`
   - If `state != "observing"` → log status and skip submission
10. Log number of relevant claims found
11. Close socket
12. On any exception: log error, continue (don't break monitoring cycle)

**RPC Request Format:**
```json
{
  "method": "RPCService.GetClaimStatuses",
  "params": [{
    "claims": [
      {"transaction_uuid": "uuid-string", "max_claim_block_number": 1000},
      ...
    ]
  }],
  "id": 1
}
```

**RPC Response Format:**
```json
{
  "result": {
    "statuses": [
      {"transaction_uuid": "uuid-string", "max_claim_block_number": 1000, "state": "observing"},
      ...
    ]
  },
  "error": null,
  "id": 1
}
```

### 2. Signature Submission Method
**File:** `src/core/observing/ObservingHandler.cpp`

Implement new private method `submitFinalizedSignatures`:

```cpp
/**
 * Submits finalized participant signatures to observer for claim resolution.
 * Retrieves signatures from database and sends via RPCService.SubmitClaimVotes.
 *
 * @param transactionUUID Transaction identifier
 * @param maxBlockNumber Max block number for claiming
 */
void submitFinalizedSignatures(
    const TransactionUUID& transactionUUID,
    BlockNumber maxBlockNumber);
```

**Behavior:**
1. Log submission start
2. Get IO transaction and retrieve participant signatures:
   - Call `paymentParticipantsVotesHandler()->participantsSignatures(transactionUUID)`
   - Returns: `map<PaymentNodeID, Signature::Shared>`
3. If signatures empty → log warning and return (no submission)
4. Log number of signatures retrieved
5. Construct votes array:
   - For each (nodeId, signature): create JSON object `{"index": nodeId, "signature": signature->toString()}`
6. Construct RPC request for `RPCService.SubmitClaimVotes`:
   - Include: `transaction_uuid`, `max_claim_block_number`, `votes` array
   - Set `public_key` to empty string with TODO comment
   - Set `signature` to empty string with TODO comment
7. Check if `mObservers` is empty → log warning and return
8. Get first observer from `mObservers`
9. Send RPC request to observer (TCP connection)
10. Parse JSON response
11. Check for RPC errors → log and return
12. Extract `result.success` and `result.message`
13. If `success == true` → log success
14. If `success == false` → log warning with message
15. Close socket
16. On any exception: log error, continue (don't break monitoring cycle)

**RPC Request Format:**
```json
{
  "method": "RPCService.SubmitClaimVotes",
  "params": [{
    "transaction_uuid": "uuid-string",
    "max_claim_block_number": 1000,
    "votes": [
      {"index": 0, "signature": "base64-signature"},
      {"index": 1, "signature": "base64-signature"}
    ],
    "public_key": "",
    "signature": ""
  }],
  "id": 1
}
```

**TODO Comments:**
```cpp
// TODO: Populate public_key with node's public key for authentication
{"public_key", ""},

// TODO: Populate signature with submission signature for verification
{"signature", ""}
```

**RPC Response Format:**
```json
{
  "result": {
    "success": true,
    "message": "votes submitted successfully"
  },
  "error": null,
  "id": 1
}
```

### 3. Integration with Monitoring Cycle
**File:** `src/core/observing/ObservingHandler.cpp`

Update `monitorSuccessfulTransactions()` method (Task 13-03):

Replace TODO comment:
```cpp
// Step 3: Check for relevant claims (placeholder for Task 13-04)
// TODO: Implement checkForRelevantClaims(transactions) in Task 13-04
```

With actual call:
```cpp
// Step 3: Check for relevant claims
checkForRelevantClaims(transactions);
```

### 4. Method Declarations
**File:** `src/core/observing/ObservingHandler.h`

Add to private methods section:

```cpp
/**
 * Checks observer for claims related to node's successful transactions.
 */
void checkForRelevantClaims(
    const vector<pair<TransactionUUID, BlockNumber>>& transactions);

/**
 * Submits finalized participant signatures to observer.
 */
void submitFinalizedSignatures(
    const TransactionUUID& transactionUUID,
    BlockNumber maxBlockNumber);
```

### 5. Debug Logging
Add debug logging under `#ifdef DEBUG_LOG_OBSEVING_HANDLER`:
- Bulk claims query sending
- Claims response received
- Number of relevant claims found
- Votes submission sending
- Votes submission response received
- Signature retrieval count

## Definition of Done
- [ ] `checkForRelevantClaims()` method declared in header
- [ ] `submitFinalizedSignatures()` method declared in header
- [ ] `checkForRelevantClaims()` implemented with:
  - Bulk RPC request construction
  - TCP connection to observer
  - JSON parsing and error handling
  - Status filtering ("observing" only)
  - Absent UUID handling (skip submission)
  - Non-observing status logging
  - Exception handling (log and continue)
- [ ] `submitFinalizedSignatures()` implemented with:
  - Signature retrieval from database
  - Empty signature handling
  - Votes array construction
  - RPC request construction with TODO comments
  - TCP connection to observer
  - Response parsing and success/failure handling
  - Exception handling (log and continue)
- [ ] Integration with `monitorSuccessfulTransactions()` completed (TODO replaced with call)
- [ ] Code compiles without errors or warnings
- [ ] Debug logging added for troubleshooting
- [ ] RPC protocol follows observer/README.md specification
- [ ] Error handling ensures monitoring cycle continues despite failures
- [ ] TODO comments present for `public_key` and `signature` fields

# Implementation Plan

## Step 1: Add Method Declarations to Header
**File:** `src/core/observing/ObservingHandler.h`

1. Locate private methods section
2. Add two method declarations after existing methods
3. Include parameter documentation comments

## Step 2: Implement checkForRelevantClaims Method
**File:** `src/core/observing/ObservingHandler.cpp`

### Sub-step 2.1: Method skeleton and observer check
1. Create method implementation at end of file
2. Add debug log for method entry
3. Check `mObservers.empty()` → log warning and return

### Sub-step 2.2: Prepare bulk RPC request
1. Get first observer: `auto firstObserver = mObservers[0];`
2. Create `json::array_t claimsArray;`
3. Loop through transactions:
   - Convert UUID to string: `boost::uuids::to_string(uuid)`
   - Create JSON object: `{"transaction_uuid": uuidStr, "max_claim_block_number": blockNumber}`
   - Push to claimsArray
4. Create request JSON:
   ```cpp
   json request = {
       {"method", "RPCService.GetClaimStatuses"},
       {"params", json::array({
           {{"claims", claimsArray}}
       })},
       {"id", 1}
   };
   ```
5. Serialize: `string requestStr = request.dump() + "\n";`

### Sub-step 2.3: Send RPC request via TCP
1. Wrap in try-catch block
2. Get IOCtx reference: `auto &ioCtx = static_cast<IOCtx &>(mSuccessfulTransactionsMonitorTimer.get_executor().context());`
3. Create resolver: `tcp::resolver resolver(ioCtx);`
4. Resolve endpoint: `auto endpoints = resolver.resolve(firstObserver->host(), to_string(firstObserver->port()), errorCode);`
5. Check errorCode → throw if error
6. Create socket: `tcp::socket socket(ioCtx);`
7. Connect: `boost::asio::connect(socket, endpoints, errorCode);`
8. Check errorCode → throw if error
9. Add debug log: "Sending bulk claims query to observer"
10. Write request: `boost::asio::write(socket, boost::asio::buffer(requestStr));`
11. Read response: `boost::asio::read_until(socket, responseBuffer, '\n', errorCode);`
12. Check errorCode (allow EOF)
13. Extract response line from buffer

**Reference pattern:** See `getActualBlockNumber()` method for TCP/RPC pattern (lines 150-200 approx)

### Sub-step 2.4: Parse response and process claims
1. Add debug log: "Received bulk claims response"
2. Parse JSON: `json response = json::parse(responseLine);`
3. Check for RPC error:
   ```cpp
   if (response.contains("error") && !response["error"].is_null()) {
       string errorMsg = response["error"].is_string()
           ? response["error"].get<string>()
           : response["error"].dump();
       throw runtime_error("Observer RPC error: " + errorMsg);
   }
   ```
4. Validate response structure:
   ```cpp
   if (!response.contains("result") || !response["result"].contains("statuses")) {
       warning() << "Invalid RPC response: missing statuses";
       socket.close();
       return;
   }
   ```
5. Get statuses array: `auto statusesArray = response["result"]["statuses"];`
6. Check if array: `if (!statusesArray.is_array()) { warning() << ...; return; }`
7. Log claims found: `info() << "Found " << statusesArray.size() << " relevant claims";`

### Sub-step 2.5: Process each claim
1. Loop through statusesArray:
   ```cpp
   for (const auto& claimStatus : statusesArray) {
       try {
           string uuidStr = claimStatus.at("transaction_uuid").get<string>();
           BlockNumber blockNumber = claimStatus.at("max_claim_block_number").get<BlockNumber>();
           string state = claimStatus.at("state").get<string>();

           if (state == "observing") {
               TransactionUUID uuid(uuidStr);
               submitFinalizedSignatures(uuid, blockNumber);
           } else {
               // Log non-observing status
               debug() << "Transaction " << uuidStr << " has non-observing status: " << state;
           }
       } catch (const std::exception &e) {
           warning() << "Error processing claim status: " << e.what();
           continue;
       }
   }
   ```
2. Close socket
3. Outer catch block:
   ```cpp
   } catch (const std::exception &e) {
       error() << "Failed to check for relevant claims from observer "
               << firstObserver->fullAddress() << ": " << e.what();
   }
   ```

## Step 3: Implement submitFinalizedSignatures Method
**File:** `src/core/observing/ObservingHandler.cpp`

### Sub-step 3.1: Method skeleton and signature retrieval
1. Create method implementation
2. Log start: `info() << "Submitting finalized signatures for transaction: " << transactionUUID;`
3. Wrap in try-catch block
4. Get IO transaction and retrieve signatures:
   ```cpp
   auto ioTransaction = mStorageHandler->beginTransaction();
   auto participantsSignatures = ioTransaction->paymentParticipantsVotesHandler()
       ->participantsSignatures(transactionUUID);
   ```
5. Check if empty:
   ```cpp
   if (participantsSignatures.empty()) {
       warning() << "No participant signatures found for transaction: " << transactionUUID;
       return;
   }
   ```
6. Add debug log: `debug() << "Retrieved " << participantsSignatures.size() << " participant signatures";`

### Sub-step 3.2: Construct votes array
1. Create votes array:
   ```cpp
   json::array_t votesArray;
   for (const auto& [nodeId, signature] : participantsSignatures) {
       json vote = {
           {"index", nodeId},
           {"signature", signature->toString()}
       };
       votesArray.push_back(vote);
   }
   ```

### Sub-step 3.3: Construct RPC request
1. Create request JSON:
   ```cpp
   json request = {
       {"method", "RPCService.SubmitClaimVotes"},
       {"params", json::array({
           {
               {"transaction_uuid", boost::uuids::to_string(transactionUUID)},
               {"max_claim_block_number", maxBlockNumber},
               {"votes", votesArray},
               {"public_key", ""},  // TODO: populate with node's public key
               {"signature", ""}    // TODO: populate with submission signature
           }
       })},
       {"id", 1}
   };
   ```
2. Add TODO comments as shown above

### Sub-step 3.4: Send RPC request via TCP
1. Check observers: `if (mObservers.empty()) { warning() << ...; return; }`
2. Get first observer: `auto firstObserver = mObservers[0];`
3. Serialize request: `string requestStr = request.dump() + "\n";`
4. Follow same TCP pattern as Step 2.3:
   - Get IOCtx
   - Resolve endpoint
   - Connect socket
   - Write request
   - Read response
5. Add debug log: "Sending votes submission request"
6. Add debug log: "Received votes submission response"

### Sub-step 3.5: Parse response and handle result
1. Parse JSON: `json response = json::parse(responseLine);`
2. Check for RPC error (same pattern as Step 2.4)
3. Validate result:
   ```cpp
   if (!response.contains("result")) {
       throw runtime_error("Invalid RPC response: missing result");
   }
   ```
4. Get result: `auto result = response["result"];`
5. Get success: `bool success = result.value("success", false);`
6. Handle success:
   ```cpp
   if (success) {
       info() << "Successfully submitted finalized signatures for: " << transactionUUID;
   } else {
       string message = result.value("message", "unknown error");
       warning() << "Observer rejected signature submission: " << message;
   }
   ```
7. Close socket
8. Outer catch block:
   ```cpp
   } catch (const std::exception &e) {
       error() << "Failed to submit finalized signatures: " << e.what();
   }
   ```

## Step 4: Integrate with Monitoring Cycle
**File:** `src/core/observing/ObservingHandler.cpp`

1. Locate `monitorSuccessfulTransactions()` method
2. Find TODO comment: `// TODO: Implement checkForRelevantClaims(transactions) in Task 13-04`
3. Replace with: `checkForRelevantClaims(transactions);`
4. Remove TODO comment

## Step 5: Build and Test
1. Build project: `cmake --build build`
2. Verify compilation succeeds
3. If test environment available:
   - Run with observer service active
   - Verify RPC requests sent successfully
   - Check logs for claim detection and signature submission
   - Verify no crashes on RPC errors or network issues

## Step 6: Code Review Checklist
- [ ] RPC request formats match observer/README.md specification
- [ ] JSON construction uses correct field names
- [ ] TCP connection pattern follows existing code
- [ ] Error handling prevents monitoring cycle failure
- [ ] TODO comments present for public_key and signature
- [ ] Debug logging sufficient for troubleshooting
- [ ] No memory leaks (socket closed in all paths)
- [ ] Exception handling at appropriate levels

# Test Plan

**Testing approach:** Manual/integration testing with running observer service. Focus on RPC integration correctness and error handling.

## Manual Integration Tests

### Test 1: Successful Claim Detection and Submission
**Setup:**
- Observer service running
- Node has successful transaction in database (state=0, blockNumber > current)
- Observer has claim for that transaction with status "observing"

**Execute:**
- Wait for monitoring cycle (60s or 10s in test mode)

**Verify:**
- Logs show: "Sending bulk claims query to observer"
- Logs show: "Received bulk claims response"
- Logs show: "Found 1 relevant claims"
- Logs show: "Submitting finalized signatures for transaction: [uuid]"
- Logs show: "Successfully submitted finalized signatures"
- Observer receives votes successfully

### Test 2: No Claims Available
**Setup:**
- Observer service running
- Node has successful transactions in database
- Observer has NO claims for those transactions

**Execute:**
- Wait for monitoring cycle

**Verify:**
- Logs show: "Found 0 relevant claims"
- No signature submission attempted
- Cycle completes normally
- Timer reschedules

### Test 3: Non-Observing Status Handling
**Setup:**
- Observer has claim with status "finalized" or "rejected"

**Execute:**
- Wait for monitoring cycle

**Verify:**
- Logs show non-observing status
- No signature submission attempted
- Transaction continues polling in next cycle
- No errors logged

### Test 4: Observer Connection Failure
**Setup:**
- Observer service NOT running or unreachable

**Execute:**
- Wait for monitoring cycle

**Verify:**
- Error logged: "Failed to check for relevant claims"
- Monitoring cycle continues (timer reschedules)
- No crash
- Next cycle attempts connection again

### Test 5: Empty Signatures in Database
**Setup:**
- Claim exists with "observing" status
- No signatures in payment_participants_votes table for transaction

**Execute:**
- Wait for monitoring cycle

**Verify:**
- Warning logged: "No participant signatures found"
- No RPC submission attempted
- Cycle continues normally

### Test 6: Observer Rejects Submission
**Setup:**
- Observer returns `{"success": false, "message": "invalid votes"}`

**Execute:**
- Submit signatures

**Verify:**
- Warning logged: "Observer rejected signature submission: invalid votes"
- No exception thrown
- Cycle continues normally

### Test 7: Invalid RPC Response Format
**Setup:**
- Observer returns malformed JSON or missing fields

**Execute:**
- Wait for monitoring cycle

**Verify:**
- Warning/error logged about invalid response
- Exception caught and logged
- Cycle continues (timer reschedules)
- No crash

### Test 8: Multiple Transactions with Mixed Results
**Setup:**
- 3 transactions in database
- Observer has claims for 2 (both "observing")
- Signatures available for 1, missing for other

**Execute:**
- Wait for monitoring cycle

**Verify:**
- 2 claims detected
- 1 signature submission successful
- 1 signature submission skipped (no signatures)
- Cycle completes normally

## Regression Testing
- Verify existing ObservingHandler functionality unchanged
- Verify other timers (mPaymentClaimsTimer, etc.) still work
- Verify no performance degradation in other operations

# Verification and Validation

**Complexity Level:** Moderate (RPC integration, network communication, error handling)

## Architecture integrity
- [ ] RPC protocol follows observer/README.md specification
- [ ] Integration with ObservingHandler follows existing patterns
- [ ] No changes to core observer architecture
- [ ] TCP connection pattern consistent with existing code
- [ ] JSON library usage consistent with project

## Security
- [ ] RPC requests use parameterized queries (no injection)
- [ ] No sensitive data in logs (only UUIDs and counts)
- [ ] TODO markers for future authentication (public_key, signature)
- [ ] Observer responses validated before use
- [ ] No buffer overflows in JSON parsing

## Performance
- [ ] Bulk RPC query reduces network overhead (one request for multiple transactions)
- [ ] TCP connection reused within method (not per transaction)
- [ ] No blocking operations (async IO context)
- [ ] Exception handling doesn't degrade performance
- [ ] JSON parsing efficient for typical response sizes

## Scalability
- [ ] Bulk query scales to 100 transactions per cycle
- [ ] Individual signature submissions independent (failure doesn't block others)
- [ ] Network timeout prevents indefinite waits
- [ ] Transaction processing bounded by kMaxTransactionsPerMonitoringCycle

## Reliability
- [ ] Network failures don't crash monitoring cycle
- [ ] RPC errors handled gracefully (log and continue)
- [ ] Invalid responses logged and skipped
- [ ] Empty signatures handled without errors
- [ ] Socket cleanup in all code paths (even on exceptions)
- [ ] Persistent polling ensures eventual delivery (if observer available)

## Maintainability
- [ ] Clear method names and documentation
- [ ] Debug logging aids troubleshooting
- [ ] TODO comments mark future enhancements
- [ ] Error messages descriptive and actionable
- [ ] Code follows existing RPC patterns (easy to understand)

## Cost
- [ ] Network overhead minimal (bulk query, single TCP connection per cycle)
- [ ] No additional infrastructure required
- [ ] Observer service cost unchanged
- [ ] Bandwidth usage proportional to transaction count (bounded by limit)

## Compliance
- [ ] RPC protocol matches observer specification
- [ ] JSON format follows documented structure
- [ ] No deviations from established patterns
- [ ] Error handling follows project standards

# Restrictions
- Commit changes only after successful compilation and basic testing
- Do not modify observer RPC protocol (follow observer/README.md)
- Do not implement retry logic for individual transactions (periodic polling provides retries)
- Do not update database observing_state in this task (out of scope)
- Keep public_key and signature fields empty with TODO comments (no authentication in this iteration)
- Do not add multi-observer support (use first observer only)
- Follow existing RPC patterns from ObservingHandler methods
