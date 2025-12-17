# 13-03 - Timer Infrastructure and Basic Monitoring Cycle

# Links
- [PRD 13: Observer Successful Transactions Monitoring](../../../prd/vtcpd/13-observer-successful-transactions-monitoring.md)
- [Previous task: 13-01 Database Query Method Implementation](13-01-database-query-method.md)
- [Previous task: 13-02 Database Query Method Unit Tests](13-02-database-query-tests.md)

# Description
Implement timer infrastructure in ObservingHandler for periodic monitoring of successful payment transactions. Add timer field, constants for configuration, initialization logic, and basic monitoring cycle that retrieves transactions from database and handles the periodic execution loop. This establishes the foundation for observer claim detection and signature submission (Task 13-04).

The monitoring cycle will run every 60 seconds (10 seconds in test mode), retrieve transactions using the database method from Task 13-01, and provide hooks for future RPC integration.

# Requirements and DOD

## Functional Requirements

### 1. Timer Field and Constants
**File:** `src/core/observing/ObservingHandler.h`

Add to class `ObservingHandler` private section:

```cpp
// Timer for successful transaction monitoring
as::steady_timer mSuccessfulTransactionsMonitorTimer;

// Timer period constants
static constexpr uint32_t kSuccessfulTransactionsMonitoringPeriodSeconds = 60;
#ifdef TESTS
static constexpr uint32_t kSuccessfulTransactionsMonitoringPeriodSecondsTests = 10;
#endif

// Processing limit constant
static constexpr uint32_t kMaxTransactionsPerMonitoringCycle = 100;
```

Add to class `ObservingHandler` private methods section:

```cpp
/**
 * Main monitoring cycle for successful transactions.
 * Retrieves transactions from database and checks for relevant observer claims.
 * Automatically reschedules for next cycle.
 */
void monitorSuccessfulTransactions();

/**
 * Reschedules the successful transactions monitoring timer.
 */
void rescheduleSuccessfulTransactionsMonitor();
```

### 2. Timer Initialization
**File:** `src/core/observing/ObservingHandler.cpp`

Add to `ObservingHandler` constructor (after existing timer initializations):

```cpp
// Initialize successful transactions monitoring timer
#ifdef TESTS
mSuccessfulTransactionsMonitorTimer.expires_after(
    chrono::seconds(kSuccessfulTransactionsMonitoringPeriodSecondsTests));
#else
mSuccessfulTransactionsMonitorTimer.expires_after(
    chrono::seconds(kSuccessfulTransactionsMonitoringPeriodSeconds));
#endif

mSuccessfulTransactionsMonitorTimer.async_wait(
    [this](const boost::system::error_code &e) {
        if (e == boost::asio::error::operation_aborted) {
            return;
        }
        monitorSuccessfulTransactions();
    });
```

### 3. Timer Rescheduling Method
**File:** `src/core/observing/ObservingHandler.cpp`

Implement `rescheduleSuccessfulTransactionsMonitor()`:

```cpp
void ObservingHandler::rescheduleSuccessfulTransactionsMonitor() {
#ifdef TESTS
    mSuccessfulTransactionsMonitorTimer.expires_after(
        chrono::seconds(kSuccessfulTransactionsMonitoringPeriodSecondsTests));
#else
    mSuccessfulTransactionsMonitorTimer.expires_after(
        chrono::seconds(kSuccessfulTransactionsMonitoringPeriodSeconds));
#endif

    mSuccessfulTransactionsMonitorTimer.async_wait(
        [this](const boost::system::error_code &e) {
            if (e == boost::asio::error::operation_aborted) {
                return;
            }
            monitorSuccessfulTransactions();
        });
}
```

### 4. Main Monitoring Cycle
**File:** `src/core/observing/ObservingHandler.cpp`

Implement `monitorSuccessfulTransactions()`:

```cpp
void ObservingHandler::monitorSuccessfulTransactions() {
#ifdef DEBUG_LOG_OBSEVING_HANDLER
    debug() << "Starting successful transactions monitoring cycle";
#endif

    try {
        // Step 1: Get current observer block number
        BlockNumber currentBlockNumber;
        try {
            currentBlockNumber = getActualBlockNumber();
        } catch (const std::exception &e) {
            error() << "Failed to get actual block number: " << e.what();
            rescheduleSuccessfulTransactionsMonitor();
            return;
        }

#ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Current observer block number: " << currentBlockNumber;
#endif

        // Step 2: Retrieve transactions for monitoring
        auto ioTransaction = mStorageHandler->beginTransaction();
        auto transactions = ioTransaction->paymentTransactionsHandler()
            ->transactionsForObserverMonitoring(
                currentBlockNumber,
                kMaxTransactionsPerMonitoringCycle);

#ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Retrieved " << transactions.size()
                << " transactions for monitoring";
#endif

        if (transactions.empty()) {
            // No transactions to monitor
#ifdef DEBUG_LOG_OBSEVING_HANDLER
            debug() << "No transactions to monitor, rescheduling";
#endif
            rescheduleSuccessfulTransactionsMonitor();
            return;
        }

        // Step 3: Check for relevant claims (placeholder for Task 13-04)
        // TODO: Implement checkForRelevantClaims(transactions) in Task 13-04

        info() << "Monitoring cycle completed: processed " << transactions.size()
               << " transactions";

    } catch (const std::exception &e) {
        error() << "Error in successful transactions monitoring: " << e.what();
        // Continue - don't let exceptions break the timer
    }

    // Step 4: Reschedule timer
    rescheduleSuccessfulTransactionsMonitor();
}
```

