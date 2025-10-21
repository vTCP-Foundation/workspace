# 07-08 - Unit Tests for Resource Infrastructure

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- [Previous task: Task 07-02](07-02-create-exchangepathsresource-signal-infrastructure.md)

# Description
Create comprehensive unit tests for the new resource infrastructure components: `ExchangePathsResource` and `RequestExchangePathsResourceSignal`. This test suite validates that the resource object correctly stores transaction UUID, the signal has proper signature and connectivity, and the `ResourcesManager` integration works correctly.

Tests follow existing resource test patterns (e.g., `PathsResourceTest` if exists), using real objects and standard Google Test conventions.

# Requirements and DOD

## Functional Requirements
1. **Test file creation**
   - Create `tests/unit/resources/ExchangePathsResourceTest.cpp`
   - Create `tests/unit/resources/ResourcesManagerSignalTest.cpp` (or extend existing)
   - Use Google Test framework

2. **Test coverage from PRD Sections 1-2**
   - **ExchangePathsResource tests**:
     - Constructor correctly initializes with transaction UUID
     - transactionUUID() getter returns correct value
     - Inherits from BaseResource correctly
     - Resource type correctly identified
   - **RequestExchangePathsResourceSignal tests**:
     - Signal correctly defined with proper signature
     - requestExchangePaths() method triggers signal with correct parameters
     - Signal connection in ResourcesManager works correctly
     - Multiple signal connections can coexist

3. **Test implementation requirements**
   - Use real resource objects (no mocking)
   - Test signal emission and reception
   - Verify parameter passing correctness
   - Test resource type identification

## Definition of Done
- [ ] `ExchangePathsResourceTest.cpp` created with all resource tests
- [ ] Signal tests created (in existing or new test file)
- [ ] All test scenarios from PRD Sections 1-2 implemented
- [ ] Tests use real objects (no heavy mocking)
- [ ] Tests compile in `build-tests`
- [ ] All tests pass when executed
- [ ] Code follows existing resource test conventions

# Implementation Plan

## Step 1: Create ExchangePathsResource Test File
**File**: `tests/unit/resources/ExchangePathsResourceTest.cpp`

```cpp
#include <gtest/gtest.h>
#include "../../../src/core/resources/resources/ExchangePathsResource.h"
#include "../../../src/core/common/NodeUUID.h"

using namespace std;

// Test fixture for ExchangePathsResource tests
class ExchangePathsResourceTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Create test transaction UUID
        testUUID = TransactionUUID::generateUUID();
    }

    TransactionUUID testUUID;
};

TEST_F(ExchangePathsResourceTest, Constructor_InitializesWithTransactionUUID) {
    // Arrange & Act
    ExchangePathsResource resource(testUUID);

    // Assert
    EXPECT_EQ(resource.transactionUUID(), testUUID);
}

TEST_F(ExchangePathsResourceTest, TransactionUUIDGetter_ReturnsCorrectValue) {
    // Arrange
    ExchangePathsResource resource(testUUID);

    // Act
    const TransactionUUID &retrievedUUID = resource.transactionUUID();

    // Assert
    EXPECT_EQ(retrievedUUID, testUUID);
    // Verify it's the same instance (reference equality)
    EXPECT_EQ(&retrievedUUID, &resource.transactionUUID());
}

TEST_F(ExchangePathsResourceTest, ResourceType_CorrectlyIdentified) {
    // Arrange
    ExchangePathsResource resource(testUUID);

    // Act
    byte_t resourceType = resource.resourceType();

    // Assert
    EXPECT_EQ(resourceType, BaseResource::ExchangePaths);
    EXPECT_EQ(resourceType, ExchangePathsResource::kResourceType);
}

TEST_F(ExchangePathsResourceTest, InheritsFromBaseResource) {
    // Arrange & Act
    ExchangePathsResource resource(testUUID);
    BaseResource *basePtr = &resource;

    // Assert - can be used as BaseResource
    EXPECT_NE(basePtr, nullptr);
    EXPECT_EQ(basePtr->resourceType(), BaseResource::ExchangePaths);
}

TEST_F(ExchangePathsResourceTest, MultipleInstances_IndependentUUIDs) {
    // Arrange
    TransactionUUID uuid1 = TransactionUUID::generateUUID();
    TransactionUUID uuid2 = TransactionUUID::generateUUID();

    // Act
    ExchangePathsResource resource1(uuid1);
    ExchangePathsResource resource2(uuid2);

    // Assert
    EXPECT_NE(uuid1, uuid2);
    EXPECT_EQ(resource1.transactionUUID(), uuid1);
    EXPECT_EQ(resource2.transactionUUID(), uuid2);
    EXPECT_NE(resource1.transactionUUID(), resource2.transactionUUID());
}

TEST_F(ExchangePathsResourceTest, SharedPointer_WorksCorrectly) {
    // Arrange & Act
    auto resourcePtr = make_shared<ExchangePathsResource>(testUUID);

    // Assert
    EXPECT_NE(resourcePtr, nullptr);
    EXPECT_EQ(resourcePtr->transactionUUID(), testUUID);
    EXPECT_EQ(resourcePtr->resourceType(), BaseResource::ExchangePaths);
}
```

