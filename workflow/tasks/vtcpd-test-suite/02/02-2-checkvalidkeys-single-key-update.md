# 02-2 - CheckValidKeys Methods Single-Key Architecture Update

# Links
- [PRD-02: SPHINCS+ Compatibility Update](../../prd/vtcpd-test-suite/02-sphincs-plus-compatibility.md)

# Description
Update CheckValidKeys methods and all related test calls to work with SPHINCS+ single-key architecture. Remove the expectedOwnValidKeysCount and expectedContractorValidKeysCount parameters and modify the methods to simply verify that at least one key exists per trust line, without checking the is_valid field that no longer exists in the SPHINCS+ schema.

# Requirements and DOD

## Requirements
1. **CheckValidKeysSQLite Method Update**:
   - Remove `expectedOwnValidKeysCount` and `expectedContractorValidKeysCount` parameters
   - Update SQL query to remove `WHERE is_valid = 1` condition
   - Change query to `SELECT COUNT(*) FROM own_keys` and `SELECT COUNT(*) FROM contractor_keys`
   - Verify that at least one key exists (count >= 1) for each table
   - Update error messages to reflect single-key validation context

2. **CheckValidKeysPostgreSQL Method Update**:
   - Remove `expectedOwnValidKeysCount` and `expectedContractorValidKeysCount` parameters
   - Update SQL query to remove `WHERE is_valid = true` condition
   - Change query to `SELECT COUNT(*) FROM own_keys` and `SELECT COUNT(*) FROM contractor_keys`
   - Verify that at least one key exists (count >= 1) for each table
   - Update error messages to reflect single-key validation context

3. **CheckValidKeys Dispatcher Method Update**:
   - Remove `expectedOwnValidKeysCount` and `expectedContractorValidKeysCount` parameters
   - Update method signature to `CheckValidKeys(t *testing.T)`
   - Update calls to SQLite and PostgreSQL specific methods to pass no count parameters
   - Maintain database type detection logic

4. **Test Files Updates**:
   - Update all calls to `CheckValidKeys` methods throughout the test suite
   - Remove all count parameters from method calls
   - Update test expectations to focus on key presence rather than specific counts
   - Remove dependencies on `DefaultKeysCount` constant for key validation

5. **Error Message Updates**:
   - Update all error messages to reflect single-key validation
   - Remove references to expected vs actual key counts
   - Focus messages on key presence/absence rather than count mismatches
   - Maintain clear debugging information for test failures

## Definition of Done
- [ ] CheckValidKeysSQLite method updated to single-key validation
- [ ] CheckValidKeysPostgreSQL method updated to single-key validation
- [ ] CheckValidKeys dispatcher method updated with new signature
- [ ] All test files updated to use new method signature
- [ ] All database queries updated to remove is_valid field references
- [ ] Error messages updated for single-key context
- [ ] All existing tests pass with updated methods
- [ ] No references to old multi-key counting logic remain
- [ ] Code passes linting and quality checks

# Implementation Plan

## Step 1: Analyze Current Usage
- Identify all files that call CheckValidKeys methods
- Document current parameter usage patterns
- List all test files that need updates
- Identify any complex key counting logic that needs simplification

## Step 2: Update CheckValidKeysSQLite Method
- Modify method signature to remove count parameters in `pkg/testsuite/node.go`
- Update SQL queries to remove `WHERE is_valid = 1` condition
- Change validation logic to check for `count >= 1` instead of exact matches
- Update error messages for single-key context
- Add comments explaining the single-key validation approach

## Step 3: Update CheckValidKeysPostgreSQL Method
- Modify method signature to remove count parameters in `pkg/testsuite/node.go`
- Update SQL queries to remove `WHERE is_valid = true` condition
- Change validation logic to check for `count >= 1` instead of exact matches
- Update error messages for single-key context
- Ensure consistency with SQLite method implementation

## Step 4: Update CheckValidKeys Dispatcher Method
- Modify method signature to remove count parameters in `pkg/testsuite/node.go`
- Update calls to database-specific methods to pass no parameters
- Maintain existing database type detection logic
- Ensure proper error handling and logging

## Step 5: Update All Test Files
- Search for all calls to `CheckValidKeys` methods in test files
- Remove count parameters from all method calls
- Update test logic to focus on key presence validation
- Remove complex arithmetic based on `DefaultKeysCount`
- Simplify test assertions and expectations

## Step 6: Update Error Messages and Logging
- Review all error messages in the updated methods
- Ensure messages are clear and helpful for debugging
- Remove references to expected vs actual counts
- Add context about single-key validation approach
- Maintain consistency across SQLite and PostgreSQL variants

## Step 7: Testing and Validation
- Run all affected tests to ensure they pass
- Verify both SQLite and PostgreSQL scenarios work correctly
- Test edge cases (no keys, multiple keys, database errors)
- Validate error messages are helpful and accurate

