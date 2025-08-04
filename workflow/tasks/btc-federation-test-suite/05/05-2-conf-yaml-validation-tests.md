# 05-2 conf.yaml Validation Tests

# Links
- [PRD](../../prd/btc-federation-test-suite/05-configuration-file-testing.md)
- [Task 05-1](05-1-test-infrastructure-setup.md)

# Description
Implement comprehensive test suite for conf.yaml validation covering all 20 test scenarios including missing files, invalid formats, malformed YAML, and configuration validation errors. Tests validate that the btc-federation node properly handles configuration scenarios with correct exit codes. Must use existing cluster.RunConfigValidation method and node.NewNode for container management and validation.

# Requirements and DOD

## Functional Requirements
1. **Complete Test Coverage**: Implement all 20 test scenarios for conf.yaml validation as defined in PRD
2. **Exit Code Validation**: Verify correct exit codes (0 for success, 1 for errors) for each scenario
3. **File Generation Testing**: Validate scenarios where node generates default config or private keys
4. **Error Handling Testing**: Verify proper error handling for various invalid configuration scenarios
5. **Docker Container Testing**: Use existing cluster.RunConfigValidation method to run isolated test scenarios and validate behavior
6. **Log Message Validation**: Use existing cluster.CheckNodeLogContains method for log validation where applicable
7. **Node Configuration**: Use node.NewNode method to create properly configured test nodes

## Test Scenarios Coverage
### Success Scenarios (Exit Code 0)
1. Missing `conf.yaml` - should generate default config and start successfully
4. Missing `private_key` field - should generate key, save to file, and start successfully

### Error Scenarios (Exit Code 1)
2. Empty `conf.yaml`
3. Invalid YAML structure (malformed YAML)
5. Invalid `private_key` format
6. Missing `network` section
7. Missing `network.addresses` field
8. Empty `network.addresses`
9. Invalid address format in `network.addresses`
10. Missing `peers` section
11. Missing `peers.connection_timeout` field
12. Invalid `peers.connection_timeout` value
13. Missing `logging` section
14. Missing `logging.level` field
15. Invalid `logging.level` value
16. Missing `logging.format` field
17. Invalid `logging.format` value
18. Invalid boolean values in configuration fields
19. Missing `logging.file_name` when `logging.file_output=true`
20. Invalid `logging.file_max_size` format

## Technical Requirements
1. **Container Isolation**: Each test runs using cluster.RunConfigValidation method for isolated Docker container execution
2. **Temporary Files**: Use temporary configuration files for each test scenario following node.go patterns
3. **Timeout Management**: Proper timeout handling for container operations using cluster.go timeout patterns
4. **Cleanup**: Ensure proper cleanup using cluster.Cleanup method and temporary file cleanup
5. **Parallel Execution**: Support for parallel test execution where possible using existing cluster infrastructure
6. **Infrastructure Reuse**: Leverage existing cluster.NewCluster and node.NewNode methods for consistent test setup

## Definition of Done
- [ ] All 20 test scenarios implemented in `test/conf/conf_test.go` using cluster.RunConfigValidation
- [ ] Each test validates correct exit code (0 or 1 as specified) via cluster.RunConfigValidation result
- [ ] Success scenarios verify file generation (config/key creation) using cluster methods
- [ ] Error scenarios verify proper error handling through cluster.RunConfigValidation
- [ ] All tests use cluster.RunConfigValidation for proper Docker container isolation
- [ ] Tests integrate with existing cluster infrastructure using cluster.NewCluster and node.NewNode
- [ ] Proper cleanup implemented using cluster.Cleanup and file cleanup for all test scenarios
- [ ] Test execution completes within 3 minutes for all scenarios using efficient cluster methods
- [ ] All tests are deterministic and repeatable using existing infrastructure patterns

# Implementation Plan

## Step 1: Test File Structure Setup
- Create `test/conf/conf_test.go` with proper package and imports
- Set up test constants for all scenarios using DRY principles
- Define test helper functions for common operations

## Step 2: Success Scenarios Implementation (Exit Code 0)
- **Test 1**: Missing conf.yaml scenario
  - Verify default config generation
  - Validate successful container startup
  - Check generated config file content
