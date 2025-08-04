# 05-3 peers.yaml Validation Tests

# Links
- [PRD](../../prd/btc-federation-test-suite/05-configuration-file-testing.md)
- [Task 05-1](05-1-test-infrastructure-setup.md)

# Description
Implement test suite for peers.yaml validation covering all 3 test scenarios including missing files, empty files, and standalone mode operation. Tests validate that the btc-federation node properly handles peer configuration scenarios with correct exit codes and log messages. Must use existing cluster.CheckNodeLogContains method for log validation and cluster.RunConfigValidation for container management.

# Requirements and DOD

## Functional Requirements
1. **Complete Test Coverage**: Implement all 3 test scenarios for peers.yaml validation as defined in PRD
2. **Exit Code Validation**: Verify correct exit codes (0 for standalone, 1 for errors) using cluster.RunConfigValidation for each scenario
3. **Log Message Validation**: Use existing cluster.CheckNodeLogContains method to verify standalone mode messages
4. **Standalone Mode Testing**: Validate proper standalone mode operation when no peers configured using cluster methods
5. **Docker Container Testing**: Use cluster.RunConfigValidation method to isolate test scenarios and validate behavior
6. **Error Handling Testing**: Verify proper error handling for invalid peers.yaml files through cluster.RunConfigValidation
7. **Node Configuration**: Use node.NewNode method to create properly configured test nodes with peers configuration

## Test Scenarios Coverage
### Success Scenarios (Exit Code 0 + Log Message)
1. Missing `peers.yaml` - should start successfully with log message "No peers configured - running in standalone mode"
3. Empty `peers` value in `peers.yaml` - should start successfully with log message "No peers configured - running in standalone mode"

### Error Scenarios (Exit Code 1)
2. Empty `peers.yaml` - should exit with code 1

## Technical Requirements
1. **Container Isolation**: Each test runs using cluster.RunConfigValidation method for isolated Docker container execution
2. **Log Message Validation**: Integration with cluster.CheckNodeLogContains for standalone mode message verification
3. **Temporary Files**: Use temporary peers.yaml files for each test scenario following node.go patterns
4. **Timeout Management**: Proper timeout handling for container operations using cluster.go timeout patterns
5. **Cleanup**: Ensure proper cleanup using cluster.Cleanup method and temporary file cleanup after tests
6. **Infrastructure Reuse**: Leverage existing cluster.NewCluster and node.NewNode methods for consistent test setup

## Definition of Done
- [ ] All 3 test scenarios implemented in `test/conf/peers_test.go` using cluster.RunConfigValidation
- [ ] Each test validates correct exit code (0 or 1 as specified) via cluster.RunConfigValidation result
- [ ] Standalone scenarios verify log message "No peers configured - running in standalone mode" using cluster.CheckNodeLogContains
- [ ] Error scenarios verify proper error handling through cluster.RunConfigValidation
- [ ] All tests use cluster.RunConfigValidation for proper Docker container isolation
- [ ] Tests integrate with existing cluster infrastructure using cluster.NewCluster, node.NewNode, and cluster.CheckNodeLogContains
- [ ] Proper cleanup implemented using cluster.Cleanup and file cleanup for all test scenarios
- [ ] Test execution completes within 1 minute for all scenarios using efficient cluster methods
- [ ] All tests are deterministic and repeatable using existing infrastructure patterns

# Implementation Plan

## Step 1: Test File Structure Setup
- Create `test/conf/peers_test.go` with proper package and imports
- Set up test constants for all scenarios using DRY principles
- Define test helper functions for peers.yaml operations
- Set up integration with cluster.CheckNodeLogContains method

## Step 2: Standalone Mode Success Scenarios (Tests 1 & 3)
- **Test 1**: Missing peers.yaml scenario
  - Use cluster.RunConfigValidation to start container without peers.yaml file
  - Verify container starts successfully (exit code 0) via cluster.RunConfigValidation result
  - Use cluster.CheckNodeLogContains to verify standalone mode message
  - Validate node operates in standalone mode using cluster methods

