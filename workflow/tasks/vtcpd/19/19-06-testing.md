# 19-06 - Testing for PRD-19 Implementation

# Links
- [PRD-19: Debt Receipt Signature Structure Update](../../prd/vtcpd/19-debt-receipt-signature-structure-update.md)
- [Previous Task 19-01: Contractor Payment Public Key Storage](19-01-contractor-payment-key-storage.md)
- [Previous Task 19-02: Channel Messages Payment Public Key Extension](19-02-channel-messages-payment-key.md)
- [Previous Task 19-03: Channel Transactions Payment Key Exchange](19-03-channel-transactions-key-exchange.md)
- [Previous Task 19-04: Debt Receipt Signature Structure Update](19-04-debt-receipt-signature-structure.md)
- [Previous Task 19-05: Payment Verification and Related Transactions Update](19-05-payment-verification-and-related-transactions.md)

# Description
This task implements unit tests and PostgreSQL integration tests for components modified or created in PRD-19. This includes:

1. **Unit Tests for InitChannelMessage** - New test file for message serialization/deserialization
2. **Unit Tests for ConfirmChannelMessage** - New test file for message serialization/deserialization
3. **Unit Tests for Contractor** - Validate `paymentPublicKey` getter/setter behavior
4. **Update ContractorsHandlerSQLite Tests** - Add tests for payment public key persistence
5. **Update ContractorsHandlerPostgreSQL Tests** - Add tests for payment public key persistence
6. **PostgreSQL Integration Tests for ContractorsHandler** - Verify payment public key persistence

**Context from PRD - Testing Strategy:**
- Unit tests for message serialization with payment key
- Unit tests for Contractor payment public key handling
- Unit tests for ContractorsHandler payment key persistence
- Integration tests for ContractorsHandler payment key persistence

# Requirements and DOD

## Functional Requirements

### 1. Unit Tests for InitChannelMessage
**File:** `tests/unit/messages/InitChannelMessageTest.cpp` (new file)

Test cases:
- Serialization with valid payment public key
- Deserialization and key extraction
- Round-trip serialization (serialize → deserialize → compare)
- Edge cases: message with various key sizes
- Verify serialization format matches `ParticipantVoteMessage` pattern

### 2. Unit Tests for ConfirmChannelMessage
**File:** `tests/unit/messages/ConfirmChannelMessageTest.cpp` (new file)

Test cases:
- Serialization with valid payment public key
- Deserialization and key extraction
- Round-trip serialization (serialize → deserialize → compare)
- Edge cases: message with various key sizes
- Verify serialization format matches `ParticipantVoteMessage` pattern

### 3. Update ContractorsHandlerSQLite Tests
**File:** `tests/unit/sqlite/ContractorsHandlerSQLiteTest.cpp`

New test cases:
- `SaveContractor_WithPaymentPublicKey_SavesSuccessfully`
- `SaveContractorFull_WithPaymentPublicKey_SavesSuccessfully`
- `AllContractors_WithPaymentPublicKey_RetrievesCorrectly`
- `UpdatePaymentPublicKey_ExistingContractor_UpdatesSuccessfully`
- `UpdatePaymentPublicKey_NonExistentContractor_ThrowsError`
- `SaveContractor_WithNullPaymentPublicKey_SavesSuccessfully`
- `AllContractors_WithNullPaymentPublicKey_RetrievesNull`

### 4. Update ContractorsHandlerPostgreSQL Tests
**File:** `tests/storage/integration/postgresql/ContractorsHandlerPostgreSQLIntegrationTest.cpp`

New test cases (mirror SQLite tests):
- `saveContractor_WithPaymentPublicKey_SavesSuccessfully`
- `saveContractorFull_WithPaymentPublicKey_SavesSuccessfully`
- `allContractors_WithPaymentPublicKey_RetrievesCorrectly`
- `updatePaymentPublicKey_ExistingContractor_UpdatesSuccessfully`
- `updatePaymentPublicKey_NonExistentContractor_ThrowsValueError`
- `saveContractor_WithNullPaymentPublicKey_SavesSuccessfully`
- `rawDatabaseValidation_PaymentPublicKey_StoredAsBytea`

