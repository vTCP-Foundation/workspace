# 10-02 - Validation Logic and Total Reserved Amount Calculation

# Links
- [PRD](../../prd/vtcpd/10-allowable-payment-amount-control.md)
- [Previous task: 10-01 Command Extension](10-01-command-extension-and-result-code.md)

# Description
Implement validation logic to enforce user-specified maximum allowable payment amount at two stages: before starting reservations (early validation) and after each path reservation completion (late validation). Add `calculateTotalReservedPaymentAmount()` method to compute total reserved amount across all payment equivalents. When limit is exceeded, call `reject()` to drop all reservations via existing `rollBack()` mechanism, then return result code 415.

# Requirements and DOD

## Functional Requirements

### 1. Total Reserved Amount Calculation Method
1. Add method `TrustLineAmount calculateTotalReservedPaymentAmount() const` to `CoordinatorExchangePaymentTransaction`
2. Method iterates through `mCommand->exchangeEquivalents()`
3. For each equivalent, call `totalReservedAmount(AmountReservation::Outgoing, equivalent)`
4. Sum all returned amounts
5. Return `TrustLineAmount(0)` if no reservations
6. **Important**: Use existing aggregates through `totalReservedAmount()` API, do not manually iterate `mPathsStats`

### 2. Early Validation in runPathsResourceProcessingStage()
1. After calculating `mExchangeAmount` (Step 0) and before checking `kTotalOutgoingPossibilities` (Step 0.5)
2. If `mCommand->maxAllowablePaymentAmount().has_value()`:
   - Compare `mExchangeAmount` with `*mCommand->maxAllowablePaymentAmount()`
   - If `mExchangeAmount > maxAllowablePaymentAmount`: log warning, return `resultAllowablePaymentAmountExceeded()`
   - If within limit: log debug message and continue
3. If parameter not provided: skip validation

### 3. Late Validation in processRemoteNodeResponse()
1. Inside `if (path->isLastIntermediateNodeProcessed())` condition
2. After existing path processing logic
3. If `mCommand->maxAllowablePaymentAmount().has_value()`:
   - Call `calculateTotalReservedPaymentAmount()`
   - If total > limit: log warning with both amounts and pathID, call `reject("Allowable payment amount exceeded")`, return `resultAllowablePaymentAmountExceeded()`
   - If within limit: log debug message and continue
4. Validation runs even if target `mAmount` for receiver already achieved
5. Pattern follows existing code at line 2590-2591: `reject(...); return result...();`

### 4. Late Validation in processNeighborFurtherReservationResponse()
1. Identical logic to processRemoteNodeResponse
2. Inside `if (path->isLastIntermediateNodeProcessed())` condition
3. Log message includes "via neighbor" for distinction

## Definition of Done
- [ ] Method `calculateTotalReservedPaymentAmount()` implemented in `CoordinatorExchangePaymentTransaction`
- [ ] Method correctly sums amounts from all exchange equivalents
- [ ] Method returns 0 when no reservations exist
- [ ] Early validation added to `runPathsResourceProcessingStage()` after `mExchangeAmount` calculation
- [ ] Early validation logs warning and returns 415 when limit exceeded
- [ ] Early validation skipped when parameter not provided
- [ ] Late validation added to `processRemoteNodeResponse()` in correct location
- [ ] Late validation added to `processNeighborFurtherReservationResponse()` in correct location
- [ ] Late validation calls `reject()` without return, then returns `resultAllowablePaymentAmountExceeded()`
- [ ] Late validation runs even when `mAmount` target achieved
- [ ] All reservations dropped via existing `rollBack()` mechanism when limit exceeded
- [ ] All code compiles without errors or warnings
- [ ] Logging includes both amounts (reserved and limit) and pathID where applicable
- [ ] Code follows existing patterns (like line 2590-2591)

# Implementation Plan

## Step 1: Implement calculateTotalReservedPaymentAmount()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

Add protected method declaration:
```cpp
TrustLineAmount calculateTotalReservedPaymentAmount() const;
```

**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

