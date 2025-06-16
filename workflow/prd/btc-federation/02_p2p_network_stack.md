# Project Requirements Document (PRD)

## Document Information
- **Project Name**: BTC Federation
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.1
- **Date**: 2025-06-04
- **Author(s)**: Dima Chizhevsky
- **Stakeholders**: Dima Chizhevsky, Mykola Ilashchuk
- **Status**: Active
- **Previous PRD**: n/a
- **Related Documents**: 
  - [Task 2.1: Project Foundation](../../tasks/btc-federation/2/task_2_1_project_foundation.md)
  - [Task 2.2: Network Host](../../tasks/btc-federation/2/task_2_2_network_host.md)
  - [Task 2.3: Peer Storage](../../tasks/btc-federation/2/task_2_3_peer_storage.md)
  - [Task 2.4: Peer Exchange](../../tasks/btc-federation/2/task_2_4_peer_exchange.md)
  - [Task 2.5: Config Hot-reload](../../tasks/btc-federation/2/task_2_5_config_hotreload.md)

## Executive Summary
**Objective**: Implement a basic p2p network node using libp2p that serves as the foundation for the BTC Federation protocol.

**Key Deliverables**:
- Functional p2p network node with libp2p integration
- Comprehensive test suite covering connectivity and peer management
- Configuration-driven peer management system
- Documentation and operational procedures

**Success Criteria**:
- Nodes can establish connections within 3 seconds on local network
- Peer discovery and exchange works reliably (>95% success rate)
- Configuration hot-reload works without dropping existing connections
- All automated tests pass consistently

## Problem Statement
### Background
The BTC Federation project requires a robust p2p network foundation that can:
1. Establish and maintain connections between nodes
2. Exchange peer information reliably
3. Support dynamic configuration updates
4. Provide foundation for future protocol message processing

This implementation will use Go and libp2p to create a production-ready network stack (but not focusing on actual messages processing).

### Success Metrics
- **Primary KPIs**:
    - Connection establishment time: < 3 seconds (local), < 10 seconds (internet)
    - Peer discovery success rate: > 95%
    - Configuration reload time: < 2 seconds
    - Test coverage: > 90%
    - Zero memory leaks during 10 minutes stress test

## Project Scope
### This Iteration's Scope
#### Core Features

**1. Basic Connectivity**
- Nodes can connect to each other using libp2p
- Support for IPv4/IPv6 and DNS resolution
- **TCP transport protocol implementation** (changed from QUIC for stability and compatibility)
- Custom protocol: 'vtcp/btc-foundation/v1.0.0'

**2. Persistent Peer Storage**
- YAML-based peer storage (`peers.yaml`)
- Atomic file operations with conflict detection
- Schema validation and error recovery

**3. Configuration Management**
- YAML configuration file (`conf.yaml`)
- Hot-reload capability via SIGHUP signal
- Validation and error handling
- Secure private key management

**4. Peer Exchange Protocol**
- Automatic peer discovery on startup
- Peer announcement on new connections
- Periodic peer list synchronization
- Duplicate and invalid peer filtering

**5. Observability**
- Structured logging with configurable levels
- Event tracking (connections, disconnections, errors)
- Debug mode for message tracing (for future messages processing)


#### Acceptance Criteria

**Connectivity**:
- [ ] Node can bind to configured interfaces
- [ ] Successful connection establishment between any two nodes
- [ ] Support for both IPv4 and IPv6 addresses
- [ ] DNS resolution works for peer addresses
- [ ] Graceful handling of connection failures

**Peer Storage**:
- [ ] Peers persist across restarts
- [ ] Manual edits to peers.yaml are detected and handled
- [ ] Invalid peers are filtered out and logged
- [ ] Backup creation for corrupted files

**Configuration**:
- [ ] Valid YAML configuration is loaded successfully
- [ ] Invalid configuration prevents startup with clear error
    - [ ] Does not affect hot reload
- [ ] Hot-reload updates configuration without restart
- [ ] Private key is generated if missing

**Peer Exchange**:
- [ ] New peers are discovered and added automatically
- [ ] Peer announcements reach all connected nodes
- [ ] Duplicate peers are filtered out
- [ ] Invalid peer information is rejected

