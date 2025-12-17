# 13-02 - Database Query Method Unit Tests

# Links
- [PRD 13: Observer Successful Transactions Monitoring](../../../prd/vtcpd/13-observer-successful-transactions-monitoring.md)
- [Previous task: 13-01 Database Query Method Implementation](13-01-database-query-method.md)

# Description
Implement comprehensive unit tests for the `transactionsForObserverMonitoring` method added in Task 13-01. Create 8 unit tests for SQLite and 8 integration tests for PostgreSQL to ensure correctness, performance, and edge case handling of the new database query method.

Tests validate filtering logic (block number and state), ordering, limit parameter, boundary conditions, and performance characteristics.

# Requirements and DOD

## Functional Requirements

### SQLite Unit Tests
**File:** `tests/unit/sqlite/PaymentTransactionsHandlerSQLiteTest.cpp`

Add 8 test cases in new test fixture or extend existing `PaymentTransactionsHandlerSQLiteTest`:

1. **TransactionsForObserverMonitoring_BasicRetrieval_ReturnsMatchingTransactions**
   - Setup: Insert 5 transactions with blockNumbers: 50, 100, 150, 200, 250 (all state=0)
   - Execute: `transactionsForObserverMonitoring(100, 10)`
   - Verify: Returns 3 transactions (150, 200, 250) in ascending order

2. **TransactionsForObserverMonitoring_StateFiltering_OnlyReturnsUncertainState**
   - Setup: Insert 3 transactions with same blockNumber but different states (0, 1, 2)
   - Execute: `transactionsForObserverMonitoring(0, 10)`
   - Verify: Returns only transaction with state=0

3. **TransactionsForObserverMonitoring_LimitParameter_RespectsLimit**
   - Setup: Insert 10 transactions with blockNumbers 10-100 step 10 (all state=0)
   - Execute: `transactionsForObserverMonitoring(0, 3)`
   - Verify: Returns exactly 3 transactions with smallest blockNumbers (10, 20, 30)

4. **TransactionsForObserverMonitoring_AscendingOrder_OldestFirst**
   - Setup: Insert transactions with blockNumbers: 30, 10, 20 (inserted in this order)
   - Execute: `transactionsForObserverMonitoring(0, 10)`
   - Verify: Returns in sorted order: 10, 20, 30 (regardless of insertion order)

5. **TransactionsForObserverMonitoring_EmptyResult_NoMatchingTransactions**
   - Setup: Insert transactions with blockNumbers all <= 100
   - Execute: `transactionsForObserverMonitoring(100, 10)`
   - Verify: Returns empty vector

6. **TransactionsForObserverMonitoring_BoundaryCondition_ExactBlockNumber**
   - Setup: Insert transaction with blockNumber = 100
   - Execute: `transactionsForObserverMonitoring(100, 10)`
   - Verify: Returns empty (> comparison, not >=)

7. **TransactionsForObserverMonitoring_ZeroLimit_ReturnsEmpty**
   - Setup: Insert 5 transactions
   - Execute: `transactionsForObserverMonitoring(0, 0)`
   - Verify: Returns empty vector

8. **TransactionsForObserverMonitoring_LargeDataset_ReturnsExpectedRange**
   - Setup: Insert 200 transactions with sequential blockNumbers
   - Execute: `transactionsForObserverMonitoring(50, 100)`
   - Verify:
     - Returns correct 100 transactions (blockNumbers 51-150)

### PostgreSQL Integration Tests
**File:** `tests/storage/integration/postgresql/PaymentTransactionsHandlerPostgreSQLIntegrationTest.cpp`

Add 8 test cases in new test fixture or extend existing `PaymentTransactionsHandlerPostgreSQLIntegrationTest`:

Same 8 test cases as SQLite (adapted for PostgreSQL integration test environment):
1. BasicRetrieval_ReturnsMatchingTransactions
2. StateFiltering_OnlyReturnsUncertainState
3. LimitParameter_RespectsLimit
4. AscendingOrder_OldestFirst
5. EmptyResult_NoMatchingTransactions
6. BoundaryCondition_ExactBlockNumber
7. ZeroLimit_ReturnsEmpty
8. LargeDataset_ReturnsExpectedRange

## Test Infrastructure Requirements
- Use existing test fixtures and setup patterns from SQLite and PostgreSQL test files
- SQLite tests use in-memory database with `payment_keys` table setup
- PostgreSQL tests use `DatabaseTestHelper` for connection management
- Follow existing test naming conventions: `MethodName_Scenario_ExpectedBehavior`
- Use Google Test assertions: `EXPECT_EQ`, `EXPECT_TRUE`, `ASSERT_EQ`, etc.

## Definition of Done
- [ ] 8 SQLite unit tests implemented in `PaymentTransactionsHandlerSQLiteTest.cpp`
- [ ] 8 PostgreSQL integration tests implemented in `PaymentTransactionsHandlerPostgreSQLIntegrationTest.cpp`
- [ ] All 16 tests compile without errors or warnings
- [ ] All 16 tests pass when executed
- [ ] Tests follow existing patterns in respective test files
- [ ] Test names clearly describe scenario and expected behavior
- [ ] Tests can be run independently and in any order
- [ ] Test setup and teardown properly managed (no resource leaks)

# Implementation Plan

## Step 1: Analyze Existing Test Patterns
1. Read `tests/unit/sqlite/PaymentTransactionsHandlerSQLiteTest.cpp` to understand:
   - Test fixture setup (database creation, payment_keys table)
   - Helper methods (`createTestUUID`, `createTestBlockNumber`)
   - Assertion patterns
   - Existing tests for similar methods (e.g., `transactionsWithUncertainObservingState`)

