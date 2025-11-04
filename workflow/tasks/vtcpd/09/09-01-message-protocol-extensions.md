# 09-01 - Message Protocol Extensions for Exchange Rate and Commission Validation

# Links
- [PRD](../../../prd/vtcpd/09-exchange-rate-commission-change-handling.md)

# Description
Extend the payment protocol messages to support transmission of expected and actual exchange rates and commissions between coordinator and intermediate nodes. This enables intermediate nodes to validate reservation requests against their current conditions and inform the coordinator about mismatches.

This task adds optional fields to CoordinatorReservationRequestMessage and CoordinatorReservationResponseMessage, along with a new response type RejectedDueConditionsChanged, forming the foundation for dynamic condition change handling during payment execution.

# Requirements and DOD

## Requirements

### R1: CoordinatorReservationRequestMessage Extension
- Add `optional<pair<TrustLineAmount, int16_t>> mExpectedExchangeRate` field to CoordinatorReservationRequestMessage
  - Stores only rate and shift (pair.first = exchangeRate, pair.second = exchangeRateShift)
  - equivalentFrom/To not stored - derived from reservation context at intermediate node
- Add `optional<TrustLineAmount> mExpectedCommission` field to CoordinatorReservationRequestMessage
- Implement `const optional<pair<TrustLineAmount, int16_t>>& expectedExchangeRate() const` getter
- Implement `const optional<TrustLineAmount>& expectedCommission() const` getter
- Add two new constructors (in addition to default constructor):
  - Constructor with expected exchange rate: takes rate and shift as separate parameters
  - Constructor with expected commission: takes commission amount as parameter
  - Default constructor leaves both fields as nullopt
- Update message serialization to include optional fields:
  - For ExchangeRate: serialize only `exchangeRate` (TrustLineAmount) and `exchangeRateShift` (int16_t)
  - For commission: serialize the TrustLineAmount directly
  - Use byte_t flags (0/1) to indicate field presence
- Update message deserialization to handle optional fields
- Ensure backward compatibility with messages not containing these fields
- Note: Coordinator extracts rate and shift from full ExchangeRate object when populating message

### R2: CoordinatorReservationResponseMessage Extension
- Add `optional<pair<TrustLineAmount, int16_t>> mActualExchangeRate` field to CoordinatorReservationResponseMessage
  - Stores only rate and shift (pair.first = exchangeRate, pair.second = exchangeRateShift)
  - equivalentFrom/To not stored - derived from reservation context at coordinator
- Add `optional<TrustLineAmount> mActualCommission` field to CoordinatorReservationResponseMessage
- Implement `const optional<pair<TrustLineAmount, int16_t>>& actualExchangeRate() const` getter
- Implement `const optional<TrustLineAmount>& actualCommission() const` getter
- Add two new constructors (in addition to default constructor):
  - Constructor with actual exchange rate: takes reservedAmount, rate and shift as parameters
  - Constructor with actual commission: takes reservedAmount and commission as parameters
  - Default constructor leaves both optional fields as nullopt
- Update message serialization to include optional fields:
  - For ExchangeRate: serialize only `exchangeRate` (TrustLineAmount) and `exchangeRateShift` (int16_t)
  - For commission: serialize the TrustLineAmount directly
  - Use byte_t flags (0/1) to indicate field presence
- Update message deserialization to handle optional fields
- Ensure backward compatibility

### R3: RejectedDueConditionsChanged Response Type
- Add `RejectedDueConditionsChanged = 12` to ResponseMessage::OperationState enum
- Ensure the value doesn't conflict with existing enum values
- Update any switch statements or response handling logic to recognize the new state

## Definition of Done

1. CoordinatorReservationRequestMessage successfully serializes and deserializes with:
   - Only expectedExchangeRate populated
   - Only expectedCommission populated
   - Both fields empty (nullopt)
   - Messages from previous protocol versions (without new fields)

2. CoordinatorReservationResponseMessage successfully serializes and deserializes with:
   - Only actualExchangeRate populated
   - Only actualCommission populated
   - Both fields empty (nullopt)
   - Messages from previous protocol versions (without new fields)

3. RejectedDueConditionsChanged enum value added and recognized in response processing

4. All existing message handling code continues to work (backward compatibility)

5. Unit tests pass for all message serialization/deserialization scenarios

6. Code compiles without warnings

# Implementation Plan

## Step 1: Extend CoordinatorReservationRequestMessage

**File**: `src/core/network/messages/payments/CoordinatorReservationRequestMessage.h`

