# 09-03 - Coordinator Condition Change Detection and Path Adaptation

# Links
- [PRD](../../../prd/vtcpd/09-exchange-rate-commission-change-handling.md)
- [Previous task 1: 09-01](09-01-message-protocol-extensions.md)
- [Previous task 2: 09-02](09-02-intermediate-node-validation.md)

# Description
Implement coordinator-side logic for detecting exchange rate and commission changes reported by intermediate nodes, adapting payment execution by invalidating affected paths, creating new paths with updated conditions, and updating ExchangeRatesManager and CommissionsManager with current network values.

This task enables the coordinator to populate expected conditions in reservation requests, detect RejectedDueConditionsChanged responses, create replacement paths with correct conditions, and update subsequent paths that contain the affected node, ensuring successful payment completion despite dynamic condition changes.

# Requirements and DOD

## Requirements

### R1: Populate Expected Conditions in Reservation Requests
- Update `askRemoteNodeToApproveReservation()` to populate expected conditions from path data
- Update `askNeighborToApproveFurtherNodeReservation()` to populate expected conditions from path data
- For each node in path:
  - Determine if node is exchanger (different input/output equivalents)
  - Determine if node charges commission (from path ExchangeStep data)
  - Populate expectedExchangeRate if node is exchanger
  - Populate expectedCommission if node charges commission
  - Leave both fields empty if neither applies
- Ensure mutual exclusivity (never populate both fields)

### R2: Detect RejectedDueConditionsChanged Responses
- In `processRemoteNodeResponse()`: detect RejectedDueConditionsChanged
- In `processNeighborFurtherReservationResponse()`: detect RejectedDueConditionsChanged
- Extract actualExchangeRate and actualCommission from response
- Log detection of condition change

### R3: Drop Reservations on Invalidated Path
- Call `dropReservationsOnPath(pathStats, pathID, sendToLastProcessedNode=false)`
- Send FinalPathExchangeConfigurationMessage to intermediate nodes to drop their reservations
- Log reservation dropping action

### R4: Mark Path Unusable
- Call `pathStats->setUnusable()` to mark path as invalid
- Prevent path from being processed again

### R5: Update Exchange Rates Manager
- If actualExchangeRate received:
  - Determine affected node and equivalent pair from path structure
  - Call `mExchangeRatesManager->set(incomingEquiv, outgoingEquiv, actualExchangeRate)`
  - Log manager update

### R6: Update Commissions Manager
- If actualCommission received:
  - Determine affected node and equivalent from path structure
  - If actualCommission > 0: call `mCommissionsManager->set(equivalent, actualCommission)`
  - If actualCommission == 0: call `mCommissionsManager->remove(equivalent)`
  - Log manager update

### R7: Create New Path with Updated Conditions
- Create new OptimalPathResult with:
  - Same path structure as invalidated path
  - `optimal_flow = mMaxPathFlow` from invalidated path
  - Recalculate `received_amount` using `calculateReceivedAmountWithUpdatedConditions()`
  - Recalculate `flows` using `calculateFlows(optimal_flow)`
- Insert new path into `mPathsStats` with next PathID
- Add new PathID to `mPathIDs` vector at position after current path
- Shift all subsequent PathIDs by 1
- Log new path creation

### R8: Implement calculateReceivedAmountWithUpdatedConditions()
- Forward simulate through path applying conditions
- Use updated exchange rate or commission for affected node
- Use original conditions for other nodes
- Return final received amount at receiver
- Handle errors gracefully (e.g., amount exhausted by commission)

### R9: Continue Processing
- After creating new path, call `tryProcessNextPath()` to continue payment

## Definition of Done

1. Both `askRemoteNodeToApproveReservation()` and `askNeighborToApproveFurtherNodeReservation()` correctly populate expected conditions

2. Expected conditions determined correctly from path data:
   - Exchanger nodes get expectedExchangeRate
   - Commission nodes get expectedCommission
   - Other nodes get neither

3. RejectedDueConditionsChanged detected in both response processing methods

4. Reservations dropped correctly on invalidated path

5. Path marked unusable after condition change

6. ExchangeRatesManager updated when exchange rate changes

7. CommissionsManager updated or cleared when commission changes

8. New path created with correct:
   - optimal_flow (same as old)
   - received_amount (recalculated with new conditions)
   - flows (recalculated via calculateFlows)

9. New path inserted at correct position (next PathID after current)

10. calculateReceivedAmountWithUpdatedConditions() correctly simulates forward flow

