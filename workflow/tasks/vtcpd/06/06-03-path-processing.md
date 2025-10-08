# 06-03 - Path Processing in CoordinatorExchangePaymentTransaction

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task](06-02-data-structures-and-transaction-classes.md)

# Description
Implement path processing logic in CoordinatorExchangePaymentTransaction to retrieve and prepare optimal paths from ExchangePathsManager for payment execution. This includes:
1. Modifying runPaymentInitializationStage() to remove paths resource request and reduce timeout
2. Implementing runPathsResourceProcessingStage() to retrieve paths from ExchangePathsManager
3. Implementing addPathForFurtherProcessing() to initialize OptimalPathResult with nodes conversion
4. Working with mPathsStats map using OptimalPathResult

This task bridges path calculation (PRD 04, 05) with actual payment execution.

# Requirements and DOD

## Requirements

### runPaymentInitializationStage() Modification
1. Remove paths resource request (no mResourcesManager usage for paths)
2. Reduce timeout from maxNetworkDelay(10) to maxNetworkDelay(4)
3. Stage should end with waiting for receiver response only

### runPathsResourceProcessingStage() Implementation
4. Do NOT use popNextResource<PathsResource>() (no resource-based path retrieval)
5. Retrieve optimal paths directly from ExchangePathsManager
6. Iterate through each sender equivalent in mExchangeEquivalents
7. For each equivalent, retrieve paths using PathCacheKey{contractorID, senderEquiv, receiverEquiv}
8. Add paths via addPathForFurtherProcessing() until totalAddedFlow >= mAmount
9. Break when sufficient flow accumulated
10. Use maxNetworkDelay(4) for waiting

### addPathForFurtherProcessing() Implementation
11. Accept OptimalPathResult parameter
12. Initialize mIntermediateNodesStates vector with ReservationRequestDoesntSent for all nodes
13. Convert ids to nodes vector via ContractorsManager
14. Add OptimalPathResult to mPathsStats with unique PathID
15. Handle ContractorID → BaseAddress conversion errors gracefully

### mPathsStats Management
16. Type: map<PathID, unique_ptr<OptimalPathResult>>
17. Generate unique PathID for each added path
18. Support iteration and state updates

## Definition of Done
- [x] runPaymentInitializationStage() modified: no paths resource request, maxNetworkDelay(4)
- [x] runPathsResourceProcessingStage() implemented with ExchangePathsManager integration
- [x] Paths retrieved for each exchangeEquivalent using PathCacheKey
- [x] totalAddedFlow tracked and compared with mAmount
- [x] Loop breaks when totalAddedFlow >= mAmount
- [x] addPathForFurtherProcessing() implemented with all initialization steps
- [x] mIntermediateNodesStates initialized with correct length and values
- [x] ContractorID → BaseAddress conversion via ContractorsManager works
- [x] ValueError thrown if contractor not found
- [x] OptimalPathResult added to mPathsStats with unique PathID
- [x] PathID generation function implemented (generateNextPathID())
- [x] Code compiles without errors
- [x] Integration with ExchangePathsManager functional

# Implementation Plan

## Step 1: Modify runPaymentInitializationStage()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

Current implementation (CoordinatorPaymentTransaction) likely ends with:
```cpp
mResourcesManager->requestPathsResource(...);
return resultWaitForMessageTypes({...}, maxNetworkDelay(10));
```

New implementation:
```cpp
TransactionResult::SharedConst CoordinatorExchangePaymentTransaction::runPaymentInitializationStage()
{
    // ... initialization checks (same as original)

    // Send ReceiverInitPaymentRequestMessage to receiver
    sendMessage<ReceiverInitPaymentRequestMessage>(
        mCommand->contractorAddresses(),
        mEquivalent,
        currentTransactionUUID(),
        mCommand->amount(),
        mCommand->payload());

    // Wait for receiver response with reduced timeout
    // NO paths resource request here
    return resultWaitForMessageTypes(
        {Message::Payments_ReceiverInitPaymentResponse},
        maxNetworkDelay(4));  // Reduced from 10 to 4
}
```

**Key Changes**:
- Remove `mResourcesManager->requestPathsResource()` call
- Change `maxNetworkDelay(10)` → `maxNetworkDelay(4)`

## Step 2: Implement runPathsResourceProcessingStage()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

