# 07-01 - Extend ExchangePathsManager with Custom TTL Support

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- Previous tasks: None (foundation task)

# Description
Extend `ExchangePathsManager::retrievePaths()` method to support custom TTL (Time-To-Live) values for different use cases. Currently, the method uses a fixed 600-second TTL for all path cache expiry checks. This task adds an optional `customTTL` parameter to allow callers to specify shorter TTLs (e.g., 150s for payment execution) while maintaining backward compatibility with the default 600s TTL for estimation use cases.

Additionally, this task standardizes TTL expiry semantics to `age >= TTL` (inclusive boundary), ensuring consistency across the codebase.

# Requirements and DOD

## Functional Requirements
1. **Add customTTL parameter to retrievePaths()**
   - Method signature: `optional<vector<OptimalPathResult>> retrievePaths(const PathCacheKey &key, optional<uint32_t> customTTL = nullopt)`
   - Parameter is optional with default value `nullopt`
   - When `nullopt`, use existing `kPathResultsTTLSeconds` (600s)
   - When provided, use the custom value for expiry checking

2. **Implement TTL expiry semantics**
   - Expiry condition: `age >= customTTL` (inclusive, not strict greater-than)
   - `age` calculated as: `utc_now() - cachedResult.computedAt`
   - Paths with `age >= TTL` are considered expired and removed from cache
   - Paths with `age < TTL` are considered valid

3. **Maintain backward compatibility**
   - Existing calls without `customTTL` parameter continue to work unchanged
   - Default behavior (600s TTL) preserved for existing code
   - No breaking changes to method interface

4. **Return behavior**
   - Return `nullopt` if key not found in cache
   - Return `nullopt` if paths expired according to TTL
   - Return `vector<OptimalPathResult>` if paths valid

## Definition of Done
- [ ] `ExchangePathsManager.h` updated with new method signature
- [ ] `ExchangePathsManager.cpp` implements customTTL logic correctly
- [ ] TTL semantics changed to `age >= TTL` (from previous `expiresAt <= now`)
- [ ] Method uses `customTTL.value_or(kPathResultsTTLSeconds)` for TTL selection
- [ ] Expired paths are removed from cache (existing behavior preserved)
- [ ] Method compiles without errors or warnings
- [ ] Backward compatibility verified (no changes needed in existing callers)
- [ ] Code follows existing `ExchangePathsManager` style and conventions
- [ ] Thread safety maintained (mutex lock usage unchanged)

# Implementation Plan

## Step 1: Update Method Signature
**File**: `src/core/paths/ExchangePathsManager.h`

Update the `retrievePaths()` declaration:
```cpp
optional<vector<OptimalPathResult>> retrievePaths(
    const PathCacheKey &key,
    optional<uint32_t> customTTL = nullopt);
```

## Step 2: Implement Custom TTL Logic
**File**: `src/core/paths/ExchangePathsManager.cpp`

Modify the `retrievePaths()` implementation:

1. **Current implementation reference** (lines 55-79):
   - Already has mutex lock
   - Already checks for key existence
   - Already calculates expiry
   - Already removes expired entries

2. **Changes needed**:
   ```cpp
   optional<vector<OptimalPathResult>> ExchangePathsManager::retrievePaths(
       const PathCacheKey &key,
       optional<uint32_t> customTTL)
   {
       lock_guard<mutex> lock(mCacheMutex);

       auto it = mCachedPaths.find(key);
       if (it == mCachedPaths.end()) {
           return nullopt;
       }

       // Use custom TTL if provided, otherwise default
       uint32_t ttlToUse = customTTL.value_or(kPathResultsTTLSeconds);

       auto now = utc_now();
       auto age = now - it->second.computedAt;

       // TTL Semantics: age >= TTL means expired
       if (age.total_seconds() >= ttlToUse) {
           debug() << "Cached paths expired (age=" << age.total_seconds()
                   << "s, TTL=" << ttlToUse << "s), removing";
           mCachedPaths.erase(it);
           return nullopt;
       }

       debug() << "Retrieved " << it->second.paths.size() << " cached paths";
       return it->second.paths;
   }
   ```

3. **Key implementation notes**:
   - Replace `auto expiresAt = it->second.computedAt + boost::posix_time::seconds(kPathResultsTTLSeconds);` with age calculation
   - Replace `if (expiresAt <= now)` with `if (age.total_seconds() >= ttlToUse)`
   - Add TTL value to debug logging for traceability
   - Maintain all existing mutex locking and cache manipulation

