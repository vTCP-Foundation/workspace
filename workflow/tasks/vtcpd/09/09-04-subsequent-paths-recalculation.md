# 09-04 - Subsequent Paths Recalculation with Updated Conditions

# Links
- [PRD](../../../prd/vtcpd/09-exchange-rate-commission-change-handling.md)
- [Previous task: 09-03](09-03-coordinator-condition-change-detection.md)

# Description
Implement recalculation of all subsequent paths that contain a node whose exchange rate or commission has changed. When a condition change is detected and handled (Task 09-03), all paths scheduled for processing after the current path that involve the same node must have their flows and received_amount updated to reflect the new conditions.

This ensures that subsequent paths don't fail due to outdated cached conditions and that flow calculations remain accurate throughout the payment execution.

# Requirements and DOD

## Requirements

### R1: Identify Subsequent Paths with Affected Node
- After creating new path in `handleConditionChange()`, iterate through all paths with PathID > current
- For each subsequent path:
  - Check if path contains the affected node (node that sent RejectedDueConditionsChanged)
  - Compare node addresses from path structure

### R2: Recalculate received_amount for Affected Paths
- For each subsequent path containing the affected node:
  - Keep `optimal_flow` unchanged
  - Call `calculateReceivedAmountWithUpdatedConditions()` with:
    - Path stats
    - Current optimal_flow
    - Updated exchange rate (if applicable)
    - Updated commission (if applicable)
  - Update `received_amount` with recalculated value

### R3: Recalculate flows for Affected Paths
- For each affected subsequent path:
  - Call `calculateFlows(optimal_flow)` to regenerate flows vector
  - Handle exceptions from flow calculation
  - Log recalculation action

### R4: Add TODO for ExchangePath mPath Updates
- Add TODO comments in code for future update of ExchangePath mPath fields:
  - exchangeSteps
  - minCapacity
  - effectiveExchangeRate
  - totalCommissions
- Document which fields need updating and why
- Reference this task in TODO comments

### R5: Log Recalculation Actions
- Log each subsequent path that gets recalculated
- Include old and new received_amount values
- Log any errors during recalculation

## Definition of Done

1. All subsequent paths containing affected node are identified correctly

2. received_amount recalculated correctly for each affected path using updated conditions

3. flows recalculated correctly for each affected path via calculateFlows()

4. Paths not containing affected node remain unchanged

5. TODO comments added for ExchangePath mPath fields update

6. Comprehensive logging of recalculation actions

7. Error handling for flow recalculation failures (log and continue)

8. Unit tests pass for subsequent path recalculation scenarios

9. No compilation warnings

# Implementation Plan

## Step 1: Extend handleConditionChange() with Subsequent Paths Update

**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

