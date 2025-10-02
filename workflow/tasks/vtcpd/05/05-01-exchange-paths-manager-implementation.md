# 05-01 - Exchange Paths Manager Implementation

# Links
- [PRD](/home/mc/Personal/vtcp/vtcpd/workspace/workflow/prd/vtcpd/05-payment-estimation.md)

# Description
Extract data structures (`ExchangePath`, `OptimalPathResult`, `EdgeKey`) from `InitiateMaxFlowExchangeCalculationTransaction` into separate header files, implement `ExchangePathsManager` with TTL-based automatic expiration using `boost::asio::steady_timer`, and integrate with `InitiateMaxFlowExchangeCalculationTransaction` to automatically cache optimal paths split by sender equivalent after max flow calculation completes.

# Requirements and DOD

## Data Structure Extraction
1. Extract `ExchangePath` structure to `/home/mc/Personal/vtcp/vtcpd/src/core/paths/lib/ExchangePath.h`
   - Include all fields: `nodes`, `equivalents`, `exchangeSteps`, `minCapacity`, `effectiveExchangeRate`, `totalCommissions`
   - Include all methods: `calculateMaxCapacity()`, `calculateEffectiveExchangeRate()`, `startsWithEquivalent()`, `isValid()`
   - Extract nested `ExchangeStep` structure within same header
2. Extract `OptimalPathResult` structure to `/home/mc/Personal/vtcp/vtcpd/src/core/paths/lib/OptimalPathResult.h`
   - Include all fields: `path`, `optimal_flow`, `received_amount`, `effective_exchange_rate`, `path_efficiency`
   - Include dependency on `ExchangePath.h`
3. Extract `EdgeKey` structure to `/home/mc/Personal/vtcp/vtcpd/src/core/paths/lib/EdgeKey.h`
   - Include fields: `ContractorID from`, `ContractorID to`, `SerializedEquivalent equivalent`
   - Include `operator<` for use as map key
4. Update `InitiateMaxFlowExchangeCalculationTransaction` to include extracted headers instead of inline definitions

## ExchangePathsManager Implementation
1. Create `ExchangePathsManager` class in `/home/mc/Personal/vtcp/vtcpd/src/core/paths/ExchangePathsManager.h` and `.cpp`
   - Constructor accepts `as::io_context&` and `Logger&`
   - Define `PathCacheKey` structure with fields: `ContractorID contractor`, `SerializedEquivalent senderEquivalent`, `SerializedEquivalent receiverEquivalent`
   - Implement `operator<` for `PathCacheKey` to enable use as map key
2. Implement storage with TTL:
   - Define `CachedPathResult` structure: `vector<OptimalPathResult> paths`, `DateTime computedAt`
   - Member: `map<PathCacheKey, CachedPathResult> mCachedPaths`
   - Member: `mutex mCacheMutex` for thread safety
   - Constant: `kPathResultsTTLSeconds = 600` (10 minutes)
3. Implement public methods:
   - `void storePaths(const PathCacheKey& key, const vector<OptimalPathResult>& paths)`: Store paths with current timestamp, schedule timer if needed
   - `optional<vector<OptimalPathResult>> retrievePaths(const PathCacheKey& key)`: Return paths if valid (within TTL), nullopt otherwise
   - `void invalidatePaths(const PathCacheKey& key)`: Remove cached paths for specific key
   - `void invalidatePathsForContractor(ContractorID contractor)`: Remove all cached paths for contractor
   - `void invalidatePathsForEquivalent(SerializedEquivalent equivalent)`: Remove all cached paths involving equivalent (as sender or receiver)
4. Implement TTL timer management (pattern from `ExchangeRatesManager` and `TopologyTrustLinesManager`):
   - Member: `unique_ptr<as::steady_timer> mExpiryTimer`
   - Private method: `void scheduleExpiryTimer()`: Calculate earliest expiry time, schedule timer
   - Private method: `void onExpiryTimer(const boost::system::error_code& error)`: Timer callback to remove expired paths and reschedule
   - Private method: `DateTime earliestExpiryTime() const`: Find earliest expiry among cached paths
   - Private method: `void removeExpiredPaths()`: Remove all paths exceeding TTL
5. Thread safety:
   - All public methods acquire `mCacheMutex` before accessing `mCachedPaths`
   - Timer callback acquires `mCacheMutex` during cleanup

