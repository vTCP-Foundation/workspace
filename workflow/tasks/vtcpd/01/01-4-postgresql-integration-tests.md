# 01-4 - PostgreSQL Integration Tests

# Links
- [PRD](../../../prd/vtcpd/01-database-provider-abstraction.md)

# Description
Implement comprehensive integration tests for all PostgreSQL handler classes using real database connections to validate functionality with actual PostgreSQL database operations.

# Requirements and DOD
- Create integration test infrastructure in `src/tests/storage/integration/` directory
- Implement integration tests for all 15 PostgreSQL handler classes:
  - AddressHandlerPostgreSQL
  - AuditHandlerPostgreSQL
  - AuditRulesHandlerPostgreSQL
  - ContractorKeysHandlerPostgreSQL
  - ContractorsHandlerPostgreSQL
  - FeaturesHandlerPostgreSQL
  - IncomingPaymentReceiptHandlerPostgreSQL
  - OutgoingPaymentReceiptHandlerPostgreSQL
  - OwnKeysHandlerPostgreSQL
  - PaymentKeysHandlerPostgreSQL
  - PaymentParticipantsVotesHandlerPostgreSQL
  - PaymentTransactionsHandlerPostgreSQL
  - TransactionsHandlerPostgreSQL
  - TrustLineHandlerPostgreSQL
  - CommunicatorMessagesQueueHandlerPostgreSQL
- Test every public method of each handler class
- Use real PostgreSQL database connections (no mocks)
- Hardcode database credentials in test code for simplicity
- Implement test fixtures and helper classes for database setup/teardown
- Ensure test isolation (each test cleans up its data)
- Achieve 100% method coverage for all specified PostgreSQL handler classes
- All integration tests must pass
- Integration tests execution time ≤ 5 minutes in CI environment

# Implementation Plan
1. **Test Infrastructure Setup**:
   - Create `src/tests/storage/integration/` directory structure
   - Implement `DatabaseTestHelper` class for connection management and cleanup
   - Create `PostgreSQLTestFixtures` class for test data generation
   - Update CMakeLists.txt to include integration tests

2. **PostgreSQL Handler Integration Tests**:
   - Implement integration test class for each of the 15 PostgreSQL handler classes
   - Each test class should test all public methods of the corresponding handler
   - Use Google Test framework with SetUp/TearDown methods for database preparation
   - Hardcode connection parameters: host="localhost", port=5432, user="vtcpd_test", password="test_password"

3. **Test Coverage and Validation**:
   - Verify table creation and schema correctness
   - Test data persistence (insert, update, delete, select operations)
   - Test error handling for database connection failures and constraint violations
   - Test transaction consistency (rollback and commit behavior)
   - Ensure test data isolation between different test cases

4. **Test Environment Prerequisites**:
   - Document PostgreSQL server requirements
   - Document test database creation requirements ("storageDB" and "communicatorStorageDB")
   - Provide setup instructions for test environment

# Test Plan
- **Integration Tests**: Test all public methods of each PostgreSQL handler class with real database operations
- **Database Operations**: Verify correct insert, update, delete, select operations
- **Error Handling**: Test database connection failures and constraint violations
- **Transaction Testing**: Verify transaction rollback and commit behavior
- **Test Isolation**: Ensure tests don't interfere with each other through proper cleanup
- **Performance**: Verify integration tests complete within 5 minutes

# Verification and Validation
User approval that validation criteria appropriate to task complexity have been met

## Architecture integrity
- Integration tests follow established testing patterns and directory structure
- Test code integrates properly with existing Google Test framework
- Clear separation between unit tests and integration tests maintained

## Security
- Database credentials are properly handled in test environment
- Test databases are isolated from production data
- No sensitive data exposure in test code

## Performance
- Integration tests complete within 5 minutes in CI environment
- Database connections are properly managed to avoid resource leaks
- Test cleanup is efficient and doesn't leave orphaned data

## Scalability
- Test infrastructure can easily accommodate additional PostgreSQL handler classes
- Test fixtures are reusable across different handler tests
- Database helper classes support multiple test scenarios

## Reliability
- All integration tests pass consistently
- Test data isolation prevents test interference
- Database connection handling is robust and handles failures gracefully

## Maintainability
- Test code is well-structured and follows consistent patterns
- Test fixtures and helpers are reusable and maintainable
- Clear documentation for test environment setup and requirements

## Cost
- Integration tests provide value by validating real database operations
- Test execution time is reasonable for CI environment
- Resource usage is appropriate for test environment

## Compliance
- Tests follow project testing standards and conventions
- Integration tests complement unit tests without duplication
- All PostgreSQL handler classes are covered as specified

# Restrictions
- make commits only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task) 