# 18-07 - Unit Tests

# Links
- [PRD-18: Audit Mechanism Based on Finalized Transactions](../../prd/vtcpd/18-audit-mechanism-finalized-transactions.md)
- [Previous task: 18-01-audit-messages-extension](18-01-audit-messages-extension.md)
- [Previous task: 18-02-receipt-handlers-and-zero-audit-number](18-02-receipt-handlers-and-zero-audit-number.md)
- [Previous task: 18-03-trust-lines-manager-receipts-preservation](18-03-trust-lines-manager-receipts-preservation.md)
- [Previous task: 18-04-audit-source-transaction](18-04-audit-source-transaction.md)
- [Previous task: 18-05-audit-target-transaction](18-05-audit-target-transaction.md)
- [Previous task: 18-06-other-audit-transactions](18-06-other-audit-transactions.md)

# Description

This task implements unit tests for all components modified or added in PRD-18, plus PostgreSQL integration tests for receipt handler methods. The tests verify correct functionality of:
- Message serialization/deserialization with transaction lists
- Transaction list sorting and hashing
- Receipt handler new methods
- TrustLinesManager receipt amount preservation
- Audit signature payload with transaction list hash

Unit tests focus on individual component behavior in isolation, using mocks where necessary.

# Requirements and DOD

## Requirements

### AuditMessage Tests
1. Test serialization with empty transaction list
2. Test serialization with single transaction UUID
3. Test serialization with multiple transaction UUIDs
4. Test deserialization correctly reconstructs transaction list
5. Test round-trip serialization/deserialization preserves data
6. Test transaction list is sorted after construction

### AuditResponseMessage Tests
7. Test serialization with empty transaction list (typical case)
8. Test serialization with transaction list (Audit_UpdateTransactionsList case)
9. Test deserialization correctly reconstructs transaction list
10. Test round-trip serialization/deserialization

### Transaction List Sorting Tests
11. Test sorting produces lexicographic order by raw UUID bytes
12. Test sorting is stable and deterministic
13. Test sorting handles empty list
14. Test sorting handles single element
15. Test sorting matches `compareTransactionUUID` behavior

### Transaction List Hash Tests
16. Test hash of empty list produces consistent result
17. Test hash of single UUID produces expected format
18. Test hash of multiple UUIDs produces consistent result
19. Test hash changes when list content changes
20. Test hash is same for same content regardless of initial order (after sorting)
21. Test hash format: SHA-256 over count + concatenated UUIDs

### Receipt Handler Tests (SQLite)
22. Test `getFinalizedReceiptsWithZeroAuditNumber` returns only auditNumber=0 receipts
23. Test `getFinalizedReceiptsWithZeroAuditNumber` returns only finalized transactions
24. Test `getFinalizedReceiptsWithZeroAuditNumber` returns empty for no matches
25. Test `updateAuditNumberByTransactionUUIDs` updates specified receipts
26. Test `updateAuditNumberByTransactionUUIDs` doesn't affect other receipts
27. Test `updateAuditNumberByTransactionUUIDs` with empty list

### Receipt Handler Tests (PostgreSQL Integration)
28. Test `getFinalizedReceiptsWithZeroAuditNumber` returns only auditNumber=0 receipts
29. Test `getFinalizedReceiptsWithZeroAuditNumber` returns only finalized transactions
30. Test `getFinalizedReceiptsWithZeroAuditNumber` returns empty for no matches
31. Test `updateAuditNumberByTransactionUUIDs` updates specified receipts
32. Test `updateAuditNumberByTransactionUUIDs` doesn't affect other receipts
33. Test `updateAuditNumberByTransactionUUIDs` with empty list

### TrustLinesManager Tests
34. Test `updateTrustLineTotalReceiptsAmounts` correctly calculates new balance
35. Test `updateTrustLineTotalReceiptsAmounts` preserves excluded incoming amount
36. Test `updateTrustLineTotalReceiptsAmounts` preserves excluded outgoing amount
37. Test `updateTrustLineTotalReceiptsAmounts` with zero excluded amounts
38. Test `updateTrustLineTotalReceiptsAmounts` with zero included amounts

### Audit Signature Payload Tests
39. Test signature payload includes transaction list hash
40. Test signature payload hash is in correct position (after balance, before equivalentRegistryAddress)

## Definition of Done

- [ ] All AuditMessage tests implemented and passing
- [ ] All AuditResponseMessage tests implemented and passing
- [ ] All sorting tests implemented and passing
- [ ] All hash tests implemented and passing
- [ ] All SQLite receipt handler tests implemented and passing
- [ ] All PostgreSQL receipt handler integration tests implemented and passing
- [ ] All TrustLinesManager tests implemented and passing
- [ ] Audit signature payload tests implemented and passing
- [ ] Tests follow existing test patterns in codebase
- [ ] Tests are included in CMake build
- [ ] All tests pass

# Implementation Plan

## Step 1: Analyze Existing Test Infrastructure

1. Identify existing test directories and patterns
2. Understand test framework used (likely Google Test or similar)
3. Find existing message serialization tests as reference
4. Find existing handler tests as reference
5. Understand test database setup for SQLite/PostgreSQL tests

## Step 2: Create Test File Structure

1. Create or identify test file for message tests
2. Create or identify test file for receipt handler tests
3. Create or identify test file for TrustLinesManager tests
4. Update CMakeLists.txt if necessary

## Step 3: Implement AuditMessage Tests

