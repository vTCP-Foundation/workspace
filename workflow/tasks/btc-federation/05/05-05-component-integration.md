# 05-05 - Component Integration Framework

# Links
- [PRD](/workflow/prd/federation/05_hotstuff_consensus.md)
- [Previous task](/workflow/tasks/federation/05/05-04-mock-infrastructure.md)
- [ADR-005 HotStuff Consensus Protocol](/architecture/federation/adrs/ADR-005-hotstuff-consensus-protocol.md)

# Description
Integrate all consensus components with a clean dependency injection framework, implementing message handling, phase management, view management, and leader election to create a cohesive consensus system.

# Requirements and DOD
- **MessageHandler**: Route and process different consensus message types
- **PhaseManager**: Coordinate state transitions between consensus phases  
- **ViewManager**: Handle view advancement, timeouts, and leader rotation
- **LeaderElection**: Implement round-robin leader selection mechanism
- **Dependency Injection**: Wire all components together with clean interfaces
- **Component Lifecycle**: Proper initialization, startup, and shutdown procedures
- **Documentation**: All integration components documented with clear responsibilities
- **Demo**: Working demonstration of integrated consensus system processing messages

# Implementation Plan

## Step 1: MessageHandler Implementation
- Implement `MessageHandler` struct with fields:
  - `consensus *HotStuffConsensus` - Main consensus coordinator
  - `phaseManager *PhaseManager` - Phase transition controller
  - `viewManager *ViewManager` - View and timeout management
  - `network NetworkInterface` - Network communication layer
- Add methods:
  - `HandleProposal(msg *ProposalMsg)` - Process block proposals
  - `HandleVote(msg *VoteMsg)` - Process vote messages
  - `HandleTimeout(msg *TimeoutMsg)` - Process view timeout messages
  - `HandleNewView(msg *NewViewMsg)` - Process new view announcements
  - `Start()` and `Stop()` for lifecycle management

## Step 2: PhaseManager Implementation
- Implement `PhaseManager` struct with fields:
  - `currentPhase ConsensusPhase` - Current consensus phase
  - `consensus *HotStuffConsensus` - State machine reference
  - `viewManager *ViewManager` - View coordination
- Add methods:
  - `ProcessPrepare(proposal *ProposalMsg)` - Handle prepare phase
  - `ProcessPreCommit(qc *QuorumCertificate)` - Handle precommit phase  
  - `ProcessCommit(qc *QuorumCertificate)` - Handle commit phase
  - `AdvancePhase(newPhase ConsensusPhase)` - Transition between phases
  - `GetCurrentPhase() ConsensusPhase` - Get current phase

## Step 3: ViewManager Implementation
- Implement `ViewManager` struct with fields:
  - `currentView ViewNumber` - Current view number
  - `viewTimeout time.Duration` - Timeout for current view
  - `timer *time.Timer` - View timeout timer
  - `leaderElection *LeaderElection` - Leader selection component
- Add methods:
  - `AdvanceView()` - Move to next view
  - `ResetTimeout()` - Reset view timeout timer
  - `OnTimeout()` - Handle view timeout events
  - `GetCurrentLeader() NodeID` - Get leader for current view
  - `IsLeader(nodeID NodeID) bool` - Check if node is current leader

## Step 4: LeaderElection Implementation
- Implement `LeaderElection` struct with fields:
  - `validators []NodeID` - List of validator nodes
  - `currentView ViewNumber` - Current view for leader calculation
- Add methods:
  - `GetLeader(view ViewNumber) NodeID` - Get leader for specific view
  - `IsValidLeader(nodeID NodeID, view ViewNumber) bool` - Validate leader
  - `GetNextLeader(currentLeader NodeID) NodeID` - Get next leader in rotation
  - Round-robin selection: `leader = validators[view % len(validators)]`

## Step 5: Dependency Injection Container
- Implement `ConsensusNode` struct as main container:
  - `nodeID NodeID` - This node's identifier
  - `consensus *HotStuffConsensus` - Core consensus state machine
  - `messageHandler *MessageHandler` - Message routing
  - `phaseManager *PhaseManager` - Phase coordination
  - `viewManager *ViewManager` - View management
  - `leaderElection *LeaderElection` - Leader selection
  - `network NetworkInterface` - Network layer
  - `storage StorageInterface` - Storage layer
  - `crypto CryptoInterface` - Crypto layer
- Add methods:
  - `NewConsensusNode(config NodeConfig) *ConsensusNode` - Constructor with DI
  - `Start() error` - Initialize and start all components
  - `Stop() error` - Gracefully shutdown all components
  - `ProcessMessage(msg ConsensusMessage)` - Main message entry point

## Step 6: Demo Implementation
- Create demo showing:
  1. Multi-node consensus network instantiation
  2. Message flow through integrated components
  3. Phase transitions and view advancement
  4. Leader election and rotation
  5. Complete consensus round from proposal to commitment
  6. Graceful startup and shutdown procedures

# Test Plan
Testing will be handled in dedicated Task 05-09 (Integration tests for component wiring). This task focuses solely on implementation and demo validation.

# Verification and Validation

## Architecture integrity
- Clean dependency injection without circular dependencies
- Proper separation of concerns between components
- Interface-based design enables easy testing and mocking

## Security
- All message processing validates sender and content
- State transitions enforce safety and liveness properties
- Component isolation prevents cross-contamination

## Performance
- Efficient message routing without unnecessary copying
- Minimal lock contention between components
- Fast component initialization and shutdown

## Reliability
- Graceful error handling across component boundaries
- Proper resource cleanup during shutdown
- Deterministic component initialization order

## Maintainability
- Clear component responsibilities and interfaces
- Easy to add new components or replace existing ones
- Well-documented component lifecycle and interactions

# Restrictions
- Commit changes only after successfully executing the demo
- All components must be properly integrated through interfaces
- Focus solely on implementation - testing handled in Task 05-09
- All public APIs must be documented with Go doc comments