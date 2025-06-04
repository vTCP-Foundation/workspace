# BTC-FED-2-2 - Network Host & libp2p Integration

# Links
- [PRD](../../prd/btc-federation/02_p2p_network_stack.md)

# Description
Implement the core networking layer using libp2p with QUIC transport, custom protocol registration, and connection management. This task creates the foundation for all peer-to-peer communication in the BTC Federation network.

# Requirements and DOD
- [ ] libp2p host initialization with proper configuration
- [ ] QUIC transport protocol implementation
- [ ] IPv4/IPv6 dual-stack support with DNS resolution
- [ ] Custom protocol registration ('vtcp/btc-foundation/v1.0.0')
- [ ] Connection establishment and management
- [ ] Peer connection state tracking (connecting, connected, disconnecting, failed)
- [ ] Graceful connection handling with timeout management
- [ ] Network interface binding based on configuration
- [ ] Connection event handling and callbacks
- [ ] Resource cleanup and proper shutdown procedures
- [ ] Unit tests for all networking functionality (>90% coverage)
- [ ] Integration tests for connection scenarios

# Implementation Plan

## Step 1: libp2p Host Setup 
1. Create NetworkManager struct with libp2p host
2. Implement host initialization with configuration:
   - Private key integration from config
   - Listen address configuration
   - Transport setup (QUIC)
   - Security configuration
3. Add dual-stack IPv4/IPv6 support
4. Implement DNS resolution capabilities
5. Create host lifecycle management (start/stop)

## Step 2: Protocol Registration & Handling 
1. Define custom protocol ID: 'vtcp/btc-foundation/v1.0.0'
2. Implement protocol stream handler interface
3. Create protocol registration system
4. Add stream management and lifecycle
5. Implement protocol handshake mechanism
6. Add protocol version negotiation preparation

## Step 3: Connection Management 
1. Implement connection state machine:
   - States: idle, connecting, connected, disconnecting, failed
   - State transitions with proper validation
   - Timeout handling for each state
2. Create connection tracking system:
   - Active connections registry
   - Connection metadata storage
   - Peer identification and mapping
3. Add connection event system:
   - Connection established events
   - Connection lost events
   - Connection error events
4. Implement connection pooling and limits

## Step 4: Network Operations & Error Handling 
1. Add peer dialing functionality:
   - Address parsing and validation
   - Connection timeout management
   - Retry logic with exponential backoff
2. Implement connection monitoring:
   - Health checks and keep-alive
   - Dead connection detection
   - Automatic cleanup procedures
3. Create comprehensive error handling:
   - Network error classification
   - Recovery strategies
   - Error reporting and logging
4. Add graceful shutdown procedures

## Step 5: Testing & Integration 
1. Write comprehensive unit tests:
   - Host initialization scenarios
   - Protocol registration and handling
   - Connection state machine transitions
   - Error handling and recovery
   - Resource cleanup verification
2. Create integration tests:
   - Peer-to-peer connection establishment
   - Multiple peer scenarios
   - Network partition handling
   - Configuration-driven network setup
3. Add performance and load testing
4. Create network utilities and debugging tools

# Test Plan

## Unit Tests
- **Host Initialization**: Valid/invalid configurations, key management
- **Protocol Handling**: Registration, stream management, handshakes
- **Connection Management**: State transitions, timeout handling, cleanup
- **Address Resolution**: IPv4/IPv6, DNS resolution, invalid addresses
- **Error Scenarios**: Network failures, timeout handling, resource exhaustion

## Integration Tests
- **Peer Connection**: Two-node connection establishment
- **Multiple Peers**: Connect to multiple peers simultaneously
- **Network Resilience**: Connection drops, reconnection logic
- **Configuration Integration**: Network settings from config system
- **Resource Management**: Memory and connection leak detection

## Load Tests
- **Connection Stress**: Multiple simultaneous connections
- **Protocol Load**: High-frequency protocol message handling
- **Memory Usage**: Extended operation without leaks

# Verification and Validation

## Architecture integrity
- Clean separation between network layer and business logic
- Interface-based design for protocol handlers
- Proper abstraction of libp2p implementation details
- Event-driven architecture for connection management

## Security
- Secure transport using libp2p encryption
- Proper private key handling and protection
- Protocol version validation and security
- Address validation to prevent injection attacks

## Performance
- Efficient connection establishment (< 3 seconds local, < 10 seconds internet)
- Minimal resource overhead per connection
- Optimized message handling and routing
- Connection pooling and reuse where appropriate

## Scalability
- Efficient peer tracking and management
- Protocol extensibility for future features
- Resource usage scales appropriately with peer count

## Reliability
- Robust error handling and recovery mechanisms
- Connection monitoring and health checks
- Graceful degradation under network stress
- Proper cleanup and resource management

## Maintainability
- Clear interfaces and abstractions
- Comprehensive logging and debugging support
- Modular design for easy testing and modification
- Well-documented protocol and connection handling

## Cost
- Standard libp2p protocols and practices