# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Path Capacity Adjustment for Exchange Payments
- **PRD ID**: 08
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.2
- **Date**: 2025-10-21
- **Author(s)**: Claude Code, based on Architect's requirements
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-10-21
- **Previous PRD**: [07-exchange-payment-topology-collection.md](07-exchange-payment-topology-collection.md)
- **Related Documents**:
  - [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
  - [Exchange Payment Topology Collection PRD](07-exchange-payment-topology-collection.md)
  - [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This PRD implements dynamic path capacity adjustment during exchange payment reservation when intermediate nodes return reservations with amounts lower than requested. Currently, `CoordinatorExchangePaymentTransaction` has a `shortageReservationsOnPath` method copied from single-equivalent `CoordinatorPaymentTransaction` that doesn't correctly handle multi-equivalent scenarios with exchanges and commissions. Similarly, `IntermediateNodeExchangePaymentTransaction` lacks logic to calculate adjusted incoming reservations when outgoing reservations are reduced.

### Current project state
- Exchange Payment with Commissions (PRD 06) implements multi-equivalent payment execution
- Exchange Payment Topology Collection (PRD 07) adds automatic path collection
- `CoordinatorExchangePaymentTransaction` uses `shortageReservationsOnPath` method that doesn't account for multi-equivalent path structure
- `IntermediateNodeExchangePaymentTransaction` has TODO (line 491) for calculating incoming reservation when outgoing is reduced
- Path building in `runPathsResourceProcessingStage` uses method `addPathForFurtherProcessing` that adds limited number of paths with capacity truncation

### This iteration's focus
- Fix `shortageReservationsOnPath` in `CoordinatorExchangePaymentTransaction` to correctly update `mMaxPathFlow` and `received_amount` accounting for exchanges and commissions
- Implement incoming reservation calculation in `IntermediateNodeExchangePaymentTransaction` when outgoing reservation is reduced
- Modify `runPathsResourceProcessingStage` to add ALL paths from `mPathsStats` without truncation
- Add path filtering before `switchToNextPath()` to remove paths with `mInaccessibleNodes` or `mRejectedTrustLines`
- Add capacity truncation check before taking next path into processing

### Connection to overall vision
This completes the robustness of exchange payment execution by properly handling capacity variations during reservation phase, ensuring payments succeed even when intermediate nodes have less available capacity than initially calculated.

## Iteration Context
### Previous Iterations Summary
- **PRD 04**: Exchange Flow Calculation implements topology collection and OR-Tools-based max flow calculation
- **PRD 05**: Payment Estimation provides bidirectional estimation using cached paths
- **PRD 06**: Exchange Payment with Commissions implements multi-equivalent payment execution with proper commission handling
- **PRD 07**: Exchange Payment Topology Collection adds automatic path availability checking and collection triggering
- **Completed Features**: Exchange rate management, optimal path calculation, exchange payment execution, automatic topology collection

### Lessons Learned
- Path capacity can vary between calculation and execution due to concurrent operations
- Single-equivalent reservation logic doesn't transfer directly to multi-equivalent scenarios
- Exchanges and commissions require inverse calculation when working backwards through path
- Early path filtering improves efficiency by avoiding processing of unusable paths

### Current State Analysis
- **What's working well**: Exchange payments execute successfully when all nodes reserve requested amounts
- **Pain points identified**:
  - Coordinator doesn't correctly adjust path capacity when nodes return smaller reservations
  - Intermediate nodes don't calculate incoming reservation adjustment
  - Path building adds only needed paths but may miss alternatives when early paths fail
  - No filtering of paths with known-bad nodes/trustlines before processing
- **Performance metrics**: Exchange payments succeed in ideal conditions but fail when capacity varies

## Problem Statement
### Background
During exchange payment reservation, intermediate nodes may reserve less than requested due to:
1. **Capacity limits**: Available outgoing capacity less than requested
2. **Concurrent operations**: Other transactions consuming capacity
3. **Commission constraints**: After applying commission, remaining amount insufficient
4. **Exchange limits**: Amount falls outside min/max exchange bounds

When this occurs, the coordinator must:
1. Reduce its own reservation
2. Update path statistics (`mMaxPathFlow`, `received_amount`)
3. Continue with reduced amount or try alternative paths

Additionally, intermediate nodes must:
1. Calculate required incoming reservation based on reduced outgoing
2. Account for exchanges (inverse calculation)
3. Account for commissions (add back if same equivalent)
4. Handle cases where exchange rate not found

Finally, path building strategy must:
1. Add all available paths (not just minimum needed)
2. Filter paths with known problematic nodes/trustlines
3. Truncate capacity only when actually taking path into processing

### Problem Description
**Who is affected**: Coordinators and intermediate nodes processing exchange payments with capacity variations

**When and where**:
- Coordinator: in `processRemoteNodeResponse` and `processNeighborFurtherReservationResponse` when receiving reservation response with reduced amount
- Intermediate node: in `runNextNeighborResponseProcessingStage` when receiving reduced outgoing reservation (line 491 TODO)
- Path building: in `runPathsResourceProcessingStage` / `addPathForFurtherProcessing` when selecting paths

**Current limitations**:
- `shortageReservationsOnPath` doesn't update `OptimalPathResult` fields (`mMaxPathFlow`, `optimal_flow`, `received_amount`, `flows`)
- No forward simulation to recalculate received amount through exchanges and commissions
- Path building truncates capacity during initial addition instead of during processing
- No filtering of paths with `mInaccessibleNodes` or `mRejectedTrustLines` before processing
- Coordinator has no way to estimate new received amount after capacity reduction

### Impact of not solving this problem
- Payments fail when intermediate nodes have slightly less capacity than calculated
- Network appears less reliable due to unnecessary payment failures
- Alternative paths not utilized when primary path has reduced capacity
- Processing cycles wasted on paths with known-bad nodes
- Inconsistent state between coordinator's view and actual path capacity

### Success Metrics
**Primary KPIs**:
- Successful payment adjustment when capacity reduced (target: 100% correct adjustment)
- Correct incoming reservation calculation on intermediate nodes (target: 100% accuracy)
- Improved payment success rate with capacity variations (target: +15% vs current)
- Reduced processing of bad paths (target: -50% attempts on filtered paths)

**Target Values**:
- 100% correct updates of ALL path fields (`mMaxPathFlow`, `optimal_flow`, `received_amount`, `flows`) after capacity reduction
- 100% correct `received_amount` calculation through exchanges and commissions
- Zero payment failures due to incorrect capacity adjustment
- All paths with `mInaccessibleNodes` filtered before processing

## Goals
The primary goals of this iteration are to enable robust capacity adjustment during exchange payment execution.

*   **Goal 1: Enable correct path capacity adjustment in coordinator.**
    *   **Description:** Update `shortageReservationsOnPath` to correctly adjust ALL path fields (`mMaxPathFlow`, `optimal_flow`, `received_amount`, `flows`) when node returns reduced reservation.
    *   **Success Metric:** Coordinator correctly updates all path capacity fields 100% of the time, accounting for exchanges and commissions.

*   **Goal 2: Implement incoming reservation calculation on intermediate nodes.**
    *   **Description:** Calculate required incoming reservation amount when outgoing is reduced, using inverse exchange rates and commission addition.
    *   **Success Metric:** Intermediate nodes correctly calculate incoming reservation 100% of the time, handling exchange rate lookups and commission logic.

*   **Goal 3: Improve path building strategy.**
    *   **Description:** Add all paths without initial truncation, filter bad paths before processing, truncate capacity only when taking path.
    *   **Success Metric:** Path processing efficiency improved by 50% (fewer attempts on filtered paths), payments succeed more often with alternative paths.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **Enhanced shortageReservationsOnPath**: Correct recalculation of ALL path fields (`mMaxPathFlow`, `optimal_flow`, `received_amount`, `flows`) with exchange and commission awareness
2. **Incoming reservation calculation**: New logic in IntermediateNodeExchangePaymentTransaction for calculating incoming amount from reduced outgoing
3. **Full path addition strategy**: Modified `addPathForFurtherProcessing` to add all paths without truncation
4. **Path filtering before processing**: Check for `mInaccessibleNodes` and `mRejectedTrustLines` in `switchToNextPath()`
5. **Capacity truncation at processing**: Move truncation logic from initial addition to pre-processing check

#### Technical Infrastructure
- Enhanced `shortageReservationsOnPath` with forward simulation and complete field recalculation
- New helper method `calculateIncomingAmountForOutgoing` in IntermediateNode
- Modified loop in `runPathsResourceProcessingStage` to add all paths
- Path validation in `switchToNextPath()` before taking path
- Capacity check and truncation before `switchToNextPath()`

#### Integration Points
- Integration with existing `OptimalPathResult::calculateFlows()` for inverse calculation
- Usage of `ExchangeRatesManager` for exchange rate lookups
- Usage of `CommissionsManager` for commission lookups
- Extension of path processing logic in coordinator

### Explicitly Out of Scope
- Automatic path invalidation after capacity changes (future work)
- Predictive capacity monitoring or preemptive path recalculation
- Advanced path selection strategies beyond filtering
- UI/API for capacity adjustment monitoring
- Distributed coordination of capacity changes

### Dependencies from Previous Iterations
- **PRD 06**: `CoordinatorExchangePaymentTransaction` and `IntermediateNodeExchangePaymentTransaction` implementations
- **PRD 06**: `OptimalPathResult` structure with `mMaxPathFlow`, `received_amount`, `flows` fields
- **PRD 06**: `ExchangePath` structure with exchanges and commissions
- **PRD 07**: Automatic path collection and processing flow

### Future Roadmap Impact
This iteration establishes foundation for:
- **Advanced capacity prediction**: Using historical adjustment patterns to predict capacity variations
- **Distributed capacity coordination**: Nodes communicating capacity changes proactively
- **Dynamic path rebalancing**: Redistributing flows across paths when capacity changes
- **Capacity reservation optimization**: Optimizing initial requests to minimize adjustments

## User Stories & Requirements

### User Personas
#### Primary User: Exchange Payment Coordinator
- **Role**: Node operator coordinating multi-equivalent payments
- **Goals**: Successfully complete payments even when capacity varies during execution
- **Pain Points**: Payments fail due to incorrect capacity adjustment; no visibility into adjustment logic
- **Technical Proficiency**: Advanced

#### Secondary User: Intermediate Node Operator
- **Role**: Network participant providing exchange and routing services
- **Goals**: Correctly process reservations and adjustments without errors
- **Pain Points**: TODO comment at line 491 indicates incomplete logic
- **Technical Proficiency**: Advanced

### Functional Requirements
#### New Features for This Iteration

1. **Enhanced shortageReservationsOnPath in CoordinatorExchangePaymentTransaction**
   - **Description**: Fix method to correctly adjust ALL path fields (`mMaxPathFlow`, `optimal_flow`, `received_amount`, `flows`) when node returns reduced reservation
   - **User Story**: As a payment coordinator, I need to correctly adjust path capacity when intermediate node reserves less than requested
   - **Rationale**: Current implementation from single-equivalent transaction doesn't account for exchanges and commissions
   - **Builds Upon**: Existing `shortageReservationsOnPath` method and path structure
   - **Acceptance Criteria**:
     - Method signature: `void shortageReservationsOnPath(ContractorID neighborID, const PathID &pathID, const TrustLineAmount &kNewAmount)`
     - Find path in `mPathsStats` by `pathID`
     - Update coordinator's own reservation to `kNewAmount` (existing logic)
     - Calculate new `received_amount` by forward-simulating path from `kNewAmount` through all exchanges and commissions
     - Update ALL path statistics fields:
       - `pathStats->mMaxPathFlow` = `kNewAmount`
       - `pathStats->optimal_flow` = `kNewAmount`
       - `pathStats->received_amount` = calculated value from simulation
       - `pathStats->flows` = recalculated via `pathStats->calculateFlows(kNewAmount)`
     - Recalculation uses forward simulation through path structure
     - Handle exchange rate not found (log warning, keep old values or mark path unusable)
     - Handle commission overflow (where kNewAmount < sum of commissions)
     - Called from: `processRemoteNodeResponse`, `processNeighborFurtherReservationResponse`
   - **Priority**: High
   - **Dependencies**: `OptimalPathResult` structure with `mMaxPathFlow` and `received_amount` fields

2. **Incoming Reservation Calculation in IntermediateNodeExchangePaymentTransaction**
   - **Description**: Implement TODO at line 491 to calculate required incoming reservation when outgoing is reduced
   - **User Story**: As an intermediate node, I need to calculate how much incoming reservation I need when my outgoing reservation is reduced
   - **Rationale**: Maintains balance of reservations accounting for exchanges and commissions
   - **Builds Upon**: Existing reservation structure and exchange/commission infrastructure
   - **Acceptance Criteria**:
     - Replace TODO comment at line 491 in `runNextNeighborResponseProcessingStage`
     - Find incoming reservation by same PathID
     - Get incoming reservation equivalent from reservation object
     - Get outgoing reservation equivalent from reservation object
     - **If equivalents same**:
       - Check if commission exists for this equivalent (via `mCommissionsManager->get(equivalent)`)
       - If no commission: `incomingAmount = outgoingAmount`
       - If commission exists: `incomingAmount = outgoingAmount + commission->amount()`
     - **If equivalents different**:
       - Look up exchange rate: `mExchangeRatesManager->get(incomingEquiv, outgoingEquiv)`
       - If rate not found: send error message to coordinator, terminate transaction
       - Calculate: `incomingAmount = inverseExchangeCalculation(outgoingAmount, exchangeRate)`
       - Use ceiling division to avoid rounding down (favor higher incoming to cover outgoing)
     - Call `shortageIncomingReservationsOnPath(pathID, incomingEquivalent, incomingAmount)`
     - Handle errors: if calculation fails, send error to coordinator and terminate
   - **Priority**: High
   - **Dependencies**: `ExchangeRatesManager`, `CommissionsManager`, existing `shortageIncomingReservationsOnPath` method

3. **Add All Paths Without Truncation in runPathsResourceProcessingStage**
   - **Description**: Modify path addition loop to add ALL paths from ExchangePathsManager without capacity truncation
   - **User Story**: As a payment coordinator, I want all available paths added so alternative paths are available if primary paths fail
   - **Rationale**: Current strategy truncates last path to exact needed capacity, limiting flexibility
   - **Builds Upon**: Existing `runPathsResourceProcessingStage` and `addPathForFurtherProcessing`
   - **Acceptance Criteria**:
     - In `runPathsResourceProcessingStage`, remove early break when `totalAddedFlow >= mAmount`
     - Change loop to iterate through ALL paths from `cachedPaths` for each sender equivalent
     - Call `addPathForFurtherProcessing(pathResult, pathResult.received_amount)` with full path capacity
     - Remove any capacity truncation logic in `addPathForFurtherProcessing` (lines 496-497 reference)
     - Result: ALL optimal paths added to `mPathsStats` with their full `received_amount` capacity
   - **Priority**: High
   - **Dependencies**: ExchangePathsManager cached paths structure

4. **Capacity Truncation Before Taking Next Path**
   - **Description**: Add capacity check and truncation in `switchToNextPath()` before actually processing path
   - **User Story**: As a payment coordinator, I want to use only the needed capacity from each path to avoid over-reservation
   - **Rationale**: Truncation should happen at processing time, not at addition time
   - **Builds Upon**: Existing `switchToNextPath()` method
   - **Acceptance Criteria**:
     - Before calling `switchToNextPath()` (or at beginning of method), calculate `remainingNeeded = mAmount - totalReservedSoFar`
     - Get next path's `received_amount` (or `mMaxPathFlow` for sending side)
     - If `received_amount > remainingNeeded`:
       - Truncate: call `pathStats->shortageMaxFlow(truncatedAmount)` where `truncatedAmount` calculated to deliver exactly `remainingNeeded`
       - Use inverse calculation: find input amount that delivers `remainingNeeded` output
       - Recalculate `flows` if needed: call `pathStats->calculateFlows(truncatedAmount)`
     - Then proceed with path processing
   - **Priority**: Medium
   - **Dependencies**: Path capacity calculation logic

5. **Path Filtering in switchToNextPath**
   - **Description**: Filter out paths containing nodes from `mInaccessibleNodes` or trust lines from `mRejectedTrustLines`
   - **User Story**: As a payment coordinator, I want to skip paths that are known to be problematic before wasting processing cycles
   - **Rationale**: Improves efficiency by avoiding known-bad paths
   - **Builds Upon**: Existing path processing and bad node tracking
   - **Acceptance Criteria**:
     - In `switchToNextPath()` (or before taking next path), validate next path:
       - Check if any node in `path.nodes` (BaseAddress::Shared) appears in `mInaccessibleNodes`
       - Check if any edge `(nodes[i], nodes[i+1])` appears in `mRejectedTrustLines`
       - Note: Use `path.nodes` for filtering (populated by `addPathForFurtherProcessing` early in transaction)
     - If validation fails:
       - Mark path as unusable: `pathStats->setUnusable()`
       - Remove from processing queue
       - Move to next path in `mPathIDs`
     - Repeat validation until finding valid path or exhausting all paths
     - Precondition: `path.nodes` must be populated (guaranteed by `addPathForFurtherProcessing`)
   - **Priority**: Medium
   - **Dependencies**: Existing `mInaccessibleNodes` and `mRejectedTrustLines` tracking

#### Enhancements to Existing Features

1. **OptimalPathResult Field Updates**
   - **Current State**: Fields `mMaxPathFlow` and `received_amount` set during path calculation, not updated during reservation
   - **Proposed Changes**: Enable runtime updates of these fields when capacity adjusts
   - **Impact Assessment**: Pure enhancement, no breaking changes
   - **Migration Strategy**: Direct field updates in shortage methods

2. **shortageIncomingReservationsOnPath Method**
   - **Current State**: Method exists in `IntermediateNodeExchangePaymentTransaction` (lines 63-72 in .h)
   - **Proposed Changes**: Will be called with calculated incoming amount
   - **Impact Assessment**: No changes to method itself, just new caller
   - **Migration Strategy**: Use existing method as-is

3. **shortageOutgoingReservationsOnPath Method**
   - **Current State**: Method exists in `IntermediateNodeExchangePaymentTransaction` (lines 68-72 in .h)
   - **Proposed Changes**: Currently called, continues to work as before
   - **Impact Assessment**: No changes needed
   - **Migration Strategy**: No migration

### Non-Functional Requirements
#### Performance
- Path capacity adjustment overhead < 5ms per path
- Inverse calculation through exchanges < 10ms per path
- Path filtering overhead < 1ms per path check
- Total adjustment overhead < 50ms for typical payment (5-10 paths)

#### Security
- No security implications (internal calculation only)
- No exposure of internal path structure to external parties
- Consistent error handling for exchange rate lookup failures

#### Scalability
- Support adjustment for up to 50 paths per payment
- Handle up to 10 exchanges per path
- Efficient path filtering for up to 100 inaccessible nodes
- Support up to 500 rejected trust lines

#### Reliability
- Adjustment failures don't crash transaction (log and continue or mark path unusable)
- Exchange rate lookup failures handled gracefully (error message to coordinator)
- Commission overflow handled (log warning, mark path unusable)
- Path filtering failures handled (skip filter, log warning)

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Coordinator adjusts own reservation but doesn't update path capacity; intermediate nodes don't calculate incoming adjustment
- **Proposed Changes**: Full capacity adjustment chain from coordinator through intermediate nodes with inverse calculation
- **Backwards Compatibility**: No protocol changes; internal logic enhancement only
- **Migration Requirements**: None (runtime behavior change only)

### Technology Stack Updates
#### New Technologies/Libraries
- No new external libraries required
- Reuses existing calculation helpers and exchange/commission infrastructure

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- Enhanced integration between `shortageReservationsOnPath` and `OptimalPathResult` fields
- New integration between intermediate node and `ExchangeRatesManager` / `CommissionsManager` for incoming calculation

#### Modified Integrations
- Modified path processing loop in `runPathsResourceProcessingStage`
- Modified `switchToNextPath` behavior with filtering and truncation

### Data Requirements
#### Data Models

No new data models needed. Existing structures used:
- `OptimalPathResult` (fields `mMaxPathFlow`, `received_amount`, `mPath`, `flows`)
- `ExchangePath` (fields `ids`, `nodes`, `equivalents`, `exchangeSteps`)
  - **Important distinction**:
    - `ids` (vector<ContractorID>): Used for iteration and calculations (exchanges, commissions)
    - `nodes` (vector<BaseAddress::Shared>): Used for filtering (inaccessible nodes, rejected trust lines)
    - `nodes` is populated by `addPathForFurtherProcessing` early in transaction lifecycle
    - At filtering stage (during reservation), `nodes` is guaranteed to be populated
- Reservation structures (PathID, equivalent, amount)

#### Data Storage
- No persistent storage changes
- Runtime-only field updates in transaction state

#### Data Migration
- No migration needed (internal logic change)

### Algorithm Specifications

#### Enhanced shortageReservationsOnPath in Coordinator

**Purpose**: Adjust path capacity when intermediate node returns reduced reservation amount

**Important**: This algorithm uses `path.ids` and `path.equivalents` for iteration and calculations (exchanges, commissions), NOT `path.nodes`. The `ids` field contains ContractorID values needed for `findExchangeStep` lookups.

**Algorithm**:
```cpp
void CoordinatorExchangePaymentTransaction::shortageReservationsOnPath(
    ContractorID neighborID,
    const PathID &pathID,
    const TrustLineAmount &kNewAmount)
{
    debug() << "shortageReservationsOnPath: pathID=" << pathID
            << ", neighborID=" << neighborID
            << ", newAmount=" << kNewAmount;

    // Step 1: Update coordinator's own reservation (existing logic)
    auto nodeReservations = mReservations[neighborID];
    for (const auto &pathIDAndReservation : nodeReservations) {
        if (pathIDAndReservation.first == pathID) {
            const auto equivalent = pathIDAndReservation.second->equivalent();
            shortageReservation(
                neighborID,
                pathIDAndReservation.second,
                kNewAmount,
                pathID,
                equivalent);
            break;  // coordinator has only one reservation per path
        }
    }

    // Step 2: Find path in mPathsStats
    auto pathStatsIt = mPathsStats.find(pathID);
    if (pathStatsIt == mPathsStats.end()) {
        warning() << "Path not found in mPathsStats for pathID=" << pathID;
        return;
    }

    OptimalPathResult *pathStats = pathStatsIt->second.get();
    const auto &path = pathStats->path();

    // Step 3: Calculate new received_amount by forward-simulating through path
    // The kNewAmount is the amount coordinator sends to first node (in sender equivalent)
    // We need to calculate what receiver gets (in receiver equivalent)

    TrustLineAmount newReceivedAmount;
    try {
        // Forward simulate through path: apply exchanges and subtract commissions
        // This uses the same logic as calculateFlows but simpler
        // Note: We use path.ids and path.equivalents for iteration (NOT path.nodes)
        // because findExchangeStep expects ContractorID, not BaseAddress

        TrustLineAmount currentAmount = kNewAmount;

        for (size_t idx = 0; idx + 1 < path.ids.size(); ++idx) {
            const ContractorID fromNode = path.ids[idx];
            const ContractorID toNode = path.ids[idx + 1];
            const SerializedEquivalent currentEquiv = path.equivalents[idx];
            const SerializedEquivalent nextEquiv = path.equivalents[idx + 1];

            // Check for exchange (same node, different equivalent)
            if (fromNode == toNode && currentEquiv != nextEquiv) {
                // Find exchange step
                const auto *exchangeStep = findExchangeStep(
                    path, fromNode, currentEquiv, nextEquiv);

                if (!exchangeStep) {
                    warning() << "Exchange step not found during shortage calculation";
                    return;  // Keep old values
                }

                // Apply exchange forward
                currentAmount = applyExchangeForward(currentAmount, *exchangeStep);
                continue;  // Don't add to flows, this is in-place exchange
            }

            // Check for commission at arrival node (if not receiver)
            if (idx + 1 < path.ids.size() - 1) {  // Not last node (receiver)
                const auto *commissionStep = findExchangeStep(
                    path, toNode, nextEquiv, nextEquiv);

                if (commissionStep && commissionStep->commission > TrustLineAmount(0)) {
                    if (currentAmount < commissionStep->commission) {
                        warning() << "Amount exhausted by commission during shortage, "
                                  << "marking path unusable";
                        pathStats->setUnusable();
                        return;
                    }
                    currentAmount = currentAmount - commissionStep->commission;
                }
            }
        }

        newReceivedAmount = currentAmount;

    } catch (const std::exception &e) {
        warning() << "Error calculating new received amount: " << e.what();
        return;  // Keep old values
    }

    // Step 4: Update ALL path statistics fields
    pathStats->mMaxPathFlow = kNewAmount;
    pathStats->optimal_flow = kNewAmount;
    pathStats->received_amount = newReceivedAmount;

    // Step 5: Recalculate flows vector
    try {
        pathStats->calculateFlows(kNewAmount);
    } catch (const std::exception &e) {
        warning() << "Error recalculating flows: " << e.what();
        // flows may be inconsistent, but main fields are updated
    }

    info() << "Path capacity adjusted: pathID=" << pathID
           << ", newMaxFlow=" << kNewAmount
           << ", newOptimalFlow=" << kNewAmount
           << ", newReceivedAmount=" << newReceivedAmount
           << " (was " << pathStats->received_amount << ")"
           << ", flows recalculated";
}
```

**Key Points**:
- Updates coordinator's own reservation (existing logic preserved)
- Finds path in `mPathsStats`
- Forward-simulates through path to calculate new `received_amount`
- **Uses `path.ids` and `path.equivalents` for iteration** (NOT `path.nodes`)
  - Reason: `findExchangeStep` expects `ContractorID`, not `BaseAddress`
  - `path.ids` always populated; `path.nodes` is optional in general case
- Uses same exchange/commission logic as `calculateFlows`
- Updates ALL fields: `mMaxPathFlow`, `optimal_flow`, `received_amount`
- Recalculates `flows` vector via `calculateFlows(kNewAmount)`
- Handles errors gracefully (logs warning, keeps old values or marks unusable)
- Called from: `processRemoteNodeResponse` when remote node returns reduced amount
- Called from: `processNeighborFurtherReservationResponse` when neighbor returns reduced amount for further propagation

#### Incoming Reservation Calculation in Intermediate Node

**Purpose**: Calculate required incoming reservation amount when outgoing reservation is reduced

**Location**: `IntermediateNodeExchangePaymentTransaction::runNextNeighborResponseProcessingStage()`, replacing TODO at line 491

**Algorithm**:
```cpp
// This code replaces the TODO comment at line 491

// Step 1: Get outgoing reservation details
const TrustLineAmount outgoingAmount = message->amount();
const SerializedEquivalent outgoingEquiv = /* get from outgoing reservation */;
const PathID pathID = /* get from context */;

// Step 2: Find incoming reservation by same PathID
AmountReservation::ConstShared incomingReservation = nullptr;
SerializedEquivalent incomingEquiv;

for (const auto &[contractorID, reservationsMap] : mReservations) {
    for (const auto &[resPathID, reservation] : reservationsMap) {
        if (resPathID == pathID &&
            reservation->direction() == AmountReservation::Incoming) {
            incomingReservation = reservation;
            incomingEquiv = reservation->equivalent();
            break;
        }
    }
    if (incomingReservation) break;
}

if (!incomingReservation) {
    error() << "Incoming reservation not found for pathID=" << pathID;
    return sendErrorMessageOnNextNodeResponse(
        ResponseMessage::Rejected);
}

// Step 3: Calculate required incoming amount
TrustLineAmount incomingAmount;

if (incomingEquiv == outgoingEquiv) {
    // Same equivalent: check for commission
    auto commission = mCommissionsManager->get(incomingEquiv);

    if (commission && commission->amount() > TrustLineAmount(0)) {
        // Add commission to outgoing to get incoming
        incomingAmount = outgoingAmount + commission->amount();

        debug() << "Incoming amount with commission: "
                << "outgoing=" << outgoingAmount
                << ", commission=" << commission->amount()
                << ", incoming=" << incomingAmount;
    } else {
        // No commission: amounts equal
        incomingAmount = outgoingAmount;

        debug() << "Incoming amount (no commission): " << incomingAmount;
    }
} else {
    // Different equivalents: need exchange rate
    auto exchangeRate = mExchangeRatesManager->get(incomingEquiv, outgoingEquiv);

    if (!exchangeRate) {
        error() << "Exchange rate not found: "
                << incomingEquiv << " -> " << outgoingEquiv;
        return sendErrorMessageOnNextNodeResponse(
            ResponseMessage::Rejected);
    }

    // Calculate incoming amount using inverse exchange
    // outgoingAmount = incomingAmount * rate * 10^shift
    // incomingAmount = outgoingAmount / (rate * 10^shift)
    // Use ceiling division to favor higher incoming

    try {
        incomingAmount = invertExchangeForRequiredInput(
            outgoingAmount,
            exchangeRate->exchangeRate(),
            exchangeRate->exchangeRateShift());

        debug() << "Incoming amount via exchange: "
                << "outgoing=" << outgoingAmount << " (eq " << outgoingEquiv << ")"
                << ", incoming=" << incomingAmount << " (eq " << incomingEquiv << ")"
                << ", rate=" << exchangeRate->exchangeRate()
                << ", shift=" << exchangeRate->exchangeRateShift();

    } catch (const std::exception &e) {
        error() << "Error calculating incoming amount: " << e.what();
        return sendErrorMessageOnNextNodeResponse(
            ResponseMessage::Rejected);
    }
}

// Step 4: Update incoming reservation
shortageIncomingReservationsOnPath(pathID, incomingEquiv, incomingAmount);

info() << "Incoming reservation adjusted: pathID=" << pathID
       << ", incomingAmount=" << incomingAmount
       << ", incomingEquiv=" << incomingEquiv;
```

**Helper Function** (if not exists):
```cpp
TrustLineAmount invertExchangeForRequiredInput(
    const TrustLineAmount &outputAmount,
    const TrustLineAmount &exchangeRate,
    int16_t exchangeRateShift)
{
    if (exchangeRate == TrustLineAmount(0)) {
        throw ValueError("Zero exchange rate");
    }

    // outputAmount = inputAmount * rate * 10^shift
    // inputAmount = outputAmount / (rate * 10^shift)

    cpp_int numerator = cpp_int(outputAmount);
    cpp_int denominator = cpp_int(exchangeRate);

    if (exchangeRateShift >= 0) {
        denominator *= pow10(static_cast<size_t>(exchangeRateShift));
    } else {
        numerator *= pow10(static_cast<size_t>(-exchangeRateShift));
    }

    // Ceiling division
    return ceilDivideToAmount(numerator, denominator);
}
```

**Key Points**:
- Finds incoming reservation by PathID
- Checks equivalent match
- If same: adds commission (if exists)
- If different: looks up exchange rate and inverts
- Uses ceiling division to favor higher incoming (ensures outgoing covered)
- Calls existing `shortageIncomingReservationsOnPath`
- Handles errors: exchange rate not found → error message to coordinator

#### Modified runPathsResourceProcessingStage

**Purpose**: Add all paths without capacity truncation during initial addition

**Changes**:
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::runPathsResourceProcessingStage()
{
    // ... [existing mExchangeAmount calculation] ...

    // Step 1: Add ALL paths from ExchangePathsManager (no truncation)
    for (const auto& exchangeEquiv : mExchangeEquivalents) {
        PathCacheKey key{mContractorID, exchangeEquiv, mEquivalent};
        auto cachedPaths = mExchangePathsManager->retrievePaths(key);

        if (!cachedPaths) {
            debug() << "No cached paths for exchangeEquiv=" << exchangeEquiv;
            continue;
        }

        // Add ALL paths with their full capacity (removed break on totalAddedFlow >= mAmount)
        for (const auto& pathResult : *cachedPaths) {
            // Add with full received_amount capacity (no truncation)
            addPathForFurtherProcessing(pathResult, pathResult.received_amount);
        }
    }

    // Step 2: Reduce timeout from maxNetworkDelay(10) to maxNetworkDelay(4)
    return resultWaitForMessageTypes(
        {Message::Payments_ReceiverInitPaymentResponse},
        maxNetworkDelay(4));
}
```

**Key Changes**:
- Removed early break when `totalAddedFlow >= mAmount`
- Changed to add ALL paths from all equivalents
- Pass `pathResult.received_amount` as second parameter (full capacity)
- Removed any truncation logic (lines 496-497 reference removed)

#### Path Filtering and Capacity Truncation in switchToNextPath

**Purpose**: Filter bad paths and truncate capacity before processing

**Important**: This algorithm uses `path.nodes` (BaseAddress::Shared) for filtering, NOT `path.ids`. The `nodes` field is populated by `addPathForFurtherProcessing` early in transaction and is guaranteed to be available at reservation stage.

**Helper Functions Used** (described below algorithm):
- `calculateTotalReservedAmount()`: Calculates total amount already reserved across all paths
- `proceedToNextStage()`: Transitions transaction to next stage after reservation complete
- `tryProcessNextPath()`: Attempts to process next available path, handling path not found scenarios
- `calculateRequiredInputForPath()`: Inverse calculation to find input needed for desired output

**Algorithm** (pseudo-code, exact location TBD):
```cpp
// Before calling switchToNextPath() or at beginning of method

// Step 1: Check if more capacity needed
TrustLineAmount totalReserved = calculateTotalReservedAmount();
if (totalReserved >= mAmount) {
    // Sufficient capacity reserved, no more paths needed
    return proceedToNextStage();
}

TrustLineAmount remainingNeeded = mAmount - totalReserved;

// Step 2: Get next path
PathID nextPathID = mPathIDs[mCurrentPathIndex];  // or however next path determined
auto pathStatsIt = mPathsStats.find(nextPathID);
if (pathStatsIt == mPathsStats.end()) {
    warning() << "Next path not found";
    return tryProcessNextPath();
}

OptimalPathResult *pathStats = pathStatsIt->second.get();

// Step 3: Filter path for bad nodes/trustlines
bool pathValid = true;
const auto &path = pathStats->path();

// Important: Use path.nodes for filtering (BaseAddress::Shared)
// path.nodes is populated by addPathForFurtherProcessing and guaranteed to be available here

// Check for inaccessible nodes
for (const auto &nodeAddress : path.nodes) {
    if (std::find(mInaccessibleNodes.begin(),
                  mInaccessibleNodes.end(),
                  nodeAddress) != mInaccessibleNodes.end()) {
        info() << "Path contains inaccessible node: "
               << nodeAddress->fullAddress();
        pathValid = false;
        break;
    }
}

// Check for rejected trust lines
if (pathValid) {
    for (size_t i = 0; i + 1 < path.nodes.size(); ++i) {
        auto source = path.nodes[i];
        auto dest = path.nodes[i + 1];

        for (const auto &[rejSource, rejDest] : mRejectedTrustLines) {
            if (source == rejSource && dest == rejDest) {
                info() << "Path contains rejected trust line: "
                       << source->fullAddress() << " -> "
                       << dest->fullAddress();
                pathValid = false;
                break;
            }
        }
        if (!pathValid) break;
    }
}

// Step 4: Handle invalid path
if (!pathValid) {
    pathStats->setUnusable();
    // Remove from processing or skip to next
    return tryProcessNextPath();
}

// Step 5: Truncate capacity if path exceeds remaining need
if (pathStats->received_amount > remainingNeeded) {
    info() << "Truncating path capacity: "
           << "available=" << pathStats->received_amount
           << ", needed=" << remainingNeeded;

    // Calculate truncated input amount that delivers exactly remainingNeeded
    // Use inverse calculation through path
    TrustLineAmount truncatedInput = calculateRequiredInputForPath(
        *pathStats, remainingNeeded);

    // Update path capacity
    pathStats->shortageMaxFlow(truncatedInput);
    pathStats->received_amount = remainingNeeded;

    // Recalculate flows if needed for reservation
    pathStats->calculateFlows(truncatedInput);
}

// Step 6: Proceed with path processing
switchToNextPath();  // or actual processing
```

**Helper Functions Descriptions**:

1. **`calculateTotalReservedAmount()`**
   - **Purpose**: Calculate total amount already successfully reserved across all processed paths
   - **Implementation approach**:
     ```cpp
     TrustLineAmount calculateTotalReservedAmount() {
         TrustLineAmount total = TrustLineAmount(0);
         for (const auto &[pathID, pathStats] : mPathsStats) {
             // Sum received_amount for all successfully reserved paths
             // Only count paths where reservation is approved/completed
             if (pathStats->isLastIntermediateNodeApproved() ||
                 /* other completion criteria */) {
                 total = total + pathStats->received_amount;
             }
         }
         return total;
     }
     ```
   - **Returns**: Total amount reserved in receiver equivalent (mEquivalent)

2. **`proceedToNextStage()`**
   - **Purpose**: Transition to next transaction stage after sufficient capacity reserved
   - **Implementation approach**:
     ```cpp
     TransactionResult::SharedConst proceedToNextStage() {
         // Transition to final amounts configuration or next reservation stage
         mStep = Coordinator_FinalAmountsConfiguration;  // or appropriate next stage
         return sendFinalAmountsConfigurationToAllParticipants();
     }
     ```
   - **Returns**: TransactionResult for next stage

3. **`tryProcessNextPath()`**
   - **Purpose**: Attempt processing next available path, handling edge cases
   - **Implementation approach**:
     ```cpp
     TransactionResult::SharedConst tryProcessNextPath() {
         // Existing method in CoordinatorExchangePaymentTransaction
         // Moves to next PathID in mPathIDs vector
         // If no more paths: check if sufficient capacity reserved
         // If insufficient: return error (no paths available)
         // If more paths: continue with next path processing
     }
     ```
   - **Returns**: TransactionResult for continuing path processing or error

4. **`calculateRequiredInputForPath(pathStats, desiredOutput)`**
   - **Purpose**: Calculate required input amount to achieve desired output, working backwards through path
   - **Implementation approach**:
     ```cpp
     TrustLineAmount calculateRequiredInputForPath(
         const OptimalPathResult &pathResult,
         const TrustLineAmount &desiredOutputAmount)
     {
         // Start from receiver end with desiredOutputAmount
         // Work backwards: add commissions, invert exchanges
         // Return required input amount at sender end
         // Similar to existing helper in CoordinatorExchangePaymentTransaction.cpp
     }
     ```
   - **Parameters**:
     - `pathResult`: Path structure with exchanges and commissions
     - `desiredOutputAmount`: Amount needed at receiver (in receiver equivalent)
   - **Returns**: Required input amount at sender (in sender equivalent)

**Key Points**:
- **Uses `path.nodes` (BaseAddress::Shared) for filtering** (NOT `path.ids`)
  - Reason: `mInaccessibleNodes` and `mRejectedTrustLines` store BaseAddress::Shared
  - `path.nodes` populated by `addPathForFurtherProcessing` early in transaction
  - Guaranteed to be available at reservation stage (filtering happens during reservation)
- Checks for inaccessible nodes in `path.nodes`
- Checks for rejected trust lines in edges `(path.nodes[i], path.nodes[i+1])`
- Marks path unusable if invalid, moves to next via `tryProcessNextPath()`
- Truncates capacity only if path exceeds remaining need (calculated via `calculateTotalReservedAmount()`)
- Uses `calculateRequiredInputForPath()` for inverse calculation to find input for desired output
- Updates path fields and recalculates flows

### Error Handling Specifications

#### Error Conditions
1. **Exchange rate not found during incoming calculation (Intermediate Node)**:
  - Send error message to coordinator: `ResponseMessage::Rejected`
   - Terminate transaction
   - Log: `error() << "Exchange rate not found: " << incomingEquiv << " -> " << outgoingEquiv;`

2. **Commission overflow in coordinator shortage**:
   - Mark path unusable: `pathStats->setUnusable()`
   - Log warning: `warning() << "Amount exhausted by commission during shortage"`
   - Continue with next path

3. **Path not found in mPathsStats during shortage**:
   - Log warning: `warning() << "Path not found in mPathsStats for pathID=" << pathID;`
   - Return early (no update)
   - Continue transaction

4. **Calculation error during received amount update**:
   - Catch exception
   - Log warning: `warning() << "Error calculating new received amount: " << e.what();`
   - Keep old values
   - Continue with other paths

5. **Invalid path structure during filtering**:
   - Mark path unusable
   - Move to next path
   - Log: `info() << "Path contains inaccessible node/trustline"`

6. **Truncation calculation fails**:
   - Log error
   - Mark path unusable
   - Move to next path
   - If no more paths available, terminate with error

## Implementation Plan
### This Iteration Timeline
- **Duration**: 2-3 weeks implementation + 1 week testing
- **Sprint Breakdown**:
  - Sprint 1 (Week 1): Enhanced `shortageReservationsOnPath` in coordinator with inverse calculation
  - Sprint 2 (Week 1-2): Incoming reservation calculation in intermediate node (TODO line 491)
  - Sprint 3 (Week 2): Modified path addition strategy (add all paths)
  - Sprint 4 (Week 2-3): Path filtering and capacity truncation in `switchToNextPath`
  - Sprint 5 (Week 3): Integration testing and edge case validation

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Coordinator Shortage Fixed | Week 1 | Enhanced `shortageReservationsOnPath` with correct calculations | None | Medium |
| Intermediate Incoming Calculation | Week 2 | TODO line 491 implemented with exchange/commission logic | Coordinator shortage | Medium |
| Path Addition Strategy Modified | Week 2 | All paths added without truncation | None | Low |
| Path Filtering Implemented | Week 3 | Bad path filtering and capacity truncation in `switchToNextPath` | Path addition | Low |
| Integration Testing Complete | Week 3 | All components working together | All previous | Medium |

### Dependencies on Other Teams/Projects
- No external team dependencies identified

### Integration Points with Previous Work
- Builds upon `OptimalPathResult` structure from PRD 06
- Uses `shortageIncomingReservationsOnPath` from PRD 06
- Extends path processing from PRD 07
- Uses `ExchangeRatesManager` and `CommissionsManager` from PRD 03-04

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++ and payment transaction experience
- **Developers**: 1 developer for implementation support
- **QA Engineers**: 1 engineer for unit testing

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Inverse calculation errors | High | Medium | Thorough unit testing with known examples; validate against forward calculation |
| Commission overflow edge cases | Medium | Low | Explicit checks and path marking unusable when detected |
| Exchange rate lookup failures | Medium | Medium | Proper error handling with coordinator notification |
| Path filtering logic errors | Low | Low | Simple validation logic; extensive testing |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Adjustment still not handling all cases | Medium | Low | Comprehensive test coverage; real-world testing scenarios |

## Testing Strategy
### Testing Approach (Unit-Only)
- All testing is unit-only (no integration/E2E tests)
- Tests are built and executed exclusively in `build-tests`
- Use real objects following ExchangePathsManagerTest pattern
- Use TestEnvironment helper classes for consistent test setup
- Test files located in `tests/unit/transactions/` subdirectory

### Testing Best Practices
- **Real Objects Over Mocks**: Use real instances of managers and components
- **TestEnvironment Helpers**: Create helper classes for test initialization
- **Exception Testing**: Test that methods throw correct exceptions using EXPECT_THROW
- **Edge Case Coverage**: Test boundary values, overflow, missing data
- **Parameterized Tests**: Use for testing different equivalent/commission combinations

#### Unit Tests: New/Modified Components

**1. CoordinatorExchangePaymentTransaction::shortageReservationsOnPath Tests**

*Test 1: Shortage on simple path without exchanges or commissions*
- **Setup**:
  - Path: A→B→C→D (all in eq 1001)
  - Initial `mMaxPathFlow`: 1000, `optimal_flow`: 1000, `received_amount`: 1000
  - Node B returns reservation: 700 (coordinator requested 1000)
- **Execution**: Call `shortageReservationsOnPath(B_id, pathID, 700)`
- **Expected**:
  - Coordinator's own reservation reduced to 700 via `shortageReservation`
  - `pathStats->mMaxPathFlow` updated to 700
  - `pathStats->optimal_flow` updated to 700
  - `pathStats->received_amount` updated to 700 (no exchanges/commissions)
  - `pathStats->flows` recalculated via `calculateFlows(700)`
  - Info log: "Path capacity adjusted: pathID=X, newMaxFlow=700, newOptimalFlow=700, newReceivedAmount=700"
- **Assertions**:
  - `pathStats->mMaxPathFlow == 700`
  - `pathStats->optimal_flow == 700`
  - `pathStats->received_amount == 700`
  - `pathStats->flows.size() > 0` (recalculated)

*Test 2: Shortage on path with exchange*
- **Setup**:
  - Path: A→B (eq 1001) →C [exchange 1001→2002, rate=0.5] →D (eq 2002)
  - Initial `mMaxPathFlow`: 2000, `optimal_flow`: 2000, `received_amount`: 1000 (2000 * 0.5)
  - Node B returns reservation: 1400
- **Execution**: Call `shortageReservationsOnPath(B_id, pathID, 1400)`
- **Expected**:
  - Forward simulate: 1400 in eq 1001 → exchange at C → 700 in eq 2002
  - `pathStats->mMaxPathFlow` = 1400
  - `pathStats->optimal_flow` = 1400
  - `pathStats->received_amount` = 700
  - `pathStats->flows` recalculated
- **Assertions**:
  - `pathStats->mMaxPathFlow == 1400`
  - `pathStats->optimal_flow == 1400`
  - `pathStats->received_amount == 700`
  - `pathStats->flows` contains correct edge flows with exchange applied

*Test 3: Shortage on path with commission*
- **Setup**:
  - Path: A→B→C→D (all in eq 1001)
  - Commission at node C: 10 units
  - Initial `mMaxPathFlow`: 1000, `optimal_flow`: 1000, `received_amount`: 990 (1000 - 10)
  - Node B returns reservation: 700
- **Execution**: Call `shortageReservationsOnPath(B_id, pathID, 700)`
- **Expected**:
  - Forward simulate: 700 → arrive at C → subtract commission 10 → 690
  - `pathStats->mMaxPathFlow` = 700
  - `pathStats->optimal_flow` = 700
  - `pathStats->received_amount` = 690
  - `pathStats->flows` recalculated
- **Assertions**:
  - `pathStats->mMaxPathFlow == 700`
  - `pathStats->optimal_flow == 700`
  - `pathStats->received_amount == 690`
  - `pathStats->flows` reflects commission deduction

*Test 4: Shortage with commission overflow*
- **Setup**:
  - Path: A→B→C→D
  - Commission at C: 50 units
  - Node B returns reservation: 30 (less than commission)
- **Execution**: Call `shortageReservationsOnPath(B_id, pathID, 30)`
- **Expected**:
  - Detect: 30 < 50 (commission)
  - Mark path unusable: `pathStats->setUnusable()` called
  - Warning log: "Amount exhausted by commission during shortage, marking path unusable"
  - `mMaxPathFlow`, `optimal_flow`, `received_amount`, `flows` not updated (return early)
- **Assertions**:
  - `pathStats->isValid() == false`
  - Old field values preserved

*Test 5: Path not found in mPathsStats*
- **Setup**: Empty `mPathsStats` or pathID not present
- **Execution**: Call `shortageReservationsOnPath(B_id, nonexistentPathID, 700)`
- **Expected**:
  - Warning log: "Path not found in mPathsStats for pathID=X"
  - Return early, no crash
- **Assertions**: No crash, warning logged

**2. IntermediateNodeExchangePaymentTransaction Incoming Calculation Tests**

*Test 6: Incoming calculation - same equivalent without commission*
- **Setup**:
  - Outgoing reservation: 500 in eq 1001
  - Incoming reservation exists with same eq 1001
  - No commission configured for eq 1001
- **Execution**: Run incoming calculation logic (TODO line 491 replacement)
- **Expected**:
  - `incomingAmount` = 500 (same as outgoing)
  - Call `shortageIncomingReservationsOnPath(pathID, 1001, 500)`
- **Assertions**:
  - `shortageIncomingReservationsOnPath` called with correct parameters
  - Debug log: "Incoming amount (no commission): 500"

*Test 7: Incoming calculation - same equivalent with commission*
- **Setup**:
  - Outgoing reservation: 500 in eq 1001
  - Incoming reservation: eq 1001
  - Commission for eq 1001: 20 units
- **Execution**: Run incoming calculation
- **Expected**:
  - `incomingAmount` = 520 (500 + 20)
  - Call `shortageIncomingReservationsOnPath(pathID, 1001, 520)`
- **Assertions**:
  - `shortageIncomingReservationsOnPath` called with amount 520
  - Debug log: "Incoming amount with commission: outgoing=500, commission=20, incoming=520"

*Test 8: Incoming calculation - different equivalents with exchange*
- **Setup**:
  - Outgoing reservation: 100 in eq 2002
  - Incoming reservation: eq 1001
  - Exchange rate: 1001→2002, rate=0.5 (so inverse: 2002→1001 rate=2.0)
- **Execution**: Run incoming calculation
- **Expected**:
  - Look up exchange rate 1001→2002
  - Invert: `incomingAmount` = 100 / 0.5 = 200
  - Call `shortageIncomingReservationsOnPath(pathID, 1001, 200)`
- **Assertions**:
  - `shortageIncomingReservationsOnPath` called with amount 200
  - Debug log contains: "Incoming amount via exchange: outgoing=100 (eq 2002), incoming=200 (eq 1001)"

*Test 9: Incoming calculation - exchange rate not found*
- **Setup**:
  - Outgoing: eq 2002
  - Incoming: eq 3003
  - No exchange rate configured for 3003→2002
- **Execution**: Run incoming calculation
- **Expected**:
  - Exchange rate lookup returns nullptr
  - Error log: "Exchange rate not found: 3003 -> 2002"
  - Call `sendErrorMessageOnNextNodeResponse(ResponseMessage::Rejected)`
  - Transaction terminates
- **Assertions**:
  - Error message sent
  - Transaction ends with error state

*Test 10: Incoming reservation not found by PathID*
- **Setup**: No incoming reservation exists with matching PathID
- **Execution**: Run incoming calculation
- **Expected**:
  - Error log: "Incoming reservation not found for pathID=X"
  - Call `sendErrorMessageOnNextNodeResponse(...)`
- **Assertions**:
  - Error message sent
  - Transaction terminates

**3. Path Addition Strategy Tests**

*Test 11: Add all paths without truncation*
- **Setup**:
  - 3 cached paths: capacity 500, 300, 200
  - Payment amount (`mAmount`): 600
- **Execution**: Run `runPathsResourceProcessingStage()`
- **Expected**:
  - ALL 3 paths added to `mPathsStats`
  - Each path added with full `received_amount` (no truncation)
  - Path 1: 500, Path 2: 300, Path 3: 200
- **Assertions**:
  - `mPathsStats.size() == 3`
  - Each path's `received_amount` matches original (not truncated)

**4. Path Filtering Tests**

*Test 12: Filter path with inaccessible node*
- **Setup**:
  - Path contains node C
  - `mInaccessibleNodes` contains node C
- **Execution**: Path validation in `switchToNextPath` (or wherever filtering added)
- **Expected**:
  - Validation detects C in `mInaccessibleNodes`
  - Path marked unusable: `pathStats->setUnusable()`
  - Info log: "Path contains inaccessible node: C"
  - Move to next path
- **Assertions**:
  - `pathStats->isValid() == false`
  - Next path processed

*Test 13: Filter path with rejected trust line*
- **Setup**:
  - Path contains edge A→B
  - `mRejectedTrustLines` contains pair (A, B)
- **Execution**: Path validation
- **Expected**:
  - Validation detects (A, B) in `mRejectedTrustLines`
  - Path marked unusable
  - Info log: "Path contains rejected trust line: A -> B"
- **Assertions**:
  - `pathStats->isValid() == false`

**5. Capacity Truncation Tests**

*Test 14: Truncate path exceeding remaining need*
- **Setup**:
  - Already reserved: 400 (toward `mAmount` = 600)
  - Remaining needed: 200
  - Next path `received_amount`: 500
- **Execution**: Capacity truncation logic before taking path
- **Expected**:
  - Detect: 500 > 200
  - Calculate truncated input via inverse calculation to deliver exactly 200
  - Call `pathStats->shortageMaxFlow(truncatedInput)`
  - Update `pathStats->received_amount` = 200
  - Recalculate flows: `pathStats->calculateFlows(truncatedInput)`
- **Assertions**:
  - `pathStats->received_amount == 200`
  - `pathStats->mMaxPathFlow` updated correctly

*Test 15: No truncation when path fits remaining need*
- **Setup**:
  - Already reserved: 400
  - Remaining needed: 200
  - Next path `received_amount`: 150
- **Execution**: Capacity truncation logic
- **Expected**:
  - Detect: 150 < 200 (no truncation needed)
  - Proceed with full path capacity
- **Assertions**:
  - `pathStats->received_amount` unchanged (still 150)

#### Regression Testing (Unit)
- Scope: Ensure changes don't break existing exchange payment execution
- Verify single-equivalent payments remain unaffected
- Validate path processing continues correctly when no adjustments needed
- Confirm existing `shortageIncomingReservationsOnPath` / `shortageOutgoingReservationsOnPath` still work

#### Execution in CI/Locally
- Build tests in `build-tests` and run the produced binaries
- All unit tests must pass before PRD completion

### Quality Gates
- All unit tests pass in `build-tests`
- Coordinator correctly adjusts path capacity in 100% of test cases
- Intermediate node correctly calculates incoming reservation in 100% of test cases
- Path filtering correctly identifies bad paths in 100% of test cases
- No regressions in existing payment execution

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Bug fix and enhancement (backward compatible)
- **Rollout Strategy**: Direct deployment; internal logic change only, no protocol modifications
- **Rollback Plan**: Revert to previous version if critical calculation errors found

### Database Migrations
- No database migrations required (runtime logic only)

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: No external communication needed (internal fix)
- **Documentation Updates**: Update internal payment flow documentation with adjustment logic

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**:
  - Correct path capacity adjustment (target: 100% accuracy in unit tests)
  - Correct incoming reservation calculation (target: 100% accuracy)
  - Payment success rate improvement when capacity varies (target: +15%)
  - Reduced processing of filtered paths (target: -50% attempts on bad paths)
- **Leading Indicators**: Unit test pass rate, manual testing with capacity variations
- **Baseline Values**: Current payment failure rate with capacity changes
- **Target Values**:
  - 100% unit test pass rate
  - Zero payment failures due to incorrect adjustment
  - All edge cases handled (commission overflow, exchange rate not found, etc.)

### Monitoring Plan
- **New Dashboards/Alerts**: Not applicable (internal logic enhancement)
- **Enhanced Monitoring**: Extended logging for capacity adjustments and incoming calculations
- **A/B Testing**: Not applicable

### Review Schedule
- **Daily**: Development progress and unit test status
- **Weekly**: Integration testing with manual scenarios
- **Post-Implementation Review**: Payment success rate analysis with real variations

## Appendices
### Glossary
- **Capacity Adjustment**: Process of updating path capacity when node reserves less than requested
- **Forward Simulation**: Computing output amount from input by applying exchanges and subtracting commissions in path direction
- **Inverse Calculation**: Computing required input amount from desired output, working backwards through exchanges and commissions
- **Commission Overflow**: Situation where reservation amount is less than commission, making path unusable
- **Incoming Reservation**: Reservation from previous node to current intermediate node
- **Outgoing Reservation**: Reservation from current node to next node
- **path.ids vs path.nodes**:
  - `path.ids` (vector<ContractorID>): Used for iteration and calculations; always populated
  - `path.nodes` (vector<BaseAddress::Shared>): Used for filtering; populated by `addPathForFurtherProcessing`

### References
- [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
- [Exchange Payment Topology Collection PRD](07-exchange-payment-topology-collection.md)
- [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
- [CoordinatorExchangePaymentTransaction Implementation](../../../src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h)
- [IntermediateNodeExchangePaymentTransaction Implementation](../../../src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h)

### Detailed Component Specifications

#### Modified Methods

##### CoordinatorExchangePaymentTransaction::shortageReservationsOnPath
- **Location**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`
- **Current Signature**: `void shortageReservationsOnPath(ContractorID neighborID, const PathID &pathID, const TrustLineAmount &kNewAmount)`
- **Called From**:
  - `processRemoteNodeResponse`: when remote intermediate node returns reduced reservation
  - `processNeighborFurtherReservationResponse`: when direct neighbor returns reduced reservation for further propagation
- **Modifications**:
  - Add path lookup in `mPathsStats`
  - Add forward simulation through exchanges and commissions to calculate new `received_amount`
  - **Iteration uses `path.ids` and `path.equivalents`** (NOT `path.nodes`), because `findExchangeStep` expects ContractorID
  - Update ALL path fields: `mMaxPathFlow`, `optimal_flow`, `received_amount`
  - Recalculate `flows` vector via `pathStats->calculateFlows(kNewAmount)`
  - Add error handling for calculation failures (commission overflow, exchange not found)
- **Dependencies**: `findExchangeStep`, `applyExchangeForward` helper functions, `calculateFlows` method

##### IntermediateNodeExchangePaymentTransaction::runNextNeighborResponseProcessingStage
- **Location**: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.cpp`, line 491
- **Modification Type**: Replace TODO comment with implementation
- **New Logic**:
  - Find incoming reservation by PathID
  - Get equivalents from incoming/outgoing reservations
  - Calculate incoming amount based on equivalent match
  - Handle exchange rate lookup and commission application
  - Call `shortageIncomingReservationsOnPath`
  - Handle errors with coordinator notification
- **Dependencies**: `ExchangeRatesManager`, `CommissionsManager`, existing `shortageIncomingReservationsOnPath` method

##### CoordinatorExchangePaymentTransaction::runPathsResourceProcessingStage
- **Location**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`
- **Current Behavior**: Adds paths until `totalAddedFlow >= mAmount`, truncates last path capacity
- **Modified Behavior**:
  - Remove early break condition
  - Add ALL paths from all equivalents
  - Pass full `received_amount` to `addPathForFurtherProcessing`
  - Remove truncation logic (lines 496-497 reference)
- **Impact**: More paths available, better alternative path support

##### CoordinatorExchangePaymentTransaction::switchToNextPath (or calling location)
- **Location**: Before/during `switchToNextPath()` method
- **New Additions**:
  - Path validation for `mInaccessibleNodes` using `path.nodes`
  - Trust line validation for `mRejectedTrustLines` using `path.nodes`
  - **Filtering uses `path.nodes` (BaseAddress::Shared)**, because `mInaccessibleNodes` and `mRejectedTrustLines` store addresses
  - Precondition: `path.nodes` populated by `addPathForFurtherProcessing` (guaranteed at reservation stage)
  - Capacity truncation check and execution
  - Mark unusable and skip logic
- **Dependencies**:
  - `mInaccessibleNodes`, `mRejectedTrustLines` tracking
  - Helper functions: `calculateTotalReservedAmount()`, `proceedToNextStage()`, `tryProcessNextPath()`, `calculateRequiredInputForPath()`
  - See "Helper Functions Descriptions" in Algorithm Specifications section for details

#### Helper Functions

##### findExchangeStep (existing)
- Already exists in codebase
- Used to locate ExchangeStep by node and equivalents
- Returns pointer to ExchangeStep or nullptr

##### applyExchangeForward (existing or similar)
- May exist in codebase as part of `calculateFlows` logic
- Applies exchange rate forward: `output = input * rate * 10^shift`

##### invertExchangeForRequiredInput (may need creation)
- Inverts exchange calculation: `input = output / (rate * 10^shift)`
- Uses ceiling division to favor higher input
- Similar to existing `invertExchangeForRequiredInput` in coordinator cpp file

##### ceilDivideToAmount (existing)
- Already exists in coordinator cpp file (line 24)
- Performs ceiling division and converts to TrustLineAmount

##### calculateRequiredInputForPath (existing pattern or new)
- **Purpose**: Calculate required input to achieve desired output by working backwards through path
- **Signature**: `TrustLineAmount calculateRequiredInputForPath(const OptimalPathResult &pathResult, const TrustLineAmount &desiredOutputAmount)`
- **Algorithm**:
  1. Start with `desiredOutputAmount` at receiver end
  2. Work backwards through `path.ids` in reverse order
  3. For each step backwards:
     - If commission step: add commission to amount
     - If exchange step: invert exchange (divide by rate, use ceiling)
  4. Return final amount needed at sender end
- **Note**: Pattern exists in coordinator, may need adaptation for new context
- **Used by**: Capacity truncation logic in path filtering

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-10-21 | Claude Code | Initial draft for path capacity adjustment | Phase 1 |
| 1.1 | 2025-10-21 | Claude Code | Fixed inconsistency: clarified path.ids (for iteration/calculations) vs path.nodes (for filtering); added preconditions and notes throughout document | Phase 1 |
| 1.2 | 2025-10-21 | Claude Code | Added explicit descriptions for helper functions referenced in pseudocode (calculateTotalReservedAmount, proceedToNextStage, tryProcessNextPath, calculateRequiredInputForPath) | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [07-exchange-payment-topology-collection.md](07-exchange-payment-topology-collection.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)
- **Payment Protocol**: [payment-protocol.md](../../../architecture/vtcpd/protocols/payment-protocol.md)
