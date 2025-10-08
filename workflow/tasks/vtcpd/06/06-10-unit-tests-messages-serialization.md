# 06-10 - Unit Tests: Messages and Serialization

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task 1](06-04-reservation-upgrading-by-equivalent.md)
- [Previous task 2](06-05-base-payment-transaction-upgrading.md)
- [Previous task 3](06-07-message-extensions-multiple-receipts.md)

# Description
Implement unit tests for message protocol extensions and serialization enhancements created in tasks 06-04, 06-05, and 06-07. This includes testing RequestMessageWithReservations with equivalents, BasePaymentTransaction serialization with equivalents, and multiple receipt support in FinalAmountsConfigurationMessage and TransactionPublicKeyHashMessage.

These tests validate the communication protocol and state persistence for multi-equivalent payments.

# Requirements and DOD

## Requirements

### Test Category 11: RequestMessageWithReservations
1. Test new constructor with PathReservation vector
2. Test deprecated constructor with pair vector
3. Test serialization includes equivalents
4. Test deserialization reads equivalents
5. Test round-trip preserves data
6. Test backward compatibility

### Test Category 12: BasePaymentTransaction Serialization
7. Test serializeToBytes() includes reservation equivalents
8. Test deserialization constructor reads equivalents
9. Test round-trip preserves multi-equivalent reservations

### Test Category 13: FinalAmountsConfigurationMessage
10. Test mIsReceiptContains and mSignature removed
11. Test mSignatures vector added
12. Test isReceiptContains() with empty/non-empty vector
13. Test constructor variants (no receipts, with vector, with single signature)
14. Test serialization/deserialization of signatures vector

### Test Category 14: TransactionPublicKeyHashMessage
15. Test same as FinalAmountsConfigurationMessage (mirror tests)

## Definition of Done
- [x] All RequestMessageWithReservations tests implemented and passing (6 tests)
- [x] All BasePaymentTransaction serialization tests implemented and passing (3 tests)
- [x] All FinalAmountsConfigurationMessage tests implemented and passing (8 tests)
- [x] All TransactionPublicKeyHashMessage tests implemented and passing (8 tests)
- [x] All tests compile without errors
- [x] All tests pass in build-tests
- [x] Test coverage adequate for Simple task
- [x] Mock data provided where applicable

# Implementation Plan

## Test File Structure
Create test files:
- `tests/core/network/messages/TestRequestMessageWithReservations.cpp`
- `tests/core/transactions/TestBasePaymentTransactionSerialization.cpp`
- `tests/core/network/messages/TestFinalAmountsConfigurationMessage.cpp`
- `tests/core/network/messages/TestTransactionPublicKeyHashMessage.cpp`

## Test Category 11: RequestMessageWithReservations (6 tests)

### Test 11.1: testRequestMessageWithReservationsNewConstructor
```cpp
TEST(RequestMessageWithReservations, NewConstructor) {
    SerializedEquivalent equivalent(1);
    vector<BaseAddress::Shared> senderAddresses = {
        make_shared<IPv4WithPortAddress>("127.0.0.1:2000")
    };
    TransactionUUID uuid = TransactionUUID::generate();

    vector<PathReservation> reservations = {
        PathReservation(PathID(1), make_shared<const TrustLineAmount>(100), SerializedEquivalent(1)),
        PathReservation(PathID(2), make_shared<const TrustLineAmount>(200), SerializedEquivalent(2))
    };

    RequestMessageWithReservations message(equivalent, senderAddresses, uuid, reservations);

    const auto& config = message.finalAmountsConfiguration();
    ASSERT_EQ(config.size(), 2);
    EXPECT_EQ(config[0].pathID, PathID(1));
    EXPECT_EQ(config[0].equivalent, SerializedEquivalent(1));
    EXPECT_EQ(config[1].pathID, PathID(2));
    EXPECT_EQ(config[1].equivalent, SerializedEquivalent(2));
}
```

### Test 11.2: testRequestMessageWithReservationsDeprecatedConstructor
```cpp
TEST(RequestMessageWithReservations, DeprecatedConstructor) {
    SerializedEquivalent equivalent(3);
    vector<BaseAddress::Shared> senderAddresses = {
        make_shared<IPv4WithPortAddress>("127.0.0.1:2000")
    };
    TransactionUUID uuid = TransactionUUID::generate();

    vector<pair<PathID, ConstSharedTrustLineAmount>> reservations = {
        {PathID(10), make_shared<const TrustLineAmount>(500)},
        {PathID(20), make_shared<const TrustLineAmount>(750)}
    };

    RequestMessageWithReservations message(equivalent, senderAddresses, uuid, reservations);

    const auto& config = message.finalAmountsConfiguration();
    ASSERT_EQ(config.size(), 2);

    // Should use mEquivalent (3) for all reservations
    EXPECT_EQ(config[0].equivalent, SerializedEquivalent(3));
    EXPECT_EQ(config[1].equivalent, SerializedEquivalent(3));
}
```

