# 08-05 - Unit Tests for Path Capacity Adjustment

# Links
- [PRD](../../../prd/vtcpd/08-path-capacity-adjustment.md)
- [Previous task 1](08-01-enhanced-shortage-reservations.md)
- [Previous task 2](08-02-incoming-reservation-calculation.md)
- [Previous task 3](08-03-path-addition-and-helpers.md)
- [Previous task 4](08-04-path-filtering-and-truncation.md)

# Description
Implement comprehensive unit tests for all functionality developed in tasks 08-01 through 08-04. Tests validate path capacity adjustment in coordinator, incoming reservation calculation in intermediate nodes, path addition strategy, helper functions, and path filtering/truncation logic.

Tests follow project standards:
- Real objects over mocks (following ExchangePathsManagerTest pattern)
- TestEnvironment helper classes for consistent setup
- Exception testing using EXPECT_THROW
- Edge case coverage
- Located in `tests/unit/transactions/`

# Requirements and DOD

## Test Coverage Requirements

### 1. CoordinatorExchangePaymentTransaction::shortageReservationsOnPath Tests (5 tests)
From task 08-01:

**Test 1: Shortage on simple path without exchanges or commissions**
- Setup: Path A→B→C→D (all in eq 1001), initial `mMaxPathFlow`=1000, `optimal_flow`=1000, `received_amount`=1000
- Node B returns reservation: 700
- Expected: All fields updated (700, 700, 700), flows recalculated
- Assertions: `mMaxPathFlow==700`, `optimal_flow==700`, `received_amount==700`, `flows.size()>0`

**Test 2: Shortage on path with exchange**
- Setup: Path A→B (eq 1001) →C [exchange 1001→2002, rate=0.5] →D (eq 2002)
- Initial: `mMaxPathFlow`=2000, `optimal_flow`=2000, `received_amount`=1000
- Node B returns: 1400
- Expected: Forward simulate 1400 → exchange → 700
- Assertions: `mMaxPathFlow==1400`, `optimal_flow==1400`, `received_amount==700`, flows correct

**Test 3: Shortage on path with commission**
- Setup: Path A→B→C→D (eq 1001), commission at C = 10 units
- Initial: `mMaxPathFlow`=1000, `optimal_flow`=1000, `received_amount`=990
- Node B returns: 700
- Expected: 700 → arrive at C → subtract 10 → 690
- Assertions: `mMaxPathFlow==700`, `optimal_flow==700`, `received_amount==690`, flows reflect commission

**Test 4: Shortage with commission overflow**
- Setup: Path A→B→C→D, commission at C = 50 units
- Node B returns: 30 (less than commission)
- Expected: Path marked unusable (`setUnusable()` called), fields not updated
- Assertions: `pathStats->isValid()==false`, old field values preserved

**Test 5: Path not found in mPathsStats**
- Setup: Empty `mPathsStats` or nonexistent pathID
- Expected: Warning logged, no crash, return early
- Assertions: No crash, warning in log

### 2. IntermediateNodeExchangePaymentTransaction Incoming Calculation Tests (5 tests)
From task 08-02:

**Test 6: Incoming calculation - same equivalent without commission**
- Setup: Outgoing=500 in eq 1001, Incoming eq=1001, no commission
- Expected: `incomingAmount=500`, `shortageIncomingReservationsOnPath` called with correct params
- Assertions: Method called with (pathID, 1001, 500)

**Test 7: Incoming calculation - same equivalent with commission**
- Setup: Outgoing=500 in eq 1001, Incoming eq=1001, commission=20
- Expected: `incomingAmount=520` (500+20)
- Assertions: Method called with (pathID, 1001, 520)

**Test 8: Incoming calculation - different equivalents with exchange**
- Setup: Outgoing=100 in eq 2002, Incoming eq=1001, exchange rate 1001→2002 = 0.5
- Expected: Invert: `incomingAmount=200` (100/0.5)
- Assertions: Method called with (pathID, 1001, 200)

