# 12-01 - ObservingPaymentClaim Class Implementation

# Links
- [PRD](../../../prd/vtcpd/12-observer-ambiguous-transaction-handling.md)

# Description
Create the ObservingPaymentClaim class to represent ambiguous payment transactions in the ObservingHandler context. This class stores all necessary data for submitting claims to observers and tracking their processing status through a state machine.

The class must include transaction identification (UUID and max claiming block number), participant public keys, node's own public key and signature, and a status enum tracking the claim lifecycle from submission to finalization.

# Requirements and DOD

## Functional Requirements
1. Class `ObservingPaymentClaim` created in `src/core/observing/ObservingPaymentClaim.h` and `.cpp`
2. Status enum `ClaimStatus` with exactly 5 values:
   - `NoInfo = 0`
   - `Observing = 1`
   - `ParticipantsVotesPresent = 2`
   - `RejectedByObserving = 3`
   - `Done = 4`
3. Class contains the following private fields:
   - `TransactionUUID mTransactionUUID`
   - `BlockNumber mMaxBlockNumberForClaiming`
   - `map<PaymentNodeID, sphincs::PublicKey::Shared> mParticipantsPublicKeys`
   - `sphincs::PublicKey::Shared mPublicKey`
   - `sphincs::Signature::Shared mSignature`
   - `ClaimStatus mStatus`
4. Constructor signature:
   ```cpp
   ObservingPaymentClaim(
       const TransactionUUID &transactionUUID,
       BlockNumber maxBlockNumberForClaiming,
       const map<PaymentNodeID, sphincs::PublicKey::Shared> &participantsPublicKeys,
       sphincs::PublicKey::Shared publicKey,
       sphincs::Signature::Shared signature);
   ```
5. Constructor initializes `mStatus` to `NoInfo`
6. Public typedef: `typedef shared_ptr<ObservingPaymentClaim> Shared;`
7. Const getter methods for all fields:
   - `const TransactionUUID& transactionUUID() const`
   - `BlockNumber maxBlockNumberForClaiming() const`
   - `const map<PaymentNodeID, sphincs::PublicKey::Shared>& participantsPublicKeys() const`
   - `sphincs::PublicKey::Shared publicKey() const`
   - `sphincs::Signature::Shared signature() const`
   - `ClaimStatus status() const`
8. Status setter method: `void setStatus(ClaimStatus status)`
9. Class does not inherit from any base class
10. No namespace (global scope, consistent with project style)

## Definition of Done
- [ ] Header file `ObservingPaymentClaim.h` created with complete class declaration
- [ ] Implementation file `ObservingPaymentClaim.cpp` created with constructor and methods
- [ ] All required includes present (TransactionUUID, BlockNumber, PaymentNodeID, sphincs types)
- [ ] Status enum defined with correct integer values
- [ ] Constructor initializes all fields from parameters
- [ ] Constructor sets mStatus to NoInfo by default
- [ ] All getter methods implemented and return correct values
- [ ] setStatus method implemented
- [ ] Shared typedef defined
- [ ] Code compiles without errors or warnings
- [ ] Files added to appropriate CMakeLists.txt if needed

# Implementation Plan

## Step 1: Create Header File
1. Create file `src/core/observing/ObservingPaymentClaim.h`
2. Add include guards
3. Add necessary includes:
   ```cpp
   #include "../common/Types.h"
   #include "../crypto/sphincskeys.h"
   #include "../crypto/sphincsscheme.h"
   #include <map>
   #include <memory>
   ```
4. Declare ClaimStatus enum with specified values
5. Declare class with all private fields
6. Declare Shared typedef
7. Declare constructor with full signature
8. Declare all getter methods
9. Declare setStatus method

## Step 2: Create Implementation File
1. Create file `src/core/observing/ObservingPaymentClaim.cpp`
2. Include the header file
3. Implement constructor:
   - Initialize all member fields from parameters using initializer list
   - Set mStatus to NoInfo
4. Implement all getter methods (simple return statements)
5. Implement setStatus method (simple assignment)

## Step 3: Update Build System
1. Check if `src/core/observing/CMakeLists.txt` exists
2. If yes, add ObservingPaymentClaim.cpp to source list
3. If no CMakeLists.txt in observing directory, files will be picked up by parent CMakeLists.txt

## Step 4: Verify Compilation
1. Build project in debug mode
2. Verify no compilation errors
3. Verify no warnings related to new files

# Test Plan

**Test Type**: Simple task - basic class implementation

**Test Scope**: This task focuses on creating the data structure. Unit tests will be implemented in task 12-08a.

**Validation Approach**:
- Code review to verify all fields, methods, and enum values are present
- Compilation success confirms correct syntax and includes
- Manual inspection of generated class structure

**Tests to be implemented in 12-08a**:
- Constructor initializes all fields correctly
- Constructor sets status to NoInfo
- All getters return correct values
- setStatus updates status correctly
- Enum values have correct integer assignments

# Verification and Validation

## Architecture integrity
- Class follows existing project patterns (similar to ObservingTransaction)
- Located in appropriate directory (src/core/observing/)
- Uses standard project types (TransactionUUID, BlockNumber, PaymentNodeID)
- Uses existing crypto infrastructure (sphincs)

## Security
- No security validation required for data structure definition

## Performance
- Minimal performance impact (simple data holder)
- All getters return by const reference where appropriate (map, TransactionUUID)
- Shared pointer typedef enables efficient sharing

## Scalability
- Class size proportional to number of participants in map
- No scalability concerns for individual claim objects

## Reliability
- Constructor ensures all fields initialized
- No dynamic allocation beyond shared_ptr parameters
- No error conditions in simple getters/setters

## Maintainability
- Clear field names and method signatures
- Standard C++ practices (const correctness, initializer lists)
- Well-documented enum values
- Consistent with ObservingTransaction style

## Cost
- No cost validation required

## Compliance
- Follows project coding standards
- Consistent with existing observing infrastructure

# Restrictions
- Commit changes only after successful compilation
- Do not add functionality beyond specified fields and methods
- Do not add serialization/deserialization (not required in this PRD)