### 1.1: Add fields to class definition
```cpp
protected:
    // ... existing fields ...

    // New fields for condition validation
    // Exchange rate stored as pair<rate, shift> - equivalents derived from context
    optional<pair<TrustLineAmount, int16_t>> mExpectedExchangeRate;
    optional<TrustLineAmount> mExpectedCommission;

public:
    // New getters
    const optional<pair<TrustLineAmount, int16_t>>& expectedExchangeRate() const;
    const optional<TrustLineAmount>& expectedCommission() const;
```

### 1.2: Update serialization (in .cpp)
```cpp
pair<BytesShared, size_t> CoordinatorReservationRequestMessage::serializeToBytes() const
{
    BytesSerializer serializer;
    serializer.enqueue(RequestMessageWithReservations::serializeToBytes());

    // Serialize mNextPathNode (existing field)
    auto serializedAddress = mNextPathNode->serializeToBytes();
    serializer.enqueue(serializedAddress, mNextPathNode->serializedSize());

    // Serialize expected exchange rate (only rate and shift, NOT equivalents or dates)
    if (mExpectedExchangeRate.has_value()) {
        serializer.copy((byte_t)1);  // Flag: exchange rate present
        serializer.copy(mExpectedExchangeRate->exchangeRate());  // TrustLineAmount
        serializer.copy(mExpectedExchangeRate->exchangeRateShift());  // int16_t
    } else {
        serializer.copy((byte_t)0);  // Flag: no exchange rate
    }

    // Serialize expected commission
    if (mExpectedCommission.has_value()) {
        serializer.copy((byte_t)1);  // Flag: commission present
        serializer.copy(*mExpectedCommission);  // TrustLineAmount
    } else {
        serializer.copy((byte_t)0);  // Flag: no commission
    }

    return serializer.collect();
}
```

**Note**: Only `exchangeRate` and `exchangeRateShift` are serialized. The intermediate node reconstructs the full context:
- **equivalentTo**: from the outgoing reservation equivalent being requested (available in message context)
- **equivalentFrom**: from the incoming reservation equivalent already created on the same path (available in transaction reservations)
- With the pair (equivalentFrom, equivalentTo), the intermediate node can:
  1. Look up the full ExchangeRate object from ExchangeRatesManager
  2. Compare the received (rate, shift) with the current ExchangeRate from manager
  3. Store the complete ExchangeRate object in transaction context if validation passes
- Other fields (expiresAt, min/maxExchangeAmount) are retrieved from ExchangeRatesManager, not from the message

### 1.3: Update deserialization (in .cpp)
```cpp
// In constructor from BytesShared
CoordinatorReservationRequestMessage::CoordinatorReservationRequestMessage(
    BytesShared buffer) :
    RequestMessageWithReservations(buffer)
{
    size_t offset = RequestMessageWithReservations::kOffsetToInheritedBytes();

    // Deserialize mNextPathNode (existing field)
    mNextPathNode = deserializeAddress(buffer.get() + offset);
    offset += mNextPathNode->serializedSize();

    // Deserialize expected exchange rate flag
    byte_t hasExchangeRate = buffer.get()[offset];
    offset += sizeof(byte_t);

    if (hasExchangeRate == 1) {
        // Deserialize only rate and shift
        TrustLineAmount rate;
        int16_t shift;

        memcpy(&rate, buffer.get() + offset, sizeof(TrustLineAmount));
        offset += sizeof(TrustLineAmount);

        memcpy(&shift, buffer.get() + offset, sizeof(int16_t));
        offset += sizeof(int16_t);

        // Store as struct with only rate and shift
        // Note: equivalents will be determined from context by intermediate node
        mExpectedExchangeRate = {rate, shift};
    }
    // else mExpectedExchangeRate remains nullopt

    // Deserialize expected commission flag
    byte_t hasCommission = buffer.get()[offset];
    offset += sizeof(byte_t);

    if (hasCommission == 1) {
        TrustLineAmount commission;
        memcpy(&commission, buffer.get() + offset, sizeof(TrustLineAmount));
        offset += sizeof(TrustLineAmount);

        mExpectedCommission = commission;
    }
    // else mExpectedCommission remains nullopt
}
```

**Important**: Since we only serialize rate and shift, we need a lightweight structure to store these values. We'll store the full ExchangeRate object but only the rate and shift fields will be meaningful from the message - equivalents and other fields will be reconstructed from context.