## Integration with InitiateMaxFlowExchangeCalculationTransaction
1. Add `ExchangePathsManager*` dependency to transaction constructor
2. After OR-Tools optimization completes (in `applyCustomLogic()` after `mOptimalPathResults` populated):
   - Split `mOptimalPathResults[contractorID]` by sender equivalent (first equivalent in each path's `equivalents` vector)
   - For each sender equivalent group, create `PathCacheKey{contractorID, senderEq, mEquivalent}` where `mEquivalent` is the receiver equivalent
   - Call `mExchangePathsManager->storePaths(key, paths)` for each group
3. No changes to existing max flow calculation logic or path ordering

## DOD Criteria
- All data structures extracted to separate headers and successfully included in both `InitiateMaxFlowExchangeCalculationTransaction` and `ExchangePathsManager`
- `ExchangePathsManager` compiles without errors and follows existing manager patterns in codebase
- TTL timer correctly schedules, executes cleanup, and reschedules based on remaining cached paths
- Path splitting logic correctly groups paths by sender equivalent and stores with appropriate keys
- Integration with `InitiateMaxFlowExchangeCalculationTransaction` preserves existing max flow calculation behavior
- No memory leaks in TTL cleanup (verified through manual inspection and test execution)

# Implementation Plan

## Step 1: Extract Data Structures
1. Create directory `/home/mc/Personal/vtcp/vtcpd/src/core/paths/lib/` if not exists
2. Create `ExchangePath.h`:
   - Copy `ExchangeStep` structure from transaction implementation
   - Copy `ExchangePath` structure with all fields and methods
   - Add include guards and necessary dependencies (`ContractorID`, `SerializedEquivalent`, `TrustLineAmount`)
3. Create `OptimalPathResult.h`:
   - Include `ExchangePath.h`
   - Copy `OptimalPathResult` structure with all fields
4. Create `EdgeKey.h`:
   - Define `EdgeKey` structure with `from`, `to`, `equivalent` fields
   - Implement `operator<` for map key usage
5. Update `InitiateMaxFlowExchangeCalculationTransaction.h` and `.cpp`:
   - Replace inline structure definitions with `#include` directives for extracted headers
   - Verify compilation succeeds

## Step 2: Implement ExchangePathsManager Class
1. Create `ExchangePathsManager.h`:
   - Define `PathCacheKey` structure with `operator<`
   - Define `CachedPathResult` structure
   - Declare class with constructor, public methods (store, retrieve, invalidate variants), private timer methods
   - Add data members: `mCachedPaths`, `mCacheMutex`, `mExpiryTimer`, `mLog`, `mIOContext`
2. Create `ExchangePathsManager.cpp`:
   - Implement constructor: initialize timer, logger, io_context reference
   - Implement `storePaths()`: acquire lock, insert into map with current timestamp, call `scheduleExpiryTimer()`
   - Implement `retrievePaths()`: acquire lock, check if key exists, validate TTL, return paths or nullopt
   - Implement invalidation methods: acquire lock, remove matching entries from map
   - Implement `scheduleExpiryTimer()`: calculate earliest expiry, set timer duration, async_wait with `onExpiryTimer` callback
   - Implement `onExpiryTimer()`: acquire lock, call `removeExpiredPaths()`, reschedule timer if paths remain
   - Implement `earliestExpiryTime()`: iterate `mCachedPaths`, find minimum `computedAt + TTL`
   - Implement `removeExpiredPaths()`: calculate cutoff time, erase entries with `computedAt < cutoff`

## Step 3: Integrate with InitiateMaxFlowExchangeCalculationTransaction
1. Add `ExchangePathsManager* mExchangePathsManager` member to transaction
2. Update constructor to accept and initialize `mExchangePathsManager` parameter
3. Locate point in `applyCustomLogic()` where `mOptimalPathResults[contractorID]` is fully populated
4. Add path splitting logic:
   ```cpp
   // Split paths by sender equivalent
   map<SerializedEquivalent, vector<OptimalPathResult>> pathsBySenderEq;
   for (const auto& pathResult : mOptimalPathResults[contractorID]) {
       SerializedEquivalent senderEq = pathResult.path.equivalents.front();
       pathsBySenderEq[senderEq].push_back(pathResult);
   }

   // Store each group with appropriate key
   for (const auto& [senderEq, paths] : pathsBySenderEq) {
       PathCacheKey key{contractorID, senderEq, mEquivalent};
       mExchangePathsManager->storePaths(key, paths);
   }
   ```
5. Update transaction factory to inject `ExchangePathsManager` dependency

## Step 4: Verification
1. Build project and verify no compilation errors
2. Run existing max flow calculation tests to ensure no regression
3. Inspect implementation for potential memory leaks (timer lifetime management)
4. Review thread safety: all map accesses protected by mutex

# Test Plan

## Complexity Level
Moderate

## Test Scope
- Data structure extraction: verify successful compilation and inclusion
- ExchangePathsManager: store, retrieve, TTL expiration, invalidation, thread safety
- Integration: verify path splitting and storage without affecting max flow calculation

## Test Approach (Unit Tests Only)
All tests execute in `build-tests` with no Docker dependencies. Mock all external dependencies (topology, commissions, etc.).

### Data Structure Tests
1. **Compilation Test**: Verify all headers include successfully and compile without errors
2. **ExchangePath Methods Test**: Verify `startsWithEquivalent()`, `isValid()`, `calculateMaxCapacity()` work correctly with mock data
3. **EdgeKey Ordering Test**: Verify `operator<` correctly orders EdgeKey instances for map usage

### ExchangePathsManager Tests
1. **Store and Retrieve Test**: Store paths with specific `PathCacheKey`, retrieve immediately, verify exact match
2. **TTL Expiration Test**:
   - Mock `DateTime::now()` to control time progression
   - Store paths, advance mock time beyond TTL (600s), trigger timer callback
   - Verify expired paths removed automatically
3. **Invalidate by Key Test**: Store multiple keys, invalidate one specific key, verify only that key removed
4. **Invalidate by Contractor Test**: Store paths for same contractor with different equivalents, invalidate contractor, verify all removed
5. **Invalidate by Equivalent Test**: Store paths with mixed sender/receiver equivalents, invalidate one equivalent, verify correct subset removed
6. **Thread Safety Test**: Use multiple threads to concurrently store and retrieve paths, verify no race conditions or crashes
7. **Edge Cases**:
   - Retrieve non-existent key: returns nullopt
   - Store empty path list: stores successfully, retrieves empty list
   - Same-equivalent scenario: `senderEq == receiverEq`, verify storage and retrieval work correctly

### Integration Tests
1. **Path Splitting Test**:
   - Mock `mOptimalPathResults` with 3 paths: 2 starting in equivalent 1, 1 starting in equivalent 2
   - Run transaction, verify `ExchangePathsManager` receives two `storePaths()` calls with correct keys and path counts
2. **Receiver Equivalent Binding Test**:
   - Run transaction with `mEquivalent = 5` (receiver equivalent)
   - Verify all stored `PathCacheKey` instances have `receiverEquivalent = 5`
3. **Regression Test**: Run existing max flow calculation tests, verify identical `mOptimalPathResults` output before and after integration

## Success Criteria
- All unit tests pass in `build-tests`
- No memory leaks detected during TTL timer tests
- Thread safety tests complete without data corruption or crashes
- Path splitting produces correct key-path associations for all test scenarios
- Existing max flow calculation behavior unchanged (regression tests pass)

# Verification and Validation

## Architecture integrity
- **ExchangePathsManager** follows existing manager patterns (`ExchangeRatesManager`, `TopologyTrustLinesManager`) with TTL timer using `boost::asio::steady_timer`
- Data structure extraction reduces coupling and improves reusability across transaction and manager components
- Integration with `InitiateMaxFlowExchangeCalculationTransaction` is minimally invasive (additive only)
- Path splitting by sender equivalent enables future per-equivalent estimation queries

## Security
- No cryptographic material stored in cached paths (only topology and flow data)
- TTL expiration ensures bounded cache lifetime (600s), reducing stale data exposure
- Thread safety via mutex prevents race conditions in concurrent access scenarios
- No authentication/authorization logic required (inherits from parent transaction context)

## Performance
- In-memory storage provides fast access (O(log n) map lookup by PathCacheKey)
- TTL timer avoids continuous polling; cleanup triggered only on expiration
- Path splitting logic executes once per max flow calculation (minimal overhead)
- Memory usage bounded by TTL and practical limits on concurrent contractor-equivalent pairs

## Scalability
- Supports up to 100 cached (contractor, sender_eq, receiver_eq) keys simultaneously (per PRD target)
- Each cached entry can store up to 100 paths (typical path set size)
- TTL cleanup scales linearly with number of cached entries
- Thread-safe design enables concurrent estimation queries without blocking

## Reliability
- TTL expiration guarantees automatic purging of stale paths (no manual cleanup required)
- Mutex-based thread safety prevents data corruption under concurrent access
- Integration with existing transaction preserves max flow calculation reliability
- Error handling: missing paths return nullopt (no exceptions for normal cache misses)

## Maintainability
- Extracted data structures in separate headers improve code organization and readability
- Manager follows established patterns, reducing learning curve for future maintainers
- Path splitting logic is localized and well-commented
- TTL timer implementation reuses proven pattern from existing managers

## Cost
- Minimal memory overhead: ~10MB for 100 cached contractor-equivalent pairs (per PRD estimate)
- No persistent storage costs (in-memory only)
- Negligible CPU overhead for timer-based cleanup

## Compliance
- Follows project policy: task-driven development, no scope creep, explicit DOD criteria
- Adheres to existing code conventions and architectural patterns
- No external dependencies introduced (reuses boost::asio)

# Restrictions
- Commit changes only after successfully passing the tests (if they are provided for by the task)