**Test 9: Incoming calculation - exchange rate not found**
- Setup: Outgoing eq=2002, Incoming eq=3003, no exchange rate
- Expected: Error logged, error message to coordinator, transaction terminates
- Assertions: Error message sent, transaction ends

**Test 10: Incoming reservation not found by PathID**
- Setup: No incoming reservation with matching PathID
- Expected: Error logged, error message to coordinator
- Assertions: Error message sent, transaction ends

### 3. Path Addition Strategy Tests (1 test)
From task 08-03 Part A:

**Test 11: Add all paths without truncation**
- Setup: 3 cached paths with capacity 500, 300, 200; payment amount=600
- Expected: ALL 3 paths added to `mPathsStats` with full capacity (not truncated)
- Assertions: `mPathsStats.size()==3`, each path's `received_amount` matches original

### 4. Path Filtering Tests (2 tests)
From task 08-04:

**Test 12: Filter path with inaccessible node**
- Setup: Path contains node C, `mInaccessibleNodes` contains node C
- Expected: Path marked unusable, moved to next path
- Assertions: `pathStats->isValid()==false`, next path processed

**Test 13: Filter path with rejected trust line**
- Setup: Path contains edge A→B, `mRejectedTrustLines` contains (A, B)
- Expected: Path marked unusable, info log
- Assertions: `pathStats->isValid()==false`

### 5. Capacity Truncation Tests (2 tests)
From task 08-04:

**Test 14: Truncate path exceeding remaining need**
- Setup: Already reserved=400 (toward mAmount=600), remaining=200, next path `received_amount`=500
- Expected: Path truncated to deliver exactly 200, `received_amount` updated to 200
- Assertions: `pathStats->received_amount==200`, `mMaxPathFlow` updated correctly

**Test 15: No truncation when path fits remaining need**
- Setup: Already reserved=400, remaining=200, next path `received_amount`=150
- Expected: No truncation (150 < 200), proceed with full capacity
- Assertions: `pathStats->received_amount==150` (unchanged)

## Definition of Done
- [ ] All 15 unit tests implemented
- [ ] Tests located in `tests/unit/transactions/` (or appropriate subdirectory)
- [ ] Tests use real objects (following ExchangePathsManagerTest pattern)
- [ ] TestEnvironment helper classes created for consistent setup (if needed)
- [ ] Exception testing uses EXPECT_THROW where appropriate
- [ ] Edge cases covered (empty data, null pointers, overflow, etc.)
- [ ] All tests pass in `build-tests`
- [ ] Test file follows project naming conventions
- [ ] Tests are well-commented and self-documenting
- [ ] No test interdependencies (each test can run independently)

# Implementation Plan

## Step 1: Create test file structure
- Location: `tests/unit/transactions/TestCoordinatorExchangePaymentCapacityAdjustment.cpp` (or similar name)
- Include necessary headers:
  ```cpp
  #include <gtest/gtest.h>
  #include "core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h"
  #include "core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h"
  #include "core/paths/lib/OptimalPathResult.h"
  #include "core/paths/lib/ExchangePath.h"
  // Add other necessary includes
  ```

## Step 2: Create TestEnvironment helper class (if needed)
Following ExchangePathsManagerTest pattern:

```cpp
class CoordinatorExchangePaymentTestEnvironment {
public:
    CoordinatorExchangePaymentTestEnvironment() {
        // Initialize managers, components, etc.
        // Similar to ExchangePathsManagerTest setup
    }

    // Helper methods for creating paths, reservations, etc.
    OptimalPathResult createSimplePath(
        const vector<ContractorID> &ids,
        const vector<SerializedEquivalent> &equivalents,
        const TrustLineAmount &capacity);

    OptimalPathResult createPathWithExchange(
        const ContractorID exchangeNode,
        const SerializedEquivalent fromEquiv,
        const SerializedEquivalent toEquiv,
        const TrustLineAmount &exchangeRate,
        int16_t shift);

    OptimalPathResult createPathWithCommission(
        const ContractorID commissionNode,
        const TrustLineAmount &commission);

    // Add other helpers as needed
};
```

