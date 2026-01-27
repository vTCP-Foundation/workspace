# 20-02 - Handler PostgreSQL Implementation

# Links
- [PRD-20: Cleanup Historical Crypto Data](../../prd/vtcpd/20-cleanup-historical-crypto-data.md)
- [Task 20-01: Handler Interfaces and SQLite Implementation](20-01-handler-interfaces-sqlite.md)

# Description
Implement the `deleteRecordsByAuditNumber(TrustLineID, AuditNumber)` method for PostgreSQL storage backend in both `IncomingPaymentReceiptHandler` and `OutgoingPaymentReceiptHandler`.

This task is part of PRD-20 which implements automatic cleanup of obsolete historical crypto data. The PostgreSQL implementations complement the SQLite implementations from Task 20-01, ensuring the cleanup functionality works across both supported database backends.

# Requirements and DOD

## Requirements
1. Implement `deleteRecordsByAuditNumber` in PostgreSQL `IncomingPaymentReceiptHandler`
2. Implement `deleteRecordsByAuditNumber` in PostgreSQL `OutgoingPaymentReceiptHandler`
3. Both implementations must delete all records matching the given TrustLineID AND audit_number
4. Methods must throw `IOError` on database errors
5. Implementation must follow PostgreSQL-specific patterns used in existing methods

## Definition of Done
- [ ] PostgreSQL implementation in `src/core/io/storage/postgresql/IncomingPaymentReceiptHandler.h/.cpp`
- [ ] PostgreSQL implementation in `src/core/io/storage/postgresql/OutgoingPaymentReceiptHandler.h/.cpp`
- [ ] Code compiles without errors
- [ ] Methods follow existing PostgreSQL code patterns and conventions

# Implementation Plan

## Step 1: Implement PostgreSQL IncomingPaymentReceiptHandler
**Files**:
- `src/core/io/storage/postgresql/IncomingPaymentReceiptHandler.h`
- `src/core/io/storage/postgresql/IncomingPaymentReceiptHandler.cpp`

Add method declaration to header and implement in cpp:
```cpp
void IncomingPaymentReceiptHandler::deleteRecordsByAuditNumber(
    const TrustLineID trustLineID,
    const AuditNumber auditNumber)
{
    // SQL: DELETE FROM incoming_payment_receipts
    //      WHERE trust_line_id = $1 AND audit_number = $2
    // Follow existing patterns from deleteRecords methods
    // Use pqxx transaction and parameterized queries
}
```

## Step 2: Implement PostgreSQL OutgoingPaymentReceiptHandler
**Files**:
- `src/core/io/storage/postgresql/OutgoingPaymentReceiptHandler.h`
- `src/core/io/storage/postgresql/OutgoingPaymentReceiptHandler.cpp`

Add method declaration to header and implement in cpp:
```cpp
void OutgoingPaymentReceiptHandler::deleteRecordsByAuditNumber(
    const TrustLineID trustLineID,
    const AuditNumber auditNumber)
{
    // SQL: DELETE FROM outgoing_payment_receipts
    //      WHERE trust_line_id = $1 AND audit_number = $2
    // Follow existing patterns from deleteRecords methods
    // Use pqxx transaction and parameterized queries
}
```

## Reference: Existing PostgreSQL Delete Methods
Use existing `deleteRecords(const TrustLineID trustLineID)` in PostgreSQL handlers as a pattern for:
- pqxx::work transaction usage
- Parameter binding with `$1`, `$2` placeholders
- Error handling and exception wrapping
- Logging patterns

# Test Plan

**Complexity**: Simple

Testing will be covered by Task 20-07 (Integration Tests for PostgreSQL Handlers).

**Validation for this task**:
- Code compiles successfully
- Methods follow PostgreSQL-specific patterns (pqxx usage)
- SQL queries use correct PostgreSQL parameter syntax ($1, $2)

# Verification and Validation

## Architecture integrity
- Consistent implementation with SQLite counterpart from Task 20-01
- Follows PostgreSQL handler patterns established in codebase
- No changes to existing method signatures

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
- Uses pqxx transaction for atomicity
- Follows existing error handling patterns

## Maintainability
- Consistent naming with SQLite implementation
- Clear, simple implementation following established patterns

## Cost
- N/A

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- This task depends on Task 20-01 being completed (interfaces must exist)
