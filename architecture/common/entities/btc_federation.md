# BTC Federation
BTC-federation provides trustless custody of BTC on L1 through multisig implementation, utilizing taproot and the FROST cryptographic scheme.

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