# 05-02 - Event-Based Testing Infrastructure

**Status**: 5.2 - Task demo validated  
**Demo Location**: [`cmd/demo/hotstuff/event-tracer-demo/`](/cmd/demo/hotstuff/event-tracer-demo/)

# Links
- [PRD](/workflow/prd/btc-federation/05_hotstuff_consensus.md)
- [ADR-007 Event-Based Consensus Testing](/architecture/btc-federation/adrs/ADR-007-event-based-consensus-testing.md)
- [Data Flow Diagram](/architecture/btc-federation/protocols/hot-stuff-consensus/data-flow.mermaid)

# Description
Implement the event-based testing infrastructure from ADR-007 to enable comprehensive flow-by-flow testing of the HotStuff consensus protocol with complete path coverage and Byzantine behavior separation.

# Requirements and DOD
- **Event Tracer Interface**: Production-safe interface with no-op implementation
- **Event Collection**: Centralized event aggregation for multi-node tests
- **Validation Rules Engine**: Comprehensive rule-based event sequence validation
- **Build Tag Separation**: Byzantine behaviors isolated from production code
- **Mock Infrastructure Integration**: Event tracing added to existing mock components
- **Complete Rule Set**: Happy path, Byzantine detection, and partition recovery rules
- **Zero Production Overhead**: No-op implementation has no performance impact
- **Demo Validation**: Comprehensive demo proves all requirements and validates infrastructure readiness

# Implementation Plan

## Step 1: Event Tracer Interface Implementation
Create production-safe event tracing interfaces:

```go
// pkg/consensus/interfaces/event_tracer.go
type EventTracer interface {
    RecordEvent(nodeID uint16, eventType EventType, payload EventPayload)
    RecordTransition(nodeID uint16, from, to State, trigger string)
    RecordMessage(nodeID uint16, direction MessageDirection, msgType string, payload EventPayload)
}

// Production implementation (zero overhead)
type NoOpEventTracer struct{}
func (t *NoOpEventTracer) RecordEvent(nodeID uint16, eventType EventType, payload EventPayload) {}

// Testing implementation (centralized collection)
type ConsensusEventTracer struct {
    events []ConsensusEvent
    mutex  sync.RWMutex
}
```

## Step 2: Event Types and Payload System
Define comprehensive event taxonomy:

```go
// pkg/consensus/interfaces/events.go
type EventType string

const (
    // Proposal Events
    EventProposalCreated   EventType = "proposal_created"
    EventProposalReceived  EventType = "proposal_received"
    EventProposalValidated EventType = "proposal_validated"
    EventProposalRejected  EventType = "proposal_rejected"
    
    // Vote Events
    EventVoteCreated   EventType = "vote_created"
    EventVoteSent      EventType = "vote_sent"
    EventVoteReceived  EventType = "vote_received"
    EventVoteValidated EventType = "vote_validated"
    
    // QC Events
    EventQCFormed     EventType = "qc_formed"
    EventQCReceived   EventType = "qc_received"
    EventQCValidated  EventType = "qc_validated"
    
    // View Events
    EventViewTimeout    EventType = "view_timeout"
    EventViewChange     EventType = "view_change"
    EventNewViewStarted EventType = "new_view_started"
    
    // Byzantine Events  
    EventByzantineDetected EventType = "byzantine_detected"
    EventEquivocationFound EventType = "equivocation_found"
    EventEvidenceStored    EventType = "evidence_stored"
    
    // Block Events
    EventBlockCommitted  EventType = "block_committed"
    EventBlockAdded      EventType = "block_added"
    EventBlockValidated  EventType = "block_validated"
)
```

## Step 3: Validation Rules Engine
Implement rule-based event validation system:

```go
// pkg/consensus/testing/validation.go
type ValidationRule interface {
    Validate(events []ConsensusEvent) []ValidationError
    Description() string
}

// Ordering constraints
type MustHappenBeforeRule struct {
    EventA      EventType
    EventB      EventType
    Description string
}

// Quorum requirements
type QuorumRule struct {
    EventType     EventType
    MinNodes      uint16
    Description   string
}

// Node role validation
type LeaderOnlyRule struct {
    EventType   EventType
    Description string
}

// Cross-node coordination
type CrossNodeCoordinationRule struct {
    InitiatorEvent EventType
    ResponseEvent  EventType
    MinResponders  uint16
    MaxDelay       time.Duration
    Description    string
}
```

