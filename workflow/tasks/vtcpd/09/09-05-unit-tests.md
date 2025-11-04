# 09-05 - Unit Tests for Exchange Rate and Commission Change Handling

# Links
- [PRD](../../../prd/vtcpd/09-exchange-rate-commission-change-handling.md)
- [Previous task 1: 09-01](09-01-message-protocol-extensions.md)
- [Previous task 2: 09-02](09-02-intermediate-node-validation.md)
- [Previous task 3: 09-03](09-03-coordinator-condition-change-detection.md)
- [Previous task 4: 09-04](09-04-subsequent-paths-recalculation.md)

# Description
Implement comprehensive unit tests for all functionality related to exchange rate and commission change handling during payment execution. This includes testing message protocol extensions, intermediate node validation logic, coordinator condition change detection and adaptation, subsequent paths recalculation, and regression testing to ensure existing functionality remains intact.

These tests validate the complete flow from condition mismatch detection through path recalculation and manager updates, ensuring robust handling of dynamic network conditions.

# Requirements and DOD

## Requirements

### R1: Message Protocol Tests
- Test CoordinatorReservationRequestMessage serialization/deserialization with all field combinations
- Test CoordinatorReservationResponseMessage serialization/deserialization with all field combinations
- Test RejectedDueConditionsChanged enum value recognition
- Test backward compatibility with old message formats

### R2: Intermediate Node Validation Tests
- Test protocol violation detection (both fields present)
- Test exchange rate validation (matching, mismatching, from context, not found, exists but not expected)
- Test commission validation (matching, mismatching, from context, removed, exists but not expected)
- Test context storage (first path storage, subsequent path usage, not overwritten)
- Test transaction completion logic with context
- Test outgoing/incoming reservation validation (without conditions, with commission, with exchange rate)
- Test commission "charge once per payment" logic

### R3: Coordinator Condition Change Tests
- Test expected condition population in both ask* methods
- Test RejectedDueConditionsChanged detection in both response processing methods
- Test exchange rate change handling (higher, lower, path recalculation)
- Test commission change handling (higher, removed, path recalculation)
- Test manager updates (ExchangeRatesManager, CommissionsManager)
- Test new path creation with correct values
- Test reservation dropping on invalidated path

### R4: Subsequent Paths Recalculation Tests
- Test subsequent path identification
- Test received_amount recalculation for affected paths
- Test flows recalculation for affected paths
- Test paths not containing affected node remain unchanged
- Test error handling during recalculation

### R5: Regression Tests
- Test payments without condition changes work unchanged
- Test single-equivalent payments unaffected
- Test path capacity adjustment (PRD 08) still works with new code

## Definition of Done

1. All message protocol tests pass (13 tests from Task 09-01)

2. All intermediate node validation tests pass (25 tests: Tests 0, 1a, 1-20, 29-31 from PRD)

3. All coordinator condition change tests pass (14 tests: Tests 21-28, 32-34, plus condition population tests)

4. All subsequent paths recalculation tests pass (11 tests: Tests 35-45)

5. All regression tests pass (3 test suites)

6. 100% code coverage for new validation and condition change logic

7. Tests execute in build-tests directory

8. All tests pass in CI environment

9. No memory leaks detected in test execution

10. Test documentation complete (test names, purpose, setup, expectations)

# Implementation Plan

## Step 1: Create Test Environment Helpers

**File**: `tests/unit/transactions/payments/ExchangePaymentTestEnvironment.h`