### 5. Debug Logging Macro
**Verify/Add if needed:** `DEBUG_LOG_OBSEVING_HANDLER` macro definition in ObservingHandler or project build system

## Definition of Done
- [ ] Timer field `mSuccessfulTransactionsMonitorTimer` added to ObservingHandler
- [ ] Constants added: `kSuccessfulTransactionsMonitoringPeriodSeconds`, `kSuccessfulTransactionsMonitoringPeriodSecondsTests`, `kMaxTransactionsPerMonitoringCycle`
- [ ] Method declarations added to header file
- [ ] Timer initialized in ObservingHandler constructor
- [ ] `rescheduleSuccessfulTransactionsMonitor()` implemented
- [ ] `monitorSuccessfulTransactions()` implemented with:
  - Block number retrieval
  - Transaction retrieval from database
  - Empty result handling
  - Exception handling
  - Debug logging
  - Timer rescheduling
- [ ] Code compiles without errors or warnings
- [ ] Timer starts automatically when ObservingHandler is constructed
- [ ] Monitoring cycle can execute successfully (even with empty transactions)
- [ ] No crashes or resource leaks in timer processing
- [ ] TODO comment added for Task 13-04 integration point

# Implementation Plan

## Step 1: Add Timer Field and Constants to Header
**File:** `src/core/observing/ObservingHandler.h`

1. Locate class `ObservingHandler` definition
2. Find existing timer fields (e.g., `mPaymentClaimsTimer`, `mTransactionsTimer`)
3. Add new timer field `mSuccessfulTransactionsMonitorTimer` in same section
4. Find existing constants section (if any) or create one
5. Add three constants for period and limit
6. Find private methods section
7. Add two method declarations: `monitorSuccessfulTransactions()` and `rescheduleSuccessfulTransactionsMonitor()`

**Reference pattern:** Existing timer pattern in ObservingHandler (lines 83-100 in .h file)

## Step 2: Initialize Timer in Constructor
**File:** `src/core/observing/ObservingHandler.cpp`

1. Locate `ObservingHandler` constructor
2. Find existing timer initializations (search for `expires_after`)
3. Add new timer initialization after existing ones
4. Follow pattern: check `#ifdef TESTS`, set appropriate period, use lambda for async_wait
5. Lambda should check for `operation_aborted` and call `monitorSuccessfulTransactions()`

**Reference pattern:** See initialization of `mPaymentClaimsTimer` in constructor

## Step 3: Implement Rescheduling Method
**File:** `src/core/observing/ObservingHandler.cpp`

1. Add method implementation at end of file (or near other timer methods)
2. Follow exact same pattern as timer initialization
3. Set expires_after based on TESTS macro
4. Set async_wait with same lambda pattern

**Reference pattern:** See similar pattern in `scheduleCheckTransactionsTimer()` method

## Step 4: Implement Main Monitoring Cycle
**File:** `src/core/observing/ObservingHandler.cpp`

1. Add method implementation after rescheduling method
2. Implement try-catch block to prevent exceptions from breaking timer
3. Add debug logging at cycle start
4. Call `getActualBlockNumber()`:
   - Wrap in try-catch
   - On error: log, reschedule, return early
5. Get IO transaction and call `transactionsForObserverMonitoring()`
6. Log transaction count
7. Check if transactions empty:
   - If empty: log, reschedule, return early
8. Add TODO comment for Task 13-04 integration
9. Log cycle completion
10. Catch exceptions at outer level
11. Always call reschedule at end (outside try-catch)

**Error handling strategy:**
- `getActualBlockNumber()` failure: log error, reschedule, return
- Database query failure: exception caught at outer level, logged, timer continues
- Never let exceptions propagate out (timer must continue)

**Reference pattern:** See `processPaymentClaims()` method for similar monitoring cycle pattern

## Step 5: Verify Timer Initialization Constructor Parameter
**File:** `src/core/observing/ObservingHandler.cpp`

