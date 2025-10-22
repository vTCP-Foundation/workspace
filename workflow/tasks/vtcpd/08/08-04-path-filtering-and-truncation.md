# 08-04 - Path Filtering and Capacity Truncation

# Links
- [PRD](../../../prd/vtcpd/08-path-capacity-adjustment.md)
- [Previous task](08-03-path-addition-and-helpers.md)

# Description
Implement path filtering and capacity truncation logic before or during `switchToNextPath()` in `CoordinatorExchangePaymentTransaction`. This prevents wasting processing cycles on paths that are known to be problematic and ensures capacity is appropriately truncated when a path would over-reserve.

**Path Filtering**: Before taking a path into processing, validate it doesn't contain:
- Nodes from `mInaccessibleNodes` (nodes that failed to respond or rejected previous requests)
- Trust line edges from `mRejectedTrustLines` (trust lines that were rejected)

**Capacity Truncation**: Before processing a path, check if it would exceed remaining needed capacity. If yes, truncate its capacity to deliver exactly the remaining amount (not more).

**Key technical note**: Use `path.nodes` (BaseAddress::Shared) for filtering because `mInaccessibleNodes` and `mRejectedTrustLines` store addresses. The `path.nodes` field is populated by `addPathForFurtherProcessing` (task 08-03) and is guaranteed to be available at this stage.

# Requirements and DOD

## Functional Requirements

### 1. Path Filtering Logic
**Location**: Before or at beginning of `switchToNextPath()` method

1. **Get next path**:
   - Retrieve next PathID from processing queue (`mPathIDs`)
   - Find path in `mPathsStats`
   - If not found: call `tryProcessNextPath()` and return

2. **Validate path against inaccessible nodes**:
   - Iterate through `path.nodes` (BaseAddress::Shared)
   - Check if any node appears in `mInaccessibleNodes`
   - If found: mark path unusable, call `tryProcessNextPath()`, return

3. **Validate path against rejected trust lines**:
   - Iterate through edges in `path.nodes` (consecutive pairs)
   - Check if any edge `(nodes[i], nodes[i+1])` appears in `mRejectedTrustLines`
   - If found: mark path unusable, call `tryProcessNextPath()`, return

4. **Precondition verification**:
   - `path.nodes` must be populated (guaranteed by `addPathForFurtherProcessing` in task 08-03)
   - If empty, log error and skip path

### 2. Capacity Truncation Logic
**Location**: After filtering, before actually processing path

1. **Calculate total reserved so far**:
   - Call `calculateTotalReservedAmount()` (from task 08-03)
   - If `totalReserved >= mAmount`: call `proceedToNextStage()`, return

2. **Calculate remaining needed**:
   - `remainingNeeded = mAmount - totalReserved`

3. **Check if truncation needed**:
   - If `pathStats->received_amount <= remainingNeeded`: no truncation, proceed normally
   - If `pathStats->received_amount > remainingNeeded`: truncate as follows

4. **Truncate path capacity**:
   - Calculate truncated input: call `calculateRequiredInputForPath(pathStats, remainingNeeded)`
   - Update path: `pathStats->shortageMaxFlow(truncatedInput)`
   - Update received: `pathStats->received_amount = remainingNeeded`
   - Recalculate flows: `pathStats->calculateFlows(truncatedInput)`
   - Log truncation action

5. **Proceed with processing**:
   - Call actual `switchToNextPath()` or path processing logic

## Definition of Done
- [ ] Path filtering logic implemented before/in `switchToNextPath()`
- [ ] Uses `path.nodes` for filtering (NOT `path.ids`)
- [ ] Checks `mInaccessibleNodes`: if match found, path marked unusable and skipped
- [ ] Checks `mRejectedTrustLines`: if edge match found, path marked unusable and skipped
- [ ] Capacity truncation logic implemented
- [ ] Calls `calculateTotalReservedAmount()` to determine remaining need
- [ ] If sufficient capacity: calls `proceedToNextStage()` and exits
- [ ] If path exceeds remaining: truncates using `calculateRequiredInputForPath()`
- [ ] Updates `mMaxPathFlow`, `received_amount`, and `flows` after truncation
- [ ] Appropriate logging at each decision point (info for filtering, debug for truncation)
- [ ] No crashes on edge cases (empty nodes, missing paths, etc.)

# Implementation Plan

## Step 1: Understand current path processing flow
- Locate `switchToNextPath()` method or equivalent
- Understand how paths are currently taken from `mPathIDs`
- Identify best insertion point for filtering and truncation logic

## Step 2: Implement path filtering validation function
Create helper or inline validation:

```cpp
// Option 1: Inline before switchToNextPath()
// Option 2: Helper method (recommended for clarity)

bool CoordinatorExchangePaymentTransaction::validatePathForProcessing(
    const OptimalPathResult *pathStats)
{
    const auto &path = pathStats->path();

    // Important: Use path.nodes for filtering (BaseAddress::Shared)
    // path.nodes is populated by addPathForFurtherProcessing and guaranteed available here

    // Check for inaccessible nodes
    for (const auto &nodeAddress : path.nodes) {
        if (std::find(mInaccessibleNodes.begin(),
                      mInaccessibleNodes.end(),
                      nodeAddress) != mInaccessibleNodes.end()) {
            info() << "Path contains inaccessible node: "
                   << nodeAddress->fullAddress();
            return false;
        }
    }

    // Check for rejected trust lines
    for (size_t i = 0; i + 1 < path.nodes.size(); ++i) {
        auto source = path.nodes[i];
        auto dest = path.nodes[i + 1];

        for (const auto &[rejSource, rejDest] : mRejectedTrustLines) {
            if (source == rejSource && dest == rejDest) {
                info() << "Path contains rejected trust line: "
                       << source->fullAddress() << " -> "
                       << dest->fullAddress();
                return false;
            }
        }
    }

    return true;  // Path is valid
}
```

