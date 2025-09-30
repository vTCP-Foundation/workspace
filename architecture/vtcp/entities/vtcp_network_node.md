# vTCP Node

## Overview

A vTCP Network Node is any participant in the [vTCP Network](/architecture/vtcp/entities/vtcp_network.md) that can maintain [settlement lines](/architecture/vtcp/entities/vtcp_settlement_line.md) and route payments.

## Node Capabilities

### Payment Routing

**Routing Fees:** Each node independently sets its own routing fees for forwarding payments through its channels.

**Path Discovery:** Nodes participate in route discovery mechanisms. **[TBD: Link to route discovery protocol once merged from development branch]**

**Information Exposure:** Network topology and liquidity information exposure is defined by the route discovery protocol. **[TBD: Link to route discovery protocol specification]**

### Cross-Asset Exchange

**Exchanger Role:** Any vTCP node can act as an exchanger, facilitating cross-asset payments by maintaining settlement lines in multiple assets.

**Exchange Rates:** The node implementation provides tools for exchange rate updates, which are propagated to the network. Rate-setting mechanisms are implementation-specific.

**Liquidity Management:** Reserve management for cross-asset exchanges is entirely the responsibility of the node operator. The protocol does not enforce reserve requirements or liquidity guarantees.

## Node Types

Different node types (User nodes, [Hubs](/architecture/federation/entities/hub.md), etc.) are specialized configurations with additional capabilities, but all share the base vTCP node functionality.

## References

- [vTCP Network](/architecture/vtcp/entities/vtcp_network.md)
- [vTCP Settlement Line](/architecture/vtcp/entities/vtcp_settlement_line.md)
- [Hub](/architecture/federation/entities/hub.md)
- [Route Discovery Protocol](/architecture/vtcp/protocols/protocol_route_discovery.md)

