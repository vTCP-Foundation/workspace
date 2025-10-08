# 06-02 - Data Structures and Transaction Classes

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task](06-01-credit-usage-exchange-command.md)

# Description
Create foundational data structures and new transaction class hierarchy for multi-equivalent payment support. This includes:
1. PathReservation structure for (PathID, Amount, Equivalent) tuples
2. OptimalPathResult enhancement with PathStats fields and methods
3. ExchangePath enhancement with both ContractorID and BaseAddress vectors
4. BaseExchangePaymentTransaction base class using EquivalentsSubsystemsRouter
5. CoordinatorExchangePaymentTransaction, ReceiverExchangePaymentTransaction, IntermediateNodeExchangePaymentTransaction

This task establishes the core architecture for multi-equivalent payments, replacing direct manager access with EquivalentsSubsystemsRouter pattern.

# Requirements and DOD

## Requirements

### PathReservation Structure
1. Create PathReservation structure with fields: pathID, amount, equivalent
2. Location: `src/core/transactions/transactions/regular/payments/base/PathReservation.h`
3. Replace all usages of `pair<PathID, ConstSharedTrustLineAmount>` with PathReservation

### OptimalPathResult Enhancement
4. Add fields from PathStats: mMaxPathFlow, mIsValid, mIntermediateNodesStates (vector), NodeState enum
5. Add all 17 methods from PathStats class
6. Do NOT add mPath (Path::Shared) field - use existing ExchangePath path field
7. path() method returns ExchangePath& reference, not Path::Shared

### ExchangePath Enhancement
8. Rename existing `nodes` field to `ids` (vector<ContractorID>)
9. Add new `nodes` field (vector<BaseAddress::Shared>)
10. Add all methods from Path class
11. ContractorID → BaseAddress conversion happens in transaction code via ContractorsManager

### BaseExchangePaymentTransaction
12. Create base class analogous to BasePaymentTransaction
13. Use EquivalentsSubsystemsRouter instead of direct managers (iAmGateway, TrustLinesManager, TopologyCacheManager, MaxFlowCacheManager)
14. Methods accept SerializedEquivalent parameter where needed
15. Location: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.h`
16. Inherits core payment logic structure from BasePaymentTransaction

### CoordinatorExchangePaymentTransaction
17. Create class analogous to CoordinatorPaymentTransaction
18. Inherit from BaseExchangePaymentTransaction
19. Constructor accepts CreditUsageExchangeCommand
20. Use ExchangePathsManager pointer instead of PathsManager
21. mPathsStats uses OptimalPathResult instead of PathStats
22. Location: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

### ReceiverExchangePaymentTransaction
23. Create class analogous to ReceiverPaymentTransaction
24. Inherit from BaseExchangePaymentTransaction
25. Location: `src/core/transactions/transactions/regular/payments/ReceiverExchangePaymentTransaction.h`

### IntermediateNodeExchangePaymentTransaction
26. Create class analogous to IntermediateNodePaymentTransaction
27. Inherit from BaseExchangePaymentTransaction
28. Location: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h`

## Definition of Done
- [x] PathReservation.h created with all three fields (pathID, amount, equivalent)
- [x] OptimalPathResult.h updated with all PathStats fields (mMaxPathFlow, mIsValid, mIntermediateNodesStates, NodeState enum)
- [x] OptimalPathResult has all 17 methods from PathStats implemented
- [x] OptimalPathResult does NOT have mPath field
- [x] ExchangePath.h updated: nodes→ids, new nodes (BaseAddress) field added
- [x] ExchangePath has all methods from Path class
- [x] BaseExchangePaymentTransaction.h/cpp created with EquivalentsSubsystemsRouter
- [x] CoordinatorExchangePaymentTransaction.h/cpp created inheriting BaseExchangePaymentTransaction
- [x] CoordinatorExchangePaymentTransaction uses ExchangePathsManager
- [x] CoordinatorExchangePaymentTransaction mPathsStats uses OptimalPathResult
- [x] ReceiverExchangePaymentTransaction.h/cpp created inheriting BaseExchangePaymentTransaction
- [x] IntermediateNodeExchangePaymentTransaction.h/cpp created inheriting BaseExchangePaymentTransaction
- [x] All classes compile without errors
- [x] Constructor signatures match requirements

# Implementation Plan

## Step 1: Create PathReservation Structure
**File**: `src/core/transactions/transactions/regular/payments/base/PathReservation.h`

