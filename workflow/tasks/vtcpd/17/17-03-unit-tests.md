# 17-03 - Unit Tests for BlockNumberCache

# Links
- [PRD-17: Block Number Cache](../../prd/vtcpd/17-block-number-cache.md)
- [Task 17-01: BlockNumberCache Class](17-01-block-number-cache-class.md)

# Description

Implement comprehensive unit tests for the `BlockNumberCache` class created in Task 17-01. These tests verify cache state management, TTL/refresh logic, and block number prediction functionality.

The tests follow the existing project testing patterns (see `tests/unit/rates/ExchangeRatesManagerTest.cpp` for reference) and use an injectable clock pattern via `utc_now()` for time-dependent tests.

# Requirements and DOD

## Functional Requirements

1. **FR-01**: Create test file `tests/unit/network/rpc/BlockNumberCacheTest.cpp`
2. **FR-02**: Implement the following 8 unit tests:

### Cache State Tests
| ID | Test Name | Description |
|----|-----------|-------------|
| T1 | `testInitialStateHasNoCache` | After construction, `needsRefresh()` returns `true` |
| T2 | `testUpdateStoresBlockNumberAndTimestamp` | After `update()`, `needsRefresh()` returns `false` and value is stored |
| T3 | `testClearResetsCache` | After `clear()`, cache returns to initial state (`needsRefresh() = true`) |

### TTL and Refresh Logic Tests
| ID | Test Name | Description |
|----|-----------|-------------|
| T4 | `testNeedsRefreshReturnsTrueWhenCacheEmpty` | Empty cache always needs refresh |
| T5 | `testNeedsRefreshReturnsFalseWithinTTL` | Cache valid when current time < cache_time + TTL |

> **Note**: Test for TTL expiry (`testNeedsRefreshReturnsTrueAfterTTLExpired`) was removed - it requires 600 seconds wait time without injectable clock, which is not suitable for unit tests.

### Block Number Prediction Tests
| ID | Test Name | Description |
|----|-----------|-------------|
| T6 | `testGetCachedBlockNumberReturnsExactValueWhenJustUpdated` | Returns exact cached value immediately after update (0 elapsed time) |
| T7 | `testGetCachedBlockNumberNeverDecreasesOnPrediction` | Predicted value is always >= initial cached value |

> **Note**: Test for prediction accuracy (`testGetCachedBlockNumberPredictsCorrectly`) was removed - it requires 60+ seconds wait time (kBlockGenerationSeconds = 60), which is not suitable for unit tests.

### Constants Tests
| ID | Test Name | Description |
|----|-----------|-------------|
| T8 | `testConstantsHaveExpectedValues` | `kBlockGenerationSeconds == 60`, `kCacheTTLSeconds == 600` |

3. **FR-03**: Add test file to `tests/unit/CMakeLists.txt`
4. **FR-04**: All tests must pass

## Non-Functional Requirements

1. **NFR-01**: Follow existing test patterns from `ExchangeRatesManagerTest.cpp`
2. **NFR-02**: Use Google Test framework (gtest)
3. **NFR-03**: Use time manipulation for TTL tests (sleep or mock time)
4. **NFR-04**: Each test must be independent and repeatable

## Definition of Done

- [ ] Test file created at correct location
- [ ] All 8 tests implemented
- [ ] Tests added to CMakeLists.txt
- [ ] All tests pass (`make test` or `ctest`)
- [ ] Tests follow project conventions

# Implementation Plan

## Step 1: Create Test File

Create `tests/unit/network/rpc/BlockNumberCacheTest.cpp`:

```cpp
#include <gtest/gtest.h>
#include <thread>
#include <chrono>

#include "core/network/rpc/BlockNumberCache.h"
#include "core/logger/Logger.h"

class BlockNumberCacheTest : public ::testing::Test {
protected:
    void SetUp() override {
        logger = std::make_unique<Logger>();
        cache = std::make_unique<BlockNumberCache>(*logger);
    }

    void TearDown() override {
        cache.reset();
        logger.reset();
    }

    std::unique_ptr<Logger> logger;
    std::unique_ptr<BlockNumberCache> cache;
};
```

## Step 2: Implement Cache State Tests

