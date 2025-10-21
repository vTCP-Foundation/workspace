# 07-03 - Implement FindPathsByMaxFlowExchangeTransaction

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- [Previous task: 07-02](07-02-create-exchangepathsresource-signal-infrastructure.md)
- [Reference implementation: InitiateMaxFlowExchangeCalculationTransaction](../../../../src/core/transactions/transactions/max_flow_calculation/InitiateMaxFlowExchangeCalculationTransaction.h)

# Description
Create a new transaction `FindPathsByMaxFlowExchangeTransaction` that collects network topology and builds optimal exchange payment paths on demand. This transaction is triggered by the ResourcesManager when a payment coordinator needs exchange paths but they are missing or expired from cache.

Unlike `InitiateMaxFlowExchangeCalculationTransaction` (which calculates max flow and returns it as command result), this transaction:
- Does NOT return command results (no user-facing command)
- Caches built paths in `ExchangePathsManager`
- Returns `ExchangePathsResource` via `ResourcesManager` to notify the requesting coordinator

The transaction reuses topology collection infrastructure from `BaseCollectTopologyForExchangeTransaction` and OR-Tools path building logic from `InitiateMaxFlowExchangeCalculationTransaction`.

# Requirements and DOD

## Functional Requirements

### 1. Transaction Class Structure
- **Location**: `src/core/transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.h/.cpp`
- **Inheritance**: Extends `BaseCollectTopologyForExchangeTransaction`
- **Purpose**: Collect topology and build exchange paths for resource-based path requests

### 2. Constructor Parameters
Match `InitiateMaxFlowExchangeCalculationTransaction` parameters EXCEPT command:
```cpp
FindPathsByMaxFlowExchangeTransaction(
    BaseAddress::Shared contractorAddress,           // Target contractor
    const TransactionUUID &requestedTransactionUUID, // Requesting coordinator UUID
    const SerializedEquivalent receiverEquivalent,   // Receiver's equivalent
    const vector<SerializedEquivalent> &exchangeEquivalents, // Sender's equivalents
    ContractorsManager *contractorsManager,
    ResourcesManager *resourcesManager,
    EquivalentsSubsystemsRouter *equivalentsSubsystemsRouter,
    TailManager *tailManager,
    ExchangePathsManager *exchangePathsManager,
    ExchangeRatesManager *exchangeRatesManager,
    CommissionsManager *commissionsManager,
    Logger &logger,
    HopsCount_t hopsCount);
```

### 3. Key Methods

#### sendRequestForCollectingTopology()
- Override from `BaseCollectTopologyForExchangeTransaction`
- Initiates topology collection for specified equivalents
- Returns `TransactionResult::SharedConst`

#### processCollectingTopology()
- Override from `BaseCollectTopologyForExchangeTransaction`
- Builds paths using OR-Tools (reuse logic from `InitiateMaxFlowExchangeCalculationTransaction::applyCustomLogic()`)
- Caches paths in `ExchangePathsManager` for each `PathCacheKey{contractorID, exchangeEquiv, receiverEquiv}`
- Returns `ExchangePathsResource` via `mResourcesManager->putResource()`
- Returns `resultDone()`

### 4. Path Building and Caching
- Call `mExchangePathsManager->calculateMaxFlow()` with parameters:
  - `contractorID`: from `mContractorID`
  - `receiverEquivalent`: from `mReceiverEquivalent`
  - `exchangeEquivalents`: from `mExchangeEquivalents`
  - `senderID`: `TopologyTrustLinesManager::kCurrentNodeID`
  - `hopsCount`: from `mHopsCount`
- Paths automatically cached by `calculateMaxFlow()` method
- Handle exceptions gracefully (log warning, continue to resource return)

### 5. Resource Return
- Create `ExchangePathsResource` with `mRequestedTransactionUUID`
- Call `mResourcesManager->putResource(resource)`
- Return even if no paths found (coordinator handles empty cache)

