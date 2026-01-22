# 18-02 - Receipt Handlers Extension and Zero Audit Number

# Links
- [PRD-18: Audit Mechanism Based on Finalized Transactions](../../prd/vtcpd/18-audit-mechanism-finalized-transactions.md)
- [Previous task: 18-01-audit-messages-extension](18-01-audit-messages-extension.md)

# Description

This task extends the receipt handler interfaces and implementations to support the new audit mechanism based on finalized transactions. It also modifies the receipt saving logic to use `auditNumber = 0` for new receipts, indicating they haven't been included in any audit yet.

The task covers two related changes:
1. **New receipt handler methods**: Query receipts with `auditNumber = 0` joined with finalized transactions (`PaymentObservingState::Committed`), and update audit numbers after successful audit
2. **Save receipts with auditNumber = 0**: Modify keychain and payment transaction code to save new receipts with `auditNumber = 0` instead of the current audit number

# Requirements and DOD

## Requirements

### Part A: Receipt Handler Extensions

1. **OutgoingPaymentReceiptHandler Interface** (`src/core/io/storage/interfaces/OutgoingPaymentReceiptHandler.h`)
   - Add method: `vector<OutgoingPaymentReceipt> getFinalizedReceiptsWithZeroAuditNumber(TrustLineID trustLineID)`
     - Returns receipts where `audit_number = 0` AND corresponding transaction has `observing_state = 1` (Committed)
   - Add method: `void updateAuditNumberByTransactionUUIDs(TrustLineID trustLineID, AuditNumber auditNumber, const vector<TransactionUUID>& transactionUUIDs)`
     - Updates `audit_number` for specified receipts after successful audit

2. **IncomingPaymentReceiptHandler Interface** (`src/core/io/storage/interfaces/IncomingPaymentReceiptHandler.h`)
   - Add method: `vector<IncomingPaymentReceipt> getFinalizedReceiptsWithZeroAuditNumber(TrustLineID trustLineID)`
   - Add method: `void updateAuditNumberByTransactionUUIDs(TrustLineID trustLineID, AuditNumber auditNumber, const vector<TransactionUUID>& transactionUUIDs)`

3. **SQLite Implementations** (`src/core/io/storage/sqlite/`)
   - Implement `getFinalizedReceiptsWithZeroAuditNumber` with SQL:
     ```sql
     SELECT r.*
     FROM outgoing_payment_receipts r
     JOIN payment_transactions pt ON r.transaction_uuid = pt.transaction_uuid
     WHERE r.trust_line_id = ?
       AND r.audit_number = 0
       AND pt.observing_state = 1
     ```
   - Implement `updateAuditNumberByTransactionUUIDs` with SQL:
     ```sql
     UPDATE outgoing_payment_receipts
     SET audit_number = ?
     WHERE trust_line_id = ?
       AND transaction_uuid IN (?, ?, ...)
     ```
   - Same for incoming receipts

4. **PostgreSQL Implementations** (`src/core/io/storage/postgresql/`)
   - Same implementations as SQLite with PostgreSQL syntax

### Part B: Save Receipts with auditNumber = 0

5. **Keychain Modifications** (`src/core/crypto/keychain.h/.cpp`)
   - Modify `saveOutgoingPaymentReceipt` calls to pass `auditNumber = 0`
   - Modify `saveIncomingPaymentReceipt` calls to pass `auditNumber = 0`

6. **Payment Transaction Modifications**
   - Verify all places where receipts are saved pass `auditNumber = 0`
   - Files in `src/core/transactions/transactions/regular/payments/`

## Definition of Done

- [ ] `OutgoingPaymentReceiptHandler` interface extended with new methods
- [ ] `IncomingPaymentReceiptHandler` interface extended with new methods
- [ ] SQLite implementation for `OutgoingPaymentReceiptHandler` new methods
- [ ] SQLite implementation for `IncomingPaymentReceiptHandler` new methods
- [ ] PostgreSQL implementation for `OutgoingPaymentReceiptHandler` new methods
- [ ] PostgreSQL implementation for `IncomingPaymentReceiptHandler` new methods
- [ ] SQL queries correctly JOIN with `payment_transactions` table
- [ ] Keychain methods modified to save receipts with `auditNumber = 0`
- [ ] All payment transaction receipt saving uses `auditNumber = 0`
- [ ] Code compiles without errors
- [ ] Existing receipt functionality not broken

