# 06-04 - Reservation Upgrading by Equivalent

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task 1](06-02-data-structures-and-transaction-classes.md)
- [Previous task 2](06-03-path-processing.md)

# Description
Implement multi-equivalent reservation support across transaction classes and message protocol. This includes:
1. Using PathReservation structure in mNodesFinalAmountsConfiguration
2. Extending RequestMessageWithReservations and inheritors with reservation equivalents
3. Implementing updateReservations() with equivalent validation in ReceiverExchangePaymentTransaction and IntermediateNodeExchangePaymentTransaction
4. Implementing checkReservationsDirections() with multi-equivalent logic, exchange rate conversion, and commission handling

This task enables proper reservation tracking and validation across different equivalents during payment execution.

# Requirements and DOD

## Requirements

### PathReservation in mNodesFinalAmountsConfiguration
1. Replace `vector<pair<PathID, ConstSharedTrustLineAmount>>` with `vector<PathReservation>` in mNodesFinalAmountsConfiguration
2. Update all usages in CoordinatorExchangePaymentTransaction, ReceiverExchangePaymentTransaction, IntermediateNodeExchangePaymentTransaction

### RequestMessageWithReservations Enhancement
3. Extend RequestMessageWithReservations to serialize/deserialize reservation equivalents
4. Two constructor variants:
   - With equivalents: accepts `vector<PathReservation>`
   - Without equivalents (deprecated): accepts `vector<pair<PathID, ConstSharedTrustLineAmount>>`, uses mEquivalent for all
5. Both constructors produce same serialization format (always include equivalents)
6. Update all inheritors: IntermediateNodeReservationRequestMessage, CoordinatorReservationRequestMessage, etc.

### updateReservations() Enhancement
7. Implement in ReceiverExchangePaymentTransaction with equivalent validation
8. Implement in IntermediateNodeExchangePaymentTransaction with equivalent validation
9. Validate that reservation equivalent matches expected equivalent for (pathID, equivalent) pair
10. Reject updates with mismatched equivalents

### checkReservationsDirections() Implementation
11. Implement in ReceiverExchangePaymentTransaction with multi-equivalent logic
12. Implement in IntermediateNodeExchangePaymentTransaction with exchange and commission logic
13. Integration with ExchangeRatesManager for exchange rate conversion
14. Integration with CommissionsManager for commission lookup
15. Follow 5 validation rules:
    - All outgoing reservations must be in single equivalent
    - Incoming reservations can be in multiple equivalents
    - Convert incoming amounts to outgoing equivalent using exchange rates
    - Deduct commission once per incoming equivalent only if same as outgoing (transit only, not exchange)
    - Return false if exchange rate not found or sums don't match

## Definition of Done
- [x] mNodesFinalAmountsConfiguration uses `vector<PathReservation>` in all transaction classes
- [x] All usages updated to use PathReservation fields (pathID, amount, equivalent)
- [x] RequestMessageWithReservations has two constructors (with/without equivalents)
- [x] RequestMessageWithReservations serializes/deserializes equivalents correctly
- [x] All inheritor messages updated (IntermediateNodeReservationRequestMessage, etc.)
- [x] Backward compatibility: deprecated constructor uses mEquivalent for all reservations
- [x] updateReservations() implemented in ReceiverExchangePaymentTransaction
- [x] updateReservations() implemented in IntermediateNodeExchangePaymentTransaction
- [x] updateReservations() validates equivalent matches
- [x] checkReservationsDirections() implemented in ReceiverExchangePaymentTransaction
- [x] checkReservationsDirections() implemented in IntermediateNodeExchangePaymentTransaction
- [x] checkReservationsDirections() validates all 5 rules
- [x] ExchangeRatesManager integration functional
- [x] CommissionsManager integration functional
- [x] Code compiles without errors

# Implementation Plan

## Step 1: Update mNodesFinalAmountsConfiguration Type
**Files**:
- `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`
- `src/core/transactions/transactions/regular/payments/ReceiverExchangePaymentTransaction.h`
- `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h`

Change declaration:
```cpp
// OLD
map<NodeUUID, vector<pair<PathID, ConstSharedTrustLineAmount>>> mNodesFinalAmountsConfiguration;

// NEW
map<NodeUUID, vector<PathReservation>> mNodesFinalAmountsConfiguration;
```

Update all usages:
```cpp
// OLD
for (const auto& [pathID, amount] : reservations) { ... }

// NEW
for (const auto& reservation : reservations) {
    PathID pathID = reservation.pathID;
    ConstSharedTrustLineAmount amount = reservation.amount;
    SerializedEquivalent equiv = reservation.equivalent;
    ...
}
```

