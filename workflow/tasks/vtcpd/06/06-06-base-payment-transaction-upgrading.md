# 06-06 - BasePaymentTransaction + BaseExchangePaymentTransaction Upgrading

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task](06-05-amount-reservation-handler-upgrading.md)

# Description
Extend BasePaymentTransaction and BaseExchangePaymentTransaction with equivalent-aware helper methods and serialization support for reservation equivalents. This includes:
1. Adding SerializedEquivalent parameter to totalReservedIncomingAmountToNode() and totalReservedAmount()
2. Extending serializeToBytes() to include reservation equivalents
3. Updating deserialization constructor to read reservation equivalents
4. Updating all call sites to provide equivalent parameter

These methods enable per-equivalent reservation tracking and persistent storage of multi-equivalent transaction state.

# Requirements and DOD

## Requirements

### Helper Methods Enhancement
1. Add SerializedEquivalent parameter to totalReservedIncomingAmountToNode(ContractorID contractorID, SerializedEquivalent equivalent)
2. Add SerializedEquivalent parameter to totalReservedAmount(ReservationDirection direction, SerializedEquivalent equivalent)
3. Both methods calculate totals only for specified equivalent
4. Update all call sites in BasePaymentTransaction and derived classes to provide equivalent
5. In old transaction classes (BasePaymentTransaction, CoordinatorPaymentTransaction, etc.), always pass mEquivalent
6. In new transaction classes (BaseExchangePaymentTransaction, CoordinatorExchangePaymentTransaction, etc.), pass appropriate equivalent per context

### Serialization Enhancement
7. Extend serializeToBytes() in BasePaymentTransaction to serialize reservation equivalents
8. Serialization format: for each reservation, add SerializedEquivalent after PathID and Amount
9. Update deserialization constructor in BasePaymentTransaction to read reservation equivalents
10. Backward compatibility: old serialized data handled by using mEquivalent for all reservations (detected by data format version or length)
11. Same enhancements apply to BaseExchangePaymentTransaction

## Definition of Done
- [x] totalReservedIncomingAmountToNode() accepts SerializedEquivalent parameter in BasePaymentTransaction
- [x] totalReservedIncomingAmountToNode() accepts SerializedEquivalent parameter in BaseExchangePaymentTransaction
- [x] Method calculates total only for specified equivalent
- [x] totalReservedAmount() accepts SerializedEquivalent parameter in BasePaymentTransaction
- [x] totalReservedAmount() accepts SerializedEquivalent parameter in BaseExchangePaymentTransaction
- [x] Method calculates total only for specified equivalent and direction
- [x] All call sites in BasePaymentTransaction updated to pass mEquivalent
- [x] All call sites in derived old transaction classes use mEquivalent
- [x] All call sites in new transaction classes pass appropriate equivalent
- [x] serializeToBytes() in BasePaymentTransaction serializes reservation equivalents
- [x] serializeToBytes() in BaseExchangePaymentTransaction serializes reservation equivalents
- [x] Deserialization constructor in BasePaymentTransaction reads equivalents
- [x] Deserialization constructor in BaseExchangePaymentTransaction reads equivalents
- [x] Round-trip serialization preserves all data including equivalents
- [x] Code compiles without errors

# Implementation Plan

## Step 1: Update totalReservedIncomingAmountToNode() Signature and Implementation
**File**: `src/core/transactions/transactions/regular/payments/base/BasePaymentTransaction.h`

Update declaration:
```cpp
// OLD
const TrustLineAmount totalReservedIncomingAmountToNode(
    ContractorID contractorID);

// NEW
const TrustLineAmount totalReservedIncomingAmountToNode(
    ContractorID contractorID,
    const SerializedEquivalent equivalent);
```

**File**: `src/core/transactions/transactions/regular/payments/base/BasePaymentTransaction.cpp`

Update implementation:
```cpp
const TrustLineAmount BasePaymentTransaction::totalReservedIncomingAmountToNode(
    ContractorID contractorID,
    const SerializedEquivalent equivalent)
{
    TrustLineAmount total = TrustLineAmount(0);

    auto it = mReservations.find(contractorID);
    if (it == mReservations.end()) {
        return total;
    }

    for (const auto& [pathID, reservation] : it->second) {
        // NEW: Filter by equivalent
        if (reservation->direction() == AmountReservation::Incoming &&
            reservation->equivalent() == equivalent) {
            total = total + reservation->amount();
        }
    }

    return total;
}
```

