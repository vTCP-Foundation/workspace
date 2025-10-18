# 06-08 - Unit Tests: Data Structures and Infrastructure

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Previous task 1](06-02-data-structures-and-transaction-classes.md)
- [Previous task 2](06-05-base-payment-transaction-upgrading.md)
- [Previous task 3](06-06-amount-reservation-handler-upgrading.md)

# Description
Implement comprehensive unit tests for data structures and infrastructure components created in tasks 06-02, 06-05, and 06-06. This includes testing PathReservation, OptimalPathResult enhancement, ExchangePath enhancement, AmountReservation enhancement, and AmountReservationsHandler enhancement.

These tests validate the foundation layer for multi-equivalent payment support.

# Requirements and DOD

## Requirements

### Test Category 1: PathReservation
1. Test structure construction with all fields
2. Test field access (pathID, amount, equivalent)

### Test Category 2: OptimalPathResult Enhancement
3. Test all new fields added (mMaxPathFlow, mIsValid, mIntermediateNodesStates)
4. Test all 17 methods from PathStats
5. Verify NO mPath field exists
6. Test path() returns ExchangePath& (not Path::Shared)

### Test Category 3: ExchangePath Enhancement
7. Test field rename: nodes → ids
8. Test new nodes (BaseAddress) field
9. Test all Path methods
10. Test ContractorID → BaseAddress conversion via ContractorsManager
11. Test conversion error handling

### Test Category 4: AmountReservation Enhancement
12. Test SerializedEquivalent field
13. Test equivalent() getter
14. Test constructor with equivalent
15. Test equality operator includes equivalent

### Test Category 5: AmountReservationsHandler Enhancement
16. Test reserve() with equivalent
17. Test updateReservation() with equivalent
18. Test free() with equivalent
19. Test totalReserved() filters by equivalent
20. Test getReservation() matches by equivalent
21. Test multiple reservations with different equivalents on same trust line

## Definition of Done
- [ ] All PathReservation tests implemented and passing (2 tests)
- [ ] All OptimalPathResult tests implemented and passing (20+ tests)
- [ ] All ExchangePath tests implemented and passing (5 tests)
- [ ] All AmountReservation tests implemented and passing (4 tests)
- [ ] All AmountReservationsHandler tests implemented and passing (9 tests)
- [ ] All tests compile without errors
- [ ] All tests pass in build-tests
- [ ] Test coverage adequate for Simple/Moderate complexity components
- [ ] Test files added to tests/unit/CMakeLists.txt
- [ ] Real objects used instead of mocks (following ExchangePathsManagerTest pattern)
- [ ] Exception handling tested (ValueError, NotFoundError)
- [ ] Edge cases covered (empty vectors, null values, boundary conditions)

# Implementation Plan

## Test Strategy and Best Practices

### Use Real Objects Instead of Mocks
- Follow the pattern from ExchangePathsManagerTest.cpp
- Create TestEnvironment helper classes where needed
- Use real instances of ContractorsManager, StorageHandlerSQLite, etc.
- Only mock external third-party services (none in this task)

### Exception Testing
- Test that methods throw correct exceptions (ValueError, NotFoundError)
- Use EXPECT_THROW and ASSERT_THROW from GTest
- Verify exception messages where applicable

### Edge Case Coverage
- Test empty vectors and collections
- Test boundary values (0, max values)
- Test null/invalid inputs where applicable
- Test concurrent operations where relevant

### Parameterized Tests
- Use for AmountReservationsHandler to test different equivalent combinations
- Reduces code duplication
- Improves test coverage

## Test File Structure
Create test files in appropriate test directory:
- `tests/unit/transactions/TestPathReservation.cpp`
- `tests/unit/paths/TestOptimalPathResult.cpp`
- `tests/unit/paths/TestExchangePath.cpp`
- `tests/unit/payments/TestAmountReservation.cpp`
- `tests/unit/payments/TestAmountReservationsHandler.cpp`

All test files must be added to `tests/unit/CMakeLists.txt`

## Test Category 1: PathReservation (2 tests)

