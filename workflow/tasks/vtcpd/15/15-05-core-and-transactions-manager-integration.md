# 15-05 - Core and TransactionsManager Integration

# Links
- [PRD](../../prd/vtcpd/15-completed-payments-observer-monitoring.md)
- [Task 02 - DelayedTask](15-02-completed-payments-monitoring-delayed-task.md)
- [Task 03 - TransactionsScheduler Check](15-03-transactions-scheduler-active-transaction-check.md)
- [Task 04 - Transaction](15-04-completed-payments-observer-monitoring-transaction.md)

# Description
Integrate the `CompletedPaymentsMonitoringDelayedTask` and `CompletedPaymentsObserverMonitoringTransaction` into the Core and TransactionsManager infrastructure. This task wires up the delayed task's signal to TransactionsManager, which creates and launches the monitoring transaction.

The integration includes:
1. Core: Create and initialize the delayed task
2. TransactionsManager: Subscribe to the monitoring signal and launch transactions
3. Duplicate prevention: Check if a monitoring transaction is already running before creating a new one

# Requirements and DOD

## Functional Requirements

### Core Integration
1. Add member `unique_ptr<CompletedPaymentsMonitoringDelayedTask>` to Core class
2. Create method `Core::initCompletedPaymentsMonitoringDelayedTask()`
3. Call initialization in `Core::init()` after `initObserverRpcCommunicator()`
4. Pass required dependencies (io_context, logger) to delayed task constructor
5. Connect delayed task signal to TransactionsManager

### TransactionsManager Integration
6. Add method `subscribeForCompletedPaymentsMonitoringSignal(MonitoringSignal& signal)`
7. Add slot method `onCompletedPaymentsMonitoringSlot()`
8. Add method `launchCompletedPaymentsObserverMonitoringTransaction()`
9. In slot: check via TransactionsScheduler if transaction already exists
10. If exists: log and return without creating new transaction
11. If not exists: create and schedule the monitoring transaction

### Signal/Slot Connection
12. Subscribe to signal during TransactionsManager initialization or Core signal wiring
13. Ensure proper signal lifetime management

## Definition of Done
- [ ] Core has delayed task member variable
- [ ] Core initialization method created and called
- [ ] TransactionsManager signal subscription method created
- [ ] TransactionsManager slot method created
- [ ] TransactionsManager transaction launch method created
- [ ] Duplicate transaction check implemented
- [ ] Signal properly connected
- [ ] Logging for all significant events
- [ ] Code compiles without warnings

# Implementation Plan

## Step 1: Update Core.h
Add member and method declaration:
```cpp
private:
    unique_ptr<CompletedPaymentsMonitoringDelayedTask> mCompletedPaymentsMonitoringDelayedTask;

private:
    int initCompletedPaymentsMonitoringDelayedTask();
```

Add include for `CompletedPaymentsMonitoringDelayedTask.h`

## Step 2: Implement Core initialization
In `Core.cpp`:

```cpp
int Core::initCompletedPaymentsMonitoringDelayedTask()
{
    try {
        mCompletedPaymentsMonitoringDelayedTask = make_unique<CompletedPaymentsMonitoringDelayedTask>(
            mIOCtx,
            *mLog);
        info() << "Completed Payments Monitoring Delayed Task is successfully initialized";
        return 0;
    } catch (const exception &e) {
        mLog->logException("Core", e);
        return -1;
    }
}
```

Call this method in `Core::init()` after `initObserverRpcCommunicator()`.

## Step 3: Update TransactionsManager.h
Add method declarations:
```cpp
private:
    void subscribeForCompletedPaymentsMonitoringSignal(
        CompletedPaymentsMonitoringDelayedTask::MonitoringSignal &signal);

    void onCompletedPaymentsMonitoringSlot();

    void launchCompletedPaymentsObserverMonitoringTransaction();
```

Add include for the transaction header.

## Step 4: Implement TransactionsManager methods
In `TransactionsManager.cpp`:

### subscribeForCompletedPaymentsMonitoringSignal
```cpp
void TransactionsManager::subscribeForCompletedPaymentsMonitoringSignal(
    CompletedPaymentsMonitoringDelayedTask::MonitoringSignal &signal)
{
    signal.connect(
        boost::bind(
            &TransactionsManager::onCompletedPaymentsMonitoringSlot,
            this));
}
```

### onCompletedPaymentsMonitoringSlot
```cpp
void TransactionsManager::onCompletedPaymentsMonitoringSlot()
{
    // Check if transaction already running
    if (mScheduler->hasActiveTransactionOfType(
            TransactionType::Payments_CompletedPaymentsObserverMonitoring)) {
        info() << "CompletedPaymentsObserverMonitoring transaction already active, skipping";
        return;
    }

    launchCompletedPaymentsObserverMonitoringTransaction();
}
```

### launchCompletedPaymentsObserverMonitoringTransaction
```cpp
void TransactionsManager::launchCompletedPaymentsObserverMonitoringTransaction()
{
    try {
        auto transaction = make_shared<CompletedPaymentsObserverMonitoringTransaction>(
            mStorageHandler,
            mKeystore,
            *mLog);

        subscribeForOutgoingRpcRequestSignal(
            transaction->outgoingRpcRequestSignal);

        prepareAndSchedule(transaction);

        info() << "CompletedPaymentsObserverMonitoring transaction launched";
    } catch (const exception &e) {
        mLog->logException("TransactionsManager", e);
    }
}
```

## Step 5: Wire up signal in Core
In `Core::connectSignalsToSlots()` or appropriate location:
```cpp
mTransactionsManager->subscribeForCompletedPaymentsMonitoringSignal(
    mCompletedPaymentsMonitoringDelayedTask->monitoringSignal);
```

Alternatively, pass the signal during TransactionsManager construction or initialization.

## Step 6: Add logging
- Log when delayed task is initialized
- Log when signal is received (in slot)
- Log when transaction is skipped (already running)
- Log when transaction is launched

# Test Plan

**Complexity**: Moderate

This is an integration task. Functional verification through manual testing or integration tests:
- Node starts successfully with new delayed task
- After 60 seconds, monitoring transaction is created
- Subsequent cycles occur every 300 seconds
- Duplicate transactions are prevented

Unit tests for individual components are covered in Task 06.

# Verification and Validation

## Architecture integrity
- Follows existing Core initialization patterns (reference: `initTopologyEventDelayedTask`)
- Follows existing TransactionsManager signal subscription patterns
- Signal/slot pattern consistent with other delayed tasks
- Transaction launch pattern consistent with other transaction types

## Security
- No new security concerns
- Uses existing secure transaction creation patterns
- Keystore passed securely to transaction

## Performance
- Minimal overhead: one signal subscription
- Duplicate check is O(n) but runs infrequently (every 300s)
- No impact on critical paths

## Scalability
- N/A - integration layer only

## Reliability
- Exception handling in initialization
- Graceful handling of duplicate transaction attempts
- Logging for debugging

## Maintainability
- Consistent with existing integration patterns
- Clear method names
- Proper separation of concerns

## Cost
- N/A

## Compliance
- Follows project architecture patterns

# Restrictions
- Do not modify delayed task or transaction logic (implemented in previous tasks)
- Do not change existing signal/slot patterns
- Commit changes only after code compiles without warnings and node starts successfully
