# ADR-007: Event-Based Consensus Testing Framework

## Status
Accepted

## Date
2025-08-26

## Author(s)
Dima Chizhevsky, Mykola Ilashchuk

## Context

The HotStuff consensus protocol implementation requires comprehensive testing that validates all possible execution paths, including Byzantine attack vectors and edge cases. Traditional testing approaches face several challenges in distributed consensus systems:

### Problem Statement

1. **Path Coverage Complexity**: Consensus protocols have numerous execution paths with complex interdependencies across distributed nodes
2. **Byzantine Behavior Testing**: Need to test malicious behaviors without including attack code in production builds
3. **Distributed Event Validation**: Events occur across multiple nodes in a shared runtime, requiring coordinated validation
4. **Attack Vector Coverage**: Must ensure 100% coverage of all possible logic paths, including rare Byzantine scenarios

### Existing Approaches Limitations

- **Path-based tracing**: Would clutter production code with path-specific logic
- **Line-by-line annotations**: Creates maintenance overhead and diagram clutter  
- **Manual test scenarios**: Cannot guarantee complete coverage of all execution paths
- **Traditional unit tests**: Cannot validate distributed consensus flow across multiple nodes

### Requirements

Based on [PRD-05 HotStuff Consensus](/workspace/workflow/prd/btc-federation/05_hotstuff_consensus.md):
- **Test Coverage**: >90% unit test coverage for HotStuff implementation
- **BFT Resilience**: System must handle 15+ specific Byzantine behaviors with automated detection
- **Path Validation**: 100% coverage of logic paths in [data flow diagram](/workspace/architecture/btc-federation/protocols/hot-stuff-consensus/data-flow.mermaid)
- **Production Safety**: Zero overhead and no attack surface in production builds

## Decision

We adopt an **Event-Based Consensus Testing Framework** with the following key components:

### 1. Interface-Based Event Tracing

```go
// Production-safe interface injection
type EventTracer interface {
    RecordEvent(nodeID uint16, eventType EventType, payload EventPayload)
    RecordTransition(nodeID uint16, from, to State, trigger string)
    RecordMessage(nodeID uint16, direction MessageDirection, msgType string, payload EventPayload)
}

// Production: No-op implementation (zero overhead)
type NoOpEventTracer struct{}

// Testing: Centralized event collection
type ConsensusEventTracer struct {
    events []ConsensusEvent
}
```

### 2. Build Tags for Byzantine Code

```go
//go:build testing
// +build testing

// Malicious behaviors only available in testing builds
func (cn *ConsensusNode) EnableByzantineBehavior(behavior string) {
    // Byzantine logic excluded from production
}
```

### 3. Rule-Based Event Validation

