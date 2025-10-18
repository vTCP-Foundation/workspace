# 06-11 - Exchange Amount Calculation in CoordinatorExchangePaymentTransaction

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task](06-10-unit-tests-messages-serialization.md)

# Description
Implement mExchangeAmount field and calculation logic in CoordinatorExchangePaymentTransaction to determine the amount coordinator needs to pay (in sender equivalent) to deliver the desired amount (in receiver equivalent). This includes:
1. Adding mExchangeAmount field to the class
2. Implementing calculation in runPaymentInitializationStage() using cached optimal paths
3. Replacing uses of mAmount with mExchangeAmount in sender-side payment checks
4. Using simplified proportional calculation instead of full inverseSimulatePath

This ensures the coordinator knows exactly how much to pay in sender equivalent before starting the payment execution.

# Requirements and DOD

## Requirements

### mExchangeAmount Field
1. Add `TrustLineAmount mExchangeAmount` field to CoordinatorExchangePaymentTransaction
2. Field represents amount sender needs to pay in sender equivalent (mExchangeEquivalent)
3. Distinct from `mAmount` which represents amount receiver gets in receiver equivalent (mEquivalent)
4. Add comment explaining the difference between mAmount and mExchangeAmount

### Calculation in runPaymentInitializationStage()
5. Calculate mExchangeAmount after selfContractor check
6. Retrieve cached optimal paths from ExchangePathsManager using PathCacheKey{mContractorID, mExchangeEquivalent, mEquivalent}
7. Use simplified proportional calculation: `requiredPayment = (deliveredAmount / received_amount) * optimal_flow`
8. Accumulate totalPayment across paths until remainingReceive == 0
9. Return resultNoPathsError() if no cached paths available
10. Return resultInsufficientFundsError() if paths cannot deliver full amount
11. Log calculated mExchangeAmount with sender and receiver equivalents

### Replace mAmount with mExchangeAmount
12. Replace mAmount with mExchangeAmount in totalOutgoingPossibilities checks (runPaymentInitializationStage)
13. Replace in tryReserveAmountDirectlyOnReceiver() for remaining amount calculation
14. Replace in askNeighborToReserveAmount() for remaining amount calculation
15. Replace in runDirectAmountReservationResponseProcessingStage() for total amount checks
16. Replace in processNeighborFurtherReservationResponse() for total amount checks
17. Replace in processRemoteNodeResponse() for total amount checks
18. All totalReservedAmount() calls use senderEquivalent when checking against mExchangeAmount

## Definition of Done
- [x] mExchangeAmount field added to CoordinatorExchangePaymentTransaction.h with comment
- [x] Calculation implemented in runPaymentInitializationStage() after selfContractor check
- [x] PathCacheKey constructed with {mContractorID, mExchangeEquivalent, mEquivalent}
- [x] Simplified proportional calculation implemented (no inverseSimulatePath)
- [x] resultNoPathsError() returned when no cached paths
- [x] resultInsufficientFundsError() returned when insufficient paths
- [x] Logging of calculated mExchangeAmount with equivalents
- [x] All sender-side checks use mExchangeAmount instead of mAmount
- [x] All totalReservedAmount() calls use senderEquivalent with mExchangeAmount
- [x] Code compiles without errors
- [x] Unit tests pass for calculation logic
- [x] PRD updated with mExchangeAmount requirement

# Implementation Plan

## Step 1: Add mExchangeAmount Field
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

Add field in class members section:
```cpp
private:
    TrustLineAmount mAmount;
    TrustLineAmount mExchangeAmount;  // Amount to be paid in sender equivalent (mExchangeEquivalent)
    CommandUUID mCommandUUID;
```

**Key Points**:
- Place right after mAmount for clarity
- Comment explains difference: sender payment vs receiver delivery
- Type is TrustLineAmount (same as mAmount)

## Step 2: Implement Calculation in runPaymentInitializationStage()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

