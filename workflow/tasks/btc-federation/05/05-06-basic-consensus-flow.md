# 05-06 - Basic Consensus Flow (Happy Path)

# Links
- [PRD](/workflow/prd/btc-federation/05_hotstuff_consensus.md)
- [Previous task](/workflow/tasks/btc-federation/05/05-05-component-integration.md)
- [ADR-005 HotStuff Consensus Protocol](/architecture/btc-federation/adrs/ADR-005-hotstuff-consensus-protocol.md)

# Description
Implement the complete basic consensus flow for the happy path scenario, including block proposal logic, vote collection, quorum certificate formation, and the three-phase consensus progression that leads to block commitment and view advancement.

# Requirements and DOD
- **Block Proposal Logic**: Leader creates and broadcasts block proposals
- **Vote Collection**: Nodes validate proposals and cast votes
- **QC Formation**: Collect ≥2f+1 votes to form quorum certificates
- **Three-Phase Flow**: Complete Prepare → PreCommit → Commit progression
- **Block Commitment**: Finalize blocks after successful commit phase
- **View Advancement**: Progress to next view after successful consensus round
- **Happy Path**: Focus on normal operation without Byzantine behavior
- **Documentation**: Clear documentation of consensus flow and state transitions
- **Demo**: Working demonstration of complete consensus round with multiple nodes

# Implementation Plan

## Step 1: Block Proposal Logic
- Implement leader proposal behavior:
  - `CreateProposal(payload []byte) *Block` - Leader creates new block
  - `BroadcastProposal(block *Block)` - Send proposal to all nodes
  - `ValidateProposalTiming()` - Ensure proposal sent at correct time
  - `CreateGenesisBlock()` - Handle special case of first block
- Add proposal validation:
  - Verify proposer is current leader
  - Check block extends valid parent
  - Validate block structure and signatures

## Step 2: Vote Collection and Validation
- Implement voting behavior:
  - `CreateVote(block *Block, phase ConsensusPhase) *Vote` - Node creates vote
  - `ValidateVoteConditions(block *Block)` - Check safety rules before voting
  - `BroadcastVote(vote *Vote)` - Send vote to all nodes
  - `CollectVotes(blockHash BlockHash, phase ConsensusPhase)` - Gather votes
- Add vote validation:
  - Verify voter is valid participant
  - Check vote phase matches current consensus phase
  - Validate vote signature and format

## Step 3: Quorum Certificate Formation
- Implement QC creation logic:
  - `CheckQuorumThreshold(votes []Vote) bool` - Verify ≥2f+1 votes
  - `FormQuorumCertificate(votes []Vote) *QuorumCertificate` - Create QC
  - `ValidateQC(qc *QuorumCertificate) error` - Validate formed QC
  - `BroadcastQC(qc *QuorumCertificate)` - Share QC with network
- Add QC validation:
  - Verify vote count meets threshold
  - Check all votes are for same block and phase
  - Validate voter bitmap consistency

## Step 4: Three-Phase Consensus Implementation
- **Prepare Phase**:
  - Leader proposes block
  - Nodes validate and vote on proposal
  - Form prepare QC from collected votes
- **PreCommit Phase**:
  - Process prepare QC and update locked QC
  - Nodes vote on precommit
  - Form precommit QC from collected votes
- **Commit Phase**:
  - Process precommit QC and prepare for commitment
  - Nodes vote on commit
  - Form commit QC and execute block commitment

## Step 5: Block Commitment and Finality
- Implement commitment logic:
  - `CommitBlock(qc *QuorumCertificate)` - Finalize block
  - `UpdateCommittedHeight(height Height)` - Track committed chain tip
  - `ExecuteBlockPayload(block *Block)` - Process block contents
  - `NotifyCommitment(block *Block)` - Signal successful commitment
- Add finality guarantees:
  - Ensure committed blocks cannot be reverted
  - Update all nodes with committed state
  - Persist commitment to storage

## Step 6: View Advancement Logic
- Implement view progression:
  - `AdvanceToNextView()` - Move to next consensus round
  - `ResetPhaseToNew()` - Reset to prepare phase for new view
  - `UpdateLeaderForView(view ViewNumber)` - Select new leader
  - `CleanupPreviousRound()` - Clear temporary state from completed round
- Add view coordination:
  - Synchronize view advancement across all nodes
  - Handle view number consistency
  - Prepare for next consensus round

## Step 7: Demo Implementation
- Create comprehensive demo showing:
  1. 5-node network initialization
  2. Genesis block creation and commitment
  3. Multiple consensus rounds with different leaders
  4. Complete three-phase flow for each round
  5. Block commitment and chain growth
  6. View advancement and leader rotation
  7. State consistency across all nodes

# Test Plan
Testing will be handled in dedicated Task 05-10 (End-to-end tests for basic consensus flow). This task focuses solely on implementation and demo validation.

# Verification and Validation

## Architecture integrity
- Consensus flow follows HotStuff protocol specification exactly
- Clean state transitions between phases
- Proper separation between proposal, voting, and commitment logic

## Security
- Safety rules prevent conflicting commitments
- Liveness properties ensure progress in normal conditions
- All state transitions validate prerequisites

## Performance
- Efficient message handling for high throughput
- Minimal latency in happy path scenarios
- Optimal vote collection and QC formation

## Reliability
- Deterministic consensus progression
- Consistent state across all participating nodes
- Robust handling of edge cases in normal operation

## Maintainability
- Clear separation of consensus phases
- Well-documented state machine transitions
- Easy to trace message flow and state changes

# Restrictions
- Commit changes only after successfully executing the demo
- Focus on happy path - Byzantine behavior handled in later tasks
- Focus solely on implementation - testing handled in Task 05-10
- All public APIs must be documented with Go doc comments