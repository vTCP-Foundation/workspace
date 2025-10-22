# 08-03 - Full Path Addition Strategy and Helper Functions

# Links
- [PRD](../../../prd/vtcpd/08-path-capacity-adjustment.md)

# Description
Implement two related improvements to path processing in `CoordinatorExchangePaymentTransaction`:

**Part A: Full Path Addition Strategy**
Modify `runPathsResourceProcessingStage` to add ALL paths from `ExchangePathsManager` without initial capacity truncation. Currently, the method truncates the last path to match exact payment amount, limiting flexibility when paths fail. The new approach adds all paths at full capacity, deferring truncation to actual processing time.

**Part B: Helper Functions for Path Filtering**
Create helper functions needed for path filtering and capacity management:
1. `calculateTotalReservedAmount()` - calculates total successfully reserved capacity
2. `proceedToNextStage()` - transitions to next stage after sufficient reservation
3. Verify `tryProcessNextPath()` exists and works correctly
4. `calculateRequiredInputForPath()` - inverse calculation for capacity truncation

These helpers enable intelligent path selection and capacity management during reservation.

# Requirements and DOD

## Part A: Full Path Addition Strategy

### Functional Requirements
1. **Modify `runPathsResourceProcessingStage` method**:
   - Remove early break condition when `totalAddedFlow >= mAmount`
   - Change loop to iterate through ALL paths from `cachedPaths` for each sender equivalent
   - Call `addPathForFurtherProcessing(pathResult, pathResult.received_amount)` with FULL path capacity
   - Remove any capacity truncation logic (e.g., lines 496-497 reference in PRD)
2. **Result**: ALL optimal paths added to `mPathsStats` with their full `received_amount` capacity

### Definition of Done - Part A
- [ ] Early break on `totalAddedFlow >= mAmount` removed
- [ ] Loop processes ALL paths from ALL sender equivalents
- [ ] `addPathForFurtherProcessing` called with full `pathResult.received_amount`
- [ ] No truncation during initial path addition
- [ ] All paths from `ExchangePathsManager` present in `mPathsStats` after stage completion

## Part B: Helper Functions

### Functional Requirements

#### 1. `calculateTotalReservedAmount()`
- **Purpose**: Calculate total amount already successfully reserved across all processed paths
- **Returns**: `TrustLineAmount` - total in receiver equivalent (mEquivalent)
- **Algorithm**:
  ```cpp
  TrustLineAmount calculateTotalReservedAmount() {
      TrustLineAmount total = TrustLineAmount(0);
      for (const auto &[pathID, pathStats] : mPathsStats) {
          // Sum received_amount for all successfully reserved paths
          // Only count paths where reservation is approved/completed
          if (pathStats->isLastIntermediateNodeApproved() ||
              /* other completion criteria based on existing logic */) {
              total = total + pathStats->received_amount;
          }
      }
      return total;
  }
  ```

#### 2. `proceedToNextStage()`
- **Purpose**: Transition to next transaction stage after sufficient capacity reserved
- **Returns**: `TransactionResult::SharedConst`
- **Algorithm**:
  ```cpp
  TransactionResult::SharedConst proceedToNextStage() {
      // Transition to final amounts configuration or next reservation stage
      mStep = Coordinator_FinalAmountsConfiguration;  // or appropriate next stage
      return sendFinalAmountsConfigurationToAllParticipants();
  }
  ```
- **Note**: May already exist in similar form - verify and adapt if needed

#### 3. `tryProcessNextPath()` - Verification
- **Action**: Verify this method exists and works correctly
- **Expected behavior**:
  - Moves to next PathID in `mPathIDs` vector
  - If no more paths: checks if sufficient capacity reserved
  - If insufficient: returns error (no paths available)
  - If more paths: continues with next path processing
- **Note**: This is likely an existing method - document if modifications needed

#### 4. `calculateRequiredInputForPath(pathStats, desiredOutput)`
- **Purpose**: Calculate required input amount to achieve desired output (inverse calculation)
- **Signature**: `TrustLineAmount calculateRequiredInputForPath(const OptimalPathResult &pathResult, const TrustLineAmount &desiredOutputAmount)`
- **Returns**: Required input amount at sender (in sender equivalent)
- **Algorithm**:
  1. Start with `desiredOutputAmount` at receiver end
  2. Work backwards through `path.ids` in reverse order
  3. For each step backwards:
     - If commission step: add commission to amount
     - If exchange step: invert exchange (divide by rate, use ceiling)
  4. Return final amount needed at sender end
- **Note**: Pattern may exist in coordinator - adapt for new context if needed

