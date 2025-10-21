# 07-04 - Connect RequestExchangePathsResourceSignal in Core

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- [Previous task: Task 07-03](07-03-implement-findpathsbymaxflowexchangetransaction.md)

# Description
Add signal connection in `Core::connectResourcesManagerSignals()` to link `RequestExchangePathsResourceSignal` to the transaction launcher. This enables automatic launching of `FindPathsByMaxFlowExchangeTransaction` when `CoordinatorExchangePaymentTransaction` requests exchange path collection.

This task implements the Core-level infrastructure connecting the resource management system to the path collection transaction, completing the signal-based communication chain.

# Requirements and DOD

## Functional Requirements
1. **Add onExchangePathsResourceRequestedSlot() method to Core**
   - Method signature: `void onExchangePathsResourceRequestedSlot(const TransactionUUID&, BaseAddress::Shared, const vector<SerializedEquivalent>&, const SerializedEquivalent)`
   - Parameters match `RequestExchangePathsResourceSignal` signature
   - Launch `FindPathsByMaxFlowExchangeTransaction` with all required parameters
   - Schedule transaction via `mTransactionsManager->scheduleTransaction()`

2. **Connect signal in connectResourcesManagerSignals()**
   - Add signal connection after existing resource signal connections
   - Use `boost::bind()` to bind slot method with placeholders
   - Connection persists for application lifetime

3. **Slot implementation requirements**
   - Create `FindPathsByMaxFlowExchangeTransaction` instance with:
     - `contractorAddress` from signal parameter
     - `requestedTransactionUUID` from signal parameter
     - `receiverEquivalent` from signal parameter
     - `exchangeEquivalents` from signal parameter
     - All manager components from Core members
     - `hopsCount = 7` (default topology depth)
   - Schedule created transaction immediately

## Definition of Done
- [ ] `Core.h` declares `onExchangePathsResourceRequestedSlot()` method
- [ ] `Core.cpp` implements slot method correctly
- [ ] Signal connection added in `connectResourcesManagerSignals()`
- [ ] `FindPathsByMaxFlowExchangeTransaction` launched with all required parameters
- [ ] Transaction scheduled via `mTransactionsManager`
- [ ] Code compiles without errors or warnings
- [ ] Code follows existing Core signal connection patterns
- [ ] Method visibility is private (slot methods are internal)

# Implementation Plan

## Step 1: Add Slot Method Declaration to Core.h
**File**: `src/core/Core.h`

Add the method declaration in the private slots section (after existing resource slot methods):

```cpp
private:
    // ... existing slot methods ...

    void onExchangePathsResourceRequestedSlot(
        const TransactionUUID &transactionUUID,
        BaseAddress::Shared contractorAddress,
        const vector<SerializedEquivalent> &exchangeEquivalents,
        const SerializedEquivalent receiverEquivalent);
```

**Location**: Find the section with other resource-related slots (e.g., `onPathsResourceRequestedSlot`) and add this method in the same area.

## Step 2: Implement Slot Method in Core.cpp
**File**: `src/core/Core.cpp`

Add the implementation (find a location near other resource slot implementations):

```cpp
void Core::onExchangePathsResourceRequestedSlot(
    const TransactionUUID &transactionUUID,
    BaseAddress::Shared contractorAddress,
    const vector<SerializedEquivalent> &exchangeEquivalents,
    const SerializedEquivalent receiverEquivalent)
{
    info() << "Exchange paths requested for transaction " << transactionUUID
           << ", contractor " << contractorAddress->fullAddress()
           << ", " << exchangeEquivalents.size() << " exchange equivalent(s)"
           << ", receiver equivalent " << receiverEquivalent;

    auto transaction = make_shared<FindPathsByMaxFlowExchangeTransaction>(
        contractorAddress,
        transactionUUID,
        receiverEquivalent,
        exchangeEquivalents,
        mContractorsManager.get(),
        mResourcesManager.get(),
        mEquivalentsSubsystemsRouter.get(),
        mTailManager.get(),
        mExchangePathsManager.get(),
        mExchangeRatesManager.get(),
        mCommissionsManager.get(),
        mLogger,
        7); // hopsCount - default topology collection depth

    mTransactionsManager->scheduleTransaction(transaction);

    debug() << "Scheduled FindPathsByMaxFlowExchangeTransaction for exchange paths";
}
```

**Implementation notes**:
- Logging follows Core conventions (info for major events, debug for internal details)
- `hopsCount = 7` matches existing topology collection defaults
- All manager pointers passed via `.get()` (raw pointers from unique_ptr)
- Transaction scheduled immediately (no delay)

## Step 3: Add Signal Connection in connectResourcesManagerSignals()
**File**: `src/core/Core.cpp`

Find the `Core::connectResourcesManagerSignals()` method and add connection:

