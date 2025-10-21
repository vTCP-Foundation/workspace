# 07-11 - Unit Tests for CoordinatorExchangePaymentTransaction Path Checking

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- [Previous task 1: Task 07-05](07-05-add-path-availability-checking.md)
- [Previous task 2: Task 07-06](07-06-relocate-exchange-amount-calculation.md)

# Description
Create comprehensive unit tests for the modified `CoordinatorExchangePaymentTransaction` covering both path availability checking (Task 07-05) and exchange amount calculation relocation (Task 07-06). This test suite validates automatic path collection triggering, resource handling, calculation timing, and error scenarios.

Tests follow existing payment transaction test patterns, using real components and TestEnvironment helpers to simulate complete payment flows.

# Requirements and DOD

## Functional Requirements
1. **Test file creation**
   - Extend `tests/unit/transactions/regular/payments/CoordinatorExchangePaymentTransactionTest.cpp`
   - Use Google Test framework
   - Include necessary test helpers and mocks

2. **Test coverage from PRD Sections 4-5, 7**
   - **Path availability checking (7 tests from Section 4)**:
     - Test 1: All paths available and fresh
     - Test 2: Some paths missing
     - Test 3: Some paths expired
     - Test 4: Mixed missing and expired
     - Test 5: All paths missing
     - Test 6: Resource arrival triggers path processing
     - Test 7: Custom TTL parameter used
   - **Exchange amount calculation (4 tests from Section 5)**:
     - Test 1: Calculation in path processing stage
     - Test 2: Calculation not in initialization stage
     - Test 3: Outgoing capacity validation in path processing
     - Test 4: Calculation with missing paths
   - **Additional integration tests**

3. **Test implementation requirements**
   - Use real CoordinatorExchangePaymentTransaction instances
   - Use TestEnvironment for component setup
   - Mock or control time for TTL testing
   - Verify resource requests and arrivals
   - Test calculation correctness and error handling

## Definition of Done
- [ ] Test file extended at `tests/unit/transactions/regular/payments/CoordinatorExchangePaymentTransactionTest.cpp`
- [ ] All 11+ test scenarios implemented
- [ ] Tests use real transaction objects with TestEnvironment
- [ ] Tests compile in `build-tests`
- [ ] All tests pass when executed
- [ ] Code follows existing transaction test conventions
- [ ] Both success and error paths tested

# Implementation Plan

## Step 1: Extend Test File Structure
**File**: `tests/unit/transactions/regular/payments/CoordinatorExchangePaymentTransactionTest.cpp`