## Step 4: Build Tag Structure for Byzantine Behaviors
Implement build constraints for testing-only code:

```go
// pkg/consensus/testing/byzantine.go
//go:build testing
// +build testing

// Byzantine behaviors only available in testing builds
func (cn *ConsensusNode) EnableByzantineBehavior(behavior ByzantineBehavior) {
    cn.byzantineMode = behavior
}

type ByzantineBehavior string

const (
    DoubleVoteMode     ByzantineBehavior = "double_vote"
    InvalidQCMode      ByzantineBehavior = "invalid_qc"
    ConflictingProposalMode ByzantineBehavior = "conflicting_proposal"
    MessageWithholdingMode  ByzantineBehavior = "message_withholding"
    TimingAttackMode        ByzantineBehavior = "timing_attack"
)
```

## Step 5: Mock Infrastructure Integration
Extend existing mock components with event tracing:

```go
// pkg/consensus/mocks/mock_network.go
type MockNetwork struct {
    nodeID      types.NodeID
    eventTracer interfaces.EventTracer
    // ... existing fields
}

func (mn *MockNetwork) Send(msg interface{}) error {
    if mn.eventTracer != nil {
        mn.eventTracer.RecordMessage(uint16(mn.nodeID), interfaces.MessageOutbound, 
            reflect.TypeOf(msg).Name(), interfaces.EventPayload{
                "destination": msg.GetDestination(),
                "message_type": msg.GetType(),
            })
    }
    // ... existing logic
}
```

## Step 6: Complete Rule Set Implementation
Implement comprehensive validation rule sets:

### Happy Path Rules
```go
func GetHappyPathRules(nodeCount, faultyNodes uint16) []ValidationRule {
    quorumSize := 2*faultyNodes + 1
    
    return []ValidationRule{
        &MustHappenBeforeRule{
            EventA:      "proposal_received",
            EventB:      "vote_sent", 
            Description: "Proposal must be received before voting",
        },
        &QuorumRule{
            EventType:   "vote_sent",
            MinNodes:    quorumSize,
            Description: "Quorum of nodes must vote",
        },
        &LeaderOnlyRule{
            EventType:   "proposal_created",
            Description: "Only leader should create proposals",
        },
        // ... additional happy path rules
    }
}
```

### Byzantine Detection Rules
```go
func GetByzantineDetectionRules() []ValidationRule {
    return []ValidationRule{
        &CrossNodeCoordinationRule{
            InitiatorEvent: "byzantine_behavior_occurred",
            ResponseEvent:  "byzantine_detected",
            MinResponders:  2,
            MaxDelay:       10 * time.Second,
            Description:    "Byzantine behavior must be detected by honest nodes",
        },
        // ... additional Byzantine rules
    }
}
```

## Step 7: Test Framework Integration
Create helper functions for test setup:

```go
// pkg/consensus/testing/helpers.go
func createTestNetwork(nodeCount int, tracer interfaces.EventTracer) map[types.NodeID]*integration.HotStuffCoordinator {
    // Setup network with event tracing enabled
}

func executeConsensusRound(nodes map[types.NodeID]*integration.HotStuffCoordinator, payload []byte) error {
    // Execute complete consensus round with event collection
}

func ValidateEvents(events []ConsensusEvent, rules []ValidationRule) []ValidationError {
    // Run all validation rules against collected events
}
```

# Verification and Validation

## Architecture integrity
- Clean separation between production and testing code
- Interface-based design maintains existing code style
- Build tags prevent Byzantine code in production builds

## Security
- Zero attack surface in production (no-op tracer)
- Byzantine behaviors completely isolated
- No cryptographic weaknesses introduced

## Performance  
- Production overhead: zero (interface optimized away)
- Test performance: minimal impact from event collection
- Efficient rule validation algorithms

## Reliability
- Comprehensive event validation catches protocol violations
- Complete path coverage ensures robustness
- Rule-based approach enables systematic testing

# Restrictions
- Must achieve zero production overhead
- All Byzantine behaviors must use build tags
- Event collection only in testing environment
- Complete integration with existing mock infrastructure