## Step 3: Implement Tests 1-5 (shortageReservationsOnPath)

### Test 1 Example:
```cpp
TEST(CoordinatorExchangePaymentTest, ShortageOnSimplePath) {
    // Arrange
    CoordinatorExchangePaymentTestEnvironment env;
    auto path = env.createSimplePath(
        {1, 2, 3, 4},  // Node IDs: A=1, B=2, C=3, D=4
        {1001, 1001, 1001, 1001},  // All same equivalent
        1000);  // Initial capacity

    ASSERT_EQ(path.mMaxPathFlow, 1000);
    ASSERT_EQ(path.optimal_flow, 1000);
    ASSERT_EQ(path.received_amount, 1000);

    // Act
    // Simulate node B (ID=2) returning reduced reservation of 700
    // Call shortageReservationsOnPath or trigger scenario
    env.simulateNodeResponse(2, pathID, 700);

    // Assert
    EXPECT_EQ(path.mMaxPathFlow, 700);
    EXPECT_EQ(path.optimal_flow, 700);
    EXPECT_EQ(path.received_amount, 700);
    EXPECT_GT(path.flows.size(), 0);  // Flows recalculated
}
```

### Test 2: With Exchange
Setup exchange step at node C (1001→2002, rate=0.5), verify correct forward simulation.

### Test 3: With Commission
Add commission step, verify commission subtraction in received_amount.

### Test 4: Commission Overflow
Create scenario where amount < commission, verify `setUnusable()` called, use mocking or spy pattern if needed.

### Test 5: Path Not Found
Pass invalid pathID, verify warning logged (may need log capture mechanism).

## Step 4: Implement Tests 6-10 (Incoming calculation)

### Test 6 Example:
```cpp
TEST(IntermediateNodeExchangePaymentTest, IncomingCalculationSameEquivNoCommission) {
    // Arrange
    IntermediateNodeTestEnvironment env;
    env.setupReservations(pathID, 500, 1001, 1001);  // outgoing=500, both eq=1001
    env.setCommission(1001, nullopt);  // No commission

    // Act
    env.triggerIncomingCalculation();

    // Assert
    EXPECT_TRUE(env.wasShortageIncomingCalled());
    EXPECT_EQ(env.getIncomingAmount(), 500);
    EXPECT_EQ(env.getIncomingEquiv(), 1001);
}
```

### Test 7: With Commission
Set commission=20, verify incoming=520.

### Test 8: With Exchange
Setup exchange rate, verify inverse calculation.

### Test 9: Exchange Rate Not Found
Mock exchange rate lookup failure, verify error message sent.

### Test 10: Incoming Reservation Not Found
Setup without incoming reservation, verify error handling.

## Step 5: Implement Test 11 (Path Addition)

```cpp
TEST(CoordinatorExchangePaymentTest, AddAllPathsWithoutTruncation) {
    // Arrange
    CoordinatorExchangePaymentTestEnvironment env;
    env.setupCachedPaths({
        {500, 1001},  // Path 1: capacity=500, equiv=1001
        {300, 1001},  // Path 2: capacity=300
        {200, 1001}   // Path 3: capacity=200
    });
    env.setPaymentAmount(600);  // Only need 600, but all should be added

    // Act
    env.runPathsResourceProcessingStage();

    // Assert
    EXPECT_EQ(env.getPathsStatsSize(), 3);  // All 3 paths added
    EXPECT_EQ(env.getPathCapacity(0), 500);  // Not truncated
    EXPECT_EQ(env.getPathCapacity(1), 300);
    EXPECT_EQ(env.getPathCapacity(2), 200);
}
```

