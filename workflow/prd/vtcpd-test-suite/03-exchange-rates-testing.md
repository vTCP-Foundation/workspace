## [03] Exchange Rates Testing Implementation

## Document Information
- **Project Name**: vtcpd-test-suite
- **PRD ID**: 03
- **Phase/Iteration**: PRD 03
- **Document Version**: 1.0
- **Date**: 2025-08-25
- **Author(s)**: Mykola Ilashchuk, Dima Chizhevsky
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-08-25
- **Previous PRD**: None (first PRD for vtcpd-test-suite)
- **Related Documents**: 
  - vtcpd PRD: workspace/workflow/prd/vtcpd/03-exchange-rates-manager.md
  - vtcpd-cli PRD: workspace/workflow/prd/vtcpd-cli/03-exchange-rates-cli-and-http.md
  - Policy: workspace/policy.md

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This iteration implements comprehensive testing for the exchange rates functionality that was added to vtcpd and vtcpd-cli projects. The testing suite will validate HTTP endpoints, validation rules, conversion logic between real decimal and native (value + shift) formats, TTL expiration, and CRUD operations for exchange rates. The implementation includes extending the existing Node struct with exchange rates methods and creating dedicated test cases covering both positive and negative scenarios.

## Iteration Context
### Previous Iterations Summary
- **vtcpd**: Implemented ExchangeRatesManager with TTL-based expiry, commands (SET:RATE, GET:RATE, LIST:RATES, DEL:RATE, CLEAR:RATES), and transactions in branch prd/03-exchange-rates-manager
- **vtcpd-cli**: Added HTTP endpoints and CLI commands for exchange rates management with validation and conversion logic in branch prd/03-exchange-rates-cli-and-http
- **vtcpd-test-suite**: Existing test infrastructure with Docker containerization, Node struct for HTTP operations, and established testing patterns

### Current State Analysis
- **What's working well**: Established testing infrastructure, consistent patterns for node operations, Docker-based test environments
- **Pain points identified**: No test coverage for exchange rates functionality despite implementation being complete
- **Performance metrics**: Current test suite covers channels, settlement lines, and payments but lacks exchange rates validation

## Problem Statement
### Background
The vtcpd project has implemented comprehensive exchange rates functionality including in-memory storage with TTL expiration, command/transaction interfaces, and HTTP/CLI access through vtcpd-cli. However, the test suite lacks coverage for this critical functionality, creating a gap in validation of the exchange rates implementation.

### Problem Description
- No automated tests exist to validate exchange rates HTTP endpoints functionality
- Missing validation of conversion logic between real decimal and native (value, shift) formats
- No tests for validation rules and error handling scenarios
- TTL expiration behavior is not tested
- CRUD operations for exchange rates lack test coverage
- Integration between vtcpd-cli and vtcpd for exchange rates is untested

### Success Metrics
- **Primary KPIs**: 
  - 100% test coverage for all exchange rates HTTP endpoints
  - All validation scenarios tested with expected error responses
  - Conversion accuracy validated for all test cases
  - TTL expiration behavior verified
- **Secondary KPIs**:
  - Test execution time < 6 minutes for TTL test
  - All tests pass consistently across different environments
- **Target Values**: 
  - 14 test scenarios implemented and passing (11 original + 3 additional edge cases)
  - 0 critical bugs found in exchange rates functionality post-testing

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- **Node.go HTTP Methods**: Implement 6 new methods in pkg/testsuite/node.go:
  - SetExchangeRate() - for real decimal rate setting
  - SetExchangeRateNative() - for native value+shift rate setting  
  - GetExchangeRate() - for retrieving specific exchange rate
  - ListExchangeRates() - for listing all exchange rates
  - DeleteExchangeRate() - for deleting specific exchange rate
  - ClearExchangeRates() - for clearing all exchange rates

