# ADR-005: HotStuff Consensus Protocol Adoption

## Status
Accepted

## Date
2025-08-05

## Author(s)
Dima Chizhevsky, Mykola Ilashchuk

## Context
The BTC Federation requires a Byzantine Fault Tolerant (BFT) consensus protocol to coordinate a large-scale federation of up to 512 participants [(PRD-03)](/workflow/prd/federation/03_musig2_frost_comparative_analysis.md). The choice of BFT consensus emerges directly from Bitcoin L1's security model limitations: _Bitcoin Script provides only basic multisig primitives as cryptographic building blocks_. Since Bitcoin L1 cannot enforce complex consensus rules natively, the federation **must** implement distributed coordination **off-chain** while maintaining cryptographic security guarantees through Bitcoin's multisig mechanism.

### Requirements
- Byzantine Fault Tolerant consensus for up to 512 participants [(PRD-03)](/workflow/prd/federation/03_musig2_frost_comparative_analysis.md)
    - Robust consensus protocol to synchronize operations between federation participants
    - Scalable consensus suitable for large federation sizes
    - Deterministic finality for coordination of federation operations

### Bitcoin L1 Security Model Constraints
The Bitcoin L1 chain provides only multisigs as crypto primitives for distributed consensus on funds. This creates a unique security model where:
- **Vote == Signature**: Every consensus vote must be backed by a cryptographic signature
- **Multisig Threshold**: Decentralization is ensured through sufficiently large threshold requirements
- **Forced Correspondence**: The BFT consensus must align with Bitcoin's signature-based security model

### Federation Architecture Context
Based on the analysis in [PRD-03 MuSig2 vs FROST Comparative Analysis](/workflow/prd/federation/03_musig2_frost_comparative_analysis.md):
- **Target Size**: 512 participants federation (max) [(PRD-03)](/workflow/prd/federation/03_musig2_frost_comparative_analysis.md)
- **Signature Scheme**: **SPHINCS+** for Federation Participants Voting [(ADR-4)](architecture/federation/adrs/ADR-004-sphincs-signature-scheme.md) 
- **Security Model**: BFT consensus is acceptable because vote validation is enforced by Bitcoin L1's multisig requirements

### Alternative Considerations
Other consensus protocols considered:
- **PBFT (Practical Byzantine Fault Tolerance)**: Original BFT protocol, but lacks optimizations for large participant counts
- **Tendermint**: Popular BFT consensus, but designed for smaller validator sets
- **Raft**: Not Byzantine fault tolerant, unsuitable for adversarial environments

### HotStuff Protocol Reference
- **Academic Paper**: [HotStuff: BFT Consensus with Linearity and Responsiveness](https://arxiv.org/abs/1803.05069) (PODC 2019)
- **HotStuff Data Flow**: [HotStuff Data Flow](/architecture/federation/protocols/hot-stuff-consensus/data-flow.mermaid)

## Decision
We have decided to adopt **HotStuff** as our Byzantine Fault Tolerant consensus protocol for the BTC Federation.

### Primary Rationale
1. **Large-Scale BFT**: HotStuff is specifically designed to handle large numbers of participants efficiently
2. **Linear Communication Complexity**: O(n) communication complexity per consensus round, optimal for 512 participants
3. **Deterministic Finality**: Provides clear finality guarantees required for federation coordination
4. **Security Model Alignment**: BFT properties complement Bitcoin Script's limited consensus primitives
5. **Academic Foundation**: Proven theoretical foundation with formal security analysis
6. **Industry Adoption**: Successfully deployed in production blockchain systems (Libra/Diem)

### Technical Specifications
- **Protocol**: HotStuff BFT consensus algorithm
- **Fault Tolerance**: Byzantine fault tolerant up to f < n/3 malicious participants
- **Communication Complexity**: O(n) messages per consensus round
- **Finality**: Deterministic finality after 3 phases (prepare, pre-commit, commit)
- **Leader Rotation**: Built-in leader rotation for liveness guarantees
- **Target Scale**: Optimized for 512 participants (tolerates up to 170 Byzantine nodes)

## Consequences

### Positive
- **Scalability**: Linear communication complexity enables 512+ participant federations
- **Deterministic Finality**: Clear finality guarantees for federation operation coordination
- **Byzantine Fault Tolerance**: Robust against up to 1/3 malicious participants
- **Security Model Fit**: Complements Bitcoin Script's limited consensus primitives with off-chain coordination
- **Proven Design**: Academic rigor with formal security proofs and production deployment history
- **Leader Rotation**: Built-in liveness guarantees through automatic leader rotation
- **Low-Frequency Design**: Optimized for federation coordination operations

### Negative
- **Implementation Complexity**: More complex than simple majority voting schemes
- **3-Phase Commitment**: Requires 3 communication phases for finality (higher latency than 1-phase schemes)
- **Leader Dependency**: Temporary progress dependency on current leader (mitigated by rotation)
- **Network Partitions**: Cannot make progress during network partitions affecting >1/3 of participants
- **State Synchronization**: Requires robust state synchronization for participants rejoining after downtime

### Security Considerations
- **Byzantine Threshold**: Security requires f < n/3, meaning maximum 170 Byzantine nodes in 512 participant federation
- **Cryptographic Security**: Security relies on the underlying cryptographic primitives and HotStuff's BFT guarantees
- **Network Assumptions**: Assumes partial synchrony for liveness (bounded message delays)
- **Leader Election**: Secure leader rotation critical for preventing targeted attacks
- **State Consistency**: Requires secure state synchronization to prevent inconsistency attacks

### Operational Considerations
- **Participant Availability**: Requires >2/3 participants online for progress
- **Network Requirements**: Reliable networking between all participants
- **Monitoring**: Comprehensive monitoring required for consensus health and Byzantine behavior detection
- **Recovery Procedures**: Well-defined procedures for handling network partitions and participant failures

## Implementation Requirements

### Integration Points
- **P2P Network**: Integration with federation P2P networking layer from [PRD-02 P2P Network Stack](/workflow/prd/federation/02_p2p_network_stack.md)
- **Monitoring**: Integration with logging system from [PRD-04 Logging System](/workflow/prd/federation/04_logging_system_implementation.md)

### Performance Targets
- **Consensus Latency**: Target 3-10 seconds for finality in 512 participant federation
- **Throughput**: Support federation operation rates (low-frequency design)
- **Scalability**: Maintain performance up to 512 participants
- **Availability**: >99% uptime with >2/3 participant availability

## References
- **FROST Analysis**: [PRD-03 MuSig2 vs FROST Comparative Analysis](/workflow/prd/federation/03_musig2_frost_comparative_analysis.md)
- **Bitcoin Security Model**: BIP-340 (Schnorr Signatures), BIP-341 (Taproot)
- **Related Architecture**: [BTC Federation Entity](/architecture/common/entities/federation_btc.md)