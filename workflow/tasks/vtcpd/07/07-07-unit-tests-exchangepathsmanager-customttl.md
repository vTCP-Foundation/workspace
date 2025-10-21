# 07-07 - Unit Tests for ExchangePathsManager customTTL Extension

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- [Previous task: Task 07-01](07-01-extend-exchangepathsmanager-custom-ttl.md)

# Description
Create comprehensive unit tests for the `ExchangePathsManager::retrievePaths()` customTTL extension implemented in Task 07-01. This test suite validates that the optional `customTTL` parameter works correctly, maintains backward compatibility, and properly implements TTL expiry semantics (`age >= TTL`).

Tests follow the pattern established in existing `ExchangePathsManagerTest.cpp` (if exists) or similar manager test files, using real objects and TestEnvironment helpers.

# Requirements and DOD

## Functional Requirements
1. **Test file creation**
   - Create or extend `tests/unit/paths/ExchangePathsManagerTest.cpp`
   - Use Google Test framework (matches existing test infrastructure)
   - Include all necessary headers for ExchangePathsManager testing

2. **Test coverage from PRD Section 8**
   - Test 1: Default behavior preserved (no customTTL)
   - Test 2: Custom TTL used when provided
   - Test 3: TTL boundary semantics (age >= TTL)
   - Test 4: Just under TTL is valid (age < TTL)
   - Test 5: Missing paths return nullopt
   - Test 6: Thread safety with customTTL

3. **Test implementation requirements**
   - Use real `ExchangePathsManager` instances (no mocking)
   - Use TestEnvironment helpers for initialization
   - Test both positive and negative cases
   - Verify logging output where applicable
   - Test edge cases (0s TTL, very large TTL)

## Definition of Done
- [ ] Test file created at `tests/unit/paths/ExchangePathsManagerTest.cpp`
- [ ] All 6 test scenarios from PRD Section 8 implemented
- [ ] Tests use real ExchangePathsManager objects
- [ ] Tests compile in `build-tests`
- [ ] All tests pass when executed
- [ ] Test coverage includes edge cases
- [ ] Code follows existing test conventions
- [ ] Tests are well-documented with clear names

# Implementation Plan

## Step 1: Create Test File Structure
**File**: `tests/unit/paths/ExchangePathsManagerTest.cpp`

Create the basic test file structure:

```cpp
#include <gtest/gtest.h>
#include "../../../src/core/paths/ExchangePathsManager.h"
#include "../../../src/core/common/time/TimeUtils.h"
#include <thread>
#include <chrono>

using namespace std;

// Test fixture for ExchangePathsManager tests
class ExchangePathsManagerTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Create fresh ExchangePathsManager instance for each test
        manager = make_unique<ExchangePathsManager>(logger);
    }

    void TearDown() override {
        manager.reset();
    }

    // Helper method to create test paths
    vector<OptimalPathResult> createTestPaths(size_t count) {
        vector<OptimalPathResult> paths;
        for (size_t i = 0; i < count; ++i) {
            OptimalPathResult path;
            path.optimal_flow = TrustLineAmount(100 + i * 10);
            path.received_amount = TrustLineAmount(90 + i * 9);
            // Add other required fields...
            paths.push_back(path);
        }
        return paths;
    }

    // Helper method to cache paths for testing
    void cachePaths(const PathCacheKey &key, const vector<OptimalPathResult> &paths) {
        // Use ExchangePathsManager's caching method
        // This may require accessing protected members or using public interface
        manager->cachePaths(key, paths); // Adjust based on actual API
    }

    unique_ptr<ExchangePathsManager> manager;
    Logger logger; // Initialize with appropriate log level
};
```

**Note**: Adjust the `cachePaths()` helper based on the actual `ExchangePathsManager` API. If `cachePaths()` is not public, you may need to:
- Use the `calculateMaxFlow()` method which caches results
- Add a friend declaration in `ExchangePathsManager` for testing
- Or directly test via the public API flow

## Step 2: Implement Test 1 - Default Behavior Preserved
**Test Name**: `RetrievePathsWithoutCustomTTL_UsesDefaultTTL`

```cpp
TEST_F(ExchangePathsManagerTest, RetrievePathsWithoutCustomTTL_UsesDefaultTTL) {
    // Arrange
    PathCacheKey key{123, 1, 2}; // contractorID=123, senderEquiv=1, receiverEquiv=2
    auto testPaths = createTestPaths(3);
    cachePaths(key, testPaths);

    // Act - call without customTTL parameter
    auto result = manager->retrievePaths(key);

    // Assert - paths should be returned (default 600s TTL, just cached)
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->size(), 3);

    // Wait 599 seconds would still be valid (can't actually wait, so this is conceptual)
    // Immediate retrieval should always work with default TTL
}
```

## Step 3: Implement Test 2 - Custom TTL Used When Provided
**Test Name**: `RetrievePathsWithCustomTTL_UsesProvidedTTL`