```go
type ValidationRule interface {
    Validate(events []ConsensusEvent) []ValidationError
    Description() string
}

type ValidationError struct {
    Rule        string
    Description string
    Events      []ConsensusEvent
    Severity    Severity
}

type Severity string
const (
    SeverityError   Severity = "error"   // Must not happen
    SeverityWarning Severity = "warning" // Unusual but allowed
    SeverityInfo    Severity = "info"    // Informational
)

// Ordering constraint validation
type MustHappenBeforeRule struct {
    EventA      EventType
    EventB      EventType
    Description string
}

func (r *MustHappenBeforeRule) Validate(events []ConsensusEvent) []ValidationError {
    errors := []ValidationError{}
    nodeEvents := groupEventsByNode(events)
    
    for nodeID, nodeEventList := range nodeEvents {
        eventAIndex := -1
        eventBIndex := -1
        
        for i, event := range nodeEventList {
            if event.EventType == r.EventA {
                eventAIndex = i
            }
            if event.EventType == r.EventB {
                eventBIndex = i
            }
        }
        
        if eventBIndex != -1 && (eventAIndex == -1 || eventAIndex > eventBIndex) {
            errors = append(errors, ValidationError{
                Rule:        r.Description,
                Description: fmt.Sprintf("Node %d: %s must happen before %s", 
                    nodeID, r.EventA, r.EventB),
                Severity:    SeverityError,
            })
        }
    }
    
    return errors
}

// Forbidden event validation
type MustNotHappenRule struct {
    EventType   EventType
    Condition   func(event ConsensusEvent) bool
    Description string
}

func (r *MustNotHappenRule) Validate(events []ConsensusEvent) []ValidationError {
    errors := []ValidationError{}
    
    for _, event := range events {
        if event.EventType == r.EventType && (r.Condition == nil || r.Condition(event)) {
            errors = append(errors, ValidationError{
                Rule:        r.Description,
                Description: fmt.Sprintf("Forbidden event occurred: %s on node %d", 
                    r.EventType, event.NodeID),
                Events:      []ConsensusEvent{event},
                Severity:    SeverityError,
            })
        }
    }
    
    return errors
}

// Quorum threshold validation
type QuorumRule struct {
    EventType     EventType
    MinNodes      uint16
    Description   string
}

func (r *QuorumRule) Validate(events []ConsensusEvent) []ValidationError {
    nodesWithEvent := make(map[uint16]bool)
    
    for _, event := range events {
        if event.EventType == r.EventType {
            nodesWithEvent[event.NodeID] = true
        }
    }
    
    if uint16(len(nodesWithEvent)) < r.MinNodes {
        return []ValidationError{{
            Rule:        r.Description,
            Description: fmt.Sprintf("Only %d nodes had %s, minimum %d required", 
                len(nodesWithEvent), r.EventType, r.MinNodes),
            Severity:    SeverityError,
        }}
    }
    
    return []ValidationError{}
}

// Node role-specific validation
type LeaderOnlyRule struct {
    EventType   EventType
    Description string
}

func (r *LeaderOnlyRule) Validate(events []ConsensusEvent) []ValidationError {
    errors := []ValidationError{}
    
    for _, event := range events {
        if event.EventType == r.EventType {
            view := getViewFromEvent(event)
            expectedLeader := calculateLeader(view, getTotalNodes())
            
            if event.NodeID != expectedLeader {
                errors = append(errors, ValidationError{
                    Rule:        r.Description,
                    Description: fmt.Sprintf("Event %s occurred on node %d but leader for view %d should be node %d", 
                        r.EventType, event.NodeID, view, expectedLeader),
                    Events:      []ConsensusEvent{event},
                    Severity:    SeverityError,
                })
            }
        }
    }
    
    return errors
}

// Validator-only validation
type ValidatorOnlyRule struct {
    EventType   EventType
    Description string
}

func (r *ValidatorOnlyRule) Validate(events []ConsensusEvent) []ValidationError {
    errors := []ValidationError{}
    
    for _, event := range events {
        if event.EventType == r.EventType {
            view := getViewFromEvent(event)
            expectedLeader := calculateLeader(view, getTotalNodes())
            
            if event.NodeID == expectedLeader {
                errors = append(errors, ValidationError{
                    Rule:        r.Description,
                    Description: fmt.Sprintf("Event %s occurred on leader node %d but should only occur on validators", 
                        r.EventType, event.NodeID),
                    Events:      []ConsensusEvent{event},
                    Severity:    SeverityError,
                })
            }
        }
    }
    
    return errors
}

// Causal dependency validation
type CausalDependencyRule struct {
    TriggerEvent   EventType
    RequiredEvent  EventType
    TimeWindow     time.Duration
    Description    string
}

func (r *CausalDependencyRule) Validate(events []ConsensusEvent) []ValidationError {
    errors := []ValidationError{}
    nodeEvents := groupEventsByNode(events)
    
    for nodeID, nodeEventList := range nodeEvents {
        for i, event := range nodeEventList {
            if event.EventType == r.TriggerEvent {
                found := false
                
                // Look for required event within time window before trigger
                for j := i - 1; j >= 0; j-- {
                    prevEvent := nodeEventList[j]
                    if prevEvent.EventType == r.RequiredEvent {
                        if event.Timestamp.Sub(prevEvent.Timestamp) <= r.TimeWindow {
                            found = true
                            break
                        }
                    }
                }
                
                if !found {
                    errors = append(errors, ValidationError{
                        Rule:        r.Description,
                        Description: fmt.Sprintf("Node %d: %s occurred without %s within %v", 
                            nodeID, r.TriggerEvent, r.RequiredEvent, r.TimeWindow),
                        Events:      []ConsensusEvent{event},
                        Severity:    SeverityError,
                    })
                }
            }
        }
    }
    
    return errors
}

// Cross-node coordination validation
type CrossNodeCoordinationRule struct {
    InitiatorEvent EventType
    ResponseEvent  EventType
    MinResponders  uint16
    MaxDelay       time.Duration
    Description    string
}

func (r *CrossNodeCoordinationRule) Validate(events []ConsensusEvent) []ValidationError {
    errors := []ValidationError{}
    
    // Find all initiator events
    initiatorEvents := []ConsensusEvent{}
    for _, event := range events {
        if event.EventType == r.InitiatorEvent {
            initiatorEvents = append(initiatorEvents, event)
        }
    }
    
    // For each initiator, count valid responses
    for _, initEvent := range initiatorEvents {
        validResponses := 0
        cutoffTime := initEvent.Timestamp.Add(r.MaxDelay)
        
        for _, event := range events {
            if event.EventType == r.ResponseEvent && 
               event.NodeID != initEvent.NodeID &&
               event.Timestamp.After(initEvent.Timestamp) &&
               event.Timestamp.Before(cutoffTime) {
                
                // Check if this response corresponds to the initiator event
                if r.isValidResponse(initEvent, event) {
                    validResponses++
                }
            }
        }
        
        if uint16(validResponses) < r.MinResponders {
            errors = append(errors, ValidationError{
                Rule:        r.Description,
                Description: fmt.Sprintf("Initiator node %d got only %d responses, minimum %d required", 
                    initEvent.NodeID, validResponses, r.MinResponders),
                Events:      []ConsensusEvent{initEvent},
                Severity:    SeverityError,
            })
        }
    }
    
    return errors
}

// State transition validation
type StateTransitionRule struct {
    FromState   string
    ToState     string
    ValidEvents []EventType
    Description string
}

func (r *StateTransitionRule) Validate(events []ConsensusEvent) []ValidationError {
    errors := []ValidationError{}
    nodeStates := make(map[uint16]string)
    
    for _, event := range events {
        currentState := nodeStates[event.NodeID]
        
        if event.EventType == "state_transition" {
            fromState := event.FromState
            toState := event.ToState
            
            if string(fromState) == r.FromState && string(toState) == r.ToState {
                // Check if transition was triggered by valid event
                if !r.isValidTriggerEvent(event.Trigger) {
                    errors = append(errors, ValidationError{
                        Rule:        r.Description,
                        Description: fmt.Sprintf("Node %d: Invalid state transition %s->%s triggered by %s", 
                            event.NodeID, fromState, toState, event.Trigger),
                        Events:      []ConsensusEvent{event},
                        Severity:    SeverityError,
                    })
                }
            }
            
            nodeStates[event.NodeID] = string(toState)
        }
    }
    
    return errors
}

// Helper functions for rule validation
func groupEventsByNode(events []ConsensusEvent) map[uint16][]ConsensusEvent {
    nodeEvents := make(map[uint16][]ConsensusEvent)
    for _, event := range events {
        nodeEvents[event.NodeID] = append(nodeEvents[event.NodeID], event)
    }
    return nodeEvents
}

func getViewFromEvent(event ConsensusEvent) uint64 {
    if view, ok := event.Payload["view"].(uint64); ok {
        return view
    }
    return 0
}

func calculateLeader(view uint64, totalNodes uint16) uint16 {
    return uint16(view % uint64(totalNodes))
}

func getTotalNodes() uint16 {
    // Return from consensus configuration
    return 5 // Example value
}
```