```cpp
#include <gtest/gtest.h>
#include "../../../../../src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h"
#include "../../../../../src/core/paths/ExchangePathsManager.h"
#include "../../../../../src/core/resources/manager/ResourcesManager.h"
#include "../../../../../src/core/resources/resources/ExchangePathsResource.h"
#include "../../../../helpers/TestEnvironment.h"
#include <thread>
#include <chrono>

using namespace std;

// Test fixture for path availability checking tests
class CoordinatorExchangePaymentTransactionPathCheckingTest : public ::testing::Test {
protected:
    void SetUp() override {
        testEnv = make_unique<TestEnvironment>();

        // Create test data
        contractorAddress = make_shared<BaseAddress>("127.0.0.1:2000");
        contractorID = testEnv->contractorsManager()->addContractor(contractorAddress);
        amount = TrustLineAmount(1000);
        receiverEquivalent = 5;
        exchangeEquivalents = {1, 2, 3};

        // Setup test exchange rates and commissions
        setupExchangeRates();
        setupCommissions();
    }

    void TearDown() override {
        testEnv.reset();
    }

    // Helper: Create transaction instance
    shared_ptr<CoordinatorExchangePaymentTransaction> createTransaction() {
        // Create command or use transaction constructor directly
        // Implementation depends on transaction construction API
        return make_shared<CoordinatorExchangePaymentTransaction>(
            contractorAddress,
            amount,
            receiverEquivalent,
            exchangeEquivalents,
            /* other required parameters */);
    }

    // Helper: Cache paths for specific equivalent
    void cachePaths(SerializedEquivalent senderEquiv, SerializedEquivalent receiverEquiv,
                    size_t pathCount, bool makeExpired = false) {
        PathCacheKey key{contractorID, senderEquiv, receiverEquiv};

        // Create test paths
        vector<OptimalPathResult> paths;
        for (size_t i = 0; i < pathCount; ++i) {
            OptimalPathResult path;
            path.optimal_flow = TrustLineAmount(100 + i * 50);
            path.received_amount = TrustLineAmount(90 + i * 45);
            // Add other required fields...
            paths.push_back(path);
        }

        // Cache paths
        testEnv->exchangePathsManager()->cachePaths(key, paths);

        // If expired requested, manipulate timestamp
        if (makeExpired) {
            // This may require test-only API to modify cached entry timestamp
            testEnv->exchangePathsManager()->setPathAge(key, 160); // Older than 150s TTL
        }
    }

    // Helper: Setup exchange rates for testing
    void setupExchangeRates() {
        // Add exchange rates between all relevant equivalents
        for (auto senderEquiv : exchangeEquivalents) {
            testEnv->exchangeRatesManager()->setRate(senderEquiv, receiverEquivalent, 0.9);
        }
    }

    // Helper: Setup commissions
    void setupCommissions() {
        // Add commission structures if needed
    }

    unique_ptr<TestEnvironment> testEnv;
    BaseAddress::Shared contractorAddress;
    ContractorID contractorID;
    TrustLineAmount amount;
    SerializedEquivalent receiverEquivalent;
    vector<SerializedEquivalent> exchangeEquivalents;
};
```

## Step 2: Test 1 - All Paths Available and Fresh
**Test Name**: `InitializationStage_AllPathsAvailableAndFresh_ProceedsDirectly`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       InitializationStage_AllPathsAvailableAndFresh_ProceedsDirectly) {
    // Arrange - cache fresh paths for all exchange equivalents
    for (auto exchangeEquiv : exchangeEquivalents) {
        cachePaths(exchangeEquiv, receiverEquivalent, 3, false); // Fresh paths
    }

    auto transaction = createTransaction();

    // Act
    auto result = transaction->runPaymentInitializationStage();

    // Assert - should proceed directly to path processing (no resource wait)
    EXPECT_NE(result->resultCode(), TransactionResult::kWaitForResource);
    // Verify no resource request was made
    EXPECT_FALSE(testEnv->resourcesManager()->wasRequestMade(BaseResource::ExchangePaths));
}
```

## Step 3: Test 2 - Some Paths Missing
**Test Name**: `InitializationStage_SomePathsMissing_RequestsCollection`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       InitializationStage_SomePathsMissing_RequestsCollection) {
    // Arrange - cache paths for only 1 of 3 equivalents
    cachePaths(exchangeEquivalents[0], receiverEquivalent, 3, false);
    // equivalents[1] and equivalents[2] have no cached paths

    auto transaction = createTransaction();

    vector<SerializedEquivalent> requestedEquivs;
    testEnv->resourcesManager()->setRequestInterceptor(
        [&](const TransactionUUID&, BaseAddress::Shared,
            const vector<SerializedEquivalent> &missingEquivs,
            const SerializedEquivalent) {
            requestedEquivs = missingEquivs;
        });

    // Act
    auto result = transaction->runPaymentInitializationStage();

    // Assert - should wait for resource
    EXPECT_EQ(result->resultCode(), TransactionResult::kWaitForResource);

    // Verify missing equivalents were requested
    EXPECT_EQ(requestedEquivs.size(), 2);
    EXPECT_TRUE(find(requestedEquivs.begin(), requestedEquivs.end(), exchangeEquivalents[1])
                != requestedEquivs.end());
    EXPECT_TRUE(find(requestedEquivs.begin(), requestedEquivs.end(), exchangeEquivalents[2])
                != requestedEquivs.end());
}
```

