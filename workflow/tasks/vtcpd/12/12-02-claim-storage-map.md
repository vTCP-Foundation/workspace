# 12-02 - Claim Storage Map in ObservingHandler

# Links
- [PRD](../../../prd/vtcpd/12-observer-ambiguous-transaction-handling.md)
- [Previous task: 12-01](12-01-observing-payment-claim-class.md)

# Description
Add claim storage infrastructure to ObservingHandler to track pending payment claims. This includes a map using composite key (TransactionUUID, BlockNumber) and a public method to add new claims to the map.

The storage is designed for single-threaded access (vtcpd execution model) and enables efficient claim lookup for processing and cleanup operations.

# Requirements and DOD

## Functional Requirements
1. Add private field to ObservingHandler class (`src/core/observing/ObservingHandler.h`):
   ```cpp
   map<pair<TransactionUUID, BlockNumber>, ObservingPaymentClaim::Shared> mPaymentClaims;
   ```
2. Include ObservingPaymentClaim.h in ObservingHandler.h
3. Add public method to ObservingHandler:
   ```cpp
   void addPaymentClaim(
       const TransactionUUID &transactionUUID,
       BlockNumber maxBlockNumberForClaiming,
       const map<PaymentNodeID, sphincs::PublicKey::Shared> &participantsPublicKeys,
       sphincs::PublicKey::Shared publicKey,
       sphincs::Signature::Shared signature);
   ```
4. Implement addPaymentClaim in `src/core/observing/ObservingHandler.cpp`:
   - Create ObservingPaymentClaim::Shared instance with provided parameters
   - Create composite key: `make_pair(transactionUUID, maxBlockNumberForClaiming)`
   - Insert into mPaymentClaims map: `mPaymentClaims[key] = claim`
   - Add debug logging (under `#ifdef DEBUG_LOG_OBSEVING_HANDLER`):
     - Log claim addition with transactionUUID and maxBlockNumberForClaiming
5. Map ensures uniqueness: if key already exists, new claim replaces old one (map behavior)
6. No explicit thread synchronization required (single-threaded execution context)

## Definition of Done
- [ ] mPaymentClaims field added to ObservingHandler.h
- [ ] ObservingPaymentClaim.h included in ObservingHandler.h
- [ ] addPaymentClaim method declared in ObservingHandler.h
- [ ] addPaymentClaim method implemented in ObservingHandler.cpp
- [ ] Method creates ObservingPaymentClaim with all parameters
- [ ] Method inserts claim into map with composite key
- [ ] Debug logging added for claim addition
- [ ] Code compiles without errors or warnings
- [ ] Existing ObservingHandler functionality unchanged

# Implementation Plan

## Step 1: Update Header File
1. Open `src/core/observing/ObservingHandler.h`
2. Add include: `#include "ObservingPaymentClaim.h"`
3. Locate private member section (after mCheckedTransactions around line 157)
4. Add field: `map<pair<TransactionUUID, BlockNumber>, ObservingPaymentClaim::Shared> mPaymentClaims;`
5. Add comment: `// Payment claims for observer-based resolution`
6. Locate public method section (after requestActualBlockNumber around line 61)
7. Add method declaration with full signature
8. Add comment explaining method purpose

## Step 2: Implement addPaymentClaim Method
1. Open `src/core/observing/ObservingHandler.cpp`
2. Add method implementation after getActualBlockNumber (around line 467):
   ```cpp
   void ObservingHandler::addPaymentClaim(
       const TransactionUUID &transactionUUID,
       BlockNumber maxBlockNumberForClaiming,
       const map<PaymentNodeID, sphincs::PublicKey::Shared> &participantsPublicKeys,
       sphincs::PublicKey::Shared publicKey,
       sphincs::Signature::Shared signature)
   {
   #ifdef DEBUG_LOG_OBSEVING_HANDLER
       debug() << "Adding payment claim: " << transactionUUID
               << " maxBlockNumber: " << maxBlockNumberForClaiming;
   #endif

       auto claim = make_shared<ObservingPaymentClaim>(
           transactionUUID,
           maxBlockNumberForClaiming,
           participantsPublicKeys,
           publicKey,
           signature);

       auto key = make_pair(transactionUUID, maxBlockNumberForClaiming);
       mPaymentClaims[key] = claim;

   #ifdef DEBUG_LOG_OBSEVING_HANDLER
       debug() << "Payment claims map size: " << mPaymentClaims.size();
   #endif
   }
   ```

## Step 3: Verify Compilation
1. Build project in debug mode
2. Verify no compilation errors
3. Check that existing ObservingHandler methods still compile
4. Verify ObservingTransaction functionality unaffected

## Step 4: Code Review
1. Verify map key type is correct (pair<TransactionUUID, BlockNumber>)
2. Verify Shared typedef used correctly
3. Verify method signature matches requirements
4. Check debug logging messages are clear

# Test Plan

**Test Type**: Simple task - data structure and accessor method

**Test Scope**: Adding storage field and insertion method to existing class.

**Validation Approach**:
- Code review to verify field and method added correctly
- Compilation success confirms integration with ObservingPaymentClaim
- Debug log inspection confirms method execution

**Tests to be implemented in 12-08a**:
- addPaymentClaim creates entry in map
- Composite key ensures uniqueness
- Duplicate claim (same key) replaces previous
- Map retrieval by key returns correct claim
- Empty map handled correctly
- Multiple claims with different keys coexist

# Verification and Validation

## Architecture integrity
- Follows existing ObservingHandler pattern (similar to mClaims map)
- Maintains separation between ObservingTransaction and ObservingPaymentClaim
- Uses standard C++ containers (map with pair key)
- Public method provides controlled access to private map

## Security
- No security concerns for internal data storage
- Access controlled through public method only

## Performance
- Map provides O(log n) lookup by composite key
- No performance impact on existing ObservingHandler operations
- Memory usage proportional to number of active claims

## Scalability
- Map efficiently handles up to 1000 claims (PRD requirement)
- Composite key ensures efficient lookup without collisions
- No scalability concerns for expected load

## Reliability
- Map automatically handles key collisions (replacement behavior)
- make_shared ensures proper memory management
- No failure modes in insertion operation

## Maintainability
- Clear field naming (mPaymentClaims)
- Intuitive method name (addPaymentClaim)
- Debug logging aids troubleshooting
- Consistent with existing codebase patterns

## Cost
- No cost validation required

## Compliance
- Follows project coding standards
- Single-threaded execution model (no synchronization required per PRD)

# Restrictions
- Commit changes only after successful compilation
- Do not modify existing ObservingTransaction-related code
- Do not add processing logic (timer processing is separate task 12-03)
- Keep method simple (just insertion, no validation beyond type checking)
