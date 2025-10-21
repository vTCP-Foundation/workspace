# 07-09 - Unit Tests for FindPathsByMaxFlowExchangeTransaction

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- [Previous task: Task 07-03](07-03-implement-findpathsbymaxflowexchangetransaction.md)

# Description
Create comprehensive unit tests for `FindPathsByMaxFlowExchangeTransaction` implemented in Task 07-03. This test suite validates that the transaction correctly collects topology, builds paths using OR-Tools, caches results in `ExchangePathsManager`, and returns `ExchangePathsResource` to notify the coordinator.

Tests follow existing transaction test patterns (e.g., `InitiateMaxFlowExchangeCalculationTransactionTest`), using real components and TestEnvironment helpers.

# Requirements and DOD

## Functional Requirements
1. **Test file creation**
   - Create `tests/unit/transactions/find_path/FindPathsByMaxFlowExchangeTransactionTest.cpp`
   - Use Google Test framework
   - Include necessary headers and test helpers

2. **Test coverage from PRD Section 3**
   - Constructor initializes with all required parameters
   - sendRequestForCollectingTopology() initiates topology collection
   - processCollectingTopology() builds paths using OR-Tools
   - Paths correctly cached in ExchangePathsManager for all PathCacheKey combinations
   - Returns ExchangePathsResource via ResourcesManager
   - Handles OR-Tools failures gracefully (returns resource with empty cache)
   - Reuses InitiateMaxFlowExchangeCalculationTransaction logic correctly

3. **Test implementation requirements**
   - Use real transaction instances
   - Use TestEnvironment for component initialization
   - Mock or simulate topology collection responses
   - Verify path caching via ExchangePathsManager queries
   - Test both success and failure scenarios

## Definition of Done
- [ ] Test file created at `tests/unit/transactions/find_path/FindPathsByMaxFlowExchangeTransactionTest.cpp`
- [ ] All test scenarios from PRD Section 3 implemented
- [ ] Tests use real transaction objects with TestEnvironment
- [ ] Tests compile in `build-tests`
- [ ] All tests pass when executed
- [ ] Code follows existing transaction test conventions
- [ ] Both success and failure paths tested

# Implementation Plan

## Step 1: Create Test File Structure
**File**: `tests/unit/transactions/find_path/FindPathsByMaxFlowExchangeTransactionTest.cpp`

```cpp
#include <gtest/gtest.h>
#include "../../../../src/core/transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.h"
#include "../../../../src/core/paths/ExchangePathsManager.h"
#include "../../../../src/core/resources/manager/ResourcesManager.h"
#include "../../../helpers/TestEnvironment.h"

using namespace std;

// Test fixture for FindPathsByMaxFlowExchangeTransaction tests
class FindPathsByMaxFlowExchangeTransactionTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Initialize test environment with all required components
        testEnv = make_unique<TestEnvironment>();

        // Create test data
        contractorAddress = make_shared<BaseAddress>("127.0.0.1:2000");
        requestedTransactionUUID = TransactionUUID::generateUUID();
        receiverEquivalent = 5;
        exchangeEquivalents = {1, 2, 3};
        hopsCount = 7;

        // Setup test topology (may need helper methods)
        setupTestTopology();
    }

    void TearDown() override {
        testEnv.reset();
    }

    // Helper: Create transaction instance
    shared_ptr<FindPathsByMaxFlowExchangeTransaction> createTransaction() {
        return make_shared<FindPathsByMaxFlowExchangeTransaction>(
            contractorAddress,
            requestedTransactionUUID,
            receiverEquivalent,
            exchangeEquivalents,
            testEnv->contractorsManager(),
            testEnv->resourcesManager(),
            testEnv->equivalentsSubsystemsRouter(),
            testEnv->tailManager(),
            testEnv->exchangePathsManager(),
            testEnv->exchangeRatesManager(),
            testEnv->commissionsManager(),
            testEnv->logger(),
            hopsCount);
    }

    // Helper: Setup test topology for path building
    void setupTestTopology() {
        // Add test nodes, trust lines, exchange rates, etc.
        // Implementation depends on TestEnvironment capabilities
    }

    // Helper: Verify paths cached for specific key
    bool arePathsCached(const PathCacheKey &key) {
        auto paths = testEnv->exchangePathsManager()->retrievePaths(key);
        return paths.has_value() && !paths->empty();
    }

    unique_ptr<TestEnvironment> testEnv;
    BaseAddress::Shared contractorAddress;
    TransactionUUID requestedTransactionUUID;
    SerializedEquivalent receiverEquivalent;
    vector<SerializedEquivalent> exchangeEquivalents;
    HopsCount_t hopsCount;
};
```

## Step 2: Test Constructor Initialization
**Test Name**: `Constructor_InitializesWithAllParameters`