## Step 4: Test 3 - Some Paths Expired
**Test Name**: `InitializationStage_SomePathsExpired_RequestsCollection`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       InitializationStage_SomePathsExpired_RequestsCollection) {
    // Arrange - cache paths with different ages
    cachePaths(exchangeEquivalents[0], receiverEquivalent, 3, false);  // Fresh (age < 150s)
    cachePaths(exchangeEquivalents[1], receiverEquivalent, 3, true);   // Expired (age >= 150s)
    cachePaths(exchangeEquivalents[2], receiverEquivalent, 3, true);   // Expired (age >= 150s)

    auto transaction = createTransaction();

    vector<SerializedEquivalent> requestedEquivs;
    testEnv->resourcesManager()->setRequestInterceptor(
        [&](const TransactionUUID&, BaseAddress::Shared,
            const vector<SerializedEquivalent> &missingEquivs,
            const SerializedEquivalent) {
            requestedEquivs = missingEquivs;
        });

    // Act
    auto result = transaction->runPaymentInitializationStage();

    // Assert - should wait for resource
    EXPECT_EQ(result->resultCode(), TransactionResult::kWaitForResource);

    // Verify expired equivalents were requested (not the fresh one)
    EXPECT_EQ(requestedEquivs.size(), 2);
    EXPECT_TRUE(find(requestedEquivs.begin(), requestedEquivs.end(), exchangeEquivalents[1])
                != requestedEquivs.end());
    EXPECT_TRUE(find(requestedEquivs.begin(), requestedEquivs.end(), exchangeEquivalents[2])
                != requestedEquivs.end());
    // Fresh equivalent should NOT be requested
    EXPECT_FALSE(find(requestedEquivs.begin(), requestedEquivs.end(), exchangeEquivalents[0])
                 != requestedEquivs.end());
}
```

## Step 5: Test 4 - Mixed Missing and Expired
**Test Name**: `InitializationStage_MixedMissingAndExpired_RequestsBoth`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       InitializationStage_MixedMissingAndExpired_RequestsBoth) {
    // Arrange
    cachePaths(exchangeEquivalents[0], receiverEquivalent, 3, false);  // Fresh
    cachePaths(exchangeEquivalents[1], receiverEquivalent, 3, true);   // Expired
    // equivalents[2] has no cached paths (missing)

    auto transaction = createTransaction();

    vector<SerializedEquivalent> requestedEquivs;
    testEnv->resourcesManager()->setRequestInterceptor(
        [&](const TransactionUUID&, BaseAddress::Shared,
            const vector<SerializedEquivalent> &missingEquivs,
            const SerializedEquivalent) {
            requestedEquivs = missingEquivs;
        });

    // Act
    auto result = transaction->runPaymentInitializationStage();

    // Assert - should request both expired and missing
    EXPECT_EQ(result->resultCode(), TransactionResult::kWaitForResource);
    EXPECT_EQ(requestedEquivs.size(), 2);
    EXPECT_TRUE(find(requestedEquivs.begin(), requestedEquivs.end(), exchangeEquivalents[1])
                != requestedEquivs.end());
    EXPECT_TRUE(find(requestedEquivs.begin(), requestedEquivs.end(), exchangeEquivalents[2])
                != requestedEquivs.end());
}
```