### Test 1.1: testPathReservationConstruction
```cpp
TEST(PathReservation, Construction) {
    PathID pathID = 123;
    auto amount = make_shared<const TrustLineAmount>(1000);
    SerializedEquivalent equivalent = 5;

    PathReservation reservation(pathID, amount, equivalent);

    EXPECT_EQ(reservation.pathID, pathID);
    EXPECT_EQ(*reservation.amount, TrustLineAmount(1000));
    EXPECT_EQ(reservation.equivalent, equivalent);
}
```

### Test 1.2: testPathReservationFieldAccess
```cpp
TEST(PathReservation, FieldAccess) {
    PathID pathID = 456;
    auto amount = make_shared<const TrustLineAmount>(2500);
    SerializedEquivalent equivalent = 10;

    PathReservation reservation(pathID, amount, equivalent);

    // Access all fields directly
    PathID accessedPathID = reservation.pathID;
    TrustLineAmount accessedAmount = *reservation.amount;
    SerializedEquivalent accessedEquiv = reservation.equivalent;

    EXPECT_EQ(accessedPathID, 456);
    EXPECT_EQ(accessedAmount, TrustLineAmount(2500));
    EXPECT_EQ(accessedEquiv, 10);
}
```

## Test Category 2: OptimalPathResult Enhancement (20 tests)

### Test 2.1: testOptimalPathResultFieldsAdded
```cpp
TEST(OptimalPathResult, FieldsAdded) {
    OptimalPathResult result;

    // Verify new fields exist and are accessible
    result.mMaxPathFlow = TrustLineAmount(1000);
    result.mIsValid = true;
    result.mIntermediateNodesStates.push_back(OptimalPathResult::NodeState::ReservationRequestDoesntSent);

    EXPECT_EQ(result.mMaxPathFlow, TrustLineAmount(1000));
    EXPECT_TRUE(result.mIsValid);
    EXPECT_EQ(result.mIntermediateNodesStates.size(), 1);
}
```

### Test 2.2: testOptimalPathResultSetNodeState
```cpp
TEST(OptimalPathResult, SetNodeState) {
    OptimalPathResult result;
    result.mIntermediateNodesStates.resize(5, OptimalPathResult::NodeState::ReservationRequestDoesntSent);

    result.setNodeState(2, OptimalPathResult::NodeState::ReservationApproved);

    EXPECT_EQ(result.mIntermediateNodesStates[2], OptimalPathResult::NodeState::ReservationApproved);
    EXPECT_EQ(result.mIntermediateNodesStates[0], OptimalPathResult::NodeState::ReservationRequestDoesntSent);
}
```

### Test 2.3-2.20: Implement remaining PathStats methods tests
- maxFlow()
- shortageMaxFlow()
- path() returns ExchangePath&
- containsIntermediateNodes()
- currentIntermediateNodeAndPos()
- nextIntermediateNodeAndPos()
- reservationRequestSentToAllNodes()
- isNeighborAmountReserved()
- isWaitingForNeighborReservationResponse()
- isWaitingForNeighborReservationPropagationResponse()
- isWaitingForReservationResponse()
- isReadyToSendNextReservationRequest()
- isLastIntermediateNodeProcessed()
- isLastIntermediateNodeApproved()
- isValid()
- setUnusable()
- Verify NO mPath field exists (compilation test)

## Test Category 3: ExchangePath Enhancement (5 tests)

### Test 3.1: testExchangePathFieldRenamed
```cpp
TEST(ExchangePath, FieldRenamed) {
    ExchangePath path;

    // ids field should exist (renamed from nodes)
    path.ids.push_back(ContractorID(1));
    path.ids.push_back(ContractorID(2));

    EXPECT_EQ(path.ids.size(), 2);
    EXPECT_EQ(path.ids[0], ContractorID(1));
}
```

### Test 3.2: testExchangePathNodesFieldAdded
```cpp
TEST(ExchangePath, NodesFieldAdded) {
    ExchangePath path;

    // nodes (BaseAddress) field should exist
    auto address1 = make_shared<IPv4WithPortAddress>("127.0.0.1:2000");
    auto address2 = make_shared<IPv4WithPortAddress>("127.0.0.1:2001");

    path.nodes.push_back(address1);
    path.nodes.push_back(address2);

    EXPECT_EQ(path.nodes.size(), 2);
}
```