11. Payment continues processing after adaptation

12. Unit tests pass for all condition change scenarios (Tests 21-28, 32-34 from PRD)

13. No compilation warnings

# Implementation Plan

## Step 1: Populate Expected Conditions in askRemoteNodeToApproveReservation()

**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

### 1.1: Determine node conditions from path
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::askRemoteNodeToApproveReservation(
    OptimalPathResult *pathStats,
    BaseAddress::Shared remoteNode,
    const SerializedPositionInPath remoteNodePositionInPath,
    BaseAddress::Shared nextAfterRemoteNode)
{
    // ... existing code ...

    // Determine expected conditions for remote node
    optional<ExchangeRate> expectedRate;
    optional<TrustLineAmount> expectedCommission;

    const auto &path = pathStats->path();
    const auto &flows = pathStats->flows;

    // Get remote node ID
    ContractorID remoteNodeID = /* get from path.ids[remoteNodePositionInPath] */;

    // Get equivalents at this position
    SerializedEquivalent incomingEquiv = path.equivalents[remoteNodePositionInPath];
    SerializedEquivalent outgoingEquiv = path.equivalents[remoteNodePositionInPath + 1];

    // Check if node is exchanger (different equivalents at same node ID)
    if (incomingEquiv != outgoingEquiv) {
        // Node is exchanger - find exchange rate
        const auto *exchangeStep = findExchangeStep(
            path, remoteNodeID, incomingEquiv, outgoingEquiv);

        if (exchangeStep) {
            expectedRate = exchangeStep->exchangeRate;
        }
    }
    // Check if node charges commission (same equivalent, commission present)
    else {
        const auto *commissionStep = findExchangeStep(
            path, remoteNodeID, incomingEquiv, incomingEquiv);

        if (commissionStep && commissionStep->commission > TrustLineAmount(0)) {
            expectedCommission = commissionStep->commission;
        }
    }

    // Create request message
    auto request = make_shared<CoordinatorReservationRequestMessage>(
        currentTransactionUUID(),
        mCurrentAmountReservingPathIdentifier,
        amount,
        equivalent);

    // Populate expected conditions (mutual exclusivity ensured by if/else above)
    if (expectedRate) {
        request->setExpectedExchangeRate(*expectedRate);
    }
    if (expectedCommission) {
        request->setExpectedCommission(*expectedCommission);
    }

    // ... continue with existing message sending logic ...
}
```

## Step 2: Populate Expected Conditions in askNeighborToApproveFurtherNodeReservation()

### 2.1: Add same logic as Step 1
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::askNeighborToApproveFurtherNodeReservation(
    BaseAddress::Shared neighbor,
    OptimalPathResult *pathStats)
{
    // ... existing code ...

    // Determine expected conditions for neighbor
    // (Same logic as askRemoteNodeToApproveReservation, position = kFirstIntermediateNodeIndex)

    optional<ExchangeRate> expectedRate;
    optional<TrustLineAmount> expectedCommission;

    const auto &path = pathStats->path();
    SerializedPositionInPath neighborPos = kFirstIntermediateNodeIndex;

    ContractorID neighborID = path.ids[neighborPos];
    SerializedEquivalent incomingEquiv = path.equivalents[neighborPos];
    SerializedEquivalent outgoingEquiv = path.equivalents[neighborPos + 1];

    if (incomingEquiv != outgoingEquiv) {
        const auto *exchangeStep = findExchangeStep(
            path, neighborID, incomingEquiv, outgoingEquiv);
        if (exchangeStep) {
            expectedRate = exchangeStep->exchangeRate;
        }
    } else {
        const auto *commissionStep = findExchangeStep(
            path, neighborID, incomingEquiv, incomingEquiv);
        if (commissionStep && commissionStep->commission > TrustLineAmount(0)) {
            expectedCommission = commissionStep->commission;
        }
    }

    // Create request and populate conditions
    auto request = /* create CoordinatorReservationRequestMessage */;

    if (expectedRate) {
        request->setExpectedExchangeRate(*expectedRate);
    }
    if (expectedCommission) {
        request->setExpectedCommission(*expectedCommission);
    }

    // ... continue with existing logic ...
}
```

## Step 3: Detect RejectedDueConditionsChanged in processRemoteNodeResponse()