## Definition of Done
- [ ] `FindPathsByMaxFlowExchangeTransaction.h` created with correct class declaration
- [ ] `FindPathsByMaxFlowExchangeTransaction.cpp` created with complete implementation
- [ ] Constructor initializes all member variables correctly
- [ ] `sendRequestForCollectingTopology()` implemented (may reuse parent logic)
- [ ] `processCollectingTopology()` builds paths using OR-Tools
- [ ] Paths cached in `ExchangePathsManager` via `calculateMaxFlow()`
- [ ] `ExchangePathsResource` returned via `ResourcesManager`
- [ ] Error handling: OR-Tools failures logged, resource still returned
- [ ] `logHeader()` method implemented for debugging
- [ ] All files compile without errors or warnings
- [ ] No memory leaks (smart pointers used correctly)
- [ ] Code follows existing transaction patterns

# Implementation Plan

## Step 1: Create Header File
**File**: `src/core/transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.h`

```cpp
#ifndef VTCPD_FINDPATHSBYMAXFLOWEXCHANGETRANSACTION_H
#define VTCPD_FINDPATHSBYMAXFLOWEXCHANGETRANSACTION_H

#include "../base/BaseCollectTopologyForExchangeTransaction.h"
#include "../../../paths/ExchangePathsManager.h"
#include "../../../rates/manager/ExchangeRatesManager.h"
#include "../../../rates/manager/CommissionsManager.h"
#include "../../../resources/manager/ResourcesManager.h"
#include "../../../resources/resources/ExchangePathsResource.h"

class FindPathsByMaxFlowExchangeTransaction : public BaseCollectTopologyForExchangeTransaction
{
public:
    typedef shared_ptr<FindPathsByMaxFlowExchangeTransaction> Shared;

public:
    FindPathsByMaxFlowExchangeTransaction(
        BaseAddress::Shared contractorAddress,
        const TransactionUUID &requestedTransactionUUID,
        const SerializedEquivalent receiverEquivalent,
        const vector<SerializedEquivalent> &exchangeEquivalents,
        ContractorsManager *contractorsManager,
        ResourcesManager *resourcesManager,
        EquivalentsSubsystemsRouter *equivalentsSubsystemsRouter,
        TailManager *tailManager,
        ExchangePathsManager *exchangePathsManager,
        ExchangeRatesManager *exchangeRatesManager,
        CommissionsManager *commissionsManager,
        Logger &logger,
        HopsCount_t hopsCount);

protected:
    const string logHeader() const override;

private:
    TransactionResult::SharedConst sendRequestForCollectingTopology() override;

    TransactionResult::SharedConst processCollectingTopology() override;

private:
    static const uint32_t kTopologyCollectingMillisecondsTimeout = 300;

private:
    ContractorID mContractorID;
    BaseAddress::Shared mContractorAddress;
    TransactionUUID mRequestedTransactionUUID;
    SerializedEquivalent mReceiverEquivalent;
    vector<SerializedEquivalent> mExchangeEquivalents;
    ExchangePathsManager *mExchangePathsManager;
    ExchangeRatesManager *mExchangeRatesManager;
    CommissionsManager *mCommissionsManager;
    ResourcesManager *mResourcesManager;
    HopsCount_t mHopsCount;
};

#endif // VTCPD_FINDPATHSBYMAXFLOWEXCHANGETRANSACTION_H
```

## Step 2: Implement Constructor
**File**: `src/core/transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.cpp`

```cpp
#include "FindPathsByMaxFlowExchangeTransaction.h"
#include "../../../topology/manager/TopologyTrustLinesManager.h"

FindPathsByMaxFlowExchangeTransaction::FindPathsByMaxFlowExchangeTransaction(
    BaseAddress::Shared contractorAddress,
    const TransactionUUID &requestedTransactionUUID,
    const SerializedEquivalent receiverEquivalent,
    const vector<SerializedEquivalent> &exchangeEquivalents,
    ContractorsManager *contractorsManager,
    ResourcesManager *resourcesManager,
    EquivalentsSubsystemsRouter *equivalentsSubsystemsRouter,
    TailManager *tailManager,
    ExchangePathsManager *exchangePathsManager,
    ExchangeRatesManager *exchangeRatesManager,
    CommissionsManager *commissionsManager,
    Logger &logger,
    HopsCount_t hopsCount) :

    BaseCollectTopologyForExchangeTransaction(
        BaseTransaction::FindPathsByMaxFlowExchangeTransaction,
        contractorsManager,
        equivalentsSubsystemsRouter,
        tailManager,
        logger),

    mContractorAddress(contractorAddress),
    mRequestedTransactionUUID(requestedTransactionUUID),
    mReceiverEquivalent(receiverEquivalent),
    mExchangeEquivalents(exchangeEquivalents),
    mExchangePathsManager(exchangePathsManager),
    mExchangeRatesManager(exchangeRatesManager),
    mCommissionsManager(commissionsManager),
    mResourcesManager(resourcesManager),
    mHopsCount(hopsCount)
{
    // Get contractor ID from address
    auto contractor = mContractorsManager->contractorByAddress(contractorAddress);
    if (contractor == nullptr) {
        throw ValueError("FindPathsByMaxFlowExchangeTransaction: "
                        "Contractor not found for address");
    }
    mContractorID = contractor->getID();
}

const string FindPathsByMaxFlowExchangeTransaction::logHeader() const {
    stringstream s;
    s << "[FindPathsByMaxFlowExchangeTransaction: " << currentTransactionUUID() << "] ";
    return s.str();
}
```

