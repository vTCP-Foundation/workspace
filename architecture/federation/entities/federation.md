# Federation

A Federation provides trustless custody of assets on L1 through multisig implementation. Each asset type has its own dedicated Federation.

## Asset Scope
- **One Federation per asset** (e.g., BTC Federation, USDT Federation, USDC Federation)
- Each Federation maintains independent consensus
- Federations share the same protocol architecture but operate independently
- Hubs can connect to multiple Federations simultaneously for multi-asset operations

## Purpose
Federations serve as custody solutions for asset deposits, operating as low-frequency custody systems not designed for high-frequency trading operations.

## Participant Composition
- Federation participants are expected to be institutional partners
- Participants must be approved by the current member set (approval process TBD)
- Members can join or leave dynamically once per epoch (see [Federation Membership Protocol](/architecture/common/protocols/protocol_federation_membership.md))

## Number of Participants
- The number of federation participants can be variable
- **Min = 64** (for security considerations)
- **Max = 512** (physical limitations from cryptographic schemes)
- **Step = 2**

## Quorum Threshold
- Uses a **threshold signing scheme** where **M-of-N** participants are required
- **M = 75% + 1 of N**

## Consensus Protocol
Each Federation uses HotStuff BFT consensus to coordinate operations between participants.

## Staking
- Every participant must stake assets to become a Federation member
- The stake amount is defined by the Federation itself
- Stake amount defines the max capacity of funds that can be held by the Federation (users ⟷ Hubs custody)
- Participants must maintain stake equal to their share of custody responsibilities

## Hub Selection
- Federations **select one or several Hubs** to work with
- Selection is critical because **L1-L2 mirroring depends on Hub reliability**
- L1-L2 mirroring is one of the Federation's most important services
- Each Hub-Federation relationship is independent
- Hubs can work with multiple Federations simultaneously for multi-asset operations

## Asset-Specific Implementations
- [BTC Federation](/architecture/common/entities/federation_btc.md) - Bitcoin custody using taproot and FROST
- Additional asset federations follow the same pattern

## Related Protocols
- [Federation Membership Protocol](/architecture/common/protocols/protocol_federation_membership.md)
- Staking Cascade: TBD
- Random Participants Shuffle: TBD