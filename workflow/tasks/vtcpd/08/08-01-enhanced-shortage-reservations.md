# 08-01 - Enhanced shortageReservationsOnPath in Coordinator

# Links
- [PRD](../../../prd/vtcpd/08-path-capacity-adjustment.md)

# Description
Fix `CoordinatorExchangePaymentTransaction::shortageReservationsOnPath` method to correctly adjust ALL path capacity fields when an intermediate node returns a reduced reservation amount. The current implementation (copied from single-equivalent `CoordinatorPaymentTransaction`) doesn't account for multi-equivalent scenarios with exchanges and commissions.

The method must:
- Update coordinator's own reservation to the new amount (existing logic)
- Forward-simulate through the path to calculate new `received_amount` (accounting for exchanges and commissions)
- Update ALL OptimalPathResult fields: `mMaxPathFlow`, `optimal_flow`, `received_amount`
- Recalculate `flows` vector via `calculateFlows(kNewAmount)`
- Handle error conditions (commission overflow, exchange step not found)

This method is called from two locations:
1. `processRemoteNodeResponse` - when remote intermediate node returns reduced reservation
2. `processNeighborFurtherReservationResponse` - when direct neighbor returns reduced reservation for further propagation

# Requirements and DOD

## Functional Requirements
1. Method signature preserved: `void shortageReservationsOnPath(ContractorID neighborID, const PathID &pathID, const TrustLineAmount &kNewAmount)`
2. Find path in `mPathsStats` by `pathID` - if not found, log warning and return early
3. Update coordinator's own reservation to `kNewAmount` via existing `shortageReservation` call
4. Forward-simulate through path structure:
   - Use `path.ids` and `path.equivalents` for iteration (NOT `path.nodes`)
   - For each step: detect exchanges (same node, different equivalent) or trust line edges
   - Apply exchanges forward using existing helpers (`findExchangeStep`, `applyExchangeForward`)
   - Subtract commissions at intermediate nodes (not receiver)
   - Calculate final `newReceivedAmount` at receiver
5. Update ALL path statistics fields:
   - `pathStats->mMaxPathFlow = kNewAmount`
   - `pathStats->optimal_flow = kNewAmount`
   - `pathStats->received_amount = newReceivedAmount`
6. Recalculate flows vector: `pathStats->calculateFlows(kNewAmount)` in try-catch
7. Error handling:
   - Commission overflow (amount < commission): mark path unusable via `setUnusable()`, log warning, return early
   - Exchange step not found: log warning, return early (keep old values)
   - `calculateFlows` exception: log warning, continue (main fields already updated)
8. Logging:
   - Info log with all updated fields: `newMaxFlow`, `newOptimalFlow`, `newReceivedAmount`, old `received_amount`
   - Include "flows recalculated" confirmation

## Definition of Done
- [ ] Method correctly updates all 4 fields (`mMaxPathFlow`, `optimal_flow`, `received_amount`, `flows`)
- [ ] Forward simulation logic matches `calculateFlows` pattern (exchanges and commissions)
- [ ] Uses `path.ids` and `path.equivalents` for iteration (as per PRD note)
- [ ] Handles commission overflow by marking path unusable
- [ ] Handles missing exchange step gracefully
- [ ] Calls existing `shortageReservation` for coordinator's reservation
- [ ] Logs all changes with appropriate level (info for success, warning for errors)
- [ ] No crashes on edge cases (empty path, missing fields, etc.)

# Implementation Plan

## Step 1: Locate and analyze current implementation
- File: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`
- Current method at line ~2228 (may vary)
- Understand current logic: calls `shortageReservation` but doesn't update `OptimalPathResult` fields
- Identify helper functions already available: `findExchangeStep`, `applyExchangeForward`

## Step 2: Implement path lookup and existing reservation update
```cpp
void CoordinatorExchangePaymentTransaction::shortageReservationsOnPath(
    ContractorID neighborID,
    const PathID &pathID,
    const TrustLineAmount &kNewAmount)
{
    debug() << "shortageReservationsOnPath: pathID=" << pathID
            << ", neighborID=" << neighborID
            << ", newAmount=" << kNewAmount;

    // Step 1: Update coordinator's own reservation (EXISTING LOGIC - preserve)
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

    // Continue to Step 3...
}
```

## Step 3: Implement forward simulation
```cpp
    // Step 3: Calculate new received_amount by forward-simulating through path
    TrustLineAmount newReceivedAmount;
    try {
        // Forward simulate: apply exchanges and subtract commissions
        // Use path.ids and path.equivalents (NOT path.nodes)
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
```

## Step 4: Update all fields and recalculate flows
```cpp
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

## Step 5: Verify call sites
- Ensure method is called from `processRemoteNodeResponse` when remote node returns reduced amount
- Ensure method is called from `processNeighborFurtherReservationResponse` when neighbor returns reduced amount
- If not called yet, add calls in appropriate locations

# Test Plan
Tests will be implemented in a separate task (08-05). This task focuses on implementation only.

Expected test scenarios (for reference):
- Simple path without exchanges/commissions
- Path with exchange (verify exchange applied correctly)
- Path with commission (verify commission subtracted)
- Commission overflow (verify path marked unusable)
- Path not found in mPathsStats (verify graceful handling)

# Verification and Validation

## Architecture integrity
- **Simple Task Validation**: Verify method signature unchanged (backward compatible)
- Verify uses existing helpers (`findExchangeStep`, `applyExchangeForward`) without modification
- Verify uses existing `path.ids` and `path.equivalents` (correct path structure usage as per PRD)
- Verify calls existing `shortageReservation` (no change to reservation logic)

## Security
- **Simple Task**: No security implications (internal calculation only)
- No exposure of path structure to external parties
- Error handling prevents crashes on malformed data

## Performance
- **Simple Task**: Forward simulation overhead < 5ms per path (typical 3-5 steps)
- `calculateFlows` call overhead acceptable (already called elsewhere)
- No loops over large datasets (path length limited to ~10 nodes max)

## Scalability
- **Simple Task**: Handles paths up to 10 nodes (typical max in network)
- Handles up to 5 exchanges per path (per PRD limit)
- No impact on overall transaction scalability

## Reliability
- **Simple Task**: Error handling prevents crashes
- Graceful degradation: on error, keeps old values (safe fallback)
- Commission overflow detection prevents invalid state
- Missing exchange step detection prevents undefined behavior

## Maintainability
- **Simple Task**: Code structure matches `calculateFlows` pattern (consistency)
- Clear comments explain `path.ids` vs `path.nodes` usage
- Logging provides debugging information
- Error messages descriptive

## Cost
- **Simple Task**: No additional resource usage
- Reuses existing calculation helpers

## Compliance
- **Simple Task**: Follows PRD 08 specification exactly
- Adheres to project coding standards
- Maintains backward compatibility

# Restrictions
- Commit changes only after successfully passing the tests (tests will be created in task 08-05)
- Do not modify helper functions (`findExchangeStep`, `applyExchangeForward`) - use as-is
- Do not change method signature (backward compatibility)
- Do not modify existing `shortageReservation` call logic
