# 06-07 - Message Extensions for Multiple Receipts

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task](06-04-reservation-upgrading-by-equivalent.md)

# Description
Extend FinalAmountsConfigurationMessage and TransactionPublicKeyHashMessage to support multiple payment receipts per trust line (one per equivalent). This enables nodes to track separate debt records for each equivalent on the same trust line during multi-equivalent payments.

Changes include:
1. Replacing single receipt fields (mIsReceiptContains, mSignature) with vector of (equivalent, signature) pairs
2. Updating isReceiptContains() to check vector size
3. Updating serialization/deserialization to handle receipt vector
4. Maintaining backward compatibility for single-equivalent transactions

# Requirements and DOD

## Requirements

### FinalAmountsConfigurationMessage
1. Remove `bool mIsReceiptContains` field
2. Remove `sphincs::Signature::Shared mSignature` field
3. Add `vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> mSignatures` field
4. Update isReceiptContains() to return `!mSignatures.empty()`
5. Add signatures() getter returning the vector
6. Two constructor variants:
   - With signatures vector (new transactions)
   - With single signature (old transactions, creates single-element vector with mEquivalent)
7. Serialize/deserialize signatures vector

### TransactionPublicKeyHashMessage
8. Remove `bool mIsReceiptContains` field
9. Remove `sphincs::Signature::Shared mSignature` field
10. Add `vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> mSignatures` field
11. Update isReceiptContains() to return `!mSignatures.empty()`
12. Add signatures() getter returning the vector
13. Two constructor variants:
   - With signatures vector (new transactions)
   - With single signature (old transactions, creates single-element vector with mEquivalent)
14. Serialize/deserialize signatures vector

### Backward Compatibility
15. Old transaction constructors create single-element vector with {mEquivalent, signature}
16. Empty signature case: old constructor creates empty vector
17. Serialization format handles both empty and non-empty vectors

## Definition of Done
- [x] FinalAmountsConfigurationMessage: mIsReceiptContains and mSignature removed
- [x] FinalAmountsConfigurationMessage: mSignatures vector added
- [x] FinalAmountsConfigurationMessage: isReceiptContains() returns !mSignatures.empty()
- [x] FinalAmountsConfigurationMessage: signatures() getter implemented
- [x] FinalAmountsConfigurationMessage: Two constructors (with vector, with single signature)
- [x] FinalAmountsConfigurationMessage: Serialization/deserialization works
- [x] TransactionPublicKeyHashMessage: mIsReceiptContains and mSignature removed
- [x] TransactionPublicKeyHashMessage: mSignatures vector added
- [x] TransactionPublicKeyHashMessage: isReceiptContains() returns !mSignatures.empty()
- [x] TransactionPublicKeyHashMessage: signatures() getter implemented
- [x] TransactionPublicKeyHashMessage: Two constructors (with vector, with single signature)
- [x] TransactionPublicKeyHashMessage: Serialization/deserialization works
- [x] Backward compatibility maintained for old transactions
- [x] Code compiles without errors

# Implementation Plan

## Step 1: Update FinalAmountsConfigurationMessage Header
**File**: `src/core/network/messages/payments/FinalAmountsConfigurationMessage.h`

```cpp
class FinalAmountsConfigurationMessage : public RequestMessageWithReservations
{
public:
    typedef shared_ptr<FinalAmountsConfigurationMessage> Shared;

public:
    // Constructor without receipts
    FinalAmountsConfigurationMessage(
        const SerializedEquivalent equivalent,
        vector<BaseAddress::Shared> senderAddresses,
        const TransactionUUID &transactionUUID,
        const vector<PathReservation> &finalAmountsConfig,
        const map<PaymentNodeID, Contractor::Shared> &paymentParticipants,
        const BlockNumber maximalClaimingBlockNumber);

    // NEW: Constructor with multiple receipts (for new transactions)
    FinalAmountsConfigurationMessage(
        const SerializedEquivalent equivalent,
        vector<BaseAddress::Shared> senderAddresses,
        const TransactionUUID &transactionUUID,
        const vector<PathReservation> &finalAmountsConfig,
        const map<PaymentNodeID, Contractor::Shared> &paymentParticipants,
        const BlockNumber maximalClaimingBlockNumber,
        const vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> &signatures);

    // DEPRECATED: Constructor with single receipt (for old transactions backward compatibility)
    FinalAmountsConfigurationMessage(
        const SerializedEquivalent equivalent,
        vector<BaseAddress::Shared> senderAddresses,
        const TransactionUUID &transactionUUID,
        const vector<pair<PathID, ConstSharedTrustLineAmount>> &finalAmountsConfig,
        const map<PaymentNodeID, Contractor::Shared> &paymentParticipants,
        const BlockNumber maximalClaimingBlockNumber,
        const sphincs::Signature::Shared signature);

    FinalAmountsConfigurationMessage(BytesShared buffer);

    const MessageType typeID() const override;
    const map<PaymentNodeID, Contractor::Shared> &paymentParticipants() const;
    const BlockNumber maximalClaimingBlockNumber() const;

    bool isReceiptContains() const;  // Returns !mSignatures.empty()
    const vector<pair<SerializedEquivalent, sphincs::Signature::Shared>>& signatures() const;  // NEW

    pair<BytesShared, size_t> serializeToBytes() const override;

private:
    map<PaymentNodeID, Contractor::Shared> mPaymentParticipants;
    BlockNumber mMaximalClaimingBlockNumber;
    vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> mSignatures;  // NEW
};
```