### 3.1: Add detection and handling
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::processRemoteNodeResponse()
{
    // ... existing response retrieval ...

    if (response->state() == ResponseMessage::RejectedDueConditionsChanged) {
        info() << "Condition change detected from remote node on pathID=" << pathID;

        // Extract actual conditions
        auto actualExchangeRate = response->actualExchangeRate();
        auto actualCommission = response->actualCommission();

        // Handle condition change
        return handleConditionChange(
            pathID,
            actualExchangeRate,
            actualCommission);
    }

    // ... existing response handling ...
}
```

## Step 4: Detect RejectedDueConditionsChanged in processNeighborFurtherReservationResponse()

### 4.1: Add same detection logic
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::processNeighborFurtherReservationResponse()
{
    // ... existing response retrieval ...

    if (response->state() == ResponseMessage::RejectedDueConditionsChanged) {
        info() << "Condition change detected from neighbor on pathID=" << pathID;

        auto actualExchangeRate = response->actualExchangeRate();
        auto actualCommission = response->actualCommission();

        return handleConditionChange(
            pathID,
            actualExchangeRate,
            actualCommission);
    }

    // ... existing response handling ...
}
```

## Step 5: Implement handleConditionChange() Method

### 5.1: Main condition change handling logic
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::handleConditionChange(
    const PathID &pathID,
    const optional<ExchangeRate> &actualExchangeRate,
    const optional<TrustLineAmount> &actualCommission)
{
    // Step 1: Get path stats
    auto pathStatsIt = mPathsStats.find(pathID);
    if (pathStatsIt == mPathsStats.end()) {
        warning() << "Path not found for pathID=" << pathID;
        return tryProcessNextPath();
    }

    OptimalPathResult *pathStats = pathStatsIt->second.get();

    // Step 2: Drop reservations on this path
    dropReservationsOnPath(pathStats, pathID, /* sendToLastProcessedNode */ false);

    // Step 3: Mark path unusable
    pathStats->setUnusable();
    info() << "Marked path " << pathID << " as unusable due to condition change";

    // Step 4: Update managers
    if (actualExchangeRate) {
        updateExchangeRateManager(pathStats, actualExchangeRate.value());
    }

    if (actualCommission) {
        updateCommissionManager(pathStats, actualCommission.value());
    }

    // Step 5: Create new path with updated conditions
    TrustLineAmount oldOptimalFlow = pathStats->mMaxPathFlow;

    TrustLineAmount newReceivedAmount = calculateReceivedAmountWithUpdatedConditions(
        pathStats,
        oldOptimalFlow,
        actualExchangeRate,
        actualCommission);

    // Create new path
    auto newPathStats = make_unique<OptimalPathResult>();
    newPathStats->path = pathStats->path;
    newPathStats->optimal_flow = oldOptimalFlow;
    newPathStats->mMaxPathFlow = oldOptimalFlow;
    newPathStats->received_amount = newReceivedAmount;

    // Recalculate flows
    try {
        newPathStats->calculateFlows(oldOptimalFlow);
    } catch (const exception &e) {
        error() << "Error calculating flows for new path: " << e.what();
        return tryProcessNextPath();
    }

    // Step 6: Insert new path and shift subsequent paths
    PathID newPathID = generateNextPathID();
    mPathsStats[newPathID] = std::move(newPathStats);

    // Find current path index
    auto currentIt = std::find(mPathIDs.begin(), mPathIDs.end(), pathID);
    if (currentIt != mPathIDs.end()) {
        size_t currentIndex = std::distance(mPathIDs.begin(), currentIt);
        mPathIDs.insert(mPathIDs.begin() + currentIndex + 1, newPathID);
    }

    info() << "Created new path " << newPathID
           << " with optimalFlow=" << oldOptimalFlow
           << ", receivedAmount=" << newReceivedAmount;

    // Step 7: Continue processing
    return tryProcessNextPath();
}
```

### 5.2: Helper method to update ExchangeRatesManager
```cpp
void CoordinatorExchangePaymentTransaction::updateExchangeRateManager(
    OptimalPathResult *pathStats,
    const ExchangeRate &newRate)
{
    // Determine affected node and equivalents from path
    // (Find the exchange step that triggered the rejection)

    const auto &path = pathStats->path();
    // Implementation depends on tracking which node sent the rejection
    // For now, find the exchange step in path

    SerializedEquivalent sourceEquiv = /* determine from path */;
    SerializedEquivalent targetEquiv = /* determine from path */;

    mExchangeRatesManager->set(sourceEquiv, targetEquiv, newRate);

    info() << "Updated exchange rate: " << sourceEquiv << "->" << targetEquiv
           << " = " << newRate;
}
```

### 5.3: Helper method to update CommissionsManager
```cpp
void CoordinatorExchangePaymentTransaction::updateCommissionManager(
    OptimalPathResult *pathStats,
    const TrustLineAmount &newCommission)
{
    SerializedEquivalent equivalent = /* determine from path */;

    if (newCommission > TrustLineAmount(0)) {
        mCommissionsManager->set(equivalent, newCommission);
        info() << "Updated commission: eq=" << equivalent
               << " commission=" << newCommission;
    } else {
        mCommissionsManager->remove(equivalent);
        info() << "Removed commission: eq=" << equivalent;
    }
}
```

## Step 6: Implement calculateReceivedAmountWithUpdatedConditions()

### 6.1: Forward simulation through path
```cpp
TrustLineAmount
CoordinatorExchangePaymentTransaction::calculateReceivedAmountWithUpdatedConditions(
    OptimalPathResult *pathStats,
    const TrustLineAmount &inputFlow,
    const optional<ExchangeRate> &updatedRate,
    const optional<TrustLineAmount> &updatedCommission)
{
    TrustLineAmount currentAmount = inputFlow;
    const auto &path = pathStats->path();

    // Simulate forward through path
    for (size_t idx = 0; idx + 1 < path.ids.size(); ++idx) {
        const ContractorID fromNode = path.ids[idx];
        const ContractorID toNode = path.ids[idx + 1];
        const SerializedEquivalent currentEquiv = path.equivalents[idx];
        const SerializedEquivalent nextEquiv = path.equivalents[idx + 1];

        // Check for exchange
        if (fromNode == toNode && currentEquiv != nextEquiv) {
            const auto *exchangeStep = findExchangeStep(
                path, fromNode, currentEquiv, nextEquiv);

            if (!exchangeStep) {
                throw ValueError("Exchange step not found");
            }

            // Use updated rate if this is the affected exchange
            // NOTE: We need to track which node triggered the change
            // For simplicity, if updatedRate provided and equivalents match, use it
            if (updatedRate &&
                exchangeStep->sourceEquivalent == currentEquiv &&
                exchangeStep->targetEquivalent == nextEquiv) {

                currentAmount = applyExchangeForward(currentAmount, *updatedRate);
            } else {
                currentAmount = applyExchangeForward(
                    currentAmount, exchangeStep->exchangeRate);
            }
            continue;
        }

        // Check for commission (skip if at receiver)
        if (idx + 1 < path.ids.size() - 1) {
            const auto *commissionStep = findExchangeStep(
                path, toNode, nextEquiv, nextEquiv);

            if (commissionStep && commissionStep->commission > TrustLineAmount(0)) {
                TrustLineAmount commissionToApply;

                // Use updated commission if this is the affected node
                if (updatedCommission) {
                    commissionToApply = *updatedCommission;
                } else {
                    commissionToApply = commissionStep->commission;
                }

                if (currentAmount < commissionToApply) {
                    throw ValueError("Amount exhausted by commission");
                }

                currentAmount = currentAmount - commissionToApply;
            }
        }
    }

    return currentAmount;
}
```

### 6.2: Helper for applying exchange rate forward
```cpp
TrustLineAmount applyExchangeForward(
    const TrustLineAmount &amount,
    const ExchangeRate &rate)
{
    // Apply exchange rate: amount * rate
    // Implementation depends on ExchangeRate structure
    // Example:
    cpp_int result = amount.convert_to<cpp_int>() * rate.value();
    if (rate.shift() > 0) {
        result *= pow10(rate.shift());
    } else if (rate.shift() < 0) {
        result /= pow10(-rate.shift());
    }
    return result.convert_to<TrustLineAmount>();
}
```

## Step 7: Add Method Signatures to Header

**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

### 7.1: Add private methods
```cpp
private:
    TransactionResult::SharedConst handleConditionChange(
        const PathID &pathID,
        const optional<ExchangeRate> &actualExchangeRate,
        const optional<TrustLineAmount> &actualCommission);

    void updateExchangeRateManager(
        OptimalPathResult *pathStats,
        const ExchangeRate &newRate);

    void updateCommissionManager(
        OptimalPathResult *pathStats,
        const TrustLineAmount &newCommission);

    TrustLineAmount calculateReceivedAmountWithUpdatedConditions(
        OptimalPathResult *pathStats,
        const TrustLineAmount &inputFlow,
        const optional<ExchangeRate> &updatedRate,
        const optional<TrustLineAmount> &updatedCommission);
