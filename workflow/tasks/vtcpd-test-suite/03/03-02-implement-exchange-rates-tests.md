# 03-02 - Implement Exchange Rates Tests

# Links
- [PRD](../../prd/vtcpd-test-suite/03-exchange-rates-testing.md)
- [Previous task](03-01-implement-exchange-rates-http-methods.md)

# Description
Implement comprehensive test suite for exchange rates functionality covering validation, conversion logic, edge cases, CRUD operations, and TTL expiration. Create 3 test functions in tests/exchange_rates/exchange_rates_single_node_test.go that validate all 14 test scenarios defined in the PRD (11 original + 3 additional edge cases).

# Requirements and DOD
- Create new directory: `tests/exchange_rates/`
- Create test file: `tests/exchange_rates/exchange_rates_single_node_test.go`
- Implement 3 test functions:
  - `TestExchangeRatesValidationAndConversion` - covers test cases 1-12 (validation errors, conversion logic, and edge cases)
  - `TestExchangeRatesCRUD` - covers test case 13 (CRUD operations)
  - `TestExchangeRatesTTL` - covers test case 14 (TTL expiration after 5.5 minutes)
- All tests must use single node setup following existing test patterns
- Test cases must validate specific scenarios:
  - **Case 1**: Setting rate without specifying rate value → HTTP 400
  - **Case 2**: Setting rate with both real_rate and native (value+shift) → HTTP 400
  - **Case 3**: Setting real_rate with >16 decimal places → HTTP 400
  - **Case 4**: Native input (1001→2002, value=11207154, shift=4) → real_rate=112071.54
  - **Case 5**: Real input (1001→2002, real_rate=112071.54) → value=11207154, shift=4
  - **Case 6**: Native input (1001→2002, value=112071, shift=2) → real_rate=112071
  - **Case 7**: Real input (1001→2002, real_rate=112071) → value=112071, shift=2
  - **Case 8**: Native input (2002→1001, value=89228719, shift=11) → real_rate=0.0000089228719
  - **Case 9**: Real input (2002→1001, real_rate=0.0000089228719) → value=89228719, shift=11
  - **Case 10**: Getting non-existent exchange rate → appropriate error/empty response
  - **Case 11**: Setting rate with maximum int16 shift value (32767) → correct conversion
  - **Case 12**: Setting rate with minimum int16 shift value (-32768) → correct conversion
  - **Case 13**: Multiple rates CRUD operations with selective deletion
  - **Case 14**: Rate expiration after 5.5 minutes
- Tests must use existing test infrastructure patterns:
  - Node creation and container management
  - Test setup and cleanup
  - Error handling via test failures
- All test constants must be defined at package level
- Tests must be deterministic and repeatable (except TTL test timing)
- Follow existing naming conventions and code style

# Implementation Plan
1. **Create test directory structure**:
   - Create `tests/exchange_rates/` directory
   - Create `exchange_rates_single_node_test.go` file
   - Add package declaration and necessary imports

2. **Define test constants**:
   ```go
   const (
       // Test equivalents from decimals map
       EQUIVALENT_101  = "101"   // 2 decimals
       EQUIVALENT_1001 = "1001"  // 8 decimals  
       EQUIVALENT_1002 = "1002"  // 8 decimals
       EQUIVALENT_2002 = "2002"  // 6 decimals
       
       // Test values for conversion validation
       TEST_VALUE_11207154 = "11207154"
       TEST_SHIFT_4        = int16(4)
       TEST_REAL_RATE_112071_54 = "112071.54"
       TEST_VALUE_112071 = "112071"
       TEST_SHIFT_2 = int16(2)
       TEST_REAL_RATE_112071 = "112071"
       TEST_VALUE_89228719 = "89228719"
       TEST_SHIFT_11 = int16(11)
       TEST_REAL_RATE_SMALL = "0.0000089228719"
       
       // Edge case constants
       UNKNOWN_EQUIVALENT_1 = "999"  // Not in DecimalsMap
       UNKNOWN_EQUIVALENT_2 = "1111" // Not in DecimalsMap
       MAX_INT16_SHIFT = int16(32767)  // Maximum int16 value
       MIN_INT16_SHIFT = int16(-32768) // Minimum int16 value
       
       // TTL test constants
       TTL_WAIT_DURATION = 5*time.Minute + 30*time.Second
   )
   ```

