# 13-01 - Database Query Method Implementation

# Links
- [PRD 13: Observer Successful Transactions Monitoring](../../../prd/vtcpd/13-observer-successful-transactions-monitoring.md)

# Description
Implement a new database query method `transactionsForObserverMonitoring` in the PaymentTransactionsHandler interface and its concrete implementations (SQLite and PostgreSQL). This method retrieves payment transactions that are still within their claiming period and in uncertain state, sorted by block number for efficient monitoring by ObservingHandler.

This is foundational infrastructure required for the observer successful transactions monitoring feature, enabling the node to identify which of its successful transactions may need assistance from observers.

# Requirements and DOD

## Functional Requirements
1. **Interface Method Addition**
   - Add pure virtual method to `PaymentTransactionsHandler` interface (`src/core/io/storage/interfaces/PaymentTransactionsHandler.h`)
   - Method signature: `virtual vector<pair<TransactionUUID, BlockNumber>> transactionsForObserverMonitoring(BlockNumber minBlockNumber, uint32_t limit) = 0;`
   - Method returns vector of (TransactionUUID, BlockNumber) pairs

2. **SQLite Implementation**
   - Implement method in `PaymentTransactionsHandlerSQLite` (`src/core/io/storage/sqlite/PaymentTransactionsHandlerSQLite.h` and `.cpp`)
   - SQL query filters: `WHERE maximal_claiming_block_number > ? AND observing_state = 0`
   - SQL query ordering: `ORDER BY maximal_claiming_block_number ASC`
   - SQL query limit: `LIMIT ?`
   - Proper parameter binding for `minBlockNumber` (BLOB) and `limit` (INTEGER)
   - Use `SQLiteStatementRAII` for statement management
   - Proper error handling with `IOError` exceptions

3. **PostgreSQL Implementation**
   - Implement method in `PaymentTransactionsHandlerPostgreSQL` (`src/core/io/storage/postgresql/PaymentTransactionsHandlerPostgreSQL.h` and `.cpp`)
   - SQL query filters: `WHERE maximal_claiming_block_number > $1 AND observing_state = 0`
   - SQL query ordering: `ORDER BY maximal_claiming_block_number ASC`
   - SQL query limit: `LIMIT $2`
   - Proper parameter binding: binary format for `minBlockNumber`, text format for `limit`
   - Use `PQexecParams` for parameterized query execution
   - Use `checkTuples` helper for result validation
   - Proper error handling with `IOError` exceptions

4. **Query Semantics**
   - Transactions with `observing_state = 0` are treated as "successful transactions" (no additional success validation required)
   - Filter returns transactions where claiming window has not expired: `maximal_claiming_block_number > minBlockNumber`
   - Results sorted by `maximal_claiming_block_number ASC` to process oldest transactions first
   - `limit` parameter restricts result count for pagination/performance

5. **Debug Logging**
   - Add debug logging under `#ifdef STORAGE_HANDLER_DEBUG_LOG` for both implementations
   - Log: number of transactions retrieved

## Definition of Done
- [ ] Method signature added to `PaymentTransactionsHandler` interface
- [ ] SQLite implementation complete with proper SQL query, parameter binding, and error handling
- [ ] PostgreSQL implementation complete with proper SQL query, parameter binding, and error handling
- [ ] Debug logging added to both implementations
- [ ] Code compiles without errors or warnings
- [ ] All implementations follow existing code style and patterns in respective handler classes
- [ ] Method documentation comments added explaining parameters and return value
- [ ] No violations of existing database handler patterns

# Implementation Plan

## Step 1: Interface Method Addition
**File:** `src/core/io/storage/interfaces/PaymentTransactionsHandler.h`

1. Add method declaration after existing virtual methods:
```cpp
/**
 * Retrieves transactions for observer monitoring.
 * Returns transactions where maximal_claiming_block_number > minBlockNumber
 * and observing_state = 0, sorted by maximal_claiming_block_number ascending.
 *
 * @param minBlockNumber Minimum block number (transactions with block > this are returned)
 * @param limit Maximum number of transactions to return
 * @return Vector of (TransactionUUID, BlockNumber) pairs sorted by block number ascending
 */
virtual vector<pair<TransactionUUID, BlockNumber>> transactionsForObserverMonitoring(
    BlockNumber minBlockNumber,
    uint32_t limit) = 0;
```

## Step 2: SQLite Implementation
**Files:**
- `src/core/io/storage/sqlite/PaymentTransactionsHandlerSQLite.h` (declaration)
- `src/core/io/storage/sqlite/PaymentTransactionsHandlerSQLite.cpp` (implementation)

