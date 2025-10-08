# 06-05 - AmountReservation + AmountReservationsHandler Upgrading by Equivalent

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task](06-04-reservation-upgrading-by-equivalent.md)

# Description
Extend AmountReservation and AmountReservationsHandler to support equivalent tracking. This enables the reservation system to distinguish between reservations in different equivalents on the same trust line, which is essential for multi-equivalent payments.

Changes include:
1. Adding SerializedEquivalent field to AmountReservation
2. Adding SerializedEquivalent parameter to all AmountReservationsHandler methods
3. Updating all call sites to provide equivalent parameter

# Requirements and DOD

## Requirements

### AmountReservation Enhancement
1. Add `SerializedEquivalent mEquivalent` field to AmountReservation
2. Add `const SerializedEquivalent& equivalent() const` getter method
3. Update constructor to accept `SerializedEquivalent equivalent` parameter
4. Constructor signature: `AmountReservation(const TransactionUUID &transactionUUID, const TrustLineAmount &amount, const ReservationDirection direction, const SerializedEquivalent equivalent)`

### AmountReservationsHandler Enhancement
5. Add SerializedEquivalent parameter to `reserve()` method
6. Add SerializedEquivalent parameter to `updateReservation()` method
7. Add SerializedEquivalent parameter to `free()` method
8. Add SerializedEquivalent parameter to `totalReserved()` method
9. Add SerializedEquivalent parameter to `getReservation()` method
10. All methods use equivalent when creating/finding/updating AmountReservation objects
11. Support multiple reservations with same amount/direction but different equivalents on same trust line

### Call Sites Updates
12. Update all call sites in BasePaymentTransaction to always pass mEquivalent
13. Update all call sites in derived old transaction classes to pass mEquivalent
14. Update all call sites in BaseExchangePaymentTransaction and derived classes to pass appropriate equivalent from context

## Definition of Done
- [x] AmountReservation has mEquivalent field
- [x] AmountReservation has equivalent() getter
- [x] AmountReservation constructor accepts equivalent parameter
- [x] AmountReservationsHandler::reserve() accepts SerializedEquivalent parameter
- [x] AmountReservationsHandler::updateReservation() accepts SerializedEquivalent parameter
- [x] AmountReservationsHandler::free() accepts SerializedEquivalent parameter
- [x] AmountReservationsHandler::totalReserved() accepts SerializedEquivalent parameter
- [x] AmountReservationsHandler::getReservation() accepts SerializedEquivalent parameter
- [x] All methods use equivalent when working with AmountReservation
- [x] Multiple reservations with different equivalents supported
- [x] All call sites updated to provide equivalent
- [x] Code compiles without errors

# Implementation Plan

## Step 1: Update AmountReservation Class
**File**: `src/core/payments/reservations/AmountReservation.h`

Add field:
```cpp
class AmountReservation
{
public:
    // ... existing typedefs and enums

public:
    AmountReservation(
        const TransactionUUID &transactionUUID,
        const TrustLineAmount &amount,
        const ReservationDirection direction,
        const SerializedEquivalent equivalent);  // NEW parameter

    const TrustLineAmount& amount() const;
    const TransactionUUID& transactionUUID() const;
    const ReservationDirection direction() const;
    const SerializedEquivalent& equivalent() const;  // NEW getter

    // ... existing operators

protected:
    TrustLineAmount mBlockedAmount;
    TransactionUUID mTransactionUUID;
    ReservationDirection mDirection;
    SerializedEquivalent mEquivalent;  // NEW field
};
```

**File**: `src/core/payments/reservations/AmountReservation.cpp`

Update constructor:
```cpp
AmountReservation::AmountReservation(
    const TransactionUUID &transactionUUID,
    const TrustLineAmount &amount,
    const ReservationDirection direction,
    const SerializedEquivalent equivalent)
    :
    mTransactionUUID(transactionUUID),
    mBlockedAmount(amount),
    mDirection(direction),
    mEquivalent(equivalent)  // NEW initialization
{}
```

Add getter:
```cpp
const SerializedEquivalent& AmountReservation::equivalent() const
{
    return mEquivalent;
}
```

Update operators if they compare equivalents:
```cpp
bool AmountReservation::operator==(const AmountReservation &rhs) const
{
    return mTransactionUUID == rhs.mTransactionUUID &&
           mBlockedAmount == rhs.mBlockedAmount &&
           mDirection == rhs.mDirection &&
           mEquivalent == rhs.mEquivalent;  // NEW comparison
}
```

## Step 2: Update AmountReservationsHandler Methods
**File**: `src/core/payments/reservations/AmountReservationsHandler.h`