- **Test Implementation**: Create comprehensive test suite in tests/exchange_rates/:
  - exchange_rates_single_node_test.go with 3 test functions
  - Validation tests (cases 1-12) in single test function including edge cases
  - CRUD operations test (case 13) in dedicated test function  
  - TTL expiration test (case 14) in dedicated test function

- **Test Infrastructure**: Extend existing patterns for exchange rates testing

#### Bug Fixes & Technical Improvements
- Ensure consistent error handling patterns across all new methods
- Validate HTTP status codes and response formats match existing patterns

#### Modifications to Existing Features
- Extend Node struct with exchange rates methods following existing patterns
- Add new test directory structure under tests/

### Explicitly Out of Scope
- Changes to vtcpd or vtcpd-cli exchange rates implementation
- Performance optimization of existing test infrastructure
- Testing of exchange rates persistence (rates are ephemeral by design)
- CLI command testing (only HTTP endpoints testing)

### Dependencies from Previous Iterations
- vtcpd exchange rates implementation (branch prd/03-exchange-rates-manager)
- vtcpd-cli HTTP endpoints implementation (branch prd/03-exchange-rates-cli-and-http)
- Existing test infrastructure and Docker containerization
- Node struct patterns and HTTP client functionality

### Future Roadmap Impact
- Provides foundation for testing future exchange operations and flows
- Establishes patterns for testing complex validation and conversion logic
- Enables confident deployment of exchange rates functionality

## User Stories & Requirements

### User Personas
#### Primary User: QA Engineer/Developer
- **Role**: Quality Assurance Engineer or Developer
- **Goals**: Validate exchange rates functionality works correctly across all scenarios
- **Pain Points**: Manual testing of conversion logic and validation rules is time-consuming and error-prone
- **Technical Proficiency**: Advanced - familiar with Go testing patterns and HTTP APIs

#### Secondary User: DevOps/CI Pipeline
- **Role**: Automated testing system
- **Goals**: Catch regressions in exchange rates functionality during CI/CD
- **Pain Points**: Need reliable, fast-running tests that don't require manual intervention
- **Technical Proficiency**: Advanced - automated execution environment

### Functional Requirements
#### New Features for This Iteration
1. **Exchange Rates HTTP Methods in Node struct**
   - **Description**: Add 6 new methods to pkg/testsuite/node.go for exchange rates operations
   - **User Story**: As a test developer, I want HTTP helper methods so that I can easily test exchange rates functionality
   - **Rationale**: Consistent with existing Node patterns for other operations
   - **Builds Upon**: Existing Node HTTP methods pattern
   - **Acceptance Criteria**: 
     - All methods follow existing Node method patterns (accept t *testing.T, handle errors internally)
     - Methods return appropriate Go structs (RateItem, RatesListResponse, etc.)
     - HTTP status codes are validated within methods
     - Failed requests cause test failure via t.Fatalf()
   - **Priority**: High
   - **Dependencies**: vtcpd-cli HTTP endpoints implementation

2. **Validation Error Tests (Cases 1-3)**
   - **Description**: Test validation rules for exchange rate setting
   - **User Story**: As a QA engineer, I want to verify that invalid inputs are properly rejected
   - **Rationale**: Ensures data integrity and proper error handling
   - **Builds Upon**: vtcpd-cli validation logic
   - **Acceptance Criteria**:
     - Case 1: Setting rate without specifying rate value returns HTTP 400
     - Case 2: Setting rate with both real_rate and native (value+shift) returns HTTP 400  
     - Case 3: Setting real_rate with >16 decimal places returns HTTP 400
     - Case 12: Setting rate with unknown equivalents (not in DecimalsMap) returns HTTP 400
   - **Priority**: High
   - **Dependencies**: SetExchangeRate and SetExchangeRateNative methods

