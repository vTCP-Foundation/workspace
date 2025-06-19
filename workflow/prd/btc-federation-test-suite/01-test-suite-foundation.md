# PRD-01 BTC Federation Test Suite Foundation

## Document Information
- **Project Name**: BTC Federation Test Suite
- **Phase/Iteration**: Phase 1 - Foundation Setup
- **Document Version**: 1.0
- **Date**: 2025-01-27
- **Author(s)**: Development Team
- **Stakeholders**: BTC Federation Development Team, QA Team
- **Status**: Draft
- **Previous PRD**: N/A (Initial PRD)
- **Related Documents**: vTCP Test Suite Reference Implementation

## Executive Summary
This iteration establishes the foundational test suite infrastructure for BTC federation nodes, enabling automated testing of multi-node federation scenarios in isolated Docker environments. The test suite will provide the core framework for running BTC federation nodes in containerized environments with configurable network conditions and comprehensive testing capabilities.

## Iteration Context
### Previous Iterations Summary
- **Completed Features**: N/A (Initial implementation)
- **Lessons Learned**: N/A
- **Technical Debt**: N/A
- **User Feedback**: N/A

### Current State Analysis
- **What's working well**: N/A (Starting from scratch)
- **Pain points identified**: Manual testing of BTC federation nodes is time-consuming and error-prone
- **Performance metrics**: N/A (Baseline to be established)

## Problem Statement
### Background
Currently, testing BTC federation nodes requires manual setup and coordination of multiple node instances, making it difficult to consistently test federation scenarios, network conditions, and edge cases.

### Problem Description
The lack of automated testing infrastructure for BTC federation nodes creates several challenges:
- Manual node setup is time-consuming and error-prone
- Difficult to simulate various network conditions and failure scenarios
- Inconsistent testing environments across development and CI/CD
- Limited ability to test multi-node federation scenarios systematically

### Success Metrics
- **Primary KPIs**: 
  - Successful automated startup of multiple BTC federation nodes in isolated containers
  - Ability to run basic federation tests with 2+ nodes
  - Test execution time under 30 seconds for basic scenarios
- **Secondary KPIs**: 
  - Test suite setup time under 5 minutes
  - Docker image build success rate > 95%
- **Target Values**: 
  - 100% success rate for 2-node federation startup
  - Zero manual configuration required for basic test execution

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- Docker-based test environment with support for Manjaro and Ubuntu distributions
- Configurable BTC federation node deployment with YAML configuration
- Basic cluster management capabilities for multi-node testing
- Foundation test case demonstrating 2-node federation startup
- Go-based test framework with proper module structure

#### Bug Fixes & Technical Improvements
- N/A (Initial implementation)

#### Modifications to Existing Features
- N/A (Initial implementation)

### Explicitly Out of Scope
- Advanced network simulation and fault injection (deferred to future iterations)
- Performance benchmarking and metrics collection (deferred to future iterations)
- Integration with CI/CD pipelines (deferred to future iterations)
- Advanced test scenarios beyond basic node startup (deferred to future iterations)

### Dependencies from Previous Iterations
- BTC federation node binary must be available and functional

### Future Roadmap Impact
This foundation enables future iterations to focus on complex testing scenarios, performance analysis, and advanced network condition simulation.

## User Stories & Requirements

### User Personas
#### Primary User: BTC Federation Developer
- **Role**: Software Developer working on BTC federation protocol
- **Goals**: Test federation node behavior in various scenarios
- **Pain Points**: Manual testing setup, inconsistent environments
- **Technical Proficiency**: Advanced

#### Secondary User: QA Engineer
- **Role**: Quality Assurance Engineer
- **Goals**: Validate federation node functionality and edge cases
- **Pain Points**: Complex test environment setup, limited automation
- **Technical Proficiency**: Intermediate

### Functional Requirements
#### New Features for This Iteration
1. **Docker Container Support**
   - **Description**: Create containerized environments for BTC federation nodes based on vtcpd-test-suite Dockerfile
   - **User Story**: As a developer, I want to run BTC federation nodes in isolated Docker containers so that I can test federation scenarios without environment conflicts
   - **Rationale**: Provides consistent, reproducible test environments following proven vtcpd-test-suite patterns
   - **Builds Upon**: vtcpd-test-suite Dockerfile architecture
   - **Acceptance Criteria**: 
     - Dockerfile copied and adapted from [vtcpd-test-suite/Dockerfile](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/Dockerfile)
     - Docker containers can be built for Manjaro and Ubuntu distributions
     - Containers include BTC federation node binary from `deps/btc-federation-node`
     - Configuration is generated from PRIVATE_KEY, IP_ADDRESS, and PORT environment variables
     - Container startup process adapted for single binary execution
   - **Priority**: High
   - **Dependencies**: BTC federation node binary availability, vtcpd-test-suite reference code

