# BTC Federation

The BTC Federation is a [Federation](/architecture/common/entities/federation.md) implementation that provides access to Bitcoin L1.

## Bitcoin-Specific Implementation

### Custody Protocol
Provides trustless custody of BTC on L1 through multisig implementation, utilizing taproot and the FROST cryptographic scheme as described in the [BTC ⟷ vTCP Custody Protocol](/architecture/federation/protocols/protocol_btc<>vtcp_custody_human_description.md).

### Consensus Protocol
Uses [HotStuff BFT consensus](/architecture/federation/adrs/ADR-005-hotstuff-consensus-protocol.md) to coordinate operations between participants. This choice emerges from Bitcoin L1's security model limitations where only basic multisig primitives are available via Bitcoin Script.

### Cryptographic Scheme
- **Max = 512** participants comes from physical limitations imposed by the FROST cryptographic scheme according to the results obtained during [this research](/workflow/prd/federation/01_taproot_multisig.md) (see the embedded tasks for this PRD)

### Performance Requirements

#### Multisig operations
- Complete multisig workflow (signature only) must execute within 2 minutes for all participant counts

#### Consensus operations
- Target 3-10 seconds for consensus finality in 512 participant federation
- Support custody operation rates (low-frequency operations)

### Staking
- Participants stake **BTC** to become Federation members
- Stake amount defines max capacity of BTC that can be held by the Federation

## Related Entities
- [Federation (Generic)](/architecture/common/entities/federation.md)
- [Federation Channels Registry](/architecture/federation/entities/federation_channels_registry.md)

## Related Protocols
- [BTC ⟷ vTCP Custody Protocol](/architecture/federation/protocols/protocol_btc<>vtcp_custody_human_description.md)
- [Federation Membership Protocol](/architecture/common/protocols/protocol_federation_membership.md)