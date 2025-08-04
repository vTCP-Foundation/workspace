# 05 Configuration File Testing

## Document Information
- **Project Name**: BTC Federation Test Suite
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.0
- **Date**: 2024-12-19
- **Author(s)**: AI Agent
- **Stakeholders**: Development Team
- **Status**: Draft
- **Previous PRD**: N/A
- **Related Documents**: [btc-federation config.go](../../btc-federation/internal/config/config.go), [btc-federation peer.go](../../btc-federation/internal/storage/peer.go)

## Executive Summary
This PRD defines comprehensive testing for BTC Federation node configuration files (`conf.yaml` and `peers.yaml`) validation and error handling. The testing will verify that the node correctly handles various configuration scenarios including missing files, invalid formats, incorrect values, and edge cases. Tests will run btc-federation nodes in Docker containers with prepared configuration files to validate behavior and exit codes.

### Current project state
- BTC Federation node has configuration validation logic in `internal/config/config.go` and `internal/storage/peer.go`
- Docker infrastructure exists for running nodes in containers
- Test suite framework is available with cluster management

### This iteration's focus
Create comprehensive test coverage for configuration file validation including 20 test cases for `conf.yaml` and 3 test cases for `peers.yaml` validation scenarios.

### Connection to overall vision
This testing ensures robust configuration handling for the BTC Federation network, preventing runtime failures due to configuration errors.

## Iteration Context
### Previous Iterations Summary
- **Completed Features**: Basic test suite infrastructure, Docker containerization, P2P connection testing
- **Lessons Learned**: Container-based testing provides reliable isolation for configuration testing
- **Technical Debt**: None relevant to this iteration
- **User Feedback**: Need for comprehensive configuration validation testing

### Current State Analysis
- **What's working well**: Docker test infrastructure, cluster management
- **Pain points identified**: Lack of systematic configuration validation testing
- **Performance metrics**: N/A for this iteration

## Problem Statement
### Background
The BTC Federation node relies on configuration files (`conf.yaml` and `peers.yaml`) for proper operation. Invalid configurations can cause runtime failures, startup issues, or incorrect behavior. Currently, there is no comprehensive test coverage for configuration validation scenarios.

### Problem Description
Without systematic testing of configuration file validation, the following issues can occur:
- Nodes failing to start with unclear error messages
- Incorrect behavior due to malformed configurations
- Lack of confidence in configuration validation logic
- Difficult debugging when configuration issues arise

This affects developers who need to ensure robust configuration handling and users who need reliable node startup.

### Success Metrics
- **Primary KPIs**: 100% of defined configuration test scenarios pass
- **Secondary KPIs**: All invalid configurations properly exit with code 1, valid configurations start successfully
- **Target Values**: 23 automated test cases (20 for conf.yaml + 3 for peers.yaml)

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- Configuration file validation test suite for `conf.yaml` (20 test cases)
- Configuration file validation test suite for `peers.yaml` (3 test cases)
- Docker container-based testing infrastructure for configuration scenarios
- Test utilities for creating temporary configuration files and validating exit codes

#### Test Cases for conf.yaml
1. Missing `conf.yaml` - should generate default config and start successfully (exit code 0)
2. Empty `conf.yaml` - should exit with code 1
3. Invalid YAML structure (malformed YAML) - should exit with code 1
4. Missing `private_key` field - should generate key, save to file, and start successfully (exit code 0)
5. Invalid `private_key` format - should exit with code 1
6. Missing `network` section - should exit with code 1
7. Missing `network.addresses` field - should exit with code 1
8. Empty `network.addresses` - should exit with code 1
9. Invalid address format in `network.addresses` - should exit with code 1
10. Missing `peers` section - should exit with code 1
11. Missing `peers.connection_timeout` field - should exit with code 1
12. Invalid `peers.connection_timeout` value - should exit with code 1
13. Missing `logging` section - should exit with code 1
14. Missing `logging.level` field - should exit with code 1
15. Invalid `logging.level` value - should exit with code 1
16. Missing `logging.format` field - should exit with code 1
17. Invalid `logging.format` value - should exit with code 1
18. Invalid boolean values in configuration fields - should exit with code 1
19. Missing `logging.file_name` when `logging.file_output=true` - should exit with code 1
20. Invalid `logging.file_max_size` format - should exit with code 1