2. **Configuration Management**
   - **Description**: Generate YAML configuration files for federation nodes
   - **User Story**: As a developer, I want node configurations to be automatically generated so that I can focus on testing rather than manual setup
   - **Rationale**: Eliminates manual configuration errors and enables parameterized testing
   - **Builds Upon**: Docker Container Support
   - **Acceptance Criteria**: 
     - conf.yaml generated with configurable private_key, IP address, and port
     - Configuration supports network addresses and peer exchange settings
     - Logging configuration included with appropriate levels
   - **Priority**: High
   - **Dependencies**: Docker Container Support

3. **Cluster Management**
   - **Description**: Programmatic management of multi-node federation clusters adapted from vtcpd-test-suite cluster.go
   - **User Story**: As a developer, I want to programmatically start and manage multiple federation nodes so that I can test federation scenarios efficiently
   - **Rationale**: Enables automated multi-node testing scenarios using proven vtcpd-test-suite patterns
   - **Builds Upon**: Configuration Management, vtcpd-test-suite cluster architecture
   - **Acceptance Criteria**: 
     - Code adapted from [vtcpd-test-suite/pkg/testsuite/cluster.go](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/pkg/testsuite/cluster.go)
     - NewCluster method creates cluster management instances
     - RunNode and RunNodes methods start individual and multiple BTC federation nodes
     - ConfigureNetworkConditions and RemoveNetworkConditions methods implemented
     - Container configuration modified for single BTC federation binary
   - **Priority**: High
   - **Dependencies**: Configuration Management, vtcpd-test-suite reference code

4. **Basic Test Framework**
   - **Description**: Go-based test framework with basic federation test following vtcpd-test-suite patterns
   - **User Story**: As a developer, I want a working test example so that I can understand how to create additional tests
   - **Rationale**: Provides template and verification of framework functionality based on vtcpd-test-suite architecture
   - **Builds Upon**: Cluster Management, vtcpd-test-suite test patterns
   - **Acceptance Criteria**: 
     - Test patterns adapted from vtcpd-test-suite test structure
     - Test successfully starts 2 BTC federation nodes using cluster management
     - Test runs for exactly 5 seconds and cleanly terminates
     - Test can be executed via standard Go testing tools
     - Proper error handling and cleanup implemented
   - **Priority**: Medium
   - **Dependencies**: Cluster Management, vtcpd-test-suite reference code

#### User Interface Requirements
- Command-line interface via Makefile commands
- Go test framework integration
- Docker CLI compatibility

### Non-Functional Requirements
#### Performance
- Container startup time < 10 seconds per node
- Test framework initialization < 5 seconds
- Memory usage < 512MB per container

#### Security
- Isolated container environments
- Configurable private keys for testing
- No sensitive data in container images

#### Scalability
- Support for minimum 2 nodes, extensible to more
- Resource-efficient container design
- Modular test framework architecture

#### Reliability
- Graceful container shutdown and cleanup
- Error handling for container failures
- Deterministic test execution

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: N/A (New project)
- **Proposed Changes**: Create modular Go-based test framework with Docker integration
- **Backwards Compatibility**: N/A
- **Migration Requirements**: N/A

### Technology Stack Updates
#### New Technologies/Libraries
- Docker: Containerization platform for node isolation
- Go: Primary development language for test framework
- YAML: Configuration file format

#### Version Updates
- Go: Latest stable version
- Docker: Compatible with target OS distributions

### Integration Requirements
#### New Integrations
- Docker Engine: Container management and execution
- BTC Federation Node Binary: Target application under test