```cpp
TEST_F(FindPathsByMaxFlowExchangeTransactionTest, Constructor_InitializesWithAllParameters) {
    // Act
    auto transaction = createTransaction();

    // Assert
    EXPECT_NE(transaction, nullptr);
    // Verify transaction type (if accessible)
    // Verify internal state initialization (may require getters or friend declaration)
}
```

## Step 3: Test Topology Collection Initiation
**Test Name**: `SendRequestForCollectingTopology_InitiatesCollection`

```cpp
TEST_F(FindPathsByMaxFlowExchangeTransactionTest, SendRequestForCollectingTopology_InitiatesCollection) {
    // Arrange
    auto transaction = createTransaction();

    // Act
    auto result = transaction->sendRequestForCollectingTopology();

    // Assert
    // Verify topology collection requests sent
    // This may require checking message queue or mock verification
    EXPECT_TRUE(result == TransactionResult::kSuccess ||
                result == TransactionResult::kWaitForMessageTypes);
    // Additional assertions based on implementation
}
```

## Step 4: Test Path Building Success Scenario
**Test Name**: `ProcessCollectingTopology_BuildsPathsSuccessfully`

```cpp
TEST_F(FindPathsByMaxFlowExchangeTransactionTest, ProcessCollectingTopology_BuildsPathsSuccessfully) {
    // Arrange
    auto transaction = createTransaction();

    // Setup topology data
    setupTestTopology();

    // Simulate topology collection completion
    // May need to call sendRequestForCollectingTopology() first
    // and simulate message responses

    // Act
    auto result = transaction->processCollectingTopology();

    // Assert - transaction completes successfully
    EXPECT_EQ(result->resultCode(), TransactionResult::kDone);

    // Verify paths cached for all exchange equivalents
    ContractorID contractorID = testEnv->contractorsManager()->contractorIDByAddress(contractorAddress);
    for (const auto &exchangeEquiv : exchangeEquivalents) {
        PathCacheKey key{contractorID, exchangeEquiv, receiverEquivalent};
        EXPECT_TRUE(arePathsCached(key))
            << "Paths not cached for equivalent " << exchangeEquiv;
    }
}
```

## Step 5: Test ExchangePathsResource Return
**Test Name**: `ProcessCollectingTopology_ReturnsExchangePathsResource`

```cpp
TEST_F(FindPathsByMaxFlowExchangeTransactionTest, ProcessCollectingTopology_ReturnsExchangePathsResource) {
    // Arrange
    auto transaction = createTransaction();
    setupTestTopology();

    // Setup resource capture
    shared_ptr<ExchangePathsResource> capturedResource;
    testEnv->resourcesManager()->attachResourceSignal.connect(
        [&](BaseResource::Shared resource) {
            if (resource->resourceType() == BaseResource::ExchangePaths) {
                capturedResource = dynamic_pointer_cast<ExchangePathsResource>(resource);
            }
        });

    // Act
    transaction->processCollectingTopology();

    // Assert - resource returned
    ASSERT_NE(capturedResource, nullptr);
    EXPECT_EQ(capturedResource->transactionUUID(), requestedTransactionUUID);
}
```

## Step 6: Test Path Caching for Multiple Equivalents
**Test Name**: `ProcessCollectingTopology_CachesPathsForAllEquivalents`

```cpp
TEST_F(FindPathsByMaxFlowExchangeTransactionTest, ProcessCollectingTopology_CachesPathsForAllEquivalents) {
    // Arrange
    vector<SerializedEquivalent> multipleEquivs = {1, 2, 3, 4};
    exchangeEquivalents = multipleEquivs;
    auto transaction = createTransaction();
    setupTestTopology();

    // Act
    transaction->processCollectingTopology();

    // Assert - paths cached for each equivalent
    ContractorID contractorID = testEnv->contractorsManager()->contractorIDByAddress(contractorAddress);
    for (const auto &exchangeEquiv : multipleEquivs) {
        PathCacheKey key{contractorID, exchangeEquiv, receiverEquivalent};

        auto cachedPaths = testEnv->exchangePathsManager()->retrievePaths(key);
        ASSERT_TRUE(cachedPaths.has_value())
            << "No paths cached for equivalent " << exchangeEquiv;

        // Verify path structure
        EXPECT_GT(cachedPaths->size(), 0)
            << "Empty paths cached for equivalent " << exchangeEquiv;
    }
}
```

## Step 7: Test OR-Tools Failure Handling
**Test Name**: `ProcessCollectingTopology_HandlesORToolsFailureGracefully`