### 1.1: Create helper class for test setup
```cpp
class ExchangePaymentTestEnvironment {
public:
    ExchangePaymentTestEnvironment();

    // Setup methods
    void setupExchangeRatesManager();
    void setupCommissionsManager();
    void setupPathsManager();
    void setupContractorsManager();

    // Factory methods
    CoordinatorReservationRequestMessage::Shared createRequestMessage(
        optional<ExchangeRate> expectedRate = nullopt,
        optional<TrustLineAmount> expectedCommission = nullopt);

    CoordinatorReservationResponseMessage::Shared createResponseMessage(
        ResponseMessage::OperationState state,
        optional<ExchangeRate> actualRate = nullopt,
        optional<TrustLineAmount> actualCommission = nullopt);

    OptimalPathResult createTestPath(
        vector<ContractorID> nodes,
        vector<SerializedEquivalent> equivalents,
        vector<ExchangeRate> rates,
        vector<TrustLineAmount> commissions);

    // Member variables (real objects, not mocks)
    unique_ptr<ExchangeRatesManager> exchangeRatesManager;
    unique_ptr<CommissionsManager> commissionsManager;
    unique_ptr<ExchangePathsManager> exchangePathsManager;
    unique_ptr<ContractorsManager> contractorsManager;
    // ... other managers as needed
};
```

## Step 2: Message Protocol Tests

**File**: `tests/unit/network/messages/CoordinatorReservationConditionsMessageTest.cpp`

### 2.1: CoordinatorReservationRequestMessage tests
```cpp
TEST(CoordinatorReservationRequestMessage, SerializeWithExchangeRate) {
    // Create message with expectedExchangeRate
    ExchangeRate rate{5, -1};
    auto message = /* create with rate */;

    // Serialize
    auto [buffer, size] = message->serializeToBytes();

    // Deserialize
    auto deserialized = make_shared<CoordinatorReservationRequestMessage>(buffer);

    // Verify
    EXPECT_TRUE(deserialized->expectedExchangeRate().has_value());
    EXPECT_EQ(deserialized->expectedExchangeRate()->value(), 5);
    EXPECT_EQ(deserialized->expectedExchangeRate()->shift(), -1);
    EXPECT_FALSE(deserialized->expectedCommission().has_value());
}

TEST(CoordinatorReservationRequestMessage, SerializeWithCommission) {
    // Similar structure for commission
}

TEST(CoordinatorReservationRequestMessage, SerializeWithBothEmpty) {
    // Test both fields nullopt
}

TEST(CoordinatorReservationRequestMessage, BackwardCompatibility) {
    // Create old format buffer (without new fields)
    // Deserialize with new class
    // Verify both fields are nullopt
}

// ... similar tests for CoordinatorReservationResponseMessage
```

### 2.2: ResponseMessage::OperationState tests
```cpp
TEST(ResponseMessage, RejectedDueConditionsChangedRecognized) {
    auto response = /* create with RejectedDueConditionsChanged */;
    EXPECT_EQ(response->state(), ResponseMessage::RejectedDueConditionsChanged);
    EXPECT_EQ(response->state(), 12);
}
```

## Step 3: Intermediate Node Validation Tests

**File**: `tests/unit/transactions/payments/IntermediateNodeExchangePaymentTransactionValidationTest.cpp`

### 3.1: Protocol violation test
```cpp
TEST_F(IntermediateNodeValidationTest, ProtocolViolation_BothFieldsPresent) {
    // Setup
    auto request = createRequestWithBothFields();
    auto transaction = createIntermediateTransaction(request);

    // Execute
    auto result = transaction->runCoordinatorRequestProcessingStage();

    // Verify
    EXPECT_TRUE(isRejectedStatus(result));
    EXPECT_FALSE(isRejectedDueConditionsChanged(result));
    // Verify error log contains "Protocol violation"
}
```