### 1.1: Add subsequent paths recalculation after new path insertion
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::handleConditionChange(
    const PathID &pathID,
    const optional<ExchangeRate> &actualExchangeRate,
    const optional<TrustLineAmount> &actualCommission)
{
    // ... [Steps 1-6 from Task 09-03: drop reservations, update managers, create new path] ...

    // Step 7: Update all subsequent paths containing affected node
    updateSubsequentPathsWithChangedConditions(
        pathID,
        affectedNode,  // Node that sent RejectedDueConditionsChanged
        actualExchangeRate,
        actualCommission);

    // Step 8: Continue processing
    return tryProcessNextPath();
}
```

## Step 2: Implement updateSubsequentPathsWithChangedConditions()

### 2.1: Iterate through subsequent paths and update
```cpp
void CoordinatorExchangePaymentTransaction::updateSubsequentPathsWithChangedConditions(
    const PathID &currentPathID,
    BaseAddress::Shared affectedNode,
    const optional<ExchangeRate> &updatedRate,
    const optional<TrustLineAmount> &updatedCommission)
{
    // Find current path index
    auto currentIt = std::find(mPathIDs.begin(), mPathIDs.end(), currentPathID);
    if (currentIt == mPathIDs.end()) {
        warning() << "Current path not found in mPathIDs";
        return;
    }

    size_t currentIndex = std::distance(mPathIDs.begin(), currentIt);

    // Iterate through all subsequent paths (starting from currentIndex + 2,
    // because currentIndex + 1 is the newly created path)
    for (size_t idx = currentIndex + 2; idx < mPathIDs.size(); ++idx) {
        PathID subsequentPathID = mPathIDs[idx];
        auto pathIt = mPathsStats.find(subsequentPathID);

        if (pathIt == mPathsStats.end()) {
            warning() << "Path stats not found for subsequent pathID=" << subsequentPathID;
            continue;
        }

        OptimalPathResult *pathStats = pathIt->second.get();

        // Check if path contains affected node
        if (!pathContainsNode(pathStats, affectedNode)) {
            continue;  // Skip paths that don't involve this node
        }

        info() << "Updating subsequent path " << subsequentPathID
               << " affected by condition change";

        // Store old value for logging
        TrustLineAmount oldReceivedAmount = pathStats->received_amount;

        // Recalculate received_amount
        try {
            TrustLineAmount newReceivedAmount = calculateReceivedAmountWithUpdatedConditions(
                pathStats,
                pathStats->optimal_flow,
                updatedRate,
                updatedCommission);

            pathStats->received_amount = newReceivedAmount;

            info() << "Updated path " << subsequentPathID
                   << ": receivedAmount changed from " << oldReceivedAmount
                   << " to " << newReceivedAmount;

        } catch (const exception &e) {
            warning() << "Error recalculating received_amount for path "
                      << subsequentPathID << ": " << e.what();
            continue;
        }

        // Recalculate flows
        try {
            pathStats->calculateFlows(pathStats->optimal_flow);

            info() << "Recalculated flows for path " << subsequentPathID;

        } catch (const exception &e) {
            warning() << "Error recalculating flows for path "
                      << subsequentPathID << ": " << e.what();
        }

        // TODO: Update ExchangePath mPath fields
        // The following fields in pathStats->path need to be updated:
        // - exchangeSteps: recalculate exchange steps with new rate/commission
        // - minCapacity: recalculate minimum capacity along path
        // - effectiveExchangeRate: recalculate overall exchange rate
        // - totalCommissions: recalculate total commissions across path
        //
        // This update is deferred to maintain task scope.
        // Reference: Task 09-04, Step 2.1
    }

    info() << "Completed updating subsequent paths affected by condition change";
}
```

## Step 3: Implement pathContainsNode() Helper

### 3.1: Check if path contains specific node
```cpp
bool CoordinatorExchangePaymentTransaction::pathContainsNode(
    OptimalPathResult *pathStats,
    BaseAddress::Shared targetNode) const
{
    const auto &path = pathStats->path();

    // Check all node addresses in path
    for (const auto &nodeAddress : path.nodes) {
        if (nodeAddress->fullAddress() == targetNode->fullAddress()) {
            return true;
        }
    }

    return false;
}
```

## Step 4: Track Affected Node in handleConditionChange()

### 4.1: Determine which node sent rejection
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::handleConditionChange(
    const PathID &pathID,
    const optional<ExchangeRate> &actualExchangeRate,
    const optional<TrustLineAmount> &actualCommission)
{
    // ... [existing code] ...

    // Determine affected node from context
    // This requires tracking which node we were processing when rejection occurred
    // Option 1: Track in class member variable during askRemoteNodeToApproveReservation
    // Option 2: Extract from path based on current processing position

    BaseAddress::Shared affectedNode = mCurrentProcessingNode;  // Set during ask* methods

    // ... [rest of handleConditionChange] ...

    // Update subsequent paths
    updateSubsequentPathsWithChangedConditions(
        pathID,
        affectedNode,
        actualExchangeRate,
        actualCommission);

    // ... [continue processing] ...
}
```

### 4.2: Track current processing node
```cpp
// In askRemoteNodeToApproveReservation() and askNeighborToApproveFurtherNodeReservation()
// Add before sending message:

mCurrentProcessingNode = remoteNode;  // or neighbor
```

## Step 5: Add Member Variable for Current Processing Node

**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

### 5.1: Add private field
```cpp
private:
    // ... existing fields ...

    // Track node currently being processed for condition change handling
    BaseAddress::Shared mCurrentProcessingNode;
```

## Step 6: Add Method Signatures to Header

### 6.1: Add private methods
```cpp
private:
    void updateSubsequentPathsWithChangedConditions(
        const PathID &currentPathID,
        BaseAddress::Shared affectedNode,
        const optional<ExchangeRate> &updatedRate,
        const optional<TrustLineAmount> &updatedCommission);

    bool pathContainsNode(
        OptimalPathResult *pathStats,
        BaseAddress::Shared targetNode) const;
```

## Step 7: Add Comprehensive TODO Comments

### 7.1: Document deferred ExchangePath updates
```cpp
// In updateSubsequentPathsWithChangedConditions(), after flow recalculation:

// TODO [Task 09-04]: Update ExchangePath mPath fields for subsequent paths
//
// When exchange rates or commissions change, the ExchangePath structure in
// each affected subsequent path should be updated to reflect new conditions.
//
// Required updates:
// 1. exchangeSteps: For each ExchangeStep in the path:
//    - If step involves affected node with changed exchange rate:
//      Update ExchangeStep.exchangeRate to new value
//    - If step involves affected node with changed commission:
//      Update ExchangeStep.commission to new value
//
// 2. minCapacity: Recalculate minimum capacity along path
//    - Forward simulation with updated conditions
//    - Find bottleneck capacity
//
// 3. effectiveExchangeRate: Recalculate overall exchange rate for path
//    - Compound all exchange rates in path
//    - Account for updated rate if applicable
//
// 4. totalCommissions: Recalculate total commissions across path
//    - Sum all commissions in path
//    - Use updated commission if applicable
//
// Example implementation approach:
//   auto &pathExchange = pathStats->path;
//   for (auto &step : pathExchange.exchangeSteps) {
//       if (step.nodeID == affectedNodeID) {
//           if (updatedRate && step.isExchange) {
//               step.exchangeRate = *updatedRate;
//           }
//           if (updatedCommission && step.isCommission) {
//               step.commission = *updatedCommission;
//           }
//       }
//   }
//   pathExchange.minCapacity = recalculateMinCapacity(pathStats);
//   pathExchange.effectiveExchangeRate = recalculateEffectiveRate(pathStats);
//   pathExchange.totalCommissions = recalculateTotalCommissions(pathStats);
//
// Priority: Medium (paths still functional with current received_amount/flows updates)
// Complexity: Moderate (requires careful recalculation logic)
```