**Apply same changes to BaseExchangePaymentTransaction.**

## Step 2: Update totalReservedAmount() Signature and Implementation
**File**: `src/core/transactions/transactions/regular/payments/base/BasePaymentTransaction.h`

Update declaration:
```cpp
// OLD
const TrustLineAmount totalReservedAmount(
    AmountReservation::ReservationDirection reservationDirection) const;

// NEW
const TrustLineAmount totalReservedAmount(
    AmountReservation::ReservationDirection reservationDirection,
    const SerializedEquivalent equivalent) const;
```

**File**: `src/core/transactions/transactions/regular/payments/base/BasePaymentTransaction.cpp`

Update implementation:
```cpp
const TrustLineAmount BasePaymentTransaction::totalReservedAmount(
    AmountReservation::ReservationDirection reservationDirection,
    const SerializedEquivalent equivalent) const
{
    TrustLineAmount total = TrustLineAmount(0);

    for (const auto& [contractorID, reservations] : mReservations) {
        for (const auto& [pathID, reservation] : reservations) {
            // NEW: Filter by direction AND equivalent
            if (reservation->direction() == reservationDirection &&
                reservation->equivalent() == equivalent) {
                total = total + reservation->amount();
            }
        }
    }

    return total;
}
```

**Apply same changes to BaseExchangePaymentTransaction.**

## Step 3: Update All Call Sites
**Search for all usages:**
```bash
grep -r "totalReservedIncomingAmountToNode" src/core/transactions/
grep -r "totalReservedAmount" src/core/transactions/
```

**Update each call site:**

In old transaction classes (BasePaymentTransaction, CoordinatorPaymentTransaction, ReceiverPaymentTransaction, IntermediateNodePaymentTransaction):
```cpp
// OLD
auto total = totalReservedIncomingAmountToNode(contractorID);

// NEW
auto total = totalReservedIncomingAmountToNode(contractorID, mEquivalent);
```

In new transaction classes (BaseExchangePaymentTransaction and derived):
```cpp
// Use appropriate equivalent from context
// For example, in checkReservationsDirections():
auto total = totalReservedAmount(AmountReservation::Outgoing, outgoingEquivalent);
```

## Step 4: Extend serializeToBytes() in BasePaymentTransaction
**File**: `src/core/transactions/transactions/regular/payments/base/BasePaymentTransaction.cpp`

Current serialization likely includes reservations like:
```cpp
// Serialize reservations
for (const auto& [contractorID, reservations] : mReservations) {
    for (const auto& [pathID, reservation] : reservations) {
        // serialize contractorID, pathID, amount, direction
    }
}
```

Update to include equivalent:
```cpp
pair<BytesShared, size_t> BasePaymentTransaction::serializeToBytes() const
{
    // ... existing serialization (type, UUID, etc.)

    // Serialize reservations count
    size_t reservationsCount = 0;
    for (const auto& [contractorID, reservations] : mReservations) {
        reservationsCount += reservations.size();
    }
    memcpy(buffer + bytesBufferOffset, &reservationsCount, sizeof(reservationsCount));
    bytesBufferOffset += sizeof(reservationsCount);

    // Serialize each reservation
    for (const auto& [contractorID, reservations] : mReservations) {
        for (const auto& [pathID, reservation] : reservations) {
            // Serialize contractorID
            memcpy(buffer + bytesBufferOffset, &contractorID, sizeof(ContractorID));
            bytesBufferOffset += sizeof(ContractorID);

            // Serialize pathID
            memcpy(buffer + bytesBufferOffset, &pathID, sizeof(PathID));
            bytesBufferOffset += sizeof(PathID);

            // Serialize amount
            vector<byte> amountBytes = trustLineAmountToBytes(reservation->amount());
            memcpy(buffer + bytesBufferOffset, amountBytes.data(), amountBytes.size());
            bytesBufferOffset += amountBytes.size();

            // Serialize direction
            AmountReservation::SerializedReservationDirectionSize direction =
                static_cast<AmountReservation::SerializedReservationDirectionSize>(
                    reservation->direction());
            memcpy(buffer + bytesBufferOffset, &direction, sizeof(direction));
            bytesBufferOffset += sizeof(direction);

            // NEW: Serialize equivalent
            SerializedEquivalent equiv = reservation->equivalent();
            memcpy(buffer + bytesBufferOffset, &equiv, sizeof(SerializedEquivalent));
            bytesBufferOffset += sizeof(SerializedEquivalent);
        }
    }

    // ... rest of serialization

    return make_pair(buffer, bytesBufferOffset);
}
```

