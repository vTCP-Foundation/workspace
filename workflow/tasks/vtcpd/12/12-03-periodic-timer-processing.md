# 12-03 - Periodic Timer Processing for Payment Claims

# Links
- [PRD](../../../prd/vtcpd/12-observer-ambiguous-transaction-handling.md)
- [Previous task: 12-02](12-02-claim-storage-map.md)

# Description
Implement periodic timer-based processing of payment claims in ObservingHandler. The timer automatically processes all pending claims according to their status, dispatches to appropriate RPC methods, and performs cleanup of completed claims.

This establishes the core state machine driving claim progression from submission through observer interaction to finalization.

# Requirements and DOD

## Functional Requirements
1. Add private timer field to ObservingHandler (`src/core/observing/ObservingHandler.h`):
   ```cpp
   as::steady_timer mPaymentClaimsTimer;
   ```
2. Add timer period constants to ObservingHandler.h (in private section after line 150):
   ```cpp
   static constexpr uint32_t kPaymentClaimProcessingPeriodSeconds = 60;
   #ifdef TESTS
   static constexpr uint32_t kPaymentClaimProcessingPeriodSecondsTests = 10;
   #endif
   ```
3. Add protected method declaration to ObservingHandler.h:
   ```cpp
   void processPaymentClaims();
   ```
4. Initialize timer in ObservingHandler constructor (in initialization list):
   ```cpp
   mPaymentClaimsTimer(ioCtx)
   ```
5. Start timer in constructor body (after existing timer initializations):
   - Set initial expiration: `kPaymentClaimProcessingPeriodSeconds` (or test version)
   - Register async callback to `processPaymentClaims`
6. Implement `processPaymentClaims` method in ObservingHandler.cpp:
   - Add debug log: "Processing payment claims, count: X"
   - Create vector for claims to remove: `vector<pair<TransactionUUID, BlockNumber>> claimsToRemove`
   - Iterate through `mPaymentClaims` map
   - For each claim, wrap in try-catch and dispatch based on status:
     - `NoInfo` → call `sendClaim(claim)` (stub for now)
     - `Observing` → call `checkTransaction(claim)` (stub for now)
     - `ParticipantsVotesPresent` → call `getParticipantsSignatures(claim)` (stub for now)
     - `RejectedByObserving` → call `rejectTransaction(claim)` (stub for now)
     - `Done` → add key to claimsToRemove vector
   - Catch exceptions: log warning with claim UUID and error, continue processing
   - Cleanup phase: iterate claimsToRemove and erase from mPaymentClaims
   - Reschedule timer with same period
   - Register async callback to processPaymentClaims
7. Add protected method stubs (empty implementations for now):
   ```cpp
   void sendClaim(ObservingPaymentClaim::Shared claim);
   void checkTransaction(ObservingPaymentClaim::Shared claim);
   void getParticipantsSignatures(ObservingPaymentClaim::Shared claim);
   void rejectTransaction(ObservingPaymentClaim::Shared claim);
   ```

## Definition of Done
- [ ] mPaymentClaimsTimer field added to ObservingHandler.h
- [ ] Timer period constants defined (production and test versions)
- [ ] processPaymentClaims method declared
- [ ] Stub methods declared for RPC operations
- [ ] Timer initialized in constructor initialization list
- [ ] Timer started in constructor body
- [ ] processPaymentClaims method implemented with state machine dispatch
- [ ] Exception handling prevents single claim failure from stopping processing
- [ ] Cleanup phase removes Done claims from map
- [ ] Timer rescheduling implemented
- [ ] Debug logging added for processing start and claim removal
- [ ] Code compiles without errors or warnings
- [ ] Timer executes processPaymentClaims periodically

# Implementation Plan

## Step 1: Update Header File
1. Open `src/core/observing/ObservingHandler.h`
2. Add timer field in private section (after mTransactionsTimer around line 164):
   ```cpp
   as::steady_timer mPaymentClaimsTimer;
   ```
3. Add timer period constants (after kApproximateBlockNumberIncrementingPeriodSecondsTests around line 149):
   ```cpp
   static constexpr uint32_t kPaymentClaimProcessingPeriodSeconds = 60;
   #ifdef TESTS
   static constexpr uint32_t kPaymentClaimProcessingPeriodSecondsTests = 10;
   #endif
   ```
4. Add protected method declarations (after runTransactionsChecking around line 102):
   ```cpp
   void processPaymentClaims();
   void sendClaim(ObservingPaymentClaim::Shared claim);
   void checkTransaction(ObservingPaymentClaim::Shared claim);
   void getParticipantsSignatures(ObservingPaymentClaim::Shared claim);
   void rejectTransaction(ObservingPaymentClaim::Shared claim);
   ```

## Step 2: Initialize Timer in Constructor
1. Open `src/core/observing/ObservingHandler.cpp`
2. Locate constructor initialization list (around line 18)
3. Add after mRequestsTimer initialization:
   ```cpp
   mPaymentClaimsTimer(ioCtx),
   ```
4. Locate constructor body end (around line 51)
5. Add timer start before closing brace:
   ```cpp
   // Start payment claims processing timer
   mPaymentClaimsTimer.expires_after(
       chrono::seconds(kPaymentClaimProcessingPeriodSeconds));
   #ifdef TESTS
   mPaymentClaimsTimer.expires_after(
       chrono::seconds(kPaymentClaimProcessingPeriodSecondsTests));
   #endif

   mPaymentClaimsTimer.async_wait([this](const boost::system::error_code &e) {
       if (e == boost::asio::error::operation_aborted) {
           return;
       }
       processPaymentClaims();
   });
   ```

