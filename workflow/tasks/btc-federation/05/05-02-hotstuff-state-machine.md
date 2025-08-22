# 05-02 - HotStuff State Machine

# Links
- [PRD](/workflow/prd/btc-federation/05_hotstuff_consensus.md)
- [Previous task](/workflow/tasks/btc-federation/05/05-01-consensus-foundation-setup.md)
- [ADR-005 HotStuff Consensus Protocol](/architecture/btc-federation/adrs/ADR-005-hotstuff-consensus-protocol.md)

# Description
Implement the core HotStuff consensus state machine including the main coordinator, block tree management, safety rules, voting rules, and basic state transitions for the three consensus phases.

# Requirements and DOD
- **HotStuffConsensus**: Main coordinator struct managing consensus state and orchestrating components
- **BlockTree**: Implementation with fork management, ancestor tracking, and safe block advancement
- **SafetyRules**: Component preventing conflicting votes and ensuring safety properties
- **VotingRule**: Component handling vote validation and quorum certificate formation
- **State Transitions**: Implement Prepare → PreCommit → Commit phase transitions
- **View Management**: Basic view number tracking and advancement logic
- **Documentation**: All public types and methods have Go doc comments
- **Demo**: Working demonstration of state machine processing consensus messages

# Implementation Plan

## Step 1: HotStuffConsensus Main Coordinator
- Implement `HotStuffConsensus` struct with fields:
  - `nodeID NodeID` - This node's identifier
  - `view ViewNumber` - Current view number
  - `blockTree *BlockTree` - Block tree manager
  - `safetyRules *SafetyRules` - Safety validation component
  - `votingRule *VotingRule` - Vote processing component
  - `lockedQC *QuorumCertificate` - Highest locked QC
  - `prepareQC *QuorumCertificate` - Highest prepare QC
- Add methods: `ProcessProposal()`, `ProcessVote()`, `ProcessTimeout()`, `AdvanceView()`

## Step 2: BlockTree Implementation
- Implement `BlockTree` struct with fields:
  - `blocks map[BlockHash]*Block` - All known blocks
  - `children map[BlockHash][]*Block` - Parent-to-children mapping
  - `root *Block` - Genesis block
  - `committed *Block` - Highest committed block
- Add methods: `AddBlock()`, `GetBlock()`, `GetChildren()`, `GetAncestors()`, `IsAncestor()`, `FindLCA()`

## Step 3: SafetyRules Implementation
- Implement `SafetyRules` struct with fields:
  - `lockedQC *QuorumCertificate` - Locked QC for safety
  - `prepareQC *QuorumCertificate` - Prepare QC for liveness
- Add methods: `CanVote()`, `UpdateLockedQC()`, `UpdatePrepareQC()`, `ValidateProposal()`

## Step 4: VotingRule Implementation
- Implement `VotingRule` struct with fields:
  - `threshold int` - Votes needed for quorum (2f+1)
  - `pendingVotes map[BlockHash]map[ConsensusPhase][]Vote` - Vote collection
- Add methods: `AddVote()`, `CheckQuorum()`, `FormQC()`, `ValidateVote()`

## Step 5: State Transition Logic
- Implement three-phase consensus flow:
  - **Prepare Phase**: Process block proposals, validate with safety rules
  - **PreCommit Phase**: Process prepare QCs, advance locked QC
  - **Commit Phase**: Process precommit QCs, commit blocks
- Add view advancement logic for timeouts and successful rounds

## Step 6: Demo Implementation
- Create demo showing:
  1. Block proposal processing in prepare phase
  2. Vote collection and QC formation
  3. State transitions through all three phases
  4. Block commitment and view advancement
  5. Fork handling and safety rule enforcement

# Test Plan
Testing will be handled in dedicated Task 05-07 (Unit tests for core data structures). This task focuses solely on implementation and demo validation.

# Verification and Validation

## Architecture integrity
- Clean separation between coordinator, tree, safety, and voting components
- Interfaces designed for testability and mock injection

## Security
- Safety rules prevent double voting and conflicting commitments
- All state transitions validate prerequisites before advancing

## Performance
- Efficient block tree operations with hash-based lookups
- Minimal copying of data structures during state transitions

## Reliability
- State machine handles invalid inputs gracefully
- Deterministic behavior for identical input sequences

## Maintainability
- Modular design enables independent component testing
- Clear method boundaries and responsibilities

# Restrictions
- Commit changes only after successfully executing the demo
- Focus solely on implementation - testing handled in Task 05-07
- All public APIs must be documented with Go doc comments