```cpp
// T1: testInitialStateHasNoCache
TEST_F(BlockNumberCacheTest, testInitialStateHasNoCache) {
    EXPECT_TRUE(cache->needsRefresh());
}

// T2: testUpdateStoresBlockNumberAndTimestamp
TEST_F(BlockNumberCacheTest, testUpdateStoresBlockNumberAndTimestamp) {
    BlockNumber blockNumber = 12345;
    cache->update(blockNumber);

    EXPECT_FALSE(cache->needsRefresh());
    EXPECT_EQ(cache->getCachedBlockNumber(), blockNumber);
}

// T3: testClearResetsCache
TEST_F(BlockNumberCacheTest, testClearResetsCache) {
    cache->update(12345);
    EXPECT_FALSE(cache->needsRefresh());

    cache->clear();
    EXPECT_TRUE(cache->needsRefresh());
}
```

## Step 3: Implement TTL and Refresh Logic Tests

```cpp
// T4: testNeedsRefreshReturnsTrueWhenCacheEmpty
TEST_F(BlockNumberCacheTest, testNeedsRefreshReturnsTrueWhenCacheEmpty) {
    // Fresh cache without any update
    EXPECT_TRUE(cache->needsRefresh());

    // After clear
    cache->update(100);
    cache->clear();
    EXPECT_TRUE(cache->needsRefresh());
}

// T5: testNeedsRefreshReturnsFalseWithinTTL
TEST_F(BlockNumberCacheTest, testNeedsRefreshReturnsFalseWithinTTL) {
    cache->update(12345);

    // Immediately after update
    EXPECT_FALSE(cache->needsRefresh());

    // After small delay (still within TTL)
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    EXPECT_FALSE(cache->needsRefresh());
}
```

## Step 4: Implement Block Number Prediction Tests

```cpp
// T6: testGetCachedBlockNumberReturnsExactValueWhenJustUpdated
TEST_F(BlockNumberCacheTest, testGetCachedBlockNumberReturnsExactValueWhenJustUpdated) {
    BlockNumber blockNumber = 99999;
    cache->update(blockNumber);

    // Immediately after update, should return exact value
    BlockNumber result = cache->getCachedBlockNumber();
    EXPECT_EQ(result, blockNumber);
}

// T7: testGetCachedBlockNumberNeverDecreasesOnPrediction
TEST_F(BlockNumberCacheTest, testGetCachedBlockNumberNeverDecreasesOnPrediction) {
    BlockNumber initialBlock = 5000;
    cache->update(initialBlock);

    BlockNumber first = cache->getCachedBlockNumber();
    EXPECT_GE(first, initialBlock);

    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    BlockNumber second = cache->getCachedBlockNumber();
    EXPECT_GE(second, first);
}
```

## Step 5: Implement Constants Test

```cpp
// T8: testConstantsHaveExpectedValues
TEST_F(BlockNumberCacheTest, testConstantsHaveExpectedValues) {
    EXPECT_EQ(BlockNumberCache::kBlockGenerationSeconds, 60);
    EXPECT_EQ(BlockNumberCache::kCacheTTLSeconds, 600);
}
```

## Step 6: Update CMakeLists.txt

Add to `tests/unit/CMakeLists.txt` (in appropriate section with other network/rpc tests):

```cmake
network/rpc/BlockNumberCacheTest.cpp
```

## Step 7: Build and Run Tests

```bash
make unit_tests
./bin/unit_tests --gtest_filter="BlockNumberCacheTest.*"
```

# Test Plan

**Complexity**: Simple (Testing-Focused Task)

This is a testing task - the tests ARE the implementation.

**Test Demonstration:**
- Execute all BlockNumberCache tests
- Show all 8 tests passing
- Show test output with timing information

**Success Criteria:**
- All 8 tests pass
- Tests execute in reasonable time (< 1 second total)
- No flaky tests (timing-based tests have appropriate tolerance)

# Verification and Validation

## Architecture integrity
- Tests placed in correct location (`tests/unit/network/rpc/`)
- Follows existing test patterns
- Uses project test infrastructure (gtest)

## Security
- N/A for unit tests

## Performance
- Tests complete in reasonable time
- Sleep durations minimized while maintaining test validity

## Scalability
- N/A for unit tests

## Reliability
- Tests are deterministic and repeatable
- Timing-based tests have appropriate tolerances
- Each test is independent

## Maintainability
- Clear test names describe what is being tested
- Tests are self-documenting
- Easy to add new tests if needed

## Cost
- N/A for unit tests

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Only create test file and update CMakeLists.txt
- Do not modify BlockNumberCache implementation
- Do not modify Core or other production code