Update method signatures:
```cpp
class AmountReservationsHandler
{
public:
    AmountReservation::ConstShared reserve(
        ContractorID trustLineContractor,
        const TransactionUUID &transactionUUID,
        const TrustLineAmount &amount,
        const AmountReservation::ReservationDirection direction,
        const SerializedEquivalent equivalent);  // NEW parameter

    AmountReservation::ConstShared updateReservation(
        ContractorID trustLineContractor,
        const AmountReservation::ConstShared reservation,
        const TrustLineAmount &newAmount,
        const SerializedEquivalent equivalent);  // NEW parameter

    void free(
        ContractorID trustLineContractor,
        const AmountReservation::ConstShared reservation,
        const SerializedEquivalent equivalent);  // NEW parameter

    ConstSharedTrustLineAmount totalReserved(
        ContractorID trustLineContractor,
        const AmountReservation::ReservationDirection direction,
        const SerializedEquivalent equivalent,  // NEW parameter
        const TransactionUUID *transactionUUID = nullptr) const;

    AmountReservation::ConstShared getReservation(
        ContractorID trustLineContractor,
        const TransactionUUID &transactionUUID,
        const TrustLineAmount &amount,
        const AmountReservation::ReservationDirection direction,
        const SerializedEquivalent equivalent);  // NEW parameter

    // ... existing helper methods
};
```

**File**: `src/core/payments/reservations/AmountReservationsHandler.cpp`

Update `reserve()`:
```cpp
AmountReservation::ConstShared AmountReservationsHandler::reserve(
    ContractorID trustLineContractor,
    const TransactionUUID &transactionUUID,
    const TrustLineAmount &amount,
    const AmountReservation::ReservationDirection direction,
    const SerializedEquivalent equivalent)
{
    auto reservation = make_shared<const AmountReservation>(
        transactionUUID,
        amount,
        direction,
        equivalent);  // NEW parameter

    // ... existing logic to add to mReservations

    return reservation;
}
```

Update `updateReservation()`:
```cpp
AmountReservation::ConstShared AmountReservationsHandler::updateReservation(
    ContractorID trustLineContractor,
    const AmountReservation::ConstShared reservation,
    const TrustLineAmount &newAmount,
    const SerializedEquivalent equivalent)
{
    // ... existing logic to find and remove old reservation

    // Create new reservation with updated amount and equivalent
    auto updatedReservation = make_shared<const AmountReservation>(
        reservation->transactionUUID(),
        newAmount,
        reservation->direction(),
        equivalent);  // NEW parameter

    // ... existing logic to add updated reservation

    return updatedReservation;
}
```

Update `free()`:
```cpp
void AmountReservationsHandler::free(
    ContractorID trustLineContractor,
    const AmountReservation::ConstShared reservation,
    const SerializedEquivalent equivalent)
{
    // ... existing logic to find and remove reservation
    // Now also match by equivalent when searching

    auto it = mReservations.find(trustLineContractor);
    if (it == mReservations.end()) {
        return;
    }

    auto& reservations = *it->second;
    for (auto resIt = reservations.begin(); resIt != reservations.end(); ++resIt) {
        if ((*resIt)->transactionUUID() == reservation->transactionUUID() &&
            (*resIt)->amount() == reservation->amount() &&
            (*resIt)->direction() == reservation->direction() &&
            (*resIt)->equivalent() == equivalent) {  // NEW check
            reservations.erase(resIt);
            return;
        }
    }
}
```

Update `totalReserved()`:
```cpp
ConstSharedTrustLineAmount AmountReservationsHandler::totalReserved(
    ContractorID trustLineContractor,
    const AmountReservation::ReservationDirection direction,
    const SerializedEquivalent equivalent,
    const TransactionUUID *transactionUUID) const
{
    TrustLineAmount total = TrustLineAmount(0);

    auto reservations = this->reservations(trustLineContractor, transactionUUID);
    for (const auto& reservation : reservations) {
        if (reservation->direction() == direction &&
            reservation->equivalent() == equivalent) {  // NEW filter
            total = total + reservation->amount();
        }
    }

    return make_shared<const TrustLineAmount>(total);
}
```

Update `getReservation()`:
```cpp
AmountReservation::ConstShared AmountReservationsHandler::getReservation(
    ContractorID trustLineContractor,
    const TransactionUUID &transactionUUID,
    const TrustLineAmount &amount,
    const AmountReservation::ReservationDirection direction,
    const SerializedEquivalent equivalent)
{
    auto it = mReservations.find(trustLineContractor);
    if (it == mReservations.end()) {
        throw NotFoundError("No reservations for contractor");
    }

    for (const auto& reservation : *it->second) {
        if (reservation->transactionUUID() == transactionUUID &&
            reservation->amount() == amount &&
            reservation->direction() == direction &&
            reservation->equivalent() == equivalent) {  // NEW match
            return reservation;
        }
    }

    throw NotFoundError("Reservation not found");
}
```

## Step 3: Update All Call Sites
**Search for all usages:**
```bash
grep -r "->reserve(" src/core/
grep -r "->updateReservation(" src/core/
grep -r "->free(" src/core/
grep -r "->totalReserved(" src/core/
grep -r "->getReservation(" src/core/
```

**In old transaction classes** (always use mEquivalent):
```cpp
// OLD
mTrustLinesManager->amountReservationsHandler()->reserve(
    contractorID, transactionUUID, amount, direction);

// NEW
mTrustLinesManager->amountReservationsHandler()->reserve(
    contractorID, transactionUUID, amount, direction, mEquivalent);
```