## Step 2: Implement FinalAmountsConfigurationMessage Constructors
**File**: `src/core/network/messages/payments/FinalAmountsConfigurationMessage.cpp`

```cpp
// Constructor without receipts
FinalAmountsConfigurationMessage::FinalAmountsConfigurationMessage(
    const SerializedEquivalent equivalent,
    vector<BaseAddress::Shared> senderAddresses,
    const TransactionUUID &transactionUUID,
    const vector<PathReservation> &finalAmountsConfig,
    const map<PaymentNodeID, Contractor::Shared> &paymentParticipants,
    const BlockNumber maximalClaimingBlockNumber)
    :
    RequestMessageWithReservations(equivalent, senderAddresses, transactionUUID, finalAmountsConfig),
    mPaymentParticipants(paymentParticipants),
    mMaximalClaimingBlockNumber(maximalClaimingBlockNumber)
    // mSignatures empty
{}

// NEW: Constructor with multiple receipts
FinalAmountsConfigurationMessage::FinalAmountsConfigurationMessage(
    const SerializedEquivalent equivalent,
    vector<BaseAddress::Shared> senderAddresses,
    const TransactionUUID &transactionUUID,
    const vector<PathReservation> &finalAmountsConfig,
    const map<PaymentNodeID, Contractor::Shared> &paymentParticipants,
    const BlockNumber maximalClaimingBlockNumber,
    const vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> &signatures)
    :
    RequestMessageWithReservations(equivalent, senderAddresses, transactionUUID, finalAmountsConfig),
    mPaymentParticipants(paymentParticipants),
    mMaximalClaimingBlockNumber(maximalClaimingBlockNumber),
    mSignatures(signatures)
{}

// DEPRECATED: Constructor with single receipt (backward compatibility)
FinalAmountsConfigurationMessage::FinalAmountsConfigurationMessage(
    const SerializedEquivalent equivalent,
    vector<BaseAddress::Shared> senderAddresses,
    const TransactionUUID &transactionUUID,
    const vector<pair<PathID, ConstSharedTrustLineAmount>> &finalAmountsConfig,
    const map<PaymentNodeID, Contractor::Shared> &paymentParticipants,
    const BlockNumber maximalClaimingBlockNumber,
    const sphincs::Signature::Shared signature)
    :
    RequestMessageWithReservations(equivalent, senderAddresses, transactionUUID, finalAmountsConfig),
    mPaymentParticipants(paymentParticipants),
    mMaximalClaimingBlockNumber(maximalClaimingBlockNumber)
{
    // Create single-element vector with mEquivalent and signature
    if (signature) {
        mSignatures.emplace_back(equivalent, signature);
    }
    // If signature is null, mSignatures remains empty
}

bool FinalAmountsConfigurationMessage::isReceiptContains() const
{
    return !mSignatures.empty();
}

const vector<pair<SerializedEquivalent, sphincs::Signature::Shared>>&
FinalAmountsConfigurationMessage::signatures() const
{
    return mSignatures;
}
```

## Step 3: Implement FinalAmountsConfigurationMessage Serialization
**File**: `src/core/network/messages/payments/FinalAmountsConfigurationMessage.cpp`

```cpp
pair<BytesShared, size_t> FinalAmountsConfigurationMessage::serializeToBytes() const
{
    // ... existing serialization (base class, participants, blockNumber)

    // Serialize signatures count
    size_t signaturesCount = mSignatures.size();
    memcpy(buffer + bytesBufferOffset, &signaturesCount, sizeof(signaturesCount));
    bytesBufferOffset += sizeof(signaturesCount);

    // Serialize each (equivalent, signature) pair
    for (const auto& [equivalent, signature] : mSignatures) {
        // Serialize equivalent
        memcpy(buffer + bytesBufferOffset, &equivalent, sizeof(SerializedEquivalent));
        bytesBufferOffset += sizeof(SerializedEquivalent);

        // Serialize signature
        auto signatureBytes = signature->serialize();
        memcpy(buffer + bytesBufferOffset, &signatureBytes.size(), sizeof(size_t));
        bytesBufferOffset += sizeof(size_t);
        memcpy(buffer + bytesBufferOffset, signatureBytes.data(), signatureBytes.size());
        bytesBufferOffset += signatureBytes.size();
    }

    return make_pair(buffer, bytesBufferOffset);
}
```