## Step 3: Implement processPaymentClaims Method
1. Add implementation after addPaymentClaim method:
   ```cpp
   void ObservingHandler::processPaymentClaims()
   {
   #ifdef DEBUG_LOG_OBSEVING_HANDLER
       debug() << "Processing payment claims, count: " << mPaymentClaims.size();
   #endif

       // Step 1: Process each claim based on status
       vector<pair<TransactionUUID, BlockNumber>> claimsToRemove;

       for (auto& [key, claim] : mPaymentClaims) {
           try {
               switch (claim->status()) {
                   case ObservingPaymentClaim::NoInfo:
                       sendClaim(claim);
                       break;

                   case ObservingPaymentClaim::Observing:
                       checkTransaction(claim);
                       break;

                   case ObservingPaymentClaim::ParticipantsVotesPresent:
                       getParticipantsSignatures(claim);
                       break;

                   case ObservingPaymentClaim::RejectedByObserving:
                       rejectTransaction(claim);
                       break;

                   case ObservingPaymentClaim::Done:
                       claimsToRemove.push_back(key);
                       break;
               }
           } catch (const std::exception &e) {
               warning() << "Error processing claim " << claim->transactionUUID()
                         << ": " << e.what();
           }
       }

       // Step 2: Cleanup completed claims
       for (const auto& key : claimsToRemove) {
   #ifdef DEBUG_LOG_OBSEVING_HANDLER
           debug() << "Removing completed claim: " << key.first;
   #endif
           mPaymentClaims.erase(key);
       }

       // Step 3: Reschedule timer
       mPaymentClaimsTimer.expires_after(
           chrono::seconds(kPaymentClaimProcessingPeriodSeconds));
   #ifdef TESTS
       mPaymentClaimsTimer.expires_after(
           chrono::seconds(kPaymentClaimProcessingPeriodSecondsTests));
   #endif

       mPaymentClaimsTimer.async_wait([this](const boost::system::error_code &e) {
           if (e == boost::asio::error::operation_aborted) {
               return;
           }
           processPaymentClaims();
       });
   }
   ```

## Step 4: Add Method Stubs
1. Add stub implementations after processPaymentClaims:
   ```cpp
   void ObservingHandler::sendClaim(ObservingPaymentClaim::Shared claim)
   {
       // TODO: Implement in task 12-04
   }

   void ObservingHandler::checkTransaction(ObservingPaymentClaim::Shared claim)
   {
       // TODO: Implement in task 12-05
   }

   void ObservingHandler::getParticipantsSignatures(ObservingPaymentClaim::Shared claim)
   {
       // TODO: Implement in task 12-06
   }

   void ObservingHandler::rejectTransaction(ObservingPaymentClaim::Shared claim)
   {
       // TODO: Implement in task 12-07
   }
   ```

## Step 5: Verify Implementation
1. Build project in debug mode
2. Verify compilation succeeds
3. Verify no warnings
4. Check timer starts in constructor
5. Verify processPaymentClaims can be called without crashing (with empty map)

# Test Plan

**Test Type**: Moderate task - timer infrastructure with state machine dispatch

**Test Scope**: Timer initialization, periodic execution, state machine dispatch, cleanup logic.

**Validation Approach**:
- Code review to verify timer initialization and callback registration
- Debug build confirms timer starts in constructor
- Manual testing with added claims verifies dispatch occurs
- Exception handling prevents crashes

**Tests to be implemented in 12-08a**:
- Timer starts in constructor
- Timer has correct period (production and test versions)
- Timer reschedules after each cycle
- processPaymentClaims invoked periodically
- NoInfo status dispatches to sendClaim (stub)
- Observing status dispatches to checkTransaction (stub)
- ParticipantsVotesPresent dispatches to getParticipantsSignatures (stub)
- RejectedByObserving dispatches to rejectTransaction (stub)
- Done status adds claim to removal list
- Cleanup phase erases Done claims
- Exception in one claim doesn't stop processing others
- Empty map handled without errors

# Verification and Validation

## Architecture integrity
- Follows existing ObservingHandler timer pattern (mClaimsTimer, mTransactionsTimer)
- State machine dispatch provides clear separation of concerns
- Protected methods enable future enhancement without interface changes
- Exception handling prevents timer failure

## Security
- No security concerns for timer infrastructure
- Exception handling prevents denial of service via malformed claims

## Performance
- Single timer for all claims (efficient)
- Linear iteration through map (acceptable for up to 1000 claims per PRD)
- Cleanup in same cycle prevents accumulation
- Timer period configurable (60s production, 10s tests)

## Scalability
- Processing completes within 5 seconds for 100 claims (PRD requirement)
- Can handle up to 1000 concurrent claims (PRD requirement)
- Cleanup prevents unbounded growth

## Reliability
- Exception handling per claim prevents cascading failures
- Timer automatically reschedules (continuous operation)
- operation_aborted check prevents double-processing
- Cleanup ensures map doesn't grow indefinitely

## Maintainability
- Clear state machine structure (switch on status)
- Method stubs document future implementation points
- Debug logging aids troubleshooting
- Consistent with existing timer implementations
- Test-specific timing via #ifdef TESTS

## Cost
- No cost validation required

## Compliance
- Follows project coding standards
- Consistent with existing ObservingHandler patterns
- Single-threaded execution (no locks required)

# Restrictions
- Commit changes only after successful compilation
- Do not implement RPC methods (stubs only)
- Do not modify existing timer implementations (mClaimsTimer, mTransactionsTimer)
- Keep method stubs empty (implementation in separate tasks)
- Ensure timer starts in constructor (not conditionally)