**Apply same changes to BaseExchangePaymentTransaction.**

## Step 5: Update Deserialization Constructor
**File**: `src/core/transactions/transactions/regular/payments/base/BasePaymentTransaction.cpp`

```cpp
BasePaymentTransaction::BasePaymentTransaction(
    BytesShared buffer,
    ...)
{
    // ... existing deserialization (type, UUID, etc.)

    size_t bytesBufferOffset = /* current offset */;

    // Deserialize reservations count
    size_t reservationsCount;
    memcpy(&reservationsCount, buffer.get() + bytesBufferOffset, sizeof(reservationsCount));
    bytesBufferOffset += sizeof(reservationsCount);

    // Deserialize each reservation
    for (size_t i = 0; i < reservationsCount; ++i) {
        // Deserialize contractorID
        ContractorID contractorID;
        memcpy(&contractorID, buffer.get() + bytesBufferOffset, sizeof(ContractorID));
        bytesBufferOffset += sizeof(ContractorID);

        // Deserialize pathID
        PathID pathID;
        memcpy(&pathID, buffer.get() + bytesBufferOffset, sizeof(PathID));
        bytesBufferOffset += sizeof(PathID);

        // Deserialize amount
        TrustLineAmount amount = bytesToTrustLineAmount(buffer.get() + bytesBufferOffset);
        bytesBufferOffset += kTrustLineAmountBytesCount;

        // Deserialize direction
        AmountReservation::SerializedReservationDirectionSize direction;
        memcpy(&direction, buffer.get() + bytesBufferOffset, sizeof(direction));
        bytesBufferOffset += sizeof(direction);

        // NEW: Deserialize equivalent
        SerializedEquivalent equivalent;
        memcpy(&equivalent, buffer.get() + bytesBufferOffset, sizeof(SerializedEquivalent));
        bytesBufferOffset += sizeof(SerializedEquivalent);

        // Create reservation
        auto reservation = make_shared<const AmountReservation>(
            currentTransactionUUID(),
            amount,
            static_cast<AmountReservation::ReservationDirection>(direction),
            equivalent);

        // Add to mReservations
        mReservations[contractorID].emplace_back(pathID, reservation);
    }

    // ... rest of deserialization
}
```

**Backward Compatibility Note**: If old serialized data exists without equivalents, need to detect and use mEquivalent. This can be done via:
1. Format version field at start of serialization
2. Check buffer size (old format will be shorter)
3. For this task, assume new deployment only (no backward compatibility needed as per PRD)

**Apply same changes to BaseExchangePaymentTransaction.**

## Step 6: Verify Round-Trip Serialization
Create test that:
1. Creates transaction with multi-equivalent reservations
2. Calls serializeToBytes()
3. Creates new transaction from bytes
4. Verifies all reservations with correct equivalents

# Test Plan

## Unit Tests
As this is a Moderate task, focused testing on core functionality:

### Test Category: totalReservedIncomingAmountToNode()
1. **testTotalReservedIncomingAmountToNodeSingleEquivalent**:
   - Setup: 3 incoming reservations to same contractor in equiv 1
   - Expected: Sum of all 3 reservations

2. **testTotalReservedIncomingAmountToNodeMultipleEquivalents**:
   - Setup: incoming reservations in equiv 1 (100), equiv 2 (50), equiv 1 (30)
   - Call with equiv 1: Expected 130
   - Call with equiv 2: Expected 50