## Step 3: Implement sendRequestForCollectingTopology()
This method should initiate topology collection. Reference `BaseCollectTopologyForExchangeTransaction` or `FindPathByMaxFlowTransaction` for pattern:

```cpp
TransactionResult::SharedConst FindPathsByMaxFlowExchangeTransaction::sendRequestForCollectingTopology() {
    info() << "Requesting topology collection for exchange paths to contractor "
           << mContractorID;

    // Send topology requests for all exchange equivalents
    for (const auto &equivalent : mExchangeEquivalents) {
        sendMessage<MaxFlowCalculationSourceFstLevelMessage>(
            mContractorAddress,
            equivalent,
            mEquivalentsSubsystemsRouter->iAmGateway(equivalent),
            mHopsCount);
    }

    // Also send for receiver equivalent if not in exchange equivalents
    if (find(mExchangeEquivalents.begin(), mExchangeEquivalents.end(), mReceiverEquivalent)
        == mExchangeEquivalents.end()) {
        sendMessage<MaxFlowCalculationSourceFstLevelMessage>(
            mContractorAddress,
            mReceiverEquivalent,
            mEquivalentsSubsystemsRouter->iAmGateway(mReceiverEquivalent),
            mHopsCount);
    }

    // Wait for topology responses
    return resultAwakeAfterMilliseconds(kTopologyCollectingMillisecondsTimeout);
}
```

**Note**: Verify exact message types and methods from `BaseCollectTopologyForExchangeTransaction` parent class. Adapt as needed.

## Step 4: Implement processCollectingTopology()
This is the core method that builds paths:

```cpp
TransactionResult::SharedConst FindPathsByMaxFlowExchangeTransaction::processCollectingTopology() {
    info() << "Building exchange paths for contractor " << mContractorID;

    try {
        // Call ExchangePathsManager to calculate max flow and build paths
        auto result = mExchangePathsManager->calculateMaxFlow(
            mContractorID,
            mReceiverEquivalent,
            mExchangeEquivalents,
            TopologyTrustLinesManager::kCurrentNodeID,
            mHopsCount);

        // Paths are automatically cached by calculateMaxFlow() for each
        // PathCacheKey{mContractorID, exchangeEquiv, mReceiverEquivalent}

        info() << "Exchange path building complete, found " << result.optimalPaths.size()
               << " optimal paths with max flow " << result.maxFlow;

    } catch (const exception &e) {
        warning() << "Error building exchange paths: " << e.what();
        // Continue - return resource even if no paths found
        // Coordinator will handle empty cache
    }

    // Return ExchangePathsResource to notify coordinator
    auto resource = make_shared<ExchangePathsResource>(mRequestedTransactionUUID);
    mResourcesManager->putResource(resource);

    info() << "Returned ExchangePathsResource for transaction "
           << mRequestedTransactionUUID;

    return resultDone();
}
```

## Step 5: Add Transaction Type to BaseTransaction Enum
**File**: `src/core/transactions/transactions/base/BaseTransaction.h`

Add new transaction type to enum (if not already present):
```cpp
enum TransactionType {
    // ... existing types ...
    FindPathsByMaxFlowExchangeTransaction = XXX,  // Use next available number
    // ...
};
```

Check existing enum values and use next sequential number.

## Step 6: Verify Compilation
```bash
cd build-tests
cmake ..
make FindPathsByMaxFlowExchangeTransaction -j4
```

