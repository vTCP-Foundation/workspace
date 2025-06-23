# 03-2 - Extend node.go with Log Analysis Methods

# Links
- [PRD](../../prd/btc-federation-test-suite/03-p2p-connection-testing.md)

# Description
Extend the node.go module with methods to analyze log files within Docker containers. This enables verification of P2P connection events at the application level by searching for specific patterns in btc-federation node logs.

# Requirements and DOD
## Requirements
- Implement `CheckLogContains(pattern string) (bool, error)` method
- Implement `GetPeersConfig() (map[string]interface{}, error)` method
- Support searching for multiple connection success patterns in logs
- Handle various log formats (JSON and text)
- Pattern matching must be case-sensitive and exact
- Method must work with btc-federation log files within containers
- Proper error handling for container access and log file reading failures

## Definition of Done
- [ ] `CheckLogContains` method implemented with pattern matching capability
- [ ] `GetPeersConfig` method implemented to retrieve peers configuration
- [ ] Support for multiple connection success patterns:
  - "Connection established with peer"
  - "Successfully established connection" 
  - "Successfully connected to peer"
- [ ] Robust error handling for log file access failures
- [ ] Unit tests for pattern matching logic
- [ ] Integration tests with real container log files
- [ ] Code documentation updated
- [ ] No breaking changes to existing node.go functionality

# Implementation Plan
## Step 1: Implement CheckLogContains method
- Access container using existing cluster reference
- Use container execution to read log files (cat, grep, or tail commands)
- Implement pattern matching against log content
- Handle both JSON and text log formats
- Return boolean result with error information

## Step 2: Implement GetPeersConfig method
- Read peers.yaml file from container
- Parse YAML content into map structure
- Handle file not found scenarios
- Return structured configuration data

## Step 3: Log file location and access
- Determine standard log file locations in containers
- Handle different log file names based on node configuration
- Use container execution methods for file access

## Step 4: Pattern matching optimization
- Implement efficient search algorithms
- Handle large log files gracefully
- Support multiple pattern checking in single operation

## Step 5: Error handling and edge cases
- Container not running scenarios
- Log files not found or empty
- Permission issues accessing log files
- Malformed log content

# Test Plan
## Objective
Verify that log analysis methods can accurately detect P2P connection events in btc-federation node logs and retrieve peer configuration information.

## Test Scope
- `CheckLogContains` method with various log content scenarios
- `GetPeersConfig` method functionality
- Pattern matching accuracy for connection success messages
- Error handling for file access and parsing failures

## Environment & Setup
- Docker containers with btc-federation nodes running
- Test containers with known log content
- Mock log files for various scenario testing

## Mocking Strategy
- Mock container execution for unit tests
- Use real containers with actual logs for integration tests
- Mock file system operations for error scenario testing

## Key Test Scenarios
1. **Pattern Matching**: 
   - Search for each connection success pattern individually
   - Verify case-sensitive matching
   - Test with JSON and text log formats
   
2. **Log File Access**:
   - Standard log file locations
   - Custom log file names
   - Empty or missing log files

3. **Peers Configuration**:
   - Valid peers.yaml file parsing
   - Empty or missing peers.yaml file
   - Malformed YAML content

4. **Error Scenarios**:
   - Container not accessible
   - Permission denied for log files
   - Corrupted log content

## Success Criteria
- All unit tests pass
- Pattern matching works correctly with real btc-federation logs
- Configuration retrieval handles all edge cases gracefully
- Error scenarios return appropriate error messages without panics

# Verification and Validation

## Architecture integrity
- Methods integrate seamlessly with existing node.go structure
- Consistent with existing method patterns and naming conventions
- No modifications to existing public interfaces
- Proper error handling consistent with project standards

## Security
- No security risks introduced by log file access
- Container access uses existing secure patterns
- No exposure of sensitive configuration data in logs

## Performance
- Log analysis completes within reasonable time (<5 seconds)
- Efficient pattern matching for large log files
- No memory issues with log content processing

## Scalability
- Methods handle varying log file sizes efficiently
- Pattern matching scales with log content volume
- Configuration parsing handles different YAML structures

## Reliability
- Consistent results across multiple executions
- Robust error handling for all failure scenarios
- Graceful degradation when log files are unavailable

## Maintainability
- Clear method interfaces for future extension
- Well-documented pattern matching behavior
- Testable design with appropriate separation of concerns

## Cost
- No additional infrastructure costs
- Minimal computational overhead for log analysis

## Compliance
- Follows project coding standards and error handling patterns
- Adheres to existing node.go API design principles 