### 3.2: Exchange rate validation tests
```cpp
TEST_F(IntermediateNodeValidationTest, ExchangeRateMatches_ValidationPasses) {
    // Setup: Expected rate = actual rate in ExchangeRatesManager
    ExchangeRate rate{5, -1};
    env.exchangeRatesManager->set(equiv1, equiv2, rate);
    auto request = createRequestWithExchangeRate(rate);

    // Execute
    auto transaction = createIntermediateTransaction(request);
    auto result = transaction->runCoordinatorRequestProcessingStage();

    // Verify
    EXPECT_TRUE(isAccepted(result));
    EXPECT_TRUE(isRateInContext(transaction, equiv1, equiv2, rate));
}

TEST_F(IntermediateNodeValidationTest, ExchangeRateMismatch_RejectedWithActual) {
    // Setup: Expected rate != actual rate
    ExchangeRate expectedRate{5, -1};
    ExchangeRate actualRate{6, -1};
    env.exchangeRatesManager->set(equiv1, equiv2, actualRate);
    auto request = createRequestWithExchangeRate(expectedRate);

    // Execute
    auto transaction = createIntermediateTransaction(request);
    auto result = transaction->runCoordinatorRequestProcessingStage();

    // Verify
    EXPECT_TRUE(isRejectedDueConditionsChanged(result));
    auto response = getResponse(result);
    EXPECT_TRUE(response->actualExchangeRate().has_value());
    EXPECT_EQ(*response->actualExchangeRate(), actualRate);
    EXPECT_FALSE(hasIncomingReservation(transaction, pathID));
}

// Tests 3-5: Similar structure for other exchange rate scenarios
```

### 3.3: Commission validation tests
```cpp
TEST_F(IntermediateNodeValidationTest, CommissionMatches_ValidationPasses) {
    // Similar to exchange rate matching test
}

TEST_F(IntermediateNodeValidationTest, CommissionMismatch_RejectedWithActual) {
    // Similar to exchange rate mismatch test
}

TEST_F(IntermediateNodeValidationTest, CommissionFromContext_SecondPath) {
    // Test commission "charge once" logic
    // First path: commission stored in context
    // Second path: context value used, not charged again
}

// Tests 7-10: Other commission scenarios
```

### 3.4: Context storage tests
```cpp
TEST_F(IntermediateNodeValidationTest, ContextNotOverwrittenOnSubsequentPaths) {
    // First path with rate1
    // Second path with rate2 in request
    // Verify context still has rate1
}

TEST_F(IntermediateNodeValidationTest, TransactionCompletionBlockedByContext) {
    // No reservations, but context has rates
    // Verify canCompleteTransaction() returns false
}
```

### 3.5: Outgoing/incoming validation tests
```cpp
TEST_F(IntermediateNodeValidationTest, SameEquivalentWithoutCommission_Valid) {
    // Incoming = 100, outgoing = 100, no commission
    // Validation passes
}

TEST_F(IntermediateNodeValidationTest, SameEquivalentWithCommission_Valid) {
    // Incoming = 110, outgoing = 100, commission = 10
    // Validation passes
}

TEST_F(IntermediateNodeValidationTest, SameEquivalentCommissionMismatch_Rejected) {
    // Incoming = 110, outgoing = 105, commission = 10
    // Expected outgoing = 100, actual = 105
    // Rejected
}

// Tests 19-20: Exchange rate validation
```

### 3.6: Commission "charge once" tests
```cpp
TEST_F(IntermediateNodeValidationTest, CommissionChargedOnFirstPathOnly) {
    // Path 1: commission 10 charged
    // Path 2: commission not charged (already in context)
    // Verify amounts
}
```

## Step 4: Coordinator Condition Change Tests

**File**: `tests/unit/transactions/payments/CoordinatorExchangePaymentTransactionConditionChangeTest.cpp`

### 4.1: Condition population tests
```cpp
TEST_F(CoordinatorConditionChangeTest, PopulateExchangeRateInRemoteRequest) {
    // Setup path with exchanger at remote position
    auto path = createPathWithExchanger();

    // Execute askRemoteNodeToApproveReservation
    auto coordinator = createCoordinator(path);
    coordinator->askRemoteNodeToApproveReservation(...);

    // Verify request message contains expectedExchangeRate
    auto sentMessage = getSentMessage();
    EXPECT_TRUE(sentMessage->expectedExchangeRate().has_value());
}

TEST_F(CoordinatorConditionChangeTest, PopulateCommissionInNeighborRequest) {
    // Test askNeighborToApproveFurtherNodeReservation
}
```