## Step 6: Test 5 - All Paths Missing
**Test Name**: `InitializationStage_AllPathsMissing_RequestsAll`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       InitializationStage_AllPathsMissing_RequestsAll) {
    // Arrange - no cached paths for any equivalent
    auto transaction = createTransaction();

    vector<SerializedEquivalent> requestedEquivs;
    testEnv->resourcesManager()->setRequestInterceptor(
        [&](const TransactionUUID&, BaseAddress::Shared,
            const vector<SerializedEquivalent> &missingEquivs,
            const SerializedEquivalent) {
            requestedEquivs = missingEquivs;
        });

    // Act
    auto result = transaction->runPaymentInitializationStage();

    // Assert - should request all equivalents
    EXPECT_EQ(result->resultCode(), TransactionResult::kWaitForResource);
    EXPECT_EQ(requestedEquivs.size(), exchangeEquivalents.size());
    for (auto equiv : exchangeEquivalents) {
        EXPECT_TRUE(find(requestedEquivs.begin(), requestedEquivs.end(), equiv)
                    != requestedEquivs.end())
            << "Equivalent " << equiv << " not requested";
    }
}
```

## Step 7: Test 6 - Resource Arrival Triggers Path Processing
**Test Name**: `ResourceArrival_TransitionsToPathProcessing`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       ResourceArrival_TransitionsToPathProcessing) {
    // Arrange - no cached paths initially
    auto transaction = createTransaction();

    // Run initialization (will wait for resource)
    auto initResult = transaction->runPaymentInitializationStage();
    EXPECT_EQ(initResult->resultCode(), TransactionResult::kWaitForResource);

    // Cache paths (simulating successful collection)
    for (auto exchangeEquiv : exchangeEquivalents) {
        cachePaths(exchangeEquiv, receiverEquivalent, 3, false);
    }

    // Create and deliver ExchangePathsResource
    auto resource = make_shared<ExchangePathsResource>(transaction->currentTransactionUUID());
    testEnv->resourcesManager()->putResource(resource);

    // Act - resource arrival should trigger continuation
    auto resourceResult = transaction->handleResourceArrival(resource);

    // Assert - should transition to path processing stage
    // Verify transaction proceeds to runPathsResourceProcessingStage()
    // This may require checking transaction state or next stage execution
}
```

## Step 8: Test 7 - Custom TTL Parameter Used
**Test Name**: `InitializationStage_CustomTTL_UsesCorrectExpiryThreshold`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       InitializationStage_CustomTTL_UsesCorrectExpiryThreshold) {
    // Arrange - cache paths with age = 160s (expired with 150s TTL, valid with 600s TTL)
    for (auto exchangeEquiv : exchangeEquivalents) {
        cachePaths(exchangeEquiv, receiverEquivalent, 3, false);
        testEnv->exchangePathsManager()->setPathAge(
            PathCacheKey{contractorID, exchangeEquiv, receiverEquivalent}, 160);
    }

    auto transaction = createTransaction();

    // Act
    auto result = transaction->runPaymentInitializationStage();

    // Assert - should request collection (paths expired with 150s TTL)
    EXPECT_EQ(result->resultCode(), TransactionResult::kWaitForResource);

    // Verify that retrievePaths() with 600s TTL would return paths (for comparison)
    auto pathsWithLongerTTL = testEnv->exchangePathsManager()->retrievePaths(
        PathCacheKey{contractorID, exchangeEquivalents[0], receiverEquivalent}, 600);
    EXPECT_TRUE(pathsWithLongerTTL.has_value())
        << "Paths should be valid with 600s TTL";
}
```

## Step 9: Test 8 - Calculation in Path Processing Stage
**Test Name**: `PathProcessingStage_CalculatesExchangeAmount`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       PathProcessingStage_CalculatesExchangeAmount) {
    // Arrange - cache paths for all equivalents
    for (auto exchangeEquiv : exchangeEquivalents) {
        cachePaths(exchangeEquiv, receiverEquivalent, 3, false);
    }

    auto transaction = createTransaction();

    // Skip initialization stage (already tested)
    // Jump directly to path processing stage

    // Act
    auto result = transaction->runPathsResourceProcessingStage();

    // Assert - mExchangeAmount should be calculated
    // Verify calculation result (may require getter for mExchangeAmount)
    auto exchangeAmount = transaction->exchangeAmount();
    EXPECT_GT(exchangeAmount, TrustLineAmount(0))
        << "Exchange amount not calculated";

    // Verify calculation correctness (should be >= receiver amount due to exchange rates)
    EXPECT_GE(exchangeAmount, amount)
        << "Exchange amount should cover receiver amount";
}
```