### Explicitly Out of Scope
- Protocol message processing beyond peer exchange
- Bitcoin network integration
- Advanced security features (beyond basic encryption)
- Metrics collection, dashboard or monitoring UI
- DDoS protection mechanisms

### Technical Specifications

#### Configuration Schema (`conf.yaml`)
```yaml
node:
  private_key: "<base64-encoded-private-key>"  # Generated if empty

network:
  addresses:
    - "/ip4/0.0.0.0/tcp/9000"    # Updated to TCP transport
    - "/ip6/::/tcp/9000"         # Updated to TCP transport

peers:
  exchange_interval: "30s"
  connection_timeout: "10s"

logging:
  level: "info"  # debug, info, warn, error
  format: "json"  # json, text
```

#### Peer Storage Schema (`peers.yaml`)
```yaml
peers:
  - public_key: "<base64-encoded-public-key>"
    addresses:
      - "/ip4/192.168.1.100/tcp/9000"        # Updated to TCP transport
      - "/dns4/node1.example.com/tcp/9000"   # Updated to TCP transport
```

#### Architecture Components

1. **Network Manager**: libp2p host management and connection handling
2. **Peer Manager**: Peer discovery, storage, and lifecycle management  
    2.1. **Storage Manager**: Thread-safe YAML file operations
3. **Config Manager**: Configuration loading, validation, and hot-reload
4. **Protocol Handler**: Custom protocol implementation for peer exchange

#### Transport Protocol Decision

**Critical Change**: The implementation uses **TCP instead of QUIC** as the primary transport protocol.

**Rationale for TCP Selection**:
- **Stability**: TCP is well-tested and stable in libp2p
- **Compatibility**: Better compatibility across different network environments  
- **Simplicity**: Simplified configuration and debugging
- **Development Velocity**: Faster implementation and testing
- **Security**: Equivalent security with libp2p's Noise protocol over TCP

**Impact on Requirements**:
- Connection establishment time remains within 3-second target
- All other performance and functionality requirements unchanged
- Enhanced stability and compatibility across diverse network environments

See [ADR-002: Transport Protocol Selection](../../../workspace/architecture/btc-federation/ADR-002-transport-protocol-selection.md) for detailed decision rationale.

#### Data Storage
- yaml file for peers storage

## Implementation Plan

### Strategic Overview

This implementation requires careful orchestration of several interconnected systems. The primary challenge is building a robust P2P foundation that can handle dynamic network conditions while maintaining data consistency and providing a stable platform for future protocol extensions.

### Critical Success Factors

1. **State Management Consistency**: Multiple components (config, peers, network) must maintain synchronized state
2. **Graceful Degradation**: System should continue operating when individual components fail
3. **Atomic Operations**: File operations and network state changes must be atomic to prevent corruption
4. **Resource Management**: Proper cleanup and resource lifecycle management to prevent leaks

### Phase 1: Foundation Architecture

#### Conceptual Approach: Dependency Injection and Separation of Concerns

**Day 1-2: System Architecture Design**

*Core Insight*: The application needs a clear separation between data layer, business logic, and network layer. Each component should be independently testable and replaceable.

*Key Decisions*:
- Configuration system as the single source of truth
- Event-driven architecture for component communication with core as an orchestrator and low-level decision maker.
- Interface-based design for all major components

*Potential Issues*:
- **Hot-reload Race Conditions**: Multiple components reacting to config changes simultaneously
  - *Solution*: Sequential notification with rollback capability

**Configuration Management Logic**

*Core Challenge*: Hot-reload must be atomic and reversible. If any component fails to apply new config, entire system should rollback.

*Implementation Strategy*:
1. Load and validate new configuration in isolation
2. Create a "staging" environment for validation
3. Notify components in dependency order
4. Implement rollback mechanism if any component fails

*Edge Cases to Handle*:
- **File Corruption During Read**: Backup/recovery mechanism needed
- **Partial Write Scenarios**: Atomic file operations with temp files
- **Signal Handling Race**: Multiple SIGHUP signals in quick succession
- **Validation Edge Cases**: Invalid addresses, malformed keys, impossible timeout values