1. Check constructor initialization list
2. Verify `mSuccessfulTransactionsMonitorTimer` initialized with IOCtx reference
3. Should follow pattern: `mSuccessfulTransactionsMonitorTimer(static_cast<IOCtx &>(mPaymentClaimsTimer.get_executor().context()))`

**Reference:** See initialization of other timers in constructor initialization list

## Step 6: Build and Basic Testing
1. Build project: `cmake --build build`
2. Verify compilation succeeds
3. If available, run basic smoke test to verify:
   - ObservingHandler constructs successfully
   - Timer starts automatically
   - Monitoring cycle can execute (logs appear)
   - No crashes or immediate errors

## Step 7: Code Review Checklist
- [ ] Timer field initialized in constructor initialization list
- [ ] Timer scheduled in constructor body
- [ ] Both TESTS and non-TESTS paths work (ifdef logic correct)
- [ ] Exception handling prevents timer failure
- [ ] Debug logging uses correct macro
- [ ] No memory leaks (timer managed by ObservingHandler lifetime)
- [ ] TODO comment clear for Task 13-04
- [ ] Code style matches existing ObservingHandler code

# Test Plan

**Testing approach:** This task establishes infrastructure without full functionality. Testing focuses on structural verification and basic execution.

## Manual Verification Tests

### Test 1: Compilation
- Build succeeds without errors or warnings
- No linking errors related to new timer

### Test 2: Timer Initialization
- ObservingHandler constructs successfully
- No crashes during initialization
- Timer starts automatically (verify via debug logs if available)

### Test 3: Basic Monitoring Cycle Execution
- If test environment available:
  - Run ObservingHandler in test mode
  - Verify monitoring cycle executes periodically
  - Check debug logs show:
    - "Starting successful transactions monitoring cycle"
    - Block number retrieval
    - Transaction count (may be 0)
    - "No transactions to monitor" or cycle completion
  - Verify cycle repeats after timer period

### Test 4: Exception Handling
- If test environment available:
  - Simulate observer connection failure
  - Verify error logged: "Failed to get actual block number"
  - Verify timer continues (reschedules despite error)
  - Verify no crash

### Test 5: Empty Transaction Handling
- Normal case: no transactions in claiming window
- Verify cycle handles gracefully
- Verify early return and rescheduling

## Automated Testing
- Full automated testing deferred to Task 13-04 (requires complete RPC integration)
- This task focuses on structural correctness

# Verification and Validation

**Complexity Level:** Moderate (infrastructure setup, timer integration, async execution)

## Architecture integrity
- [ ] Follows existing timer pattern in ObservingHandler
- [ ] Consistent with other monitoring cycles (processPaymentClaims, checkTransactionsTimer)
- [ ] No changes to core ObservingHandler responsibilities
- [ ] Timer lifecycle managed by ObservingHandler lifetime
- [ ] Uses existing IOCtx for async operations

## Security
- [ ] No security implications (internal monitoring only)
- [ ] No exposure of sensitive data in logs
- [ ] Error messages don't leak implementation details

## Performance
- [ ] Timer overhead minimal (async, non-blocking)
- [ ] 60-second period prevents excessive polling
- [ ] Database query limited to 100 transactions per cycle
- [ ] No performance impact on existing timers (independent execution)
- [ ] Exception handling prevents performance degradation from errors

## Scalability
- [ ] Transaction limit prevents unbounded processing
- [ ] Independent timer doesn't interfere with existing timers
- [ ] Async execution allows concurrent operations
- [ ] Database query scalability ensured by Task 13-01

## Reliability
- [ ] Exception handling ensures timer continues after errors
- [ ] Block number retrieval failure handled gracefully
- [ ] Database errors don't crash monitoring cycle
- [ ] Timer auto-reschedules regardless of outcome
- [ ] No resource leaks on error paths

## Maintainability
- [ ] Clear separation of concerns (timer management vs business logic)
- [ ] Debug logging aids troubleshooting
- [ ] TODO comment marks integration point for Task 13-04
- [ ] Follows established patterns (easy for developers to understand)
- [ ] Constants make configuration explicit and changeable

## Cost
- [ ] Minimal CPU overhead (periodic execution every 60s)
- [ ] Memory overhead negligible (single timer object)
- [ ] Database query cost bounded by limit parameter
- [ ] No additional infrastructure required

## Compliance
- [ ] Follows project coding standards
- [ ] Consistent with existing ObservingHandler implementation
- [ ] Debug logging uses standard macros
- [ ] Error handling follows project patterns

# Restrictions
- Commit changes only after code compiles successfully
- Do not implement RPC integration in this task (deferred to Task 13-04)
- Do not modify existing timer infrastructure
- Do not change behavior of existing monitoring cycles
- Stay strictly within scope: timer infrastructure and basic cycle only
- Add TODO comment for Task 13-04 integration (do not implement claim detection here)