### 1.4: Implement getters
```cpp
const optional<pair<TrustLineAmount, int16_t>>&
CoordinatorReservationRequestMessage::expectedExchangeRate() const
{
    return mExpectedExchangeRate;
}

const optional<TrustLineAmount>&
CoordinatorReservationRequestMessage::expectedCommission() const
{
    return mExpectedCommission;
}
```

## Step 2: Extend CoordinatorReservationResponseMessage

**File**: `src/core/network/messages/payments/CoordinatorReservationResponseMessage.h`

### 2.1: Add fields and methods (similar structure to Step 1)
```cpp
protected:
    // Exchange rate stored as pair<rate, shift> - equivalents derived from context
    optional<pair<TrustLineAmount, int16_t>> mActualExchangeRate;
    optional<TrustLineAmount> mActualCommission;

public:
    const optional<pair<TrustLineAmount, int16_t>>& actualExchangeRate() const;
    const optional<TrustLineAmount>& actualCommission() const;
```

### 2.2: Implement serialization/deserialization (in .cpp)
- Follow same pattern as CoordinatorReservationRequestMessage
- Ensure flags and field ordering are consistent

### 2.3: Implement getters
```cpp
const optional<pair<TrustLineAmount, int16_t>>&
CoordinatorReservationResponseMessage::actualExchangeRate() const
{
    return mActualExchangeRate;
}

const optional<TrustLineAmount>&
CoordinatorReservationResponseMessage::actualCommission() const
{
    return mActualCommission;
}
```

## Step 3: Add RejectedDueConditionsChanged Response Type

**File**: `src/core/network/messages/payments/base/ResponseMessage.h`

### 3.1: Extend OperationState enum
```cpp
enum OperationState {
    Accepted = 1,
    Rejected = 2,
    // ... other existing values ...
    RejectedDueObserving = 11,
    RejectedDueConditionsChanged = 12,  // NEW
};
```

### 3.2: Verify no conflicts
- Check that value 12 is not already used
- Ensure enum can accommodate future additions

## Step 4: Backward Compatibility Testing

### 4.1: Test old message handling
- Ensure messages without new fields deserialize correctly
- Verify optional fields default to nullopt when not present

### 4.2: Test new message handling
- Ensure new messages with fields serialize/deserialize correctly
- Verify both old and new nodes can coexist (new node receives old message format)

# Test Plan

## Test Scope
This task focuses on message protocol extensions only. Validation logic and condition change handling are tested in subsequent tasks.

## Unit Tests

### Test Suite: CoordinatorReservationRequestMessage

**Test 1: Serialize with expectedExchangeRate only**
- Create message with expectedExchangeRate = ExchangeRate{value: 5, shift: -1}
- expectedCommission = nullopt
- Serialize to bytes
- Deserialize from bytes
- Verify expectedExchangeRate matches original
- Verify expectedCommission is nullopt

**Test 2: Serialize with expectedCommission only**
- Create message with expectedCommission = TrustLineAmount(10)
- expectedExchangeRate = nullopt
- Serialize to bytes
- Deserialize from bytes
- Verify expectedCommission matches original
- Verify expectedExchangeRate is nullopt

**Test 3: Serialize with both fields empty**
- Create message with both fields = nullopt
- Serialize to bytes
- Deserialize from bytes
- Verify both fields remain nullopt

**Test 4: Backward compatibility - deserialize old format**
- Create byte buffer representing old message format (without new fields)
- Deserialize into new message class
- Verify both new fields are nullopt
- Verify all existing fields deserialize correctly

**Test 5: Boundary values**
- Test with maximum TrustLineAmount for commission
- Test with extreme ExchangeRate values (very large/small)
- Verify correct serialization/deserialization

### Test Suite: CoordinatorReservationResponseMessage

**Test 6-10: Same structure as Tests 1-5**
- Replace expectedExchangeRate/expectedCommission with actualExchangeRate/actualCommission
- Test appropriate constructors for creating responses with different field combinations

**Test 11: Constructor-based response creation**
- Create response using appropriate constructor with actualExchangeRate
- Verify field is set correctly
- Create response using appropriate constructor with actualCommission
- Verify field is set correctly

### Test Suite: ResponseMessage::OperationState

**Test 12: RejectedDueConditionsChanged recognized**
- Create ResponseMessage with state = RejectedDueConditionsChanged
- Verify state() returns RejectedDueConditionsChanged
- Verify state compares equal to enum value 12

**Test 13: No enum conflicts**
- Verify all OperationState values are unique
- Verify value 12 is assigned to RejectedDueConditionsChanged

## Success Criteria
- All 13 unit tests pass
- No compilation warnings
- Backward compatibility verified with old message formats
- New fields serialize/deserialize correctly in all combinations