3. **Conversion Logic Tests (Cases 4-9)**
   - **Description**: Validate conversion between real decimal and native formats
   - **User Story**: As a developer, I want to ensure conversion logic works correctly for all scenarios
   - **Rationale**: Critical for exchange rate accuracy and platform stability
   - **Builds Upon**: vtcpd-cli conversion implementation
   - **Acceptance Criteria**:
     - Case 4: Native input (1001→2002, value=11207154, shift=4) produces real_rate=112071.54
     - Case 5: Real input (1001→2002, real_rate=112071.54) produces value=11207154, shift=4
     - Case 6: Native input (1001→2002, value=112071, shift=2) produces real_rate=112071
     - Case 7: Real input (1001→2002, real_rate=112071) produces value=112071, shift=2
     - Case 8: Native input (2002→1001, value=89228719, shift=11) produces real_rate=0.0000089228719
     - Case 9: Real input (2002→1001, real_rate=0.0000089228719) produces value=89228719, shift=11
   - **Priority**: High
   - **Dependencies**: GetExchangeRate method

5. **Edge Case Tests (Cases 10-12)**
   - **Description**: Test boundary conditions and edge cases for exchange rates
   - **User Story**: As a QA engineer, I want to verify system behavior at boundary conditions
   - **Rationale**: Ensures robust error handling and prevents system failures at edge cases
   - **Builds Upon**: vtcpd-cli validation and conversion logic
   - **Acceptance Criteria**:
     - Case 10: Getting non-existent exchange rate returns appropriate error/empty response
     - Case 11: Setting rate with maximum int16 shift value (32767) works correctly
     - Case 12: Setting rate with minimum int16 shift value (-32768) works correctly
   - **Priority**: Medium
   - **Dependencies**: All exchange rates methods

6. **CRUD Operations Test (Case 13)**
   - **Description**: Test complete lifecycle of exchange rate management
   - **User Story**: As a system administrator, I want to manage exchange rates through CRUD operations
   - **Rationale**: Validates complete functionality of exchange rates management
   - **Builds Upon**: All exchange rates HTTP methods
   - **Acceptance Criteria**:
     - Multiple rates can be set for different pairs
     - ListExchangeRates returns all set rates
     - DeleteExchangeRate removes specific rate while preserving others
     - ClearExchangeRates removes all rates
     - Operations are atomic and consistent
   - **Priority**: High
   - **Dependencies**: All exchange rates methods

7. **TTL Expiration Test (Case 14)**
   - **Description**: Validate automatic expiration of exchange rates after 5 minutes
   - **User Story**: As a system operator, I want rates to expire automatically to ensure freshness
   - **Rationale**: Critical for ensuring stale rates don't persist in the system
   - **Builds Upon**: vtcpd ExchangeRatesManager TTL implementation
   - **Acceptance Criteria**:
     - Rate is accessible immediately after setting
     - Rate is automatically removed after 5.5 minutes
     - No manual intervention required for expiration
   - **Priority**: Medium
   - **Dependencies**: SetExchangeRate and GetExchangeRate methods

#### Test Structure Requirements
1. **Test Organization**
   - Location: tests/exchange_rates/exchange_rates_single_node_test.go
   - 3 test functions: TestExchangeRatesValidationAndConversion, TestExchangeRatesCRUD, TestExchangeRatesTTL
   - Single node setup for all tests
   - Consistent with existing test patterns

2. **Error Handling**
   - All HTTP methods must handle errors internally
   - Test failures via t.Fatalf() for any HTTP or validation errors
   - Clear error messages indicating specific failure points

### Non-Functional Requirements
#### Performance
- Test execution time: 
  - Validation and conversion tests: < 30 seconds
  - CRUD operations test: < 60 seconds  
  - TTL expiration test: < 6 minutes (due to 5.5 minute wait)
- Memory usage within existing test suite limits

#### Security
- No additional security requirements beyond existing test infrastructure
- Use of existing Docker containerization for isolation

#### Reliability
- Tests must be deterministic and repeatable
- No flaky behavior due to timing issues (except intentional TTL test)
- Proper cleanup between test cases

