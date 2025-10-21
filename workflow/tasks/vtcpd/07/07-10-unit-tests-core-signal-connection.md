# 07-10 - Unit Tests for Core Signal Connection

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- [Previous task: Task 07-04](07-04-connect-requestexchangepaths-signal-in-core.md)

# Description
Create unit tests for the Core signal connection implemented in Task 07-04, validating that `RequestExchangePathsResourceSignal` is properly connected in `Core::connectResourcesManagerSignals()` and correctly launches `FindPathsByMaxFlowExchangeTransaction` with all required parameters.

Tests verify the complete signal chain: signal emission → slot invocation → transaction creation → transaction scheduling.

# Requirements and DOD

## Functional Requirements
1. **Test file creation**
   - Extend existing `tests/unit/core/CoreTest.cpp` or create new file
   - Use Google Test framework
   - Include necessary mocking for Core initialization

2. **Test coverage from PRD Section 6**
   - RequestExchangePathsResourceSignal connected in connectResourcesManagerSignals()
   - Signal emission launches FindPathsByMaxFlowExchangeTransaction
   - Transaction UUID correctly passed to launched transaction
   - Contractor address, exchange equivalents, receiver equivalent all passed correctly

3. **Test implementation requirements**
   - Use real Core instance or minimal mock
   - Verify transaction launching mechanism
   - Validate all parameters passed correctly
   - Test signal connection lifecycle

## Definition of Done
- [ ] Test file created/extended at `tests/unit/core/CoreTest.cpp`
- [ ] All test scenarios from PRD Section 6 implemented
- [ ] Tests verify signal connection and transaction launching
- [ ] Tests compile in `build-tests`
- [ ] All tests pass when executed
- [ ] Code follows existing Core test conventions

# Implementation Plan

## Step 1: Extend Core Test File
**File**: `tests/unit/core/CoreTest.cpp` (extend existing or create)

```cpp
#include <gtest/gtest.h>
#include "../../../src/core/Core.h"
#include "../../../src/core/transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.h"
#include "../../../src/core/transactions/manager/TransactionsManager.h"
#include "../../helpers/TestEnvironment.h"

using namespace std;

// Test fixture for Core signal connection tests
class CoreSignalConnectionTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Create Core instance or mock
        // This may require significant setup (all components)
        testEnv = make_unique<TestEnvironment>();

        // Create test data
        testUUID = TransactionUUID::generateUUID();
        contractorAddress = make_shared<BaseAddress>("127.0.0.1:2000");
        exchangeEquivalents = {1, 2, 3};
        receiverEquivalent = 5;
    }

    void TearDown() override {
        testEnv.reset();
    }

    unique_ptr<TestEnvironment> testEnv;
    TransactionUUID testUUID;
    BaseAddress::Shared contractorAddress;
    vector<SerializedEquivalent> exchangeEquivalents;
    SerializedEquivalent receiverEquivalent;
};
```

## Step 2: Test Signal Connection Exists
**Test Name**: `ConnectResourcesManagerSignals_ConnectsExchangePathsSignal`

```cpp
TEST_F(CoreSignalConnectionTest, ConnectResourcesManagerSignals_ConnectsExchangePathsSignal) {
    // Arrange
    auto core = testEnv->core(); // Get Core instance from TestEnvironment

    // Act
    core->connectResourcesManagerSignals();

    // Assert - signal should be connected (at least one slot)
    auto &signal = core->resourcesManager()->requestExchangePathsResourceSignal;
    EXPECT_GT(signal.num_slots(), 0)
        << "RequestExchangePathsResourceSignal has no connected slots";
}
```

## Step 3: Test Transaction Launch on Signal Emission
**Test Name**: `SignalEmission_LaunchesFindPathsByMaxFlowExchangeTransaction`