#### Code Adaptation Requirements
- **Primary Reference**: All implementation must be based on and adapted from [vtcpd-test-suite](https://github.com/vTCP-Foundation/vtcpd-test-suite)
- **Adaptation Strategy**: Copy relevant code patterns and structures, then adapt for BTC federation node specifics
- **Maintained Compatibility**: Preserve architectural patterns and interface designs from the reference implementation

### Detailed File Specifications

#### 1. Dockerfile
- **Location**: `./Dockerfile`
- **Reference**: Based on [vtcpd-test-suite/Dockerfile](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/Dockerfile)
- **Requirements**:
  - Support for both Manjaro and Ubuntu distributions
  - Environment variables for PRIVATE_KEY, IP_ADDRESS, and PORT
  - Binary location: `/btc-federation-node` (copied from `deps/btc-federation-node`)
  - YAML configuration generation during container startup
  - Working directory setup with `conf.yaml` alongside binary

#### 2. Makefile.example
- **Location**: `./Makefile.example`
- **Reference**: Based on [vtcpd-test-suite/Makefile.example](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/Makefile.example)
- **Requirements**:
  - Docker build commands for test images
  - Test execution commands
  - Binary path configuration: `deps/btc-federation-node`
  - Single binary dependency (unlike vtcpd-test-suite's multiple binaries)
  - Environment variable configuration for node parameters

#### 3. pkg/testsuite/cluster.go
- **Location**: `./pkg/testsuite/cluster.go`
- **Reference**: Adapted from [vtcpd-test-suite/pkg/testsuite/cluster.go](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/pkg/testsuite/cluster.go)
- **Required Methods**:
  - `NewCluster()` - Create cluster management instance
  - `RunNode()` - Start single BTC federation node
  - `RunNodes()` - Start multiple BTC federation nodes
  - `ConfigureNetworkConditions()` - Apply network conditions to cluster
  - `RemoveNetworkConditions()` - Remove network conditions from cluster
- **Adaptation Notes**: Modify container configuration for single binary execution

#### 4. pkg/testsuite/node.go
- **Location**: `./pkg/testsuite/node.go`
- **Reference**: Adapted from [vtcpd-test-suite/pkg/testsuite/node.go](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/pkg/testsuite/node.go)
- **Required Methods**:
  - `NewNode()` - Create node instance with configuration
- **Configuration**: Support for BTC federation node parameters (private_key, network addresses, peers config)

#### 5. test/common/common_test.go
- **Location**: `./test/common/common_test.go`
- **Reference**: Based on vtcpd-test-suite test patterns
- **Requirements**:
  - Test function that starts 2 BTC federation nodes
  - 5-second execution duration
  - Clean shutdown and cleanup
  - Proper error handling and assertions

#### 6. README.md
- **Location**: `./README.md`
- **Reference**: Based on [vtcpd-test-suite/readme.md](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/readme.md)
- **Requirements**:
  - Overview section adapted for BTC federation testing
  - Prerequisites and setup instructions
  - Build and run instructions
  - Directory structure documentation
  - Troubleshooting section

#### 7. Go Module Files
- **go.mod Location**: `./go.mod`
- **go.sum Location**: `./go.sum`
- **Requirements**:
  - Module name: `btc-federation-test-suite`
  - Go version: Latest stable
  - Dependencies: Docker client, YAML processing, testing utilities

#### 8. Configuration Template
- **conf.yaml Template** (generated in container):
```yaml
node:
    private_key: ${PRIVATE_KEY}
network:
    addresses:
        - /ip4/${IP_ADDRESS}/tcp/${PORT}
peers:
    exchange_interval: 30s
    connection_timeout: 10s
logging:
    level: info
    format: json
```

### Data Requirements
#### Data Models
- Node configuration structure (YAML format)
- Cluster state management  
- Test execution metadata

#### Data Storage
- Configuration files (temporary, container lifecycle)
- Test logs and results
- No persistent data storage required

## Implementation Plan
### This Iteration Timeline
- **Duration**: 1 week
- **Key Deliverables**: Complete test suite foundation with working 2-node test

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Docker Infrastructure | Day 2 | Dockerfile and container build system | BTC federation binary | Medium |
| Configuration System | Day 3 | YAML config generation and management | Docker Infrastructure | Low |
| Go Framework | Day 5 | Core Go modules and cluster management | Configuration System | Medium |
| Basic Test | Day 6 | Working 2-node federation test | Go Framework | Low |
| Documentation | Day 7 | README and setup documentation | Basic Test | Low |

### Dependencies on Other Teams/Projects
- BTC federation node binary must be provided by development team
- Docker runtime environment on target systems

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer
- **Developers**: 1-2 developers
- **QA Engineers**: 1 engineer for validation

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| BTC federation binary compatibility issues | High | Medium | Early testing with provided binary, fallback to stub implementation |
| Docker environment inconsistencies | Medium | Low | Standardize base images, thorough testing on target platforms |
| Go module dependency conflicts | Low | Low | Use stable dependency versions, proper dependency management |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Delayed binary availability | High | Medium | Coordinate closely with development team, create timeline buffers |

## Testing Strategy
### Testing Approach for This Iteration
#### New Feature Testing
- Unit testing for configuration generation
- Integration testing for Docker container functionality
- End-to-end testing for complete test framework workflow

#### Performance Testing
- **Baseline Metrics**: Container startup time, memory usage
- **Target Metrics**: <10s startup, <512MB memory per container

### Quality Gates
- All Docker containers build successfully on target platforms
- Basic 2-node test executes successfully
- Documentation is complete and accurate
- Code passes Go linting and formatting standards

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Initial release (v1.0.0)
- **Rollout Strategy**: Direct deployment to development environments
- **Rollback Plan**: Revert to manual testing processes if framework fails

### Communication Plan
- **Internal**: Notify development team of test suite availability
- **Documentation Updates**: Create comprehensive README and setup guides

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**: Successful execution of 2-node federation test
- **Target Values**: 100% success rate for basic test execution
- **Baseline Values**: N/A (Initial implementation)

### Review Schedule
- **Daily**: Development progress and blocker identification
- **Weekly**: N/A (1-week iteration)
- **Post-Release Review**: Framework effectiveness evaluation after 1 week of usage

## Appendices
### Glossary
- **BTC Federation**: Bitcoin federation protocol implementation
- **Docker Container**: Isolated runtime environment for applications
- **Go Module**: Go language package management unit
- **YAML**: Human-readable data serialization standard

### References
- vTCP Test Suite Implementation: https://github.com/vTCP-Foundation/vtcpd-test-suite
- Docker Documentation: https://docs.docker.com/
- Go Testing Documentation: https://golang.org/pkg/testing/

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-01-27 | Development Team | Initial PRD for test suite foundation | Phase 1 |

**Related Documents**
- **Task List**: workflow/tasks/01/tasks.md

## Overview
This PRD establishes the foundational infrastructure for automated testing of BTC federation nodes in containerized environments, based on the vTCP test suite reference implementation.

## Business Justification
Manual testing of BTC federation nodes creates bottlenecks in development cycles and introduces risks from inconsistent test environments. An automated test suite foundation will:
- Reduce manual testing effort by 80%
- Eliminate environment-related testing inconsistencies
- Enable systematic testing of multi-node federation scenarios
- Provide a scalable foundation for complex testing scenarios

## Conditions of Satisfaction (CoS)
1. **Dockerfile Implementation**: 
   - Dockerfile builds successfully for both Manjaro and Ubuntu distributions
   - Container includes BTC federation node binary from `deps/btc-federation-node`
   - YAML configuration is generated from PRIVATE_KEY, IP_ADDRESS, and PORT environment variables
   - Binary and conf.yaml are properly positioned in container

2. **Makefile.example Creation**:
   - All commands from vtcpd-test-suite adapted for BTC federation single binary
   - Docker build and test execution commands functional
   - Proper dependency management for `deps/btc-federation-node`

3. **Go Framework Implementation**:
   - `pkg/testsuite/cluster.go` with all required methods (NewCluster, RunNode, RunNodes, ConfigureNetworkConditions, RemoveNetworkConditions)
   - `pkg/testsuite/node.go` with NewNode method
   - Code adapted from vtcpd-test-suite with BTC federation-specific modifications

4. **Test Suite Functionality**:
   - `test/common/common_test.go` successfully starts 2 BTC federation nodes
   - Test runs for exactly 5 seconds and terminates cleanly
   - Proper test assertions and error handling implemented

5. **Project Infrastructure**:
   - `go.mod` and `go.sum` files with correct module name and dependencies
   - Directory structure matches project layout requirements
   - All files reference and adapt code from vtcpd-test-suite repository

6. **Documentation Completion**:
   - `README.md` adapted from vtcpd-test-suite with BTC federation-specific instructions
   - Setup, build, and usage instructions complete and accurate
   - Reference to vtcpd-test-suite source clearly documented

## Scope and Boundaries
### In Scope
- Docker containerization for Manjaro and Ubuntu distributions
- YAML configuration management system
- Go-based cluster management framework
- Basic 2-node federation test
- Project infrastructure (go.mod, Makefile.example, README.md)

### Out of Scope (Future Iterations)
- Advanced network simulation and fault injection
- Performance benchmarking and metrics collection
- CI/CD pipeline integration
- Complex multi-scenario testing

## Dependencies
- BTC federation node binary must be available at deps/btc-federation-node
- Docker runtime environment on target systems
- Go development environment (latest stable version)

## Assumptions and Constraints
### Assumptions
- BTC federation node binary is compatible with containerized deployment
- Standard Docker networking is sufficient for basic federation testing
- Go standard library and minimal external dependencies are preferred

### Constraints
- Must follow vTCP test suite architectural patterns
- Container resource usage limited to 512MB per node
- Test execution time should not exceed 30 seconds for basic scenarios

## Task List
Reference: `workflow/tasks/01/tasks.md` 