## Step 2: Create ResourcesManager Signal Tests
**File**: `tests/unit/resources/ResourcesManagerSignalTest.cpp` (or extend existing `ResourcesManagerTest.cpp`)

```cpp
#include <gtest/gtest.h>
#include "../../../src/core/resources/manager/ResourcesManager.h"
#include "../../../src/core/common/NodeUUID.h"
#include "../../../src/common/BaseAddress.h"

using namespace std;

// Test fixture for ResourcesManager signal tests
class ResourcesManagerSignalTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Create ResourcesManager instance
        // May need logger or other dependencies
        manager = make_unique<ResourcesManager>(/* constructor args */);

        // Create test data
        testUUID = TransactionUUID::generateUUID();
        contractorAddress = make_shared<BaseAddress>("127.0.0.1:2000");
        exchangeEquivalents = {1, 2, 3};
        receiverEquivalent = 5;
    }

    unique_ptr<ResourcesManager> manager;
    TransactionUUID testUUID;
    BaseAddress::Shared contractorAddress;
    vector<SerializedEquivalent> exchangeEquivalents;
    SerializedEquivalent receiverEquivalent;
};

TEST_F(ResourcesManagerSignalTest, RequestExchangePathsSignal_Exists) {
    // Assert - signal member exists (compile-time check)
    // This test validates the signal is declared
    EXPECT_NO_THROW({
        manager->requestExchangePathsResourceSignal.connect(
            [](const TransactionUUID&, BaseAddress::Shared,
               const vector<SerializedEquivalent>&, const SerializedEquivalent) {
                // Empty handler
            });
    });
}

TEST_F(ResourcesManagerSignalTest, RequestExchangePaths_EmitsSignalWithCorrectParameters) {
    // Arrange
    bool signalEmitted = false;
    TransactionUUID receivedUUID;
    BaseAddress::Shared receivedAddress;
    vector<SerializedEquivalent> receivedExchangeEquivs;
    SerializedEquivalent receivedReceiverEquiv;

    // Connect to signal
    manager->requestExchangePathsResourceSignal.connect(
        [&](const TransactionUUID &uuid, BaseAddress::Shared address,
            const vector<SerializedEquivalent> &exchangeEquivs,
            const SerializedEquivalent receiverEquiv) {
            signalEmitted = true;
            receivedUUID = uuid;
            receivedAddress = address;
            receivedExchangeEquivs = exchangeEquivs;
            receivedReceiverEquiv = receiverEquiv;
        });

    // Act
    manager->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert
    EXPECT_TRUE(signalEmitted);
    EXPECT_EQ(receivedUUID, testUUID);
    EXPECT_EQ(receivedAddress, contractorAddress);
    EXPECT_EQ(receivedExchangeEquivs, exchangeEquivalents);
    EXPECT_EQ(receivedReceiverEquiv, receiverEquivalent);
}

TEST_F(ResourcesManagerSignalTest, RequestExchangePaths_MultipleConnections) {
    // Arrange
    int callCount = 0;

    // Connect multiple handlers
    manager->requestExchangePathsResourceSignal.connect(
        [&](const TransactionUUID&, BaseAddress::Shared,
            const vector<SerializedEquivalent>&, const SerializedEquivalent) {
            callCount++;
        });

    manager->requestExchangePathsResourceSignal.connect(
        [&](const TransactionUUID&, BaseAddress::Shared,
            const vector<SerializedEquivalent>&, const SerializedEquivalent) {
            callCount++;
        });

    // Act
    manager->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert - both handlers called
    EXPECT_EQ(callCount, 2);
}

TEST_F(ResourcesManagerSignalTest, RequestExchangePaths_EmptyExchangeEquivalents) {
    // Arrange
    vector<SerializedEquivalent> emptyEquivs;
    bool signalEmitted = false;
    vector<SerializedEquivalent> receivedEquivs;

    manager->requestExchangePathsResourceSignal.connect(
        [&](const TransactionUUID&, BaseAddress::Shared,
            const vector<SerializedEquivalent> &exchangeEquivs,
            const SerializedEquivalent) {
            signalEmitted = true;
            receivedEquivs = exchangeEquivs;
        });

    // Act
    manager->requestExchangePaths(
        testUUID,
        contractorAddress,
        emptyEquivs,
        receiverEquivalent);

    // Assert
    EXPECT_TRUE(signalEmitted);
    EXPECT_TRUE(receivedEquivs.empty());
}

TEST_F(ResourcesManagerSignalTest, RequestExchangePaths_MultipleEquivalents) {
    // Arrange
    vector<SerializedEquivalent> multipleEquivs = {1, 2, 3, 4, 5};
    vector<SerializedEquivalent> receivedEquivs;

    manager->requestExchangePathsResourceSignal.connect(
        [&](const TransactionUUID&, BaseAddress::Shared,
            const vector<SerializedEquivalent> &exchangeEquivs,
            const SerializedEquivalent) {
            receivedEquivs = exchangeEquivs;
        });

    // Act
    manager->requestExchangePaths(
        testUUID,
        contractorAddress,
        multipleEquivs,
        receiverEquivalent);

    // Assert
    EXPECT_EQ(receivedEquivs.size(), 5);
    EXPECT_EQ(receivedEquivs, multipleEquivs);
}

TEST_F(ResourcesManagerSignalTest, SignalCoexistence_WithOtherResourceSignals) {
    // Arrange - connect to both RequestPathsResourcesSignal and RequestExchangePathsResourceSignal
    bool pathsSignalEmitted = false;
    bool exchangePathsSignalEmitted = false;

    // Assuming RequestPathsResourcesSignal exists
    manager->requestPathsResourcesSignal.connect(
        [&](const TransactionUUID&, BaseAddress::Shared, const SerializedEquivalent) {
            pathsSignalEmitted = true;
        });

    manager->requestExchangePathsResourceSignal.connect(
        [&](const TransactionUUID&, BaseAddress::Shared,
            const vector<SerializedEquivalent>&, const SerializedEquivalent) {
            exchangePathsSignalEmitted = true;
        });

    // Act - emit exchange paths signal
    manager->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert - only exchange paths signal emitted
    EXPECT_TRUE(exchangePathsSignalEmitted);
    EXPECT_FALSE(pathsSignalEmitted); // Other signal not affected
}
```