### 5. Unit Tests for Contractor
**File:** `tests/unit/contractors/ContractorTest.cpp` (new file)

Test cases:
- `PaymentPublicKey_DefaultsToNull`
- `PaymentPublicKey_SetGet_RoundTrip`
- `PaymentPublicKey_SetToNull_ClearsValue`

## Definition of Done
- [ ] `InitChannelMessageTest.cpp` created with all specified test cases
- [ ] `ConfirmChannelMessageTest.cpp` created with all specified test cases
- [ ] `ContractorTest.cpp` created with payment public key tests
- [ ] `ContractorsHandlerSQLiteTest.cpp` updated with payment public key tests
- [ ] `ContractorsHandlerPostgreSQLIntegrationTest.cpp` updated with payment public key tests
- [ ] All new unit tests pass
- [ ] All existing tests pass (no regression)
- [ ] Test coverage meets project standards

# Implementation Plan

## Step 1: Create Test Infrastructure
**Actions:**
1. Create directory `tests/unit/messages/` if it doesn't exist
2. Create directory `tests/unit/contractors/` if it doesn't exist
3. Add new test files to CMakeLists.txt
4. Set up test fixtures for SPHINCS+ key generation

## Step 2: Implement InitChannelMessage Tests
**File:** `tests/unit/messages/InitChannelMessageTest.cpp`

```cpp
#include <gtest/gtest.h>
#include "src/core/network/messages/trust_line_channels/InitChannelMessage.h"
#include "src/core/crypto/sphincs/sphincs.h"
// ... other includes

class InitChannelMessageTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Generate test key pair
        mTestKeyPair = sphincs::generateKeyPair();
        mTestPublicKey = mTestKeyPair->publicKey();
    }

    sphincs::KeyPair::Shared mTestKeyPair;
    sphincs::PublicKey::Shared mTestPublicKey;
};

TEST_F(InitChannelMessageTest, Serialization_WithValidPaymentKey_SerializesCorrectly) {
    // Create message with payment key
    // Serialize to bytes
    // Verify bytes contain key data
}

TEST_F(InitChannelMessageTest, Deserialization_ValidBytes_ExtractsKeyCorrectly) {
    // Create serialized message bytes
    // Deserialize to message object
    // Verify paymentPublicKey() returns correct key
}

TEST_F(InitChannelMessageTest, RoundTrip_SerializeDeserialize_KeyMatches) {
    // Create message with key
    // Serialize
    // Deserialize
    // Compare original and deserialized keys
}

// ... more tests
```

## Step 3: Implement ConfirmChannelMessage Tests
**File:** `tests/unit/messages/ConfirmChannelMessageTest.cpp`

Apply same pattern as InitChannelMessage tests.

## Step 4: Add Contractor Unit Tests
**File:** `tests/unit/contractors/ContractorTest.cpp`

**Add new test cases:**
```cpp
TEST(ContractorTest, PaymentPublicKey_DefaultsToNull) {
    auto contractor = createTestContractor(1, 0, false);
    EXPECT_EQ(contractor->paymentPublicKey(), nullptr);
}

TEST(ContractorTest, PaymentPublicKey_SetGet_RoundTrip) {
    auto keyPair = sphincs::generateKeyPair();
    auto contractor = createTestContractor(1, 0, false);
    contractor->setPaymentPublicKey(keyPair->publicKey());
    EXPECT_NE(contractor->paymentPublicKey(), nullptr);
    // Compare key bytes if helper exists
}

TEST(ContractorTest, PaymentPublicKey_SetToNull_ClearsValue) {
    auto keyPair = sphincs::generateKeyPair();
    auto contractor = createTestContractor(1, 0, false);
    contractor->setPaymentPublicKey(keyPair->publicKey());
    contractor->setPaymentPublicKey(nullptr);
    EXPECT_EQ(contractor->paymentPublicKey(), nullptr);
}
```

