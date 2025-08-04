# Project Requirements Document (PRD)

## Document Information
- **Project Name**: BTC Federation Test Suite
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.0
- **Date**: 2025-01-27
- **Author(s)**: AI Assistant
- **Stakeholders**: Dima Chizhevsky, Mykola Ilashchuk
- **Status**: Draft
- **Previous PRD**: [01-test-suite-foundation.md](./01-test-suite-foundation.md), [02-log-rotation-testing.md](./02-log-rotation-testing.md)
- **Related Documents**: [P2P Network Stack PRD](../btc-federation/02_p2p_network_stack.md)

## Executive Summary
Brief overview of this iteration, its purpose, and expected outcomes. Include:
- **Current project state**: Test suite foundation has been implemented with Docker cluster management and node lifecycle management. Log rotation testing has been successfully implemented.
- **This iteration's focus**: Implement P2P connection testing to verify that btc-federation nodes can establish and maintain peer-to-peer connections using the libp2p framework with TCP transport.
- **Connection to overall vision**: This PRD provides comprehensive testing capabilities for the P2P networking functionality implemented in the btc-federation project, ensuring network reliability and proper peer discovery mechanisms.

## Iteration Context
### Previous Iterations Summary
- **Completed Features**: 
  - Docker-based test cluster management (`pkg/testsuite/cluster.go`, `pkg/testsuite/node.go`)
  - Container lifecycle management with health checks
  - Log rotation testing with file size validation
  - Environment variable-based node configuration
- **Lessons Learned**: 
  - Docker container management requires careful cleanup and resource management
  - Configuration through environment variables provides flexibility for testing
  - Health checks are essential for reliable container startup detection
- **Technical Debt**: None significant for this iteration
- **User Feedback**: Need for comprehensive P2P connectivity testing to validate network stack implementation

### Current State Analysis
- **What's working well**: 
  - Docker cluster management provides stable foundation for multi-node testing
  - Environment variable configuration pattern works effectively
  - Container lifecycle management is reliable
- **Pain points identified**: 
  - No existing validation of P2P connectivity between nodes
  - Limited network-level testing capabilities
  - Need for log analysis capabilities to verify network events
- **Performance metrics**: Container startup time ~5-10 seconds, health check reliability >95%

## Problem Statement
### Background
The BTC Federation project has implemented a P2P network stack using libp2p with TCP transport protocol. The network stack includes peer discovery, connection management, and peer exchange capabilities. However, there is currently no automated testing to verify that:

1. Nodes can successfully establish P2P connections with each other
2. The peer discovery mechanism works correctly
3. Connection state is properly tracked and logged
4. The peers.yaml configuration is correctly processed

### Problem Description
Without comprehensive P2P connection testing, there is risk that:
- Network connectivity issues go undetected during development
- Peer discovery mechanisms fail silently
- Configuration problems prevent proper node communication
- Performance degradation in connection establishment goes unnoticed

The testing gap affects the development team's ability to validate network functionality and could lead to deployment of non-functional networking features.

### Success Metrics
- **Primary KPIs**: 
  - P2P connection establishment success rate: 100%
  - Connection detection accuracy: 100% (both TCP-level and log-level verification)
  - Test execution time: < 60 seconds
  - Test reliability: > 95% pass rate across multiple runs
- **Secondary KPIs**: 
  - Container startup time: < 30 seconds for 2 nodes
  - Log processing accuracy: 100% detection of connection events
- **Target Values**: 
  - 2 nodes establish exactly 1 or 2 TCP connections (bidirectional)
  - Connection success logged in both node logs
  - Zero false positives in connection detection

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- **P2P Connection Test Implementation**: Create comprehensive test at `test/p2p/p2p_test.go` that validates end-to-end P2P connectivity
- **TCP Connection Verification**: Implement methods to analyze TCP connections at Docker container level
- **Log Analysis Capability**: Add functionality to search and verify log entries for connection events
- **Peers Configuration Support**: Extend node configuration to support peers.yaml generation through environment variables
- **Dockerfile Enhancement**: Modify Dockerfile to create peers.yaml file based on environment variables

#### Bug Fixes & Technical Improvements
- Enhance error handling in container management for network-related failures
- Improve timeout handling for connection establishment scenarios
- Add better logging for debugging connection issues during testing

#### Modifications to Existing Features
- **NodeConfig Extension**: Add fields to support peers.yaml configuration
- **Cluster Management Enhancement**: Add network analysis methods
- **Container Configuration**: Extend environment variable support for peer configuration

