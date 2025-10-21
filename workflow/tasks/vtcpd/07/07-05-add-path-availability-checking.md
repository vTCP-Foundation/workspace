# 07-05 - Add Path Availability Checking in CoordinatorExchangePaymentTransaction

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- [Previous task 1: Task 07-01](07-01-extend-exchangepathsmanager-custom-ttl.md)
- [Previous task 2: Task 07-04](07-04-connect-requestexchangepaths-signal-in-core.md)

# Description
Modify `CoordinatorExchangePaymentTransaction::runPaymentInitializationStage()` to automatically detect missing or expired exchange paths and trigger topology collection when needed. This brings exchange payments to feature parity with single-equivalent payments, which already perform automatic path checking.

The modification checks path availability for all `mExchangeEquivalents` using the custom TTL (150s), requests collection for missing/expired paths via `ResourcesManager`, and waits for `ExchangePathsResource` before proceeding.

Additionally, this task removes `mExchangeAmount` calculation and `kTotalOutgoingPossibilities` validation from the initialization stage, as these will be relocated to the path processing stage (Task 07-06).

# Requirements and DOD

## Functional Requirements
1. **Path availability checking**
   - Iterate through all `mExchangeEquivalents`
   - For each equivalent, create `PathCacheKey{mContractorID, exchangeEquiv, mEquivalent}`
   - Call `mExchangePathsManager->retrievePaths(key, kExchangePathsCacheTTLSeconds)`
   - If `retrievePaths()` returns `nullopt`, add equivalent to `missingEquivalents` list
   - Check performed before any path processing or amount calculation

2. **Resource request for missing paths**
   - If `missingEquivalents` is not empty:
     - Log path collection request
     - Call `mResourcesManager->requestExchangePaths(currentTransactionUUID(), mContractorAddresses[0], missingEquivalents, mEquivalent)`
     - Return `resultWaitForResourceTypes({BaseResource::ExchangePaths}, maxNetworkDelay(4))`
   - If all paths available (empty `missingEquivalents`):
     - Proceed directly to `runPathsResourceProcessingStage()`

3. **Remove exchange amount calculation**
   - Delete all `mExchangeAmount` calculation logic from `runPaymentInitializationStage()`
   - Calculation will be moved to `runPathsResourceProcessingStage()` in Task 07-06

4. **Remove kTotalOutgoingPossibilities check**
   - Delete the outgoing capacity validation (lines ~383-400 in current implementation)
   - Validation will be moved to `runPathsResourceProcessingStage()` in Task 07-06

5. **Add constant kExchangePathsCacheTTLSeconds**
   - Define `static const uint32_t kExchangePathsCacheTTLSeconds = 150;` in class
   - Represents maximum age (in seconds) for cached paths to be considered fresh for payment execution

## Definition of Done
- [ ] `CoordinatorExchangePaymentTransaction.h` declares constant `kExchangePathsCacheTTLSeconds`
- [ ] Path availability checking implemented for all `mExchangeEquivalents`
- [ ] Missing/expired equivalents collected in vector
- [ ] Resource request triggered when paths unavailable
- [ ] Wait state for `ExchangePathsResource` with appropriate timeout
- [ ] Direct transition to path processing when all paths available
- [ ] `mExchangeAmount` calculation removed from initialization stage
- [ ] `kTotalOutgoingPossibilities` validation removed from initialization stage
- [ ] Code compiles without errors or warnings
- [ ] Code follows existing transaction stage patterns
- [ ] Logging added for path checking and resource requests

# Implementation Plan

## Step 1: Add Constant to Header
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

Add the constant in the class definition (private or protected section):

```cpp
class CoordinatorExchangePaymentTransaction : public BaseExchangePaymentTransaction {
    // ... existing members ...

private:
    static const uint32_t kExchangePathsCacheTTLSeconds = 150;

    // ... rest of class ...
};
```

**Location**: Add near other constants (if any exist in the class) or in the private section.

## Step 2: Modify runPaymentInitializationStage() - Part 1: Path Checking
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

Find `CoordinatorExchangePaymentTransaction::runPaymentInitializationStage()` and add path checking logic:

**Insert AFTER existing self-contractor check and initialization, BEFORE any amount calculation:**

