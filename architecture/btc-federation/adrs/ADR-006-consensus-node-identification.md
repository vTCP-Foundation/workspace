# ADR-006: Consensus Node Identification and Configuration

## Status
Accepted

## Date
2025-08-18

## Context

During the implementation of the HotStuff consensus protocol foundation (Task 05-01), several design decisions needed to be made regarding node identification and consensus configuration management. The initial implementation used string-based NodeIDs, but further analysis revealed significant efficiency and design benefits from using numeric identifiers tied to a static consensus configuration.

### Problem Statement

1. **Node Identification**: How should consensus participants be uniquely identified within the protocol?
2. **Bitmap Efficiency**: How can we efficiently represent voter participation in QuorumCertificates?
3. **Configuration Management**: How should the consensus protocol manage the known set of participants?
4. **Quorum Calculations**: How should Byzantine Fault Tolerance thresholds be calculated and enforced?

### Initial Implementation Issues

The initial string-based NodeID approach had several limitations:
- **Memory Inefficiency**: Variable-length strings consume more memory than fixed-size identifiers
- **Bitmap Complexity**: Creating efficient voter bitmaps requires complex string-to-index mapping
- **Validation Overhead**: String validation is more complex than numeric range validation
- **No Static Configuration**: Lack of known participant set made quorum validation ad-hoc

## Decision

We adopt a **numeric NodeID approach with static consensus configuration** based on the following design:

### 1. NodeID Type Definition
```go
// NodeID represents a unique identifier for a consensus participant.
// It corresponds to the node's index in the consensus configuration.
type NodeID uint16
```

**Rationale:**
- `uint16` supports up to 65,535 nodes, sufficient for practical Byzantine consensus networks
- Fixed 2-byte size is memory efficient
- Direct mapping to array indices enables O(1) bitmap operations
- Simple numeric comparison for validation

### 2. Static Consensus Configuration with Public Key Mapping
```go
type PublicKey []byte  // Public key as byte array (will be replaced with proper crypto type)

type ConsensusConfig struct {
    Participants []NodeID              // Ordered list of all participants
    PublicKeys   map[NodeID]PublicKey  // Maps NodeID to public key
}

// Computed properties (no fields to avoid manual setting errors)
func (c *ConsensusConfig) TotalNodes() int        // len(c.Participants)
func (c *ConsensusConfig) FaultyNodes() int       // (TotalNodes-1)/3
func (c *ConsensusConfig) QuorumThreshold() int   // 2*FaultyNodes+1

// Public key management
func (c *ConsensusConfig) GetPublicKey(NodeID) (PublicKey, error)
func (c *ConsensusConfig) HasPublicKey(NodeID) bool
```

**Rationale:**
- **Static Configuration**: Known participant set enables proper validation
- **Cryptographic Mapping**: Each NodeID has corresponding public key for signature verification
- **Computed Properties**: Eliminates error-prone manual parameter setting
- **Byzantine Calculations**: Automatically enforces classic BFT formula n = 3f + 1
- **DRY Principle**: Single source of truth (Participants) for all derived values
- **Immutable Consistency**: Cannot have mismatched TotalNodes vs Participants length
- **Security Foundation**: Enables proper signature validation in consensus protocol

### 3. Efficient Voter Bitmaps
```go
func (c *ConsensusConfig) CreateVoterBitmap(voters []NodeID) ([]byte, error)
func (c *ConsensusConfig) ParseVoterBitmap(bitmap []byte) []NodeID
```

**Rationale:**
- **Direct Mapping**: NodeID directly maps to bit position
- **Compact Representation**: 1 bit per node vs. variable bytes per string
- **Fast Operations**: Bitwise operations for membership testing
- **Network Efficiency**: Smaller message sizes for QC transmission

### 4. Enhanced Validation
All validation methods now accept `*ConsensusConfig` parameter:
```go
func (b *Block) Validate(config *ConsensusConfig) error
func (v *Vote) Validate(config *ConsensusConfig) error  
func (qc *QuorumCertificate) Validate(config *ConsensusConfig) error
```