## Step 10: Test 9 - Calculation NOT in Initialization Stage
**Test Name**: `InitializationStage_DoesNotCalculateExchangeAmount`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       InitializationStage_DoesNotCalculateExchangeAmount) {
    // Arrange
    for (auto exchangeEquiv : exchangeEquivalents) {
        cachePaths(exchangeEquiv, receiverEquivalent, 3, false);
    }

    auto transaction = createTransaction();

    // Act
    auto result = transaction->runPaymentInitializationStage();

    // Assert - mExchangeAmount should NOT be set yet
    // This requires accessing mExchangeAmount (may need getter or it should be zero)
    auto exchangeAmount = transaction->exchangeAmount();
    EXPECT_EQ(exchangeAmount, TrustLineAmount(0))
        << "Exchange amount should not be calculated in initialization stage";
}
```

## Step 11: Test 10 - Outgoing Capacity Validation
**Test Name**: `PathProcessingStage_ValidatesOutgoingCapacity`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       PathProcessingStage_ValidatesOutgoingCapacity) {
    // Arrange - setup insufficient outgoing capacity
    for (auto exchangeEquiv : exchangeEquivalents) {
        cachePaths(exchangeEquiv, receiverEquivalent, 3, false);

        // Set very low outgoing capacity for all equivalents
        auto trustLinesManager = testEnv->equivalentsSubsystemsRouter()->trustLinesManager(exchangeEquiv);
        trustLinesManager->setTotalOutgoingCapacity(TrustLineAmount(10)); // Very low
    }

    auto transaction = createTransaction();

    // Act
    auto result = transaction->runPathsResourceProcessingStage();

    // Assert - should fail with insufficient funds error
    EXPECT_EQ(result->resultCode(), TransactionResult::kInsufficientFunds)
        << "Should fail due to insufficient outgoing capacity";
}
```

## Step 12: Test 11 - Calculation with Missing Paths
**Test Name**: `PathProcessingStage_MissingPaths_ReturnsError`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       PathProcessingStage_MissingPaths_ReturnsError) {
    // Arrange - no cached paths (edge case: paths disappeared between stages)
    auto transaction = createTransaction();

    // Act
    auto result = transaction->runPathsResourceProcessingStage();

    // Assert - should return no paths error
    EXPECT_EQ(result->resultCode(), TransactionResult::kNoPaths)
        << "Should return error when paths missing";
}
```

## Step 13: Add Integration Test
**Test Name**: `FullFlow_AutomaticPathCollection_CompletesPayment`

```cpp
TEST_F(CoordinatorExchangePaymentTransactionPathCheckingTest,
       FullFlow_AutomaticPathCollection_CompletesPayment) {
    // Arrange - no initial paths, but setup topology for successful collection
    auto transaction = createTransaction();

    // Act - run initialization (triggers collection)
    auto initResult = transaction->runPaymentInitializationStage();
    EXPECT_EQ(initResult->resultCode(), TransactionResult::kWaitForResource);

    // Simulate path collection completing
    for (auto exchangeEquiv : exchangeEquivalents) {
        cachePaths(exchangeEquiv, receiverEquivalent, 5, false);
    }

    // Deliver resource
    auto resource = make_shared<ExchangePathsResource>(transaction->currentTransactionUUID());
    transaction->handleResourceArrival(resource);

    // Continue with path processing
    auto processingResult = transaction->runPathsResourceProcessingStage();

    // Assert - payment should proceed successfully
    EXPECT_NE(processingResult->resultCode(), TransactionResult::kNoPaths);
    EXPECT_NE(processingResult->resultCode(), TransactionResult::kInsufficientFunds);
}
```

## Step 14: Build and Run Tests
**Build tests**:
```bash
cd build-tests
cmake ..
make CoordinatorExchangePaymentTransactionTest -j4
```

**Run tests**:
```bash
./tests/unit/transactions/regular/payments/CoordinatorExchangePaymentTransactionTest
# Or specific fixture
./tests/unit/transactions/regular/payments/CoordinatorExchangePaymentTransactionTest \
    --gtest_filter=CoordinatorExchangePaymentTransactionPathCheckingTest.*