```cpp
TEST_F(ExchangePathsManagerTest, RetrievePathsWithCustomTTL_UsesProvidedTTL) {
    // Arrange
    PathCacheKey key{123, 1, 2};
    auto testPaths = createTestPaths(3);
    cachePaths(key, testPaths);

    // Act - call with custom TTL of 150 seconds
    auto result = manager->retrievePaths(key, 150);

    // Assert - paths should be returned (age is 0, well under 150s)
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->size(), 3);
}
```

## Step 4: Implement Test 3 - TTL Boundary Semantics (age >= TTL)
**Test Name**: `RetrievePathsAtExactTTLBoundary_ReturnsNullopt`

```cpp
TEST_F(ExchangePathsManagerTest, RetrievePathsAtExactTTLBoundary_ReturnsNullopt) {
    // Arrange
    PathCacheKey key{123, 1, 2};
    auto testPaths = createTestPaths(3);

    // Cache paths and immediately modify cached timestamp to simulate age
    cachePaths(key, testPaths);

    // Sleep for exactly 150 seconds to test boundary (or mock time)
    // For practical testing, you may need to mock the time or modify cache entry timestamp
    std::this_thread::sleep_for(std::chrono::seconds(150));

    // Act
    auto result = manager->retrievePaths(key, 150);

    // Assert - paths should be expired (age == 150, which is >= 150)
    EXPECT_FALSE(result.has_value());
}
```

**Note**: This test may require time mocking or modification of cached entry timestamps. If sleeping 150 seconds is impractical for unit tests, consider:
- Mocking `utc_now()` function
- Adding a test-only method to modify cached entry timestamps
- Using dependency injection for time provider

## Step 5: Implement Test 4 - Just Under TTL Is Valid
**Test Name**: `RetrievePathsJustUnderTTL_ReturnsPaths`

```cpp
TEST_F(ExchangePathsManagerTest, RetrievePathsJustUnderTTL_ReturnsPaths) {
    // Arrange
    PathCacheKey key{123, 1, 2};
    auto testPaths = createTestPaths(3);
    cachePaths(key, testPaths);

    // Sleep for 149 seconds (just under 150s TTL)
    std::this_thread::sleep_for(std::chrono::seconds(149));

    // Act
    auto result = manager->retrievePaths(key, 150);

    // Assert - paths should still be valid (age = 149 < 150)
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->size(), 3);
}
```

**Note**: Same time-related considerations as Test 3.

## Step 6: Implement Test 5 - Missing Paths Return Nullopt
**Test Name**: `RetrievePathsForMissingKey_ReturnsNullopt`

```cpp
TEST_F(ExchangePathsManagerTest, RetrievePathsForMissingKey_ReturnsNullopt) {
    // Arrange
    PathCacheKey key{123, 1, 2};
    // Don't cache any paths

    // Act - retrieve with custom TTL
    auto result = manager->retrievePaths(key, 150);

    // Assert - should return nullopt (key not found)
    EXPECT_FALSE(result.has_value());

    // Also test with default TTL
    auto result2 = manager->retrievePaths(key);
    EXPECT_FALSE(result2.has_value());
}
```

## Step 7: Implement Test 6 - Thread Safety With CustomTTL
**Test Name**: `RetrievePathsConcurrently_ThreadSafe`

```cpp
TEST_F(ExchangePathsManagerTest, RetrievePathsConcurrently_ThreadSafe) {
    // Arrange
    PathCacheKey key{123, 1, 2};
    auto testPaths = createTestPaths(5);
    cachePaths(key, testPaths);

    const int numThreads = 10;
    const int retrievalsPerThread = 100;
    std::atomic<int> successCount{0};

    // Act - concurrent retrievals with different TTL values
    vector<std::thread> threads;
    for (int i = 0; i < numThreads; ++i) {
        threads.emplace_back([&, i]() {
            uint32_t customTTL = 150 + (i % 3) * 50; // Use 150, 200, or 250 seconds

            for (int j = 0; j < retrievalsPerThread; ++j) {
                auto result = manager->retrievePaths(key, customTTL);
                if (result.has_value()) {
                    successCount++;
                }
            }
        });
    }

    // Wait for all threads
    for (auto &thread : threads) {
        thread.join();
    }

    // Assert - all retrievals should succeed (paths are fresh)
    EXPECT_EQ(successCount, numThreads * retrievalsPerThread);
}
```

## Step 8: Add Edge Case Tests
**Test Name**: `RetrievePathsWithZeroTTL_AlwaysExpired`

```cpp
TEST_F(ExchangePathsManagerTest, RetrievePathsWithZeroTTL_AlwaysExpired) {
    // Arrange
    PathCacheKey key{123, 1, 2};
    auto testPaths = createTestPaths(3);
    cachePaths(key, testPaths);

    // Small delay to ensure age > 0
    std::this_thread::sleep_for(std::chrono::milliseconds(10));

    // Act - retrieve with TTL=0 (immediate expiry)
    auto result = manager->retrievePaths(key, 0);

    // Assert - should be expired (any age >= 0)
    EXPECT_FALSE(result.has_value());
}
```