Implementation:
```cpp
TrustLineAmount CoordinatorExchangePaymentTransaction::calculateTotalReservedPaymentAmount() const
{
    TrustLineAmount totalReserved = TrustLineAmount(0);

    // Iterate through all exchange equivalents
    for (const auto equivalent : mCommand->exchangeEquivalents()) {
        // Get reserved amount for this equivalent using existing API
        const auto reservedForEquivalent = totalReservedAmount(
            AmountReservation::Outgoing,
            equivalent);

        // Add to total
        totalReserved = totalReserved + reservedForEquivalent;
    }

    return totalReserved;
}
```

**Key points**:
- Reuses existing `totalReservedAmount()` API
- No manual iteration through `mPathsStats`
- No manual conversions between equivalents
- Returns sum in payment equivalent

## Step 2: Add Early Validation in runPathsResourceProcessingStage()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

**Location**: After Step 0 (mExchangeAmount calculation) and before Step 0.5 (kTotalOutgoingPossibilities check)

**Code to add**:
```cpp
// Step 0.4: Check maxAllowablePaymentAmount (if provided)
if (mCommand->maxAllowablePaymentAmount().has_value()) {
    if (mExchangeAmount > *mCommand->maxAllowablePaymentAmount()) {
        warning() << "Calculated exchange amount " << mExchangeAmount
                  << " exceeds maximum allowable payment amount "
                  << *mCommand->maxAllowablePaymentAmount();
        return resultAllowablePaymentAmountExceeded();
    }

    info() << "Exchange amount " << mExchangeAmount
           << " within allowable limit " << *mCommand->maxAllowablePaymentAmount();
}

// Step 0.5: Check kTotalOutgoingPossibilities (existing code continues)
```

## Step 3: Add Late Validation in processRemoteNodeResponse()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

**Location**: Inside `if (path->isLastIntermediateNodeProcessed())` block, after existing processing

**Code to add**:
```cpp
// Check maxAllowablePaymentAmount after path completion
if (mCommand->maxAllowablePaymentAmount().has_value()) {
    TrustLineAmount totalReserved = calculateTotalReservedPaymentAmount();

    if (totalReserved > *mCommand->maxAllowablePaymentAmount()) {
        warning() << "Total reserved payment amount " << totalReserved
                  << " exceeds maximum allowable payment amount "
                  << *mCommand->maxAllowablePaymentAmount()
                  << " after processing path " << pathID;

        // Call reject() without return - triggers rollBack() in base class
        reject("Allowable payment amount exceeded");

        // Return specific error code
        return resultAllowablePaymentAmountExceeded();
    }

    debug() << "Total reserved amount " << totalReserved
            << " within allowable limit " << *mCommand->maxAllowablePaymentAmount();
}
```

**Pattern explanation**:
- Follows existing pattern at line 2590-2591
- `reject()` called without return to trigger `BaseExchangePaymentTransaction::reject()` → `rollBack()`
- Then return specific result code 415

## Step 4: Add Late Validation in processNeighborFurtherReservationResponse()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

**Location**: Inside `if (path->isLastIntermediateNodeProcessed())` block, after existing processing

**Code**: Identical to Step 3, but log message includes "via neighbor":
```cpp
warning() << "Total reserved payment amount " << totalReserved
          << " exceeds maximum allowable payment amount "
          << *mCommand->maxAllowablePaymentAmount()
          << " after processing path " << pathID << " via neighbor";
```

# Test Plan

## Manual Testing Scenarios

### Test 1: calculateTotalReservedPaymentAmount() with No Reservations
- **Setup**: Transaction with no outgoing reservations
- **Action**: Call `calculateTotalReservedPaymentAmount()`
- **Expected**: Returns `TrustLineAmount(0)`

### Test 2: calculateTotalReservedPaymentAmount() with One Equivalent
- **Setup**: One exchange equivalent with outgoing reservations totaling 1000
- **Action**: Call `calculateTotalReservedPaymentAmount()`
- **Expected**: Returns `TrustLineAmount(1000)`

### Test 3: calculateTotalReservedPaymentAmount() with Multiple Equivalents
- **Setup**: Two exchange equivalents with outgoing reservations: 1200 and 600
- **Action**: Call `calculateTotalReservedPaymentAmount()`
- **Expected**: Returns `TrustLineAmount(1800)`

### Test 4: Early Validation Passes (Within Limit)
- **Setup**: `maxAllowablePaymentAmount = 2000`, calculated `mExchangeAmount = 1800`
- **Action**: Run `runPathsResourceProcessingStage()`
- **Expected**: Validation passes, method continues to next step, info log written