## Step 2: Extend RequestMessageWithReservations
**File**: `src/core/network/messages/payments/base/RequestMessageWithReservations.h`

Add second constructor:
```cpp
class RequestMessageWithReservations : public TransactionMessage
{
public:
    // NEW: Constructor with equivalents
    RequestMessageWithReservations(
        const SerializedEquivalent equivalent,
        vector<BaseAddress::Shared> &senderAddresses,
        const TransactionUUID &transactionUUID,
        const vector<PathReservation> &finalAmountsConfig);

    // DEPRECATED: Constructor without equivalents (uses mEquivalent for all)
    RequestMessageWithReservations(
        const SerializedEquivalent equivalent,
        vector<BaseAddress::Shared> &senderAddresses,
        const TransactionUUID &transactionUUID,
        const vector<pair<PathID, ConstSharedTrustLineAmount>> &finalAmountsConfig);

    // Existing deserialization constructor
    RequestMessageWithReservations(BytesShared buffer);

    const vector<PathReservation> &finalAmountsConfiguration() const;

protected:
    virtual pair<BytesShared, size_t> serializeToBytes() const override;

private:
    vector<PathReservation> mFinalAmountsConfiguration;  // Changed type
};
```

**File**: `src/core/network/messages/payments/base/RequestMessageWithReservations.cpp`

Implement constructors:
```cpp
// NEW: With equivalents
RequestMessageWithReservations::RequestMessageWithReservations(
    const SerializedEquivalent equivalent,
    vector<BaseAddress::Shared> &senderAddresses,
    const TransactionUUID &transactionUUID,
    const vector<PathReservation> &finalAmountsConfig)
    :
    TransactionMessage(equivalent, senderAddresses, transactionUUID),
    mFinalAmountsConfiguration(finalAmountsConfig)
{}

// DEPRECATED: Without equivalents
RequestMessageWithReservations::RequestMessageWithReservations(
    const SerializedEquivalent equivalent,
    vector<BaseAddress::Shared> &senderAddresses,
    const TransactionUUID &transactionUUID,
    const vector<pair<PathID, ConstSharedTrustLineAmount>> &finalAmountsConfig)
    :
    TransactionMessage(equivalent, senderAddresses, transactionUUID)
{
    // Convert pair vector to PathReservation vector using mEquivalent
    mFinalAmountsConfiguration.reserve(finalAmountsConfig.size());
    for (const auto& [pathID, amount] : finalAmountsConfig) {
        mFinalAmountsConfiguration.emplace_back(pathID, amount, equivalent);
    }
}
```

Implement serialization:
```cpp
pair<BytesShared, size_t> RequestMessageWithReservations::serializeToBytes() const
{
    // ... existing header serialization

    // Serialize reservations count
    size_t reservationsCount = mFinalAmountsConfiguration.size();
    memcpy(buffer + bytesBufferOffset, &reservationsCount, sizeof(reservationsCount));
    bytesBufferOffset += sizeof(reservationsCount);

    // Serialize each reservation (PathID + Amount + Equivalent)
    for (const auto& reservation : mFinalAmountsConfiguration) {
        memcpy(buffer + bytesBufferOffset, &reservation.pathID, sizeof(PathID));
        bytesBufferOffset += sizeof(PathID);

        // Serialize amount (TrustLineAmount)
        vector<byte> amountBytes = trustLineAmountToBytes(*reservation.amount);
        memcpy(buffer + bytesBufferOffset, amountBytes.data(), amountBytes.size());
        bytesBufferOffset += amountBytes.size();

        // Serialize equivalent
        memcpy(buffer + bytesBufferOffset, &reservation.equivalent, sizeof(SerializedEquivalent));
        bytesBufferOffset += sizeof(SerializedEquivalent);
    }

    return make_pair(buffer, bytesBufferOffset);
}
```