Resolve any:
- Missing includes
- Undefined methods from parent class
- Type mismatches

## Expected Files Created
- `src/core/transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.h`
- `src/core/transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.cpp`

## Expected Files Modified
- `src/core/transactions/transactions/base/BaseTransaction.h` (add enum)

# Test Plan

## Test Approach
Unit tests will be created in separate testing task (Task 07-09). This task focuses on implementation.

## Demo Requirements (Before Commit)
Create demonstration showing transaction functionality:

1. **Demo location**: `workspace/demos/07-3-findpaths-tx-demo.cpp`

2. **Demo scenarios**:
   - **Scenario 1: Transaction creation**
     - Create transaction with valid parameters
     - Verify constructor doesn't throw
     - Verify member variables initialized

   - **Scenario 2: Topology collection simulation**
     - Mock topology data in managers
     - Call transaction methods
     - Verify topology requests sent

   - **Scenario 3: Path building and caching**
     - Populate topology data
     - Run `processCollectingTopology()`
     - Verify paths cached in `ExchangePathsManager`
     - Verify `ExchangePathsResource` returned

   - **Scenario 4: Error handling**
     - Test with invalid contractor
     - Test with empty topology
     - Verify graceful failure (resource still returned)

3. **Demo execution**:
   ```bash
   cd build-tests
   cmake ..
   make 07-3-demo
   ./bin/07-3-demo
   ```

4. **Success criteria**:
   - Transaction creates successfully
   - Topology collection completes
   - Paths built and cached
   - Resource returned correctly
   - No crashes or memory leaks

## Testing Notes
- Full unit tests in Task 07-09
- Demo required before commit
- May need mock managers for demo (TestEnvironment pattern)

# Verification and Validation

## Architecture integrity
- **Pattern consistency**: Follows `FindPathByMaxFlowTransaction` pattern for single-equivalent
- **Inheritance correct**: Proper use of `BaseCollectTopologyForExchangeTransaction`
- **Separation of concerns**: Transaction orchestrates, `ExchangePathsManager` builds paths
- **No architectural violations**: Pure extension following established patterns

## Security
- **No security implications**: Internal topology collection, no external exposure
- **UUID security**: Transaction UUID prevents resource routing errors
- **No authentication changes**: Uses existing topology collection security

## Performance
- **Topology collection**: Same performance as `InitiateMaxFlowExchangeCalculationTransaction`
- **OR-Tools overhead**: Path building O(n*m*p) where n=nodes, m=edges, p=paths
- **Expected completion**: < 5 seconds for typical network sizes
- **No performance regression**: Identical algorithm to existing transaction

## Scalability
- **Network size**: Handles networks up to 1000 nodes (same as existing)
- **Equivalents count**: Supports up to 5 exchange equivalents (PRD 06 limit)
- **Concurrent requests**: Multiple transactions can run concurrently

## Reliability
- **Error handling**: OR-Tools failures caught, logged, resource returned anyway
- **Graceful degradation**: Empty results handled (coordinator detects and fails payment)
- **No data corruption**: Paths cached atomically in thread-safe manager
- **Transaction lifecycle**: Proper cleanup on completion/failure

## Maintainability
- **Code clarity**: Similar structure to `InitiateMaxFlowExchangeCalculationTransaction`
- **Logging**: Debug/info logs at key points for troubleshooting
- **Reusability**: Reuses existing topology collection and path building logic
- **Documentation**: Method purposes clear from names and comments

## Cost
- **Development cost**: ~6-8 hours implementation + 2 hours demo
- **Testing cost**: Covered in Task 07-09
- **Maintenance cost**: Moderate (shares logic with existing transaction)

## Compliance
- **Policy compliance**: Task-driven development (PRD 07)
- **Coding standards**: Follows existing transaction patterns
- **No external dependencies**: Uses existing infrastructure

# Restrictions
- Commit changes only after successfully executing the demo
- Do not implement tests in this task (tests are in Task 07-09)
- Reuse logic from `InitiateMaxFlowExchangeCalculationTransaction` where possible (no duplication)
- Do not modify existing transactions
- Return resource even if path building fails (coordinator handles empty cache)
- Follow exact pattern of `FindPathByMaxFlowTransaction` for single-equivalent paths
