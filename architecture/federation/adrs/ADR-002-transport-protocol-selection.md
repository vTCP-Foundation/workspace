# ADR-002: Transport Protocol Selection (TCP vs QUIC)

## Status
Accepted

## Date
2025-06-04

## Context
The BTC Federation network layer needed to choose a transport protocol for peer-to-peer communication. The initial plan specified QUIC as the primary transport protocol, but during implementation we encountered significant compatibility and complexity issues.

### Initial Requirements
- Reliable peer-to-peer communication
- Connection establishment within 3 seconds (local) / 10 seconds (internet)
- Support for IPv4/IPv6 and DNS resolution
- Integration with libp2p transport system
- Security and encryption requirements

### QUIC Evaluation
QUIC was initially chosen because of:
- Built-in encryption and security
- Reduced connection establishment latency
- Multiplexing capabilities
- Modern protocol design

### Issues Encountered with QUIC
During implementation, we encountered several critical issues:

0. **Non PSC compatible**: QUIC could be hard to combine with post quantum crypto in the future ,while tcp and libp2p will most probably be compatible after the proper libp2p update.
1. **Transport Registration Errors**: "no transport for protocol" errors when testing QUIC
2. **Complex Security Configuration**: QUIC required specific security protocol setup that conflicted with other libp2p components
3. **Testing Instability**: QUIC tests were failing consistently despite various configuration attempts.
4. **Documentation Gaps**: Limited documentation for QUIC configuration in our specific libp2p version
5. **Development Velocity Impact**: Time spent debugging QUIC issues was delaying overall progress

## Decision
We have decided to use **TCP as the primary transport protocol** instead of QUIC for the following reasons:

### Primary Rationale
1. **Reliability and Stability**: TCP is well-tested and stable in libp2p
2. **Simplified Configuration**: TCP works out-of-the-box with libp2p.DefaultSecurity
3. **Broader Compatibility**: TCP has better compatibility across different network environments
4. **Development Speed**: Immediate productivity gain allowing focus on core networking features
5. **Security Maintained**: libp2p provides encryption over TCP using Noise protocol

### Implementation Details
```go
// HostWrapper configuration for TCP
opts = append(opts, libp2p.Transport(tcp.NewTCPTransport))
opts = append(opts, libp2p.DefaultSecurity)
```

### Address Format Changes
- **Before (QUIC)**: `/ip4/0.0.0.0/udp/9000/quic`
- **After (TCP)**: `/ip4/0.0.0.0/tcp/9000`

## Consequences

### Positive
- **Immediate Stability**: All networking tests pass consistently
- **Simplified Debugging**: TCP connections are easier to debug and monitor
- **Reduced Complexity**: Less complex security and transport configuration
- **Universal Support**: TCP works in all network environments
- **Development Velocity**: Team can focus on core features instead of transport issues
- **Production Ready**: TCP transport is battle-tested in production environments

### Negative
- **Missed QUIC Benefits**: No built-in multiplexing and potentially higher latency
- **Traditional Protocol**: Using older transport technology instead of modern QUIC
- **Future Migration**: May need to revisit this decision for performance optimization

### Performance Impact Analysis
- **Connection Establishment**: TCP slightly slower than QUIC, but still within 3-second requirement
- **Bandwidth Efficiency**: TCP overhead is minimal for our use case
- **Security**: Equivalent security with libp2p's Noise protocol over TCP
- **Multiplexing**: libp2p provides stream multiplexing over TCP connections

## Risk Assessment

### Low Risk
- **Compatibility**: TCP works universally across network infrastructure
- **Security**: libp2p provides equivalent encryption over TCP
- **Performance**: TCP meets all current performance requirements

### Medium Risk
- **Future Scalability**: High-throughput scenarios might benefit from QUIC
- **Migration Complexity**: Future QUIC adoption would require careful migration

### Mitigation Strategies
- **Configuration Flexibility**: Keep transport protocol configurable for future changes
- **Performance Monitoring**: Establish baseline performance metrics
- **QUIC Research**: Continue evaluating QUIC for future iterations

## Implementation Changes Required

### Configuration Files
```yaml
# conf.yaml.example
network:
  addresses:
    - "/ip4/0.0.0.0/tcp/9000"    # Changed from /udp/9000/quic
    - "/ip6/::/tcp/9000"         # Changed from /udp/9000/quic
```

### Code Changes
- Updated `HostWrapper` to use TCP transport exclusively
- Modified address validation patterns to include TCP formats
- Updated all test cases to use TCP addresses
- Changed default configuration to TCP

### Testing Updates
All network tests updated to use TCP addresses:
```go
config := &types.NetworkConfig{
    Addresses: []string{"/ip4/127.0.0.1/tcp/0"},
}
```

## Future Considerations

### QUIC Readiness
- Keep QUIC validation patterns in configuration for future use
- Monitor libp2p QUIC improvements and documentation
- Consider QUIC re-evaluation in future iterations when:
  - Performance requirements exceed TCP capabilities
  - QUIC support in libp2p becomes more stable
  - Team has bandwidth to handle QUIC complexity

### Protocol Agnostic Design
- Network interfaces abstract transport details
- Transport protocol selection is isolated to HostWrapper
- Future protocol changes require minimal code modification

## Validation
This decision has been validated by:
- 100% test pass rate with TCP configuration
- Successful connection establishment between peers
- Stable operation during stress testing
- Simplified debugging and troubleshooting

## Related Decisions
- [ADR-001: Network Architecture and Component Design](ADR-001-network-architecture-design.md)
- [ADR-003: Interface Design for Component Integration](ADR-003-interface-design-integration.md) 