```cpp
TransactionResult::SharedConst CoordinatorExchangePaymentTransaction::runPathsResourceProcessingStage()
{
    debug() << "runPathsResourceProcessingStage";

    // Step 1: Initialize total flow counter
    TrustLineAmount totalAddedFlow = TrustLineAmount(0);

    // Step 2: Iterate through each sender equivalent
    for (const auto& senderEquiv : mExchangeEquivalents) {
        // Step 3: Create cache key for this sender-receiver equivalent combination
        PathCacheKey key{
            mContractorID,
            senderEquiv,
            mEquivalent  // receiver equivalent
        };

        // Step 4: Retrieve optimal paths from ExchangePathsManager
        auto optimalPaths = mExchangePathsManager->retrievePaths(key);

        if (!optimalPaths) {
            // No paths available for this equivalent combination
            debug() << "No cached paths for sender equiv " << senderEquiv;
            continue;
        }

        // Step 5: Add paths until total flow >= payment amount
        for (const auto& pathResult : *optimalPaths) {
            if (totalAddedFlow >= mCommand->amount()) {
                break;  // Sufficient flow accumulated
            }

            // Step 6: Add path to mPathsStats
            addPathForFurtherProcessing(pathResult);
            totalAddedFlow = totalAddedFlow + pathResult.received_amount;
        }

        // Check if we have enough flow
        if (totalAddedFlow >= mCommand->amount()) {
            break;  // No need to check other equivalents
        }
    }

    // Step 7: Validate that we have sufficient paths
    if (totalAddedFlow < mCommand->amount()) {
        warning() << "Insufficient total flow: " << totalAddedFlow
                  << " < " << mCommand->amount();
        return resultProtocolError();  // Or appropriate error
    }

    // Step 8: Wait for receiver response
    return resultWaitForMessageTypes(
        {Message::Payments_ReceiverInitPaymentResponse},
        maxNetworkDelay(4));
}
```

**Key Points**:
- NO `popNextResource<PathsResource>()`
- Direct ExchangePathsManager access
- PathCacheKey construction: {contractorID, senderEquiv, receiverEquiv}
- Flow accumulation with early break
- maxNetworkDelay(4)

## Step 3: Implement addPathForFurtherProcessing()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

```cpp
void CoordinatorExchangePaymentTransaction::addPathForFurtherProcessing(
    const OptimalPathResult& pathResult)
{
    debug() << "addPathForFurtherProcessing";

    // Step 1: Initialize mIntermediateNodesStates
    size_t pathLength = pathResult.path.ids.size();

    // Create mutable copy to initialize states
    auto pathCopy = make_unique<OptimalPathResult>(pathResult);

    pathCopy->mIntermediateNodesStates.clear();
    pathCopy->mIntermediateNodesStates.resize(
        pathLength,
        OptimalPathResult::NodeState::ReservationRequestDoesntSent);

    // Step 2: Initialize nodes vector in ExchangePath (ContractorID → BaseAddress conversion)
    pathCopy->path.nodes.clear();
    pathCopy->path.nodes.reserve(pathLength);

    for (const auto& contractorID : pathResult.path.ids) {
        auto contractor = mContractorsManager->contractor(contractorID);
        if (!contractor) {
            throw ValueError(
                "CoordinatorExchangePaymentTransaction::addPathForFurtherProcessing: "
                "Contractor not found for ID: " + to_string(contractorID));
        }
        pathCopy->path.nodes.push_back(contractor->mainAddress());
    }

    // Step 3: Generate unique PathID
    PathID pathID = generateNextPathID();

    // Step 4: Add to mPathsStats
    mPathsStats[pathID] = std::move(pathCopy);

    debug() << "Path " << pathID << " added with "
            << pathLength << " nodes, flow: " << pathResult.optimal_flow;
}
```

**Key Points**:
- mIntermediateNodesStates initialized with ReservationRequestDoesntSent
- ContractorID → BaseAddress via ContractorsManager
- ValueError if contractor not found
- Unique PathID generation

## Step 4: Implement generateNextPathID()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

```cpp
PathID CoordinatorExchangePaymentTransaction::generateNextPathID()
{
    // Simple incrementing ID generator
    // Could also use static counter or more sophisticated method

    if (mPathsStats.empty()) {
        return 1;
    }

    // Find max existing PathID and increment
    PathID maxID = 0;
    for (const auto& [pathID, pathResult] : mPathsStats) {
        if (pathID > maxID) {
            maxID = pathID;
        }
    }

    return maxID + 1;
}
```

Add to header:
```cpp
private:
    PathID generateNextPathID();
```

## Step 5: Update Class Members
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

Ensure fields are declared:
```cpp
private:
    ExchangePathsManager *mExchangePathsManager;
    map<PathID, unique_ptr<OptimalPathResult>> mPathsStats;
    vector<SerializedEquivalent> mExchangeEquivalents;
    ContractorID mContractorID;  // Receiver contractor ID
    CreditUsageExchangeCommand::Shared mCommand;
```

# Test Plan

## Unit Tests
As this is a Moderate task, focused testing on core functionality:

### Test Category: runPaymentInitializationStage()
1. **testRunPaymentInitializationStageNoResourcesRequest**: Verify no paths resource request made
2. **testRunPaymentInitializationStageReducedTimeout**: Verify maxNetworkDelay(4) used
3. **testRunPaymentInitializationStageSendsReceiverMessage**: ReceiverInitPaymentRequestMessage sent

### Test Category: runPathsResourceProcessingStage()
4. **testRunPathsResourceProcessingSingleEquivalent**: Process paths for single exchangeEquivalent
   - Setup: mExchangeEquivalents = [1], ExchangePathsManager has paths for equiv 1
   - Expected: Paths retrieved and added to mPathsStats

