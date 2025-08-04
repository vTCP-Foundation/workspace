# 03-1 - Extend cluster.go with TCP Analysis Methods

# Links
- [PRD](../../prd/btc-federation-test-suite/03-p2p-connection-testing.md)

# Description
Extend the cluster.go module with methods to analyze TCP connections within Docker containers. This provides the foundation for programmatic verification of P2P network connections at the TCP level.

# Requirements and DOD
## Requirements
- Implement `CheckTCPConnections(containerID string) ([]TCPConnection, error)` method
- Implement `GetActiveConnectionsCount(containerID string) (int, error)` method
- Define `TCPConnection` struct to represent network connection state
- Methods must work reliably within Docker container networking environment
- Support identification of P2P connections on ports 9000/9001
- Connection count validation must support both 1 and 2 connections (bidirectional scenario)
- Proper error handling for container access and network analysis failures

## Definition of Done
- [ ] `TCPConnection` struct defined with appropriate fields (Source IP/Port, Destination IP/Port, Connection state)
- [ ] `CheckTCPConnections` method implemented and tested
- [ ] `GetActiveConnectionsCount` method implemented and tested
- [ ] Methods correctly identify P2P connections on expected ports
- [ ] Error handling implemented for container execution failures
- [ ] Unit tests written for new methods
- [ ] Integration test with actual Docker container demonstrates functionality
- [ ] Code documentation updated
- [ ] No breaking changes to existing cluster.go functionality

# Implementation Plan
## Step 1: Define TCPConnection structure
- Create struct with fields for connection representation
- Include source/destination IP and ports
- Add connection state information
- Consider additional metadata for debugging

## Step 2: Implement CheckTCPConnections method
- Use existing `ExecInContainer` method to run network analysis commands
- Execute `netstat` or `ss` commands to get connection information
- Parse command output to extract TCP connection data
- Return structured `TCPConnection` array
- Handle parsing errors and empty results

## Step 3: Implement GetActiveConnectionsCount method
- Leverage `CheckTCPConnections` for underlying data
- Filter connections by port range (9000-9001)
- Count relevant P2P connections
- Return integer count with error handling

## Step 4: Error handling and edge cases
- Container not running scenarios
- Network command execution failures
- Output parsing errors
- Empty connection lists

## Step 5: Testing and validation
- Unit tests for parsing logic
- Integration tests with real containers
- Validation of connection count accuracy

# Test Plan
## Objective
Verify that TCP connection analysis methods work correctly within Docker container environment and can accurately detect P2P connections.

## Test Scope
- `TCPConnection` struct functionality
- `CheckTCPConnections` method with various connection scenarios
- `GetActiveConnectionsCount` method accuracy
- Error handling for container and networking failures

## Environment & Setup
- Docker container with btc-federation node running
- Test containers with known TCP connections
- Mock containers for error scenario testing

## Mocking Strategy
- Mock container execution for unit tests
- Use real containers for integration tests
- Mock network command outputs for parsing tests

## Key Test Scenarios
1. **Happy Path**: Container with active P2P connections
   - Verify correct parsing of netstat/ss output
   - Validate connection count accuracy
   
2. **Edge Cases**: 
   - Container with no connections
   - Container not running
   - Network commands failing
   - Malformed command output

3. **P2P Specific**:
   - Single connection scenario (1 connection)
   - Bidirectional scenario (2 connections)
   - Mixed connections (P2P + other ports)

## Success Criteria
- All unit tests pass
- Integration tests with real containers demonstrate accurate connection detection
- Error scenarios handled gracefully without panics
- Methods consistently identify P2P connections on ports 9000/9001

# Verification and Validation

## Architecture integrity
- Methods integrate cleanly with existing cluster.go structure
- No modifications to existing public interfaces
- Consistent error handling patterns with existing code
- Proper separation of concerns maintained

## Security
- No security implications for this testing infrastructure enhancement
- Container access uses existing secure patterns
- No exposure of sensitive connection data

## Performance
- Methods execute within reasonable time (<5 seconds per call)
- No memory leaks in connection data structures
- Efficient parsing of network command outputs

## Scalability
- Methods designed to handle multiple concurrent containers
- Connection parsing scales with number of connections
- No resource exhaustion under normal testing loads

## Reliability
- Robust error handling for all failure scenarios
- Consistent results across multiple executions
- Graceful degradation when network commands fail

## Maintainability
- Clear code structure with appropriate comments
- Well-defined interfaces for future extensions
- Testable design with good separation of concerns

## Cost
- No additional infrastructure costs
- Minimal computational overhead for network analysis

## Compliance
- Follows project coding standards and patterns
- Adheres to existing cluster.go API design principles 