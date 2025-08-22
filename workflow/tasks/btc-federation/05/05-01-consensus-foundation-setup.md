# 05-01 - Consensus Foundation Setup

# Links
- [PRD](/workflow/prd/btc-federation/05_hotstuff_consensus.md)
- [ADR-005 HotStuff Consensus Protocol](/architecture/btc-federation/adrs/ADR-005-hotstuff-consensus-protocol.md)

# Description
Setup proper consensus-related filesystem layout and Go packages, then implement core data structures for the HotStuff consensus protocol including Block, Vote, QuorumCertificate, and message types.

# Requirements and DOD
- **FS Layout**: Create proper Go package structure under `pkg/consensus/` with subpackages for core components
- **Core Data Structures**: Implement `Block`, `Vote`, `QuorumCertificate` structs with proper field definitions
- **Type Definitions**: Define `ViewNumber`, `NodeID`, `BlockHash` types with appropriate underlying types
- **Message Types**: Create `ProposalMsg`, `VoteMsg`, `TimeoutMsg`, `NewViewMsg` message structures
- **Signature Abstraction**: Design signature handling without aggregation (individual signatures only)
- **Validation**: All data structures include basic validation methods
- **Documentation**: All public types and methods have Go doc comments
- **Demo**: Working demonstration of all data structures interacting

# Implementation Plan

## Step 1: Filesystem Layout Setup
- Create `pkg/consensus/` as main consensus package
- Create subpackages:
  - `pkg/consensus/types/` - Core data types and structures
  - `pkg/consensus/messages/` - Message definitions
  - `pkg/consensus/crypto/` - Cryptographic interfaces and types
  - `pkg/consensus/storage/` - Storage interfaces
  - `pkg/consensus/network/` - Network interfaces
  - `pkg/consensus/engine/` - Main consensus engine (placeholder for Task 05-02)

## Step 2: Core Type Definitions
- Define fundamental types in `pkg/consensus/types/`:
  - `ViewNumber uint64` - Consensus view identifier
  - `NodeID string` - Unique node identifier 
  - `BlockHash [32]byte` - SHA-256 block hash
  - `Height uint64` - Block height in chain
  - `Timestamp time.Time` - Block timestamp
  - `ConsensusPhase` enum - Prepare/PreCommit/Commit phases

## Step 3: Block Structure Implementation
- Implement `Block` struct with fields:
  - `Hash BlockHash` - Block identifier
  - `ParentHash BlockHash` - Parent block hash
  - `Height Height` - Block height
  - `View ViewNumber` - View when block was proposed
  - `Proposer NodeID` - Node that proposed this block
  - `Timestamp Timestamp` - Block creation time
  - `Payload []byte` - Opaque block data
- Add methods: `CalculateHash()`, `Validate()`, `IsGenesis()`

## Step 4: Vote Structure Implementation  
- Implement `Vote` struct with fields:
  - `BlockHash BlockHash` - Hash of block being voted on
  - `View ViewNumber` - View number for this vote
  - `Phase ConsensusPhase` - Prepare/PreCommit/Commit phase
  - `Voter NodeID` - Node casting this vote
  - `Signature []byte` - Abstract signature (no aggregation)
- Add methods: `Validate()`

## Step 5: QuorumCertificate Implementation
- Implement `QuorumCertificate` struct with fields:
  - `BlockHash BlockHash` - Block this QC certifies
  - `View ViewNumber` - View number
  - `Phase ConsensusPhase` - Consensus phase
  - `Votes []Vote` - Individual votes (no signature aggregation)
  - `VoterBitmap []byte` - Bitmap of participating voters
- Add methods: `Validate()`, `HasQuorum(threshold)`, `GetVoters()`

## Step 6: Message Type Implementation
- Implement message types in `pkg/consensus/messages/`:
  - `ProposalMsg` - Block proposal message
  - `VoteMsg` - Vote message wrapper
  - `TimeoutMsg` - View timeout message
  - `NewViewMsg` - New view announcement
- Add common interface `ConsensusMessage` with methods: `Type()`, `View()`

## Step 7: Demo Implementation
- Create demo showing:
  1. Block creation and hash calculation
  2. Vote creation for different phases
  3. QC formation from multiple votes
  4. Message type discrimination
  5. Basic consensus data flow

# Test Plan
Testing will be handled in dedicated Task 05-07 (Unit tests for core data structures). This task focuses solely on implementation and demo validation.

# Verification and Validation

## Architecture integrity
- Package structure follows Go conventions and project standards
- Clear separation of concerns between types, messages, and interfaces

## Security
- All cryptographic operations use interfaces (no hardcoded implementations)
- No signature aggregation implementations

## Performance
- Efficient data structures with minimal memory allocations
- Fast hash calculations using standard library

## Reliability
- Comprehensive input validation prevents crashes
- Deterministic operations for reproducible behavior

## Maintainability
- Clear Go documentation for all public APIs
- Modular design enables isolated testing and modification

# Restrictions
- Commit changes only after successfully executing the demo
- No signature aggregation implementations
- All public APIs must be documented with Go doc comments
- Focus solely on implementation - testing handled in Task 05-07