Add calculation block after selfContractor check:
```cpp
TransactionResult::SharedConst CoordinatorExchangePaymentTransaction::runPaymentInitializationStage()
{
    // ... [existing initialization code] ...

    if (mContractor == mContractorsManager->selfContractor()) {
        warning() << "Attempt to create payment to self contractor.";
        return resultForbiddenRun();
    }

    // Calculate mExchangeAmount (amount to pay in sender equivalent)
    // using cached paths similar to EstimatePaymentForReceiveAmountTransaction
    try {
        PathCacheKey key{mContractorID, mExchangeEquivalent, mEquivalent};
        auto cachedPaths = mExchangePathsManager->retrievePaths(key);

        if (!cachedPaths) {
            warning() << "No cached optimal paths for contractor " << mContractorID
                      << " with sender_eq=" << mExchangeEquivalent
                      << " and receiver_eq=" << mEquivalent;
            return resultNoPathsError();
        }

        // Calculate required payment amount (simplified approach without inverseSimulatePath)
        TrustLineAmount remainingReceive = mCommand->amount();  // mAmount - receiver amount
        TrustLineAmount totalPayment = TrustLineAmount(0);

        for (const auto &pathResult : *cachedPaths) {
            if (remainingReceive == TrustLineAmount(0)) {
                break;
            }

            // Use the minimum of remaining needed and what this path can deliver
            TrustLineAmount deliveredAmount = min(remainingReceive, pathResult.received_amount);

            // For simplified calculation: assume optimal_flow is proportional to received_amount
            double ratio = deliveredAmount.convert_to<double>() / pathResult.received_amount.convert_to<double>();
            TrustLineAmount requiredPayment(static_cast<uint64_t>(pathResult.optimal_flow.convert_to<double>() * ratio));

            totalPayment = totalPayment + requiredPayment;
            remainingReceive = remainingReceive - deliveredAmount;
        }

        if (remainingReceive > TrustLineAmount(0)) {
            warning() << "Insufficient paths to deliver " << mCommand->amount()
                      << " to contractor " << mContractorID
                      << "; can deliver only " << (mCommand->amount() - remainingReceive);
            return resultInsufficientFundsError();
        }

        mExchangeAmount = totalPayment;
        info() << "Calculated exchange amount: " << mExchangeAmount
               << " (sender eq=" << mExchangeEquivalent << ") "
               << "to deliver " << mCommand->amount()
               << " (receiver eq=" << mEquivalent << ")";

    } catch (const exception &e) {
        error() << "Error calculating exchange amount: " << e.what();
        return resultProtocolError();
    }

    // ... [continue with rest of initialization] ...
}
```

**Key Points**:
- Calculation happens after selfContractor check
- Uses PathCacheKey with sender and receiver equivalents
- Simplified proportional ratio calculation
- Clear error handling with specific result codes
- Informative logging with equivalents

## Step 3: Replace mAmount with mExchangeAmount in Payment Checks
**Files**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

### In runPaymentInitializationStage() (after calculation):
```cpp
// Check total outgoing possibilities in sender equivalent
const auto senderEquivalent = mExchangeEquivalent;
const auto kTotalOutgoingPossibilities = totalReservedAmount(
    AmountReservation::Outgoing, senderEquivalent);

if (kTotalOutgoingPossibilities < mExchangeAmount) {  // Changed from mAmount
    // ... check audit pending ...
    if (kTotalOutgoingPossibilities + kTotalOutgoingAuditPendingAmount >= mExchangeAmount) {  // Changed from mAmount
        // ... handle audit pending case ...
    }
    // ... handle insufficient funds ...
}
```

### In tryReserveAmountDirectlyOnReceiver():
```cpp
const auto senderEquivalent = pathStats->mPath.equivalents.empty()
    ? mEquivalent
    : pathStats->mPath.equivalents.front();

const auto kReservationAmount = min(
    pathStats->maxFlow(),
    mExchangeAmount - totalReservedAmount(AmountReservation::Outgoing, senderEquivalent));  // Changed from mAmount
```

### In askNeighborToReserveAmount():
```cpp
const auto senderEquivalent = pathStats->mPath.equivalents.empty()
    ? mEquivalent
    : pathStats->mPath.equivalents.front();

const auto kReservationAmount = min(
    pathStats->maxFlow(),
    mExchangeAmount - totalReservedAmount(AmountReservation::Outgoing, senderEquivalent));  // Changed from mAmount
```