3. **testTotalReservedIncomingAmountToNodeFiltersOutgoing**:
   - Setup: incoming (100) and outgoing (50) in equiv 1
   - Expected: Only incoming counted (100)

4. **testTotalReservedIncomingAmountToNodeNoReservations**:
   - Setup: No reservations for contractor
   - Expected: 0

### Test Category: totalReservedAmount()
5. **testTotalReservedAmountOutgoingSingleEquivalent**:
   - Setup: 3 outgoing reservations in equiv 1
   - Expected: Sum of all 3

6. **testTotalReservedAmountIncomingMultipleEquivalents**:
   - Setup: incoming in equiv 1 (100), equiv 2 (50), equiv 3 (25)
   - Call with equiv 1: Expected 100
   - Call with equiv 2: Expected 50
   - Call with equiv 3: Expected 25

7. **testTotalReservedAmountFiltersDirection**:
   - Setup: incoming (100) and outgoing (50) both in equiv 1
   - Call Incoming, equiv 1: Expected 100
   - Call Outgoing, equiv 1: Expected 50

8. **testTotalReservedAmountNoReservations**:
   - Expected: 0

### Test Category: Call Sites
9. **testOldTransactionCallSitesUseMEquivalent**:
   - Verify BasePaymentTransaction call sites always pass mEquivalent

10. **testNewTransactionCallSitesUseContextEquivalent**:
    - Verify BaseExchangePaymentTransaction call sites pass appropriate equivalent from context

### Test Category: Serialization
11. **testSerializeToBytesIncludesEquivalents**:
    - Setup: Create transaction with reservations in multiple equivalents
    - Call serializeToBytes()
    - Manually inspect buffer: verify equivalents present after each reservation

12. **testSerializeToBytesReservationsCount**:
    - Setup: 5 reservations
    - Expected: Count=5 serialized correctly

### Test Category: Deserialization
13. **testDeserializationReadsEquivalents**:
    - Setup: Buffer with reservations containing equivalents
    - Create transaction from buffer
    - Expected: All reservations have correct equivalents

14. **testDeserializationMultipleEquivalents**:
    - Setup: Buffer with reservations in equiv 1, 2, 3
    - Expected: Reservations correctly reconstructed with respective equivalents

### Test Category: Round-Trip
15. **testRoundTripSerializationPreservesEquivalents**:
    - Setup: Transaction with reservations in equiv 1 (100), equiv 2 (50)
    - Serialize then deserialize
    - Expected: All data preserved including equivalents

16. **testRoundTripSerializationMultipleReservations**:
    - Setup: 10 reservations across 3 equivalents
    - Round-trip
    - Expected: All 10 reservations preserved with correct equivalents

## Success Criteria
- All 16 unit tests pass
- Code compiles without errors or warnings
- All call sites updated correctly
- Round-trip serialization works correctly

# Verification and Validation

## Architecture integrity
- Helper methods provide clean interface for per-equivalent totals
- Serialization format extends existing format (add equivalent field)
- No breaking changes to method signatures (only parameter addition)

## Security
- Equivalent tracking prevents cross-equivalent calculation errors
- Serialization includes all necessary data for recovery

## Performance
- totalReservedIncomingAmountToNode(): O(n) where n=reservations per contractor
- totalReservedAmount(): O(n) where n=total reservations
- Serialization: O(n) where n=reservations
- Deserialization: O(n)
- All acceptable for typical reservation counts (< 50)

## Scalability
- Handles multiple equivalents per transaction
- Serialization size increases linearly with reservations

## Reliability
- Equivalent filtering ensures correct calculations
- Round-trip serialization preserves all state
- Clear error handling for malformed data

## Maintainability
- Method signatures clearly indicate equivalent parameter required
- Serialization format documented and consistent
- Easy to understand equivalent filtering logic

## Cost
- Minimal memory overhead (one SerializedEquivalent per reservation in buffer)
- No additional infrastructure required

## Compliance
- Follows repository policy for task-driven development
- Maintains backward compatibility approach (though not needed for this deployment)
- No prohibited operations or scope creep

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not modify existing BasePaymentTransaction serialization logic beyond adding equivalent field
- Ensure all call sites updated (use compiler errors as checklist)