### 4.2: Condition change detection tests
```cpp
TEST_F(CoordinatorConditionChangeTest, DetectRejectedDueConditionsChanged) {
    // Setup coordinator waiting for response
    // Send RejectedDueConditionsChanged response
    // Verify detection in processRemoteNodeResponse
}
```

### 4.3: Exchange rate change handling tests
```cpp
TEST_F(CoordinatorConditionChangeTest, ExchangeRateIncreased_PathRecalculated) {
    // Path with rate 0.5, response with rate 0.6
    // Verify:
    // - Path marked unusable
    // - ExchangeRatesManager updated
    // - New path created
    // - New path received_amount correct
}

TEST_F(CoordinatorConditionChangeTest, ExchangeRateDecreased_PathRecalculated) {
    // Path with rate 0.5, response with rate 0.4
}

TEST_F(CoordinatorConditionChangeTest, NewPathFlowsRecalculated) {
    // Verify calculateFlows() called on new path
}

TEST_F(CoordinatorConditionChangeTest, ReservationsDropped) {
    // Verify dropReservationsOnPath called
    // Verify FinalPathExchangeConfigurationMessage sent
}
```

### 4.4: Commission change handling tests
```cpp
TEST_F(CoordinatorConditionChangeTest, CommissionIncreased) {
    // Similar to exchange rate tests
}

TEST_F(CoordinatorConditionChangeTest, CommissionRemoved) {
    // Commission = 0 in response
    // Verify CommissionsManager.remove() called
}
```

### 4.5: Path recalculation tests
```cpp
TEST_F(CoordinatorConditionChangeTest, CalculateReceivedAmountWithUpdatedRate) {
    // Test calculateReceivedAmountWithUpdatedConditions
    // Input: path, flow, new rate
    // Output: correct received_amount
}

TEST_F(CoordinatorConditionChangeTest, CalculateReceivedAmountWithUpdatedCommission) {
    // Test with updated commission
}
```

## Step 5: Subsequent Paths Recalculation Tests

**File**: `tests/unit/transactions/payments/SubsequentPathsRecalculationTest.cpp`

### 5.1: Path identification tests
```cpp
TEST_F(SubsequentPathsTest, IdentifyPathsContainingAffectedNode) {
    // 5 paths, paths 2, 4, 5 contain node X
    // Condition change on path 1
    // Verify paths 2, 4, 5 identified
}

TEST_F(SubsequentPathsTest, SkipNewlyCreatedPath) {
    // New path inserted at currentIndex + 1
    // Verify not recalculated
}
```

### 5.2: Recalculation tests
```cpp
TEST_F(SubsequentPathsTest, RecalculateReceivedAmountForSubsequentPaths) {
    // Paths 2, 3 contain exchanger, rate changes
    // Verify both have received_amount recalculated
}

TEST_F(SubsequentPathsTest, RecalculateFlowsForSubsequentPaths) {
    // Verify calculateFlows called for affected paths
}
```

### 5.3: Error handling tests
```cpp
TEST_F(SubsequentPathsTest, HandleRecalculationErrorsGracefully) {
    // Mock calculateReceivedAmount to throw
    // Verify error logged, path skipped, others continue
}
```

## Step 6: Regression Tests

**File**: `tests/unit/transactions/payments/ExchangePaymentRegressionTest.cpp`

