# 09-06 - Enhanced Receiver Capacity Preservation in Condition Change Handling

# Links
- [PRD](../../../prd/vtcpd/09-exchange-rate-commission-change-handling.md#enhanced-condition-change-handling-with-maximum-receiver-capacity)
- [Previous task: 09-03](09-03-coordinator-condition-change-detection.md)

# Description
Enhance the coordinator's condition change handling (implemented in task 09-03) to optimize payment throughput by prioritizing receiver-side capacity preservation. Instead of always constraining by sender-side flow when conditions change, attempt to preserve the maximum receiver-side deliverable amount by recalculating required sender-side flow. Only fall back to sender-side constraint if the new required flow exceeds available sender capacity.

This enhancement introduces `mMaxPathReceivedAmount` field to track maximum receiver-side capacity and implements backward flow simulation to calculate required sender-side input for a given receiver-side target.

# Requirements and DOD

## Requirements

### R1: Add mMaxPathReceivedAmount Field to OptimalPathResult
- Add new field `TrustLineAmount mMaxPathReceivedAmount` to OptimalPathResult class
- Initialize to 0 in constructor
- Add detailed comment explaining its purpose: maximum receiver-side amount deliverable through this path
- Distinguish from `mMaxPathFlow` (sender-side) and `received_amount` (current value)

### R2: Initialize mMaxPathReceivedAmount in addPathForFurtherProcessing
- When adding new path from cached path result:
  - Set `pathCopy->mMaxPathFlow = pathResult.optimal_flow` (existing)
  - Set `pathCopy->mMaxPathReceivedAmount = pathResult.received_amount` (NEW)
- Add comment explaining both fields represent upper capacity bounds
- Log both values for debugging

### R3: Update mMaxPathReceivedAmount in shortageReservationsOnPath
- When intermediate node returns reduced capacity:
  - Update `mMaxPathFlow` to new sender-side flow (existing)
  - Update `mMaxPathReceivedAmount` to new receiver-side amount (NEW)
- Recalculate `received_amount` through forward simulation (existing)
- Add comment distinguishing shortage (real capacity reduction) from condition change (pricing change)
- Both bounds reduced because actual path capacity decreased

### R4: Preserve mMaxPathReceivedAmount in handleConditionChange
- Save `mMaxPathReceivedAmount` BEFORE calling `dropReservationsOnPath/setUnusable`:
  ```cpp
  TrustLineAmount savedMaxPathFlow = pathStats->mMaxPathFlow;
  TrustLineAmount savedMaxPathReceivedAmount = pathStats->mMaxPathReceivedAmount;
  TrustLineAmount savedPaymentFlow = pathStats->paymentFlow;
  ```
- Critical: `setUnusable()` resets `mMaxPathFlow` to 0, so save before calling
- Use saved values for all subsequent calculations
- Set both bounds in new path from saved values (not recalculated values)

### R5: Implement calculateOptimalFlowWithUpdatedConditions Method
- Create new method in CoordinatorExchangePaymentTransaction:
  ```cpp
  TrustLineAmount calculateOptimalFlowWithUpdatedConditions(
      OptimalPathResult *pathStats,
      const TrustLineAmount &desiredReceivedAmount,
      const SerializedPositionInPath affectedNodePosition,
      const optional<pair<TrustLineAmount, int16_t>> &updatedRate,
      const optional<TrustLineAmount> &updatedCommission);
  ```
- Perform backward simulation from receiver to sender:
  1. Start with `desiredReceivedAmount` (target at receiver)
  2. Iterate backward through path (from last to first node)
  3. For each exchange: invert rate to compute required input
  4. For each commission: add commission back to required amount
  5. Apply updated rate/commission at affected position
  6. Apply original conditions at other positions
  7. Return required sender-side flow
- Add detailed comments explaining backward simulation logic
- Handle errors: throw ValueError if calculation fails

### R6: Implement invertExchangeForRequiredInput Helper
- Create helper method to invert exchange rate calculation:
  ```cpp
  TrustLineAmount invertExchangeForRequiredInput(
      const ExchangeStep &step,
      const TrustLineAmount &outputAmount);
  ```
- Given output amount, calculate required input: `input = output / (rate * 10^shift)`
- Use ceiling division for safety (ensure sufficient input)
- Handle edge cases: zero rate, overflow, etc.

### R7: Update handleConditionChange Logic
- After saving values and dropping reservations:
  1. Call `calculateOptimalFlowWithUpdatedConditions(savedMaxPathReceivedAmount, ...)`
  2. Compare result with `savedMaxPathFlow`:
     - If `newOptimalFlow <= savedMaxPathFlow`:
       - Conditions acceptable: use `finalOptimalFlow = newOptimalFlow`
       - Use `finalReceivedAmount = savedMaxPathReceivedAmount` (full capacity)
       - Log: "Conditions improved/acceptable: using full receiver capacity"
     - If `newOptimalFlow > savedMaxPathFlow`:
       - Conditions worsened: use `finalOptimalFlow = savedMaxPathFlow`
       - Recalculate `finalReceivedAmount = calculateReceivedAmountWithUpdatedConditions(savedMaxPathFlow, ...)`
       - Log: "Conditions worsened: constraining to sender capacity"
  3. Create new path with `finalOptimalFlow` and `finalReceivedAmount`
  4. Set `mMaxPathFlow = savedMaxPathFlow` and `mMaxPathReceivedAmount = savedMaxPathReceivedAmount`
  5. Call `calculateFlows(finalOptimalFlow)`

### R8: Update Logging
- Add logging of `mMaxPathReceivedAmount` wherever `mMaxPathFlow` is logged:
  - In `handleConditionChange`: log saved values
  - In `handleConditionChange`: log new path details
  - Maintain consistency with existing logging patterns

### R9: Update Tests
- Update `TestCoordinatorExchangePaymentCapacityAdjustment.cpp`:
  - Initialize `mMaxPathReceivedAmount` in helper methods:
    - `createSimplePath`: set to capacity
    - `createPathWithExchange`: set to receivedAmount
    - `createPathWithCommission`: set to received amount (capacity - commission)
  - Add assertions for `mMaxPathReceivedAmount` in all test cases:
    - `ShortageOnSimplePath_NoExchangeNoCommission`
    - `ShortageOnPath_WithExchange`
    - `ShortageOnPath_WithCommission`
- Ensure tests pass with new field and logic

## Definition of Done

1. `mMaxPathReceivedAmount` field added to OptimalPathResult with proper initialization

2. Field correctly initialized in `addPathForFurtherProcessing` with `pathResult.received_amount`

3. Field correctly updated in `shortageReservationsOnPath` when capacity reduced

4. Field correctly saved before `dropReservationsOnPath/setUnusable` in `handleConditionChange`

5. `calculateOptimalFlowWithUpdatedConditions` method implemented with backward simulation:
   - Correctly inverts exchange rates
   - Correctly adds back commissions
   - Applies updated conditions at affected position
   - Applies original conditions at other positions
   - Returns correct required sender-side flow

6. `invertExchangeForRequiredInput` helper implemented with ceiling division

7. `handleConditionChange` logic updated to:
   - First attempt preserving receiver capacity
   - Fall back to sender capacity constraint if needed
   - Use correct flow and amount for new path
   - Preserve saved bounds in new path

8. Logging updated to include `mMaxPathReceivedAmount` consistently

9. All tests in `TestCoordinatorExchangePaymentCapacityAdjustment.cpp` pass:
   - Fields initialized in test helpers
   - Assertions check `mMaxPathReceivedAmount`
   - Tests validate shortage behavior

10. Code compiles without errors or warnings

11. Behavior verified with example scenario:
    - Initial: `mMaxPathFlow=2990`, `mMaxPathReceivedAmount=146`
    - After partial payment: `optimal_flow=2070`, bounds unchanged
    - Rate change 0.05→0.04: new flow=3735, constrain to 2990, received=116 (improved from 100)

# Implementation Notes

## Backward vs Forward Simulation
- **Forward** (`calculateReceivedAmountWithUpdatedConditions`): Given sender input, compute receiver output
  - Apply exchanges: `output = input * rate * 10^shift`
  - Subtract commissions: `output = output - commission`
  - Used when constraining by sender capacity

- **Backward** (`calculateOptimalFlowWithUpdatedConditions`): Given receiver target, compute sender input
  - Invert exchanges: `input = output / (rate * 10^shift)` (ceiling division)
  - Add back commissions: `input = input + commission`
  - Used when attempting to preserve receiver capacity

## Capacity Bounds vs Current Values
- **Bounds** (`mMaxPathFlow`, `mMaxPathReceivedAmount`):
  - Set once when path added
  - Preserved in `handleConditionChange` (pricing change)
  - Updated in `shortageReservationsOnPath` (capacity change)
  - Represent upper limits independent of current flow

- **Current** (`optimal_flow`, `received_amount`):
  - Change during payment execution
  - Adjusted for partial payments
  - May be less than bounds but never exceed them

## Error Handling
- If `calculateOptimalFlowWithUpdatedConditions` throws:
  - Log error message
  - Call `tryProcessNextPath()` to continue payment
  - Don't create new path with invalid data

- If exchange rate is zero or causes overflow:
  - Throw ValueError with descriptive message
  - Coordinator will handle by skipping path

## Testing Strategy
- Unit tests validate field initialization and updates
- Unit tests validate shortage behavior (both bounds updated)
- Integration tests (if any) validate condition change scenarios
- Manual testing with example scenario from PRD

# Success Criteria
- All unit tests pass
- Code compiles without errors
- Behavior matches PRD specification
- Example scenario produces expected results (received_amount=116 vs 79)
- Logging provides clear visibility into decision making
- No regressions in existing payment functionality

# Related Tasks
- **09-03**: Implemented original condition change detection and handling
- **09-05**: Unit tests for exchange payment functionality
- **08-01**: Path capacity adjustment infrastructure (shortage handling)

# Implementation Status
- **Status**: Completed (2025-10-29)
- **Implemented by**: Claude Code (Sonnet 4.5)
- **Files Modified**:
  - `src/core/paths/lib/OptimalPathResult.h`
  - `src/core/paths/lib/OptimalPathResult.cpp`
  - `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`
  - `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`
  - `tests/unit/transactions/TestCoordinatorExchangePaymentCapacityAdjustment.cpp`
- **Tests**: All passing (3/3 in TestCoordinatorExchangePaymentCapacityAdjustment)
- **Notes**: Implementation successfully handles example scenario with rate change 0.05→0.04