#### Test Cases for peers.yaml
1. Missing `peers.yaml` - should start successfully with log message "No peers configured - running in standalone mode"
2. Empty `peers.yaml` - should exit with code 1
3. Empty `peers` value in `peers.yaml` - should start successfully with log message "No peers configured - running in standalone mode"

### Explicitly Out of Scope
- Performance testing of configuration loading
- Configuration hot-reloading testing
- Complex multi-node configuration scenarios
- Configuration file format migration testing

### Dependencies from Previous Iterations
- Docker test infrastructure
- Cluster management utilities
- Node creation and management functionality

### Future Roadmap Impact
This testing framework will serve as foundation for more complex configuration scenarios and integration testing.

## User Stories & Requirements

### User Personas
#### Primary User: BTC Federation Developer
- **Role**: Backend developer working on BTC Federation
- **Goals**: Ensure configuration validation works correctly across all scenarios
- **Pain Points**: Uncertain behavior with invalid configurations, debugging configuration issues
- **Technical Proficiency**: Advanced

#### Secondary User: DevOps Engineer
- **Role**: Infrastructure and deployment management
- **Goals**: Reliable node startup and clear error reporting
- **Pain Points**: Unclear configuration error messages, difficult troubleshooting
- **Technical Proficiency**: Advanced

### Functional Requirements
#### New Features for This Iteration
1. **Configuration Test Suite for conf.yaml**
   - **Description**: Comprehensive test coverage for conf.yaml validation scenarios
   - **User Story**: As a developer, I want to verify that all conf.yaml validation scenarios work correctly so that I can trust the configuration system
   - **Rationale**: Ensures robust configuration handling and prevents runtime failures
   - **Builds Upon**: Existing configuration validation in config.go
   - **Acceptance Criteria**: 
     - All 20 test scenarios for conf.yaml execute successfully
     - Invalid configurations exit with code 1
     - Valid configurations start successfully (exit code 0)
     - Generated files (when applicable) contain correct content
   - **Priority**: High
   - **Dependencies**: Docker test infrastructure

2. **Configuration Test Suite for peers.yaml**
   - **Description**: Test coverage for peers.yaml validation scenarios
   - **User Story**: As a developer, I want to verify that peers.yaml validation works correctly so that peer configuration errors are handled properly
   - **Rationale**: Ensures proper peer configuration handling and standalone mode operation
   - **Builds Upon**: Existing peer storage validation in peer.go
   - **Acceptance Criteria**: 
     - All 3 test scenarios for peers.yaml execute successfully
     - Standalone mode logging message appears when appropriate
     - Invalid configurations exit with code 1
   - **Priority**: High
   - **Dependencies**: Docker test infrastructure, logging system

3. **Test Infrastructure Enhancements**
   - **Description**: Utilities for configuration file testing leveraging existing cluster.go and node.go methods
   - **User Story**: As a developer, I want reusable test utilities for configuration testing so that I can easily create and validate configuration scenarios
   - **Rationale**: Reduces code duplication and improves test maintainability by reusing existing infrastructure
   - **Builds Upon**: Existing testsuite package (cluster.go and node.go)
   - **Acceptance Criteria**: 
     - Helper functions for creating temporary configuration files using node.go patterns
     - Utilities for validating exit codes using cluster.RunConfigValidation method
     - Container configuration management using cluster.NewCluster and cluster.RunConfigValidation
     - Integration with existing cluster.CheckNodeLogContains method for log validation
     - Extension of cluster.go with new methods for configuration validation scenarios if needed
     - Extension of node.go with configuration file generation utilities if needed
   - **Priority**: Medium
   - **Dependencies**: testsuite package (cluster.go, node.go)