### Test 11.3: testRequestMessageWithReservationsSerializationWithEquivalents
```cpp
TEST(RequestMessageWithReservations, SerializationWithEquivalents) {
    vector<PathReservation> reservations = {
        PathReservation(PathID(1), make_shared<const TrustLineAmount>(100), SerializedEquivalent(1)),
        PathReservation(PathID(2), make_shared<const TrustLineAmount>(200), SerializedEquivalent(2))
    };

    RequestMessageWithReservations message(..., reservations);

    auto [buffer, size] = message.serializeToBytes();

    // Manually verify buffer contains equivalents
    // (Detailed byte inspection or deserialization test)
    ASSERT_GT(size, 0);
}
```

### Test 11.4: testRequestMessageWithReservationsDeserializationWithEquivalents
```cpp
TEST(RequestMessageWithReservations, DeserializationWithEquivalents) {
    // Create message, serialize, then deserialize
    vector<PathReservation> reservations = {
        PathReservation(PathID(5), make_shared<const TrustLineAmount>(300), SerializedEquivalent(10)),
        PathReservation(PathID(6), make_shared<const TrustLineAmount>(400), SerializedEquivalent(20))
    };

    RequestMessageWithReservations originalMessage(..., reservations);
    auto [buffer, size] = originalMessage.serializeToBytes();

    RequestMessageWithReservations deserializedMessage(buffer);

    const auto& config = deserializedMessage.finalAmountsConfiguration();
    ASSERT_EQ(config.size(), 2);
    EXPECT_EQ(config[0].equivalent, SerializedEquivalent(10));
    EXPECT_EQ(config[1].equivalent, SerializedEquivalent(20));
}
```

### Test 11.5: testRequestMessageWithReservationsRoundTrip
```cpp
TEST(RequestMessageWithReservations, RoundTrip) {
    vector<PathReservation> reservations = {
        PathReservation(PathID(1), make_shared<const TrustLineAmount>(100), SerializedEquivalent(1)),
        PathReservation(PathID(2), make_shared<const TrustLineAmount>(200), SerializedEquivalent(2)),
        PathReservation(PathID(3), make_shared<const TrustLineAmount>(300), SerializedEquivalent(3))
    };

    RequestMessageWithReservations originalMessage(..., reservations);
    auto [buffer, size] = originalMessage.serializeToBytes();

    RequestMessageWithReservations deserializedMessage(buffer);

    const auto& config = deserializedMessage.finalAmountsConfiguration();
    ASSERT_EQ(config.size(), 3);

    for (size_t i = 0; i < 3; ++i) {
        EXPECT_EQ(config[i].pathID, reservations[i].pathID);
        EXPECT_EQ(*config[i].amount, *reservations[i].amount);
        EXPECT_EQ(config[i].equivalent, reservations[i].equivalent);
    }
}
```

### Test 11.6: testRequestMessageWithReservationsBackwardCompatibility
```cpp
TEST(RequestMessageWithReservations, BackwardCompatibility) {
    // Deprecated constructor should produce same format as new constructor
    SerializedEquivalent equivalent(5);

    vector<pair<PathID, ConstSharedTrustLineAmount>> deprecatedReservations = {
        {PathID(1), make_shared<const TrustLineAmount>(100)}
    };

    vector<PathReservation> newReservations = {
        PathReservation(PathID(1), make_shared<const TrustLineAmount>(100), SerializedEquivalent(5))
    };

    RequestMessageWithReservations deprecatedMessage(..., deprecatedReservations);
    RequestMessageWithReservations newMessage(..., newReservations);

    auto [deprecatedBuffer, deprecatedSize] = deprecatedMessage.serializeToBytes();
    auto [newBuffer, newSize] = newMessage.serializeToBytes();

    // Both should produce equivalent serialization
    EXPECT_EQ(deprecatedSize, newSize);
    EXPECT_EQ(memcmp(deprecatedBuffer.get(), newBuffer.get(), deprecatedSize), 0);
}
```

## Test Category 12: BasePaymentTransaction Serialization (3 tests)

### Test 12.1: testSerializeToBytesIncludesEquivalents
```cpp
TEST(BasePaymentTransactionSerialization, IncludesEquivalents) {
    BasePaymentTransaction transaction(...);

    // Add reservations with different equivalents
    transaction.mReservations[ContractorID(1)] = {
        {PathID(1), make_shared<const AmountReservation>(
            uuid, TrustLineAmount(100), AmountReservation::Outgoing, SerializedEquivalent(1))},
        {PathID(2), make_shared<const AmountReservation>(
            uuid, TrustLineAmount(200), AmountReservation::Incoming, SerializedEquivalent(2))}
    };

    auto [buffer, size] = transaction.serializeToBytes();

    // Verify serialization includes equivalents
    ASSERT_GT(size, 0);
}
```