```cpp
void Core::connectResourcesManagerSignals()
{
    // ... existing signal connections ...

    mResourcesManager->requestExchangePathsResourceSignal.connect(
        boost::bind(
            &Core::onExchangePathsResourceRequestedSlot,
            this,
            _1, _2, _3, _4));

    info() << "RequestExchangePathsResourceSignal connected";
}
```

**Connection details**:
- Add after existing `requestPathsResourcesSignal` connection (similar pattern)
- Placeholders `_1, _2, _3, _4` match signal's 4 parameters
- Logging confirms connection established

## Step 4: Add Required Include
**File**: `src/core/Core.cpp`

Ensure the header for `FindPathsByMaxFlowExchangeTransaction` is included:

```cpp
#include "../transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.h"
```

Add near other transaction includes (alphabetically or grouped with find_path transactions).

## Step 5: Verify Compilation
Build the Core component:
```bash
cd build-tests
cmake ..
make Core -j4
```

Verify:
- No compilation errors
- No warnings about missing includes or undefined references
- Transaction header included correctly

## Expected Files Modified
- `src/core/Core.h` (slot method declaration)
- `src/core/Core.cpp` (slot implementation, signal connection, include)

# Test Plan

## Test Approach
This task creates infrastructure that will be tested in Task 07-010 (unit tests for Core signal connection). The focus here is on correct implementation.

## Demo Requirements (Before Commit)
Create a simple demonstration showing signal connection and transaction launching:

1. **Demo script location**: `workspace/demos/07-4-core-signal-demo.cpp` (temporary, not committed)

2. **Demo scenario**:
   - Create minimal Core instance (or use mock objects)
   - Call `connectResourcesManagerSignals()`
   - Trigger `requestExchangePathsResourceSignal` emission
   - Verify `FindPathsByMaxFlowExchangeTransaction` created and scheduled
   - Check logging output for confirmation messages

3. **Alternative manual verification**:
   Since Core requires full initialization, manual code review is acceptable:
   - Verify slot method signature matches signal
   - Verify all parameters passed correctly to transaction constructor
   - Verify transaction scheduled via `mTransactionsManager`
   - Verify signal connection uses correct placeholders

4. **Success criteria**:
   - Code compiles without errors
   - Signal connection pattern matches existing resource signal connections
   - Slot method parameters match signal signature exactly
   - Transaction constructor receives all required parameters in correct order

## Testing Notes
- Comprehensive unit tests created in Task 07-010
- This task verified via code review and compilation checks
- Full integration testing occurs when Task 07-05 requests exchange paths

# Verification and Validation

## Architecture integrity
- **Compliance**: Follows existing Core signal connection architecture
- **Pattern consistency**: Matches pattern from `onPathsResourceRequestedSlot()` (single-equivalent version)
- **Separation of concerns**: Core orchestrates, doesn't implement business logic
- **Signal-based decoupling**: Transaction launcher decoupled from resource requestor

## Security
- **No security impact**: Signal connection is internal infrastructure
- **Transaction UUID validation**: Handled by transaction itself (not Core's responsibility)
- **No authentication/authorization**: Internal component communication

## Performance
- **Minimal overhead**: Signal connection is one-time initialization cost
- **Transaction scheduling**: Uses existing `TransactionsManager` (no additional latency)
- **Slot execution**: O(1) complexity, creates single transaction
- **Acceptable performance**: Slot execution < 1ms (transaction creation + scheduling)

## Scalability
- **Concurrent requests**: Handled by `TransactionsManager` scheduling (existing mechanism)
- **Signal thread safety**: Boost.Signals2 handles concurrent signal emissions
- **No resource contention**: Each signal emission creates independent transaction

## Reliability
- **Error handling**: Transaction creation failures logged by TransactionsManager
- **No failure propagation**: Signal emission doesn't throw (Boost.Signals2 guarantee)
- **Graceful degradation**: If transaction fails to schedule, TransactionsManager logs error

## Maintainability
- **Code clarity**: Slot method name clearly indicates purpose (`onExchangePathsResourceRequestedSlot`)
- **Consistency**: Follows naming convention of existing slot methods
- **Documentation**: Logging provides runtime insight into signal activity
- **Future extensibility**: Easy to modify hopsCount or add parameters if needed

## Cost
- **Development cost**: ~1-2 hours implementation
- **Testing cost**: Covered in Task 07-010
- **Maintenance cost**: Minimal, stable signal infrastructure

## Compliance
- **Policy compliance**: Follows task-driven development (PRD 07)
- **Coding standards**: Adheres to Core component conventions
- **Signal pattern**: Matches existing ResourcesManager signal patterns

# Restrictions
- Commit changes only after successful compilation and code review
- Do not implement tests in this task (tests are in Task 07-010)
- Do not modify other Core signal connections
- Maintain exact parameter types from `RequestExchangePathsResourceSignal`
- Use `hopsCount = 7` as default (do not make configurable in this task)