#### Maintainability
- Follow existing code patterns and conventions
- Clear test names and documentation
- Easy to extend for future exchange rates features

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Existing test suite with Node struct providing HTTP client functionality for vtcpd-cli endpoints
- **Proposed Changes**: Extend Node struct with 6 new methods for exchange rates operations
- **Backwards Compatibility**: No changes to existing functionality
- **Migration Requirements**: None - purely additive changes

### Technology Stack Updates
#### New Technologies/Libraries
- No new external dependencies required
- Utilizes existing Go testing framework, HTTP client, and JSON parsing

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- Integration with vtcpd-cli exchange rates HTTP endpoints:
  - POST /api/v1/node/rates/{equivalent_from}/{equivalent_to}/
  - GET /api/v1/node/rates/{equivalent_from}/{equivalent_to}/
  - GET /api/v1/node/rates/
  - DELETE /api/v1/node/rates/{equivalent_from}/{equivalent_to}/
  - DELETE /api/v1/node/rates/

#### Modified Integrations
- Extension of existing Node struct HTTP client patterns

### Data Requirements
#### Data Models
- **RateItem struct**: Already defined in vtcpd-cli/internal/common/common.go
  ```go
  type RateItem struct {
      EquivalentFrom             string `json:"equivalent_from"`
      EquivalentTo               string `json:"equivalent_to"`
      Value                      string `json:"value"`
      Shift                      int16  `json:"shift"`
      RealRate                   string `json:"real_rate"`
      MinExchangeAmount          string `json:"min_exchange_amount"`
      MaxExchangeAmount          string `json:"max_exchange_amount"`
      ExpiresAtUnixMicroseconds  string `json:"expires_at_unix_microseconds"`
  }
  ```

- **Response structures**: RateGetResponse, RatesListResponse, RatesActionResponse

#### Data Storage
- No persistent storage required (exchange rates are ephemeral)
- Test data generated and validated within test execution

#### Data Migration
- Not applicable - no existing data to migrate

### HTTP Methods Implementation Details
#### Method Signatures and Behavior
```go
// SetExchangeRate sets exchange rate using real decimal format
func (n *Node) SetExchangeRate(t *testing.T, equivalentFrom, equivalentTo, realRate string, minAmount, maxAmount *string) {
    // Validates inputs, sends POST request, handles errors internally
}

// SetExchangeRateNative sets exchange rate using native value+shift format  
func (n *Node) SetExchangeRateNative(t *testing.T, equivalentFrom, equivalentTo, value string, shift int16, minAmount, maxAmount *string) {
    // Validates inputs, sends POST request, handles errors internally
}

// GetExchangeRate retrieves specific exchange rate
func (n *Node) GetExchangeRate(t *testing.T, equivalentFrom, equivalentTo string) *RateItem {
    // Sends GET request, parses response, handles errors internally
}

// ListExchangeRates retrieves all exchange rates
func (n *Node) ListExchangeRates(t *testing.T) *RatesListResponse {
    // Sends GET request, parses response, handles errors internally  
}

// DeleteExchangeRate removes specific exchange rate
func (n *Node) DeleteExchangeRate(t *testing.T, equivalentFrom, equivalentTo string) {
    // Sends DELETE request, handles errors internally
}

// ClearExchangeRates removes all exchange rates
func (n *Node) ClearExchangeRates(t *testing.T) {
    // Sends DELETE request, handles errors internally
}
```

#### Error Handling Strategy
- All methods accept `t *testing.T` parameter
- HTTP errors, parsing errors, and validation errors cause `t.Fatalf()`
- No error return values - failures stop test execution immediately
- Consistent with existing Node method patterns