### 4. Centralized Event Collection

Since all nodes run in shared test runtime with mock infrastructure:
- Single `SharedPathCollector` instance injected into all test nodes
- Real-time event aggregation without distributed log correlation
- Immediate validation of cross-node event sequences

## Implementation Architecture

### Canonical Events Taxonomy (v1.1)

To eliminate ambiguity and enable strict rule validation, we introduce a phase-disambiguated, role-scoped event set. Existing generic events remain available as aliases for a transition period. The registry is updated to version 1.1.

Key principles:
- Phase prefixing: `prepare_*`, `precommit_*`, `commit_*` for phase-specific votes and QCs
- Role scoping: leader-only vs validator-only events
- Required payloads: standardized keys across events for deterministic validation

Proposal:
- `proposal_created` (leader): block proposal created
- `proposal_broadcasted` (leader): proposal broadcast
- `proposal_received` (validator): proposal delivered to validator
- `proposal_validated` (validator): structure/parent/justify/signature/safenode checks passed
- `proposal_rejected` (validator): validation failed, with reason

Prepare phase:
- `prepare_vote_created|sent` (validator)
- `prepare_vote_received|validated` (leader)
- `prepare_qc_formed|broadcasted` (leader)
- `prepare_qc_received|validated` (validator)

Pre-commit phase:
- `precommit_vote_created|sent` (validator)
- `precommit_vote_received|validated` (leader)
- `precommit_qc_formed|broadcasted` (leader)
- `precommit_qc_received|validated` (validator)

