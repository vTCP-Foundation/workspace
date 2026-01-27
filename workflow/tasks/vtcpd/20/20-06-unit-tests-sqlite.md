# 20-06 - Unit Tests for SQLite Handlers

# Links
- [PRD-20: Cleanup Historical Crypto Data](../../prd/vtcpd/20-cleanup-historical-crypto-data.md)
- [Task 20-01: Handler Interfaces and SQLite Implementation](20-01-handler-interfaces-sqlite.md)

# Description
Create unit tests for the new `deleteRecordsByAuditNumber` method in SQLite implementations of `IncomingPaymentReceiptHandler` and `OutgoingPaymentReceiptHandler`.

This task ensures the new handler methods work correctly for the SQLite storage backend, which is used in development and testing environments.

# Requirements and DOD

## Requirements
1. Add unit tests for `IncomingPaymentReceiptHandler::deleteRecordsByAuditNumber` in SQLite
2. Add unit tests for `OutgoingPaymentReceiptHandler::deleteRecordsByAuditNumber` in SQLite
3. Tests must cover:
   - Deletion of matching records (correct TrustLineID and audit_number)
   - Non-deletion of records with different audit_number
   - Non-deletion of records with different TrustLineID
   - Handling of empty table (no records to delete)

## Definition of Done
- [ ] Tests added to `tests/unit/sqlite/IncomingPaymentReceiptHandlerSQLiteTest.cpp`
- [ ] Tests added to `tests/unit/sqlite/OutgoingPaymentReceiptHandlerSQLiteTest.cpp`
- [ ] All new tests pass
- [ ] Tests follow existing test patterns in the codebase

# Implementation Plan

## Step 1: Add Tests for IncomingPaymentReceiptHandler
**File**: `tests/unit/sqlite/IncomingPaymentReceiptHandlerSQLiteTest.cpp`

Add the following test cases:

### Test 1: deleteRecordsByAuditNumber_deletesMatchingRecords
```cpp
TEST_F(IncomingPaymentReceiptHandlerSQLiteTest, deleteRecordsByAuditNumber_deletesMatchingRecords)
{
    // Setup:
    // 1. Create trust line
    // 2. Save multiple receipts with audit_number = 5
    // 3. Call deleteRecordsByAuditNumber(trustLineID, 5)
    // Verify:
    // - All receipts with audit_number = 5 are deleted
    // - receiptsByAuditNumber returns empty
}
```

### Test 2: deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers
```cpp
TEST_F(IncomingPaymentReceiptHandlerSQLiteTest, deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers)
{
    // Setup:
    // 1. Create trust line
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
TEST_F(IncomingPaymentReceiptHandlerSQLiteTest, deleteRecordsByAuditNumber_doesNotDeleteOtherTrustLines)
{
    // Setup:
    // 1. Create two trust lines (TL1, TL2)
    // 2. Save receipts with audit_number = 5 on TL1
    // 3. Save receipts with audit_number = 5 on TL2
    // 4. Call deleteRecordsByAuditNumber(TL1, 5)
    // Verify:
    // - Receipts on TL1 with audit_number = 5 are deleted
    // - Receipts on TL2 with audit_number = 5 still exist
}
```

### Test 4: deleteRecordsByAuditNumber_handlesEmptyTable
```cpp
TEST_F(IncomingPaymentReceiptHandlerSQLiteTest, deleteRecordsByAuditNumber_handlesEmptyTable)
{
    // Setup:
    // 1. Create trust line (no receipts)
    // 2. Call deleteRecordsByAuditNumber(trustLineID, 5)
    // Verify:
    // - No exception thrown
    // - Method completes successfully
}
```

## Step 2: Add Tests for OutgoingPaymentReceiptHandler
**File**: `tests/unit/sqlite/OutgoingPaymentReceiptHandlerSQLiteTest.cpp`

Add the same four test cases with identical logic:

### Test 1: deleteRecordsByAuditNumber_deletesMatchingRecords
### Test 2: deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers
### Test 3: deleteRecordsByAuditNumber_doesNotDeleteOtherTrustLines
### Test 4: deleteRecordsByAuditNumber_handlesEmptyTable

## Reference: Existing Test Patterns
Use existing tests in the same files as reference for:
- Test fixture setup (`IncomingPaymentReceiptHandlerSQLiteTest`, `OutgoingPaymentReceiptHandlerSQLiteTest`)
- Helper methods for creating test data
- Assertion patterns
- Transaction handling

Look at existing `deleteRecords` tests for similar patterns.

# Test Plan

**Complexity**: Simple

This task IS the test implementation. Validation:
- All 8 new tests pass (4 for incoming, 4 for outgoing)
- Tests cover all specified scenarios
- No regressions in existing tests

# Verification and Validation

## Architecture integrity
- Tests follow existing test patterns in the codebase
- Tests use established test fixtures

## Security
- N/A for test code

## Performance
- Tests should complete quickly (unit tests with in-memory SQLite)

## Scalability
- N/A for test code

## Reliability
- Tests are deterministic and repeatable
- Proper setup/teardown to avoid test interference

## Maintainability
- Clear test naming convention
- Each test covers one specific scenario
- Tests serve as documentation for expected behavior

## Cost
- N/A

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- This task depends on Task 20-01 being completed (methods must exist to test)
- Do not modify implementation code in this task