### In runDirectAmountReservationResponseProcessingStage():
```cpp
const auto senderEquivalent = pathStats->mPath.equivalents.empty()
    ? mEquivalent
    : pathStats->mPath.equivalents.front();

const auto kTotalAmount = totalReservedAmount(AmountReservation::Outgoing, senderEquivalent);

if (kTotalAmount > mExchangeAmount) {  // Changed from mAmount
    debug() << "Total exchange amount: " << mExchangeAmount;
    // ... shortage logic ...
}

if (kTotalAmount == mExchangeAmount) {  // Changed from mAmount
    // ... completion logic ...
}
```

### In processNeighborFurtherReservationResponse():
```cpp
const auto senderEquivalent = pathStats->mPath.equivalents.empty()
    ? mEquivalent
    : pathStats->mPath.equivalents.front();

const auto kTotalAmount = totalReservedAmount(AmountReservation::Outgoing, senderEquivalent);

if (kTotalAmount > mExchangeAmount) {  // Changed from mAmount
    // ... shortage logic ...
}

if (kTotalAmount == mExchangeAmount) {  // Changed from mAmount
    // ... completion logic ...
}
```

### In processRemoteNodeResponse():
```cpp
const auto senderEquivalent = pathStats->mPath.equivalents.empty()
    ? mEquivalent
    : pathStats->mPath.equivalents.front();

const auto kTotalAmount = totalReservedAmount(AmountReservation::Outgoing, senderEquivalent);

if (kTotalAmount > mExchangeAmount) {  // Changed from mAmount
    // ... shortage logic ...
}

if (kTotalAmount == mExchangeAmount) {  // Changed from mAmount
    // ... completion logic ...
}
```

**Key Points**:
- Always get senderEquivalent from path first
- Use mExchangeAmount for sender-side payment checks
- Use senderEquivalent with totalReservedAmount() calls
- Keep mAmount for receiver-side operations

# Test Plan

## Unit Tests
As this is a Moderate task, focused testing on core functionality:

### Test Category: mExchangeAmount Calculation
1. **testCalculateExchangeAmountSinglePath**: Calculate with single path
   - Setup: mAmount = 1000 (receiver eq 2), cached path with received_amount = 1000, optimal_flow = 500 (sender eq 1)
   - Expected: mExchangeAmount = 500 (exact ratio 1:2)

2. **testCalculateExchangeAmountMultiplePaths**: Calculate across multiple paths
   - Setup: mAmount = 1000, paths: [received=600, flow=300], [received=400, flow=200]
   - Expected: mExchangeAmount = 500 (300 + 200)

3. **testCalculateExchangeAmountPartialPath**: Use partial capacity from path
   - Setup: mAmount = 500, path: received=1000, flow=800
   - Expected: mExchangeAmount = 400 (50% of path: 0.5 * 800)

4. **testCalculateExchangeAmountNoCachedPaths**: Handle no cached paths
   - Setup: ExchangePathsManager returns nullptr
   - Expected: resultNoPathsError() returned

5. **testCalculateExchangeAmountInsufficientPaths**: Handle insufficient total flow
   - Setup: mAmount = 1000, paths can deliver only 800
   - Expected: resultInsufficientFundsError() returned

6. **testCalculateExchangeAmountProportionalCalculation**: Verify proportional ratio
   - Setup: mAmount = 100, path: received=1000, flow=2000
   - Expected: mExchangeAmount = 200 (ratio: 100/1000 = 0.1, payment: 0.1 * 2000 = 200)

7. **testCalculateExchangeAmountExceptionHandling**: Handle calculation exceptions
   - Setup: Invalid path data causing exception
   - Expected: resultProtocolError() returned

8. **testCalculateExchangeAmountLogging**: Verify logging output
   - Expected: Log includes mExchangeAmount, sender_eq, receiver_eq, mAmount

### Test Category: mExchangeAmount Usage in Payment Checks
9. **testTotalOutgoingCheckUsesExchangeAmount**: Verify runPaymentInitializationStage checks
   - Setup: kTotalOutgoingPossibilities = 400, mExchangeAmount = 500
   - Expected: Insufficient funds error (compares against mExchangeAmount, not mAmount)

