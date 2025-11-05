# 10-03 - Unit Tests for Allowable Payment Amount Control

# Links
- [PRD](../../prd/vtcpd/10-allowable-payment-amount-control.md)
- [Previous task: 10-01 Command Extension](10-01-command-extension-and-result-code.md)
- [Previous task: 10-02 Validation Logic](10-02-validation-and-calculation.md)

# Description
Create comprehensive unit tests for allowable payment amount control feature, covering command parsing, total reserved amount calculation, early and late validation, result codes, and end-to-end scenarios. Tests must verify correct behavior when limit is exceeded, within limit, and when parameter is not provided (legacy behavior).

# Requirements and DOD

## Functional Requirements

### Test Coverage Areas
1. **Command Parsing Tests** (4 tests):
   - Parse command with maxAllowablePaymentAmount parameter
   - Parse command without parameter (legacy)
   - Parse command with invalid parameter
   - Verify responseAllowablePaymentAmountExceeded() returns code 415

2. **Total Reserved Amount Calculation Tests** (4 tests):
   - Calculate with no outgoing reservations
   - Calculate with one exchange equivalent
   - Calculate with multiple exchange equivalents
   - Verify incomplete reservations not counted

3. **Early Validation Tests** (3 tests):
   - Validation passes when mExchangeAmount within limit
   - Validation fails when mExchangeAmount exceeds limit
   - Validation skipped when parameter not provided

4. **Late Validation Tests** (5 tests):
   - Validation passes in processRemoteNodeResponse
   - Validation fails in processRemoteNodeResponse
   - Validation fails even if mAmount target achieved
   - Validation skipped when parameter not provided
   - Validation in processNeighborFurtherReservationResponse

5. **Result Code Tests** (1 test):
   - resultAllowablePaymentAmountExceeded() returns code 415

6. **End-to-End Scenario Tests** (5 tests):
   - Payment succeeds when conditions stable and within limit
   - Payment fails early when estimated amount exceeds limit
   - Payment fails late when conditions change causing violation
   - Payment succeeds without limit parameter (legacy)
   - Rollback mechanism correctly drops all reservations

## Definition of Done
- [ ] All 22 unit tests implemented in `tests/unit/` subdirectory
- [ ] Tests follow existing test patterns (use real objects, TestEnvironment helpers)
- [ ] Command parsing tests verify parameter extraction
- [ ] Calculation tests verify correct summation across equivalents
- [ ] Early validation tests verify behavior in runPathsResourceProcessingStage
- [ ] Late validation tests verify behavior in both response processing methods
- [ ] Result code tests verify code 415 returned correctly
- [ ] End-to-end tests verify complete payment flows
- [ ] Rollback verification tests confirm all reservations dropped
- [ ] All tests pass in `build-tests` environment
- [ ] Tests are deterministic and repeatable
- [ ] Test names clearly describe what is being tested
- [ ] Test setup and assertions are clear and maintainable

# Implementation Plan

## Test File Structure
Create test file: `tests/unit/transactions/coordinator_exchange_payment_allowable_amount_test.cpp`

Follow existing test patterns from:
- `tests/unit/transactions/` directory structure
- Use real objects (no mocks) following project conventions
- Use TestEnvironment helper classes for setup

## Test Group 1: Command Parsing Tests

### Test 1: Parse Command with maxAllowablePaymentAmount
```cpp
TEST(CreditUsageExchangeCommandTest, ParseWithMaxAllowableAmount) {
    // Setup command string
    string commandStr = "CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1:1500";

    // Parse command
    auto command = make_shared<CreditUsageExchangeCommand>(uuid, commandStr);

    // Assertions
    ASSERT_TRUE(command->maxAllowablePaymentAmount().has_value());
    EXPECT_EQ(*command->maxAllowablePaymentAmount(), TrustLineAmount(1500));
    // Verify other parameters parsed correctly
}
```

### Test 2: Parse Command without Parameter (Legacy)
```cpp
TEST(CreditUsageExchangeCommandTest, ParseWithoutMaxAllowableAmount) {
    string commandStr = "CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1";

    auto command = make_shared<CreditUsageExchangeCommand>(uuid, commandStr);

    EXPECT_FALSE(command->maxAllowablePaymentAmount().has_value());
    // Verify other parameters parsed correctly
}
```

### Test 3: Parse Command with Invalid Parameter
```cpp
TEST(CreditUsageExchangeCommandTest, ParseWithInvalidMaxAllowableAmount) {
    // Test with leading zero
    string commandStr = "CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1:0150";

    EXPECT_THROW({
        auto command = make_shared<CreditUsageExchangeCommand>(uuid, commandStr);
    }, ValueError);
}
```

### Test 4: Result Code 415
```cpp
TEST(CreditUsageExchangeCommandTest, ResponseAllowableAmountExceeded) {
    string commandStr = "CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1:1500";
    auto command = make_shared<CreditUsageExchangeCommand>(uuid, commandStr);

    auto result = command->responseAllowablePaymentAmountExceeded();

    EXPECT_EQ(result->code(), 415);
    EXPECT_TRUE(result->message().find("Allowable payment amount has been exceeded") != string::npos);
}
```

