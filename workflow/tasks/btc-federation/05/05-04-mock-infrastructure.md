# 05-04 - Mock Infrastructure Components

# Links
- [PRD](/workflow/prd/btc-federation/05_hotstuff_consensus.md)
- [Previous task](/workflow/tasks/btc-federation/05/05-03-abstraction-interfaces.md)
- [ADR-005 HotStuff Consensus Protocol](/architecture/btc-federation/adrs/ADR-005-hotstuff-consensus-protocol.md)

# Description
Implement mock/emulated infrastructure components that satisfy the Network, Storage, and Crypto interfaces defined in Task 05-03. These mocks enable consensus testing without real network, persistence, or cryptographic dependencies.

# Requirements and DOD
- **MockNetwork**: Channel-based message passing with configurable delays and failures
- **MockStorage**: In-memory persistence with optional durability simulation
- **MockCrypto**: Simple signature operations for testing without real cryptography
- **Configuration System**: Configurable parameters for emulation behavior
- **Failure Injection**: Ability to simulate network partitions, storage failures, crypto errors
- **Documentation**: All mock implementations documented with usage examples
- **Demo**: Working demonstration of all mocks integrated with consensus components

# Implementation Plan

## Step 1: MockNetwork Implementation
- Implement `MockNetwork` struct with fields:
  - `nodeID NodeID` - This node's identifier
  - `nodes map[NodeID]*MockNetwork` - Connected peer nodes
  - `messageQueue chan ReceivedMessage` - Incoming message queue
  - `config NetworkConfig` - Configuration parameters
  - `failures FailureConfig` - Failure injection settings
- Add methods implementing `NetworkInterface`:
  - `Send()` with configurable delays and packet loss
  - `Broadcast()` with selective delivery to simulate partitions
  - `Receive()` returning message channel
  - Failure injection for dropped messages, delays, duplicates

## Step 2: MockStorage Implementation
- Implement `MockStorage` struct with fields:
  - `blocks map[BlockHash]*Block` - In-memory block storage
  - `qcs map[string]*QuorumCertificate` - QC storage (keyed by block+phase)
  - `currentView ViewNumber` - Persisted view number
  - `config StorageConfig` - Configuration parameters
  - `failures FailureConfig` - Failure injection settings
- Add methods implementing `StorageInterface`:
  - All persistence operations with optional failure simulation
  - Data corruption simulation for robustness testing
  - Configurable persistence delays

## Step 3: MockCrypto Implementation
- Implement `MockCrypto` struct with fields:
  - `nodeID NodeID` - This node's identifier
  - `keys map[NodeID][]byte` - Mock public keys for all nodes
  - `signatures map[string][]byte` - Pre-computed signatures for deterministic testing
  - `config CryptoConfig` - Configuration parameters
- Add methods implementing `CryptoInterface`:
  - `Sign()` returning deterministic mock signatures
  - `Verify()` with configurable success/failure patterns
  - `Hash()` using real SHA-256 for consistency
  - No aggregation methods (individual signatures only)

## Step 4: Configuration System
- Implement configuration structs:
  - `NetworkConfig`: Message delays, loss rates, partition scenarios
  - `StorageConfig`: Persistence delays, failure rates, corruption scenarios
  - `CryptoConfig`: Signature verification patterns, key management
  - `FailureConfig`: Unified failure injection configuration
- Add validation and default value handling

## Step 5: Multi-Node Test Environment
- Implement `TestEnvironment` struct for orchestrating multiple consensus nodes:
  - `CreateNodes(count int) []*MockConsensusNode` - Create network of nodes
  - `ConnectNodes()` - Establish mock network connections
  - `SimulatePartition(nodeIds []NodeID)` - Partition network
  - `InjectFailures(failures FailureConfig)` - Apply failure scenarios
  - `StepTime(duration time.Duration)` - Advance simulated time

## Step 6: Demo Implementation
- Create demo showing:
  1. Multi-node network creation with mock infrastructure
  2. Message passing between consensus nodes
  3. Block persistence and retrieval
  4. Signature operations and verification
  5. Failure injection (network partition, storage failure)
  6. Recovery from failure scenarios

# Test Plan
Testing will be handled in dedicated Task 05-08 (Unit tests for interfaces and mocks). This task focuses solely on implementation and demo validation.

# Verification and Validation

## Architecture integrity
- Mock implementations fully satisfy defined interfaces
- Clean separation between mock logic and test scenarios
- Configurable behavior without breaking interface contracts

## Security
- Mock crypto maintains signature verification patterns
- No real cryptographic vulnerabilities in test infrastructure
- Failure scenarios don't compromise test isolation

## Performance
- Efficient in-memory operations for fast test execution
- Configurable delays don't block test execution unnecessarily
- Minimal overhead in mock implementations

## Reliability
- Deterministic behavior for reproducible tests
- Proper cleanup of resources between test runs
- Graceful handling of configuration edge cases

## Maintainability
- Clear configuration options for different test scenarios
- Well-documented mock behavior and limitations
- Easy to extend with new failure scenarios

# Restrictions
- Commit changes only after successfully executing the demo
- Mock implementations must fully satisfy interface contracts
- Focus solely on implementation - testing handled in Task 05-08
- All public APIs must be documented with Go doc comments