```cpp
TEST(AuditMessageTest, SerializeEmptyTransactionList) {
    vector<TransactionUUID> emptyList;
    AuditMessage msg(/* params */, emptyList);
    auto serialized = msg.serializeToBytes();
    // Verify serialization
}

TEST(AuditMessageTest, SerializeMultipleTransactions) {
    vector<TransactionUUID> list = {uuid1, uuid2, uuid3};
    AuditMessage msg(/* params */, list);
    auto serialized = msg.serializeToBytes();
    // Deserialize and verify
}

TEST(AuditMessageTest, RoundTrip) {
    // Create, serialize, deserialize, compare
}
```

## Step 4: Implement AuditResponseMessage Tests

Similar pattern to AuditMessage tests, focusing on optional transaction list.

## Step 5: Implement Sorting Tests

```cpp
TEST(TransactionListSortTest, LexicographicOrder) {
    vector<TransactionUUID> list = {uuid_b, uuid_a, uuid_c};
    sortTransactionList(list);
    EXPECT_EQ(list[0], uuid_a);
    EXPECT_EQ(list[1], uuid_b);
    EXPECT_EQ(list[2], uuid_c);
}

TEST(TransactionListSortTest, MatchesCompareTransactionUUID) {
    // Verify sorting produces same order as compareTransactionUUID
}
```

## Step 6: Implement Hash Tests

```cpp
TEST(TransactionListHashTest, EmptyList) {
    vector<TransactionUUID> empty;
    auto hash = computeTransactionListHash(empty);
    EXPECT_EQ(hash.size(), 32); // SHA-256
}

TEST(TransactionListHashTest, Deterministic) {
    vector<TransactionUUID> list = {uuid1, uuid2};
    auto hash1 = computeTransactionListHash(list);
    auto hash2 = computeTransactionListHash(list);
    EXPECT_EQ(hash1, hash2);
}

TEST(TransactionListHashTest, DifferentContentDifferentHash) {
    auto hash1 = computeTransactionListHash({uuid1});
    auto hash2 = computeTransactionListHash({uuid2});
    EXPECT_NE(hash1, hash2);
}
```

## Step 7: Implement Receipt Handler Tests

```cpp
class ReceiptHandlerTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Set up test database
        // Insert test receipts and transactions
    }
    void TearDown() override {
        // Clean up
    }
};

TEST_F(ReceiptHandlerTest, GetFinalizedReceiptsOnlyZeroAuditNumber) {
    // Insert receipts with different audit numbers
    auto results = handler->getFinalizedReceiptsWithZeroAuditNumber(trustLineID);
    // Verify only auditNumber=0 returned
}

TEST_F(ReceiptHandlerTest, GetFinalizedReceiptsOnlyCommitted) {
    // Insert receipts with transactions in different states
    auto results = handler->getFinalizedReceiptsWithZeroAuditNumber(trustLineID);
    // Verify only committed transactions returned
}
```

## Step 8: Implement TrustLinesManager Tests

```cpp
TEST(TrustLinesManagerTest, UpdateReceiptsAmountsCalculatesBalance) {
    // Initial balance = 2000
    // includedIncoming = 1000, includedOutgoing = 500
    // Expected new balance = 2000 + 1000 - 500 = 2500
    manager->updateTrustLineTotalReceiptsAmounts(
        contractorID, 1000, 500, 0, 0);
    EXPECT_EQ(trustLine->balance(), 2500);
}

TEST(TrustLinesManagerTest, PreservesExcludedAmounts) {
    manager->updateTrustLineTotalReceiptsAmounts(
        contractorID, 0, 500, 1000, 300);
    EXPECT_EQ(trustLine->totalIncomingReceiptsAmount(), 1000);
    EXPECT_EQ(trustLine->totalOutgoingReceiptsAmount(), 300);
}
```

## Step 9: Implement Signature Payload Tests

```cpp
TEST(AuditSignatureTest, IncludesTransactionListHash) {
    vector<TransactionUUID> list = {uuid1, uuid2};
    auto payload = getSerializedAuditData(/* params */, list);
    auto expectedHash = computeTransactionListHash(list);
    // Verify hash is present in payload at correct position
}
```

## Step 10: Verify All Tests Pass

1. Run full test suite
2. Fix any failures
3. Ensure test coverage is adequate

## Files to Create/Modify

| File | Changes |
|------|---------|
| `test/core/network/messages/AuditMessageTest.cpp` | Create/update with new tests |
| `test/core/network/messages/AuditResponseMessageTest.cpp` | Create/update with new tests |
| `test/core/io/storage/ReceiptHandlerTest.cpp` | Create/update with new tests |
| `test/core/trust_lines/TrustLinesManagerTest.cpp` | Create/update with new tests |
| `test/CMakeLists.txt` | Update to include new test files |

# Test Plan

**Complexity Level:** Moderate

This task IS the test implementation, so the test plan describes test organization:

## Test Organization
- Group tests by component (messages, handlers, manager)
- Use descriptive test names
- Include both positive and negative test cases
- Use appropriate fixtures for database tests

## Test Coverage Goals
- All new public methods have at least one test
- Edge cases (empty lists, single elements) are covered
- Error conditions are tested where applicable

## Test Execution
- All tests must pass before task completion
- Tests must be integrated into CI pipeline (CMake)

# Verification and Validation

## Architecture integrity
- Tests follow existing test patterns in codebase
- Test organization matches source code organization
- No production code modified in this task

## Security
- N/A (testing task)

## Performance
- Tests should execute quickly
- Database tests use appropriate isolation

## Scalability
- N/A (testing task)

## Reliability
- Tests are deterministic (no flaky tests)
- Tests clean up after themselves

## Maintainability
- Clear test names describe what is being tested
- Tests are independent of each other
- Shared setup is in fixtures

## Cost
- N/A (no infrastructure changes)

## Compliance
- N/A (testing task)

# Restrictions
- Commit changes only after all tests pass
- Do not modify production code in this task
- Follow existing test conventions in the codebase
- Ensure tests are properly integrated into build system