# Test Plan

## Test Scope
Testing of subsequent paths identification and recalculation when conditions change. Integration with Task 09-03 condition change detection.

## Unit Tests

### Subsequent Path Identification Tests
**Test 35**: Identify paths containing affected node
- Setup: 5 paths, paths 2, 4, 5 contain node X
- Condition change on path 1 affecting node X
- Expected: Paths 2, 4, 5 identified for recalculation (not path 3)

**Test 36**: Skip newly created path
- Setup: Condition change creates new path at position currentIndex + 1
- Expected: New path skipped, only paths after new path recalculated

**Test 37**: No subsequent paths affected
- Setup: Condition change on last path
- Expected: No subsequent paths to recalculate, completes gracefully

### Recalculation Tests
**Test 38**: Recalculate received_amount for subsequent paths with exchange rate change
- Setup: Paths 2, 3 contain exchanger node, rate changes from 0.5 to 0.6
- Expected: Both paths have received_amount recalculated correctly

**Test 39**: Recalculate received_amount for subsequent paths with commission change
- Setup: Paths 3, 4 contain commission node, commission changes from 5 to 7
- Expected: Both paths have received_amount recalculated correctly

**Test 40**: Recalculate flows for subsequent paths
- Setup: Subsequent paths with condition change
- Expected: calculateFlows() called for each, flows updated

**Test 41**: Handle recalculation errors gracefully
- Setup: Mock calculateReceivedAmountWithUpdatedConditions to throw exception
- Expected: Error logged, path skipped, other paths still recalculated

### Integration with Task 09-03 Tests
**Test 42**: Full condition change flow with subsequent paths
- Setup: 5 paths, condition change on path 2, paths 3-5 contain affected node
- Expected: Path 2 invalidated, new path created, paths 4-6 recalculated

**Test 43**: Multiple condition changes in sequence
- Setup: Condition changes on paths 1 and 3
- Expected: Each triggers correct subsequent path recalculation

### Path Contains Node Tests
**Test 44**: pathContainsNode returns true for node in path
- Setup: Path with nodes [A, B, C, D], check for B
- Expected: true

**Test 45**: pathContainsNode returns false for node not in path
- Setup: Path with nodes [A, B, C, D], check for E
- Expected: false

## Success Criteria
- All 11 unit tests pass
- Subsequent paths correctly identified based on affected node
- received_amount and flows recalculated correctly
- Error handling prevents cascading failures
- TODO comments comprehensive and actionable
- No compilation warnings

# Verification and Validation

## Architecture integrity
**Validation Level**: Moderate task
- Extends existing handleConditionChange() cleanly
- Uses existing path iteration patterns
- No new dependencies introduced
- TODO comments provide clear upgrade path

## Security
**Validation Level**: Moderate task
- No security implications
- Iterates over controlled path collections
- No external data sources

## Performance
**Validation Level**: Moderate task
- Recalculation overhead: O(n * m) where n = subsequent paths, m = path length
- Typically affects 2-5 paths per condition change
- Total overhead: < 50ms for typical scenarios
- Logarithmic path lookup in mPathsStats

## Scalability
**Validation Level**: Moderate task
- Scales linearly with number of subsequent paths
- Bounded by total payment paths (typically < 50)
- Memory usage: no additional allocation (updates in place)

## Reliability
**Validation Level**: Moderate task
- Errors in recalculation don't stop overall process
- Comprehensive error logging
- Each path recalculated independently (failures isolated)

## Maintainability
**Validation Level**: Moderate task
- Clear helper methods (pathContainsNode, updateSubsequentPaths...)
- Comprehensive TODO comments for future work
- Logging aids debugging
- Separation of concerns maintained

## Cost
**Validation Level**: Moderate task
- No infrastructure cost changes

## Compliance
**Validation Level**: Moderate task
- Implements all PRD requirements for subsequent path updates
- Defers ExchangePath mPath updates with clear TODOs (as specified in PRD)

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not implement ExchangePath mPath fields update (defer with TODO)
- Ensure error handling doesn't block payment continuation
- Log all recalculation actions comprehensively