```cpp
TEST_F(CoreSignalConnectionTest, SignalEmission_LaunchesFindPathsByMaxFlowExchangeTransaction) {
    // Arrange
    auto core = testEnv->core();
    core->connectResourcesManagerSignals();

    // Capture scheduled transactions
    shared_ptr<FindPathsByMaxFlowExchangeTransaction> capturedTransaction;

    // Mock or intercept TransactionsManager::scheduleTransaction
    // This may require dependency injection or TransactionsManager extension
    auto originalScheduler = [&](BaseTransaction::Shared transaction) {
        auto findPathsTx = dynamic_pointer_cast<FindPathsByMaxFlowExchangeTransaction>(transaction);
        if (findPathsTx) {
            capturedTransaction = findPathsTx;
        }
    };

    // Setup transaction capture (implementation-specific)
    testEnv->transactionsManager()->setScheduleInterceptor(originalScheduler);

    // Act - emit signal via ResourcesManager
    core->resourcesManager()->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert - FindPathsByMaxFlowExchangeTransaction was created and scheduled
    ASSERT_NE(capturedTransaction, nullptr)
        << "FindPathsByMaxFlowExchangeTransaction was not scheduled";
}
```

**Note**: This test may require extending `TransactionsManager` with a test hook for intercepting scheduled transactions, or using a mock `TransactionsManager`.

## Step 4: Test Transaction UUID Parameter
**Test Name**: `LaunchedTransaction_ReceivesCorrectTransactionUUID`

```cpp
TEST_F(CoreSignalConnectionTest, LaunchedTransaction_ReceivesCorrectTransactionUUID) {
    // Arrange
    auto core = testEnv->core();
    core->connectResourcesManagerSignals();

    TransactionUUID capturedUUID;
    testEnv->transactionsManager()->setScheduleInterceptor(
        [&](BaseTransaction::Shared transaction) {
            auto findPathsTx = dynamic_pointer_cast<FindPathsByMaxFlowExchangeTransaction>(transaction);
            if (findPathsTx) {
                // Access requestedTransactionUUID (may need getter or friend declaration)
                capturedUUID = findPathsTx->requestedTransactionUUID();
            }
        });

    // Act
    core->resourcesManager()->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert
    EXPECT_EQ(capturedUUID, testUUID);
}
```

## Step 5: Test Contractor Address Parameter
**Test Name**: `LaunchedTransaction_ReceivesCorrectContractorAddress`

```cpp
TEST_F(CoreSignalConnectionTest, LaunchedTransaction_ReceivesCorrectContractorAddress) {
    // Arrange
    auto core = testEnv->core();
    core->connectResourcesManagerSignals();

    BaseAddress::Shared capturedAddress;
    testEnv->transactionsManager()->setScheduleInterceptor(
        [&](BaseTransaction::Shared transaction) {
            auto findPathsTx = dynamic_pointer_cast<FindPathsByMaxFlowExchangeTransaction>(transaction);
            if (findPathsTx) {
                capturedAddress = findPathsTx->contractorAddress();
            }
        });

    // Act
    core->resourcesManager()->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert
    EXPECT_EQ(capturedAddress, contractorAddress);
}
```

## Step 6: Test Exchange Equivalents Parameter
**Test Name**: `LaunchedTransaction_ReceivesCorrectExchangeEquivalents`

```cpp
TEST_F(CoreSignalConnectionTest, LaunchedTransaction_ReceivesCorrectExchangeEquivalents) {
    // Arrange
    auto core = testEnv->core();
    core->connectResourcesManagerSignals();

    vector<SerializedEquivalent> capturedEquivs;
    testEnv->transactionsManager()->setScheduleInterceptor(
        [&](BaseTransaction::Shared transaction) {
            auto findPathsTx = dynamic_pointer_cast<FindPathsByMaxFlowExchangeTransaction>(transaction);
            if (findPathsTx) {
                capturedEquivs = findPathsTx->exchangeEquivalents();
            }
        });

    // Act
    core->resourcesManager()->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert
    EXPECT_EQ(capturedEquivs, exchangeEquivalents);
}
```

## Step 7: Test Receiver Equivalent Parameter
**Test Name**: `LaunchedTransaction_ReceivesCorrectReceiverEquivalent`

```cpp
TEST_F(CoreSignalConnectionTest, LaunchedTransaction_ReceivesCorrectReceiverEquivalent) {
    // Arrange
    auto core = testEnv->core();
    core->connectResourcesManagerSignals();

    SerializedEquivalent capturedReceiverEquiv = 0;
    testEnv->transactionsManager()->setScheduleInterceptor(
        [&](BaseTransaction::Shared transaction) {
            auto findPathsTx = dynamic_pointer_cast<FindPathsByMaxFlowExchangeTransaction>(transaction);
            if (findPathsTx) {
                capturedReceiverEquiv = findPathsTx->receiverEquivalent();
            }
        });

    // Act
    core->resourcesManager()->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert
    EXPECT_EQ(capturedReceiverEquiv, receiverEquivalent);
}
```