3. **Implement TestExchangeRatesValidationAndConversion function**:
   - Set up single node using existing patterns
   - **Validation Tests (Cases 1-3, 12)**:
     - Test Case 1: Call SetExchangeRate with empty realRate, expect test failure (HTTP 400)
     - Test Case 2: Call SetExchangeRate AND SetExchangeRateNative for same pair, expect validation error
     - Test Case 3: Call SetExchangeRate with realRate having >16 decimal places, expect HTTP 400
     - Test Case 12: Call SetExchangeRate with unknown equivalents (UNKNOWN_EQUIVALENT_1, UNKNOWN_EQUIVALENT_2), expect HTTP 400
   - **Conversion Tests (Cases 4-9)**:
     - Set rate using native format, get rate, verify real_rate field
     - Set rate using real format, get rate, verify value and shift fields
     - Test both high precision and integer scenarios
     - Test small decimal scenarios with large shift values
   - **Edge Case Tests (Cases 10-11)**:
     - Test Case 10: Call GetExchangeRate for non-existent pair, verify appropriate error/empty response
     - Test Case 11: Set rate with MAX_INT16_SHIFT, verify correct conversion and retrieval
     - Test Case 12: Set rate with MIN_INT16_SHIFT, verify correct conversion and retrieval
   - Clean up: Clear all rates after tests

4. **Implement TestExchangeRatesCRUD function**:
   - Set up single node
   - Set multiple rates for different equivalent pairs (at least 3 different pairs)
   - Call ListExchangeRates, verify all rates are present and count matches
   - Call DeleteExchangeRate for one specific pair
   - Call ListExchangeRates again, verify deleted rate is gone and others remain
   - Call ClearExchangeRates
   - Call ListExchangeRates, verify no rates remain (count = 0)

5. **Implement TestExchangeRatesTTL function**:
   - Set up single node
   - Set one exchange rate using any valid format
   - Call GetExchangeRate immediately, verify rate is accessible
   - Wait for TTL_WAIT_DURATION (5 minutes 30 seconds)
   - Call GetExchangeRate again, expect failure or empty result (rate expired)
   - Note: This test will take ~6 minutes to complete

6. **Add helper functions if needed**:
   - Node setup function following existing patterns
   - Rate comparison helpers for validation
   - Error handling helpers for expected failures

7. **Implement proper test setup and cleanup**:
   - Use existing Docker container management patterns
   - Ensure proper node initialization and readiness checks
   - Clean up containers after each test
   - Handle test failures gracefully

8. **Add comprehensive error handling**:
   - Use t.Fatalf() for unexpected errors
   - Use t.Errorf() for test assertions
   - Provide descriptive error messages for debugging
   - Handle expected errors (validation cases) appropriately

9. **Validate test execution**:
   - Ensure all tests can run independently
   - Verify test execution times are within expected limits
   - Test against live vtcpd instances with exchange rates functionality

# Test Plan
## Objective
Verify comprehensive test coverage for exchange rates functionality including validation, conversion, CRUD operations, and TTL expiration.

## Test Scope
- All 14 test scenarios defined in PRD (11 original + 3 additional edge cases)
- Integration with vtcpd and vtcpd-cli exchange rates functionality
- Error handling and validation scenarios
- Edge case and boundary condition testing
- Timing-based TTL functionality

## Test Environment
- Single vtcpd node with exchange rates functionality
- vtcpd-cli with exchange rates HTTP endpoints
- Docker containerization for isolation
- Go test framework