### Test 3.3: testExchangePathMethodsFromPath
```cpp
TEST(ExchangePath, MethodsFromPath) {
    ExchangePath path;

    // Test methods from Path class exist and work
    bool valid = path.isValid();
    TrustLineAmount capacity = path.calculateMaxCapacity();

    // Should compile and execute without errors
    SUCCEED();
}
```

### Test 3.4: testExchangePathIdsAndNodesIndependent
```cpp
TEST(ExchangePath, IdsAndNodesIndependent) {
    ExchangePath path;

    path.ids.push_back(ContractorID(1));
    path.ids.push_back(ContractorID(2));

    auto address = make_shared<IPv4WithPortAddress>("127.0.0.1:2000");
    path.nodes.push_back(address);

    // ids and nodes should be independent
    EXPECT_EQ(path.ids.size(), 2);
    EXPECT_EQ(path.nodes.size(), 1);
}
```

### Test 3.5: testExchangePathContractorIDConversionViaManager
```cpp
TEST(ExchangePath, ContractorIDConversionViaManager) {
    // Mock ContractorsManager
    auto contractorsManager = make_shared<MockContractorsManager>();

    ExchangePath path;
    path.ids = {ContractorID(1), ContractorID(2)};

    // Setup mocks
    auto contractor1 = make_shared<Contractor>(
        make_shared<IPv4WithPortAddress>("127.0.0.1:2000"));
    auto contractor2 = make_shared<Contractor>(
        make_shared<IPv4WithPortAddress>("127.0.0.1:2001"));

    EXPECT_CALL(*contractorsManager, contractor(ContractorID(1)))
        .WillOnce(Return(contractor1));
    EXPECT_CALL(*contractorsManager, contractor(ContractorID(2)))
        .WillOnce(Return(contractor2));

    // Convert ids to nodes
    for (const auto& id : path.ids) {
        auto contractor = contractorsManager->contractor(id);
        ASSERT_NE(contractor, nullptr);
        path.nodes.push_back(contractor->mainAddress());
    }

    EXPECT_EQ(path.nodes.size(), 2);
}
```

## Test Category 4: AmountReservation Enhancement (4 tests)

### Test 4.1: testAmountReservationConstructorWithEquivalent
```cpp
TEST(AmountReservation, ConstructorWithEquivalent) {
    TransactionUUID uuid = TransactionUUID::generate();
    TrustLineAmount amount(500);
    AmountReservation::ReservationDirection direction = AmountReservation::Outgoing;
    SerializedEquivalent equivalent = 3;

    AmountReservation reservation(uuid, amount, direction, equivalent);

    EXPECT_EQ(reservation.transactionUUID(), uuid);
    EXPECT_EQ(reservation.amount(), amount);
    EXPECT_EQ(reservation.direction(), direction);
    EXPECT_EQ(reservation.equivalent(), equivalent);
}
```

### Test 4.2: testAmountReservationEquivalentGetter
```cpp
TEST(AmountReservation, EquivalentGetter) {
    TransactionUUID uuid = TransactionUUID::generate();
    TrustLineAmount amount(1000);
    SerializedEquivalent equivalent = 7;

    AmountReservation reservation(
        uuid, amount, AmountReservation::Incoming, equivalent);

    EXPECT_EQ(reservation.equivalent(), 7);
}
```

### Test 4.3: testAmountReservationEqualityWithEquivalent
```cpp
TEST(AmountReservation, EqualityWithEquivalent) {
    TransactionUUID uuid = TransactionUUID::generate();
    TrustLineAmount amount(1000);

    AmountReservation reservation1(
        uuid, amount, AmountReservation::Outgoing, SerializedEquivalent(1));
    AmountReservation reservation2(
        uuid, amount, AmountReservation::Outgoing, SerializedEquivalent(1));
    AmountReservation reservation3(
        uuid, amount, AmountReservation::Outgoing, SerializedEquivalent(2));

    EXPECT_EQ(reservation1, reservation2);
    EXPECT_NE(reservation1, reservation3);  // Different equivalent
}
```