Implement deserialization:
```cpp
RequestMessageWithReservations::RequestMessageWithReservations(BytesShared buffer)
    :
    TransactionMessage(buffer)
{
    size_t bytesBufferOffset = TransactionMessage::kOffsetToInheritedBytes();

    // Deserialize reservations count
    size_t reservationsCount;
    memcpy(&reservationsCount, buffer.get() + bytesBufferOffset, sizeof(reservationsCount));
    bytesBufferOffset += sizeof(reservationsCount);

    mFinalAmountsConfiguration.reserve(reservationsCount);

    // Deserialize each reservation
    for (size_t i = 0; i < reservationsCount; ++i) {
        PathID pathID;
        memcpy(&pathID, buffer.get() + bytesBufferOffset, sizeof(PathID));
        bytesBufferOffset += sizeof(PathID);

        // Deserialize amount
        TrustLineAmount amount = bytesToTrustLineAmount(buffer.get() + bytesBufferOffset);
        bytesBufferOffset += kTrustLineAmountBytesCount;

        // Deserialize equivalent
        SerializedEquivalent equivalent;
        memcpy(&equivalent, buffer.get() + bytesBufferOffset, sizeof(SerializedEquivalent));
        bytesBufferOffset += sizeof(SerializedEquivalent);

        mFinalAmountsConfiguration.emplace_back(
            pathID,
            make_shared<const TrustLineAmount>(amount),
            equivalent);
    }
}
```

## Step 3: Update All Inheritor Messages
Update constructors in:
- IntermediateNodeReservationRequestMessage
- CoordinatorReservationRequestMessage
- IntermediateNodeCycleReservationRequestMessage
- CoordinatorCycleReservationRequestMessage

Each should have two constructors forwarding to RequestMessageWithReservations.

## Step 4: Implement updateReservations() with Equivalent Validation
**Files**:
- `src/core/transactions/transactions/regular/payments/ReceiverExchangePaymentTransaction.cpp`
- `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.cpp`

```cpp
bool ReceiverExchangePaymentTransaction::updateReservations(
    const vector<PathReservation> &finalAmounts)
{
    debug() << "updateReservations";

    for (const auto& [contractorID, reservations] : mReservations) {
        for (auto& [pathID, reservation] : reservations) {
            // Find matching final amount by pathID and equivalent
            bool found = false;
            for (const auto& finalAmount : finalAmounts) {
                if (finalAmount.pathID == pathID &&
                    finalAmount.equivalent == reservation->equivalent()) {
                    // Update reservation to final amount
                    // ... existing update logic
                    found = true;
                    break;
                }
            }

            if (!found) {
                // Reservation with (pathID, equivalent) not in finalAmounts
                // Should be dropped or validated
                // ... existing logic
            }
        }
    }

    return true;  // Or appropriate validation result
}
```

**Key Point**: Match reservations by BOTH pathID AND equivalent.

## Step 5: Implement checkReservationsDirections() in ReceiverExchangePaymentTransaction
**File**: `src/core/transactions/transactions/regular/payments/ReceiverExchangePaymentTransaction.cpp`

Receiver has only incoming reservations, simpler validation:
```cpp
bool ReceiverExchangePaymentTransaction::checkReservationsDirections() const
{
    debug() << "checkReservationsDirections";

    // For receiver: sum all incoming reservations (may be in different equivalents)
    // All should sum to expected amount in mEquivalent

    TrustLineAmount totalIncoming = TrustLineAmount(0);

    for (const auto& [contractorID, reservations] : mReservations) {
        for (const auto& [pathID, reservation] : reservations) {
            if (reservation->direction() == AmountReservation::Incoming) {
                // All incoming should be in receiver's equivalent (mEquivalent)
                if (reservation->equivalent() != mEquivalent) {
                    warning() << "Incoming reservation in wrong equivalent";
                    return false;
                }
                totalIncoming = totalIncoming + reservation->amount();
            }
        }
    }

    // Validate total matches expected amount
    return totalIncoming == mExpectedAmount;
}
```