**Rationale:**
- **Proper Quorum Validation**: Enforces 2f+1 threshold using `config.QuorumThreshold()`
- **NodeID Range Validation**: Ensures NodeIDs are within valid range using `config.TotalNodes()`
- **Consistent Configuration**: Single source of truth for network parameters
- **Error Prevention**: Computed properties prevent manual configuration errors

## Implementation Details

### NodeID Assignment and Public Key Mapping
Nodes are assigned sequential NodeIDs starting from 0:
- Genesis validator: NodeID 0
- Additional validators: NodeID 1, 2, 3, ...
- NodeIDs correspond to indices in `ConsensusConfig.Participants`
- Each NodeID has a corresponding entry in `ConsensusConfig.PublicKeys`

#### Public Key Management
```go
// Constructor with provided public keys
config, err := NewConsensusConfig([]PublicKey{pubKey0, pubKey1, pubKey2, pubKey3, pubKey4})

// Constructor with mock keys for testing
config, err := NewConsensusConfigForTesting(5)

// Retrieve public key for signature verification
pubKey, err := config.GetPublicKey(nodeID)
```

### Quorum Calculation
For n nodes in Byzantine Fault Tolerant consensus:
- Maximum faulty nodes: `f = (n-1) / 3`
- Quorum threshold: `2f + 1`
- Examples:
  - 4 nodes: f=1, quorum=3
  - 5 nodes: f=1, quorum=3  
  - 7 nodes: f=2, quorum=5

### Bitmap Representation
Voter participation bitmap:
- Bitmap size: `(totalNodes + 7) / 8` bytes (rounded up)
- Bit position: `nodeID % 8` within byte `nodeID / 8`
- Set bit indicates node participation in vote

## Consequences

### Positive
- **Memory Efficiency**: 2-byte NodeIDs vs. variable-length strings
- **Bitmap Performance**: O(1) bit operations vs. O(n) string lookups
- **Network Efficiency**: Compact bitmap representation in QuorumCertificates
- **Validation Robustness**: Proper BFT quorum enforcement
- **Configuration Clarity**: Static participant set eliminates ambiguity
- **Error Prevention**: Computed properties eliminate manual configuration errors
- **Immutable Consistency**: Cannot have mismatched field values vs. participant count
- **DRY Compliance**: Single source of truth for all configuration parameters
- **Cryptographic Foundation**: Direct NodeID-to-PublicKey mapping enables signature verification
- **Security Readiness**: Prepared for full cryptographic signature validation
- **Testing Support**: Mock key generation for development and testing
- **Scalability**: Supports large consensus networks efficiently

### Negative
- **Static Configuration**: Requires known participant set at initialization
- **Limited Flexibility**: Harder to add/remove nodes dynamically (addressed in future ADRs)
- **Uint16 Limit**: Maximum 65,535 nodes (acceptable for current use cases)

### Neutral
- **Migration Path**: Existing string-based systems need conversion logic
- **Documentation**: Requires clear mapping between logical node names and NodeIDs

## Related Decisions

- **ADR-005**: HotStuff Consensus Protocol - Defines overall consensus mechanism
- **Future ADR**: Dynamic Node Management - Will address node addition/removal

## Implementation Status

- ✅ NodeID type changed to uint16
- ✅ ConsensusConfig struct implemented with computed properties
- ✅ Public key mapping added (NodeID → PublicKey)
- ✅ Automatic Byzantine fault tolerance calculations (TotalNodes, FaultyNodes, QuorumThreshold)
- ✅ Public key validation and retrieval methods implemented
- ✅ Bitmap operations implemented
- ✅ All validation methods updated to use computed properties
- ✅ Mock public key generation for testing
- ✅ Demo application updated and validated with public key display
- ✅ Error-prone manual configuration eliminated

## References

- [HotStuff: BFT Consensus with Linearity and Responsiveness](https://arxiv.org/abs/1803.05069)
- [Practical Byzantine Fault Tolerance](http://pmg.csail.mit.edu/papers/osdi99.pdf)
- [Task 05-01: Consensus Foundation Setup](/workflow/tasks/btc-federation/05/05-01-consensus-foundation-setup.md)