### Test 4.4: testAmountReservationFieldsComplete
```cpp
TEST(AmountReservation, FieldsComplete) {
    AmountReservation reservation(
        TransactionUUID::generate(),
        TrustLineAmount(100),
        AmountReservation::Incoming,
        SerializedEquivalent(5));

    // Verify all getters work
    auto uuid = reservation.transactionUUID();
    auto amount = reservation.amount();
    auto direction = reservation.direction();
    auto equivalent = reservation.equivalent();

    SUCCEED();  // Compilation is the test
}
```

## Test Category 5: AmountReservationsHandler Enhancement (9 tests)

### Test 5.1: testReserveWithEquivalent
```cpp
TEST(AmountReservationsHandler, ReserveWithEquivalent) {
    AmountReservationsHandler handler;
    ContractorID contractor(123);
    TransactionUUID uuid = TransactionUUID::generate();
    TrustLineAmount amount(500);
    SerializedEquivalent equivalent(2);

    auto reservation = handler.reserve(
        contractor, uuid, amount, AmountReservation::Outgoing, equivalent);

    ASSERT_NE(reservation, nullptr);
    EXPECT_EQ(reservation->equivalent(), equivalent);
    EXPECT_EQ(reservation->amount(), amount);
}
```

### Test 5.2: testReserveMultipleEquivalentsSameContractor
```cpp
TEST(AmountReservationsHandler, ReserveMultipleEquivalentsSameContractor) {
    AmountReservationsHandler handler;
    ContractorID contractor(456);
    TransactionUUID uuid1 = TransactionUUID::generate();
    TransactionUUID uuid2 = TransactionUUID::generate();

    auto reservation1 = handler.reserve(
        contractor, uuid1, TrustLineAmount(100),
        AmountReservation::Outgoing, SerializedEquivalent(1));

    auto reservation2 = handler.reserve(
        contractor, uuid2, TrustLineAmount(200),
        AmountReservation::Outgoing, SerializedEquivalent(2));

    ASSERT_NE(reservation1, nullptr);
    ASSERT_NE(reservation2, nullptr);
    EXPECT_EQ(reservation1->equivalent(), SerializedEquivalent(1));
    EXPECT_EQ(reservation2->equivalent(), SerializedEquivalent(2));
}
```

### Test 5.3-5.9: Implement remaining handler tests
- updateReservation() with equivalent
- free() matches by equivalent
- totalReserved() filters by equivalent
- getReservation() matches by equivalent
- Multiple reservations with same parameters but different equivalents
- Equivalent independence in operations

# Test Plan

## Test Execution
- Build tests in `build-tests`
- Run all test binaries
- Verify 100% pass rate

## Coverage Requirements
As this is a Moderate task:
- PathReservation: 100% coverage (simple structure)
- OptimalPathResult: 80%+ coverage (complex with 17 methods)
- ExchangePath: 80%+ coverage
- AmountReservation: 100% coverage (simple class)
- AmountReservationsHandler: 90%+ coverage (critical infrastructure)

## Mock Requirements
- Mock ContractorsManager for ContractorID conversion tests
- Mock data for SQLite and PostgreSQL where applicable (mostly in-memory tests)

# Verification and Validation

## Architecture integrity
- Tests validate data structure contracts
- Tests ensure infrastructure works correctly

## Security
- No security concerns in unit tests (in-memory only)

## Performance
- All tests should complete in < 5 seconds total

## Scalability
- Tests validate behavior with multiple equivalents

## Reliability
- Comprehensive test coverage ensures component reliability

## Maintainability
- Clear test names describe what is tested
- Easy to add new tests as components evolve

## Cost
- No additional infrastructure required (tests only)

## Compliance
- Follows repository policy for unit testing
- Tests built and executed in build-tests only

# Restrictions
- All tests must pass before task completion
- No integration tests (unit tests only)
- No Docker usage
- Mock all external dependencies