**Network Host Abstraction**

*Core Concept*: Network layer should be completely isolated from business logic. All network operations should be asynchronous and cancellable.

*Critical Considerations*:
- **Connection State Machine**: Clear states for connecting, connected, disconnecting, failed
- **Timeout Handling**: Different timeouts for different operation types
- **Error Classification**: Distinguish between recoverable and fatal errors
- **Resource Cleanup**: Ensure no goroutine leaks when connections fail

*Potential Issues*:
- **Port Binding Conflicts**: Multiple instances or port already in use
  - *Solution*: Graceful retry with exponential backoff, alternative port selection
- **IPv6/IPv4 Dual Stack**: Proper handling of dual-stack environments
- **DNS Resolution Failures**: Cached resolution with TTL, fallback mechanisms

### Phase 2: Peer Management and Discovery (Days 8-14)

#### Conceptual Approach: Event-Driven Peer Lifecycle Management

**Peer Storage with Conflict Resolution**

*Core Challenge*: Multiple processes might modify peers.yaml simultaneously (manual edits, hot-reload, peer updates).

*Sophisticated Logic Required*:
1. **Three-Way Merge**: Current memory state, file state, new updates
2. **Conflict Resolution Strategy**: Last-seen timestamp as tiebreaker
3. **Data Integrity Verification**: Checksum validation before any modification
4. **Backup and Recovery**: Automatic backup creation before risky operations

*Edge Cases*:
- **Concurrent File Access**: Multiple instances writing simultaneously
  - *Solution*: File locking with timeout, atomic writes via temp files
- **Partial File Corruption**: File becomes unreadable mid-operation
  - *Solution*: Checksums, backup restoration, emergency peer list in memory
- **Schema Evolution**: Future changes to peers.yaml format
  - *Solution*: Version field in YAML, migration logic
- **Large Peer Lists**: Performance degradation with many peers
  - *Solution*: Pagination, lazy loading, LRU cache for active peers

**Peer Discovery Protocol Design**

*Core Insight*: Peer discovery is inherently unreliable and should be designed for eventual consistency rather than immediate consistency.

*Protocol Logic*:
1. **Bootstrap Phase**: Connect to initial peers, request their peer lists
2. **Steady State**: Periodic peer exchange with connected peers

**Connection Management and State Synchronization**

*Advanced Concepts*:
- **Connection Multiplexing**: Single physical connection, multiple logical streams
- **Heartbeat and Keep-Alive**: Detecting dead connections before TCP timeout

*State Synchronization Challenges*:
- **Network State vs Storage State**: Memory contains active connections, storage contains all known peers
- **Connection Flapping**: Rapid connect/disconnect cycles
- **Stale Connection Detection**: Connections that appear open but are actually dead

### Phase 3: Integration and Resilience

#### Conceptual Approach: Defensive Programming and Fault Tolerance

**Error Handling and Recovery Strategies**

*Philosophy*: Every operation that can fail, will fail. Design for failure as the normal case.

*Error Classification System*:
1. **Recoverable Errors**: Network timeouts, temporary file locks
2. **Configuration Errors**: Invalid config, missing files
3. **Fatal Errors**: Out of memory, disk full, permission denied

*Recovery Strategies*:
- **Circuit Breaker Pattern**: Stop attempting failed operations after threshold
- **Exponential Backoff**: Avoid overwhelming failed resources
- **Graceful Degradation**: Reduce functionality rather than total failure

**Testing Strategy and Edge Case Validation**

*Testing Philosophy*: Focus on state transitions and boundary conditions rather than happy path.

*Critical Test Scenarios*:
1. **Configuration Hot-Reload Under Load**: Change config while actively exchanging peers
2. **File Corruption Recovery**: Simulate disk corruption during peer file write
3. **Network Partition Scenarios**: Split network, rejoin, verify state consistency
5. **Concurrent Access Patterns**: Multiple threads accessing shared resources

**Performance Optimization and Production Readiness**

