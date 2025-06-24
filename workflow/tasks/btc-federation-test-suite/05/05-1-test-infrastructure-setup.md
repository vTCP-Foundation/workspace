# 05-1 Test Infrastructure Setup

# Links
- [PRD](../../prd/btc-federation-test-suite/05-configuration-file-testing.md)

# Description
Create test infrastructure and utility functions for configuration file testing, including helper methods for creating temporary configuration files, managing Docker containers, and validating exit codes and log messages. Must leverage existing methods from cluster.go and node.go files and extend them as needed while maintaining compatibility.

# Requirements and DOD

## Functional Requirements
1. **Test Directory Structure**: Create proper directory structure for configuration tests in `test/conf/`
2. **Configuration File Utilities**: Implement helper functions for creating temporary configuration files with various scenarios using node.go patterns
3. **Container Management Integration**: Integrate with existing testsuite cluster methods (cluster.NewCluster, cluster.RunConfigValidation) for container lifecycle management
4. **Exit Code Validation**: Implement utilities for monitoring and validating Docker container exit codes using cluster.RunConfigValidation method
5. **Log Validation Integration**: Integrate with existing `cluster.CheckNodeLogContains` method for log message validation
6. **Test Constants**: Define constants for repeated values like timeouts, file paths, and test scenarios
7. **Infrastructure Extension**: Extend cluster.go and node.go methods if needed for configuration-specific testing requirements

## Technical Requirements
1. **File Management**: Safe creation and cleanup of temporary configuration files
2. **Docker Integration**: Proper integration with existing Docker infrastructure
3. **Error Handling**: Robust error handling for file operations and container management
4. **Code Reusability**: Create modular, reusable utility functions

## Definition of Done
- [ ] Test directory structure created in `test/conf/`
- [ ] Helper functions for creating temporary YAML files implemented
- [ ] Container exit code validation utilities implemented  
- [ ] Integration with existing CheckNodeLogContains method verified
- [ ] All utility functions have proper error handling
- [ ] Code follows existing project patterns and conventions
- [ ] All constants defined for repeated values
- [ ] Documentation for utility functions is complete

# Implementation Plan

## Step 1: Create Test Directory Structure
- Create `test/conf/` directory structure
- Add necessary imports and package declarations
- Set up common test constants following DRY principles

## Step 2: Implement Configuration File Utilities
- Create `createTempConfigFile()` function for generating temporary conf.yaml files
- Create `createTempPeersFile()` function for generating temporary peers.yaml files  
- Implement configuration scenario builders for various test cases
- Add proper file cleanup mechanisms

## Step 3: Container Management Integration
- Integrate with existing testsuite cluster infrastructure using cluster.NewCluster
- Use cluster.RunConfigValidation for configuration validation containers
- Implement exit code monitoring using existing cluster patterns
- Add timeout handling for container operations following cluster.go patterns
- Extend cluster.go with new configuration validation methods if needed

## Step 4: Log Validation Integration  
- Use existing cluster.CheckNodeLogContains method directly
- Implement log pattern validation utilities as wrappers around cluster methods
- Add helper functions for common log message checks using cluster.CheckNodeLogContains
- Follow existing cluster.go patterns for log validation

## Step 5: Error Handling and Validation
- Add comprehensive error handling for all utility functions
- Implement validation for temporary file creation
- Add proper cleanup procedures for failed operations
- Test error scenarios and edge cases

# Test Plan

## Objective
Verify that all test infrastructure utilities work correctly and integrate properly with existing cluster methods.

## Test Scope
- Utility function functionality
- Integration with existing testsuite infrastructure
- File creation and cleanup operations
- Container management wrapper functions
- Log validation integration

## Environment & Setup
- Docker environment with btc-federation-test image
- Access to existing testsuite cluster infrastructure
- Temporary directory for test file operations

## Mocking Strategy
- Use existing cluster methods without mocking
- Mock only external file system operations where necessary
- No mocking of Docker infrastructure (use real containers)

## Key Test Scenarios

### Functional Testing
1. **Configuration File Creation**:
   - Create valid conf.yaml with various configurations
   - Create invalid conf.yaml with malformed content
   - Create temporary peers.yaml files
   - Verify proper file cleanup after operations

2. **Container Integration**:
   - Start container with custom configuration files
   - Monitor container exit codes correctly
   - Validate container shutdown and cleanup

3. **Log Validation**:
   - Integration with CheckNodeLogContains works correctly
   - Log pattern matching functions work as expected
   - Proper error handling for log validation failures

### Error Handling
1. **File System Errors**:
   - Handle temporary directory creation failures
   - Manage disk space or permission issues
   - Proper cleanup on file operation failures

2. **Container Errors**:
   - Handle container startup failures gracefully
   - Manage Docker daemon connection issues
   - Proper timeout handling for container operations

## Success Criteria
- All utility functions execute without errors
- Integration with existing cluster methods works seamlessly
- Temporary files are created and cleaned up properly
- Container exit codes are monitored accurately
- Log validation integration functions correctly

# Verification and Validation

## Architecture integrity
- Functions integrate properly with existing testsuite package
- No violations of existing architectural patterns
- Proper separation of concerns between utilities

## Security
- No security vulnerabilities in file operations
- Proper cleanup prevents information leakage
- Safe handling of temporary files and directories

## Performance
- File operations complete within reasonable timeframes
- Container management doesn't introduce significant overhead
- Memory usage is appropriate for test operations

## Scalability
- Utilities can handle multiple concurrent test scenarios
- Proper resource management for parallel test execution

## Reliability
- Robust error handling and recovery mechanisms
- Consistent behavior across different test scenarios
- Reliable cleanup procedures

## Maintainability
- Code follows project conventions and patterns
- Functions are well-documented and modular
- Easy to extend for additional test scenarios

## Cost
- Minimal resource overhead for test infrastructure
- Efficient use of Docker resources
- No unnecessary complexity

## Compliance
- Follows project policy for task-driven development
- Adheres to DRY principles for constants and repeated values
- Integrates with existing infrastructure as required 