## Step 6: Implement Tests 12-13 (Path Filtering)

### Test 12 Example:
```cpp
TEST(CoordinatorExchangePaymentTest, FilterPathWithInaccessibleNode) {
    // Arrange
    CoordinatorExchangePaymentTestEnvironment env;
    auto nodeC = env.createNode("C");
    env.addToInaccessibleNodes(nodeC);
    auto path = env.createPathContainingNode(nodeC);

    // Act
    bool isValid = env.validatePathForProcessing(&path);

    // Assert
    EXPECT_FALSE(isValid);
    EXPECT_FALSE(path.isValid());  // Marked unusable
}
```

### Test 13: Rejected Trust Line
Similar setup but with edge check.

## Step 7: Implement Tests 14-15 (Capacity Truncation)

### Test 14 Example:
```cpp
TEST(CoordinatorExchangePaymentTest, TruncatePathExceedingRemaining) {
    // Arrange
    CoordinatorExchangePaymentTestEnvironment env;
    env.setAlreadyReserved(400);
    env.setTotalAmount(600);  // Remaining = 200
    auto path = env.createPath(500);  // Path capacity = 500

    // Act
    env.applyCapacityTruncation(&path);

    // Assert
    EXPECT_EQ(path.received_amount, 200);  // Truncated to remaining
    // Verify mMaxPathFlow also updated correctly
}
```

### Test 15: No Truncation
Path capacity (150) < remaining (200), verify unchanged.

## Step 8: Add tests to CMakeLists.txt
File: `tests/unit/CMakeLists.txt`

Add new test file to build:
```cmake
add_executable(unit_tests
    # ... existing test files ...
    transactions/TestCoordinatorExchangePaymentCapacityAdjustment.cpp
)
```

## Step 9: Build and run tests
```bash
cd build-tests
cmake ..
make
./bin/unit_tests --gtest_filter="*CoordinatorExchangePayment*"
./bin/unit_tests --gtest_filter="*IntermediateNodeExchangePayment*"
```

# Test Plan
This task IS the test plan implementation. Tests validate all functionality from tasks 08-01 through 08-04.

# Verification and Validation

## Architecture integrity
- **Complex Task Validation**: Comprehensive test coverage across multiple components
- Tests verify integration between coordinator and path structures
- Tests verify helper functions work correctly with real path data
- Tests follow project testing patterns (ExchangePathsManagerTest style)

## Security
- **Complex Task**: Tests verify no crashes on malicious/malformed input
- Edge case testing includes overflow, null pointers, invalid data
- No security vulnerabilities introduced by testing code

## Performance
- **Complex Task**: Tests should complete in <5 seconds total
- Individual test execution: <100ms each
- No performance regressions introduced

## Scalability
- **Complex Task**: Tests cover typical and edge case scales
- Test with paths up to 10 nodes
- Test with up to 5 exchanges per path
- Test with up to 100 inaccessible nodes (if feasible)

## Reliability
- **Complex Task**: Tests are deterministic and repeatable
- No flaky tests (same input always produces same output)
- Tests can run in any order (no interdependencies)
- Clear failure messages for debugging

## Maintainability
- **Complex Task**: Tests are well-structured and documented
- TestEnvironment helper makes tests readable
- Clear test names describe what is being tested
- Comments explain non-obvious setup or assertions

## Cost
- **Complex Task**: Test suite overhead acceptable (<5s execution)
- No excessive resource usage in test environment

## Compliance
- **Complex Task**: Follows project testing standards
- Uses EXPECT_* macros correctly
- Test file naming matches project conventions
- All tests pass before commit

# Restrictions
- Tests must pass before committing any code from tasks 08-01 through 08-04
- Use real objects (not mocks) following project patterns
- Test file must be added to CMakeLists.txt
- All 15 tests must be implemented (no skipping)
- Tests must be independent (no shared state between tests)