#### Test Data Constants
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
    
    // TTL test constants
    TTL_WAIT_DURATION = 5*time.Minute + 30*time.Second
)
```

## Implementation Plan
### This Iteration Timeline
- **Duration**: 1 iteration (single sprint)
- **Sprint Breakdown**: Single development cycle
- **Key Deliverables**: 
  - Extended Node struct with exchange rates methods
  - Comprehensive test suite covering all 11 scenarios
  - Documentation updates

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| HTTP Methods Implementation | TBD | Implement 6 exchange rates methods in Node struct | vtcpd-cli endpoints | Low |
| Validation Tests | TBD | Implement test cases 1-3 for validation scenarios | HTTP methods | Low |
| Conversion Tests | TBD | Implement test cases 4-9 for conversion logic | HTTP methods | Medium |
| CRUD Test | TBD | Implement test case 10 for complete CRUD operations | HTTP methods | Low |
| TTL Test | TBD | Implement test case 11 for TTL expiration | HTTP methods | Medium |
| Integration Testing | TBD | Validate all tests against running vtcpd instances | All prior steps | Low |

### Dependencies on Other Teams/Projects
- **vtcpd implementation**: Exchange rates manager must be available in test environment
- **vtcpd-cli implementation**: HTTP endpoints must be deployed and accessible
- **Docker infrastructure**: Container orchestration for test environments

### Integration Points with Previous Work
- Builds upon existing Node struct patterns and HTTP client functionality
- Utilizes established Docker containerization and test orchestration
- Follows existing error handling and test organization patterns

### Resource Requirements
#### Team Structure
- **Developer**: 1 developer for implementation
- **QA Engineer**: 1 QA engineer for validation and review

#### Budget Estimates
- Development effort: ~2-3 days
- Testing and validation: ~1 day

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| vtcpd-cli endpoints not stable | High | Low | Coordinate with vtcpd-cli team; validate endpoints before implementation |
| Conversion logic edge cases | Medium | Medium | Extensive test coverage including boundary conditions |
| TTL timing issues | Medium | Medium | Use appropriate wait times and validate behavior |
| Docker container resource limits | Low | Low | Monitor resource usage during TTL test |

### Business Risks
- **Testing duration**: TTL test adds ~5.5 minutes to test suite execution time
- **Maintenance overhead**: Additional test cases require ongoing maintenance

## Testing Strategy
### Testing Approach for This Iteration
#### New Feature Testing (Unit)
- **Node Methods Unit Tests**: Each HTTP method tested independently
- **Integration Testing**: End-to-end validation against live vtcpd instances
- **Validation Testing**: Error scenarios and edge cases covered

#### Test Case Implementation
1. **TestExchangeRatesValidationAndConversion**: 
   - Single test function covering cases 1-9
   - Node setup, validation tests, conversion tests, cleanup
   - Expected duration: < 30 seconds

2. **TestExchangeRatesCRUD**:
   - Dedicated test for case 10
   - Multiple rate creation, listing, selective deletion, complete clearing
   - Expected duration: < 60 seconds

3. **TestExchangeRatesTTL**:
   - Dedicated test for case 11  
   - Rate creation, immediate validation, 5.5 minute wait, expiration validation
   - Expected duration: ~6 minutes

#### Quality Gates
- All test cases must pass consistently
- No flaky behavior except for intentional timing in TTL test
- Code follows existing patterns and conventions
- HTTP error handling validated for all scenarios

### Regression Testing
- **Scope**: Existing test suite must continue to pass
- **Automated vs Manual**: Fully automated integration with existing CI/CD
- **Critical User Journeys**: Exchange rates CRUD operations, validation scenarios

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Minor feature addition to test suite
- **Rollout Strategy**: Direct integration with existing test infrastructure
- **Rollback Plan**: Remove new test files and revert Node struct changes

### Feature Flags & Gradual Rollout
- Not applicable - test-only changes

### Database Migrations
- Not applicable - no persistent storage

### Communication Plan
- **Internal**: Update team on new test coverage capabilities
- **External**: Not applicable
- **Documentation Updates**: Update test suite documentation

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**: 
  - All 11 test scenarios implemented and passing
  - 0 false positives or flaky test behavior
  - Test execution time within expected limits
- **Leading Indicators**: 
  - HTTP methods implementation completed
  - Individual test cases passing during development
- **Baseline Values**: Current test suite execution time and coverage
- **Target Values**: 
  - 100% test case implementation
  - <6 minute total execution time for TTL test
  - Consistent pass rate across environments

### Monitoring Plan
- **New Dashboards/Alerts**: Integration with existing CI/CD monitoring
- **Enhanced Monitoring**: Track test execution times and failure rates
- **A/B Testing**: Not applicable

### Review Schedule
- **Daily**: Development progress on test implementation
- **Weekly**: Test execution results and any issues
- **Post-Release Review**: Evaluate test effectiveness and coverage

## Appendices
### Glossary
- **TTL**: Time to Live - automatic expiration mechanism for exchange rates
- **Native Format**: Platform-stable integer representation (value + shift)
- **Real Rate**: Human-readable decimal representation
- **Equivalent**: Currency or token identifier in the system

### References
- vtcpd Exchange Rates Manager: workspace/workflow/prd/vtcpd/03-exchange-rates-manager.md
- vtcpd-cli HTTP Implementation: workspace/workflow/prd/vtcpd-cli/03-exchange-rates-cli-and-http.md
- Existing Node struct patterns: pkg/testsuite/node.go
- HTTP API documentation: vtcpd-cli/readme.md

### Test Case Details
#### Validation Test Cases (1-3, 12)
1. **Missing Rate Value**: POST without real_rate or value+shift → HTTP 400
2. **Conflicting Parameters**: POST with both real_rate and value+shift → HTTP 400  
3. **Excessive Decimal Precision**: POST with real_rate having >16 decimal places → HTTP 400
12. **Unknown Equivalents**: POST with equivalents not in DecimalsMap (e.g., "999", "1111") → HTTP 400

#### Conversion Test Cases (4-9)
4. **Native to Real (High Precision)**: value=11207154, shift=4 → real_rate=112071.54
5. **Real to Native (High Precision)**: real_rate=112071.54 → value=11207154, shift=4
6. **Native to Real (Integer)**: value=112071, shift=2 → real_rate=112071
7. **Real to Native (Integer)**: real_rate=112071 → value=112071, shift=2
8. **Native to Real (Small Decimal)**: value=89228719, shift=11 → real_rate=0.0000089228719
9. **Real to Native (Small Decimal)**: real_rate=0.0000089228719 → value=89228719, shift=11

#### Edge Case Test Cases (10-12)
10. **Non-existent Rate Query**: GET for non-existent rate pair → appropriate error or empty response
11. **Maximum Shift Boundary**: Native input with shift=32767 (max int16) → correct conversion
12. **Minimum Shift Boundary**: Native input with shift=-32768 (min int16) → correct conversion

#### CRUD Test Case (13)
- Set multiple rates for different equivalent pairs
- Verify all rates present via ListExchangeRates
- Delete specific rate, verify others remain
- Clear all rates, verify none remain

#### TTL Test Case (14)
- Set exchange rate
- Verify immediate accessibility  
- Wait 5.5 minutes
- Verify automatic expiration

### Error Conditions
- **HTTP 400**: Invalid request parameters or validation failures
- **HTTP 500**: Internal server errors
- **Network errors**: Connection failures or timeouts
- **Parsing errors**: Invalid JSON responses

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-01-27 | AI Assistant | Initial draft for exchange rates testing implementation | PRD 03 |

**Related Documents**
- **vtcpd Exchange Rates PRD**: workspace/workflow/prd/vtcpd/03-exchange-rates-manager.md
- **vtcpd-cli Exchange Rates PRD**: workspace/workflow/prd/vtcpd-cli/03-exchange-rates-cli-and-http.md
- **Project Policy**: workspace/policy.md
- **Test Suite Documentation**: pkg/testsuite/node.go