## Test Group 2: Total Reserved Amount Calculation Tests

### Test 5: Calculate with No Reservations
```cpp
TEST(CoordinatorExchangePaymentTest, CalculateTotalReservedWithNoReservations) {
    // Setup transaction with no outgoing reservations
    // ...

    auto total = transaction->calculateTotalReservedPaymentAmount();

    EXPECT_EQ(total, TrustLineAmount(0));
}
```

### Test 6: Calculate with One Equivalent
```cpp
TEST(CoordinatorExchangePaymentTest, CalculateTotalReservedOneEquivalent) {
    // Setup transaction with one exchange equivalent
    // Mock totalReservedAmount to return 1000
    // ...

    auto total = transaction->calculateTotalReservedPaymentAmount();

    EXPECT_EQ(total, TrustLineAmount(1000));
}
```

### Test 7: Calculate with Multiple Equivalents
```cpp
TEST(CoordinatorExchangePaymentTest, CalculateTotalReservedMultipleEquivalents) {
    // Setup transaction with two exchange equivalents
    // Equivalent 1: 1200, Equivalent 2: 600
    // ...

    auto total = transaction->calculateTotalReservedPaymentAmount();

    EXPECT_EQ(total, TrustLineAmount(1800));
}
```

### Test 8: Incomplete Reservations Not Counted
```cpp
TEST(CoordinatorExchangePaymentTest, IncompletReservationsNotCounted) {
    // Setup with partial reservations
    // Verify totalReservedAmount behavior
    // ...

    auto total = transaction->calculateTotalReservedPaymentAmount();

    // Verify only completed reservations counted
}
```

## Test Group 3: Early Validation Tests

### Test 9: Early Validation Passes (Within Limit)
```cpp
TEST(CoordinatorExchangePaymentTest, EarlyValidationPassesWithinLimit) {
    // Setup: maxAllowablePaymentAmount = 2000, mExchangeAmount = 1800
    // ...

    auto result = transaction->runPathsResourceProcessingStage();

    // Verify transaction continues (not code 415)
    EXPECT_NE(result->code(), 415);
}
```

### Test 10: Early Validation Fails (Exceeds Limit)
```cpp
TEST(CoordinatorExchangePaymentTest, EarlyValidationFailsExceedsLimit) {
    // Setup: maxAllowablePaymentAmount = 1500, mExchangeAmount = 1800
    // ...

    auto result = transaction->runPathsResourceProcessingStage();

    EXPECT_EQ(result->code(), 415);
    // Verify no reservations made
}
```

### Test 11: Early Validation Skipped (No Parameter)
```cpp
TEST(CoordinatorExchangePaymentTest, EarlyValidationSkippedNoParameter) {
    // Setup: maxAllowablePaymentAmount = nullopt, mExchangeAmount = 10000
    // ...

    auto result = transaction->runPathsResourceProcessingStage();

    // Verify transaction continues normally
    EXPECT_NE(result->code(), 415);
}
```

## Test Group 4: Late Validation Tests

### Test 12: Late Validation Passes (processRemoteNodeResponse)
```cpp
TEST(CoordinatorExchangePaymentTest, LateValidationPassesRemoteNode) {
    // Setup: maxAllowablePaymentAmount = 2000
    // Path completed, total reserved = 1800
    // ...

    auto result = transaction->processRemoteNodeResponse(...);

    EXPECT_NE(result->code(), 415);
}
```

### Test 13: Late Validation Fails (processRemoteNodeResponse)
```cpp
TEST(CoordinatorExchangePaymentTest, LateValidationFailsRemoteNode) {
    // Setup: maxAllowablePaymentAmount = 1500
    // Path completed, total reserved = 1800
    // ...

    auto result = transaction->processRemoteNodeResponse(...);

    EXPECT_EQ(result->code(), 415);
    // Verify rollBack() called
    // Verify all reservations dropped
}
```

### Test 14: Late Validation Fails Even When mAmount Achieved
```cpp
TEST(CoordinatorExchangePaymentTest, LateValidationFailsEvenWhenTargetAchieved) {
    // Setup: maxAllowablePaymentAmount = 1500
    // mAmount target achieved, but total reserved = 1800
    // ...

    auto result = transaction->processRemoteNodeResponse(...);

    EXPECT_EQ(result->code(), 415);
}
```

### Test 15: Late Validation Skipped (No Parameter)
```cpp
TEST(CoordinatorExchangePaymentTest, LateValidationSkippedNoParameter) {
    // Setup: maxAllowablePaymentAmount = nullopt
    // Total reserved = 10000
    // ...

    auto result = transaction->processRemoteNodeResponse(...);

    EXPECT_NE(result->code(), 415);
}
```

### Test 16: Late Validation in processNeighborFurtherReservationResponse
```cpp
TEST(CoordinatorExchangePaymentTest, LateValidationFailsNeighborNode) {
    // Setup: Same as Test 13 but via neighbor
    // ...

    auto result = transaction->processNeighborFurtherReservationResponse(...);

    EXPECT_EQ(result->code(), 415);
}
```

