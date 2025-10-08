# 06-09 - Unit Tests: Transactions

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task 1](06-02-data-structures-and-transaction-classes.md)
- [Previous task 2](06-03-path-processing.md)
- [Previous task 3](06-04-reservation-upgrading-by-equivalent.md)

# Description
Implement comprehensive unit tests for transaction classes created in tasks 06-02, 06-03, and 06-04. This includes testing CoordinatorExchangePaymentTransaction path processing and checkReservationsDirections() validation logic in ReceiverExchangePaymentTransaction and IntermediateNodeExchangePaymentTransaction.

These tests validate the core payment execution logic with multi-equivalent support.

# Requirements and DOD

## Requirements

### Test Category 6: BaseExchangePaymentTransaction
1. Test construction with EquivalentsSubsystemsRouter
2. Test manager retrieval for specific equivalents
3. Test iAmGateway() for specific equivalents

### Test Category 7: CoordinatorExchangePaymentTransaction
4. Test construction with CreditUsageExchangeCommand
5. Test inheritance from BaseExchangePaymentTransaction
6. Test ExchangePathsManager integration
7. Test mPathsStats uses OptimalPathResult
8. Test exchangeEquivalents from command

### Test Category 8: CoordinatorExchangePaymentTransaction Path Processing
9. Test runPathsResourceProcessingStage() retrieves paths from ExchangePathsManager
10. Test paths retrieved for each exchangeEquivalent
11. Test totalAddedFlow tracking and comparison
12. Test loop breaks when totalAddedFlow >= mAmount
13. Test addPathForFurtherProcessing() initialization
14. Test ContractorID → BaseAddress conversion
15. Test error handling for missing contractor

### Test Category 9: ReceiverExchangePaymentTransaction
16. Test construction with EquivalentsSubsystemsRouter
17. Test inheritance from BaseExchangePaymentTransaction
18. Test updateReservations() with equivalent validation
19. Test checkReservationsDirections() for receiver (all incoming in receiver equivalent)

### Test Category 10: IntermediateNodeExchangePaymentTransaction checkReservationsDirections()
20-30. Test all 11 validation scenarios from PRD

## Definition of Done
- [x] All BaseExchangePaymentTransaction tests implemented and passing (3 tests)
- [x] All CoordinatorExchangePaymentTransaction tests implemented and passing (5 tests)
- [x] All CoordinatorExchangePaymentTransaction path processing tests implemented and passing (7 tests)
- [x] All ReceiverExchangePaymentTransaction tests implemented and passing (4 tests)
- [x] All IntermediateNodeExchangePaymentTransaction checkReservationsDirections() tests implemented and passing (11 tests)
- [x] All tests compile without errors
- [x] All tests pass in build-tests
- [x] Test coverage adequate for Complex task
- [x] Mock data provided where applicable

# Implementation Plan

## Test File Structure
Create test files:
- `tests/core/transactions/TestBaseExchangePaymentTransaction.cpp`
- `tests/core/transactions/TestCoordinatorExchangePaymentTransaction.cpp`
- `tests/core/transactions/TestReceiverExchangePaymentTransaction.cpp`
- `tests/core/transactions/TestIntermediateNodeExchangePaymentTransaction.cpp`

## Test Category 6: BaseExchangePaymentTransaction (3 tests)

### Test 6.1: testBaseExchangePaymentTransactionConstruction
```cpp
TEST(BaseExchangePaymentTransaction, Construction) {
    auto contractorsManager = make_shared<MockContractorsManager>();
    auto equivalentsRouter = make_shared<MockEquivalentsSubsystemsRouter>();
    auto storageHandler = make_shared<MockStorageHandler>();
    auto resourcesManager = make_shared<MockResourcesManager>();
    auto keystore = make_shared<MockKeystore>();
    auto subsystemsController = make_shared<MockSubsystemsController>();
    auto logger = MockLogger();

    // Construction should succeed
    EXPECT_NO_THROW({
        BaseExchangePaymentTransaction transaction(
            TransactionType::Payments_CoordinatorPaymentTransaction,
            SerializedEquivalent(1),
            contractorsManager.get(),
            equivalentsRouter.get(),
            storageHandler.get(),
            resourcesManager.get(),
            keystore.get(),
            logger,
            subsystemsController.get());
    });
}
```