```cpp
TEST_F(FindPathsByMaxFlowExchangeTransactionTest, ProcessCollectingTopology_HandlesORToolsFailureGracefully) {
    // Arrange
    auto transaction = createTransaction();

    // Setup topology that will cause OR-Tools to fail or find no paths
    // E.g., no trust lines, no connectivity, etc.
    setupFailureTopology(); // Helper method to create invalid topology

    // Setup resource capture
    shared_ptr<ExchangePathsResource> capturedResource;
    testEnv->resourcesManager()->attachResourceSignal.connect(
        [&](BaseResource::Shared resource) {
            if (resource->resourceType() == BaseResource::ExchangePaths) {
                capturedResource = dynamic_pointer_cast<ExchangePathsResource>(resource);
            }
        });

    // Act
    auto result = transaction->processCollectingTopology();

    // Assert - transaction still completes (doesn't crash)
    EXPECT_EQ(result->resultCode(), TransactionResult::kDone);

    // Resource still returned (even with failure)
    ASSERT_NE(capturedResource, nullptr);
    EXPECT_EQ(capturedResource->transactionUUID(), requestedTransactionUUID);

    // Cache may be empty or have no paths (acceptable outcome)
    // Transaction doesn't fail, coordinator handles empty cache
}

// Helper method to setup failure scenario
void setupFailureTopology() {
    // Create topology with no paths
    // E.g., no trust lines to contractor
}
```

## Step 8: Test No Paths Found Scenario
**Test Name**: `ProcessCollectingTopology_NoPathsFound_StillReturnsResource`

```cpp
TEST_F(FindPathsByMaxFlowExchangeTransactionTest, ProcessCollectingTopology_NoPathsFound_StillReturnsResource) {
    // Arrange
    auto transaction = createTransaction();

    // Setup topology with no viable paths
    // E.g., contractor exists but no trust lines with sufficient capacity

    shared_ptr<ExchangePathsResource> capturedResource;
    testEnv->resourcesManager()->attachResourceSignal.connect(
        [&](BaseResource::Shared resource) {
            if (resource->resourceType() == BaseResource::ExchangePaths) {
                capturedResource = dynamic_pointer_cast<ExchangePathsResource>(resource);
            }
        });

    // Act
    transaction->processCollectingTopology();

    // Assert - resource returned even with no paths
    ASSERT_NE(capturedResource, nullptr);

    // Verify cache is empty or has no valid paths (expected)
    ContractorID contractorID = testEnv->contractorsManager()->contractorIDByAddress(contractorAddress);
    for (const auto &exchangeEquiv : exchangeEquivalents) {
        PathCacheKey key{contractorID, exchangeEquiv, receiverEquivalent};
        auto cachedPaths = testEnv->exchangePathsManager()->retrievePaths(key);
        // Either nullopt or empty vector is acceptable
        if (cachedPaths.has_value()) {
            EXPECT_TRUE(cachedPaths->empty())
                << "Expected no paths for equivalent " << exchangeEquiv;
        }
    }
}
```

## Step 9: Test Logic Reuse from InitiateMaxFlowExchangeCalculationTransaction
**Test Name**: `ProcessCollectingTopology_ReusesInitiateLogic`

```cpp
TEST_F(FindPathsByMaxFlowExchangeTransactionTest, ProcessCollectingTopology_ReusesInitiateLogic) {
    // This test verifies that the implementation uses the same algorithm
    // as InitiateMaxFlowExchangeCalculationTransaction

    // Arrange
    auto findTransaction = createTransaction();

    // Create equivalent InitiateMaxFlowExchangeCalculationTransaction for comparison
    auto initiateTransaction = make_shared<InitiateMaxFlowExchangeCalculationTransaction>(
        /* similar parameters */);

    setupTestTopology(); // Same topology for both

    // Act - run both transactions
    findTransaction->processCollectingTopology();
    initiateTransaction->applyCustomLogic(); // Or equivalent method

    // Assert - both should produce same cached paths
    ContractorID contractorID = testEnv->contractorsManager()->contractorIDByAddress(contractorAddress);
    for (const auto &exchangeEquiv : exchangeEquivalents) {
        PathCacheKey key{contractorID, exchangeEquiv, receiverEquivalent};

        auto findPaths = testEnv->exchangePathsManager()->retrievePaths(key);
        // Both should have cached paths (or both should have none)

        // Detailed comparison may require extracting and comparing path details
        // This test validates algorithmic consistency
    }
}
```

## Step 10: Test Correct PathCacheKey Usage
**Test Name**: `ProcessCollectingTopology_UsesCorrectCacheKeys`

