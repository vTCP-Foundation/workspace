# 05-03 - Abstraction Interfaces

# Links
- [PRD](/workflow/prd/federation/05_hotstuff_consensus.md)
- [Previous task](/workflow/tasks/federation/05/05-01-consensus-foundation-setup.md)
- [ADR-005 HotStuff Consensus Protocol](/architecture/federation/adrs/ADR-005-hotstuff-consensus-protocol.md)

# Description
Define clean abstraction interfaces for Network, Storage, and Crypto layers to enable the consensus engine to operate independently of specific implementations. Design interfaces without signature aggregation for future compatibility.

# Requirements and DOD
- **NetworkInterface**: Define methods for message transmission between consensus nodes
- **StorageInterface**: Define methods for persisting consensus state and block data
- **CryptoInterface**: Define methods for signature operations without aggregation support
- **Error Types**: Comprehensive error handling patterns for all interface operations
- **Interface Documentation**: All interfaces have clear contracts and usage examples
- **Demo**: Simple demonstration showing interface usage patterns

# Implementation Plan

## Step 1: NetworkInterface Definition
- Define `NetworkInterface` with methods:
  - `Send(nodeID NodeID, message ConsensusMessage) error` - Send message to specific node
  - `Broadcast(message ConsensusMessage) error` - Broadcast message to all nodes
  - `Receive() <-chan ReceivedMessage` - Channel for incoming messages
  - `SetMessageHandler(handler MessageHandler)` - Set message processing handler
  - `GetConnectedNodes() []NodeID` - Get list of connected peer nodes
- Define `ReceivedMessage` struct with sender and message content

## Step 2: StorageInterface Definition
- Define `StorageInterface` with methods:
  - `StoreBlock(block *Block) error` - Persist block to storage
  - `GetBlock(hash BlockHash) (*Block, error)` - Retrieve block by hash
  - `StoreQC(qc *QuorumCertificate) error` - Persist quorum certificate
  - `GetQC(blockHash BlockHash, phase ConsensusPhase) (*QuorumCertificate, error)` - Retrieve QC
  - `StoreView(view ViewNumber) error` - Persist current view number
  - `GetCurrentView() (ViewNumber, error)` - Retrieve current view
  - `GetBlocksByHeight(height Height) ([]*Block, error)` - Get blocks at specific height

## Step 3: CryptoInterface Definition (No Aggregation)
- Define `CryptoInterface` with methods:
  - `Sign(data []byte) ([]byte, error)` - Sign data with node's private key
  - `Verify(data []byte, signature []byte, nodeID NodeID) error` - Verify signature
  - `GetPublicKey(nodeID NodeID) ([]byte, error)` - Get public key for node
  - `Hash(data []byte) BlockHash` - Compute cryptographic hash
- **No signature aggregation methods** - individual signatures only

## Step 4: Error Type Definitions
- Define comprehensive error types:
  - `NetworkError` - Connection, timeout, message delivery failures
  - `StorageError` - Persistence, retrieval, corruption errors
  - `CryptoError` - Signature verification, key management errors
  - `ValidationError` - Protocol rule violations
- Add error wrapping and context preservation

## Step 5: Configuration Interfaces
- Define `NetworkConfig` for network layer configuration
- Define `StorageConfig` for storage layer configuration
- Define `CryptoConfig` for cryptographic configuration
- Add validation methods for all configurations

## Step 6: Demo Implementation
- Create demo showing:
  1. Interface instantiation with mock implementations
  2. Message handling through NetworkInterface
  3. Block persistence through StorageInterface
  4. Signature operations through CryptoInterface
  5. Error handling patterns across all interfaces

# Test Plan
Testing will be handled in dedicated Task 05-08 (Unit tests for interfaces and mocks). This task focuses solely on interface definition and demo validation.

# Verification and Validation

## Architecture integrity
- Interfaces follow Go idioms and conventions
- Clear separation of concerns between layers
- No dependencies between interface definitions

## Security
- CryptoInterface designed for post-quantum signature schemes
- No signature aggregation methods that would be incompatible
- Proper error handling prevents information leakage

## Performance
- Interface methods designed for efficient implementations
- Minimal data copying in method signatures
- Async patterns where appropriate (e.g., Receive channel)

## Reliability
- Comprehensive error types cover all failure modes
- Interface contracts clearly specify behavior
- No undefined behavior in interface specifications

## Maintainability
- Clean interface boundaries enable independent testing
- Well-documented contracts reduce integration complexity
- Extensible design allows future enhancement

# Restrictions
- Commit changes only after successfully executing the demo
- No signature aggregation methods in CryptoInterface
- Focus solely on interface definition - testing handled in Task 05-08
- All public interfaces must be documented with Go doc comments