## Step 6: Implement checkReservationsDirections() in IntermediateNodeExchangePaymentTransaction
**File**: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.cpp`

```cpp
bool IntermediateNodeExchangePaymentTransaction::checkReservationsDirections() const
{
    debug() << "checkReservationsDirections";

    // Step 1: Determine outgoing equivalent and validate uniformity
    SerializedEquivalent outgoingEquivalent;
    bool outgoingEquivalentSet = false;
    TrustLineAmount totalOutgoing = TrustLineAmount(0);

    for (const auto& [contractorID, reservations] : mReservations) {
        for (const auto& [pathID, reservation] : reservations) {
            if (reservation->direction() == AmountReservation::Outgoing) {
                if (!outgoingEquivalentSet) {
                    outgoingEquivalent = reservation->equivalent();
                    outgoingEquivalentSet = true;
                } else if (outgoingEquivalent != reservation->equivalent()) {
                    // All outgoing must be in same equivalent
                    warning() << "Outgoing reservations in multiple equivalents";
                    return false;
                }
                totalOutgoing = totalOutgoing + reservation->amount();
            }
        }
    }

    if (!outgoingEquivalentSet) {
        warning() << "No outgoing reservations found";
        return false;
    }

    // Step 2: Calculate total incoming converted to outgoing equivalent
    TrustLineAmount totalIncomingConverted = TrustLineAmount(0);
    set<SerializedEquivalent> processedIncomingEquivalents;

    for (const auto& [contractorID, reservations] : mReservations) {
        for (const auto& [pathID, reservation] : reservations) {
            if (reservation->direction() == AmountReservation::Incoming) {
                SerializedEquivalent incomingEquiv = reservation->equivalent();
                TrustLineAmount incomingAmount = reservation->amount();

                if (incomingEquiv == outgoingEquivalent) {
                    // Same equivalent - direct add
                    totalIncomingConverted = totalIncomingConverted + incomingAmount;

                    // Step 3: Deduct commission if same equivalent (transit only)
                    if (processedIncomingEquivalents.find(incomingEquiv) ==
                        processedIncomingEquivalents.end()) {
                        auto commission = mCommissionsManager->get(incomingEquiv);
                        if (commission) {
                            totalIncomingConverted = totalIncomingConverted - commission->amount();
                        }
                        processedIncomingEquivalents.insert(incomingEquiv);
                    }
                } else {
                    // Different equivalent - convert using exchange rate
                    auto rate = mExchangeRatesManager->get(incomingEquiv, outgoingEquivalent);
                    if (!rate) {
                        warning() << "No exchange rate from " << incomingEquiv
                                  << " to " << outgoingEquivalent;
                        return false;
                    }

                    try {
                        TrustLineAmount converted = mExchangeRatesManager->calculateConvertedAmount(
                            incomingEquiv, outgoingEquivalent, incomingAmount);
                        totalIncomingConverted = totalIncomingConverted + converted;
                    } catch (const Exception& e) {
                        warning() << "Conversion error: " << e.what();
                        return false;
                    }

                    // No commission for exchange operations
                }
            }
        }
    }

    // Step 4: Compare totals
    if (totalIncomingConverted != totalOutgoing) {
        warning() << "Totals don't match: incoming=" << totalIncomingConverted
                  << ", outgoing=" << totalOutgoing;
        return false;
    }

    return true;
}
```

## Step 7: Add Manager Dependencies
**Files**:
- `src/core/transactions/transactions/regular/payments/ReceiverExchangePaymentTransaction.h`
- `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h`

Add to constructor parameters and members:
```cpp
private:
    ExchangeRatesManager *mExchangeRatesManager;
    CommissionsManager *mCommissionsManager;