**Test Name**: `RetrievePathsWithVeryLargeTTL_AlwaysValid`

```cpp
TEST_F(ExchangePathsManagerTest, RetrievePathsWithVeryLargeTTL_AlwaysValid) {
    // Arrange
    PathCacheKey key{123, 1, 2};
    auto testPaths = createTestPaths(3);
    cachePaths(key, testPaths);

    // Wait a short time
    std::this_thread::sleep_for(std::chrono::seconds(5));

    // Act - retrieve with very large TTL (1 year)
    auto result = manager->retrievePaths(key, 365 * 24 * 60 * 60);

    // Assert - should still be valid
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result->size(), 3);
}
```

## Step 9: Build and Run Tests
**Build tests**:
```bash
cd build-tests
cmake ..
make ExchangePathsManagerTest -j4
```

**Run tests**:
```bash
./tests/unit/paths/ExchangePathsManagerTest
# Or use ctest
ctest -R ExchangePathsManagerTest -V
```

## Step 10: Verify Test Coverage
Ensure tests cover:
- [ ] Default TTL behavior (backward compatibility)
- [ ] Custom TTL parameter usage
- [ ] Expiry boundary conditions (age == TTL)
- [ ] Valid path scenarios (age < TTL)
- [ ] Missing key scenarios
- [ ] Thread safety with concurrent access
- [ ] Edge cases (0 TTL, very large TTL)

## Expected Files Created/Modified
- `tests/unit/paths/ExchangePathsManagerTest.cpp` (new or extended)
- `tests/unit/paths/CMakeLists.txt` (if needed for test registration)

# Test Plan

## Test Execution
All tests will be built and executed in the `build-tests` directory:

1. **Build tests**:
   ```bash
   cd build-tests
   cmake ..
   make -j4
   ```

2. **Run specific test suite**:
   ```bash
   ./tests/unit/paths/ExchangePathsManagerTest
   ```

3. **Run with verbose output**:
   ```bash
   ctest -R ExchangePathsManagerTest -V
   ```

## Success Criteria
- All 6 core tests from PRD Section 8 pass
- All edge case tests pass
- Thread safety test completes without race conditions or deadlocks
- Tests run in < 5 minutes (or less if time-based tests are mocked)
- No memory leaks detected (if running with valgrind)

## Time-Based Testing Strategy
For tests involving time delays (Tests 3-4):

**Option 1: Real delays (simple but slow)**
- Use `std::this_thread::sleep_for()` for actual delays
- Adjust test timeouts accordingly
- May make test suite slow (5+ minutes)

**Option 2: Time mocking (recommended)**
- Mock `utc_now()` function via dependency injection
- Control time progression in tests
- Fast test execution (< 1 minute)

**Option 3: Direct cache manipulation (pragmatic)**
- Add test-only method to modify cached entry timestamps
- Simulate time passage without actual delays
- Requires minimal changes to ExchangePathsManager

Choose Option 2 or 3 based on existing test infrastructure and ExchangePathsManager design.

# Verification and Validation

## Architecture integrity
- **Test pattern compliance**: Follows existing manager test patterns
- **Real objects**: Uses actual ExchangePathsManager (no heavy mocking)
- **TestEnvironment usage**: Consistent with other unit tests

## Security
- **No security testing needed**: TTL is a timing parameter, no security implications
- **Thread safety validated**: Concurrent access tested for race conditions

## Performance
- **Test execution time**: Target < 5 minutes for full suite
- **Thread safety test**: 1000 total retrievals across 10 threads
- **No performance regression**: Tests don't impact production code performance

## Scalability
- **Thread safety**: Tests validate concurrent access patterns
- **Multiple TTL values**: Tests cover range of TTL values (0 to 1 year)

## Reliability
- **Edge case coverage**: Zero TTL, very large TTL, missing keys
- **Boundary testing**: Exact TTL boundary (age == TTL)
- **Error conditions**: Missing paths, expired paths

## Maintainability
- **Test names**: Clear, descriptive test names indicate purpose
- **Test structure**: Arrange-Act-Assert pattern throughout
- **Documentation**: Comments explain test scenarios
- **Future extensibility**: Easy to add more TTL-related tests

## Cost
- **Development cost**: ~4-6 hours for 8 tests
- **Execution cost**: < 5 minutes per run
- **Maintenance cost**: Low, stable test suite

## Compliance
- **Policy compliance**: Tests created per PRD 07 testing requirements
- **Google Test framework**: Uses standard testing framework
- **Build system integration**: Tests integrated into build-tests

# Restrictions
- Commit tests only after all tests pass in build-tests
- Use real ExchangePathsManager objects (avoid mocking)
- Follow Google Test conventions and naming
- Ensure tests are deterministic (no flaky tests)
- Tests must run in build-tests environment
- Do not modify production code to make tests pass (except test-friendly time handling if needed)
