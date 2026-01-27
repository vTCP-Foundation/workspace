# 20-07 - Integration Tests for PostgreSQL Handlers

# Links
- [PRD-20: Cleanup Historical Crypto Data](../../prd/vtcpd/20-cleanup-historical-crypto-data.md)
- [Task 20-02: Handler PostgreSQL Implementation](20-02-handler-postgresql.md)

# Description
Create integration tests for the new `deleteRecordsByAuditNumber` method in PostgreSQL implementations of `IncomingPaymentReceiptHandler` and `OutgoingPaymentReceiptHandler`.

This task ensures the new handler methods work correctly for the PostgreSQL storage backend, which is used in production environments.

# Requirements and DOD

## Requirements
1. Add integration tests for `IncomingPaymentReceiptHandler::deleteRecordsByAuditNumber` in PostgreSQL
2. Add integration tests for `OutgoingPaymentReceiptHandler::deleteRecordsByAuditNumber` in PostgreSQL
3. Tests must cover:
   - Deletion of matching records (correct TrustLineID and audit_number)
   - Non-deletion of records with different audit_number
   - Non-deletion of records with different TrustLineID

## Definition of Done
- [ ] Tests added to `tests/storage/integration/postgresql/IncomingPaymentReceiptHandlerPostgreSQLIntegrationTest.cpp`
- [ ] Tests added to `tests/storage/integration/postgresql/OutgoingPaymentReceiptHandlerPostgreSQLIntegrationTest.cpp`
- [ ] All new tests pass against PostgreSQL database
- [ ] Tests follow existing integration test patterns in the codebase

# Implementation Plan

## Step 1: Add Tests for IncomingPaymentReceiptHandler
**File**: `tests/storage/integration/postgresql/IncomingPaymentReceiptHandlerPostgreSQLIntegrationTest.cpp`

Add the following test cases:

### Test 1: deleteRecordsByAuditNumber_deletesMatchingRecords
```cpp
TEST_F(IncomingPaymentReceiptHandlerPostgreSQLIntegrationTest, deleteRecordsByAuditNumber_deletesMatchingRecords)
{
    // Setup:
    // 1. Create trust line in PostgreSQL
    // 2. Save multiple receipts with audit_number = 5
    // 3. Call deleteRecordsByAuditNumber(trustLineID, 5)
    // Verify:
    // - All receipts with audit_number = 5 are deleted
    // - receiptsByAuditNumber returns empty
}
```

### Test 2: deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers
```cpp
TEST_F(IncomingPaymentReceiptHandlerPostgreSQLIntegrationTest, deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers)
{
    // Setup:
    // 1. Create trust line in PostgreSQL
    // 2. Save receipts with audit_number = 5
    // 3. Save receipts with audit_number = 6
    // 4. Call deleteRecordsByAuditNumber(trustLineID, 5)
    // Verify:
    // - Receipts with audit_number = 5 are deleted
    // - Receipts with audit_number = 6 still exist
}
```

### Test 3: deleteRecordsByAuditNumber_doesNotDeleteOtherTrustLines
```cpp
TEST_F(IncomingPaymentReceiptHandlerPostgreSQLIntegrationTest, deleteRecordsByAuditNumber_doesNotDeleteOtherTrustLines)
{
    // Setup:
    // 1. Create two trust lines (TL1, TL2) in PostgreSQL
    // 2. Save receipts with audit_number = 5 on TL1
    // 3. Save receipts with audit_number = 5 on TL2
    // 4. Call deleteRecordsByAuditNumber(TL1, 5)
    // Verify:
    // - Receipts on TL1 with audit_number = 5 are deleted
    // - Receipts on TL2 with audit_number = 5 still exist
}
```

## Step 2: Add Tests for OutgoingPaymentReceiptHandler
**File**: `tests/storage/integration/postgresql/OutgoingPaymentReceiptHandlerPostgreSQLIntegrationTest.cpp`

Add the same three test cases with identical logic:

### Test 1: deleteRecordsByAuditNumber_deletesMatchingRecords
### Test 2: deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers
### Test 3: deleteRecordsByAuditNumber_doesNotDeleteOtherTrustLines

## Reference: Existing Integration Test Patterns
Use existing tests in the same files as reference for:
- Test fixture setup with PostgreSQL connection
- Helper methods for creating test data
- Database cleanup between tests
- Assertion patterns
- Transaction handling with pqxx

Look at existing tests in `OutgoingPaymentReceiptHandlerPostgreSQLIntegrationTest.cpp` and `IncomingPaymentReceiptHandlerPostgreSQLIntegrationTest.cpp` for similar patterns.

## PostgreSQL Test Environment
Integration tests require:
- Running PostgreSQL instance
- Proper test database configuration
- Database cleanup/reset between test runs

# Test Plan

**Complexity**: Simple

This task IS the test implementation. Validation:
- All 6 new tests pass (3 for incoming, 3 for outgoing)
- Tests cover all specified scenarios
- No regressions in existing integration tests
- Tests run successfully against PostgreSQL database

# Verification and Validation

## Architecture integrity
- Tests follow existing integration test patterns in the codebase
- Tests use established PostgreSQL test fixtures
- Tests verify PostgreSQL-specific behavior (transactions, parameter binding)

## Security
- N/A for test code
- Tests use parameterized queries (verified indirectly)

## Performance
- Integration tests may take longer than unit tests
- Proper database cleanup ensures test isolation

## Scalability
- N/A for test code

## Reliability
- Tests are deterministic and repeatable
- Proper setup/teardown to avoid test interference
- Database state cleaned between tests

## Maintainability
- Clear test naming convention
- Each test covers one specific scenario
- Tests serve as documentation for expected behavior
- Consistent with SQLite unit tests from Task 20-06

## Cost
- N/A

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- This task depends on Task 20-02 being completed (PostgreSQL methods must exist to test)
- Do not modify implementation code in this task
- Requires PostgreSQL test environment to be available