## Test Group 5: Result Code Tests

### Test 17: resultAllowablePaymentAmountExceeded Returns Code 415
```cpp
TEST(CoordinatorExchangePaymentTest, ResultMethodReturnsCode415) {
    // Setup coordinator transaction
    // ...

    auto result = transaction->resultAllowablePaymentAmountExceeded();

    EXPECT_EQ(result->code(), 415);
}
```

## Test Group 6: End-to-End Scenario Tests

### Test 18: Payment Succeeds When Stable and Within Limit
```cpp
TEST(CoordinatorExchangePaymentTest, E2E_PaymentSucceedsWithinLimit) {
    // Setup: maxAllowablePaymentAmount = 2000
    // Estimated mExchangeAmount = 1800
    // No condition changes, final total = 1800
    // ...

    // Execute payment
    auto result = /* run payment flow */;

    EXPECT_EQ(result->code(), 201); // Success
}
```

### Test 19: Payment Fails Early When Estimate Exceeds Limit
```cpp
TEST(CoordinatorExchangePaymentTest, E2E_PaymentFailsEarlyExceedsLimit) {
    // Setup: maxAllowablePaymentAmount = 1500
    // Estimated mExchangeAmount = 1800
    // ...

    auto result = /* run payment flow */;

    EXPECT_EQ(result->code(), 415);
    // Verify no reservations made
}
```

### Test 20: Payment Fails Late When Conditions Change
```cpp
TEST(CoordinatorExchangePaymentTest, E2E_PaymentFailsLateConditionChange) {
    // Setup: maxAllowablePaymentAmount = 1800
    // Estimated mExchangeAmount = 1700
    // Path 1: 900, Path 2: 950 (condition changed)
    // Total = 1850 exceeds 1800
    // ...

    auto result = /* run payment flow */;

    EXPECT_EQ(result->code(), 415);
    // Verify all reservations dropped
}
```

### Test 21: Payment Succeeds Without Parameter (Legacy)
```cpp
TEST(CoordinatorExchangePaymentTest, E2E_PaymentSucceedsNoParameter) {
    // Setup: maxAllowablePaymentAmount = nullopt
    // Estimated mExchangeAmount = 10000 (very high)
    // ...

    auto result = /* run payment flow */;

    EXPECT_EQ(result->code(), 201); // Success despite high cost
}
```

### Test 22: Rollback Drops All Reservations
```cpp
TEST(CoordinatorExchangePaymentTest, E2E_RollbackDropsAllReservations) {
    // Setup: Create reservations on multiple paths
    // Trigger validation failure
    // ...

    auto result = /* trigger validation failure */;

    EXPECT_EQ(result->code(), 415);
    // Verify reservation state after rejection
    // Assert all reservations dropped
}
```

## Test Execution
All tests must be built and executed in `build-tests`:
```bash
cd build-tests
cmake ..
make
./bin/unit_tests --gtest_filter=*CoordinatorExchangePayment*AllowableAmount*
```

# Test Plan

## Test Environment Setup
1. Use existing test infrastructure from `tests/unit/`
2. Follow patterns from existing transaction tests
3. Use real objects (ContractorsManager, EquivalentsSubsystemsRouter, etc.)
4. Create TestEnvironment helper class if needed for common setup

## Test Data Preparation
1. Mock command strings with various parameter combinations
2. Setup test equivalents and addresses
3. Prepare test amounts for calculations
4. Create mock reservation states

## Test Execution Strategy
1. Run tests individually during development
2. Run full test suite before committing
3. Ensure tests are deterministic (no flaky tests)
4. Verify tests pass in CI environment

## Complexity
**Complex** - Comprehensive test suite covering multiple components and scenarios. Requires careful setup and verification of transaction states.

# Verification and Validation

## Architecture integrity
- Tests verify correct integration with existing infrastructure
- Tests confirm no breaking changes to existing functionality
- Tests validate backward compatibility (legacy commands work)

## Security
- Tests verify parameter validation prevents invalid values
- Tests confirm no exposure of internal state

## Performance
- Tests execute in reasonable time (< 5 seconds total)
- No performance degradation from test execution

## Scalability
- Tests cover scalability scenarios (multiple equivalents)
- Tests verify efficient calculation algorithms

## Reliability
- All tests are deterministic and repeatable
- Tests verify graceful error handling
- Tests confirm correct rollback behavior

## Maintainability
- Tests follow existing patterns and conventions
- Test names clearly describe purpose
- Tests are well-documented with comments
- Easy to add new tests in the future

## Cost
- Development effort: 2-3 days
- No infrastructure cost
- Tests execute quickly in CI

## Compliance
- Tests follow project testing standards
- Tests adhere to Google Test framework conventions
- Tests maintain code coverage standards

# Restrictions
- Commit changes only after all tests pass in `build-tests`
- Do not modify production code during test implementation (tests only)
- Do not create integration tests (unit tests only per project policy)
- Ensure tests are isolated and independent (no shared state between tests)