## Step 4: Implement FinalAmountsConfigurationMessage Deserialization
**File**: `src/core/network/messages/payments/FinalAmountsConfigurationMessage.cpp`

```cpp
FinalAmountsConfigurationMessage::FinalAmountsConfigurationMessage(BytesShared buffer)
    :
    RequestMessageWithReservations(buffer)
{
    size_t bytesBufferOffset = RequestMessageWithReservations::kOffsetToInheritedBytes();

    // ... existing deserialization (participants, blockNumber)

    // Deserialize signatures count
    size_t signaturesCount;
    memcpy(&signaturesCount, buffer.get() + bytesBufferOffset, sizeof(signaturesCount));
    bytesBufferOffset += sizeof(signaturesCount);

    mSignatures.reserve(signaturesCount);

    // Deserialize each (equivalent, signature) pair
    for (size_t i = 0; i < signaturesCount; ++i) {
        // Deserialize equivalent
        SerializedEquivalent equivalent;
        memcpy(&equivalent, buffer.get() + bytesBufferOffset, sizeof(SerializedEquivalent));
        bytesBufferOffset += sizeof(SerializedEquivalent);

        // Deserialize signature
        size_t signatureSize;
        memcpy(&signatureSize, buffer.get() + bytesBufferOffset, sizeof(size_t));
        bytesBufferOffset += sizeof(size_t);

        auto signature = make_shared<sphincs::Signature>(
            buffer.get() + bytesBufferOffset,
            signatureSize);
        bytesBufferOffset += signatureSize;

        mSignatures.emplace_back(equivalent, signature);
    }
}
```

## Step 5: Update TransactionPublicKeyHashMessage (Same Pattern)
**File**: `src/core/network/messages/payments/TransactionPublicKeyHashMessage.h`

Apply same changes as FinalAmountsConfigurationMessage:
- Remove mIsReceiptContains, mSignature
- Add mSignatures vector
- Update isReceiptContains()
- Add signatures() getter
- Three constructors: no receipt, with vector, with single signature (deprecated)

**File**: `src/core/network/messages/payments/TransactionPublicKeyHashMessage.cpp`

Implement same pattern for constructors, serialization, deserialization.

Key difference: TransactionPublicKeyHashMessage has paymentNodeID field, so constructors include it.

```cpp
// Constructor without receipts
TransactionPublicKeyHashMessage::TransactionPublicKeyHashMessage(
    const SerializedEquivalent equivalent,
    vector<BaseAddress::Shared> &senderAddresses,
    const TransactionUUID &transactionUUID,
    const PaymentNodeID paymentNodeID,
    const sphincs::KeyHash::Shared transactionPublicKeyHash)
    :
    TransactionMessage(equivalent, senderAddresses, transactionUUID),
    mPaymentNodeID(paymentNodeID),
    mTransactionPublicKeyHash(transactionPublicKeyHash)
    // mSignatures empty
{}

// NEW: Constructor with multiple receipts
TransactionPublicKeyHashMessage::TransactionPublicKeyHashMessage(
    const SerializedEquivalent equivalent,
    vector<BaseAddress::Shared> &senderAddresses,
    const TransactionUUID &transactionUUID,
    const PaymentNodeID paymentNodeID,
    const sphincs::KeyHash::Shared transactionPublicKeyHash,
    const vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> &signatures)
    :
    TransactionMessage(equivalent, senderAddresses, transactionUUID),
    mPaymentNodeID(paymentNodeID),
    mTransactionPublicKeyHash(transactionPublicKeyHash),
    mSignatures(signatures)
{}

// DEPRECATED: Constructor with single receipt (backward compatibility)
TransactionPublicKeyHashMessage::TransactionPublicKeyHashMessage(
    const SerializedEquivalent equivalent,
    vector<BaseAddress::Shared> &senderAddresses,
    const TransactionUUID &transactionUUID,
    const PaymentNodeID paymentNodeID,
    const sphincs::KeyHash::Shared transactionPublicKeyHash,
    const sphincs::Signature::Shared signature)
    :
    TransactionMessage(equivalent, senderAddresses, transactionUUID),
    mPaymentNodeID(paymentNodeID),
    mTransactionPublicKeyHash(transactionPublicKeyHash)
{
    // Create single-element vector with mEquivalent and signature
    if (signature) {
        mSignatures.emplace_back(equivalent, signature);
    }
}
```