1. Add method declaration in header file (public section, after existing methods)
2. Implement method in .cpp file:
   - Prepare SQL query string with placeholders
   - Create `SQLiteStatementRAII` for statement management
   - Bind `minBlockNumber` as BLOB (parameter index 1)
   - Bind `limit` as INTEGER (parameter index 2)
   - Execute query with `sqlite3_step` in loop
   - For each row:
     - Extract UUID from column 0 (BLOB)
     - Extract BlockNumber from column 1 (BLOB)
     - Add pair to result vector
   - Add debug logging for result count
   - Return result vector
3. Handle errors: throw `IOError` with descriptive message on binding or execution failure

**Reference pattern:** See existing `transactionsWithUncertainObservingState()` method in same file (lines 162-192) for similar implementation pattern.

## Step 3: PostgreSQL Implementation
**Files:**
- `src/core/io/storage/postgresql/PaymentTransactionsHandlerPostgreSQL.h` (declaration)
- `src/core/io/storage/postgresql/PaymentTransactionsHandlerPostgreSQL.cpp` (implementation)

1. Add method declaration in header file (public section, after existing methods)
2. Implement method in .cpp file:
   - Prepare SQL query string with $1, $2 placeholders
   - Set up parameter arrays: `params[2]`, `lengths[2]`, `formats[2]`
   - Bind `minBlockNumber` as BYTEA (binary format, index 0)
   - Bind `limit` as text (index 1)
   - Execute with `PQexecParams`
   - Call `checkTuples` to validate result
   - Iterate through rows with `PQntuples` and `PQgetvalue`
   - For each row:
     - Extract UUID bytes from column 0
     - Extract BlockNumber bytes from column 1
     - Add pair to result vector
   - Clear result with `PQclear`
   - Add debug logging for result count
   - Return result vector
3. Handle errors: `checkTuples` throws `IOError` on failure

**Reference pattern:** See existing `transactionsWithUncertainObservingState()` method in same file (lines 96-112) for similar implementation pattern.

## Step 4: Verification
1. Build project: `cmake --build build`
2. Verify compilation succeeds with no errors or warnings
3. Verify method signatures match across interface and implementations
4. Verify SQL queries are syntactically correct
5. Ready for unit testing (Task 13-2)

# Test Plan

Tests are implemented in separate task (13-2) following project separation policy.

**Test Coverage (to be implemented in 13-2):**
- SQLite unit tests (8 tests):
  1. Basic retrieval with matching transactions
  2. State filtering (only state=0 returned)
  3. Limit parameter respected
  4. Ascending order verification
  5. Empty result when no matches
  6. Boundary condition (exact blockNumber excluded)
  7. Zero limit returns empty
  8. Performance test (1000 transactions < 100ms)

- PostgreSQL integration tests (8 tests): Same coverage as SQLite

# Verification and Validation

**Complexity Level:** Moderate (database operations, two implementations)

## Architecture integrity
- [ ] Follows existing PaymentTransactionsHandler interface pattern
- [ ] Consistent with existing query methods in both SQLite and PostgreSQL handlers
- [ ] Uses established RAII patterns for resource management
- [ ] No architectural changes to database layer

## Security
- [ ] Uses parameterized queries (no SQL injection risk)
- [ ] Proper parameter binding prevents injection attacks
- [ ] No sensitive data exposure in error messages
- [ ] Database connection security unchanged

## Performance
- [ ] Query filtered at database level (efficient)
- [ ] Uses existing indexes on `maximal_claiming_block_number` field (if available)
- [ ] Limit parameter prevents unbounded result sets
- [ ] ORDER BY on indexed column (efficient)
- [ ] Target: < 100ms execution for typical datasets

## Scalability
- [ ] Limit parameter enables pagination
- [ ] Query complexity O(log n) with proper indexing
- [ ] Memory usage proportional to limit parameter (bounded)
- [ ] No table scans for large datasets

## Reliability
- [ ] RAII ensures statement cleanup in SQLite
- [ ] PQclear ensures result cleanup in PostgreSQL
- [ ] IOError exceptions provide clear error diagnostics
- [ ] No memory leaks in error paths

## Maintainability
- [ ] Clear method documentation
- [ ] Follows existing code patterns in handlers
- [ ] Debug logging aids troubleshooting
- [ ] Consistent naming with similar methods

## Cost
- [ ] No additional database infrastructure required
- [ ] Uses existing tables and connections
- [ ] Minimal CPU/memory overhead per query
- [ ] Query cost bounded by limit parameter

## Compliance
- [ ] Follows project coding standards
- [ ] Adheres to database handler interface contract
- [ ] Consistent with existing transaction query patterns
- [ ] No changes to data privacy or retention policies

# Restrictions
- Commit changes only after code compiles successfully
- Do not implement tests in this task (separate task 13-2)
- Do not modify existing PaymentTransactionsHandler methods
- Do not add new database tables or indexes
- Stay strictly within scope: only add new query method to existing handlers