## Step 8: Test All Parameters Together
**Test Name**: `LaunchedTransaction_ReceivesAllParametersCorrectly`

```cpp
TEST_F(CoreSignalConnectionTest, LaunchedTransaction_ReceivesAllParametersCorrectly) {
    // Arrange
    auto core = testEnv->core();
    core->connectResourcesManagerSignals();

    // Capture all parameters
    struct CapturedParams {
        TransactionUUID uuid;
        BaseAddress::Shared address;
        vector<SerializedEquivalent> exchangeEquivs;
        SerializedEquivalent receiverEquiv;
        bool captured = false;
    } params;

    testEnv->transactionsManager()->setScheduleInterceptor(
        [&](BaseTransaction::Shared transaction) {
            auto findPathsTx = dynamic_pointer_cast<FindPathsByMaxFlowExchangeTransaction>(transaction);
            if (findPathsTx) {
                params.uuid = findPathsTx->requestedTransactionUUID();
                params.address = findPathsTx->contractorAddress();
                params.exchangeEquivs = findPathsTx->exchangeEquivalents();
                params.receiverEquiv = findPathsTx->receiverEquivalent();
                params.captured = true;
            }
        });

    // Act
    core->resourcesManager()->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert - all parameters correct
    ASSERT_TRUE(params.captured) << "Transaction was not captured";
    EXPECT_EQ(params.uuid, testUUID);
    EXPECT_EQ(params.address, contractorAddress);
    EXPECT_EQ(params.exchangeEquivs, exchangeEquivalents);
    EXPECT_EQ(params.receiverEquiv, receiverEquivalent);
}
```

## Step 9: Test Signal Connection Lifecycle
**Test Name**: `CoreInitialization_EstablishesSignalConnection`

```cpp
TEST_F(CoreSignalConnectionTest, CoreInitialization_EstablishesSignalConnection) {
    // Arrange & Act
    auto core = testEnv->createFreshCore(); // Create new Core instance
    core->initialize(); // This should call connectResourcesManagerSignals()

    // Assert - signal connected during initialization
    auto &signal = core->resourcesManager()->requestExchangePathsResourceSignal;
    EXPECT_GT(signal.num_slots(), 0)
        << "Signal not connected during Core initialization";
}
```

## Step 10: Test Transaction Scheduling
**Test Name**: `SignalEmission_SchedulesTransactionImmediately`

```cpp
TEST_F(CoreSignalConnectionTest, SignalEmission_SchedulesTransactionImmediately) {
    // Arrange
    auto core = testEnv->core();
    core->connectResourcesManagerSignals();

    atomic<bool> transactionScheduled{false};
    testEnv->transactionsManager()->setScheduleInterceptor(
        [&](BaseTransaction::Shared transaction) {
            auto findPathsTx = dynamic_pointer_cast<FindPathsByMaxFlowExchangeTransaction>(transaction);
            if (findPathsTx) {
                transactionScheduled = true;
            }
        });

    // Act
    core->resourcesManager()->requestExchangePaths(
        testUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);

    // Assert - transaction scheduled synchronously (no delay)
    EXPECT_TRUE(transactionScheduled)
        << "Transaction was not scheduled immediately";
}
```

## Step 11: Build and Run Tests
**Build tests**:
```bash
cd build-tests
cmake ..
make CoreTest -j4
```

**Run tests**:
```bash
./tests/unit/core/CoreTest --gtest_filter=CoreSignalConnectionTest.*
# Or use ctest
ctest -R CoreSignalConnectionTest -V
```

## Step 12: Verify Test Coverage
Ensure tests cover:
- [ ] Signal connection established
- [ ] Transaction launched on signal emission
- [ ] Transaction UUID parameter correct
- [ ] Contractor address parameter correct
- [ ] Exchange equivalents parameter correct
- [ ] Receiver equivalent parameter correct
- [ ] All parameters together
- [ ] Signal connection lifecycle
- [ ] Immediate transaction scheduling