### Definition of Done - Part B
- [ ] `calculateTotalReservedAmount()` implemented and functional
- [ ] `proceedToNextStage()` implemented or verified existing
- [ ] `tryProcessNextPath()` verified functional (document if already exists)
- [ ] `calculateRequiredInputForPath()` implemented with inverse simulation
- [ ] All helpers properly declared in `.h` file
- [ ] All helpers have appropriate access modifiers (likely private or protected)
- [ ] Clear comments explain each helper's purpose

## Combined DOD
- [ ] Part A: All paths added without truncation
- [ ] Part B: All 4 helper functions implemented/verified
- [ ] No compilation errors
- [ ] Code follows project style guidelines
- [ ] Appropriate logging added for debugging

# Implementation Plan

## Part A Implementation

### Step 1: Locate current implementation
- File: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`
- Method: `runPathsResourceProcessingStage()`
- Identify current loop structure and truncation logic

### Step 2: Remove early break and truncation
Find code similar to:
```cpp
// CURRENT CODE (to be modified)
TrustLineAmount totalAddedFlow = TrustLineAmount(0);

for (const auto& exchangeEquiv : mExchangeEquivalents) {
    PathCacheKey key{mContractorID, exchangeEquiv, mEquivalent};
    auto cachedPaths = mExchangePathsManager->retrievePaths(key);

    if (!cachedPaths) {
        continue;
    }

    for (const auto& pathResult : *cachedPaths) {
        // REMOVE THIS CHECK:
        // if (totalAddedFlow >= mAmount) break;

        // REMOVE TRUNCATION (lines ~496-497):
        // TrustLineAmount pathAmount = min(pathResult.received_amount, mAmount - totalAddedFlow);

        // NEW: Add with full capacity
        addPathForFurtherProcessing(pathResult, pathResult.received_amount);

        // REMOVE THIS UPDATE:
        // totalAddedFlow += pathResult.received_amount;
    }
}
```

### Step 3: Implement modified loop
```cpp
// NEW CODE
for (const auto& exchangeEquiv : mExchangeEquivalents) {
    PathCacheKey key{mContractorID, exchangeEquiv, mEquivalent};
    auto cachedPaths = mExchangePathsManager->retrievePaths(key);

    if (!cachedPaths) {
        debug() << "No cached paths for exchangeEquiv=" << exchangeEquiv;
        continue;
    }

    // Add ALL paths with their full capacity (no truncation, no break)
    for (const auto& pathResult : *cachedPaths) {
        addPathForFurtherProcessing(pathResult, pathResult.received_amount);
    }
}
```

## Part B Implementation

### Step 1: Add helper function declarations to header
File: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

```cpp
protected:  // or private:
    // Helper functions for path filtering and capacity management
    TrustLineAmount calculateTotalReservedAmount();
    TransactionResult::SharedConst proceedToNextStage();
    TrustLineAmount calculateRequiredInputForPath(
        const OptimalPathResult &pathResult,
        const TrustLineAmount &desiredOutputAmount);
    // tryProcessNextPath() - verify if already declared
```

### Step 2: Implement `calculateTotalReservedAmount()`
```cpp
TrustLineAmount CoordinatorExchangePaymentTransaction::calculateTotalReservedAmount()
{
    TrustLineAmount total = TrustLineAmount(0);

    for (const auto &[pathID, pathStats] : mPathsStats) {
        // Sum received_amount for all successfully reserved paths
        // Only count paths where reservation is approved/completed
        // Adjust condition based on existing path state logic
        if (pathStats->isLastIntermediateNodeApproved() ||
            pathStats->isValid() /* add other criteria as needed */) {
            total = total + pathStats->received_amount;
        }
    }

    debug() << "Total reserved amount: " << total;
    return total;
}
```

### Step 3: Implement `proceedToNextStage()`
```cpp
TransactionResult::SharedConst CoordinatorExchangePaymentTransaction::proceedToNextStage()
{
    info() << "Sufficient capacity reserved, proceeding to next stage";

    // Transition to final amounts configuration or appropriate next stage
    // Adjust based on existing transaction flow
    mStep = Coordinator_FinalAmountsConfiguration;
    return sendFinalAmountsConfigurationToAllParticipants();
}
```

### Step 4: Verify/Document `tryProcessNextPath()`
- Check if method exists in current implementation
- If exists: document expected behavior and verify it matches PRD description
- If doesn't exist: may need to implement basic version or clarify with architect

### Step 5: Implement `calculateRequiredInputForPath()`
```cpp
TrustLineAmount CoordinatorExchangePaymentTransaction::calculateRequiredInputForPath(
    const OptimalPathResult &pathResult,
    const TrustLineAmount &desiredOutputAmount)
{
    const auto &path = pathResult.path();

    if (desiredOutputAmount == TrustLineAmount(0)) {
        return TrustLineAmount(0);
    }

    // Start from receiver with desired output
    TrustLineAmount currentAmount = desiredOutputAmount;

    // Work backwards through path (reverse iteration)
    for (int idx = path.ids.size() - 2; idx >= 0; --idx) {
        const ContractorID fromNode = path.ids[idx];
        const ContractorID toNode = path.ids[idx + 1];
        const SerializedEquivalent currentEquiv = path.equivalents[idx];
        const SerializedEquivalent nextEquiv = path.equivalents[idx + 1];

        // Check for commission at intermediate node (work backwards)
        if (idx + 1 < path.ids.size() - 1) {  // Not receiver
            const auto *commissionStep = findExchangeStep(
                path, toNode, nextEquiv, nextEquiv);

            if (commissionStep && commissionStep->commission > TrustLineAmount(0)) {
                // Add commission back when going backwards
                currentAmount = currentAmount + commissionStep->commission;
            }
        }

        // Check for exchange (same node, different equivalent)
        if (fromNode == toNode && currentEquiv != nextEquiv) {
            // Invert exchange when going backwards
            const auto *exchangeStep = findExchangeStep(
                path, fromNode, currentEquiv, nextEquiv);

            if (!exchangeStep) {
                throw ValueError("Exchange step not found during inverse calculation");
            }

            // Invert: output = input * rate * 10^shift
            // input = output / (rate * 10^shift) with ceiling
            currentAmount = invertExchangeForRequiredInput(
                currentAmount,
                exchangeStep->exchangeRate,
                exchangeStep->exchangeRateShift);
        }
    }

    return currentAmount;
}
```

Note: May need to create/reuse `invertExchangeForRequiredInput` helper (similar to task 08-02).

# Test Plan
Tests will be implemented in a separate task (08-05). This task focuses on implementation only.

Expected test scenarios (for reference):

**Part A**:
- Verify all paths added to mPathsStats (not just minimum needed)
- Verify paths have full received_amount (not truncated)

**Part B**:
- calculateTotalReservedAmount: correctly sums only approved paths
- proceedToNextStage: transitions to correct stage
- calculateRequiredInputForPath: correctly inverts exchanges and adds commissions

# Verification and Validation

## Architecture integrity
- **Moderate Task Validation**: Verify integration with existing path processing
- Part A: No modification to `addPathForFurtherProcessing` (uses existing interface)
- Part B: Helper functions follow existing patterns (similar to other coordinator helpers)
- No changes to `ExchangePathsManager` interface
- Uses existing path structure (`OptimalPathResult`, `ExchangePath`)

## Security
- **Moderate Task**: No security implications (internal logic)
- No exposure of path details to external parties
- Helper functions are protected/private (not exposed in API)

## Performance
- **Moderate Task**:
- Part A: Adds more paths to `mPathsStats` (increased memory ~1-2KB per path, acceptable for <20 paths)
- `calculateTotalReservedAmount`: O(n) where n = number of paths (typically <20)
- `calculateRequiredInputForPath`: O(m) where m = path length (typically <10 nodes)
- Overall overhead acceptable (<50ms for typical payment)

## Scalability
- **Moderate Task**:
- Handles up to 50 paths per payment (as per PRD non-functional requirements)
- `calculateRequiredInputForPath`: handles up to 10 exchanges per path
- No scaling issues identified

## Reliability
- **Moderate Task**:
- Part A: More paths available increases payment success rate (improved reliability)
- Part B: Helper functions have error handling (throw exceptions on invalid data)
- `calculateRequiredInputForPath`: validates exchange steps exist before using

## Maintainability
- **Moderate Task**:
- Part A: Simpler loop logic (no complex truncation)
- Part B: Helpers are well-documented and reusable
- Clear separation of concerns (calculation vs processing)

## Cost
- **Moderate Task**: Slight memory increase (more paths in mPathsStats), negligible

## Compliance
- **Moderate Task**: Follows PRD 08 specification
- Adheres to project coding standards
- Maintains backward compatibility (existing behavior preserved when paths already sufficient)

# Restrictions
- Commit changes only after successfully passing the tests (tests will be created in task 08-05)
- Do not modify `addPathForFurtherProcessing` signature or behavior
- Do not modify `ExchangePathsManager` interface
- Helper functions should be protected/private (not public API)
- Verify `tryProcessNextPath()` exists before implementing (may already be present)