10. **testReserveAmountDirectlyUsesExchangeAmount**: Verify tryReserveAmountDirectlyOnReceiver
    - Setup: totalReservedAmount = 300, mExchangeAmount = 500, pathStats->maxFlow = 300
    - Expected: kReservationAmount = min(300, 500-300) = 200

11. **testAskNeighborUsesExchangeAmount**: Verify askNeighborToReserveAmount
    - Setup: totalReservedAmount = 400, mExchangeAmount = 1000, pathStats->maxFlow = 700
    - Expected: kReservationAmount = min(700, 1000-400) = 600

12. **testDirectResponseUsesExchangeAmount**: Verify runDirectAmountReservationResponseProcessingStage
    - Setup: kTotalAmount = 1000, mExchangeAmount = 1000
    - Expected: Completion logic triggered (kTotalAmount == mExchangeAmount)

13. **testFurtherResponseUsesExchangeAmount**: Verify processNeighborFurtherReservationResponse
    - Setup: kTotalAmount = 1200, mExchangeAmount = 1000
    - Expected: Shortage logic triggered (kTotalAmount > mExchangeAmount)

14. **testRemoteResponseUsesExchangeAmount**: Verify processRemoteNodeResponse
    - Setup: kTotalAmount = 1000, mExchangeAmount = 1000
    - Expected: Completion logic triggered (kTotalAmount == mExchangeAmount)

### Test Category: Sender Equivalent Usage
15. **testSenderEquivalentFromPathEquivalents**: Verify senderEquivalent extraction
    - Setup: pathStats->mPath.equivalents = [1, 2, 3]
    - Expected: senderEquivalent = 1 (first element)

16. **testSenderEquivalentFallbackToMEquivalent**: Verify fallback when equivalents empty
    - Setup: pathStats->mPath.equivalents.empty() == true, mEquivalent = 5
    - Expected: senderEquivalent = 5

17. **testTotalReservedAmountWithSenderEquivalent**: Verify totalReservedAmount uses correct equivalent
    - Setup: Reservations in eq 1 (500) and eq 2 (300), senderEquivalent = 1
    - Expected: totalReservedAmount(Outgoing, 1) = 500 (only eq 1)

## Success Criteria
- All 17 unit tests pass
- Code compiles without errors or warnings
- mExchangeAmount calculation correct for single and multiple paths
- All sender-side checks use mExchangeAmount
- All totalReservedAmount calls use senderEquivalent
- PRD updated with mExchangeAmount requirement

# Verification and Validation

## Architecture integrity
- Clean separation: mAmount (receiver) vs mExchangeAmount (sender)
- Consistent use of equivalents: senderEquivalent with mExchangeAmount, receiverEquivalent with mAmount
- ExchangePathsManager integration follows existing pattern
- Simplified calculation avoids complexity of inverseSimulatePath

## Security
- Exception handling prevents crashes on invalid data
- PathCacheKey validation ensures correct path retrieval
- Error codes provide clear failure reasons

## Performance
- Simplified proportional calculation: O(P) where P is number of paths (typically < 10)
- No inverseSimulatePath overhead (complex graph simulation avoided)
- Early break when remainingReceive == 0
- Single pass through cached paths

## Scalability
- Handles up to 5 exchange equivalents (design limit)
- Supports multiple paths for single payment
- Memory overhead: single TrustLineAmount field (negligible)

## Reliability
- Clear error handling: no paths, insufficient paths, calculation exceptions
- Logging provides debugging visibility
- Proportional calculation robust for edge cases (zero, partial paths)

## Maintainability
- Well-documented field with comment
- Clear algorithm with step-by-step logic
- Consistent naming: mExchangeAmount vs mAmount
- Easy to extend for more sophisticated calculation later

## Cost
- No additional infrastructure required
- Reuses existing ExchangePathsManager cache
- Minimal computational overhead

## Compliance
- Follows repository policy for task-driven development
- Adheres to PRD requirements
- No prohibited operations or scope creep
- PRD updated with new requirement

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not modify ExchangePathsManager (use existing interface)
- Do not implement full inverseSimulatePath (use simplified approach)
- Do not change mAmount semantics (receiver amount remains unchanged)
- Update PRD with mExchangeAmount requirement before task completion