# Implementation Plan

## Step 1: Analyze Existing Code

1. Read `src/core/io/storage/interfaces/OutgoingPaymentReceiptHandler.h`
2. Read `src/core/io/storage/interfaces/IncomingPaymentReceiptHandler.h`
3. Read SQLite implementations in `src/core/io/storage/sqlite/`
4. Read PostgreSQL implementations in `src/core/io/storage/postgresql/`
5. Read `src/core/io/storage/interfaces/PaymentTransactionsHandler.h` to understand `payment_transactions` table structure
6. Read `src/core/crypto/keychain.h/.cpp` to understand receipt saving flow
7. Search for all places where `saveOutgoingPaymentReceipt` and `saveIncomingPaymentReceipt` are called

## Step 2: Extend OutgoingPaymentReceiptHandler Interface

1. Add method declaration for `getFinalizedReceiptsWithZeroAuditNumber`
2. Add method declaration for `updateAuditNumberByTransactionUUIDs`
3. Define appropriate return types and parameters

## Step 3: Extend IncomingPaymentReceiptHandler Interface

1. Add method declaration for `getFinalizedReceiptsWithZeroAuditNumber`
2. Add method declaration for `updateAuditNumberByTransactionUUIDs`

## Step 4: Implement SQLite OutgoingPaymentReceiptHandler

1. Implement `getFinalizedReceiptsWithZeroAuditNumber`:
   - Prepare SQL statement with JOIN on `payment_transactions`
   - Bind trust line ID parameter
   - Execute and collect results
   - Return vector of receipts
2. Implement `updateAuditNumberByTransactionUUIDs`:
   - Build SQL UPDATE with IN clause for transaction UUIDs
   - Bind parameters
   - Execute update

## Step 5: Implement SQLite IncomingPaymentReceiptHandler

1. Same implementation pattern as outgoing

## Step 6: Implement PostgreSQL OutgoingPaymentReceiptHandler

1. Same implementation pattern with PostgreSQL syntax (parameter placeholders: $1, $2, etc.)

## Step 7: Implement PostgreSQL IncomingPaymentReceiptHandler

1. Same implementation pattern

## Step 8: Modify Keychain Receipt Saving

1. Locate `saveOutgoingPaymentReceipt` in `keychain.cpp`
2. Change the `auditNumber` parameter passed to handler to `0`
3. Locate `saveIncomingPaymentReceipt` in `keychain.cpp`
4. Change the `auditNumber` parameter passed to handler to `0`

## Step 9: Verify Payment Transactions

1. Search for all calls to receipt saving methods
2. Verify each passes `auditNumber = 0` or goes through keychain
3. Update any direct calls if necessary

## Step 10: Verification

1. Compile the project
2. Verify no regressions in existing functionality

## Files to Modify

| File | Changes |
|------|---------|
| `src/core/io/storage/interfaces/OutgoingPaymentReceiptHandler.h` | Add new method declarations |
| `src/core/io/storage/interfaces/IncomingPaymentReceiptHandler.h` | Add new method declarations |
| `src/core/io/storage/sqlite/OutgoingPaymentReceiptHandler.h` | Add method declarations |
| `src/core/io/storage/sqlite/OutgoingPaymentReceiptHandler.cpp` | Implement new methods |
| `src/core/io/storage/sqlite/IncomingPaymentReceiptHandler.h` | Add method declarations |
| `src/core/io/storage/sqlite/IncomingPaymentReceiptHandler.cpp` | Implement new methods |
| `src/core/io/storage/postgresql/OutgoingPaymentReceiptHandler.h` | Add method declarations |
| `src/core/io/storage/postgresql/OutgoingPaymentReceiptHandler.cpp` | Implement new methods |
| `src/core/io/storage/postgresql/IncomingPaymentReceiptHandler.h` | Add method declarations |
| `src/core/io/storage/postgresql/IncomingPaymentReceiptHandler.cpp` | Implement new methods |
| `src/core/crypto/keychain.h` | Update method signatures if needed |
| `src/core/crypto/keychain.cpp` | Pass auditNumber=0 when saving receipts |
| Payment transaction files | Verify auditNumber=0 is used |