## Step 5: Update ContractorsHandlerSQLite Tests
**File:** `tests/unit/sqlite/ContractorsHandlerSQLiteTest.cpp`

**Add new test cases:**
```cpp
TEST_F(ContractorsHandlerSQLiteTest, SaveContractor_WithPaymentPublicKey_SavesSuccessfully) {
    auto keyPair = sphincs::generateKeyPair();
    auto contractor = createTestContractor(1, 0, false);
    contractor->setPaymentPublicKey(keyPair->publicKey());

    EXPECT_NO_THROW(handler->saveContractor(contractor));

    auto retrieved = handler->allContractors();
    ASSERT_EQ(retrieved.size(), 1);
    EXPECT_NE(retrieved[0]->paymentPublicKey(), nullptr);
    // Compare key bytes
}

TEST_F(ContractorsHandlerSQLiteTest, UpdatePaymentPublicKey_ExistingContractor_UpdatesSuccessfully) {
    // Save contractor without key
    // Update with key
    // Verify key is persisted
}

TEST_F(ContractorsHandlerSQLiteTest, SaveContractor_WithNullPaymentPublicKey_SavesSuccessfully) {
    auto contractor = createTestContractor(1, 0, false);
    // Don't set payment public key

    EXPECT_NO_THROW(handler->saveContractor(contractor));

    auto retrieved = handler->allContractors();
    ASSERT_EQ(retrieved.size(), 1);
    EXPECT_EQ(retrieved[0]->paymentPublicKey(), nullptr);
}
```

## Step 6: Update ContractorsHandlerPostgreSQL Tests
**File:** `tests/storage/integration/postgresql/ContractorsHandlerPostgreSQLIntegrationTest.cpp`

Apply same test patterns as SQLite, adapted for PostgreSQL integration testing patterns used in existing tests.

## Step 7: Update CMakeLists.txt
Ensure new test files are included in build:
```cmake
# In tests/unit/CMakeLists.txt
add_executable(unit_tests
    ...
    messages/InitChannelMessageTest.cpp
    messages/ConfirmChannelMessageTest.cpp
    contractors/ContractorTest.cpp
    ...
)
```

## Step 8: Run All Tests
1. Build test targets
2. Run unit tests
3. Run integration tests
4. Verify all pass

# Test Plan

**Complexity Level:** Moderate

This task IS the testing task, so verification focuses on:

1. **Test Execution**
   - All new unit tests pass
   - All updated tests pass
   - All existing tests pass (no regression)

2. **Coverage Verification**
   - New code from Tasks 19-01 through 19-05 is covered
   - Critical paths are tested
   - Error cases are tested

3. **Test Quality**
   - Tests are independent (no order dependency)
   - Tests are deterministic (no flaky tests)
   - Tests have clear assertions and error messages

# Verification and Validation
_To be completed by Architect after implementation_

## Architecture integrity
- [ ] Tests follow existing project test patterns
- [ ] Test file locations follow project conventions
- [ ] Test fixtures are reusable where appropriate

## Security
- [ ] Tests verify security-relevant behavior (key validation, error handling)
- [ ] No sensitive data in test output

## Performance
- [ ] Tests run in reasonable time
- [ ] No unnecessary resource usage

## Scalability
- N/A for this task

## Reliability
- [ ] Tests are deterministic
- [ ] Tests clean up after themselves
- [ ] No flaky tests

## Maintainability
- [ ] Tests are well-documented
- [ ] Test names clearly describe what's being tested
- [ ] Test code is readable and maintainable

## Cost
- N/A for this task

## Compliance
- N/A for this task

# Restrictions
- Commit changes only after all tests pass
- Do not modify implementation code in this task (only test code)
- Follow existing test patterns in the codebase
- Ensure all new test files are properly added to build system