```cpp
#ifndef VTCPD_PATHRESERVATION_H
#define VTCPD_PATHRESERVATION_H

#include "../../../../common/Types.h"

struct PathReservation {
    PathID pathID;
    ConstSharedTrustLineAmount amount;
    SerializedEquivalent equivalent;

    PathReservation(
        const PathID &id,
        const ConstSharedTrustLineAmount &amt,
        const SerializedEquivalent &equiv)
        : pathID(id), amount(amt), equivalent(equiv) {}
};

#endif //VTCPD_PATHRESERVATION_H
```

## Step 2: Enhance OptimalPathResult
**File**: `src/core/paths/lib/OptimalPathResult.h`

Add fields:
```cpp
TrustLineAmount mMaxPathFlow;
bool mIsValid;
vector<NodeState> mIntermediateNodesStates;

enum NodeState {
    ReservationRequestDoesntSent = 0,
    NeighbourReservationRequestSent,
    NeighbourReservationApproved,
    ReservationRequestSent,
    ReservationApproved,
    ReservationRejected
};
```

Add all 17 methods from PathStats:
- `void setNodeState(const SerializedPositionInPath positionInPath, const NodeState state)`
- `const TrustLineAmount &maxFlow() const`
- `void shortageMaxFlow(const TrustLineAmount &kAmount)`
- `const ExchangePath &path() const` (returns existing path field)
- `bool containsIntermediateNodes() const`
- `const pair<BaseAddress::Shared, SerializedPositionInPath> currentIntermediateNodeAndPos() const`
- `const pair<BaseAddress::Shared, SerializedPositionInPath> nextIntermediateNodeAndPos() const`
- `const bool reservationRequestSentToAllNodes() const`
- `const bool isNeighborAmountReserved() const`
- `const bool isWaitingForNeighborReservationResponse() const`
- `const bool isWaitingForNeighborReservationPropagationResponse() const`
- `const bool isWaitingForReservationResponse() const`
- `const bool isReadyToSendNextReservationRequest() const`
- `const bool isLastIntermediateNodeProcessed() const`
- `const bool isLastIntermediateNodeApproved() const`
- `const bool isValid() const`
- `void setUnusable()`

**Implementation File**: `src/core/paths/lib/OptimalPathResult.cpp`

## Step 3: Enhance ExchangePath
**File**: `src/core/paths/lib/ExchangePath.h`

Rename and add fields:
```cpp
struct ExchangePath {
    vector<ContractorID> ids;  // renamed from 'nodes'
    vector<BaseAddress::Shared> nodes;  // new field for addresses
    vector<SerializedEquivalent> equivalents;
    vector<ExchangeStep> exchangeSteps;
    TrustLineAmount minCapacity;
    double effectiveExchangeRate;
    TrustLineAmount totalCommissions;

    // Add all methods from Path class
    bool isValid() const;
    TrustLineAmount calculateMaxCapacity() const;
    double calculateEffectiveExchangeRate() const;
    TrustLineAmount sumFixedCommissions() const;
    bool startsWithEquivalent(SerializedEquivalent equiv) const;
    // ... other Path methods
};
```

## Step 4: Create BaseExchangePaymentTransaction
**File**: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.h`

Key differences from BasePaymentTransaction:
1. Constructor signature:
```cpp
BaseExchangePaymentTransaction(
    const TransactionType type,
    const SerializedEquivalent equivalent,
    ContractorsManager *contractorsManager,
    EquivalentsSubsystemsRouter *equivalentsSubsystemsRouter,  // NEW
    StorageHandler *storageHandler,
    ResourcesManager *resourcesManager,
    Keystore *keystore,
    Logger &log,
    SubsystemsController *subsystemsController);
```

2. Protected member:
```cpp
EquivalentsSubsystemsRouter *mEquivalentsSubsystemsRouter;
```

3. Methods with SerializedEquivalent parameter:
```cpp
TrustLinesManager* getTrustLinesManager(const SerializedEquivalent equivalent);
TopologyCacheManager* getTopologyCacheManager(const SerializedEquivalent equivalent);
MaxFlowCacheManager* getMaxFlowCacheManager(const SerializedEquivalent equivalent);
bool iAmGateway(const SerializedEquivalent equivalent);
```

Copy entire BasePaymentTransaction structure and adapt for EquivalentsSubsystemsRouter pattern.

## Step 5: Create CoordinatorExchangePaymentTransaction
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

```cpp
#ifndef VTCPD_COORDINATOREXCHANGEPAYMENTTRANSACTION_H
#define VTCPD_COORDINATOREXCHANGEPAYMENTTRANSACTION_H

#include "base/BaseExchangePaymentTransaction.h"
#include "../../../../paths/ExchangePathsManager.h"
#include "../../../../interface/commands_interface/commands/payments/CreditUsageExchangeCommand.h"
#include "../../../../paths/lib/OptimalPathResult.h"