```cpp
TransactionResult::SharedConst CoordinatorExchangePaymentTransaction::runPaymentInitializationStage()
{
    // ... [existing self-contractor check code] ...

    info() << "Starting payment initialization for exchange payment";

    // Step 1: Check path availability for all exchange equivalents
    vector<SerializedEquivalent> missingEquivalents;
    missingEquivalents.reserve(mExchangeEquivalents.size());

    for (const auto& exchangeEquiv : mExchangeEquivalents) {
        PathCacheKey key{mContractorID, exchangeEquiv, mEquivalent};

        // Check if paths exist and are fresh using custom TTL (150s)
        // retrievePaths() returns nullopt if paths missing OR expired (age >= 150s)
        auto cachedPaths = mExchangePathsManager->retrievePaths(
            key,
            kExchangePathsCacheTTLSeconds);

        if (!cachedPaths) {
            // Paths not found or expired - need collection
            debug() << "Exchange paths missing or expired for equivalent " << exchangeEquiv
                    << " -> " << mEquivalent;
            missingEquivalents.push_back(exchangeEquiv);
        } else {
            debug() << "Found " << cachedPaths->size() << " cached paths for equivalent "
                    << exchangeEquiv << " -> " << mEquivalent;
        }
    }

    // Step 2: If any equivalents missing, request path collection
    if (!missingEquivalents.empty()) {
        info() << "Exchange paths missing or expired for " << missingEquivalents.size()
               << " of " << mExchangeEquivalents.size() << " equivalents, requesting collection";

        mResourcesManager->requestExchangePaths(
            currentTransactionUUID(),
            mContractorAddresses[0], // Main contractor address
            missingEquivalents,
            mEquivalent); // Receiver equivalent

        // Wait for ExchangePathsResource
        return resultWaitForResourceTypes(
            {BaseResource::ExchangePaths},
            maxNetworkDelay(4)); // 4 hops for topology collection
    }

    info() << "All exchange paths available in cache, proceeding to path processing";

    // Step 3: All paths available, proceed to path processing stage
    // Note: mExchangeAmount calculation moved to runPathsResourceProcessingStage() (Task 07-06)
    mStep = Coordinator_PathsResourceProcessing;
    return runPathsResourceProcessingStage();
}
```

## Step 3: Modify runPaymentInitializationStage() - Part 2: Remove Calculations
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

In the same method, **DELETE the following sections**:

1. **Delete mExchangeAmount calculation** (the entire loop that calculates required payment amount):
   ```cpp
   // DELETE THIS SECTION:
   // TrustLineAmount remainingReceive = mAmount;
   // TrustLineAmount totalPayment = TrustLineAmount(0);
   // for (const auto& exchangeEquiv : mExchangeEquivalents) {
   //     // ... calculation loop ...
   // }
   // mExchangeAmount = totalPayment;
   ```

2. **Delete kTotalOutgoingPossibilities validation** (lines ~383-400):
   ```cpp
   // DELETE THIS SECTION:
   // TrustLineAmount totalOutgoingAmount = TrustLineAmount(0);
   // for (const auto& exchangeEquiv : mExchangeEquivalents) {
   //     auto manager = mEquivalentsSubsystemsRouter->trustLinesManager(exchangeEquiv);
   //     totalOutgoingAmount = totalOutgoingAmount + manager->totalOutgoingAmount();
   // }
   // if (totalOutgoingAmount < mExchangeAmount) {
   //     return resultInsufficientFundsError();
   // }
   ```

**Important**: These sections will be re-added in `runPathsResourceProcessingStage()` in Task 07-06.

## Step 4: Verify Method Flow
After modifications, `runPaymentInitializationStage()` should:

1. Perform self-contractor check (existing code, unchanged)
2. Check path availability for all `mExchangeEquivalents` (new)
3. Request paths if any missing/expired (new)
4. Wait for resource OR proceed to path processing (new)
5. **NOT** calculate `mExchangeAmount` (removed)
6. **NOT** validate outgoing capacity (removed)

## Step 5: Verify Compilation
Build the transaction component:
```bash
cd build-tests
cmake ..
make CoordinatorExchangePaymentTransaction -j4
```

