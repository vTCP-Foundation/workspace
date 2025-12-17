# 12-04 - RPC Method: sendClaim Implementation

# Links
- [PRD](../../../prd/vtcpd/12-observer-ambiguous-transaction-handling.md)
- [Previous task: 12-03](12-03-periodic-timer-processing.md)
- [Observer RPC Protocol](../../../../observer/README.md)

# Description
Implement the sendClaim method to submit payment claims to the observer via RPCService.AcceptClaim RPC endpoint. This method constructs a JSON-RPC request with claim details, sends it to the first configured observer, and updates claim status to Observing on success.

The method converts cryptographic keys and signatures to base64 format using toString() and handles network errors by throwing exceptions for retry in the next processing cycle.

# Requirements and DOD

## Functional Requirements
1. Replace stub implementation of `sendClaim` in `src/core/observing/ObservingHandler.cpp`
2. Method signature (already declared):
   ```cpp
   void ObservingHandler::sendClaim(ObservingPaymentClaim::Shared claim)
   ```
3. Implementation requirements:
   - Add debug log: "Sending claim to observer: {transactionUUID}"
   - Check if mObservers is empty: throw NotFoundError("No observers configured")
   - Use first observer: `auto firstObserver = mObservers[0]`
   - Construct JSON-RPC request:
     - method: "RPCService.AcceptClaim"
     - params: array with single object containing:
       - transaction_uuid (string from boost::uuids::to_string)
       - max_claim_block_number (uint64)
       - participants (JSON array of objects with index and public_key)
       - public_key (string from publicKey->toString())
       - signature (string from signature->toString())
     - id: 1
   - Send request using TCP socket (follow ObserverRPCClient::getBlockNumber pattern):
     - Resolve observer address
     - Connect socket
     - Write JSON + newline
     - Read response until newline
     - Parse JSON response
   - Check for RPC errors in response["error"]
   - Validate response contains result field
   - Parse result.success (bool)
   - If success is true:
     - Update claim status: `claim->setStatus(ObservingPaymentClaim::Observing)`
     - Log info: "Claim accepted by observer: {transactionUUID}"
   - If success is false:
     - Get result.message
     - Throw runtime_error with message
   - Close socket
   - On any exception:
     - Log error with observer address and error message
     - Re-throw exception (for timer's exception handler)
4. Use existing headers: nlohmann/json, boost::asio, boost::uuid

## Definition of Done
- [ ] sendClaim stub replaced with full implementation
- [ ] Empty observers check with NotFoundError
- [ ] First observer selected correctly
- [ ] JSON-RPC request constructed per observer/README.md spec
- [ ] Participants array populated from claim->participantsPublicKeys()
- [ ] Public keys converted to base64 via toString()
- [ ] Signature converted to base64 via toString()
- [ ] TCP connection established using boost::asio
- [ ] Request sent with newline terminator
- [ ] Response read and parsed
- [ ] RPC error handling implemented
- [ ] Success response updates claim status to Observing
- [ ] Failure response throws exception with message
- [ ] Network errors throw exceptions
- [ ] Debug and info logging added
- [ ] Code compiles without errors or warnings
- [ ] Method integrates with processPaymentClaims timer loop

# Implementation Plan

## Step 1: Add Required Includes
1. Open `src/core/observing/ObservingHandler.cpp`
2. Verify includes present (should already exist):
   - `#include "../../libs/json/json.h"` (nlohmann/json)
   - `#include <boost/asio/read_until.hpp>`
   - `#include <boost/asio/write.hpp>`
   - `#include <boost/uuid/uuid_io.hpp>`
3. Add if missing

## Step 2: Implement sendClaim Method
1. Locate sendClaim stub (after processPaymentClaims)
2. Replace TODO comment with full implementation
3. Start with debug logging
4. Add empty observers check
5. Get first observer
6. Wrap main logic in try-catch

## Step 3: Construct JSON-RPC Request
1. Create participants JSON array:
   ```cpp
   json::array_t participantsArray;
   for (const auto& [nodeId, publicKey] : claim->participantsPublicKeys()) {
       json participant = {
           {"index", nodeId},
           {"public_key", publicKey->toString()}
       };
       participantsArray.push_back(participant);
   }
   ```
2. Create full request JSON:
   ```cpp
   json request = {
       {"method", "RPCService.AcceptClaim"},
       {"params", json::array({
           {
               {"transaction_uuid", boost::uuids::to_string(claim->transactionUUID())},
               {"max_claim_block_number", claim->maxBlockNumberForClaiming()},
               {"participants", participantsArray},
               {"public_key", claim->publicKey()->toString()},
               {"signature", claim->signature()->toString()}
           }
       })},
       {"id", 1}
   };
   ```

## Step 4: Send Request via TCP
1. Follow ObserverRPCClient::getBlockNumber pattern (lines 17-60 in ObserverRPCClient.cpp)
2. Create resolver and resolve address
3. Create socket and connect
4. Serialize request: `string requestStr = request.dump() + "\n";`
5. Write to socket: `boost::asio::write(socket, boost::asio::buffer(requestStr));`
6. Read response: `boost::asio::read_until(socket, responseBuffer, '\n', errorCode);`
7. Parse response line

## Step 5: Process Response
1. Parse JSON: `json response = json::parse(responseLine);`
2. Check for error field:
   ```cpp
   if (response.contains("error") && !response["error"].is_null()) {
       string errorMsg = response["error"].is_string()
           ? response["error"].get<string>()
           : response["error"].dump();
       throw runtime_error("Observer RPC error: " + errorMsg);
   }
   ```
3. Validate result field:
   ```cpp
   if (!response.contains("result")) {
       throw runtime_error("Invalid RPC response: missing result");
   }
   ```
4. Check success:
   ```cpp
   auto result = response["result"];
   bool success = result.value("success", false);
   ```
5. Update claim or throw

## Step 6: Add Exception Handling
1. Wrap TCP operations in try-catch
2. Catch std::exception
3. Log error with observer address
4. Re-throw for timer handler

## Step 7: Test Compilation
1. Build project in debug mode
2. Fix any compilation errors
3. Verify method signature matches declaration

# Test Plan

**Test Type**: Moderate task - network RPC integration with error handling

**Test Scope**: JSON construction, TCP communication, response parsing, status updates, error handling.

**Validation Approach**:
- Code review to verify JSON format matches observer/README.md
- Manual testing with running observer service
- Debug logs confirm request/response flow
- Exception handling tested via network failures

**Tests to be implemented in 12-08b**:
- JSON-RPC request constructed correctly
- Participants array populated with all entries
- Public keys converted to base64
- Signature converted to base64
- Success response updates status to Observing
- Failure response throws exception with message
- Network error throws exception
- Empty mObservers throws NotFoundError
- First observer selected
- Socket closed after operation
- Exception doesn't change claim status
- Debug logging output correct

# Verification and Validation

## Architecture integrity
- Follows ObserverRPCClient::getBlockNumber pattern
- Uses existing JSON library and boost::asio
- Integrates with processPaymentClaims timer loop
- Maintains single-threaded execution model

## Security
- No validation of observer responses (per PRD - accepted as-is)
- Public keys and signatures transmitted as base64
- Uses first observer only (no multi-observer validation per PRD)
- Network communication unencrypted (TCP, not TLS)

## Performance
- Synchronous RPC call blocks timer until completion or timeout
- TCP connection per request (no persistent connection)
- JSON serialization overhead minimal
- Timeout from boost::asio default (can be configured if needed)

## Scalability
- Single observer per claim submission
- No batching (one claim per request)
- Network latency impacts processing cycle
- Can handle claim submission rate limited by timer period (60s)

## Reliability
- Exception handling prevents timer crash
- Failed claims remain in NoInfo status for retry
- Network errors logged for diagnostics
- Socket closed even on exception (via RAII)

## Maintainability
- Clear error messages for debugging
- Debug logging at key points
- JSON structure matches observer/README.md
- Consistent with existing RPC patterns

## Cost
- No cost validation required

## Compliance
- Follows observer RPC protocol specification
- Implements PRD algorithm exactly
- Uses project-standard error handling

# Restrictions
- Commit changes only after successful compilation
- Do not add retry logic (timer provides retry via NoInfo loop)
- Do not add timeout configuration (use defaults)
- Do not add multiple observer support (use first observer only)
- Do not validate observer signature (per PRD)
- Keep synchronous (no async socket operations)