- **Test 4**: Missing private_key scenario  
  - Verify private key generation
  - Validate key is saved to config file
  - Confirm successful startup

## Step 3: Basic Structure Error Scenarios (Tests 2-12)
- **Test 2**: Empty conf.yaml file
- **Test 3**: Invalid YAML structure (malformed YAML)
- **Tests 5**: Invalid private_key format validation
- **Tests 6-9**: Network section validation scenarios
- **Tests 10-12**: Peers section validation scenarios

## Step 4: Logging Configuration Error Scenarios (Tests 13-20)
- **Tests 13-17**: Basic logging field validation
- **Test 18**: Boolean values validation
- **Tests 19-20**: Advanced logging configuration validation

## Step 5: Integration and Optimization
- Implement parallel test execution where appropriate
- Add comprehensive cleanup procedures
- Optimize test execution time
- Add detailed error reporting and debugging information

# Test Plan

## Objective
Verify that btc-federation node correctly validates conf.yaml files and responds with appropriate exit codes and behaviors for all defined scenarios.

## Test Scope
- All 20 conf.yaml validation scenarios
- Container exit code validation
- File generation verification (for success scenarios)
- Error message validation (where applicable)
- Container lifecycle management

## Environment & Setup
- Docker environment with btc-federation-test image
- Temporary directory for configuration files
- Access to testsuite cluster infrastructure
- Isolated test execution environment

## Mocking Strategy
- No mocking of btc-federation binary (use real binary)
- Use real Docker containers for isolation
- Mock only external file system operations if necessary
- Use existing cluster methods without mocking

## Key Test Scenarios

### Success Path Testing
1. **Missing Configuration File**:
   - Container starts without conf.yaml
   - Verify default config file is generated
   - Validate container exits with code 0
   - Check generated config content is valid

2. **Missing Private Key**:
   - Start with config missing private_key field
   - Verify private key is generated and saved
   - Validate successful startup (exit code 0)
   - Check private key format is valid

### Error Path Testing
1. **File Format Errors**:
   - Empty configuration file → exit code 1
   - Malformed YAML syntax → exit code 1
   - Invalid boolean values → exit code 1

2. **Configuration Structure Errors**:
   - Missing required sections → exit code 1
   - Missing required fields → exit code 1
   - Invalid field values → exit code 1

3. **Network Configuration Errors**:
   - Invalid multiaddr formats → exit code 1
   - Empty address arrays → exit code 1

4. **Logging Configuration Errors**:
   - Invalid log levels → exit code 1
   - Invalid log formats → exit code 1
   - Missing file names when file output enabled → exit code 1

### Edge Cases
1. **Invalid Data Types**: Test scenarios with wrong data types for configuration fields
2. **Boundary Values**: Test edge cases for timeout values and size limits
3. **Special Characters**: Test handling of special characters in configuration values

## Success Criteria
- All 20 test scenarios execute successfully
- Correct exit codes validated for each scenario
- Success scenarios properly verify file generation
- Error scenarios properly validate error handling
- Test execution time under 3 minutes
- 100% test pass rate with deterministic results

# Verification and Validation

## Architecture integrity
- Tests follow existing test patterns and conventions
- Proper integration with testsuite infrastructure
- No violations of test architecture principles

## Security
- No security vulnerabilities in test scenarios
- Proper handling of temporary files and cleanup
- Safe container execution and management

## Performance
- Test execution completes within 3 minutes
- Efficient use of Docker resources
- Minimal overhead for test infrastructure

## Scalability
- Tests support parallel execution
- Can handle multiple concurrent test scenarios
- Scalable to additional test cases in future

## Reliability
- Tests are deterministic and repeatable
- Robust error handling and cleanup
- Consistent behavior across different environments

## Maintainability
- Clear test structure and organization
- Well-documented test scenarios
- Easy to extend for additional validation cases
- Follows project coding standards

## Cost
- Efficient resource utilization
- Minimal test execution overhead
- No unnecessary complexity

## Compliance
- Follows task-driven development policy
- Adheres to DRY principles for test constants
- Integrates with existing infrastructure as required
- Proper validation criteria proportional to task complexity 