Commit phase:
- `commit_vote_created|sent` (validator)
- `commit_vote_received|validated` (leader)
- `commit_qc_formed|broadcasted` (leader)
- `commit_qc_received|validated` (validator)

Block & Safety:
- `block_added` (all): block inserted into block tree
- `locked_qc_updated` (all): locked QC updated from pre-commit QC
- `safenode_check_passed|failed` (validator)
- `block_committed` (all): block finalized (Simple HotStuff direct commit)

View/Timeout/Pacemaker:
- `view_timer_started`, `view_timeout_detected` (all)
- `timeout_message_sent|received|validated` (all)
- `view_change_started` (all)
- `new_view_message_sent|received|validated` (leader/all)
- `new_view_started`, `leader_elected` (all)

Recovery/Storage/Sync:
- `storage_tx_begin|storage_tx_commit|storage_tx_rollback` (all) with phase context
- `state_sync_requested|state_sync_completed` (lagging nodes)

Byzantine/Exclusion:
- `equivocation_detected`, `byzantine_detected` (honest nodes)
- `evidence_collected|evidence_broadcasted|evidence_stored` (honest nodes)
- `validator_excluded`

Partition/Mode:
- `partition_mode_entered|partition_mode_exited` (all)

Auxiliary:
- `highest_qc_updated`, `view_synchronized` (all)

Required payloads (selected):
- Common: `view`
- Block: `block_hash`, `parent_hash?`, `height`, `leader?`
- Vote/QC: `phase`, `voter` (vote_*), `vote_count`, `required` (qc_*), `from_leader?`
- View/timeout: `timeout_ms`, `sender`, `has_high_qc`, `timeout_cert_count`
- Safety: `locked_view`, `justify_view`, `safenode`
- Byzantine: `offender`, `evidence_id`

Compatibility and Aliases:
- Generic events remain as aliases during migration (e.g., `vote_created` ≈ `prepare_vote_created` in happy path)
- Tests and diagram should transition to canonical names; alias usage is discouraged in new code

### Consensus Node Integration