### Explicitly Out of Scope
- Multi-node scenarios beyond 2 nodes
- Performance benchmarking of P2P connections
- Network failure simulation or resilience testing
- Advanced peer discovery scenarios (dynamic peer addition/removal)
- Security testing of P2P connections
- Cross-platform compatibility testing

### Dependencies from Previous Iterations
- Docker cluster management infrastructure (`pkg/testsuite/cluster.go`)
- Node lifecycle management (`pkg/testsuite/node.go`)
- Dockerfile configuration system
- Environment variable-based configuration pattern

### Future Roadmap Impact
This iteration establishes the foundation for:
- Multi-node network testing scenarios
- Network performance benchmarking
- Failure simulation and resilience testing
- Security and encryption testing of P2P communications

## User Stories & Requirements

### User Personas
#### Primary User: BTC Federation Developer
- **Role**: Software Developer working on BTC Federation network stack
- **Goals**: Validate P2P networking implementation, ensure reliable connections between nodes
- **Pain Points**: Manual testing is time-consuming, difficult to verify network state programmatically
- **Technical Proficiency**: Advanced - familiar with Docker, networking concepts, Go testing

#### Secondary User: DevOps Engineer
- **Role**: Infrastructure Engineer responsible for deployment and monitoring
- **Goals**: Ensure network functionality before deployment, automated validation of network features
- **Pain Points**: Limited visibility into P2P connection state, manual verification processes
- **Technical Proficiency**: Advanced - experienced with containerization and network troubleshooting

### Functional Requirements
#### New Features for This Iteration
1. **P2P Connection Test Implementation with Real Cryptographic Keys**
   - **Description**: Comprehensive test that validates P2P connectivity between two btc-federation nodes using real libp2p cryptographic key pairs
   - **User Story**: As a developer, I want to run automated tests that verify P2P connections with real cryptographic keys so that I can be confident the network stack works correctly in production-like scenarios
   - **Rationale**: Automated testing with real keys is essential for reliable network stack development and ensures cryptographic components work correctly
   - **Builds Upon**: Existing cluster management and Docker infrastructure, btc-federation keygen utility
   - **Acceptance Criteria**: 
     - Test creates 2 btc-federation nodes with proper peers.yaml configuration
     - Uses real libp2p ECDSA key pairs generated by keygen utility
     - Each node's private key is configured in its conf.yaml
     - Each node's public key is configured in the other node's peers.yaml
     - Verifies TCP connections are established (1 or 2 connections acceptable)
     - Validates connection success messages in both node logs
     - Test completes within 60 seconds
     - Test passes consistently (>95% reliability)
   - **Priority**: High
   - **Dependencies**: Existing cluster.go and node.go modules, Docker infrastructure, btc-federation keygen utility

2. **TCP Connection Analysis Methods**
   - **Description**: Methods to inspect TCP connections within Docker containers
   - **User Story**: As a developer, I want to programmatically verify network connections so that I can validate network state during tests
   - **Rationale**: TCP-level verification provides objective proof of connection establishment
   - **Builds Upon**: Existing container execution capabilities
   - **Acceptance Criteria**: 
     - `CheckTCPConnections(containerID string) ([]TCPConnection, error)` method implemented
     - `GetActiveConnectionsCount(containerID string) (int, error)` method implemented
     - Methods correctly identify P2P connections on port 9000/9001
     - Connection count validation supports both 1 and 2 connections
   - **Priority**: High
   - **Dependencies**: Container execution infrastructure

3. **Log Analysis Capability**
   - **Description**: Methods to search for specific patterns in node logs within containers
   - **User Story**: As a developer, I want to verify that connection events are properly logged so that I can debug network issues
   - **Rationale**: Log-level verification confirms application-level connection awareness
   - **Builds Upon**: Existing container execution capabilities
   - **Acceptance Criteria**: 
     - `CheckLogContains(pattern string) (bool, error)` method implemented in node.go
     - Method can search for multiple connection success patterns
     - Supports searching for patterns: "Connection established with peer", "Successfully established connection", "Successfully connected to peer"
     - Returns accurate boolean results for pattern matching
   - **Priority**: High
   - **Dependencies**: Container execution infrastructure