## Step 3: Verify Compilation
- Build the project: `cd build-tests && cmake .. && make ExchangePathsManager -j4`
- Verify no compilation errors or warnings
- Check that existing code using `retrievePaths()` compiles without changes

## Step 4: Code Review Checks
- [ ] Mutex usage correct (no deadlocks, lock held during cache access)
- [ ] Logging consistent with existing patterns
- [ ] Error handling appropriate (no uncaught exceptions)
- [ ] Code style matches existing `ExchangePathsManager` methods
- [ ] No unnecessary allocations or performance regressions

## Expected Files Modified
- `src/core/paths/ExchangePathsManager.h` (method signature)
- `src/core/paths/ExchangePathsManager.cpp` (method implementation)

# Test Plan

## Test Approach
Unit tests will be created in a separate testing task (Task 07-07). This task focuses solely on implementation.

## Demo Requirements (Before Commit)
Create a simple demonstration showing:

1. **Demo script location**: `workspace/demos/07-1-custom-ttl-demo.cpp` (temporary, not committed)

2. **Demo scenarios**:
   - Call `retrievePaths(key)` without customTTL → uses 600s TTL
   - Call `retrievePaths(key, 150)` with 150s TTL → uses 150s TTL
   - Store paths, wait 149s, retrieve with TTL=150 → returns paths (valid)
   - Store paths, wait 150s, retrieve with TTL=150 → returns nullopt (expired)
   - Store paths, wait 160s, retrieve with TTL=600 → returns paths (valid with longer TTL)
   - Verify debug logging shows correct TTL values

3. **Demo execution**:
   ```bash
   # Compile demo
   g++ -std=c++17 -I src workspace/demos/07-1-custom-ttl-demo.cpp \
       -o workspace/demos/07-1-custom-ttl-demo -lpthread -lboost_system

   # Run demo
   ./workspace/demos/07-1-custom-ttl-demo
   ```

4. **Success criteria**:
   - Demo compiles and runs without errors
   - All scenarios produce expected results
   - Logging shows correct TTL values being used
   - No segmentation faults or crashes

## Testing Notes
- Full unit test suite will be implemented in Task 07-07
- This task requires only functional demonstration before commit
- Demo code is temporary and not committed to repository

# Verification and Validation

## Architecture integrity
- **Compliance**: Changes extend existing API without modifying architecture
- **Pattern consistency**: Follows existing optional parameter pattern in codebase
- **Encapsulation**: Maintains encapsulation of `mCachedPaths` (no direct external access)
- **Thread safety**: Mutex locking unchanged, no new race conditions introduced

## Security
- **No security impact**: TTL is a timing parameter, no security implications
- **Input validation**: `customTTL` is optional uint32, no validation needed (reasonable range 1-86400s expected in practice)
- **No authentication/authorization changes**

## Performance
- **Minimal overhead**: One additional `value_or()` call per `retrievePaths()` invocation
- **No algorithmic changes**: Expiry logic complexity unchanged (O(1) cache lookup)
- **Memory impact**: None (no additional data structures)
- **Acceptable performance**: Overhead < 1μs, negligible compared to cache lookup

## Scalability
- **No scalability impact**: Method called per-path-check, frequency unchanged
- **Cache behavior unchanged**: Expiry and removal logic identical, only TTL value differs

## Reliability
- **Backward compatibility**: Existing code continues to work without modification
- **Graceful degradation**: Invalid TTL values (e.g., 0) handled by age comparison (always expired)
- **No new failure modes**: Error handling unchanged

## Maintainability
- **Code clarity**: `customTTL.value_or()` is idiomatic C++17 optional usage
- **Documentation**: Method signature self-documenting with default parameter
- **Consistency**: TTL semantics `age >= TTL` consistent with common TTL interpretations
- **Future extensibility**: Easy to add TTL to other cache methods if needed

## Cost
- **Development cost**: ~2-3 hours implementation + 1 hour demo
- **Testing cost**: Covered in Task 07-07
- **Maintenance cost**: Minimal, single method change

## Compliance
- **Policy compliance**: Follows task-driven development (PRD 07)
- **Coding standards**: Adheres to existing C++ style in `ExchangePathsManager`
- **No regulatory impact**: Internal cache management change

# Restrictions
- Commit changes only after successfully executing the demo
- Do not implement tests in this task (tests are in Task 07-07)
- Do not modify other `ExchangePathsManager` methods
- Maintain backward compatibility (no breaking changes to existing API)