# Test Plan

**Complexity Level:** Moderate

## Functional Validation
- Verify `getFinalizedReceiptsWithZeroAuditNumber` returns only receipts with `audit_number = 0`
- Verify `getFinalizedReceiptsWithZeroAuditNumber` returns only receipts with finalized transactions (`observing_state = 1`)
- Verify `getFinalizedReceiptsWithZeroAuditNumber` does not return receipts for non-existent transactions
- Verify `updateAuditNumberByTransactionUUIDs` correctly updates specified receipts
- Verify `updateAuditNumberByTransactionUUIDs` does not affect other receipts
- Verify new receipts are saved with `auditNumber = 0`

## Integration Validation
- Verify queries work correctly with both SQLite and PostgreSQL
- Verify JOIN with `payment_transactions` table is correct
- Verify existing receipt retrieval methods still work

## Error Handling Validation
- Verify behavior when no matching receipts found (should return empty vector)
- Verify behavior when transaction UUIDs list is empty

# Verification and Validation

## Architecture integrity
- Handler interfaces follow existing patterns
- SQL queries follow existing query patterns in the codebase
- Keychain modifications maintain existing API contract

## Security
- SQL queries use parameterized statements (no SQL injection)
- No sensitive data exposure

## Performance
- Queries should be efficient with proper indexes
- Consider adding index on `(trust_line_id, audit_number)` if not exists
- JOIN on `transaction_uuid` should use existing indexes

## Scalability
- Queries scale with number of receipts per trust line
- Batch update of audit numbers is more efficient than individual updates

## Reliability
- All database operations use proper error handling
- Transactions ensure atomicity where needed

## Maintainability
- Code follows existing patterns in handler implementations
- Clear method names indicate purpose
- SQL queries are readable and documented

## Cost
- N/A (no infrastructure changes)

## Compliance
- N/A (internal data layer change)

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify audit transaction logic in this task (handled in subsequent tasks)
- Do not modify TrustLinesManager in this task (separate task)
- Ensure backward compatibility with existing receipt data (receipts with non-zero audit numbers should still be accessible)

# Implementation Notes
- Task ID: 18-02
- Updated files: src/core/common/Types.h, src/core/io/storage/interfaces/OutgoingPaymentReceiptHandler.h, src/core/io/storage/interfaces/IncomingPaymentReceiptHandler.h, src/core/io/storage/interfaces/PaymentTransactionsHandler.h, src/core/io/storage/sqlite/StorageHandlerSQLite.h, src/core/io/storage/postgresql/StorageHandlerPostgreSQL.h, src/core/io/storage/sqlite/OutgoingPaymentReceiptHandlerSQLite.h, src/core/io/storage/sqlite/OutgoingPaymentReceiptHandlerSQLite.cpp, src/core/io/storage/sqlite/IncomingPaymentReceiptHandlerSQLite.h, src/core/io/storage/sqlite/IncomingPaymentReceiptHandlerSQLite.cpp, src/core/io/storage/postgresql/OutgoingPaymentReceiptHandlerPostgreSQL.h, src/core/io/storage/postgresql/OutgoingPaymentReceiptHandlerPostgreSQL.cpp, src/core/io/storage/postgresql/IncomingPaymentReceiptHandlerPostgreSQL.h, src/core/io/storage/postgresql/IncomingPaymentReceiptHandlerPostgreSQL.cpp, src/core/crypto/keychain.h, src/core/crypto/keychain.cpp, src/core/transactions/transactions/regular/payments/ReceiverExchangePaymentTransaction.cpp, src/core/transactions/transactions/regular/payments/CycleCloserInitiatorTransaction.cpp, src/core/transactions/transactions/regular/payments/CoordinatorPaymentTransaction.cpp, src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.cpp, src/core/transactions/transactions/regular/payments/ReceiverPaymentTransaction.cpp, src/core/transactions/transactions/regular/payments/IntermediateNodePaymentTransaction.cpp, src/core/transactions/transactions/regular/payments/CycleCloserIntermediateNodeTransaction.cpp, src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp
