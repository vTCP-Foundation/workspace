# 07-06 - Relocate Exchange Amount Calculation to Path Processing Stage

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- [Previous task: Task 07-05](07-05-add-path-availability-checking.md)

# Description
Move `mExchangeAmount` calculation and `kTotalOutgoingPossibilities` validation from `runPaymentInitializationStage()` to `runPathsResourceProcessingStage()` in `CoordinatorExchangePaymentTransaction`. This ensures that exchange amount calculation only occurs after paths are guaranteed to be available in the cache (either found during initialization or collected via resource request).

This relocation prevents calculation failures due to missing paths and ensures proper timing: path availability → calculation → validation → processing.

# Requirements and DOD

## Functional Requirements
1. **Add mExchangeAmount calculation to runPathsResourceProcessingStage()**
   - Insert calculation at the very beginning of the method (before "Step 1: Initialize total flow counter")
   - Use exact algorithm from PRD 06 (Exchange Payment with Commissions)
   - Calculate required sender payment amount to deliver requested receiver amount
   - Handle insufficient path capacity gracefully

2. **Add kTotalOutgoingPossibilities validation**
   - Insert validation immediately after `mExchangeAmount` calculation
   - Sum total outgoing capacity across all `mExchangeEquivalents`
   - Compare `totalOutgoingAmount` with `mExchangeAmount`
   - Return `resultInsufficientFundsError()` if insufficient capacity

3. **Error handling**
   - Return `resultNoPathsError()` if cached paths missing (shouldn't happen, but defensive)
   - Return `resultInsufficientFundsError()` if paths can't deliver full amount
   - Return `resultInsufficientFundsError()` if outgoing capacity insufficient
   - Return `resultProtocolError()` for calculation exceptions

4. **Maintain existing path processing logic**
   - Do NOT modify existing path retrieval and processing code
   - Calculation and validation are insertions at method start
   - Existing "Step 1", "Step 2", etc. remain unchanged (just renumbered if needed)

## Definition of Done
- [ ] `mExchangeAmount` calculation added at start of `runPathsResourceProcessingStage()`
- [ ] Calculation uses exact algorithm from PRD 06
- [ ] Missing paths detected and return `resultNoPathsError()`
- [ ] Insufficient path capacity returns `resultInsufficientFundsError()`
- [ ] Outgoing capacity validation added after calculation
- [ ] Validation sums capacity across all `mExchangeEquivalents`
- [ ] Insufficient outgoing capacity returns `resultInsufficientFundsError()`
- [ ] Exception handling wraps calculation logic
- [ ] Existing path processing logic unchanged
- [ ] Code compiles without errors or warnings
- [ ] Logging added for calculation and validation

# Implementation Plan

## Step 1: Locate runPathsResourceProcessingStage()
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

Find the method `CoordinatorExchangePaymentTransaction::runPathsResourceProcessingStage()`.

Current structure (before modification):
```cpp
TransactionResult::SharedConst CoordinatorExchangePaymentTransaction::runPathsResourceProcessingStage()
{
    // Step 1: Initialize total flow counter
    TrustLineAmount totalAddedFlow = TrustLineAmount(0);

    // ... existing path processing ...
}
```

## Step 2: Insert Exchange Amount Calculation
**Location**: At the very beginning of `runPathsResourceProcessingStage()`, BEFORE any existing code.

**Code to insert**:

```cpp
TransactionResult::SharedConst CoordinatorExchangePaymentTransaction::runPathsResourceProcessingStage()
{
    info() << "Processing exchange paths for payment execution";

    // Step 0: Calculate mExchangeAmount (moved from runPaymentInitializationStage)
    // This calculation now occurs after path availability is guaranteed
    try {
        TrustLineAmount remainingReceive = mAmount;  // Receiver amount to deliver
        TrustLineAmount totalPayment = TrustLineAmount(0);  // Sender amount to pay

        debug() << "Calculating required exchange amount to deliver " << mAmount
                << " in receiver equivalent " << mEquivalent;

        // Calculate required payment amount using cached paths
        for (const auto& exchangeEquiv : mExchangeEquivalents) {
            if (remainingReceive == TrustLineAmount(0)) {
                break;  // Full amount already covered by previous equivalents
            }

            PathCacheKey key{mContractorID, exchangeEquiv, mEquivalent};
            auto cachedPaths = mExchangePathsManager->retrievePaths(key);

            if (!cachedPaths) {
                // Paths disappeared between initialization and processing (rare edge case)
                warning() << "Cached paths missing for equivalent " << exchangeEquiv
                          << " -> " << mEquivalent << " between initialization and processing";
                return resultNoPathsError();
            }

            debug() << "Processing " << cachedPaths->size() << " cached paths for equivalent "
                    << exchangeEquiv;

            for (const auto &pathResult : *cachedPaths) {
                if (remainingReceive == TrustLineAmount(0)) {
                    break;  // Full amount covered
                }

                // Determine how much this path can deliver
                TrustLineAmount deliveredAmount = min(remainingReceive, pathResult.received_amount);

                // Calculate required payment for this delivered amount using ratio
                double ratio = deliveredAmount.convert_to<double>() /
                               pathResult.received_amount.convert_to<double>();
                TrustLineAmount requiredPayment(
                    static_cast<uint64_t>(pathResult.optimal_flow.convert_to<double>() * ratio));

                totalPayment = totalPayment + requiredPayment;
                remainingReceive = remainingReceive - deliveredAmount;

                debug() << "Path delivers " << deliveredAmount << ", requires payment "
                        << requiredPayment << " (ratio=" << ratio << ")";
            }
        }

        // Check if paths can deliver full amount
        if (remainingReceive > TrustLineAmount(0)) {
            warning() << "Insufficient path capacity: need to deliver " << mAmount
                      << ", but paths can only deliver " << (mAmount - remainingReceive);
            return resultInsufficientFundsError();
        }

        mExchangeAmount = totalPayment;
        info() << "Calculated exchange amount: " << mExchangeAmount
               << " (sender equivalent) to deliver " << mAmount
               << " (receiver equivalent " << mEquivalent << ")";

    } catch (const exception &e) {
        error() << "Error calculating exchange amount: " << e.what();
        return resultProtocolError();
    }

    // Step 0.5: Check total outgoing capacity (moved from runPaymentInitializationStage)
    // Validate that we have sufficient outgoing capacity across all exchange equivalents
    TrustLineAmount totalOutgoingAmount = TrustLineAmount(0);

    for (const auto& exchangeEquiv : mExchangeEquivalents) {
        auto manager = mEquivalentsSubsystemsRouter->trustLinesManager(exchangeEquiv);
        TrustLineAmount equivalentOutgoing = manager->totalOutgoingAmount();
        totalOutgoingAmount = totalOutgoingAmount + equivalentOutgoing;

        debug() << "Equivalent " << exchangeEquiv << " has outgoing capacity "
                << equivalentOutgoing;
    }

    debug() << "Total outgoing capacity across exchange equivalents: " << totalOutgoingAmount
            << ", required: " << mExchangeAmount;

    if (totalOutgoingAmount < mExchangeAmount) {
        warning() << "Insufficient total outgoing capacity: have " << totalOutgoingAmount
                  << ", need " << mExchangeAmount;
        return resultInsufficientFundsError();
    }

    info() << "Outgoing capacity validation passed";

    // Step 1: Initialize total flow counter (existing code continues below)
    TrustLineAmount totalAddedFlow = TrustLineAmount(0);

    // ... [rest of existing path processing logic] ...
}
```

## Step 3: Verify Existing Code Unchanged
After insertion, verify that:
- Existing path processing logic starts after Step 0 and Step 0.5
- No modifications to path retrieval loops (they remain as-is)
- No modifications to flow distribution logic
- Only additions are at the method start

## Step 4: Update Step Numbering (Optional)
If the existing code uses "Step 1", "Step 2", etc. in comments, optionally renumber:
- New calculation: "Step 0"
- New validation: "Step 0.5"
- Existing steps: "Step 1", "Step 2", etc. (unchanged)

This is cosmetic and not required for functionality.

## Step 5: Verify Compilation
Build the transaction component:
```bash
cd build-tests
cmake ..
make CoordinatorExchangePaymentTransaction -j4
```

Verify:
- No compilation errors
- No warnings about uninitialized `mExchangeAmount`
- All type conversions compile correctly

## Expected Files Modified
- `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp` (method modification)

# Test Plan

## Test Approach
Comprehensive unit tests will be created in Task 07-011 (combined with Task 07-05 tests). This task focuses on correct implementation.

## Demo Requirements (Before Commit)
Manual verification through code review:

1. **Code review checklist**:
   - [ ] Calculation inserted at method start (before existing code)
   - [ ] Uses exact algorithm from PRD 06
   - [ ] Handles missing paths defensively (returns `resultNoPathsError()`)
   - [ ] Checks insufficient path capacity (returns `resultInsufficientFundsError()`)
   - [ ] Outgoing capacity validation added
   - [ ] Validation sums across all exchange equivalents
   - [ ] Exception handling wraps calculation
   - [ ] Existing path processing unchanged
   - [ ] Logging added for debugging

2. **Algorithm verification**:
   - Calculation iterates `mExchangeEquivalents` in order
   - For each equivalent, retrieves cached paths
   - For each path, calculates ratio and required payment
   - Accumulates total payment in `mExchangeAmount`
   - Checks remaining delivery amount

3. **Success criteria**:
   - Code compiles without errors
   - Calculation matches PRD 06 specification
   - Error handling covers all failure modes
   - Existing functionality preserved

## Testing Notes
- Full test suite in Task 07-011 covers:
  - Successful calculation scenarios
  - Missing paths edge case
  - Insufficient path capacity
  - Insufficient outgoing capacity
  - Calculation exceptions
- This task verified via code review and compilation

# Verification and Validation

## Architecture integrity
- **Compliance**: Calculation now occurs after paths guaranteed available (correct timing)
- **Stage separation**: Path processing stage handles both calculation and execution
- **Error propagation**: Proper error returns maintain transaction flow integrity
- **Defensive programming**: Checks for missing paths even though shouldn't occur

## Security
- **No security impact**: Calculation uses cached paths (already validated during collection)
- **Exception handling**: Prevents crashes from arithmetic errors
- **No data exposure**: Calculation is local to transaction

## Performance
- **Calculation complexity**: O(p) where p = total number of cached paths across equivalents
- **Typical case**: ~5-20 paths across 2-3 equivalents = 10-60 iterations
- **Overhead**: < 10ms for typical case (simple arithmetic operations)
- **Validation overhead**: < 5ms (one trust line manager query per equivalent)
- **Acceptable performance**: Total overhead < 15ms, meets PRD requirements

## Scalability
- **Exchange equivalent limit**: Respects 5-equivalent maximum
- **Path count**: Scales with existing path processing (already optimized)
- **Memory usage**: Minimal (temporary variables only)

## Reliability
- **Missing paths**: Detected and handled gracefully
- **Insufficient capacity**: Returns appropriate error to user
- **Exceptions**: Caught and converted to protocol error
- **Edge cases**: Zero amounts, single path, multiple equivalents all handled

## Maintainability
- **Code clarity**: Calculation logic is straightforward iteration
- **Logging**: Detailed logging aids troubleshooting
- **Algorithm preservation**: Uses exact PRD 06 algorithm (no modifications)
- **Future modifications**: Easy to adjust calculation if commission rules change

## Cost
- **Development cost**: ~2-3 hours implementation
- **Testing cost**: Covered in Task 07-011
- **Maintenance cost**: Low, stable algorithm

## Compliance
- **Policy compliance**: Follows task-driven development (PRD 07)
- **Coding standards**: Adheres to transaction class conventions
- **Algorithm compliance**: Exact implementation of PRD 06 specification
- **PRD alignment**: Implements requirements from PRD Section 5 (Exchange Amount Calculation Relocation)

# Restrictions
- Commit changes only after successful compilation and code review
- Do not implement tests in this task (tests are in Task 07-011)
- Do not modify calculation algorithm (use exact PRD 06 specification)
- Do not modify existing path processing logic (only insert at method start)
- Maintain exact error return types from PRD specification
- Use exact logging format from existing transaction code