*Performance Considerations*:
- **Memory Usage Patterns**: Peer lists can grow large, need bounded memory usage
- **CPU Usage**: Avoid blocking operations on main thread
- **Network Efficiency**: Minimize unnecessary peer exchange traffic

*Production Hardening*:
- **Logging Strategy**: Structured logging with correlation IDs for tracing operations
- **Monitoring Hooks**: Metrics collection points without external dependencies
- **Graceful Shutdown**: Proper cleanup sequence, save state before exit

### Edge Cases and Design Challenges

#### 1. Configuration Schema Evolution
**Problem**: Future versions might need different config structure
**Solution**: Version field in config, migration logic, backward compatibility

#### 2. Peer Information Validation
**Problem**: Malicious or misconfigured peers providing invalid data
**Solution**: Multi-layer validation (syntax, network reachability, protocol compliance)

#### 3. Clock Skew and Time-based Logic
**Problem**: Different systems have different clocks, affects last-seen timestamps
**Solution**: Relative time comparisons, NTP awareness, monotonic clocks

#### 4. Resource Leaks in Long-Running Operation
**Problem**: Goroutines, file handles, network connections accumulating over time
**Solution**: Systematic resource tracking, periodic cleanup, resource pools

#### 5. Signal Handling Race Conditions
**Problem**: SIGHUP during config reload, SIGTERM during peer save
**Solution**: Signal masking during critical sections, atomic operations

#### 6. Peer Discovery Amplification
**Problem**: Each new peer causes peer list exchange, creating network storm
**Solution**: Rate limiting, jittered timing, exponential backoff

#### 7. Configuration Validation Dependencies
**Problem**: Some config changes require network validation (address binding)
**Solution**: Two-phase validation: syntax first, then runtime validation

### Implementation Risk Mitigation

| Risk Category | Specific Risk | Mitigation Strategy |
|---------------|---------------|-------------------|
| **Concurrency** | Race conditions in file access | File locking, atomic operations, state machines |
| **Network** | Connection storms, split brain | Rate limiting, exponential backoff, circuit breakers |
| **Configuration** | Invalid config causing startup failure | Validation layers, default fallbacks, rollback capability |
| **Performance** | Memory leaks, resource exhaustion | Resource pools, periodic cleanup, monitoring hooks |
| **Data Integrity** | File corruption, partial writes | Checksums, atomic writes, backup/recovery |

### Success Criteria Validation Strategy

Each success criterion requires specific validation approaches:

- **3-second connection time**: Measure from TCP SYN to protocol handshake completion
- **>95% peer discovery success**: Track discovery attempts vs successful additions over time
- **Hot-reload without drops**: Monitor active connections during config changes
- **No memory leaks**: Continuous monitoring of goroutine count and memory usage
- **10-minute stress test**: Automated test with connection churn and config changes

### Resource Requirements
#### Team Structure
- **Project Manager**: Dima Chizhevsky
- **Technical Lead**: Dima Chizhevsky
- **Developers**: Dima Chizhevsky
- **Designers**: n/a
- **QA Engineers**: n/a

## Testing Strategy

### Unit Testing
- **Coverage Target**: >90%
- **Focus Areas**: Configuration parsing, peer validation, file operations, TCP connectivity
- **Tools**: Go standard testing, testify for assertions

### Integration Testing
- Subject of another PRD.

### Performance Testing
- **Load Test**: 50 concurrent TCP connections
- **Stress Test**: 10-minutes continuous operation
- **Memory Test**: No memory leaks under load
- **Latency Test**: TCP connection establishment time

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-06-04 | Dima Chizhevsky | Initial version | Phase 1 |
| 1.1 | 2025-06-04 | Dima Chizhevsky | Updated transport protocol from QUIC to TCP | Phase 1 |

**Related Documents**
- **Master Project Vision**: n/a
- **Previous Iteration PRD**: n/a
- **Technical Architecture**: [ADR-001](../../../workspace/architecture/btc-federation/ADR-001-network-architecture-design.md), [ADR-002](../../../workspace/architecture/btc-federation/ADR-002-transport-protocol-selection.md), [ADR-003](../../../workspace/architecture/btc-federation/ADR-003-interface-design-integration.md)
- **User Research**: n/a