### Test 6.2: testBaseExchangePaymentTransactionGetTrustLinesManager
```cpp
TEST(BaseExchangePaymentTransaction, GetTrustLinesManager) {
    // Setup mocks
    auto equivalentsRouter = make_shared<MockEquivalentsSubsystemsRouter>();
    auto trustLinesManager1 = make_shared<MockTrustLinesManager>();
    auto trustLinesManager2 = make_shared<MockTrustLinesManager>();

    EXPECT_CALL(*equivalentsRouter, getTrustLinesManager(SerializedEquivalent(1)))
        .WillOnce(Return(trustLinesManager1.get()));
    EXPECT_CALL(*equivalentsRouter, getTrustLinesManager(SerializedEquivalent(2)))
        .WillOnce(Return(trustLinesManager2.get()));

    // Create transaction with mocked router
    BaseExchangePaymentTransaction transaction(...);

    auto manager1 = transaction.getTrustLinesManager(SerializedEquivalent(1));
    auto manager2 = transaction.getTrustLinesManager(SerializedEquivalent(2));

    EXPECT_EQ(manager1, trustLinesManager1.get());
    EXPECT_EQ(manager2, trustLinesManager2.get());
}
```

### Test 6.3: testBaseExchangePaymentTransactionIAmGateway
```cpp
TEST(BaseExchangePaymentTransaction, IAmGateway) {
    auto equivalentsRouter = make_shared<MockEquivalentsSubsystemsRouter>();

    EXPECT_CALL(*equivalentsRouter, iAmGateway(SerializedEquivalent(1)))
        .WillOnce(Return(true));
    EXPECT_CALL(*equivalentsRouter, iAmGateway(SerializedEquivalent(2)))
        .WillOnce(Return(false));

    BaseExchangePaymentTransaction transaction(...);

    EXPECT_TRUE(transaction.iAmGateway(SerializedEquivalent(1)));
    EXPECT_FALSE(transaction.iAmGateway(SerializedEquivalent(2)));
}
```

## Test Category 7: CoordinatorExchangePaymentTransaction (5 tests)

### Test 7.1: testCoordinatorExchangePaymentTransactionConstruction
```cpp
TEST(CoordinatorExchangePaymentTransaction, Construction) {
    auto command = make_shared<CreditUsageExchangeCommand>(...);
    // ... setup all dependencies

    EXPECT_NO_THROW({
        CoordinatorExchangePaymentTransaction transaction(
            command,
            contractorsManager.get(),
            equivalentsRouter.get(),
            storageHandler.get(),
            resourcesManager.get(),
            exchangePathsManager.get(),
            keystore.get(),
            true,
            eventsInterfaceManager.get(),
            logger,
            subsystemsController.get());
    });
}
```

### Test 7.2: testCoordinatorInheritsBaseExchangePaymentTransaction
```cpp
TEST(CoordinatorExchangePaymentTransaction, InheritsBase) {
    CoordinatorExchangePaymentTransaction transaction(...);

    // Should be able to cast to base
    BaseExchangePaymentTransaction* base = &transaction;
    ASSERT_NE(base, nullptr);
}
```

### Test 7.3-7.5: Test ExchangePathsManager field, mPathsStats type, exchangeEquivalents

## Test Category 8: Path Processing (7 tests)

### Test 8.1: testRunPathsResourceProcessingSingleEquivalent
```cpp
TEST(CoordinatorExchangePaymentTransaction, RunPathsResourceProcessingSingleEquivalent) {
    // Setup command with single exchangeEquivalent
    auto command = make_shared<CreditUsageExchangeCommand>(...);
    command->exchangeEquivalents = {SerializedEquivalent(1)};
    command->amount = TrustLineAmount(1000);

    // Mock ExchangePathsManager to return paths
    auto exchangePathsManager = make_shared<MockExchangePathsManager>();
    vector<OptimalPathResult> mockPaths = {
        {.optimal_flow = 600, .received_amount = 600},
        {.optimal_flow = 500, .received_amount = 500}
    };

    PathCacheKey expectedKey{contractorID, SerializedEquivalent(1), SerializedEquivalent(2)};
    EXPECT_CALL(*exchangePathsManager, retrievePaths(expectedKey))
        .WillOnce(Return(&mockPaths));

    CoordinatorExchangePaymentTransaction transaction(...);

    auto result = transaction.runPathsResourceProcessingStage();

    // Should have added paths and succeeded
    EXPECT_EQ(transaction.mPathsStats.size(), 2);
}
```