```go
type ConsensusNode struct {
    eventTracer interfaces.EventTracer  // nil-safe interface
    nodeID      uint16
}

func (cn *ConsensusNode) HandleProposal(proposal *ProposalMsg) error {
    // Record events throughout consensus logic
    if cn.eventTracer != nil {
        cn.eventTracer.RecordEvent(cn.nodeID, interfaces.EventProposalReceived, 
            interfaces.EventPayload{
                "view":       proposal.View,
                "block_hash": proposal.Block.Hash(),
            })
    }
    // ... consensus logic
}
```

### Build Configuration

**Production Build**: `go build ./...` (excludes malicious code, zero testing overhead)

**Testing Build**: `go build -tags testing ./...` (includes Byzantine behaviors)

**Test Execution**: `go test -tags testing ./pkg/consensus/...`

### Diagram & Test Anchoring (v1.1)

- The HotStuff data flow diagram is to be annotated with “EMITS:” markers using the canonical event names above, especially for the proposal path and each phase’s vote/QC lifecycle.
- The happy-path and flow-by-flow tests must assert phase-specific events (e.g., `prepare_vote_*`, `prepare_qc_*`, then `precommit_*`, `commit_*`), role-scoped occurrences, and ordering constraints.

### Validation Rule Sets

**Happy Path Rules**:
- Proposal must be received before vote sent
- Quorum threshold must be met (≥2f+1 votes)
- No double voting in same view/phase

**Byzantine Detection Rules**:
- Byzantine behavior must be detected when present
- Evidence must be stored for malicious actions
- View change should follow Byzantine detection

**Node-Specific Rules**:
- Only leader should create proposals
- Only validators should send votes to leader
- Byzantine nodes should not detect own misbehavior

### Complete Rule Set Examples

**Happy Path Consensus Rules**:
```go
func GetHappyPathRules(nodeCount, faultyNodes uint16) []ValidationRule {
    quorumSize := 2*faultyNodes + 1
    
    return []ValidationRule{
        &MustHappenBeforeRule{
            EventA:      "proposal_received",
            EventB:      "vote_sent", 
            Description: "Proposal must be received before voting",
        },
        &MustHappenBeforeRule{
            EventA:      "proposal_validated",
            EventB:      "vote_created",
            Description: "Proposal must be validated before creating vote",
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
        &ValidatorOnlyRule{
            EventType:   "vote_sent",
            Description: "Only validators should send votes",
        },
        &MustNotHappenRule{
            EventType:   "vote_sent",
            Condition:   checkDoubleVoting,
            Description: "No double voting in same view/phase",
        },
        &CausalDependencyRule{
            TriggerEvent:  "qc_formed",
            RequiredEvent: "vote_received", 
            TimeWindow:    5 * time.Second,
            Description:   "QC formation requires sufficient votes",
        },
    }
}
```

**Byzantine Detection Rules**:
```go
func GetByzantineDetectionRules() []ValidationRule {
    return []ValidationRule{
        &CrossNodeCoordinationRule{
            InitiatorEvent: "byzantine_behavior_occurred",
            ResponseEvent:  "byzantine_detected",
            MinResponders:  2, // At least 2 honest nodes must detect
            MaxDelay:       10 * time.Second,
            Description:    "Byzantine behavior must be detected by honest nodes",
        },
        &MustHappenBeforeRule{
            EventA:      "byzantine_detected",
            EventB:      "evidence_stored",
            Description: "Evidence must be stored after detection",
        },
        &MustHappenBeforeRule{
            EventA:      "byzantine_detected", 
            EventB:      "view_change",
            Description: "View change should follow byzantine detection",
        },
        &MustNotHappenRule{
            EventType: "byzantine_detected",
            Condition: func(event ConsensusEvent) bool {
                return isByzantineNode(event.NodeID)
            },
            Description: "Byzantine node should not detect own misbehavior",
        },
    }
}
```

