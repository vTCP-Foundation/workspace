# BTC Federation
BTC-federation provides trustless custody of BTC on L1 through multisig implementation, utilizing taproot and the FROST cryptographic scheme.

## Purpose
The federation serves as a custody solution for BTC deposits as described in the [BTC ⟷ vTCP Custody Protocol](/architecture/btc-federation/protocols/protocol_btc<>vtcp_custody_human_description.md). It operates as a low-frequency custody system, not designed for high-frequency trading operations.

## Consensus Protocol
The federation uses [HotStuff BFT consensus](/architecture/btc-federation/adrs/ADR-005-hotstuff-consensus-protocol.md) to coordinate operations between participants. This choice emerges from Bitcoin L1's security model limitations where only basic multisig primitives are available via Bitcoin Script.

# Number of Participants
- The number of federation participants can be variable.
- **Min = 64** 
    - for security considerations

- **Max = 512** 
    - comes from physical limitations imposed by the FROST cryptographic scheme according to the results obtained during [this research](/workflow/prd/btc-federation/01_taproot_multisig.md) (see the embedded tasks for this PRD).

- **Step = 2**

# Quorum threshold
- The federation uses a **threshold signing scheme** where **M-of-N** participants are required, where **M = 75% + 1 of N**

# Performance Requirements
## Multisig operations
- Complete multisig workflow (signature only) must execute within 2 minutes for all participant counts

## Consensus operations
- Target 3-10 seconds for consensus finality in 512 participant federation
- Support custody operation rates (low-frequency operations)


# Staking
- Every participant in the federation is expected to stake their BTC in order to be able to become a member of the Federation.
- The stake amount is defined by the Federation itself and so defines the max capacity of funds that can be held by the Federation (users ⟷ Hubs custody)
- Participants must maintain stake equal to their share of custody responsibilities


# Staking cascade
todo: document Federation A -> Federtation B -> Federation C and all other federations.


# Random participants shuffle 
todo: document how the participants are shuffled randomly.