```

# Test Plan

## Test Scope
Coordinator detection of condition changes, manager updates, and new path creation with correct recalculation. This does NOT include subsequent paths recalculation (Task 09-04) or full integration testing (Task 09-05).

## Unit Tests

### Condition Population Tests
**Test 21a**: askRemoteNodeToApproveReservation populates exchange rate
- Setup: Path with exchanger node at remote position
- Expected: CoordinatorReservationRequestMessage contains expectedExchangeRate

**Test 21b**: askNeighborToApproveFurtherNodeReservation populates exchange rate
- Setup: Path with exchanger node at neighbor position
- Expected: CoordinatorReservationRequestMessage contains expectedExchangeRate

**Test 21c**: Commission population in both methods
- Setup: Paths with commission nodes
- Expected: expectedCommission populated correctly

### Exchange Rate Change Handling Tests
**Test 21**: Receive RejectedDueConditionsChanged with higher exchange rate
- Setup: Path with rate 0.5, response with rate 0.6
- Expected: Path invalidated, ExchangeRatesManager updated, new path created

**Test 22**: Receive RejectedDueConditionsChanged with lower exchange rate
- Setup: Path with rate 0.5, response with rate 0.4
- Expected: Path invalidated, ExchangeRatesManager updated, new path created

**Test 23**: New path flows recalculated correctly
- Setup: Condition change on path
- Expected: New path has correct flows from calculateFlows()

**Test 24**: Reservations dropped on invalidated path
- Setup: Path with reservations gets condition change
- Expected: dropReservationsOnPath() called, FinalPathExchangeConfigurationMessage sent

**Test 25**: Payment continues after adaptation
- Setup: Condition change handled
- Expected: tryProcessNextPath() called, payment continues

### Commission Change Handling Tests
**Test 26**: Receive RejectedDueConditionsChanged with higher commission
- Setup: Path with commission 5, response with commission 7
- Expected: Path invalidated, CommissionsManager updated, new path created

**Test 27**: Receive RejectedDueConditionsChanged with commission removed
- Setup: Path with commission 5, response with commission 0
- Expected: CommissionsManager removes commission, new path created

**Test 28**: New path created after commission change
- Setup: Commission change
- Expected: New path with correct received_amount

### Path Recalculation Tests
**Test 32**: calculateReceivedAmountWithUpdatedConditions with exchange rate
- Setup: Path with old rate, calculate with new rate
- Expected: Correct received_amount

**Test 33**: calculateReceivedAmountWithUpdatedConditions with commission
- Setup: Path with old commission, calculate with new commission
- Expected: Correct received_amount

**Test 34**: Flow recalculation via calculateFlows()
- Setup: New path with updated conditions
- Expected: flows vector correctly populated

## Success Criteria
- All 14 unit tests pass
- Expected conditions correctly populated in both message sending methods
- RejectedDueConditionsChanged detected in both response processing methods
- Managers updated correctly
- New paths created with correct values
- No compilation warnings

# Verification and Validation

## Architecture integrity
**Validation Level**: Complex task
- Follows existing reservation request/response patterns
- Manager updates use existing interfaces
- Path creation follows OptimalPathResult conventions
- No circular dependencies

## Security
**Validation Level**: Complex task
- Manager updates use validated data from responses
- No unchecked data propagation
- Proper bounds checking in amount calculations

## Performance
**Validation Level**: Complex task
- Condition change handling overhead: < 100ms
- Path recalculation: O(n) where n = path length
- Manager updates: O(log n) map operations
- No performance regression in happy path

## Scalability
**Validation Level**: Complex task
- Supports up to 50 paths with condition changes (PRD limit)
- Manager updates scale with number of nodes
- Path recalculation bounded by path length

## Reliability
**Validation Level**: Complex task
- Calculation errors don't crash transaction
- Manager update failures logged and continued
- Path creation errors handled gracefully

## Maintainability
**Validation Level**: Complex task
- Clear separation of concerns (detect, update, create)
- Helper methods improve readability
- Comprehensive logging for debugging

## Cost
**Validation Level**: Complex task
- No infrastructure cost changes

## Compliance
**Validation Level**: Complex task
- Implements all PRD acceptance criteria
- Follows project policy for transaction logic

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not implement subsequent paths recalculation (Task 09-04)
- Ensure logging is comprehensive for debugging
- Maintain transaction state consistency