### Test 8.2: testRunPathsResourceProcessingSufficientFlowEarlyBreak
```cpp
TEST(CoordinatorExchangePaymentTransaction, SufficientFlowEarlyBreak) {
    // Setup: mAmount = 1000, first 2 paths provide 1200 flow
    auto command = make_shared<CreditUsageExchangeCommand>(...);
    command->amount = TrustLineAmount(1000);

    vector<OptimalPathResult> mockPaths = {
        {.received_amount = 700},
        {.received_amount = 500},  // Total 1200 >= 1000
        {.received_amount = 300}   // Should not be added
    };

    auto exchangePathsManager = make_shared<MockExchangePathsManager>();
    EXPECT_CALL(*exchangePathsManager, retrievePaths(_))
        .WillOnce(Return(&mockPaths));

    CoordinatorExchangePaymentTransaction transaction(...);
    transaction.runPathsResourceProcessingStage();

    // Should only add first 2 paths
    EXPECT_EQ(transaction.mPathsStats.size(), 2);
}
```

### Test 8.3: testAddPathForFurtherProcessingInitializesStates
```cpp
TEST(CoordinatorExchangePaymentTransaction, AddPathInitializesStates) {
    OptimalPathResult pathResult;
    pathResult.path.ids = {ContractorID(1), ContractorID(2), ContractorID(3), ContractorID(4), ContractorID(5)};

    // Mock ContractorsManager
    auto contractorsManager = make_shared<MockContractorsManager>();
    // Setup mock to return contractors for each ID...

    CoordinatorExchangePaymentTransaction transaction(...);
    transaction.addPathForFurtherProcessing(pathResult);

    // Should have 1 path in mPathsStats
    ASSERT_EQ(transaction.mPathsStats.size(), 1);

    auto& addedPath = transaction.mPathsStats.begin()->second;

    // mIntermediateNodesStates should be initialized
    EXPECT_EQ(addedPath->mIntermediateNodesStates.size(), 5);
    for (const auto& state : addedPath->mIntermediateNodesStates) {
        EXPECT_EQ(state, OptimalPathResult::NodeState::ReservationRequestDoesntSent);
    }
}
```

### Test 8.4: testAddPathConvertsIdsToNodes
```cpp
TEST(CoordinatorExchangePaymentTransaction, AddPathConvertsIdsToNodes) {
    OptimalPathResult pathResult;
    pathResult.path.ids = {ContractorID(10), ContractorID(20), ContractorID(30)};

    // Mock ContractorsManager
    auto contractorsManager = make_shared<MockContractorsManager>();
    auto contractor1 = make_shared<Contractor>(make_shared<IPv4WithPortAddress>("127.0.0.1:2000"));
    auto contractor2 = make_shared<Contractor>(make_shared<IPv4WithPortAddress>("127.0.0.1:2001"));
    auto contractor3 = make_shared<Contractor>(make_shared<IPv4WithPortAddress>("127.0.0.1:2002"));

    EXPECT_CALL(*contractorsManager, contractor(ContractorID(10))).WillOnce(Return(contractor1));
    EXPECT_CALL(*contractorsManager, contractor(ContractorID(20))).WillOnce(Return(contractor2));
    EXPECT_CALL(*contractorsManager, contractor(ContractorID(30))).WillOnce(Return(contractor3));

    CoordinatorExchangePaymentTransaction transaction(...);
    transaction.addPathForFurtherProcessing(pathResult);

    auto& addedPath = transaction.mPathsStats.begin()->second;

    // nodes vector should be populated
    EXPECT_EQ(addedPath->path.nodes.size(), 3);
    EXPECT_EQ(addedPath->path.nodes[0], contractor1->mainAddress());
}
```