**In new transaction classes** (use context-appropriate equivalent):
```cpp
// Example in CoordinatorExchangePaymentTransaction
mTrustLinesManager->amountReservationsHandler()->reserve(
    contractorID, transactionUUID, amount, direction, reservationEquivalent);
```

# Test Plan

## Unit Tests
As this is a Moderate task, focused testing on core functionality:

### Test Category: AmountReservation
1. **testAmountReservationConstructorWithEquivalent**: Create reservation with equivalent
2. **testAmountReservationEquivalentGetter**: Verify equivalent() returns correct value
3. **testAmountReservationEqualityWithEquivalent**: operator== considers equivalent

### Test Category: AmountReservationsHandler::reserve()
4. **testReserveWithEquivalent**: Reserve with specific equivalent
   - Expected: Reservation created with correct equivalent

5. **testReserveMultipleEquivalentsSameContractor**: Reserve in equiv 1 and equiv 2 on same trust line
   - Expected: Both reservations exist independently

### Test Category: AmountReservationsHandler::updateReservation()
6. **testUpdateReservationWithEquivalent**: Update reservation amount
   - Expected: Updated reservation has same equivalent

7. **testUpdateReservationDifferentEquivalent**: Update with different equivalent parameter
   - Expected: Updated reservation uses new equivalent

### Test Category: AmountReservationsHandler::free()
8. **testFreeReservationWithEquivalent**: Free reservation in specific equivalent
   - Setup: Reservations in equiv 1 and equiv 2
   - Free equiv 1
   - Expected: equiv 1 removed, equiv 2 remains

9. **testFreeReservationMatchesEquivalent**: Free matches by equivalent
   - Setup: Same amount/direction in equiv 1 and equiv 2
   - Free equiv 1
   - Expected: Only equiv 1 removed

### Test Category: AmountReservationsHandler::totalReserved()
10. **testTotalReservedFiltersEquivalent**: Total for specific equivalent
    - Setup: reservations in equiv 1 (100), equiv 2 (50), equiv 1 (30)
    - Call with equiv 1: Expected 130
    - Call with equiv 2: Expected 50

11. **testTotalReservedMultipleEquivalents**: Total across multiple equivalents
    - Setup: equiv 1 (100), equiv 2 (200), equiv 3 (300)
    - Verify each equivalent separately

### Test Category: AmountReservationsHandler::getReservation()
12. **testGetReservationWithEquivalent**: Get reservation by equivalent
    - Setup: Same amount/direction in equiv 1 and equiv 2
    - Get equiv 1: Expected equiv 1 reservation
    - Get equiv 2: Expected equiv 2 reservation

13. **testGetReservationNotFoundWrongEquivalent**: Request non-existent equivalent
    - Setup: Reservation in equiv 1
    - Get equiv 2: Expected NotFoundError

### Test Category: Edge Cases
14. **testMultipleReservationsSameParametersDifferentEquivalents**: Multiple reservations with identical parameters except equivalent
    - Setup: Create 3 reservations with same amount, direction, but equiv 1, 2, 3
    - Expected: All 3 exist independently

15. **testReservationEquivalentIndependence**: Operations on one equivalent don't affect another
    - Setup: Reservations in equiv 1 and equiv 2
    - Update equiv 1
    - Expected: equiv 2 unchanged

### Test Category: Call Sites
16. **testCallSitesInOldTransactionsUseMEquivalent**: Verify old transactions always pass mEquivalent
17. **testCallSitesInNewTransactionsUseContextEquivalent**: Verify new transactions pass appropriate equivalent

## Success Criteria
- All 17 unit tests pass
- Code compiles without errors or warnings
- Multiple reservations with different equivalents supported on same trust line
- All handler methods correctly filter by equivalent

# Verification and Validation

## Architecture integrity
- AmountReservation extended cleanly with equivalent field
- AmountReservationsHandler methods maintain single responsibility
- Equivalent treated as first-class reservation property alongside amount/direction

## Security
- Equivalent validation prevents cross-equivalent reservation confusion
- Proper filtering ensures reservations isolated by equivalent

## Performance
- reserve(): O(1) insertion (unchanged)
- updateReservation(): O(n) search where n=reservations per contractor
- free(): O(n) search and removal
- totalReserved(): O(n) iteration with equivalent filter
- getReservation(): O(n) search with equivalent match
- All acceptable for typical reservation counts (< 20 per contractor)

## Scalability
- Supports multiple equivalents per trust line
- No limit on number of equivalents (within system constraints)
- Linear scaling with reservation count

## Reliability
- Equivalent matching prevents accidental cross-equivalent operations
- Clear error messages for not found cases
- Robust filtering logic

## Maintainability
- Consistent parameter ordering (equivalent always last)
- Clear method signatures with equivalent parameter
- Easy to understand equivalent filtering logic

## Cost
- Minimal memory overhead (one SerializedEquivalent per reservation)
- No additional infrastructure required

## Compliance
- Follows repository policy for task-driven development
- Maintains existing AmountReservationsHandler interface pattern
- No prohibited operations or scope creep

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not change existing reservation storage structure (vector of reservations per contractor)
- Ensure all call sites updated (use compiler errors as checklist)