Verify:
- No compilation errors
- No warnings about unused variables (especially `mExchangeAmount` if it's now only set in later stage)
- Path checking logic compiles correctly

## Expected Files Modified
- `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h` (constant)
- `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp` (stage modification)

# Test Plan

## Test Approach
Comprehensive unit tests will be created in Task 07-011. This task focuses on implementation correctness.

## Demo Requirements (Before Commit)
Manual verification through code review and compilation:

1. **Code review checklist**:
   - [ ] Constant `kExchangePathsCacheTTLSeconds = 150` defined correctly
   - [ ] Path checking loop iterates all `mExchangeEquivalents`
   - [ ] `retrievePaths()` called with custom TTL parameter
   - [ ] Missing equivalents collected in vector
   - [ ] Resource request called with correct parameters
   - [ ] Wait state uses `BaseResource::ExchangePaths` type
   - [ ] Direct transition when all paths available
   - [ ] `mExchangeAmount` calculation removed
   - [ ] Outgoing capacity validation removed
   - [ ] Logging added for debugging

2. **Integration verification** (optional, if test environment available):
   - Run existing exchange payment test (if any exists)
   - Verify transaction doesn't crash in initialization stage
   - Check that logging shows path checking activity

3. **Success criteria**:
   - Code compiles without errors
   - Method structure matches PRD specification
   - Removed code sections confirmed deleted
   - New code follows transaction stage patterns

## Testing Notes
- Full test suite in Task 07-011 will cover:
  - All paths available scenario
  - Some paths missing scenario
  - Some paths expired scenario
  - Mixed missing/expired/valid scenario
  - Resource arrival handling
- This task verified via code review and compilation

# Verification and Validation

## Architecture integrity
- **Compliance**: Follows existing resource-based pattern from `CoordinatorPaymentTransaction`
- **Stage separation**: Initialization stage now focused solely on path availability
- **Resource management**: Proper use of `ResourcesManager` for asynchronous path collection
- **Transaction flow**: Wait/resume pattern matches existing transaction stages

## Security
- **No security impact**: Path checking is read-only cache operation
- **Resource timeout**: Prevents indefinite waiting (uses `maxNetworkDelay(4)`)
- **No data exposure**: Transaction UUID prevents resource hijacking

## Performance
- **Path checking overhead**: O(n) where n = number of exchange equivalents (max 5)
- **Cache lookup**: O(1) per equivalent (hash map lookup)
- **Total overhead**: < 50ms for typical case (5 equivalents × ~10ms per check)
- **Resource request**: Asynchronous, no blocking in initialization stage
- **Acceptable performance**: Meets PRD requirement of < 100ms for path checking

## Scalability
- **Exchange equivalent limit**: Respects existing 5-equivalent maximum (PRD 06)
- **Concurrent transactions**: Independent path checking per transaction (no shared state)
- **Cache scalability**: Uses existing `ExchangePathsManager` (already scaled)

## Reliability
- **Timeout handling**: Resource wait has defined timeout (4 hops network delay)
- **Missing paths**: Gracefully triggers collection instead of failing payment
- **Expired paths**: Automatically detected and re-collected
- **Error recovery**: Transaction can timeout and fail gracefully if collection fails

## Maintainability
- **Code clarity**: Path checking logic is straightforward loop
- **Logging**: Debug and info logging aids troubleshooting
- **Separation of concerns**: Initialization stage no longer does amount calculation
- **Future modifications**: Easy to adjust TTL constant or add more equivalents

## Cost
- **Development cost**: ~3-4 hours implementation + testing
- **Testing cost**: Covered in Task 07-011
- **Maintenance cost**: Low, stable infrastructure

## Compliance
- **Policy compliance**: Follows task-driven development (PRD 07)
- **Coding standards**: Adheres to transaction class conventions
- **PRD alignment**: Implements requirements from PRD Section 1 (Path Availability Checking)

# Restrictions
- Commit changes only after successful compilation and code review
- Do not implement tests in this task (tests are in Task 07-011)
- Do not modify `runPathsResourceProcessingStage()` in this task (Task 07-06 handles that)
- Maintain backward compatibility with existing exchange payment behavior (when paths cached)
- Do not change resource timeout logic (use existing `maxNetworkDelay()` pattern)
- Use exact constant value `150` for TTL (as specified in PRD)