### Test 8.5: testAddPathContractorNotFound
```cpp
TEST(CoordinatorExchangePaymentTransaction, AddPathContractorNotFound) {
    OptimalPathResult pathResult;
    pathResult.path.ids = {ContractorID(999)};  // Non-existent

    auto contractorsManager = make_shared<MockContractorsManager>();
    EXPECT_CALL(*contractorsManager, contractor(ContractorID(999)))
        .WillOnce(Return(nullptr));

    CoordinatorExchangePaymentTransaction transaction(...);

    // Should throw ValueError
    EXPECT_THROW({
        transaction.addPathForFurtherProcessing(pathResult);
    }, ValueError);
}
```

### Test 8.6-8.7: Test multiple equivalents, no paths available

## Test Category 9: ReceiverExchangePaymentTransaction (4 tests)

### Test 9.1-9.4: Test construction, inheritance, updateReservations, checkReservationsDirections

## Test Category 10: IntermediateNode checkReservationsDirections() (11 tests)

### Test 10.1: testAllOutgoingInSingleEquivalent
```cpp
TEST(IntermediateNodeExchangePaymentTransaction, AllOutgoingInSingleEquivalent) {
    IntermediateNodeExchangePaymentTransaction transaction(...);

    // Setup: outgoing in equiv 1 only
    transaction.mReservations[ContractorID(1)] = {
        {PathID(1), make_shared<AmountReservation>(
            uuid, TrustLineAmount(100), AmountReservation::Outgoing, SerializedEquivalent(1))}
    };
    transaction.mReservations[ContractorID(2)] = {
        {PathID(2), make_shared<AmountReservation>(
            uuid, TrustLineAmount(50), AmountReservation::Outgoing, SerializedEquivalent(1))}
    };

    // Incoming also equiv 1
    transaction.mReservations[ContractorID(3)] = {
        {PathID(3), make_shared<AmountReservation>(
            uuid, TrustLineAmount(150), AmountReservation::Incoming, SerializedEquivalent(1))}
    };

    bool result = transaction.checkReservationsDirections();

    EXPECT_TRUE(result);
}
```

### Test 10.2: testOutgoingInMultipleEquivalentsFail
```cpp
TEST(IntermediateNodeExchangePaymentTransaction, OutgoingInMultipleEquivalentsFail) {
    IntermediateNodeExchangePaymentTransaction transaction(...);

    // Setup: outgoing in equiv 1 and equiv 2
    transaction.mReservations[ContractorID(1)] = {
        {PathID(1), make_shared<AmountReservation>(
            uuid, TrustLineAmount(100), AmountReservation::Outgoing, SerializedEquivalent(1))}
    };
    transaction.mReservations[ContractorID(2)] = {
        {PathID(2), make_shared<AmountReservation>(
            uuid, TrustLineAmount(50), AmountReservation::Outgoing, SerializedEquivalent(2))}
    };

    bool result = transaction.checkReservationsDirections();

    EXPECT_FALSE(result);  // Should fail: multiple outgoing equivalents
}
```

### Test 10.3: testIncomingSameAsOutgoingNoCommission
```cpp
TEST(IntermediateNodeExchangePaymentTransaction, IncomingSameAsOutgoingNoCommission) {
    IntermediateNodeExchangePaymentTransaction transaction(...);

    // Setup: incoming equiv 1 (100), outgoing equiv 1 (100), no commission
    transaction.mReservations[ContractorID(1)] = {
        {PathID(1), make_shared<AmountReservation>(
            uuid, TrustLineAmount(100), AmountReservation::Incoming, SerializedEquivalent(1))}
    };
    transaction.mReservations[ContractorID(2)] = {
        {PathID(2), make_shared<AmountReservation>(
            uuid, TrustLineAmount(100), AmountReservation::Outgoing, SerializedEquivalent(1))}
    };

    // Mock CommissionsManager: no commission
    EXPECT_CALL(*commissionsManager, get(SerializedEquivalent(1)))
        .WillOnce(Return(nullptr));

    bool result = transaction.checkReservationsDirections();

    EXPECT_TRUE(result);  // Totals match
}
```