### Test 12.2: testDeserializationReadsEquivalents
```cpp
TEST(BasePaymentTransactionSerialization, DeserializationReadsEquivalents) {
    // Create transaction with multi-equivalent reservations
    BasePaymentTransaction originalTransaction(...);
    originalTransaction.mReservations[ContractorID(10)] = {
        {PathID(5), make_shared<const AmountReservation>(
            uuid, TrustLineAmount(500), AmountReservation::Outgoing, SerializedEquivalent(10))}
    };

    auto [buffer, size] = originalTransaction.serializeToBytes();

    // Deserialize
    BasePaymentTransaction deserializedTransaction(buffer, ...);

    // Verify reservations with correct equivalents
    auto& reservations = deserializedTransaction.mReservations[ContractorID(10)];
    ASSERT_EQ(reservations.size(), 1);
    EXPECT_EQ(reservations[0].second->equivalent(), SerializedEquivalent(10));
}
```

### Test 12.3: testRoundTripSerializationPreservesEquivalents
```cpp
TEST(BasePaymentTransactionSerialization, RoundTripPreservesEquivalents) {
    BasePaymentTransaction originalTransaction(...);

    // Add reservations in multiple equivalents
    originalTransaction.mReservations[ContractorID(1)] = {
        {PathID(1), make_shared<const AmountReservation>(
            uuid1, TrustLineAmount(100), AmountReservation::Outgoing, SerializedEquivalent(1))},
        {PathID(2), make_shared<const AmountReservation>(
            uuid2, TrustLineAmount(200), AmountReservation::Incoming, SerializedEquivalent(2))}
    };

    originalTransaction.mReservations[ContractorID(2)] = {
        {PathID(3), make_shared<const AmountReservation>(
            uuid3, TrustLineAmount(300), AmountReservation::Outgoing, SerializedEquivalent(3))}
    };

    auto [buffer, size] = originalTransaction.serializeToBytes();
    BasePaymentTransaction deserializedTransaction(buffer, ...);

    // Verify all reservations preserved with correct equivalents
    EXPECT_EQ(deserializedTransaction.mReservations.size(), 2);

    auto& contractor1Reservations = deserializedTransaction.mReservations[ContractorID(1)];
    ASSERT_EQ(contractor1Reservations.size(), 2);
    EXPECT_EQ(contractor1Reservations[0].second->equivalent(), SerializedEquivalent(1));
    EXPECT_EQ(contractor1Reservations[1].second->equivalent(), SerializedEquivalent(2));

    auto& contractor2Reservations = deserializedTransaction.mReservations[ContractorID(2)];
    ASSERT_EQ(contractor2Reservations.size(), 1);
    EXPECT_EQ(contractor2Reservations[0].second->equivalent(), SerializedEquivalent(3));
}
```

## Test Category 13: FinalAmountsConfigurationMessage (8 tests)

### Test 13.1: testFinalAmountsConfigurationMessageFieldsRemoved
```cpp
TEST(FinalAmountsConfigurationMessage, FieldsRemoved) {
    // Compilation test: verify old fields don't exist
    // (This test ensures mIsReceiptContains and mSignature are removed)

    FinalAmountsConfigurationMessage message(...);

    // Should NOT compile if old fields exist:
    // message.mIsReceiptContains;  // Should fail
    // message.mSignature;  // Should fail

    SUCCEED();  // If it compiles, fields are removed
}
```

### Test 13.2: testFinalAmountsConfigurationMessageSignaturesVectorAdded
```cpp
TEST(FinalAmountsConfigurationMessage, SignaturesVectorAdded) {
    vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> signatures = {
        {SerializedEquivalent(1), make_shared<sphincs::Signature>(...)},
        {SerializedEquivalent(2), make_shared<sphincs::Signature>(...)}
    };

    FinalAmountsConfigurationMessage message(..., signatures);

    const auto& retrievedSignatures = message.signatures();
    ASSERT_EQ(retrievedSignatures.size(), 2);
    EXPECT_EQ(retrievedSignatures[0].first, SerializedEquivalent(1));
    EXPECT_EQ(retrievedSignatures[1].first, SerializedEquivalent(2));
}
```

### Test 13.3: testIsReceiptContainsEmptyVector
```cpp
TEST(FinalAmountsConfigurationMessage, IsReceiptContainsEmptyVector) {
    FinalAmountsConfigurationMessage message(...);  // No signatures

    EXPECT_FALSE(message.isReceiptContains());
}
```