### 6.1: Unchanged behavior tests
```cpp
TEST_F(ExchangePaymentRegressionTest, PaymentWithoutConditionChanges) {
    // Execute full payment with matching conditions
    // Verify no condition change logic triggered
    // Verify payment succeeds as before
}

TEST_F(ExchangePaymentRegressionTest, SingleEquivalentPaymentUnaffected) {
    // Test single-equivalent payment
    // Verify new code doesn't affect it
}

TEST_F(ExchangePaymentRegressionTest, PathCapacityAdjustmentStillWorks) {
    // Test PRD 08 functionality
    // Verify capacity adjustment unaffected by new code
}
```

## Step 7: Test Utilities and Fixtures

### 7.1: Create test fixtures
```cpp
class IntermediateNodeValidationTest : public ::testing::Test {
protected:
    void SetUp() override {
        env.setupExchangeRatesManager();
        env.setupCommissionsManager();
        // ...
    }

    ExchangePaymentTestEnvironment env;
    // Helper methods...
};

class CoordinatorConditionChangeTest : public ::testing::Test {
    // Similar setup
};
```

### 7.2: Helper methods for assertions
```cpp
bool isRejectedDueConditionsChanged(const TransactionResult::SharedConst &result);
bool isRateInContext(Transaction *tx, SerializedEquivalent eq1, SerializedEquivalent eq2, ExchangeRate rate);
bool hasIncomingReservation(Transaction *tx, PathID pathID);
// ... other helpers
```

# Test Plan

## Test Organization
- **Message Protocol Tests**: `tests/unit/network/messages/CoordinatorReservationConditionsMessageTest.cpp`
- **Intermediate Node Tests**: `tests/unit/transactions/payments/IntermediateNodeExchangePaymentTransactionValidationTest.cpp`
- **Coordinator Tests**: `tests/unit/transactions/payments/CoordinatorExchangePaymentTransactionConditionChangeTest.cpp`
- **Subsequent Paths Tests**: `tests/unit/transactions/payments/SubsequentPathsRecalculationTest.cpp`
- **Regression Tests**: `tests/unit/transactions/payments/ExchangePaymentRegressionTest.cpp`

## Test Execution
- Build in `build-tests` directory
- Execute: `./build-tests/bin/unit_tests --gtest_filter=*Exchange*Condition*`
- Coverage: `--coverage` flag during build

## Test Data
- Use realistic exchange rates (0.5, 2.0, 0.6, 0.4)
- Use realistic commissions (5, 10, 7)
- Use realistic amounts (100, 200, 1000, 1990)
- Follow examples from PRD (5-node path example)

## Success Criteria
- All 66+ unit tests pass
- 100% coverage of validation and condition change logic
- No memory leaks
- Tests execute in < 5 seconds total
- All edge cases covered
- Comprehensive logging verification

# Verification and Validation

## Architecture integrity
**Validation Level**: Moderate task
- Tests follow existing test patterns
- Uses TestEnvironment helper classes
- Real objects over mocks (per project policy)
- No test interdependencies

## Security
**Validation Level**: Moderate task
- No security implications in tests
- Tests validate security aspects of main code

## Performance
**Validation Level**: Moderate task
- Test execution time: < 5 seconds total
- Individual test time: < 100ms
- Memory usage reasonable for unit tests

## Scalability
**Validation Level**: Moderate task
- Tests cover various path counts (1-50)
- Tests cover various equivalent counts (1-5)

## Reliability
**Validation Level**: Moderate task
- Tests are deterministic (no random data)
- Tests are repeatable
- Test failures are clear and actionable

## Maintainability
**Validation Level**: Moderate task
- Clear test names indicate purpose
- Test fixtures reduce duplication
- Helper methods improve readability
- Comments explain complex setups

## Cost
**Validation Level**: Moderate task
- No infrastructure cost

## Compliance
**Validation Level**: Moderate task
- Follows project testing policy
- Uses real objects per policy.md
- Covers all PRD acceptance criteria

# Restrictions
- Commit tests only after all pass
- Use real objects, not mocks (per policy.md)
- Follow ExchangePathsManagerTest pattern
- Ensure tests are self-contained
- Include clear failure messages
