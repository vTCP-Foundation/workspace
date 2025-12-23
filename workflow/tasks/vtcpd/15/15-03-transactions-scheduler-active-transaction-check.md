# 15-03 - TransactionsScheduler Active Transaction Check

# Links
- [PRD](../../prd/vtcpd/15-completed-payments-observer-monitoring.md)

# Description
Add a method to `TransactionsScheduler` to check if a transaction of a specific type is currently active or scheduled. This is needed to prevent launching multiple instances of `CompletedPaymentsObserverMonitoringTransaction` simultaneously.

When the delayed task timer fires, `TransactionsManager` should first check if a monitoring transaction is already running. If so, it should skip creating a new one and wait for the next timer cycle.

# Requirements and DOD

## Functional Requirements
1. Add method to `TransactionsScheduler` that checks for active/scheduled transaction by type
2. Method signature: `bool hasActiveTransactionOfType(TransactionType type) const`
3. Method should check both running transactions and pending/scheduled transactions
4. Return `true` if any transaction of the specified type exists, `false` otherwise

## Definition of Done
- [ ] Method declared in `TransactionsScheduler.h`
- [ ] Method implemented in `TransactionsScheduler.cpp`
- [ ] Method checks all relevant transaction collections
- [ ] Code compiles without warnings
- [ ] Existing functionality unchanged

# Implementation Plan

## Step 1: Analyze TransactionsScheduler structure
- File: `src/core/transactions/scheduler/TransactionsScheduler.h`
- File: `src/core/transactions/scheduler/TransactionsScheduler.cpp`
- Identify how transactions are stored (maps, vectors, etc.)
- Identify transaction type accessor pattern

## Step 2: Add method declaration
In `TransactionsScheduler.h`, add:
```cpp
public:
    /**
     * Checks if any transaction of the specified type is currently
     * active (running) or scheduled (pending execution).
     *
     * @param type The transaction type to check for
     * @return true if a transaction of this type exists, false otherwise
     */
    bool hasActiveTransactionOfType(TransactionType type) const;
```

## Step 3: Implement the method
In `TransactionsScheduler.cpp`:
- Iterate through active transactions collection(s)
- Check each transaction's type via `transactionType()` method
- Return true on first match, false if none found

Reference existing iteration patterns in TransactionsScheduler for the correct collections to check.

## Step 4: Consider edge cases
- Transaction just started vs. transaction about to finish
- Multiple transaction collections (if applicable)
- Thread safety (if scheduler uses locks, respect them)

# Test Plan

**Complexity**: Simple

Unit tests will be implemented in Task 06 (Unit Tests). This task focuses on implementation only.

Expected test coverage (to be implemented in Task 06):
- Returns false when no transactions exist
- Returns true when transaction of specified type exists
- Returns false when transactions exist but none match the type

# Verification and Validation

## Architecture integrity
- Read-only query method, no side effects
- Follows existing TransactionsScheduler patterns
- Does not modify transaction state or lifecycle

## Security
- N/A - internal query method only

## Performance
- O(n) where n is number of active transactions
- Typically very small n (< 100 transactions at any time)
- No blocking or expensive operations

## Scalability
- Performance scales linearly with active transaction count
- Acceptable for expected transaction volumes

## Reliability
- Const method ensures no unintended modifications
- Returns definitive boolean result
- No exceptions expected

## Maintainability
- Clear method name indicates purpose
- Single responsibility
- Well-documented with doxygen comments

## Cost
- N/A

## Compliance
- Follows project coding standards

# Restrictions
- Do not modify transaction lifecycle or state
- Do not add dependencies on specific transaction types
- Method must be const (read-only)
- Commit changes only after code compiles without warnings