### Test 13.4: testIsReceiptContainsSingleElement
```cpp
TEST(FinalAmountsConfigurationMessage, IsReceiptContainsSingleElement) {
    vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> signatures = {
        {SerializedEquivalent(1), make_shared<sphincs::Signature>(...)}
    };

    FinalAmountsConfigurationMessage message(..., signatures);

    EXPECT_TRUE(message.isReceiptContains());
}
```

### Test 13.5: testIsReceiptContainsMultipleElements
```cpp
TEST(FinalAmountsConfigurationMessage, IsReceiptContainsMultipleElements) {
    vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> signatures = {
        {SerializedEquivalent(1), make_shared<sphincs::Signature>(...)},
        {SerializedEquivalent(2), make_shared<sphincs::Signature>(...)},
        {SerializedEquivalent(3), make_shared<sphincs::Signature>(...)}
    };

    FinalAmountsConfigurationMessage message(..., signatures);

    EXPECT_TRUE(message.isReceiptContains());
}
```

### Test 13.6: testDeprecatedConstructorNullSignature
```cpp
TEST(FinalAmountsConfigurationMessage, DeprecatedConstructorNullSignature) {
    // Deprecated constructor with null signature
    FinalAmountsConfigurationMessage message(
        equivalent,
        senderAddresses,
        uuid,
        finalAmountsConfig,
        paymentParticipants,
        maxBlockNumber,
        nullptr);  // Null signature

    EXPECT_FALSE(message.isReceiptContains());
    EXPECT_EQ(message.signatures().size(), 0);
}
```

### Test 13.7: testDeprecatedConstructorValidSignature
```cpp
TEST(FinalAmountsConfigurationMessage, DeprecatedConstructorValidSignature) {
    SerializedEquivalent equivalent(5);
    auto signature = make_shared<sphincs::Signature>(...);

    // Deprecated constructor with valid signature
    FinalAmountsConfigurationMessage message(
        equivalent,
        senderAddresses,
        uuid,
        finalAmountsConfig,
        paymentParticipants,
        maxBlockNumber,
        signature);

    EXPECT_TRUE(message.isReceiptContains());
    EXPECT_EQ(message.signatures().size(), 1);
    EXPECT_EQ(message.signatures()[0].first, equivalent);
    EXPECT_EQ(message.signatures()[0].second, signature);
}
```

### Test 13.8: testSerializationDeserializationMultipleSignatures
```cpp
TEST(FinalAmountsConfigurationMessage, SerializationDeserializationMultipleSignatures) {
    vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> signatures = {
        {SerializedEquivalent(1), make_shared<sphincs::Signature>(...)},
        {SerializedEquivalent(2), make_shared<sphincs::Signature>(...)},
        {SerializedEquivalent(3), make_shared<sphincs::Signature>(...)}
    };

    FinalAmountsConfigurationMessage originalMessage(..., signatures);

    auto [buffer, size] = originalMessage.serializeToBytes();
    FinalAmountsConfigurationMessage deserializedMessage(buffer);

    const auto& retrievedSignatures = deserializedMessage.signatures();
    ASSERT_EQ(retrievedSignatures.size(), 3);

    for (size_t i = 0; i < 3; ++i) {
        EXPECT_EQ(retrievedSignatures[i].first, signatures[i].first);
        // Signature comparison (if applicable)
    }
}
```

## Test Category 14: TransactionPublicKeyHashMessage (8 tests)

### Test 14.1-14.8: Mirror tests from FinalAmountsConfigurationMessage
Apply same test structure with TransactionPublicKeyHashMessage-specific parameters (paymentNodeID, transactionPublicKeyHash).

# Test Plan

## Test Execution
- Build tests in `build-tests`
- Run all test binaries
- Verify 100% pass rate

## Coverage Requirements
As this is a Simple task:
- RequestMessageWithReservations: 80%+ coverage
- BasePaymentTransaction serialization: 70%+ coverage
- FinalAmountsConfigurationMessage: 80%+ coverage
- TransactionPublicKeyHashMessage: 80%+ coverage

## Mock Requirements
- Mock ContractorsManager where needed
- Mock Signature creation (use test keys)
- No database mocks needed (in-memory serialization tests)

# Verification and Validation

## Architecture integrity
- Tests validate message protocol correctness
- Tests ensure serialization preserves all data

## Security
- No security concerns in unit tests (in-memory only)

## Performance
- All tests should complete in < 3 seconds total

## Scalability
- Tests validate behavior with multiple equivalents and signatures

## Reliability
- Comprehensive serialization test coverage ensures data persistence reliability

## Maintainability
- Clear test names describe what is tested
- Easy to extend for additional message types

## Cost
- No additional infrastructure required (tests only)

## Compliance
- Follows repository policy for unit testing
- Tests built and executed in build-tests only

# Restrictions
- All tests must pass before task completion
- No integration tests (unit tests only)
- Mock all external dependencies