# Test Plan

## Unit Testing
- **Method Signature Tests**: Verify new method signatures compile correctly
- **Database Query Tests**: Confirm SQL queries work without is_valid field
- **Key Presence Tests**: Validate methods correctly detect key presence/absence
- **Error Handling Tests**: Verify proper error handling for database issues

## Integration Testing
- **SQLite Integration**: Test CheckValidKeysSQLite with actual SQLite database
- **PostgreSQL Integration**: Test CheckValidKeysPostgreSQL with actual PostgreSQL database
- **Multi-Database Tests**: Verify dispatcher method works with both database types
- **Test Suite Integration**: Run existing test scenarios with updated methods

## Regression Testing
- **Settlement Line Tests**: All settlement line tests must pass with updated validation
- **Payment Tests**: All payment tests must pass with updated validation
- **Channel Tests**: All channel tests must pass with updated validation
- **Key Sharing Tests**: All key sharing tests must pass with updated validation

## Edge Case Testing
- **No Keys Scenario**: Test behavior when no keys exist in database
- **Multiple Keys Scenario**: Test behavior when multiple keys exist (should still pass)
- **Database Connection Issues**: Test proper error handling for database problems
- **Invalid Database Schema**: Test graceful handling of schema issues

## Test Environment Setup
```go
// Example of updated test call
func TestSomeFeature(t *testing.T) {
    // Old call (to be updated):
    // nodeA.CheckValidKeys(t, vtcp.DefaultKeysCount-1, vtcp.DefaultKeysCount-1)
    
    // New call:
    nodeA.CheckValidKeys(t)
}
```

## Files to Update
Based on grep search results, the following files need updates:
- `tests/settlement_lines/open_settlement_line_test.go`
- `tests/settlement_lines/settlement_line_keys_sharing_init_test.go`
- `tests/settlement_lines/settlement_line_archived_test.go`
- `tests/settlement_lines/settlement_line_keys_sharing_next_test.go`
- `tests/settlement_lines/open_settlement_line_bad_internet_test.go`
- `tests/settlement_lines/set_settlement_line_bad_internet_test.go`
- And all other test files that call CheckValidKeys methods

## Success Criteria
- All CheckValidKeys method calls use new signature (no parameters)
- All database queries work without is_valid field references
- All existing tests pass with updated validation logic
- Error messages are clear and helpful for debugging
- No performance regression in test execution

# Verification and Validation

## Architecture integrity
- **Single Responsibility**: Methods focus solely on key presence validation
- **Database Abstraction**: Maintain clean separation between SQLite and PostgreSQL implementations
- **Error Handling**: Consistent error handling patterns across database types
- **Code Consistency**: Follow existing code patterns and conventions in the test suite

## Security
- **SQL Injection Prevention**: Ensure database queries are safe and parameterized where needed
- **Error Information Disclosure**: Avoid exposing sensitive database information in error messages
- **Database Connection Security**: Maintain existing secure database connection patterns
- **Test Data Isolation**: Ensure test data doesn't leak between test scenarios

## Performance
- **Query Optimization**: Simplified queries should be faster than previous multi-condition queries
- **Test Execution Speed**: No significant regression in test execution time
- **Database Load**: Reduced database load due to simpler queries
- **Memory Usage**: Maintain reasonable memory usage during test execution

## Scalability
- **Multi-Container Testing**: Updated methods must work in multi-container test scenarios
- **Concurrent Testing**: Support concurrent test execution without conflicts
- **Database Scaling**: Methods should work efficiently with varying database sizes
- **Test Suite Growth**: Architecture supports adding new tests easily

## Reliability
- **Consistent Results**: Methods produce consistent results across multiple runs
- **Error Recovery**: Proper error handling and recovery from database issues
- **Database Compatibility**: Work reliably with both SQLite and PostgreSQL
- **Test Stability**: Reduce test flakiness from complex key counting logic

## Maintainability
- **Code Simplicity**: Simplified logic is easier to understand and maintain
- **Clear Documentation**: Well-documented methods and parameters
- **Consistent Patterns**: Follow established patterns in the codebase
- **Future Changes**: Architecture supports future cryptographic changes easily

## Cost
- **Development Time**: Estimated 2-3 days for implementation and testing
- **Maintenance Overhead**: Reduced maintenance due to simplified logic
- **Testing Resources**: No significant increase in testing resource requirements
- **Documentation Updates**: Minimal documentation updates required

## Compliance
- **Code Quality Standards**: Meet existing code quality and linting standards
- **Testing Standards**: Follow established testing patterns and conventions
- **Documentation Standards**: Maintain documentation quality and completeness
- **Security Standards**: Follow secure coding practices for database operations