## Step 3: Implement filtering and truncation in path processing
Insert before/during `switchToNextPath()`:

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

// Step 3: Validate path
if (!validatePathForProcessing(pathStats)) {
    pathStats->setUnusable();
    return tryProcessNextPath();
}

// Step 4: Truncate capacity if needed
if (pathStats->received_amount > remainingNeeded) {
    info() << "Truncating path capacity: "
           << "available=" << pathStats->received_amount
           << ", needed=" << remainingNeeded;

    try {
        // Calculate truncated input amount that delivers exactly remainingNeeded
        TrustLineAmount truncatedInput = calculateRequiredInputForPath(
            *pathStats, remainingNeeded);

        // Update path capacity
        pathStats->shortageMaxFlow(truncatedInput);
        pathStats->received_amount = remainingNeeded;

        // Recalculate flows for reservation
        pathStats->calculateFlows(truncatedInput);

        info() << "Path capacity truncated to: "
               << "input=" << truncatedInput
               << ", output=" << remainingNeeded;

    } catch (const std::exception &e) {
        error() << "Error truncating path capacity: " << e.what();
        // Truncation failed, skip this path and try next
        pathStats->setUnusable();
        return tryProcessNextPath();
    }
}

// Step 5: Proceed with path processing
switchToNextPath();  // or actual processing logic
```

## Step 4: Handle edge cases
- **Empty `path.nodes`**: Log error, skip path (though should be prevented by precondition from task 08-03)
- **Truncation calculation failure**: Mark path unusable, skip to next path, log error
- **All paths filtered**: Eventually reach end of `mPathIDs`, return error if insufficient capacity
- **No `mInaccessibleNodes` or `mRejectedTrustLines`**: Filtering logic still runs (empty checks), no impact

## Step 5: Add helper declaration to header if using separate validation method
File: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

```cpp
protected:  // or private:
    bool validatePathForProcessing(const OptimalPathResult *pathStats);
```

# Test Plan
Tests will be implemented in a separate task (08-05). This task focuses on implementation only.

Expected test scenarios (for reference):
- Path with inaccessible node (should be filtered)
- Path with rejected trust line (should be filtered)
- Path exceeding remaining need (should be truncated)
- Path fitting remaining need (no truncation)
- All paths filtered (should return error)

# Verification and Validation

## Architecture integrity
- **Moderate Task Validation**: Verify integration with path processing flow
- Uses `path.nodes` for filtering (as per PRD technical note)
- Uses helper functions from task 08-03 (no duplication)
- Calls existing `setUnusable()` for path invalidation
- Calls existing `tryProcessNextPath()` for skipping
- Calls existing `shortageMaxFlow()` and `calculateFlows()` for truncation

## Security
- **Moderate Task**: No security implications (internal filtering)
- Prevents processing of known-bad paths (improves reliability, not security)
- No exposure of filter criteria to external parties

## Performance
- **Moderate Task**:
- Path filtering: O(n*m) where n = path length (~10), m = inaccessible nodes (~10), acceptable (<1ms)
- Trust line filtering: O(n*k) where n = path length, k = rejected trust lines (~100), acceptable (<1ms)
- Truncation calculation: O(n) where n = path length, acceptable (<10ms)
- Total overhead: <20ms per path, acceptable

## Scalability
- **Moderate Task**:
- Handles up to 100 inaccessible nodes (as per PRD non-functional requirements)
- Handles up to 500 rejected trust lines (as per PRD)
- No scaling issues with path length (limited to ~10 nodes)

## Reliability
- **Moderate Task**:
- Improves overall payment reliability (avoids known-bad paths)
- Prevents over-reservation (truncation ensures exact capacity)
- Graceful error handling on truncation failure (uses full capacity)
- Robust to missing data (empty filter lists → no filtering)

## Maintainability
- **Moderate Task**:
- Clear separation: filtering validation in separate method (if used)
- Well-commented code explains `path.nodes` usage
- Logging provides debugging information
- Error handling is explicit

## Cost
- **Moderate Task**: Minimal overhead (~20ms per path), negligible

## Compliance
- **Moderate Task**: Follows PRD 08 specification exactly
- Adheres to project coding standards
- Uses correct data structure (`path.nodes`) as specified in PRD

# Restrictions
- Commit changes only after successfully passing the tests (tests will be created in task 08-05)
- Must use `path.nodes` for filtering (NOT `path.ids`) - critical technical requirement
- Do not modify `mInaccessibleNodes` or `mRejectedTrustLines` tracking (read-only access)
- Do not modify `shortageMaxFlow()` or `calculateFlows()` methods (use as-is)
- Truncation is optional optimization (failure should not prevent payment, use full capacity as fallback)
