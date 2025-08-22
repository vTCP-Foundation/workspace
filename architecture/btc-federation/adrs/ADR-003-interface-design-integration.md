# ADR-003: Interface Design for Component Integration

## Status
Accepted

## Date
2025-06-04

## Context
The BTC Federation network layer (Task 2.2) serves as the foundation for several upcoming features:
- Task 2.3: Peer Storage
- Task 2.4: Peer Exchange Protocol
- Task 2.5: Configuration Hot-reload

To ensure smooth integration, we need well-defined interfaces that provide the necessary integration points without creating tight coupling between components.

### Integration Requirements
1. **Event-driven Communication**: Components should react to network events without direct coupling
2. **Protocol Extensibility**: New protocols should be easy to add and remove
3. **Configuration Flexibility**: Support dynamic configuration updates
4. **Clean Dependencies**: Each component should have minimal, well-defined dependencies

## Decision
We have designed three primary interfaces for future component integration:

### 1. ConnectionEventHandler Interface
For components that need to react to peer connection events:

```go
type ConnectionEventHandler interface {
    OnPeerConnected(peerID peer.ID, addr multiaddr.Multiaddr) error
    OnPeerDisconnected(peerID peer.ID, reason error) error
}
```

**Purpose**: Enables Task 2.3 (Peer Storage) to track peer connectivity and update persistent storage.

**Design Principles**:
- **Asynchronous**: Event handlers are called asynchronously to prevent blocking
- **Error Handling**: Return errors to indicate handler failures without crashing the network layer
- **Minimal Dependencies**: Only requires libp2p core types

### 2. ProtocolHandler Interface  
For components that implement custom protocols:

```go
type ProtocolHandler interface {
    HandleStream(stream network.Stream) error
    ProtocolID() protocol.ID
}
```

**Purpose**: Enables Task 2.4 (Peer Exchange Protocol) to implement custom messaging protocols.

**Design Principles**:
- **Stream-based**: Each protocol interaction gets a separate stream
- **Self-identifying**: Protocols declare their own ID for registration
- **Resource Management**: Handlers responsible for stream cleanup

### 3. ConfigurableComponent Interface
For components that support dynamic configuration updates:

```go
type ConfigurableComponent interface {
    UpdateConfig(config interface{}) error
    ValidateConfig(config interface{}) error
}
```

**Purpose**: Enables Task 2.5 (Hot-reload) to update component configurations without restart.

**Design Principles**:
- **Validation First**: Always validate before applying changes
- **Type Flexibility**: Uses `interface{}` to support different config types
- **Atomic Updates**: Either entire config update succeeds or fails

## Implementation Strategy

### Event Handler Registration
```go
// Network manager maintains list of event handlers
type Manager struct {
    eventHandlers []ConnectionEventHandler
    // ... other fields
}

// Register handler for connection events
func (m *Manager) RegisterConnectionHandler(handler ConnectionEventHandler) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    // Prevent duplicate registration
    for _, h := range m.eventHandlers {
        if h == handler {
            return ErrHandlerAlreadyRegistered
        }
    }
    
    m.eventHandlers = append(m.eventHandlers, handler)
    return nil
}
```

### Protocol Handler Registration
```go
// Protocol handlers stored by protocol ID
type Manager struct {
    protocolHandlers map[protocol.ID]ProtocolHandler
    // ... other fields
}

func (m *Manager) RegisterProtocolHandler(protocolID protocol.ID, handler ProtocolHandler) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    if _, exists := m.protocolHandlers[protocolID]; exists {
        return ErrProtocolAlreadyRegistered
    }
    
    // Register with libp2p host
    m.host.SetStreamHandler(protocolID, func(stream network.Stream) {
        if err := handler.HandleStream(stream); err != nil {
            // Log error and close stream
            stream.Reset()
        }
    })
    
    m.protocolHandlers[protocolID] = handler
    return nil
}
```

### Configuration Update Pattern
```go
func (m *Manager) UpdateConfig(config *types.NetworkConfig) error {
    // Validate first
    if err := m.ValidateConfig(config); err != nil {
        return fmt.Errorf("config validation failed: %w", err)
    }
    
    // Apply atomically
    m.mu.Lock()
    oldConfig := m.config
    m.config = config
    m.mu.Unlock()
    
    // Notify other configurable components
    // (implementation depends on specific requirements)
    
    return nil
}
```

