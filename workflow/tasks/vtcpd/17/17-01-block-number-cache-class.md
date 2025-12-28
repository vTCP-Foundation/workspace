# 17-01 - BlockNumberCache Class Implementation

# Links
- [PRD-17: Block Number Cache](../../prd/vtcpd/17-block-number-cache.md)

# Description

Create a new `BlockNumberCache` class that caches blockchain block numbers and provides predictive extrapolation based on known block generation period. This class will reduce redundant RPC requests to the observer by caching the block number and predicting future values based on elapsed time.

The class is a standalone component with no external dependencies (except Logger), designed to be integrated into Core in a subsequent task.

**Key behavior:**
- Cache stores block number and retrieval timestamp
- Returns cached/predicted value when within TTL (10 minutes)
- Predicts block number by adding blocks based on elapsed time (1 block per 60 seconds)
- Returns `needsRefresh() = true` when cache is empty or TTL expired
- Single-threaded design (no synchronization needed)
- Lazy initialization (cache empty until first `update()` call)

# Requirements and DOD

## Functional Requirements

1. **FR-01**: Create header file `src/core/network/rpc/BlockNumberCache.h` with class declaration
2. **FR-02**: Create implementation file `src/core/network/rpc/BlockNumberCache.cpp`
3. **FR-03**: Define public constants:
   - `static constexpr uint32_t kBlockGenerationSeconds = 60` (1 minute)
   - `static constexpr uint32_t kCacheTTLSeconds = 600` (10 minutes)
4. **FR-04**: Implement constructor `BlockNumberCache(Logger &logger)`
5. **FR-05**: Implement `bool needsRefresh() const`:
   - Returns `true` if cache is empty (no `update()` called yet)
   - Returns `true` if current time exceeds cache timestamp + TTL
   - Returns `false` otherwise
6. **FR-06**: Implement `BlockNumber getCachedBlockNumber() const`:
   - Calculate elapsed seconds since cache timestamp
   - Predict additional blocks: `elapsed_seconds / kBlockGenerationSeconds`
   - Return `cached_block_number + predicted_blocks`
   - Precondition: `!needsRefresh()` (behavior undefined if cache stale)
7. **FR-07**: Implement `void update(BlockNumber blockNumber)`:
   - Store block number
   - Store current timestamp (using `utc_now()`)
   - Mark cache as valid
8. **FR-08**: Implement `void clear()`:
   - Reset cache to initial empty state
   - Primarily for testing purposes
9. **FR-09**: Add files to `src/core/network/rpc/CMakeLists.txt`

## Non-Functional Requirements

1. **NFR-01**: Use `DateTime` type from `src/core/common/time/TimeUtils.h` for timestamps (consistent with `utc_now()`)
2. **NFR-02**: Use `BlockNumber` type from `src/core/common/Types.h`
3. **NFR-03**: Follow existing code style (see `ExchangeRatesManager` for reference)
4. **NFR-04**: Include appropriate logging for debug purposes

## Definition of Done

- [ ] All files created and added to CMakeLists.txt
- [ ] Code compiles without errors or warnings
- [ ] All functional requirements implemented
- [ ] Code follows project style conventions
- [ ] Class is ready for integration (no Core modifications in this task)

# Implementation Plan

## Step 1: Create Header File

Create `src/core/network/rpc/BlockNumberCache.h`:

```cpp
#ifndef VTCPD_BLOCKNUMBERCACHE_H
#define VTCPD_BLOCKNUMBERCACHE_H

#include "../../common/Types.h"
#include "../../common/time/TimeUtils.h"
#include "../../logger/Logger.h"

class BlockNumberCache {
public:
    static constexpr uint32_t kBlockGenerationSeconds = 60;   // 1 minute
    static constexpr uint32_t kCacheTTLSeconds = 600;         // 10 minutes

public:
    explicit BlockNumberCache(Logger &logger);

    bool needsRefresh() const;
    BlockNumber getCachedBlockNumber() const;
    void update(BlockNumber blockNumber);
    void clear();

private:
    string logHeader() const;
    LoggerStream debug() const;
    LoggerStream info() const;

private:
    BlockNumber mCachedBlockNumber;
    DateTime mCacheTimestamp;
    bool mHasValue;
    Logger &mLogger;
};

#endif // VTCPD_BLOCKNUMBERCACHE_H
```

## Step 2: Create Implementation File

Create `src/core/network/rpc/BlockNumberCache.cpp`:

1. Constructor: Initialize `mHasValue = false`, `mCachedBlockNumber = 0`
2. `needsRefresh()`: Check `mHasValue` and compare `utc_now()` with `mCacheTimestamp + kCacheTTLSeconds`
3. `getCachedBlockNumber()`: Calculate elapsed time, predict blocks, return sum
4. `update()`: Store block number, set `mCacheTimestamp = utc_now()`, set `mHasValue = true`
5. `clear()`: Set `mHasValue = false`

## Step 3: Update CMakeLists.txt

Add to `src/core/network/rpc/CMakeLists.txt`:
```cmake
BlockNumberCache.cpp
BlockNumberCache.h
```

## Step 4: Verify Build

Run build to ensure compilation succeeds without errors.

# Test Plan

**Complexity**: Simple

This task creates a standalone class without external dependencies. Testing will be covered by Task 17-03.

**Validation for this task:**
- Code compiles successfully
- Class interface matches specification
- No integration testing required (covered by Task 17-02)

**Demo:**
- Show successful build with new files
- Show class header with correct interface

# Verification and Validation

## Architecture integrity
- Class follows existing patterns (similar to ExchangeRatesManager)
- Placed in appropriate directory (`src/core/network/rpc/`)
- Uses standard project types (BlockNumber, DateTime)

## Security
- N/A - Block number is public blockchain information

## Performance
- O(1) time complexity for all operations
- No dynamic memory allocation in hot path
- Simple arithmetic for prediction

## Scalability
- Single instance per node (appropriate for single-threaded model)

## Reliability
- Clear state management (mHasValue flag)
- No external dependencies that could fail

## Maintainability
- Clean separation of concerns
- Self-documenting method names
- Consistent with project style

## Cost
- Minimal memory footprint (BlockNumber + DateTime + bool)

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify Core or any other files outside the scope of this task
- Do not implement integration with Core (covered by Task 17-02)