# Verification and Validation

## Architecture integrity
**Validation Level**: Simple task
- Message protocol extensions follow existing message structure patterns
- No changes to core transaction logic
- Follows existing serialization/deserialization conventions
- Fields are properly encapsulated (optional usage)

## Security
**Validation Level**: Simple task
- No security implications (message structure extension only)
- Optional fields prevent null pointer issues
- Serialization uses bounded types (ExchangeRate, TrustLineAmount)

## Performance
**Validation Level**: Simple task
- Serialization overhead: +2 bytes (flags) + variable data (when fields present)
- Minimal impact: typically +10-30 bytes per message
- No algorithmic complexity changes
- Deserialization overhead negligible

## Scalability
**Validation Level**: Simple task
- Message size increases are bounded and small
- No impact on message throughput
- Network bandwidth impact negligible (<1% increase)

## Reliability
**Validation Level**: Simple task
- Backward compatibility ensures mixed version deployments work
- Optional fields prevent deserialization failures
- Well-defined default behavior (nullopt)

## Maintainability
**Validation Level**: Simple task
- Clear field naming (expectedExchangeRate, actualExchangeRate, etc.)
- Consistent with existing codebase patterns
- Standard optional<T> usage

## Cost
**Validation Level**: Simple task
- No infrastructure cost changes
- Minimal bandwidth cost increase

## Compliance
**Validation Level**: Simple task
- Follows project policy for message protocol extensions
- Adheres to coding standards

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not modify transaction logic (out of scope for this task)
- Ensure backward compatibility is maintained

# Implementation Notes

## Data Type Optimization
The implementation uses `optional<pair<TrustLineAmount, int16_t>>` instead of `optional<ExchangeRate>` for message fields to optimize serialization:

**Rationale**:
- Only `exchangeRate` (TrustLineAmount) and `exchangeRateShift` (int16_t) are needed for validation
- `equivalentFrom` and `equivalentTo` are derived from context at both coordinator and intermediate node
- `expiresAt`, `minExchangeAmount`, `maxExchangeAmount` are not needed in messages

**Usage Pattern**:

1. **Coordinator side** (populating request):
   ```cpp
   // Get full ExchangeRate from manager or path data
   ExchangeRate fullRate = /* from ExchangeRatesManager */;

   // Create message with exchange rate using constructor
   auto message = make_shared<CoordinatorReservationRequestMessage>(
       equivalent,
       senderAddresses,
       transactionUUID,
       finalAmountsConfig,
       nextNodeInThePath,
       fullRate.exchangeRate(),     // rate parameter
       fullRate.exchangeRateShift()  // shift parameter
   );

   // Or create message with commission using constructor:
   TrustLineAmount commission = /* from CommissionsManager */;
   auto message = make_shared<CoordinatorReservationRequestMessage>(
       equivalent,
       senderAddresses,
       transactionUUID,
       finalAmountsConfig,
       nextNodeInThePath,
       commission  // commission parameter
   );
   ```

2. **Intermediate node side** (validating):
   ```cpp
   auto receivedPair = request->expectedExchangeRate();
   SerializedEquivalent fromEquiv = /* from incoming reservation on pathID */;
   SerializedEquivalent toEquiv = /* from outgoing reservation being requested */;

   ExchangeRate fullRate = exchangeRatesManager->get(fromEquiv, toEquiv);

   if (fullRate.exchangeRate() != receivedPair->first ||
       fullRate.exchangeRateShift() != receivedPair->second) {
       // Mismatch - create rejection response with actual exchange rate
       auto response = make_shared<CoordinatorReservationResponseMessage>(
           equivalent,
           transactionUUID,
           pathID,
           ResponseMessage::RejectedDueConditionsChanged,
           reservedAmount,
           fullRate.exchangeRate(),      // rate parameter
           fullRate.exchangeRateShift()  // shift parameter
       );
       return response;
   }

   // Store complete ExchangeRate in transaction context
   mContextExchangeRates[make_pair(fromEquiv, toEquiv)] = fullRate;
   ```

3. **Coordinator side** (receiving response):
   ```cpp
   auto actualPair = response->actualExchangeRate();
   SerializedEquivalent fromEquiv = /* known from path being processed */;
   SerializedEquivalent toEquiv = /* known from path being processed */;

   // Update manager with new rate
   exchangeRatesManager->set(fromEquiv, toEquiv, actualPair->first, actualPair->second);
   ```

This approach minimizes message size while maintaining full functionality through context reconstruction.