- **Test 3**: Empty peers value scenario
  - Create peers.yaml with empty peers array
  - Verify container starts successfully (exit code 0)
  - Use CheckNodeLogContains to verify standalone mode message
  - Validate proper handling of empty peers configuration

## Step 3: Error Scenario Implementation (Test 2)
- **Test 2**: Empty peers.yaml file
  - Create completely empty peers.yaml file
  - Verify container exits with code 1
  - Validate proper error handling for malformed file

## Step 4: Integration and Optimization
- Implement proper container lifecycle management
- Add comprehensive cleanup procedures
- Optimize test execution time
- Add detailed error reporting and debugging information
- Verify integration with existing cluster methods

# Test Plan

## Objective
Verify that btc-federation node correctly handles peers.yaml configuration scenarios with appropriate exit codes and log messages, particularly for standalone mode operation.

## Test Scope
- All 3 peers.yaml validation scenarios
- Container exit code validation
- Log message verification using CheckNodeLogContains
- Standalone mode operation validation
- Container lifecycle management

## Environment & Setup
- Docker environment with btc-federation-test image
- Valid conf.yaml file for node startup
- Temporary directory for peers.yaml files
- Access to testsuite cluster infrastructure with CheckNodeLogContains
- Isolated test execution environment

## Mocking Strategy
- No mocking of btc-federation binary (use real binary)
- Use real Docker containers for isolation
- Use existing CheckNodeLogContains method without mocking
- Mock only external file system operations if necessary

## Key Test Scenarios

### Standalone Mode Success Testing
1. **Missing peers.yaml File**:
   - Container starts with valid conf.yaml but no peers.yaml
   - Verify container starts successfully (exit code 0)
   - Use CheckNodeLogContains to find "No peers configured - running in standalone mode"
   - Validate node continues running in standalone mode

2. **Empty Peers Configuration**:
   - Create peers.yaml with structure: `peers: []` or `peers:`
   - Verify container starts successfully (exit code 0)
   - Use CheckNodeLogContains to find standalone mode message
   - Validate proper parsing of empty peers configuration

### Error Path Testing
1. **Empty peers.yaml File**:
   - Create completely empty peers.yaml file (0 bytes)
   - Verify container exits with code 1
   - Validate proper error handling for malformed YAML

### Log Message Validation
1. **CheckNodeLogContains Integration**:
   - Verify integration with existing log checking method
   - Test exact message matching for standalone mode
   - Validate proper timeout handling for log message checks
   - Test error handling when log messages are not found

## Success Criteria
- All 3 test scenarios execute successfully
- Correct exit codes validated for each scenario
- Standalone mode log messages properly detected using CheckNodeLogContains
- Error scenarios properly validate error handling
- Test execution time under 1 minute
- 100% test pass rate with deterministic results

# Verification and Validation

## Architecture integrity
- Tests follow existing test patterns and conventions
- Proper integration with testsuite infrastructure
- Effective use of existing CheckNodeLogContains method
- No violations of test architecture principles

## Security
- No security vulnerabilities in test scenarios
- Proper handling of temporary files and cleanup
- Safe container execution and management

## Performance
- Test execution completes within 1 minute
- Efficient use of Docker resources
- Minimal overhead for test infrastructure
- Quick log message validation

## Scalability
- Tests support parallel execution
- Can handle multiple concurrent test scenarios
- Scalable to additional peers.yaml validation cases

## Reliability
- Tests are deterministic and repeatable
- Robust error handling and cleanup
- Consistent behavior across different environments
- Reliable log message detection

## Maintainability
- Clear test structure and organization
- Well-documented test scenarios
- Easy to extend for additional peers validation cases
- Follows project coding standards
- Effective integration with existing infrastructure

## Cost
- Efficient resource utilization
- Minimal test execution overhead
- No unnecessary complexity
- Reuses existing infrastructure

## Compliance
- Follows task-driven development policy
- Adheres to DRY principles for test constants
- Integrates with existing infrastructure as required
- Proper validation criteria proportional to task complexity
- Uses existing CheckNodeLogContains method as specified 