## Step 6: Update Transaction Classes Usage
Find all places where these messages are created and update to use appropriate constructor:

**Old transactions**: Use deprecated constructor with single signature
**New transactions**: Use new constructor with signatures vector

# Test Plan

## Unit Tests
As this is a Moderate task, focused testing on core functionality:

### Test Category: FinalAmountsConfigurationMessage Structure
1. **testFinalAmountsConfigurationMessageFieldsRemoved**: Verify mIsReceiptContains and mSignature removed
2. **testFinalAmountsConfigurationMessageSignaturesVectorAdded**: Verify mSignatures vector exists

### Test Category: FinalAmountsConfigurationMessage Constructors
3. **testFinalAmountsConfigurationMessageNoReceipts**: Constructor without receipts, mSignatures empty
4. **testFinalAmountsConfigurationMessageWithVector**: Constructor with signatures vector
5. **testFinalAmountsConfigurationMessageDeprecatedConstructor**: Deprecated constructor creates single-element vector

### Test Category: FinalAmountsConfigurationMessage isReceiptContains()
6. **testIsReceiptContainsEmptyVector**: Empty mSignatures → false
7. **testIsReceiptContainsSingleElement**: Single signature → true
8. **testIsReceiptContainsMultipleElements**: Multiple signatures → true

### Test Category: FinalAmountsConfigurationMessage Serialization
9. **testSerializationEmptySignatures**: Serialize with empty signatures vector
10. **testSerializationSingleSignature**: Serialize with one signature
11. **testSerializationMultipleSignatures**: Serialize with 3 signatures in different equivalents

### Test Category: FinalAmountsConfigurationMessage Deserialization
12. **testDeserializationEmptySignatures**: Deserialize message with 0 signatures
13. **testDeserializationSingleSignature**: Deserialize message with 1 signature
14. **testDeserializationMultipleSignatures**: Deserialize message with 3 signatures

### Test Category: FinalAmountsConfigurationMessage Round-Trip
15. **testRoundTripEmptySignatures**: Serialize then deserialize with empty vector
16. **testRoundTripSingleSignature**: Serialize then deserialize with 1 signature
17. **testRoundTripMultipleSignatures**: Serialize then deserialize with multiple signatures, verify order preserved

### Test Category: TransactionPublicKeyHashMessage (Mirror Tests)
18-26. Same tests as FinalAmountsConfigurationMessage (9 tests)

### Test Category: Backward Compatibility
27. **testDeprecatedConstructorNullSignature**: Deprecated constructor with null signature creates empty vector
28. **testDeprecatedConstructorValidSignature**: Deprecated constructor with valid signature creates single-element vector
29. **testOldTransactionUsagePattern**: Old transaction creates message using deprecated constructor

### Test Category: New Transaction Usage
30. **testNewTransactionMultipleReceipts**: New transaction creates message with multiple receipts (one per equivalent)

## Success Criteria
- All 30 unit tests pass
- Code compiles without errors or warnings
- Serialization format handles empty and non-empty vectors
- Backward compatibility maintained
- Order of signatures preserved during serialization/deserialization

# Verification and Validation

## Architecture integrity
- Clean removal of single receipt fields
- Vector-based approach supports multiple receipts naturally
- isReceiptContains() provides consistent boolean interface
- signatures() getter enables iteration over all receipts

## Security
- Each signature tied to specific equivalent
- Signature validation remains unchanged (per existing protocol)
- No signature forgery risks introduced

## Performance
- Serialization: O(n) where n=number of signatures (typically 1-5)
- Deserialization: O(n)
- Memory: minimal overhead (vector of pairs)
- Acceptable for typical use cases

## Scalability
- Supports up to 5 signatures per message (matching 5 equivalent limit)
- Vector scales linearly with signature count

## Reliability
- Empty vector handling prevents null pointer issues
- Order preservation ensures correct equivalent-signature mapping
- Clear serialization format with count field

## Maintainability
- Consistent pattern across both message classes
- Deprecated constructors clearly marked for future removal
- Vector interface familiar to C++ developers

## Cost
- No additional infrastructure required
- Minimal memory overhead per message

## Compliance
- Follows repository policy for task-driven development
- Maintains message protocol extension pattern
- No prohibited operations or scope creep

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not modify signature validation logic (out of scope)
- Ensure backward compatibility constructors work correctly for old transactions