### Non-Functional Requirements
#### Performance
- Test execution should complete within 5 minutes for all scenarios
- Container startup time should not exceed 30 seconds per test case

#### Reliability
- Tests should be deterministic and not depend on external resources
- Test cleanup should always occur regardless of test outcome

#### Maintainability
- Test code should follow existing project patterns
- Configuration scenarios should be easily extensible

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Configuration validation in config.go and peer.go modules
- **Proposed Changes**: Addition of comprehensive test coverage using Docker containers
- **Backwards Compatibility**: No changes to existing configuration logic
- **Migration Requirements**: None

### Technology Stack Updates
#### New Technologies/Libraries
- Go testing framework (existing)
- Docker containerization for test isolation (existing)
- YAML manipulation for test file generation (existing)
- Existing testsuite cluster methods for container management and log validation

#### Integration with Existing Infrastructure
The configuration testing implementation must leverage existing methods from the testsuite package to ensure consistency and reusability:

**From cluster.go:**
- `NewCluster(config *ClusterConfig)` - Create cluster instance for test management
- `RunConfigValidation(ctx context.Context, node *Node)` - Run configuration validation containers
- `CheckNodeLogContains(containerID string, pattern string)` - Validate log messages for test scenarios
- `ExecInContainer(containerID string, cmd []string)` - Execute commands within test containers
- `Cleanup()` - Proper cleanup of test resources

**From node.go:**
- `NewNode(config *NodeConfig)` - Create node instances with test configurations
- `GetEnvironmentVariables()` - Generate environment variables for container execution
- `GetLabels()` - Apply consistent labeling for test containers

**Extension Requirements:**
When existing methods are insufficient, extend functionality by:
- Adding new methods to cluster.go for specific configuration validation needs
- Extending node.go with configuration file generation utilities
- Maintaining compatibility with existing method signatures
- Following established patterns for error handling and resource management

### Integration Requirements
#### New Integrations
- Test files integration with Docker container file system
- Log message validation using existing CheckNodeLogContains method from cluster.go
- Exit code validation through Docker container status monitoring

### Data Requirements
#### Data Models
Test configuration files with various valid and invalid scenarios

#### Data Storage
Temporary test configuration files created during test execution

## Implementation Plan
### This Iteration Timeline
- **Duration**: 3-5 days
- **Key Deliverables**: 
  - `test/conf/conf_test.go` with 20 test cases using cluster.RunConfigValidation
  - `test/conf/peers_test.go` with 3 test cases using cluster.CheckNodeLogContains
  - Test utility functions leveraging existing cluster.go and node.go methods
  - Extensions to cluster.go and node.go methods if required for configuration validation

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Test Infrastructure Setup | Day 1 | Create test directory structure and basic utilities | Docker infrastructure | Low |
| conf.yaml Test Implementation | Day 2-3 | Implement all 20 conf.yaml test scenarios | Test infrastructure | Medium |
| peers.yaml Test Implementation | Day 4 | Implement all 3 peers.yaml test scenarios | Test infrastructure | Low |
| Integration and Validation | Day 5 | End-to-end testing and validation | All tests implemented | Low |

### Dependencies on Other Teams/Projects
- None external dependencies

### Resource Requirements
#### Team Structure
- **Developer**: 1 person for implementation
- **Reviewer**: 1 person for code review

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Docker container startup failures | Medium | Low | Use existing proven container patterns |
| Configuration file race conditions | Low | Low | Use temporary directories with unique names |
| Test execution timeout | Low | Medium | Set appropriate timeouts and cleanup procedures |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Test implementation time overrun | Low | Low | Break down into smaller, manageable test cases |

## Testing Strategy
### Testing Approach for This Iteration
#### New Feature Testing
- Unit testing for configuration file creation utilities
- Integration testing with Docker containers for full scenario validation
- Exit code and log message validation

#### Regression Testing
- **Scope**: Existing configuration validation should continue to work
- **Automated vs Manual**: Fully automated through Go test framework
- **Critical User Journeys**: Node startup with various configuration scenarios