2. Read `tests/storage/integration/postgresql/PaymentTransactionsHandlerPostgreSQLIntegrationTest.cpp` to understand:
   - Test fixture setup (connection, table cleanup)
   - `DatabaseTestHelper` usage
   - Helper methods (`createTestTransactionUUID`)
   - Existing tests for similar methods

**Reference locations:**
- SQLite test fixture: lines 24-145
- PostgreSQL test fixture: lines 15-173

## Step 2: Implement SQLite Tests
**File:** `tests/unit/sqlite/PaymentTransactionsHandlerSQLiteTest.cpp`

For each of 8 tests:
1. Create test case using `TEST_F(PaymentTransactionsHandlerSQLiteTest, <TestName>)`
2. Setup: Insert test data using `handler->saveRecord(uuid, blockNumber)`
3. For state variations: Use direct SQL or `handler->updateTransactionState()` to set different states
4. Execute: Call `handler->transactionsForObserverMonitoring(minBlockNumber, limit)`
5. Verify: Assert expected results using Google Test macros
6. For performance test: Use `std::chrono` to measure execution time

**Test data patterns:**
- Use `createTestUUID()` for unique UUIDs
- Use `createTestBlockNumber(value)` for block numbers
- For large dataset test: Use loop to insert 1000 transactions

**Assertion patterns:**
```cpp
auto result = handler->transactionsForObserverMonitoring(100, 10);
EXPECT_EQ(result.size(), 3);
EXPECT_EQ(result[0].second, 150);  // Verify block number
EXPECT_EQ(result[1].second, 200);
EXPECT_EQ(result[2].second, 250);
```

## Step 3: Implement PostgreSQL Tests
**File:** `tests/storage/integration/postgresql/PaymentTransactionsHandlerPostgreSQLIntegrationTest.cpp`

For each of 8 tests:
1. Create test case using `TEST_F(PaymentTransactionsHandlerPostgreSQLIntegrationTest, <TestName>)`
2. Setup: Insert test data using `mHandler->saveRecord(uuid, blockNumber)`
3. For state variations: Use `mHandler->updateTransactionState()` to set different states
4. Execute: Call `mHandler->transactionsForObserverMonitoring(minBlockNumber, limit)`
5. Verify: Assert expected results using Google Test macros
6. For performance test: Use `std::chrono` to measure execution time

**Differences from SQLite:**
- Use `mHandler` instead of `handler`
- Use `createTestTransactionUUID(testData)` for UUIDs
- Table cleanup handled by test fixture teardown
- Connection managed by `DatabaseTestHelper`

## Step 4: Build and Execute Tests

**Build tests:**
```bash
cmake --build build-tests
```

**Run SQLite tests:**
```bash
./build-tests/bin/unit_tests --gtest_filter=*PaymentTransactionsHandlerSQLite*transactionsForObserverMonitoring*
```

**Run PostgreSQL tests:**
```bash
./build-tests/bin/postgresql_integration_tests --gtest_filter=*PaymentTransactionsHandlerPostgreSQL*transactionsForObserverMonitoring*
```

**Verify:**
- All 16 tests pass (100% pass rate)
- Large dataset query returns expected subset (no timing requirement)
- No memory leaks (run with valgrind if needed)

## Step 5: Documentation
Add brief comments above each test explaining:
- Test purpose
- Setup scenario
- Expected behavior

# Test Plan

This task IS the test implementation. Tests validate:

**Correctness:**
- Filtering by block number (> comparison)
- Filtering by state (only state=0)
- Result ordering (ascending by block number)
- Limit parameter enforcement
- Boundary conditions

**Edge Cases:**
- Empty results (no matching transactions)
- Zero limit
- Exact block number boundary
- Large datasets

# Verification and Validation

**Complexity Level:** Moderate (testing only, but 16 comprehensive tests)

## Architecture integrity
- [ ] Tests follow existing test patterns in respective files
- [ ] Test fixtures properly extend base test classes
- [ ] No modifications to production code
- [ ] Tests isolated and independent

## Security
- [ ] Test data does not contain sensitive information
- [ ] Database credentials use test environment only (PostgreSQL)
- [ ] No security vulnerabilities introduced by test code

## Performance
- [ ] Large dataset test executes successfully and returns expected subset
- [ ] Tests complete in reasonable time (< 5 seconds total)
- [ ] No resource exhaustion from test data

## Scalability
- [ ] Large dataset test (1000 transactions) validates scalability
- [ ] Tests don't leave residual data affecting future tests

## Reliability
- [ ] Tests can run independently and in any order
- [ ] Proper setup/teardown ensures clean state
- [ ] No flaky tests (deterministic behavior)
- [ ] PostgreSQL connection errors handled gracefully

## Maintainability
- [ ] Clear test names describe scenarios
- [ ] Test code follows project style
- [ ] Easy to add new test cases in future
- [ ] Comments explain non-obvious test logic

## Cost
- [ ] SQLite tests use in-memory database (no storage cost)
- [ ] PostgreSQL tests use shared test database
- [ ] Test execution time minimal (< 5 seconds)

## Compliance
- [ ] Tests follow Google Test conventions
- [ ] Consistent with existing test suite patterns
- [ ] No deviations from testing standards

# Restrictions
- Commit changes only after all 16 tests pass successfully
- Do not modify production code (PaymentTransactionsHandler implementations)
- Do not create new test fixtures if existing ones are sufficient
- Follow existing test patterns strictly
- SQLite tests must be unit tests (isolated, no external dependencies)
- PostgreSQL tests must be integration tests (real database connection)