### Test 5: Early Validation Fails (Exceeds Limit)
- **Setup**: `maxAllowablePaymentAmount = 1500`, calculated `mExchangeAmount = 1800`
- **Action**: Run `runPathsResourceProcessingStage()`
- **Expected**: Returns result code 415, warning logged with both amounts, no reservations made

### Test 6: Early Validation Skipped (No Parameter)
- **Setup**: `maxAllowablePaymentAmount = nullopt`, calculated `mExchangeAmount = 10000`
- **Action**: Run `runPathsResourceProcessingStage()`
- **Expected**: Validation skipped, method continues normally

### Test 7: Late Validation Passes (processRemoteNodeResponse)
- **Setup**: `maxAllowablePaymentAmount = 2000`, path completed, total reserved = 1800
- **Action**: Process path completion in `processRemoteNodeResponse()`
- **Expected**: Validation passes, processing continues, debug log written

### Test 8: Late Validation Fails (processRemoteNodeResponse)
- **Setup**: `maxAllowablePaymentAmount = 1500`, path completed, total reserved = 1800
- **Action**: Process path completion in `processRemoteNodeResponse()`
- **Expected**:
  - Warning logged with amounts and pathID
  - `reject()` called (triggers `rollBack()`)
  - Returns result code 415
  - All reservations dropped

### Test 9: Late Validation Fails Even When mAmount Achieved
- **Setup**: `maxAllowablePaymentAmount = 1500`, target `mAmount` achieved, total reserved = 1800
- **Action**: Process path completion
- **Expected**: Transaction still aborts with result code 415 despite achieving target

### Test 10: Late Validation Skipped (No Parameter)
- **Setup**: `maxAllowablePaymentAmount = nullopt`, total reserved = 10000
- **Action**: Process path completion
- **Expected**: Validation skipped, processing continues

### Test 11: Late Validation in processNeighborFurtherReservationResponse
- **Setup**: Same as Test 8 but path via neighbor
- **Action**: Process path completion in `processNeighborFurtherReservationResponse()`
- **Expected**: Same as Test 8, log message includes "via neighbor"

### Test 12: Rollback Mechanism Verification
- **Setup**: Late validation fails after 2 paths reserved
- **Action**: Trigger validation failure
- **Expected**:
  - `reject()` triggers `BaseExchangePaymentTransaction::rollBack()`
  - All reservations on all paths dropped
  - Verify reservation state after rejection

## Complexity
**Complex** - Involves multiple validation points, integration with existing reservation system, and coordination with rollback mechanism. Requires thorough testing of different scenarios.

# Verification and Validation

## Architecture integrity
- Uses existing `totalReservedAmount()` API for aggregation
- Integrates with existing `reject()` and `rollBack()` infrastructure
- No new cleanup mechanisms (reuses existing)
- Validation points strategically placed in transaction flow
- Follows existing pattern from line 2590-2591

## Security
- No security implications (internal validation)
- Parameter validated at parsing time
- No exposure of internal state

## Performance
- `calculateTotalReservedPaymentAmount()`: O(n) where n = number of exchange equivalents (typically 1-5)
- Validation overhead < 5ms per check
- No impact on happy path when parameter not provided
- No additional memory overhead

## Scalability
- Efficient calculation (O(n) with small n)
- Supports up to 5 exchange equivalents (existing limit)
- No scalability concerns for validation checks

## Reliability
- Validation failures handled gracefully via existing `reject()` mechanism
- All reservations properly cleaned up via existing `rollBack()`
- Transaction state consistent after abort
- Proper error reporting to user
- Calculation errors handled by existing `totalReservedAmount()` API

## Maintainability
- Clear validation logic with descriptive logging
- Follows existing patterns (line 2590-2591)
- No new infrastructure to maintain
- Self-documenting through method names
- Strategic placement of validation checks

## Cost
- Development effort: 1-2 days
- No infrastructure cost changes
- Minimal runtime overhead (validation only when parameter provided)

## Compliance
- Follows project coding standards
- Adheres to existing transaction patterns
- Uses established infrastructure
- No breaking changes

# Restrictions
- Commit changes only after successfully passing all manual tests
- Ensure all reservations are dropped when validation fails (verify via debugging)
- Do not modify base class `reject()` or `rollBack()` methods
- Do not add new cleanup mechanisms (use existing infrastructure)