### Test 10.4: testIncomingSameAsOutgoingWithCommission
```cpp
TEST(IntermediateNodeExchangePaymentTransaction, IncomingSameAsOutgoingWithCommission) {
    IntermediateNodeExchangePaymentTransaction transaction(...);

    // Setup: incoming equiv 1 (110), outgoing equiv 1 (100), commission equiv 1 = 10
    transaction.mReservations[ContractorID(1)] = {
        {PathID(1), make_shared<AmountReservation>(
            uuid, TrustLineAmount(110), AmountReservation::Incoming, SerializedEquivalent(1))}
    };
    transaction.mReservations[ContractorID(2)] = {
        {PathID(2), make_shared<AmountReservation>(
            uuid, TrustLineAmount(100), AmountReservation::Outgoing, SerializedEquivalent(1))}
    };

    // Mock CommissionsManager
    auto commission = make_shared<Commission>(TrustLineAmount(10));
    EXPECT_CALL(*commissionsManager, get(SerializedEquivalent(1)))
        .WillOnce(Return(commission));

    bool result = transaction.checkReservationsDirections();

    EXPECT_TRUE(result);  // 110 - 10 = 100
}
```

### Test 10.5: testIncomingDifferentFromOutgoingConvert
```cpp
TEST(IntermediateNodeExchangePaymentTransaction, IncomingDifferentConvert) {
    IntermediateNodeExchangePaymentTransaction transaction(...);

    // Setup: incoming equiv 1 (100), outgoing equiv 2 (200), rate 1→2 = 2.0
    transaction.mReservations[ContractorID(1)] = {
        {PathID(1), make_shared<AmountReservation>(
            uuid, TrustLineAmount(100), AmountReservation::Incoming, SerializedEquivalent(1))}
    };
    transaction.mReservations[ContractorID(2)] = {
        {PathID(2), make_shared<AmountReservation>(
            uuid, TrustLineAmount(200), AmountReservation::Outgoing, SerializedEquivalent(2))}
    };

    // Mock ExchangeRatesManager
    auto rate = make_shared<ExchangeRate>(2.0);
    EXPECT_CALL(*exchangeRatesManager, get(SerializedEquivalent(1), SerializedEquivalent(2)))
        .WillOnce(Return(rate));
    EXPECT_CALL(*exchangeRatesManager, calculateConvertedAmount(
        SerializedEquivalent(1), SerializedEquivalent(2), TrustLineAmount(100)))
        .WillOnce(Return(TrustLineAmount(200)));

    bool result = transaction.checkReservationsDirections();

    EXPECT_TRUE(result);  // 100 * 2.0 = 200
}
```

### Test 10.6-10.11: Implement remaining 6 scenarios
- Multiple incoming equivalents
- Commission only for transit (same equivalent)
- Commission charged once per equivalent
- No exchange rate found → fail
- Overflow during conversion → fail
- Sums don't match → fail

# Test Plan

## Test Execution
- Build tests in `build-tests`
- Run all test binaries
- Verify 100% pass rate

## Coverage Requirements
As this is a Complex task:
- BaseExchangePaymentTransaction: 70%+ coverage
- CoordinatorExchangePaymentTransaction: 80%+ coverage (critical path processing)
- ReceiverExchangePaymentTransaction: 70%+ coverage
- IntermediateNodeExchangePaymentTransaction: 90%+ coverage (checkReservationsDirections critical)

## Mock Requirements
- Mock EquivalentsSubsystemsRouter
- Mock ExchangePathsManager
- Mock ExchangeRatesManager
- Mock CommissionsManager
- Mock ContractorsManager
- Mock all other dependencies

# Verification and Validation

## Architecture integrity
- Tests validate transaction logic correctness
- Tests ensure multi-equivalent support works

## Security
- No security concerns in unit tests (in-memory only)

## Performance
- All tests should complete in < 10 seconds total

## Scalability
- Tests validate behavior with multiple equivalents and paths

## Reliability
- Comprehensive test coverage ensures transaction reliability
- All 11 checkReservationsDirections scenarios covered

## Maintainability
- Clear test names describe scenarios
- Easy to add new tests for edge cases

## Cost
- No additional infrastructure required (tests only)

## Compliance
- Follows repository policy for unit testing
- Tests built and executed in build-tests only

# Restrictions
- All tests must pass before task completion
- No integration tests (unit tests only)
- Mock all external dependencies