**Network Partition Rules**:
```go
func GetPartitionRecoveryRules() []ValidationRule {
    return []ValidationRule{
        &StateTransitionRule{
            FromState:   "normal_operation",
            ToState:     "partition_mode",
            ValidEvents: []EventType{"partition_detected"},
            Description: "Partition mode only on partition detection",
        },
        &MustNotHappenRule{
            EventType: "vote_sent",
            Condition: func(event ConsensusEvent) bool {
                return isInPartitionMode(event.NodeID)
            },
            Description: "No voting during partition mode",
        },
        &CausalDependencyRule{
            TriggerEvent:  "consensus_resumed",
            RequiredEvent: "state_sync_completed",
            TimeWindow:    30 * time.Second, 
            Description:   "State sync required before resuming consensus",
        },
    }
}
```

### Rule Algorithm Complexity Analysis

**Time Complexity**:
- `MustHappenBeforeRule`: O(n*m) where n=nodes, m=events per node
- `QuorumRule`: O(e) where e=total events
- `LeaderOnlyRule`: O(e) for event scanning + O(1) for leader calculation
- `CrossNodeCoordinationRule`: O(e²) for matching initiator/response pairs
- `CausalDependencyRule`: O(n*m²) for backward time window scanning

**Space Complexity**: 
- `groupEventsByNode`: O(e) additional memory
- Rule state tracking: O(n) for node-specific state
- Validation errors: O(r) where r=rule violations

**Optimization Strategies**:
- Event indexing by type for faster rule filtering
- Time-based event partitioning for causal dependency rules  
- Parallel rule execution for independent validation rules

## Consequences

### Positive

- **Clean Production Code**: Interface injection pattern maintains existing code style
- **Zero Production Overhead**: No-op implementation has no performance impact
- **Complete Path Coverage**: Event-based validation ensures all logic paths tested
- **Byzantine Behavior Testing**: Build tags enable comprehensive attack vector testing
- **Centralized Validation**: Shared runtime allows real-time cross-node event validation
- **Rule-Based Flexibility**: Can define complex validation rules for different scenarios
- **Maintainable**: Adding new events doesn't break existing validation logic
- **Debugging Support**: Complete event timeline available for failure analysis

### Negative

- **Test Setup Complexity**: Requires careful event tracer injection in test environment
- **Build Tag Management**: Must maintain separate build configurations
- **Rule Definition Overhead**: Complex scenarios require detailed validation rules
- **Mock Infrastructure Dependency**: Relies on centralized mock environment

### Security Considerations

- **Production Safety**: Byzantine code completely excluded from production builds
- **Attack Surface**: No additional attack vectors introduced in production
- **Code Separation**: Clear delineation between test and production functionality

## Implementation Requirements

### Integration Points

- **Mock Environment**: Extend existing `/pkg/consensus/mocks/` with event tracing
- **Node Factory**: Inject event tracers during test node creation
- **CI/CD Pipeline**: Separate builds for production and testing validation

### Performance Targets

- **Production Impact**: Zero performance overhead (no-op interface)
- **Test Performance**: Event collection should not significantly impact test execution time
- **Memory Usage**: Efficient event storage for long-running consensus tests

### Testing Coverage Goals

- **Logic Path Coverage**: 100% of paths in consensus data flow diagram
- **Byzantine Scenarios**: All 15+ attack vectors from PRD requirements
- **Event Validation**: Comprehensive rule-based validation for all event sequences
- **Cross-Node Coordination**: Validate proper distributed consensus behavior

## Related Decisions

- **ADR-005**: HotStuff Consensus Protocol - Defines consensus mechanism being tested
- **ADR-006**: Consensus Node Identification - NodeID used in event tracking
- **[PRD-05](/workspace/workflow/prd/btc-federation/05_hotstuff_consensus.md)**: HotStuff Consensus - Testing requirements and BFT validation criteria

## References

- [HotStuff Data Flow Diagram](/workspace/architecture/btc-federation/protocols/hot-stuff-consensus/data-flow.mermaid)
- [Go Build Constraints Documentation](https://golang.org/pkg/go/build/#hdr-Build_Constraints)
- [Policy: Testing Strategy](/workspace/policy.md#testing-strategy-and-documentation)
