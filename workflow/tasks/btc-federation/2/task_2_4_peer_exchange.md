# BTC-FED-2-4 - Peer Exchange Protocol & Discovery

# Links
- [PRD](../../prd/btc-federation/02_p2p_network_stack.md)

# Description
Implement the peer exchange protocol and discovery system that enables automatic peer discovery, peer announcements, and network growth. This task creates the core business logic for building and maintaining the peer-to-peer network through automated peer sharing and validation.

# Requirements and DOD
- [ ] Peer exchange protocol message definitions and handling
- [ ] Automatic peer discovery on startup from bootstrap peers
- [ ] Peer announcement system for new connections
- [ ] Periodic peer list synchronization with connected peers
- [ ] Duplicate peer filtering and validation
- [ ] Invalid peer information rejection and handling
- [ ] Protocol state management and error handling
- [ ] Bootstrap peer management and fallback strategies
- [ ] Integration with network layer and storage layer
- [ ] Unit tests covering all protocol scenarios (>90% coverage)
- [ ] Integration tests for peer discovery workflows

# Implementation Plan

## Step 1: Protocol Message Design
1. Define peer exchange protocol messages:
   - PeerRequest: Request peer list from connected peer
   - PeerResponse: Response with peer list
   - PeerAnnouncement: Announce new peer to network
   - PeerUpdate: Update existing peer information
2. Implement message serialization/deserialization:
   - Protobuf or JSON message format
   - Message validation and version handling
   - Error response messages
3. Create protocol constants and configuration:
   - Message size limits
   - Protocol timeouts
   - Rate limiting parameters
4. Add message routing and handling framework

## Step 2: Peer Discovery Logic
1. Implement bootstrap peer discovery:
   - Connect to configured bootstrap peers
   - Request initial peer lists
   - Validate and store discovered peers
   - Fallback strategies for bootstrap failures
2. Create peer announcement system:
   - Announce self to newly connected peers
   - Broadcast peer announcements to network
   - Handle incoming peer announcements
   - Duplicate announcement detection
3. Add peer validation pipeline:
   - Address format validation
   - Reachability checks (optional ping)
   - Public key validation
   - Network compatibility checks

## Step 3: Periodic Synchronization 
1. Implement periodic peer exchange:
   - Configurable exchange intervals
   - Smart peer selection for exchange
   - Incremental updates vs full synchronization
   - Jittered timing to prevent network storms
2. Create peer list management:
   - Merge incoming peer lists with local storage
   - Conflict resolution for duplicate peers
   - Peer freshness and staleness tracking
   - Automatic cleanup of unreachable peers
3. Add synchronization optimization:
   - Delta synchronization for large peer lists
   - Compression for large message payloads
   - Batching of multiple updates

## Step 4: Protocol State & Error Handling 
1. Implement protocol state machine:
   - Discovery phase, sync phase, maintenance phase
   - State transitions and validation
   - Recovery from error states
   - Protocol restart and reset mechanisms
2. Create comprehensive error handling:
   - Network error classification and handling
   - Protocol error recovery strategies
   - Malformed message handling
   - Timeout and retry logic
3. Add rate limiting and security:
   - Per-peer message rate limits
   - Anti-spam and abuse prevention
   - Resource exhaustion protection
   - Malicious peer detection and blocking

## Step 5: Integration & Testing
1. Integrate with network and storage layers:
   - Network event handling (connect/disconnect)
   - Storage layer integration for peer persistence
   - Configuration-driven behavior
   - Event notifications to application layer
2. Write comprehensive unit tests:
   - Protocol message handling
   - Discovery and announcement logic
   - Synchronization algorithms
   - Error handling and recovery
   - Rate limiting and security features
3. Create integration tests:
   - Multi-node peer discovery scenarios
   - Network partition and recovery
   - Large-scale peer synchronization
   - Real-world network simulation
4. Add performance testing and optimization

# Test Plan

## Unit Tests
- **Message Handling**: Serialization, validation, routing, error cases
- **Discovery Logic**: Bootstrap discovery, peer announcement, validation pipeline
- **Synchronization**: Periodic exchange, conflict resolution, optimization
- **State Management**: State transitions, error recovery, protocol restart
- **Rate Limiting**: Message limits, spam prevention, resource protection

## Integration Tests
- **Network Integration**: Connection events, message sending/receiving
- **Storage Integration**: Peer persistence, conflict resolution, data integrity
- **Multi-Node Scenarios**: 2-5 node networks, discovery propagation
- **Error Recovery**: Network failures, malformed messages, peer unavailability
- **Performance**: Large peer lists, high-frequency updates, resource usage

## Scenario Tests
- **Bootstrap Scenarios**: No peers, single peer, multiple peers, all unreachable
- **Network Growth**: Gradual peer addition, rapid network expansion
- **Churn Handling**: Peers joining/leaving rapidly, network instability
- **Security**: Malicious peers, spam attacks, resource exhaustion attempts

# Verification and Validation

## Architecture integrity
- Clear separation between protocol logic and transport layer
- Event-driven architecture for peer lifecycle management
- Interface-based design for easy testing and extensibility
- Proper abstraction of network and storage dependencies

## Security
- Input validation for all incoming protocol messages
- Rate limiting to prevent abuse and resource exhaustion
- Protection against malicious peer behavior
- Secure handling of peer information and metadata

## Performance
- Efficient peer discovery within target timeframes
- Minimal network overhead for peer synchronization
- Optimized message handling and processing
- Scalable performance with increasing peer count

## Scalability
- Efficient handling of large peer lists (1000+ peers)
- Distributed discovery that scales with network size
- Minimal per-peer resource requirements
- Support for future protocol extensions

## Reliability
- Robust discovery even with unreliable bootstrap peers
- Fault tolerance for network partitions and failures
- Automatic recovery from protocol errors
- Graceful degradation under adverse conditions

## Maintainability
- Clear protocol specification and documentation
- Modular design for easy modification and testing
- Comprehensive logging and debugging support
- Well-defined interfaces and state management