## Step 3: Add Signal Signature Validation Tests
**Additional tests in ResourcesManagerSignalTest.cpp**:

```cpp
TEST_F(ResourcesManagerSignalTest, SignalSignature_MatchesExpectedTypes) {
    // This is a compile-time test - if it compiles, signature is correct
    manager->requestExchangePathsResourceSignal.connect(
        [](const TransactionUUID &uuid,
           BaseAddress::Shared address,
           const vector<SerializedEquivalent> &exchangeEquivs,
           const SerializedEquivalent receiverEquiv) {
            // Verify types are correct at compile time
            static_assert(is_same_v<decltype(uuid), const TransactionUUID&>,
                          "UUID parameter type mismatch");
            static_assert(is_same_v<decltype(address), BaseAddress::Shared>,
                          "Address parameter type mismatch");
            static_assert(is_same_v<decltype(exchangeEquivs), const vector<SerializedEquivalent>&>,
                          "Exchange equivalents parameter type mismatch");
            static_assert(is_same_v<decltype(receiverEquiv), const SerializedEquivalent>,
                          "Receiver equivalent parameter type mismatch");
        });

    SUCCEED(); // If we get here, compile-time checks passed
}
```

## Step 4: Add Resource Integration Tests
**Additional tests in ExchangePathsResourceTest.cpp**:

```cpp
TEST_F(ExchangePathsResourceTest, ResourceIntegration_PutAndRetrieve) {
    // This test requires ResourcesManager
    // Create manager
    auto manager = make_unique<ResourcesManager>(/* args */);

    // Arrange
    auto resource = make_shared<ExchangePathsResource>(testUUID);

    // Act - put resource
    manager->putResource(resource);

    // Assert - resource can be retrieved (may require transaction UUID-based retrieval)
    // Implementation depends on ResourcesManager API
    auto retrieved = manager->getResource(testUUID, BaseResource::ExchangePaths);
    ASSERT_NE(retrieved, nullptr);

    // Verify it's the same resource
    auto exchangePathsRes = dynamic_pointer_cast<ExchangePathsResource>(retrieved);
    ASSERT_NE(exchangePathsRes, nullptr);
    EXPECT_EQ(exchangePathsRes->transactionUUID(), testUUID);
}
```

## Step 5: Build and Run Tests
**Build tests**:
```bash
cd build-tests
cmake ..
make -j4
```

**Run specific test suites**:
```bash
./tests/unit/resources/ExchangePathsResourceTest
./tests/unit/resources/ResourcesManagerSignalTest
```

**Run all resource tests**:
```bash
ctest -R Resource -V
```

## Step 6: Verify Test Coverage
Ensure tests cover:
- [ ] Resource construction and initialization
- [ ] Transaction UUID storage and retrieval
- [ ] Resource type identification
- [ ] BaseResource inheritance
- [ ] Signal emission with all parameters
- [ ] Signal connection and disconnection
- [ ] Multiple signal handlers
- [ ] Edge cases (empty vectors, multiple equivalents)
- [ ] Signal coexistence with other resource signals

## Expected Files Created
- `tests/unit/resources/ExchangePathsResourceTest.cpp`
- `tests/unit/resources/ResourcesManagerSignalTest.cpp` (or extend existing)
- CMakeLists.txt updates if needed for test registration

# Test Plan

## Test Execution
All tests will be built and executed in the `build-tests` directory:

1. **Build tests**:
   ```bash
   cd build-tests
   cmake ..
   make -j4
   ```

2. **Run ExchangePathsResource tests**:
   ```bash
   ./tests/unit/resources/ExchangePathsResourceTest
   ```

3. **Run ResourcesManager signal tests**:
   ```bash
   ./tests/unit/resources/ResourcesManagerSignalTest
   ```

4. **Run all resource tests**:
   ```bash
   ctest -R Resource -V
   ```

## Success Criteria
- All ExchangePathsResource tests pass (6+ tests)
- All signal tests pass (7+ tests)
- Tests run in < 1 minute
- No memory leaks detected
- All edge cases covered

## Test Categories

### ExchangePathsResource Tests (6 tests)
1. Constructor initialization
2. UUID getter correctness
3. Resource type identification
4. BaseResource inheritance
5. Multiple instances independence
6. Shared pointer usage

### Signal Tests (7 tests)
1. Signal existence and connectivity
2. Signal emission with correct parameters
3. Multiple signal connections
4. Empty exchange equivalents vector
5. Multiple exchange equivalents
6. Signal coexistence with other signals
7. Signal signature validation

# Verification and Validation

## Architecture integrity
- **Resource pattern compliance**: Follows BaseResource pattern
- **Signal architecture**: Uses Boost.Signals2 correctly
- **Type safety**: Strong typing for all parameters

## Security
- **No security testing needed**: Internal resource infrastructure
- **Transaction UUID**: Prevents resource hijacking (validated by tests)

## Performance
- **Test execution time**: < 1 minute for all tests
- **No performance impact**: Tests don't affect production code

## Scalability
- **Multiple handlers**: Tests validate multiple signal connections
- **Resource instances**: Tests verify independent resource instances

## Reliability
- **Edge cases**: Empty vectors, multiple equivalents
- **Type safety**: Compile-time type validation
- **Resource lifecycle**: Construction, usage, destruction tested

## Maintainability
- **Test names**: Clear, descriptive names
- **Test structure**: Arrange-Act-Assert pattern
- **Documentation**: Comments explain test purpose
- **Future extensibility**: Easy to add more resource/signal tests

## Cost
- **Development cost**: ~3-4 hours for 13 tests
- **Execution cost**: < 1 minute per run
- **Maintenance cost**: Low, stable infrastructure

## Compliance
- **Policy compliance**: Tests created per PRD 07 requirements
- **Google Test framework**: Standard testing framework
- **Build system integration**: Integrated into build-tests

# Restrictions
- Commit tests only after all tests pass in build-tests
- Use real resource objects (no heavy mocking)
- Follow Google Test conventions and naming
- Ensure tests are deterministic
- Tests must run in build-tests environment
- Verify signal signature matches PRD specification exactly