## Expected Files Modified
- `tests/unit/core/CoreTest.cpp` (extended with new test fixture and tests)
- May need `TransactionsManager` extension for test hooks

# Test Plan

## Test Execution
All tests will be built and executed in the `build-tests` directory:

1. **Build tests**:
   ```bash
   cd build-tests
   cmake ..
   make -j4
   ```

2. **Run Core signal tests**:
   ```bash
   ./tests/unit/core/CoreTest --gtest_filter=CoreSignalConnectionTest.*
   ```

3. **Run with verbose output**:
   ```bash
   ctest -R CoreSignalConnectionTest -V
   ```

## Success Criteria
- All 9+ tests pass
- Signal connection verified
- All parameter passing validated
- Transaction launching mechanism confirmed
- Tests run in < 30 seconds
- No memory leaks or crashes

## Test Categories

### Signal Infrastructure Tests (2 tests)
1. Signal connection exists
2. Signal connection lifecycle

### Transaction Launch Tests (2 tests)
1. Transaction launched on signal emission
2. Immediate transaction scheduling

### Parameter Validation Tests (5 tests)
1. Transaction UUID parameter
2. Contractor address parameter
3. Exchange equivalents parameter
4. Receiver equivalent parameter
5. All parameters together

## Implementation Notes

### TestEnvironment Requirements
The `TestEnvironment` helper needs to provide:
- `Core` instance (fully or partially initialized)
- `ResourcesManager` access
- `TransactionsManager` access with test hooks
- Component mocks if full Core initialization is too heavy

### TransactionsManager Test Hook
May need to add test-only functionality to `TransactionsManager`:

```cpp
class TransactionsManager {
public:
    #ifdef TESTING_MODE
    using ScheduleInterceptor = function<void(BaseTransaction::Shared)>;
    void setScheduleInterceptor(ScheduleInterceptor interceptor) {
        mTestScheduleInterceptor = interceptor;
    }
    #endif

    void scheduleTransaction(BaseTransaction::Shared transaction) {
        #ifdef TESTING_MODE
        if (mTestScheduleInterceptor) {
            mTestScheduleInterceptor(transaction);
        }
        #endif
        // ... normal scheduling logic ...
    }

private:
    #ifdef TESTING_MODE
    ScheduleInterceptor mTestScheduleInterceptor;
    #endif
};
```

Alternatively, use dependency injection with mock `TransactionsManager`.

# Verification and Validation

## Architecture integrity
- **Signal pattern compliance**: Validates Boost.Signals2 usage
- **Core orchestration**: Confirms Core correctly connects components
- **Transaction launching**: Verifies proper transaction lifecycle

## Security
- **No security testing needed**: Internal signal infrastructure
- **Parameter validation**: Ensures correct UUID prevents resource misrouting

## Performance
- **Test execution time**: < 30 seconds for all tests
- **No performance impact**: Tests don't affect production code
- **Synchronous scheduling**: Validates immediate transaction creation

## Scalability
- **Signal slots**: Tests support for signal connection management
- **Multiple emissions**: Validates handling of repeated signal emissions

## Reliability
- **Connection lifecycle**: Tests initialization and persistence
- **Parameter integrity**: All parameters passed correctly
- **Transaction creation**: Validates successful instantiation

## Maintainability
- **Test names**: Clear, descriptive names
- **Test structure**: Arrange-Act-Assert pattern
- **TestEnvironment**: Reusable Core setup
- **Future extensibility**: Easy to add more signal connection tests

## Cost
- **Development cost**: ~4-5 hours (including TestEnvironment setup)
- **Execution cost**: < 30 seconds per run
- **Maintenance cost**: Low (stable infrastructure)

## Compliance
- **Policy compliance**: Tests created per PRD 07 requirements
- **Google Test framework**: Standard testing framework
- **Build system integration**: Integrated into build-tests
- **PRD alignment**: Tests cover all scenarios from PRD Section 6

# Restrictions
- Commit tests only after all tests pass in build-tests
- Use real Core instance or minimal viable mock
- Follow Google Test conventions and naming
- Ensure tests are deterministic
- Tests must run in build-tests environment
- Verify signal connection pattern matches existing resource signals
- Do not modify Core logic to make tests pass (only add test-friendly hooks if needed)