class CoordinatorExchangePaymentTransaction : public BaseExchangePaymentTransaction
{
public:
    typedef shared_ptr<CoordinatorExchangePaymentTransaction> Shared;

public:
    CoordinatorExchangePaymentTransaction(
        const CreditUsageExchangeCommand::Shared command,
        ContractorsManager *contractorsManager,
        EquivalentsSubsystemsRouter *equivalentsSubsystemsRouter,
        StorageHandler *storageHandler,
        ResourcesManager *resourcesManager,
        ExchangePathsManager *exchangePathsManager,  // NEW
        Keystore *keystore,
        bool isPaymentTransactionsAllowedDueToObserving,
        EventsInterfaceManager *eventsInterfaceManager,
        Logger &log,
        SubsystemsController *subsystemsController);

    TransactionResult::SharedConst run() override;
    const CommandUUID &commandUUID() const;

protected:
    // Stage handlers (copy from CoordinatorPaymentTransaction structure)
    TransactionResult::SharedConst runPaymentInitializationStage();
    TransactionResult::SharedConst runPathsResourceProcessingStage();
    TransactionResult::SharedConst runReceiverRequestProcessingStage();
    // ... other stages

private:
    ExchangePathsManager *mExchangePathsManager;  // Instead of PathsManager
    map<PathID, unique_ptr<OptimalPathResult>> mPathsStats;  // Instead of PathStats
    vector<SerializedEquivalent> mExchangeEquivalents;
    // ... other fields from CoordinatorPaymentTransaction
};

#endif //VTCPD_COORDINATOREXCHANGEPAYMENTTRANSACTION_H
```

Copy entire CoordinatorPaymentTransaction structure and adapt:
- Replace PathsManager → ExchangePathsManager
- Replace PathStats → OptimalPathResult
- Add mExchangeEquivalents from command

## Step 6: Create ReceiverExchangePaymentTransaction
**File**: `src/core/transactions/transactions/regular/payments/ReceiverExchangePaymentTransaction.h`

Copy ReceiverPaymentTransaction structure:
1. Change inheritance to BaseExchangePaymentTransaction
2. Update constructor to use EquivalentsSubsystemsRouter
3. Keep all stage methods structure
4. Will be enhanced in later tasks with multi-equivalent validation logic

## Step 7: Create IntermediateNodeExchangePaymentTransaction
**File**: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h`

Copy IntermediateNodePaymentTransaction structure:
1. Change inheritance to BaseExchangePaymentTransaction
2. Update constructor to use EquivalentsSubsystemsRouter
3. Keep all stage methods structure
4. Will be enhanced in later tasks with multi-equivalent validation logic

## Step 8: Update All Usages of pair<PathID, Amount>
Search and replace `pair<PathID, ConstSharedTrustLineAmount>` with PathReservation in:
- mNodesFinalAmountsConfiguration declarations
- Method signatures
- Message classes (will be handled in Task 06-04)

# Test Plan

## Unit Tests
As this is a Complex task, comprehensive testing is required:

### Test Category: PathReservation
1. **testPathReservationConstruction**: Create PathReservation with all fields
2. **testPathReservationFieldAccess**: Verify all fields accessible

### Test Category: OptimalPathResult Enhancement
3. **testOptimalPathResultFieldsAdded**: Verify all new fields present (mMaxPathFlow, mIsValid, mIntermediateNodesStates)
4. **testOptimalPathResultSetNodeState**: setNodeState() updates correct position
5. **testOptimalPathResultMaxFlow**: maxFlow() returns mMaxPathFlow
6. **testOptimalPathResultShortageMaxFlow**: shortageMaxFlow() reduces flow correctly
7. **testOptimalPathResultPathMethod**: path() returns ExchangePath& (not Path::Shared)
8. **testOptimalPathResultContainsIntermediateNodes**: Validates path structure
9. **testOptimalPathResultCurrentIntermediateNode**: Returns correct current node and position
10. **testOptimalPathResultNextIntermediateNode**: Returns correct next node and position
11. **testOptimalPathResultReservationRequestSentToAllNodes**: Checks all node states
12. **testOptimalPathResultIsNeighborAmountReserved**: Validates neighbor state
13. **testOptimalPathResultIsWaitingForNeighborResponse**: Checks waiting state
14. **testOptimalPathResultIsWaitingForPropagationResponse**: Checks propagation state
15. **testOptimalPathResultIsWaitingForReservationResponse**: Validates response wait
16. **testOptimalPathResultIsReadyToSendNextRequest**: Checks readiness
17. **testOptimalPathResultIsLastIntermediateNodeProcessed**: Validates completion
18. **testOptimalPathResultIsLastIntermediateNodeApproved**: Checks approval
19. **testOptimalPathResultIsValid**: Returns mIsValid value
20. **testOptimalPathResultSetUnusable**: Sets mIsValid to false
21. **testOptimalPathResultNoPathField**: Verify mPath (Path::Shared) field does NOT exist

