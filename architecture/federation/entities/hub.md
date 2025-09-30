# vTCP Network Hub

- The Hub is responsible for network connectivity, ensuring fast and efficient routing between [vTCP Network](/architecture/common/entities/vtcp_network.md) participants and actively participating in transaction processing. The Hub operates with user-deposited funds (users top up their balance during the hub onboarding process through the [BTC <-> vTCP Custody Protocol](/architecture/federation/protocols/BTC%20<->%20vTCP%20Custody%20Protocol.md)).

- The Hub operates the vTCP protocol by maintaining one or more active [vTCP nodes](/architecture/common/entities/vtcp_network_node.md)

## Relationship with Federation

### Multi-Federation Connectivity
- Hubs can connect to multiple [Federations](/architecture/common/entities/federation.md) simultaneously
- Each Hub-Federation connection is independent
- Enables multi-asset operations across different custody systems

### Federation Membership
- Hub operators are **not required** to be Federation members
- Whether a Hub operator can also be a Federation member: **TBD**

### Fee Structure
- Hubs do **not** pay fees to Federations for using custody services
- Fee model may be added if required for proper business model (TBD)