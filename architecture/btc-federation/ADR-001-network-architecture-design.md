# ADR-001: Network Architecture and Component Design

## Status
Accepted

## Date
2025-06-04

## Context
The BTC Federation project requires a robust peer-to-peer networking foundation that can handle connection management, protocol handling, and provide a clean interface for future features like peer storage (Task 2.3), peer exchange (Task 2.4), and hot configuration reload (Task 2.5).

The networking layer needs to:
- Provide reliable peer-to-peer connectivity
- Support multiple transport protocols
- Enable protocol extensibility
- Integrate with existing configuration system
- Support connection event handling
- Allow for future hot-reload capabilities

## Decision
We have decided to implement a layered network architecture using libp2p with the following component structure:

### Core Components
1. **NetworkManager** - Main interface providing lifecycle management and orchestration
2. **HostWrapper** - libp2p host abstraction handling transport details
3. **Connection Management** - State tracking and lifecycle for peer connections
4. **Protocol System** - Registration framework for custom protocols
5. **Event System** - Connection event notification for other components

### Architecture Pattern
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│ NetworkManager  │───▶│   HostWrapper    │───▶│   libp2p Host   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐    ┌──────────────────┐
│ Event Handlers  │    │   Connections    │
└─────────────────┘    └──────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐    ┌──────────────────┐
│ Protocol Handlers│   │  State Tracking  │
└─────────────────┘    └──────────────────┘
```

### Key Interfaces

**NetworkManager Interface:**
```go
type NetworkManager interface {
    // Core lifecycle
    Start(ctx context.Context) error
    Stop() error
    
    // Connection management
    Connect(ctx context.Context, addr multiaddr.Multiaddr) error
    Disconnect(peerID peer.ID) error
    GetConnections() []ConnectionInfo
    GetHost() host.Host
    
    // Protocol management
    RegisterProtocolHandler(protocolID protocol.ID, handler ProtocolHandler) error
    UnregisterProtocolHandler(protocolID protocol.ID) error
    
    // Event handling
    RegisterConnectionHandler(handler ConnectionEventHandler) error
    UnregisterConnectionHandler(handler ConnectionEventHandler) error
    
    // Configuration management
    UpdateConfig(config *types.NetworkConfig) error
    ValidateConfig(config *types.NetworkConfig) error
    GetCurrentConfig() *types.NetworkConfig
}
```

**Connection State Machine:**
- Idle → Connecting → Connected → Disconnecting → Idle
- Failed state for error handling
- Thread-safe state transitions

## Consequences

### Positive
- **Clean Separation of Concerns**: Each component has a single responsibility
- **Testability**: Interface-based design enables easy mocking and testing
- **Extensibility**: Protocol registration system allows future protocol additions
- **Integration Ready**: Well-defined interfaces for Tasks 2.3-2.5
- **Configuration Driven**: Seamless integration with existing config system
- **Event-Driven**: Asynchronous event system prevents blocking operations

### Negative
- **Complexity**: Multiple layers and interfaces increase codebase complexity
- **Abstraction Overhead**: Additional abstractions over libp2p may impact performance slightly
- **Learning Curve**: New developers need to understand the layered architecture

### Risks and Mitigations
- **Interface Changes**: Future requirements might require interface modifications
  - *Mitigation*: Version interfaces and provide backward compatibility
- **Performance Overhead**: Multiple abstraction layers
  - *Mitigation*: Benchmark and optimize critical paths if needed
- **Integration Complexity**: Multiple components need careful coordination
  - *Mitigation*: Comprehensive integration tests and clear documentation

## Implementation Notes
- All components use dependency injection for testability
- Event handlers are called asynchronously to prevent blocking
- Connection state is tracked in memory with thread-safe operations
- Error handling follows Go conventions with wrapped errors
- Resource cleanup is handled at each layer with proper cancellation

## Related Decisions
- [ADR-002: Transport Protocol Selection](ADR-002-transport-protocol-selection.md)
- [ADR-003: Interface Design for Component Integration](ADR-003-interface-design-integration.md) 