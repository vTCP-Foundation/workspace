# 01-1 - BTC Federation Test Suite Foundation Implementation

# Links
- [PRD-01 BTC Federation Test Suite Foundation](../../../prd/btc-federation-test-suite/01-test-suite-foundation.md)
- [Reference Implementation: vtcpd-test-suite](https://github.com/vTCP-Foundation/vtcpd-test-suite)

# Description
Implement foundational test suite infrastructure for BTC federation nodes by adapting code and patterns from the [vtcpd-test-suite](https://github.com/vTCP-Foundation/vtcpd-test-suite) reference implementation. This task creates all core files needed for Docker-based automated testing of multi-node BTC federation scenarios.

# Requirements and DOD

## Core Files to Create
Based on [vtcpd-test-suite](https://github.com/vTCP-Foundation/vtcpd-test-suite) structure, the following files must be created:

### 1. Dockerfile
- **Location**: `./Dockerfile`
- **Source**: Adapt from [vtcpd-test-suite/Dockerfile](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/Dockerfile)
- **Requirements**:
  - Support Manjaro and Ubuntu distributions
  - Accept PRIVATE_KEY, IP_ADDRESS, PORT environment variables
  - Copy `deps/btc-federation-node` binary to `/btc-federation-node`
  - Generate `conf.yaml` from environment variables during startup
  - Single binary execution (vs vtcpd-test-suite's multiple binaries)

### 2. Makefile.example
- **Location**: `./Makefile.example`
- **Source**: Adapt from [vtcpd-test-suite/Makefile.example](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/Makefile.example)
- **Requirements**:
  - Docker build commands for multiple distributions
  - Test execution commands
  - Binary path: `deps/btc-federation-node` (single binary vs multiple)
  - Environment variable configuration for node parameters

### 3. pkg/testsuite/cluster.go
- **Location**: `./pkg/testsuite/cluster.go`
- **Source**: Adapt from [vtcpd-test-suite/pkg/testsuite/cluster.go](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/pkg/testsuite/cluster.go)
- **Required Methods**:
  - `NewCluster()` - Create cluster management instance
  - `RunNode()` - Start single BTC federation node
  - `RunNodes()` - Start multiple BTC federation nodes
  - `ConfigureNetworkConditions()` - Apply network conditions
  - `RemoveNetworkConditions()` - Remove network conditions
- **Adaptations**: Configure for single btc-federation-node binary

### 4. pkg/testsuite/node.go
- **Location**: `./pkg/testsuite/node.go`
- **Source**: Adapt from [vtcpd-test-suite/pkg/testsuite/node.go](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/pkg/testsuite/node.go)
- **Required Methods**:
  - `NewNode()` - Create node instance with BTC federation configuration
- **Configuration Support**: private_key, network addresses, peers config

### 5. test/common/common_test.go
- **Location**: `./test/common/common_test.go`
- **Source**: Follow vtcpd-test-suite test patterns
- **Requirements**:
  - Test function starting exactly 2 BTC federation nodes
  - 5-second execution duration
  - Clean shutdown and cleanup
  - Proper error handling and assertions

### 6. README.md
- **Location**: `./README.md`
- **Source**: Adapt from [vtcpd-test-suite/readme.md](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/readme.md)
- **Requirements**:
  - Overview adapted for BTC federation testing
  - Prerequisites, setup, build, and run instructions
  - Directory structure documentation
  - Troubleshooting section
  - Reference to vtcpd-test-suite source

### 7. Go Module Files
- **go.mod Location**: `./go.mod`
- **go.sum Location**: `./go.sum`
- **Requirements**:
  - Module name: `btc-federation-test-suite`
  - Latest stable Go version
  - Dependencies: Docker client, YAML processing, testing utilities

## Configuration Template
The `conf.yaml` template must be generated in containers:
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

## Definition of Done
- [ ] All 7 files created with proper vtcpd-test-suite code adaptation
- [ ] Dockerfile builds successfully for Manjaro and Ubuntu
- [ ] Makefile.example contains all necessary commands
- [ ] Go modules (cluster.go, node.go) implement all required methods
- [ ] Basic test executes 2 nodes for 5 seconds successfully
- [ ] README.md provides complete setup instructions
- [ ] go.mod/go.sum properly configure project dependencies
- [ ] All code references vtcpd-test-suite as source in comments
- [ ] Project structure matches policy requirements

# Implementation Plan

## Phase 1: Project Structure Setup (Day 1)
1. **Initialize Go Module**
   - Create `go.mod` with module name `btc-federation-test-suite`
   - Add initial dependencies for Docker client and YAML processing
   - Generate `go.sum` through dependency resolution

2. **Create Directory Structure**
   - `deps/` directory for btc-federation-node binary
   - `pkg/testsuite/` for Go framework code
   - `test/common/` for test files

## Phase 2: Docker Infrastructure (Day 2)
1. **Dockerfile Creation**
   - Download and analyze [vtcpd-test-suite/Dockerfile](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/Dockerfile)
   - Adapt for single btc-federation-node binary
   - Implement environment variable configuration
   - Add YAML generation logic
   - Test builds for both Manjaro and Ubuntu

2. **Makefile.example Creation**
   - Download and analyze [vtcpd-test-suite/Makefile.example](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/Makefile.example)
   - Adapt commands for single binary dependency
   - Configure Docker build and test commands
   - Add environment variable configuration

## Phase 3: Go Framework Development (Days 3-4)
1. **Node Management (pkg/testsuite/node.go)**
   - Download and analyze [vtcpd-test-suite/pkg/testsuite/node.go](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/pkg/testsuite/node.go)
   - Implement `NewNode()` method with BTC federation configuration
   - Add configuration structures for node parameters
   - Add source attribution comments

2. **Cluster Management (pkg/testsuite/cluster.go)**
   - Download and analyze [vtcpd-test-suite/pkg/testsuite/cluster.go](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/pkg/testsuite/cluster.go)
   - Implement all required methods:
     - `NewCluster()` - cluster initialization
     - `RunNode()` - single node startup
     - `RunNodes()` - multiple node startup
     - `ConfigureNetworkConditions()` - network condition setup
     - `RemoveNetworkConditions()` - network condition cleanup
   - Adapt container configuration for btc-federation-node
   - Add comprehensive error handling

## Phase 4: Test Implementation (Day 5)
1. **Basic Test Creation (test/common/common_test.go)**
   - Analyze vtcpd-test-suite test patterns
   - Implement 2-node startup test
   - Add 5-second execution timer
   - Implement clean shutdown and cleanup
   - Add proper assertions and error handling

2. **Test Validation**
   - Execute test to verify 2-node startup
   - Validate 5-second duration
   - Confirm clean termination
   - Test error scenarios

## Phase 5: Documentation (Day 6)
1. **README.md Creation**
   - Download and analyze [vtcpd-test-suite/readme.md](https://github.com/vTCP-Foundation/vtcpd-test-suite/blob/main/readme.md)
   - Adapt content for BTC federation testing
   - Update prerequisites and setup instructions
   - Document directory structure
   - Add troubleshooting section
   - Include references to vtcpd-test-suite

2. **Code Documentation**
   - Add source attribution comments to all adapted code
   - Document configuration options
   - Add inline code documentation

## Expected Outcomes
- Complete test suite foundation ready for BTC federation node testing
- All code properly attributed to vtcpd-test-suite source
- Working 2-node test demonstrating framework functionality
- Comprehensive documentation for setup and usage

# Test Plan

## Unit Testing
- **NewNode() Method**: Verify node creation with valid configurations
- **Cluster Methods**: Test each cluster management method individually
- **Configuration Generation**: Validate YAML configuration creation
- **Error Handling**: Test failure scenarios and cleanup

## Integration Testing
- **Docker Build**: Test Dockerfile builds on Manjaro and Ubuntu
- **Container Startup**: Verify containers start with proper configuration
- **Node Communication**: Basic connectivity between test nodes
- **Cleanup**: Verify proper resource cleanup after tests

## End-to-End Testing
- **Complete Test Flow**: Execute full 2-node test scenario
- **Timing Validation**: Confirm 5-second execution duration
- **Resource Usage**: Monitor container resource consumption
- **Multi-Run Stability**: Execute test multiple times for consistency

## Performance Testing
- **Container Startup Time**: Measure and validate <10s startup per node
- **Memory Usage**: Verify <512MB memory usage per container
- **Test Framework Init**: Confirm <5s framework initialization

## Success Criteria
- All unit tests pass with >95% coverage
- Integration tests execute successfully on both distributions
- End-to-end test consistently passes over 10 runs
- Performance metrics meet specified requirements
- No resource leaks detected in cleanup validation

# Verification and Validation

## Architecture integrity
- **Modular Design**: Verify separation of concerns between node, cluster, and test layers
- **vtcpd-test-suite Compatibility**: Ensure architectural patterns are preserved from reference implementation
- **Extensibility**: Confirm framework can be extended for additional test scenarios
- **Interface Consistency**: Validate consistent method signatures and return types

## Security
- **Container Isolation**: Verify proper container isolation and networking
- **Configuration Security**: Ensure no sensitive data persisted in container images
- **Environment Variables**: Validate secure handling of private keys and configuration
- **Network Security**: Confirm isolated test networks prevent external access

## Performance
- **Container Startup**: <10 seconds per node startup time achieved
- **Memory Efficiency**: <512MB memory usage per container maintained
- **Framework Initialization**: <5 seconds framework startup time confirmed
- **Test Execution**: <30 seconds total test execution time for basic scenarios

## Scalability
- **Node Scaling**: Framework supports minimum 2 nodes with extension capability
- **Resource Efficiency**: Linear resource scaling with node count
- **Container Orchestration**: Efficient Docker container management
- **Test Parallelization**: Framework design supports parallel test execution

## Reliability
- **Error Handling**: Comprehensive error handling and logging implemented
- **Cleanup Mechanisms**: Proper resource cleanup in success and failure scenarios
- **Container Recovery**: Graceful handling of container failures
- **Test Consistency**: Deterministic test execution with repeatable results

## Maintainability
- **Code Quality**: Go code follows standard formatting and linting rules
- **Documentation**: Comprehensive inline and external documentation
- **Source Attribution**: Clear references to vtcpd-test-suite source code
- **Configuration Management**: Clear separation of configuration from code

## Cost
- **Resource Optimization**: Minimal container resource usage achieved
- **Development Efficiency**: Reuse of vtcpd-test-suite patterns reduces development time
- **Infrastructure Requirements**: Standard Docker infrastructure sufficient
- **Maintenance Overhead**: Low maintenance requirements through proven patterns

## Compliance
- **Policy Adherence**: Full compliance with project policies and guidelines
- **Template Compliance**: Complete adherence to task template requirements
- **Reference Implementation**: Proper attribution and adaptation of vtcpd-test-suite code
- **Documentation Standards**: All documentation meets project standards 