```

## Step 15: Verify Test Coverage
Ensure tests cover:
- [ ] All paths available scenario
- [ ] Some paths missing scenario
- [ ] Some paths expired scenario
- [ ] Mixed missing/expired scenario
- [ ] All paths missing scenario
- [ ] Resource arrival handling
- [ ] Custom TTL usage
- [ ] Calculation in correct stage
- [ ] No calculation in initialization
- [ ] Outgoing capacity validation
- [ ] Missing paths error handling
- [ ] Full integration flow

## Expected Files Modified
- `tests/unit/transactions/regular/payments/CoordinatorExchangePaymentTransactionTest.cpp`
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

2. **Run specific test fixture**:
   ```bash
   ./tests/unit/transactions/regular/payments/CoordinatorExchangePaymentTransactionTest \
       --gtest_filter=CoordinatorExchangePaymentTransactionPathCheckingTest.*
   ```

3. **Run with verbose output**:
   ```bash
   ctest -R CoordinatorExchangePaymentTransactionPathCheckingTest -V
   ```

## Success Criteria
- All 12+ tests pass
- Path availability checking validated
- Resource request/arrival mechanism confirmed
- Calculation timing verified
- Error handling tested
- Tests run in < 3 minutes
- No memory leaks or crashes

## Test Categories

### Path Availability Tests (7 tests)
1. All paths available and fresh
2. Some paths missing
3. Some paths expired
4. Mixed missing and expired
5. All paths missing
6. Resource arrival triggers processing
7. Custom TTL parameter usage

### Calculation Relocation Tests (4 tests)
1. Calculation in path processing stage
2. No calculation in initialization stage
3. Outgoing capacity validation
4. Missing paths error handling

### Integration Tests (1+ test)
1. Full flow with automatic collection

## Time-Based Testing Strategy
For tests involving path expiry:

**Recommended approach**: Add test-only API to ExchangePathsManager:
```cpp
#ifdef TESTING_MODE
void setPathAge(const PathCacheKey &key, uint32_t ageSeconds) {
    // Modify cached entry's computedAt timestamp
    auto it = mCachedPaths.find(key);
    if (it != mCachedPaths.end()) {
        it->second.computedAt = utc_now() - boost::posix_time::seconds(ageSeconds);
    }
}
#endif
```

This avoids slow sleep-based tests and provides precise control over path age.

# Verification and Validation

## Architecture integrity
- **Transaction flow**: Validates proper stage transitions
- **Resource management**: Tests request/wait/resume pattern
- **Path caching integration**: Verifies ExchangePathsManager usage
- **Calculation timing**: Ensures correct sequence of operations

## Security
- **No security testing needed**: Internal transaction logic
- **Resource UUID**: Validates correct transaction routing

## Performance
- **Test execution time**: < 3 minutes for all tests
- **No performance impact**: Tests don't affect production code
- **Efficient setup**: Minimal test topology for fast execution

## Scalability
- **Multiple equivalents**: Tests validate handling up to 5 equivalents
- **Path count**: Tests with varying numbers of cached paths

## Reliability
- **Error scenarios**: Missing paths, expired paths, insufficient capacity
- **Edge cases**: All missing, all expired, mixed states
- **Resource handling**: Timeout, arrival, processing

## Maintainability
- **Test names**: Clear, descriptive names indicate purpose
- **Test structure**: Arrange-Act-Assert pattern throughout
- **TestEnvironment**: Reusable transaction setup
- **Future extensibility**: Easy to add more transaction stage tests

## Cost
- **Development cost**: ~6-8 hours for 12 tests + TestEnvironment extensions
- **Execution cost**: < 3 minutes per run
- **Maintenance cost**: Moderate (payment transaction complexity)

## Compliance
- **Policy compliance**: Tests created per PRD 07 requirements
- **Google Test framework**: Standard testing framework
- **Build system integration**: Integrated into build-tests
- **PRD alignment**: Tests cover all scenarios from PRD Sections 4, 5, 7

# Restrictions
- Commit tests only after all tests pass in build-tests
- Use real CoordinatorExchangePaymentTransaction objects with TestEnvironment
- Follow Google Test conventions and naming
- Ensure tests are deterministic (use time mocking, not sleep)
- Tests must run in build-tests environment
- Verify calculation algorithm matches PRD 06 specification
- Do not modify transaction logic to make tests pass (only add test-friendly hooks if needed)
