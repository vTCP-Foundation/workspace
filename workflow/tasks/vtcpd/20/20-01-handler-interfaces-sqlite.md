# 20-01 - Handler Interfaces and SQLite Implementation

# Links
- [PRD-20: Cleanup Historical Crypto Data](../../prd/vtcpd/20-cleanup-historical-crypto-data.md)

# Description
Add new method `deleteRecordsByAuditNumber(TrustLineID, AuditNumber)` to the `IncomingPaymentReceiptHandler` and `OutgoingPaymentReceiptHandler` interfaces, and implement them for SQLite storage backend.

This task is part of PRD-20 which implements automatic cleanup of obsolete historical crypto data (audits, debt receipts) from the database. The new methods enable batch deletion of receipts belonging to a specific audit number, which is essential for the cleanup transaction.

# Requirements and DOD

## Requirements
1. Add virtual method `deleteRecordsByAuditNumber(const TrustLineID trustLineID, const AuditNumber auditNumber)` to `IncomingPaymentReceiptHandler` interface
2. Add virtual method `deleteRecordsByAuditNumber(const TrustLineID trustLineID, const AuditNumber auditNumber)` to `OutgoingPaymentReceiptHandler` interface
3. Implement `deleteRecordsByAuditNumber` in SQLite `IncomingPaymentReceiptHandler`
4. Implement `deleteRecordsByAuditNumber` in SQLite `OutgoingPaymentReceiptHandler`
5. Both implementations must delete all records matching the given TrustLineID AND audit_number
6. Methods must throw `IOError` on database errors

## Definition of Done
- [ ] Interface method declared in `src/core/io/storage/interfaces/IncomingPaymentReceiptHandler.h`
- [ ] Interface method declared in `src/core/io/storage/interfaces/OutgoingPaymentReceiptHandler.h`
- [ ] SQLite implementation in `src/core/io/storage/sqlite/IncomingPaymentReceiptHandler.h/.cpp`
- [ ] SQLite implementation in `src/core/io/storage/sqlite/OutgoingPaymentReceiptHandler.h/.cpp`
- [ ] Code compiles without errors
- [ ] Methods follow existing code patterns and conventions

# Implementation Plan

## Step 1: Update IncomingPaymentReceiptHandler Interface
**File**: `src/core/io/storage/interfaces/IncomingPaymentReceiptHandler.h`

Add new virtual method declaration:
```cpp
virtual void deleteRecordsByAuditNumber(
    const TrustLineID trustLineID,
    const AuditNumber auditNumber) = 0;
```

## Step 2: Update OutgoingPaymentReceiptHandler Interface
**File**: `src/core/io/storage/interfaces/OutgoingPaymentReceiptHandler.h`

Add new virtual method declaration:
```cpp
virtual void deleteRecordsByAuditNumber(
    const TrustLineID trustLineID,
    const AuditNumber auditNumber) = 0;
```

## Step 3: Implement SQLite IncomingPaymentReceiptHandler
**Files**:
- `src/core/io/storage/sqlite/IncomingPaymentReceiptHandler.h`
- `src/core/io/storage/sqlite/IncomingPaymentReceiptHandler.cpp`

Add method declaration to header and implement in cpp:
```cpp
void IncomingPaymentReceiptHandler::deleteRecordsByAuditNumber(
    const TrustLineID trustLineID,
    const AuditNumber auditNumber)
{
    // SQL: DELETE FROM incoming_payment_receipts
    //      WHERE trust_line_id = ? AND audit_number = ?
    // Follow existing patterns from deleteRecords methods
}
```

## Step 4: Implement SQLite OutgoingPaymentReceiptHandler
**Files**:
- `src/core/io/storage/sqlite/OutgoingPaymentReceiptHandler.h`
- `src/core/io/storage/sqlite/OutgoingPaymentReceiptHandler.cpp`

Add method declaration to header and implement in cpp:
```cpp
void OutgoingPaymentReceiptHandler::deleteRecordsByAuditNumber(
    const TrustLineID trustLineID,
    const AuditNumber auditNumber)
{
    // SQL: DELETE FROM outgoing_payment_receipts
    //      WHERE trust_line_id = ? AND audit_number = ?
    // Follow existing patterns from deleteRecords methods
}
```

## Reference: Existing Delete Methods
Use existing `deleteRecords(const TrustLineID trustLineID)` as a pattern for:
- Statement preparation
- Parameter binding
- Error handling
- Logging

# Test Plan

**Complexity**: Simple

Testing will be covered by Task 20-06 (Unit Tests for SQLite Handlers).

**Validation for this task**:
- Code compiles successfully
- Methods can be called without runtime errors (basic smoke test)
- SQL queries are syntactically correct

# Verification and Validation

## Architecture integrity
- Methods follow existing handler interface patterns
- No changes to existing method signatures
- Consistent with storage abstraction layer design

## Security
- No new security concerns; uses parameterized queries (no SQL injection)
- Same security model as existing delete methods

## Performance
- Single DELETE query per method call
- Uses indexed columns (trust_line_id, audit_number)

## Scalability
- N/A for this simple task

## Reliability
- Proper error handling with IOError exceptions
- Follows existing error handling patterns

## Maintainability
- Consistent naming with existing methods
- Clear, simple implementation

## Cost
- N/A

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify PostgreSQL implementations in this task (covered by Task 20-02)