```

Constructor:
```cpp
IntermediateNodeExchangePaymentTransaction(
    ...,
    ExchangeRatesManager *exchangeRatesManager,
    CommissionsManager *commissionsManager,
    ...)
    :
    BaseExchangePaymentTransaction(...),
    mExchangeRatesManager(exchangeRatesManager),
    mCommissionsManager(commissionsManager)
{...}
```

# Test Plan

## Unit Tests
As this is a Complex task, comprehensive testing required:

### Test Category: PathReservation in mNodesFinalAmountsConfiguration
1. **testMNodesFinalAmountsConfigurationUsesPathReservation**: Type verification
2. **testPathReservationFieldAccess**: Access pathID, amount, equivalent from reservation

### Test Category: RequestMessageWithReservations
3. **testRequestMessageWithReservationsNewConstructor**: Constructor with PathReservation vector
4. **testRequestMessageWithReservationsDeprecatedConstructor**: Constructor with pair vector
5. **testRequestMessageWithReservationsSerializationWithEquivalents**: Serialize with equivalents
6. **testRequestMessageWithReservationsDeserializationWithEquivalents**: Deserialize with equivalents
7. **testRequestMessageWithReservationsRoundTrip**: Serialize then deserialize preserves data
8. **testRequestMessageWithReservationsBackwardCompatibility**: Deprecated constructor produces same format

### Test Category: updateReservations()
9. **testUpdateReservationsValidatesEquivalent**: Match by pathID and equivalent
10. **testUpdateReservationsRejectsWrongEquivalent**: Reject update with mismatched equivalent
11. **testUpdateReservationsMultipleEquivalents**: Update reservations in different equivalents

### Test Category: checkReservationsDirections() - Receiver (Simple)
12. **testReceiverCheckReservationsDirectionsAllIncoming**: All incoming in receiver equivalent
13. **testReceiverCheckReservationsDirectionsWrongEquivalent**: Incoming in wrong equivalent → false
14. **testReceiverCheckReservationsDirectionsSumMatches**: Total matches expected amount

### Test Category: checkReservationsDirections() - Intermediate (Complex - 11 scenarios)
15. **testIntermediateCheckAllOutgoingInSingleEquivalent**:
    - Setup: outgoing in equiv 1 only
    - Expected: validation proceeds

16. **testIntermediateCheckOutgoingInMultipleEquivalentsFail**:
    - Setup: outgoing in equiv 1 and equiv 2
    - Expected: return false

17. **testIntermediateCheckIncomingSameAsOutgoingNoCommission**:
    - Setup: incoming equiv 1 (100), outgoing equiv 1 (100), no commission
    - Expected: totals match, return true

18. **testIntermediateCheckIncomingSameAsOutgoingWithCommission**:
    - Setup: incoming equiv 1 (110), outgoing equiv 1 (100), commission equiv 1 = 10
    - Expected: totalIncomingConverted = 110 - 10 = 100, return true

19. **testIntermediateCheckIncomingDifferentFromOutgoingConvert**:
    - Setup: incoming equiv 1 (100), outgoing equiv 2 (200), rate 1→2 = 2.0
    - Expected: converted = 200, return true

20. **testIntermediateCheckMultipleIncomingEquivalents**:
    - Setup: incoming equiv 1 (100), incoming equiv 2 (50), outgoing equiv 3 (250), rates 1→3=2.0, 2→3=3.0
    - Expected: 100*2.0 + 50*3.0 = 250, return true

21. **testIntermediateCheckCommissionOnlyForTransit**:
    - Setup: incoming equiv 1 (110), incoming equiv 2 (50), outgoing equiv 1 (200), commission equiv 1 = 10, rate 2→1 = 2.0
    - Expected: (110 - 10) + 50*2.0 = 200, return true

22. **testIntermediateCheckCommissionChargedOncePerEquivalent**:
    - Setup: two incoming reservations in equiv 1 (60 each), outgoing equiv 1 (110), commission equiv 1 = 10
    - Expected: (60 + 60) - 10 = 110, return true (commission deducted only once)

23. **testIntermediateCheckNoExchangeRateFoundFail**:
    - Setup: incoming equiv 1 (100), outgoing equiv 2 (200), no rate 1→2
    - Expected: return false

24. **testIntermediateCheckOverflowDuringConversionFail**:
    - Setup: incoming with MAX_AMOUNT, outgoing with high rate causing overflow
    - Expected: catch exception, return false

25. **testIntermediateCheckSumsDontMatchFail**:
    - Setup: incoming equiv 1 (100), outgoing equiv 2 (201), rate 1→2 = 2.0
    - Expected: converted = 200 ≠ 201, return false

## Success Criteria
- All 25 unit tests pass
- Code compiles without errors or warnings
- Multi-equivalent reservation validation works correctly
- Exchange rate conversion accurate
- Commission "charge once" semantics enforced

# Verification and Validation

## Architecture integrity
- Clean separation: message protocol, transaction validation logic, manager integration
- PathReservation provides consistent structure across components
- updateReservations() validates by both pathID and equivalent
- checkReservationsDirections() encapsulates all validation rules

## Security
- Equivalent validation prevents cross-equivalent fraud
- Exchange rate validation prevents invalid conversions
- Commission calculation validated before commitment

## Performance
- updateReservations(): O(n*m) where n=reservations, m=finalAmounts (typically small)
- checkReservationsDirections(): O(n) where n=reservations
- Exchange rate lookup: O(1) via hash map
- Commission lookup: O(1) via hash map

## Scalability
- Supports up to 5 exchange equivalents (design limit)
- Handles multiple reservations per equivalent
- Efficient set-based tracking for processed equivalents

## Reliability
- Comprehensive validation prevents inconsistent reservations
- Graceful handling of missing exchange rates or commissions
- Clear error logging for debugging

## Maintainability
- checkReservationsDirections() algorithm clearly documented with 5 rules
- Separate handling for same-equivalent vs different-equivalent cases
- Commission "charge once" logic isolated and clear

## Cost
- No additional infrastructure required
- Manager integrations use existing infrastructure

## Compliance
- Follows repository policy for task-driven development
- Adheres to existing message protocol patterns
- No prohibited operations or scope creep

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not modify ExchangeRatesManager or CommissionsManager (use existing interfaces)
- Ensure backward compatibility for deprecated constructors