4. **Peers Configuration Support with Cryptographic Keys**
   - **Description**: Extend NodeConfig and environment variable system to support peers.yaml generation with proper libp2p cryptographic key pairs
   - **User Story**: As a developer, I want to configure P2P peers with real cryptographic keys through the test framework so that I can test authentic P2P networking scenarios
   - **Rationale**: Dynamic peer configuration with real keys is essential for realistic network testing that matches production behavior
   - **Builds Upon**: Existing environment variable configuration system, btc-federation keygen utility
   - **Acceptance Criteria**: 
     - NodeConfig extended with peers configuration fields
     - Environment variable `PEERS_YAML_CONTENT` added to Dockerfile
     - Dockerfile generates peers.yaml from environment variable
     - Test uses real libp2p key pairs generated by keygen utility
     - Private keys configured in each node's conf.yaml
     - Public keys configured in peer nodes' peers.yaml
     - Key pair constants included in test for consistent testing
   - **Priority**: High
   - **Dependencies**: Existing node configuration system, Dockerfile, btc-federation keygen utility

#### Enhancements to Existing Features
1. **NodeConfig Extension**
   - **Current State**: Supports basic node configuration (IP, port, logging settings)
   - **Proposed Changes**: Add peers configuration fields and methods
   - **Impact Assessment**: No breaking changes to existing functionality
   - **Migration Strategy**: New fields are optional, existing tests continue to work

2. **Dockerfile Enhancement**
   - **Current State**: Generates conf.yaml from environment variables
   - **Proposed Changes**: Add peers.yaml generation capability
   - **Impact Assessment**: Minimal impact, adds new functionality without breaking existing
   - **Migration Strategy**: New environment variable is optional

#### Bug Fixes & Technical Improvements
- Improve error handling for network-related container operations
- Add better timeout handling for connection establishment
- Enhance logging for network debugging during tests

#### User Interface Requirements
- Command-line test execution with clear success/failure reporting
- Detailed logging of test steps and verification results
- Clear error messages for network-related failures

### Non-Functional Requirements
#### Performance
- Test execution time: < 60 seconds for 2-node P2P connection test
- Container startup time: < 30 seconds for both nodes combined
- Log analysis response time: < 5 seconds for pattern matching

#### Security
- No security requirements for this iteration (testing infrastructure)
- Secure handling of container access and cleanup

#### Scalability
- Framework should support extension to multi-node scenarios
- Methods should be efficient for log analysis and connection checking

#### Reliability
- Test suite reliability: > 95% pass rate across multiple runs
- Connection detection accuracy: 100%
- Proper cleanup of Docker resources after test completion

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Docker-based test cluster with individual node management
- **Proposed Changes**: Add network analysis and log processing capabilities
- **Backwards Compatibility**: All existing functionality preserved, new methods added
- **Migration Requirements**: No migration needed, new functionality is additive

### Technology Stack Updates
#### New Technologies/Libraries
- Enhanced Docker container introspection for network analysis
- Log parsing and pattern matching capabilities
- No external dependencies added

#### Version Updates
- No version updates required for this iteration

### Integration Requirements
#### New Integrations
- Direct integration with btc-federation P2P network stack
- TCP connection analysis through container networking
- Log file analysis within Docker containers

#### Modified Integrations
- Enhanced environment variable system for peer configuration
- Extended container configuration capabilities

### Data Requirements
#### Data Models
Key entities and their relationships:
- **TCPConnection**: Represents network connection state
  - Source IP/Port
  - Destination IP/Port  
  - Connection state
- **PeerConfiguration**: Represents peer settings
  - Public key (libp2p cryptographic public key)
  - Addresses
  - Connection parameters
- **KeyPair**: Represents cryptographic key pair
  - Private key (base64-encoded ECDSA private key)
  - Public key (base64-encoded ECDSA public key)
  - Generated using btc-federation keygen utility

#### Data Storage
- Temporary peers.yaml files generated in containers
- Log files within containers for analysis
- No persistent storage requirements

#### Data Migration
No data migration required - new functionality only.

## Implementation Plan
### This Iteration Timeline
- **Duration**: 5-7 working days
- **Sprint Breakdown**: Single development cycle
- **Key Deliverables**: 
  - Enhanced cluster.go and node.go modules (Day 1-2)
  - Updated Dockerfile with peers.yaml support (Day 2-3)
  - P2P connection test implementation (Day 3-5)
  - Testing and validation (Day 6-7)

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Enhanced modules | Day 2 | cluster.go and node.go extended with network analysis | Existing infrastructure | Low |
| Dockerfile update | Day 3 | Peers.yaml generation capability added | Module enhancements | Low |
| Test implementation | Day 5 | Complete P2P connection test | All previous milestones | Medium |
| Validation complete | Day 7 | All tests passing consistently | Test implementation | Medium |

### Dependencies on Other Teams/Projects
- **btc-federation project**: Network stack implementation must be stable
- **Docker infrastructure**: Reliable container management required

### Integration Points with Previous Work
- Builds directly on existing cluster and node management
- Extends existing environment variable configuration pattern
- Uses established Docker container lifecycle management