```cpp
TEST_F(FindPathsByMaxFlowExchangeTransactionTest, ProcessCollectingTopology_UsesCorrectCacheKeys) {
    // Arrange
    auto transaction = createTransaction();
    setupTestTopology();

    // Act
    transaction->processCollectingTopology();

    // Assert - verify keys are in format {contractorID, exchangeEquiv, receiverEquiv}
    ContractorID contractorID = testEnv->contractorsManager()->contractorIDByAddress(contractorAddress);

    for (const auto &exchangeEquiv : exchangeEquivalents) {
        // Correct key: sender equivalent = exchangeEquiv, receiver = receiverEquivalent
        PathCacheKey correctKey{contractorID, exchangeEquiv, receiverEquivalent};
        EXPECT_TRUE(arePathsCached(correctKey))
            << "Paths not cached with correct key for equivalent " << exchangeEquiv;

        // Verify reverse key is NOT cached (would indicate wrong key usage)
        PathCacheKey reverseKey{contractorID, receiverEquivalent, exchangeEquiv};
        auto reversePaths = testEnv->exchangePathsManager()->retrievePaths(reverseKey);
        // Reverse key should not have paths (unless it happens to match another equivalent)
    }
}
```

## Step 11: Build and Run Tests
**Build tests**:
```bash
cd build-tests
cmake ..
make FindPathsByMaxFlowExchangeTransactionTest -j4
```

**Run tests**:
```bash
./tests/unit/transactions/find_path/FindPathsByMaxFlowExchangeTransactionTest
# Or use ctest
ctest -R FindPathsByMaxFlowExchangeTransactionTest -V
```

## Step 12: Verify Test Coverage
Ensure tests cover:
- [ ] Constructor initialization
- [ ] Topology collection initiation
- [ ] Successful path building
- [ ] Resource return with correct UUID
- [ ] Path caching for multiple equivalents
- [ ] OR-Tools failure handling
- [ ] No paths found scenario
- [ ] Logic reuse verification
- [ ] Correct cache key usage

## Expected Files Created
- `tests/unit/transactions/find_path/FindPathsByMaxFlowExchangeTransactionTest.cpp`
- CMakeLists.txt updates if needed

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
   ./tests/unit/transactions/find_path/FindPathsByMaxFlowExchangeTransactionTest
   ```

3. **Run with verbose output**:
   ```bash
   ctest -R FindPathsByMaxFlowExchangeTransactionTest -V
   ```

## Success Criteria
- All 9+ tests pass
- Tests cover both success and failure scenarios
- Path caching verified via ExchangePathsManager queries
- Resource return verified via ResourcesManager
- Tests run in < 2 minutes
- No memory leaks or crashes

## Test Categories

### Initialization Tests (1 test)
1. Constructor with all parameters

### Topology Collection Tests (2 tests)
1. Topology collection initiation
2. Path building success

### Resource Management Tests (2 tests)
1. ExchangePathsResource return
2. Path caching for multiple equivalents

### Error Handling Tests (2 tests)
1. OR-Tools failure gracefully handled
2. No paths found scenario

### Integration Tests (2 tests)
1. Logic reuse from InitiateMaxFlowExchangeCalculationTransaction
2. Correct cache key usage

# Verification and Validation

## Architecture integrity
- **Transaction pattern compliance**: Follows BaseCollectTopologyForExchangeTransaction
- **Resource management**: Proper use of ResourcesManager
- **Path caching**: Correct interaction with ExchangePathsManager
- **OR-Tools integration**: Validates algorithm usage

## Security
- **No security testing needed**: Transaction operates on internal topology
- **Transaction UUID**: Prevents resource misrouting (validated)

## Performance
- **Test execution time**: < 2 minutes for all tests
- **No performance impact**: Tests don't affect production code
- **Topology setup**: Minimal test topology for fast execution

## Scalability
- **Multiple equivalents**: Tests validate handling of multiple exchange equivalents
- **Path count**: Tests verify multiple paths per equivalent

## Reliability
- **Error scenarios**: OR-Tools failures, no paths found
- **Resource return**: Always returns resource (even on failure)
- **Cache consistency**: Paths cached correctly for all keys

## Maintainability
- **Test names**: Clear, descriptive names
- **Test structure**: Arrange-Act-Assert pattern
- **TestEnvironment**: Reusable setup infrastructure
- **Future extensibility**: Easy to add more transaction tests

## Cost
- **Development cost**: ~5-6 hours for 9 tests + TestEnvironment setup
- **Execution cost**: < 2 minutes per run
- **Maintenance cost**: Moderate (depends on transaction complexity)

## Compliance
- **Policy compliance**: Tests created per PRD 07 requirements
- **Google Test framework**: Standard testing framework
- **Build system integration**: Integrated into build-tests
- **PRD alignment**: Tests cover all scenarios from PRD Section 3

# Restrictions
- Commit tests only after all tests pass in build-tests
- Use real transaction objects with TestEnvironment
- Follow Google Test conventions and naming
- Ensure tests are deterministic (no flaky tests)
- Tests must run in build-tests environment
- Verify OR-Tools integration works correctly
- Do not modify production code to make tests pass (except test-friendly features)