## Integration Examples

### Task 2.3: Peer Storage Integration
```go
type PeerStorage struct {
    peers map[peer.ID]*PeerInfo
    // ... other fields
}

func (ps *PeerStorage) OnPeerConnected(peerID peer.ID, addr multiaddr.Multiaddr) error {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    
    // Update peer information
    if info, exists := ps.peers[peerID]; exists {
        info.LastSeen = time.Now()
        info.Addresses = append(info.Addresses, addr)
    } else {
        ps.peers[peerID] = &PeerInfo{
            ID:       peerID,
            Addresses: []multiaddr.Multiaddr{addr},
            LastSeen: time.Now(),
        }
    }
    
    return ps.savePeers() // Persist to disk
}
```

### Task 2.4: Peer Exchange Protocol
```go
type PeerExchangeProtocol struct {
    peerStorage *PeerStorage
    // ... other fields
}

func (pep *PeerExchangeProtocol) HandleStream(stream network.Stream) error {
    defer stream.Close()
    
    // Read peer exchange message
    msg, err := readPeerExchangeMessage(stream)
    if err != nil {
        return err
    }
    
    // Process received peers
    for _, peer := range msg.Peers {
        pep.peerStorage.AddPeer(peer)
    }
    
    // Send our known peers
    response := pep.createPeerListResponse()
    return writePeerExchangeMessage(stream, response)
}

func (pep *PeerExchangeProtocol) ProtocolID() protocol.ID {
    return "/vtcp/btc-foundation/peer-exchange/v1.0.0"
}
```

## Consequences

### Positive
- **Clean Integration**: Each task has clear integration points
- **Loose Coupling**: Components communicate through well-defined interfaces
- **Testability**: Interfaces enable easy mocking for unit tests
- **Extensibility**: New protocols and handlers can be added without modifying core network code
- **Error Isolation**: Handler failures don't crash the network layer

### Negative
- **Interface Complexity**: Additional abstraction layer for component communication
- **Runtime Overhead**: Dynamic handler registration and invocation
- **Type Safety**: `interface{}` in ConfigurableComponent reduces compile-time type checking

### Risk Mitigation
- **Documentation**: Comprehensive documentation for each interface
- **Examples**: Provide example implementations for each interface
- **Testing**: Interface compliance tests for all implementations
- **Version Management**: Plan interface evolution to maintain backward compatibility

## Usage Guidelines

### For Event Handlers
1. **Idempotent Operations**: Handlers may be called multiple times for the same event
2. **Fast Execution**: Avoid blocking operations in event handlers
3. **Error Handling**: Return errors for failures but don't panic
4. **Resource Cleanup**: Ensure proper cleanup in case of errors

### For Protocol Handlers  
1. **Stream Management**: Always close streams when done
2. **Timeout Handling**: Implement appropriate read/write timeouts
3. **Message Validation**: Validate all incoming messages
4. **Version Compatibility**: Handle protocol version negotiation

### For Configuration Updates
1. **Validation**: Always validate configuration before applying
2. **Rollback**: Design for configuration rollback on partial failures
3. **Atomicity**: Make configuration changes atomic where possible
4. **Notification**: Notify dependent components of configuration changes

## Performance Considerations

### Event Handler Performance
- Handlers called asynchronously to prevent blocking network operations
- Consider handler execution time for high-frequency events
- Use buffered channels for handler communication if needed

### Protocol Handler Performance
- Stream multiplexing allows concurrent protocol handling
- Consider connection pooling for frequently used protocols
- Implement flow control for high-throughput protocols

### Memory Management
- Event handlers should not hold long-term references to event data
- Protocol handlers should clean up resources properly
- Consider implementing handler lifecycle management

## Related Decisions
- [ADR-001: Network Architecture and Component Design](ADR-001-network-architecture-design.md)
- [ADR-002: Transport Protocol Selection](ADR-002-transport-protocol-selection.md) 