5. **testRunPathsResourceProcessingMultipleEquivalents**: Process paths for multiple exchangeEquivalents
   - Setup: mExchangeEquivalents = [1, 2, 3], paths available for all
   - Expected: Paths retrieved from multiple equivalents

6. **testRunPathsResourceProcessingSufficientFlowEarlyBreak**: Break when totalAddedFlow >= mAmount
   - Setup: mAmount = 1000, first 2 paths provide 1200 flow
   - Expected: Only 2 paths added, loop breaks early

7. **testRunPathsResourceProcessingNoPathsAvailable**: Handle case when no paths in ExchangePathsManager
   - Setup: ExchangePathsManager returns nullptr for all equivalents
   - Expected: resultProtocolError() returned

8. **testRunPathsResourceProcessingInsufficientFlow**: Handle insufficient total flow
   - Setup: mAmount = 1000, total available flow = 800
   - Expected: All paths added, but resultProtocolError() due to insufficient flow

9. **testRunPathsResourceProcessingPathCacheKeyConstruction**: Verify correct PathCacheKey creation
   - Expected: key.contractorID, key.senderEquiv, key.receiverEquiv set correctly

### Test Category: addPathForFurtherProcessing()
10. **testAddPathForFurtherProcessingInitializesStates**: mIntermediateNodesStates initialized correctly
    - Setup: Path with 5 nodes
    - Expected: mIntermediateNodesStates.size() == 5, all ReservationRequestDoesntSent

11. **testAddPathForFurtherProcessingConvertsIdsToNodes**: ContractorID → BaseAddress conversion
    - Setup: Path with 3 contractor IDs, all in ContractorsManager
    - Expected: path.nodes.size() == 3, all BaseAddresses valid

12. **testAddPathForFurtherProcessingContractorNotFound**: Handle missing contractor
    - Setup: Path with contractor ID not in ContractorsManager
    - Expected: ValueError thrown with appropriate message

13. **testAddPathForFurtherProcessingAddsToPathsStats**: Path added to mPathsStats
    - Expected: mPathsStats.size() incremented, unique PathID assigned

14. **testAddPathForFurtherProcessingMultiplePaths**: Add multiple paths
    - Expected: All paths in mPathsStats with unique PathIDs

### Test Category: generateNextPathID()
15. **testGenerateNextPathIDEmptyMap**: First PathID is 1
    - Setup: mPathsStats empty
    - Expected: generateNextPathID() returns 1

16. **testGenerateNextPathIDIncrementing**: PathID increments correctly
    - Setup: mPathsStats has paths with IDs 1, 2, 3
    - Expected: generateNextPathID() returns 4

17. **testGenerateNextPathIDNonSequential**: Handle non-sequential IDs
    - Setup: mPathsStats has paths with IDs 1, 5, 10
    - Expected: generateNextPathID() returns 11

### Test Category: Integration
18. **testEndToEndPathProcessing**: Complete flow from initialization to paths added
    - Setup: Valid exchangeEquivalents, ExchangePathsManager with paths, sufficient flow
    - Expected: runPathsResourceProcessingStage() completes successfully, mPathsStats populated

## Success Criteria
- All 18 unit tests pass
- Code compiles without errors or warnings
- ExchangePathsManager integration functional
- ContractorID → BaseAddress conversion works correctly
- Path state initialization correct

# Verification and Validation

## Architecture integrity
- Clean separation: path retrieval from ExchangePathsManager, state initialization in transaction
- No resource-based path retrieval (direct manager access)
- ContractorsManager used for ID → Address conversion
- OptimalPathResult properly initialized before adding to mPathsStats

## Security
- Contractor validation prevents invalid addresses
- ValueError on missing contractors prevents undefined behavior
- No buffer overflows in vector operations

## Performance
- Direct ExchangePathsManager access more efficient than resource-based retrieval
- Early break when sufficient flow accumulated (O(n) optimization)
- ContractorID → BaseAddress conversion: O(1) per contractor via hash map lookup
- Overall: O(P) where P is number of paths needed (typically < 10)

## Scalability
- Handles up to 5 exchange equivalents (design limit)
- Supports paths up to length 7 (max path length)
- mPathsStats can hold multiple paths (typically < 50)

## Reliability
- Graceful handling of missing paths (continue to next equivalent)
- Validation of sufficient flow before proceeding
- Clear error on contractor not found
- Reduced timeout (4 vs 10) improves responsiveness

## Maintainability
- Clear function responsibilities: retrieve paths, initialize state, convert IDs
- Well-documented algorithm with step-by-step logic
- Easy to add logging for debugging
- Standard error handling patterns

## Cost
- No additional infrastructure required
- Reduced network timeout improves resource usage

## Compliance
- Follows repository policy for task-driven development
- Adheres to existing transaction stage pattern
- No prohibited operations or scope creep

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not modify ExchangePathsManager (use existing interface)
- Do not implement reservation logic (will be done in later tasks)