### Resource Requirements
#### Team Structure
- **Project Manager**: Dima Chizhevsky
- **Technical Lead**: Dima Chizhevsky  
- **Developers**: 1 (AI Assistant implementation)
- **Designers**: Not applicable
- **QA Engineers**: Manual validation by stakeholders

#### Budget Estimates
Development effort only - no additional infrastructure costs.

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| P2P connection timing issues | High | Medium | Implement robust wait mechanisms and retry logic |
| Docker networking complexity | Medium | Low | Use established container networking patterns |
| Log parsing reliability | Medium | Medium | Implement multiple verification methods |
| Container resource conflicts | Low | Low | Ensure proper cleanup and resource management |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Test execution time too long | Medium | Low | Optimize container startup and connection timing |
| False positive/negative results | High | Low | Implement dual verification (TCP + logs) |

## Testing Strategy
### Testing Approach for This Iteration
#### New Feature Testing
- **P2P Connection Test**: End-to-end validation with 2 nodes
- **TCP Connection Analysis**: Verification of network inspection methods
- **Log Analysis**: Pattern matching accuracy testing
- **Configuration Generation**: Peers.yaml creation validation

#### Regression Testing
- **Scope**: All existing cluster and node management functionality
- **Automated vs Manual**: Manual validation of existing tests
- **Critical User Journeys**: Container lifecycle, configuration generation, cleanup

#### Performance Testing
- **Baseline Metrics**: Current container startup time ~10 seconds
- **Target Metrics**: 2-node test completion < 60 seconds
- **Load Testing**: Single test execution only for this iteration

### Quality Gates
- All existing tests continue to pass
- New P2P connection test passes consistently (>95% reliability)
- No resource leaks or cleanup issues
- Clear success/failure reporting

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Feature addition to existing test suite
- **Rollout Strategy**: Direct integration into main branch
- **Rollback Plan**: Git revert if issues discovered

### Feature Flags & Gradual Rollout
No feature flags required - test functionality only.

### Database Migrations
No database migrations required.

### Communication Plan
- **Internal**: Code review and testing validation
- **External**: Documentation updates in test suite README
- **Documentation Updates**: Test execution instructions, network testing guide

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**: 
  - P2P connection test success rate: 100%
  - Test execution time: < 60 seconds
  - Connection detection accuracy: 100%
- **Leading Indicators**: 
  - Container startup success rate
  - TCP connection establishment timing
  - Log pattern matching accuracy
- **Baseline Values**: 
  - No existing P2P testing capability
  - Current container startup: ~10 seconds
- **Target Values**: 
  - 100% reliable P2P connection detection
  - Consistent test execution within time limits

### Monitoring Plan
- **New Dashboards/Alerts**: None required for test infrastructure
- **Enhanced Monitoring**: Test execution logging and reporting
- **A/B Testing**: Not applicable

### Review Schedule
- **Daily**: Development progress review
- **Weekly**: Not applicable for short iteration
- **Post-Release Review**: Validation of test reliability and effectiveness

## Appendices
### Glossary
- **P2P**: Peer-to-peer networking
- **libp2p**: Modular networking stack for decentralized applications
- **TCP**: Transmission Control Protocol
- **peers.yaml**: Configuration file defining peer connection information

### References
- [BTC Federation P2P Network Stack PRD](../btc-federation/02_p2p_network_stack.md)
- [Test Suite Foundation PRD](./01-test-suite-foundation.md)
- btc-federation project network implementation

### Wireframes/Mockups
Not applicable for testing infrastructure.

### API Documentation
#### Enhanced cluster.go Methods
```go
// CheckTCPConnections analyzes TCP connections in container
func (c *Cluster) CheckTCPConnections(containerID string) ([]TCPConnection, error)

// GetActiveConnectionsCount returns count of active TCP connections
func (c *Cluster) GetActiveConnectionsCount(containerID string) (int, error)
```

#### Enhanced node.go Methods  
```go
// CheckLogContains searches for pattern in node logs
func (n *Node) CheckLogContains(pattern string) (bool, error)

// GetPeersConfig returns current peers configuration
func (n *Node) GetPeersConfig() (map[string]interface{}, error)
```

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-01-27 | AI Assistant | Initial draft for P2P connection testing | Phase 1 |

**Related Documents**
- **Master Project Vision**: BTC Federation Test Suite
- **Previous Iteration PRD**: [02-log-rotation-testing.md](./02-log-rotation-testing.md)
- **Technical Architecture**: Docker-based testing infrastructure
- **User Research**: Developer feedback on network testing needs