#### Performance Testing
- **Baseline Metrics**: Current container startup time
- **Target Metrics**: Test suite completion within 5 minutes

### Quality Gates
- All 23 test cases must pass
- No test flakiness (tests must be deterministic)
- Code coverage for new test utilities must be >90%
- Proper integration with existing cluster methods

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Minor addition to test suite
- **Rollout Strategy**: Direct integration into CI/CD pipeline
- **Rollback Plan**: Remove new test files if issues arise

### Communication Plan
- **Internal**: Update team on new test coverage capabilities
- **Documentation Updates**: Update test documentation with new scenarios

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**: 
  - 23/23 test cases implemented and passing
  - 100% scenario coverage for defined configuration validation
- **Target Values**: All tests green in CI/CD pipeline

### Monitoring Plan
- **Test Execution Monitoring**: CI/CD pipeline integration
- **Failure Analysis**: Detailed logging for failed test scenarios

## Conditions of Satisfaction (CoS)
1. **Complete Test Coverage**: All 20 conf.yaml test scenarios are implemented and passing
2. **Complete Test Coverage**: All 3 peers.yaml test scenarios are implemented and passing
3. **Correct Exit Codes**: Invalid configurations properly exit with code 1
4. **Correct Success Cases**: Valid configurations start successfully (exit code 0)
5. **Log Message Validation**: Standalone mode message appears in appropriate scenarios using existing CheckNodeLogContains method
6. **Test File Organization**: Tests are properly organized in `test/conf/conf_test.go` and `test/conf/peers_test.go`
7. **Code Quality**: Test code follows project patterns and leverages existing cluster methods
8. **Documentation**: Test scenarios are clearly documented and understandable

## Scope and Boundaries
### Included in This PRD
- Configuration file validation testing for conf.yaml and peers.yaml
- Docker container-based test execution
- Exit code and log message validation
- Test utility functions for configuration file manipulation

### Explicitly Excluded
- Performance benchmarking of configuration loading
- Configuration hot-reload testing
- Multi-node configuration scenarios
- Configuration file format migration

## Dependencies
- Docker test infrastructure (existing)
- btc-federation binary compilation (existing)
- testsuite package functionality (existing)

## Assumptions and Constraints
### Key Assumptions
- Docker environment is available for test execution
- btc-federation binary behaves consistently across test runs
- Configuration validation logic in config.go and peer.go is stable

### Constraints
- Tests must use existing Docker infrastructure
- Test execution time should be reasonable (< 5 minutes total)
- No modification of existing configuration validation logic

## Implementation Guidelines

### Required Methods Usage
All implementation tasks must use existing infrastructure methods from testsuite package:

**Mandatory cluster.go methods:**
- `cluster.NewCluster(config *ClusterConfig)` - Initialize cluster for tests
- `cluster.RunConfigValidation(ctx context.Context, node *Node)` - Run configuration validation containers
- `cluster.CheckNodeLogContains(containerID string, pattern string)` - Validate log messages
- `cluster.ExecInContainer(containerID string, cmd []string)` - Execute commands in containers
- `cluster.Cleanup()` - Clean up test resources

**Mandatory node.go methods:**
- `node.NewNode(config *NodeConfig)` - Create test node configurations
- `node.GetEnvironmentVariables()` - Generate container environment variables
- `node.GetLabels()` - Apply container labels

### Extension Policy
When existing methods are insufficient:
1. **First Priority**: Extend existing methods in cluster.go or node.go
2. **Second Priority**: Add new methods following existing patterns
3. **Maintain Compatibility**: Never break existing method signatures
4. **Follow Patterns**: Use established error handling and resource management patterns

### Code Reuse Requirements
- All test implementations must reuse existing infrastructure
- No duplicate functionality between test files and existing cluster/node code
- Follow DRY principles for constants and repeated operations
- Maintain consistency with existing test patterns

## Task List
Reference to the task list file for this PRD: [workflow/tasks/btc-federation-test-suite/05/tasks.md](../../tasks/btc-federation-test-suite/05/tasks.md) 