## Mocking Strategy
- No mocking required - integration tests against real services
- Use Docker containers for service isolation
- Real HTTP communication with vtcpd-cli endpoints

## Key Test Scenarios
### Validation Tests (Cases 1-3, 12)
1. **Missing Parameters**: Verify HTTP 400 when no rate parameters provided
2. **Conflicting Parameters**: Verify HTTP 400 when both real_rate and native parameters provided
3. **Invalid Precision**: Verify HTTP 400 when real_rate has >16 decimal places
12. **Unknown Equivalents**: Verify HTTP 400 when using equivalents not in DecimalsMap

### Conversion Tests (Cases 4-9)
4. **Native to Real (High Precision)**: value=11207154, shift=4 produces real_rate=112071.54
5. **Real to Native (High Precision)**: real_rate=112071.54 produces value=11207154, shift=4
6. **Native to Real (Integer)**: value=112071, shift=2 produces real_rate=112071
7. **Real to Native (Integer)**: real_rate=112071 produces value=112071, shift=2
8. **Native to Real (Small)**: value=89228719, shift=11 produces real_rate=0.0000089228719
9. **Real to Native (Small)**: real_rate=0.0000089228719 produces value=89228719, shift=11

### Edge Case Tests (Cases 10-12)
10. **Non-existent Rate**: GET for non-existent rate pair returns appropriate error/empty response
11. **Maximum Shift Boundary**: Native input with shift=32767 works correctly
12. **Minimum Shift Boundary**: Native input with shift=-32768 works correctly

### CRUD Test (Case 13)
- Multiple rate creation and verification
- Selective deletion with verification
- Complete clearing with verification

### TTL Test (Case 14)
- Rate accessibility immediately after creation
- Automatic expiration after 5.5 minutes

## Success Criteria
- All test functions execute successfully
- Validation scenarios properly reject invalid inputs
- Conversion logic produces expected results for all test cases
- CRUD operations work correctly with proper state management
- TTL expiration works as expected
- Test execution times are within acceptable limits
- No flaky behavior except for intentional TTL timing

# Verification and Validation

## Architecture integrity
- Tests follow existing test structure and patterns
- Integration with existing Docker container management
- Proper use of Node struct methods for HTTP communication
- No architectural changes to existing test infrastructure

## Security
- Tests run in isolated Docker containers
- No external network access beyond established test patterns
- Use of existing security configurations and practices
- No sensitive data exposure in test scenarios

## Performance
- Validation and conversion tests complete in <30 seconds
- CRUD operations test completes in <60 seconds
- TTL test completes in ~6 minutes (expected due to wait time)
- Memory usage within existing test suite limits
- No performance regressions in test infrastructure

## Scalability
- Tests can run in parallel where appropriate
- Resource usage is bounded and predictable
- Test execution scales with existing infrastructure
- No unbounded resource consumption

## Reliability
- Tests are deterministic and repeatable
- Proper cleanup between test cases
- Robust error handling for network issues
- TTL test handles timing appropriately
- No flaky behavior except intentional timing test

## Maintainability
- Clear test structure and naming conventions
- Comprehensive error messages for debugging
- Easy to extend for future exchange rates features
- Well-documented test scenarios and expectations
- Follows existing code style and patterns

## Cost
- Test execution time is reasonable for comprehensive coverage
- Resource usage is within acceptable limits for CI/CD
- No additional infrastructure requirements
- Development effort is proportional to test coverage provided

## Compliance
- Follows Go testing best practices
- Adheres to existing project conventions
- Compatible with CI/CD pipeline requirements
- Meets test coverage requirements for exchange rates functionality

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify existing test infrastructure or patterns
- Do not change Docker container configurations
- Follow existing test naming and organization conventions
- Ensure tests can run independently and don't interfere with each other
- Handle TTL test timing appropriately - do not make assumptions about exact timing