### Test Category: ExchangePath Enhancement
22. **testExchangePathFieldRenamed**: Verify 'ids' field exists (renamed from 'nodes')
23. **testExchangePathNodesFieldAdded**: Verify 'nodes' (BaseAddress) field exists
24. **testExchangePathMethodsFromPath**: All Path methods available
25. **testExchangePathIdsAndNodesIndependent**: ids and nodes vectors independent

### Test Category: BaseExchangePaymentTransaction
26. **testBaseExchangePaymentTransactionConstruction**: Constructor with EquivalentsSubsystemsRouter
27. **testBaseExchangePaymentTransactionGetTrustLinesManager**: Retrieve manager for specific equivalent
28. **testBaseExchangePaymentTransactionGetTopologyCacheManager**: Retrieve manager for specific equivalent
29. **testBaseExchangePaymentTransactionGetMaxFlowCacheManager**: Retrieve manager for specific equivalent
30. **testBaseExchangePaymentTransactionIAmGateway**: Check gateway status for specific equivalent

### Test Category: CoordinatorExchangePaymentTransaction
31. **testCoordinatorExchangePaymentTransactionConstruction**: Constructor accepts CreditUsageExchangeCommand
32. **testCoordinatorExchangePaymentTransactionInheritsBase**: Inherits from BaseExchangePaymentTransaction
33. **testCoordinatorExchangePaymentTransactionExchangePathsManager**: Uses ExchangePathsManager field
34. **testCoordinatorExchangePaymentTransactionPathsStatsType**: mPathsStats uses OptimalPathResult
35. **testCoordinatorExchangePaymentTransactionExchangeEquivalents**: mExchangeEquivalents from command

### Test Category: ReceiverExchangePaymentTransaction
36. **testReceiverExchangePaymentTransactionConstruction**: Constructor with EquivalentsSubsystemsRouter
37. **testReceiverExchangePaymentTransactionInheritsBase**: Inherits from BaseExchangePaymentTransaction

### Test Category: IntermediateNodeExchangePaymentTransaction
38. **testIntermediateNodeExchangePaymentTransactionConstruction**: Constructor with EquivalentsSubsystemsRouter
39. **testIntermediateNodeExchangePaymentTransactionInheritsBase**: Inherits from BaseExchangePaymentTransaction

## Success Criteria
- All 39 unit tests pass
- Code compiles without errors or warnings
- All classes properly inherit from correct base classes
- EquivalentsSubsystemsRouter integration works correctly

# Verification and Validation

## Architecture integrity
- Clean separation: data structures (PathReservation, enhanced OptimalPathResult/ExchangePath) independent from transaction classes
- BaseExchangePaymentTransaction follows same pattern as BasePaymentTransaction
- Inheritance hierarchy: BaseExchangePaymentTransaction → Coordinator/Receiver/IntermediateNode transactions
- EquivalentsSubsystemsRouter properly integrated for multi-equivalent manager access
- ExchangePathsManager replaces PathsManager in coordinator
- No circular dependencies

## Security
- No security concerns at this stage (data structures and class skeletons)
- Manager access controlled through EquivalentsSubsystemsRouter

## Performance
- PathReservation is lightweight struct (three fields)
- OptimalPathResult memory overhead: additional vector and two fields (acceptable)
- ExchangePath: additional nodes vector populated on-demand
- No performance degradation expected

## Scalability
- Supports up to 5 exchange equivalents per payment (as per design)
- OptimalPathResult handles paths up to length 7 (max path length)
- ExchangePath dual representation (ids + nodes) scales linearly

## Reliability
- All new structures have clear ownership semantics
- OptimalPathResult methods ported from tested PathStats code
- Transaction classes follow proven patterns from existing payment transactions

## Maintainability
- Clear structure separation: data structures, base class, derived classes
- OptimalPathResult encapsulates path state management
- ExchangePath provides both ContractorID and BaseAddress access
- EquivalentsSubsystemsRouter pattern simplifies multi-equivalent logic
- Code follows existing patterns for easy understanding

## Cost
- No additional infrastructure required
- Memory overhead acceptable (additional fields in existing structures)

## Compliance
- Follows repository policy for task-driven development
- Adheres to existing code structure and naming conventions
- No prohibited operations or scope creep

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not implement transaction logic beyond basic structure (will be done in later tasks)
- Do not modify existing BasePaymentTransaction, CoordinatorPaymentTransaction, etc. (old transactions remain untouched)
