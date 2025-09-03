# Task 05-04: View Change Flow Integration Testing

## Document Information
- **Project**: BTC Federation HotStuff Consensus
- **Task**: View Change Flow Integration Testing
- **Priority**: HIGH - Failure recovery validation
- **Status**: Planning Complete, Implementation Required
- **Dependencies**: Task 05-03 (Happy Path Consensus Flow Integration)
- **Estimated Duration**: 2-3 cycles

## Overview

This task implements comprehensive integration testing for the HotStuff consensus protocol's view change mechanism. The testing approach focuses on **verifying the actual protocol implementation** rather than creating test-specific protocol logic. Tests will trigger real protocol behaviors through fault injection and validate that the implementation correctly follows the data flow diagram (lines 506-559).

### Key Principle
**Tests verify the implementation matches the specification, not reimplement the specification.**

## Flow Under Test

Based on the HotStuff data flow diagram (lines 506-559):
- **Timeout Detection** → **Timeout Message Broadcast** → **NewView Collection** → **Leader Election** → **Recovery**

The protocol's built-in view change mechanism includes:
- `onViewTimeout()` - Handles timeout detection
- `broadcastTimeoutMessage()` - Broadcasts timeout to validators  
- `processTimeoutMessage()` - Processes received timeouts
- `ProcessNewViewMessage()` - Handles NewView message processing
- Built-in leader election via round-robin (view % n)

## Test File Structure

### Main Test Files
- `tests/consensus/view_change_test.go` - Core view change integration tests
- `tests/consensus/view_change_byzantine_test.go` - Byzantine scenarios (build tag: testing)
- `tests/consensus/view_change_performance_test.go` - Performance benchmarks

### Support Infrastructure
- `pkg/consensus/testing/view_change_helpers.go` - Test utilities and fault injection
- `pkg/consensus/testing/view_change_validation.go` - Event validation helpers

## Comprehensive Test Scenarios

**Test Priority Structure**:
- **Priority 1**: Critical branch coverage tests (Section 10) - Required for 100% coverage
- **Priority 2**: Core functionality tests (Sections 1-5) - Essential protocol validation
- **Priority 3**: Advanced scenarios (Sections 6-8) - Edge cases and Byzantine behaviors
- **Priority 4**: Performance benchmarks (Section 9) - Non-blocking performance validation

### 1. Basic View Change Tests (Priority 2)

#### 1.1 `TestViewTimeoutAndRecovery`
**Objective**: Test basic timeout detection and view change recovery

**Test Flow**:
```go
func TestViewTimeoutAndRecovery(t *testing.T) {
    // 1. Create 5-node network with event tracing
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // 2. Start normal consensus
    leader := calculateLeader(0, 5)
    nodes[leader].ProposeBlock([]byte("test_block"))
    
    // 3. Block leader before QC formation to trigger timeout
    network.BlockNode(leader)
    
    // 4. Wait for timeout (should be detected within configured duration)
    time.Sleep(viewTimeout + 500*time.Millisecond)
    
    // 5. Validate timeout events were emitted by all nodes
    events := tracer.GetEvents()
    assert.Contains(events, "view_timeout_detected")
    assert.Contains(events, "timeout_message_sent")
    
    // 6. Verify new leader was selected via protocol
    newLeader := calculateLeader(1, 5) 
    assert.Equal(getCurrentLeader(), newLeader)
    
    // 7. CRITICAL: Verify leader_elected occurs BEFORE new_view_started (data flow line 558)
    leaderElectedEvents := filterEventsByType(events, "leader_elected")
    newViewStartedEvents := filterEventsByType(events, "new_view_started")
    
    assert.Greater(len(leaderElectedEvents), 0, "leader_elected event must be emitted")
    assert.Greater(len(newViewStartedEvents), 0, "new_view_started event must be emitted")
    
    // Verify temporal ordering
    leaderTime := leaderElectedEvents[len(leaderElectedEvents)-1].Timestamp
    newViewTime := newViewStartedEvents[len(newViewStartedEvents)-1].Timestamp
    assert.True(leaderTime.Before(newViewTime) || leaderTime.Equal(newViewTime),
        "leader_elected must occur before or with new_view_started")
    
    // 8. Verify consensus can proceed in new view
    network.UnblockNode(leader) // Unblock for cleanup
    err = nodes[newLeader].ProposeBlock([]byte("recovery_block"))
    assert.NoError(err)
}
```

**Events to Validate**:
- `view_timer_started` - Timer initialization
- `view_timeout_detected` - Timeout occurred
- `timeout_message_sent` - Each node broadcasts timeout
- `timeout_message_received` - Nodes collect timeouts
- `timeout_message_validated` - Signature validation
- `view_change_started` - View transition begins
- `new_view_message_sent` - NewView messages sent
- `leader_elected` - New leader selected
- `new_view_started` - New view active

#### 1.2 `TestFastPathViewChange`
**Objective**: Test view change with recent QC available (no sync needed)

**Test Approach**:
- Complete one consensus round successfully
- Trigger timeout in next view
- Verify fast path using recent QC
- Validate no state sync required

#### 1.3 `TestSlowPathViewChange`
**Objective**: Test view change requiring synchronization

**Test Approach**:
- Create nodes with different view states
- Trigger view change
- Verify ViewSync request/response protocol
- Validate convergence to common view

### 2. Leader Failure Scenarios (Priority 2)

#### 2.1 `TestLeaderFailureBeforeProposal`
**Test Flow**:
- Start consensus round
- Kill leader before it sends proposal
- Verify validators timeout after expected duration
- Validate clean transition to next leader
- Confirm new leader can propose

#### 2.2 `TestLeaderFailureAfterProposal`
**Test Flow**:
- Leader sends proposal successfully
- Kill leader before PrepareQC formation
- Verify validators timeout waiting for QC
- Check new leader re-proposes block

#### 2.3 `TestLeaderFailureDuringVoteCollection`
**Test Flow**:
- Leader receives partial votes (not quorum)
- Kill leader before QC formation
- Verify validators timeout appropriately
- Validate new leader starts fresh (ignores partial votes)

### 3. Cascading View Changes (Priority 2)

#### 3.1 `TestConsecutiveViewChanges`
**Objective**: Test multiple consecutive leader failures

**Test Flow**:
```go
func TestConsecutiveViewChanges(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    failedViews := 3
    for view := 0; view < failedViews; view++ {
        leader := calculateLeader(view, 5)
        
        // Start consensus for this view
        nodes[leader].ProposeBlock([]byte(fmt.Sprintf("block_%d", view)))
        
        // Kill current leader to force view change
        network.BlockNode(leader)
        
        // Wait for timeout and view change
        time.Sleep(getViewTimeout(view) + 500*time.Millisecond)
        
        // Validate view advanced
        events := tracer.GetEvents()
        assert.Contains(events, fmt.Sprintf("view_change_to_%d", view+1))
        
        // Verify exponential backoff in timeout
        if view > 0 {
            timeout1 := getLastTimeout(events, view-1)
            timeout2 := getLastTimeout(events, view)
            assert.Greater(timeout2, timeout1*1.4) // Backoff factor
        }
    }
    
    // Verify final leader can succeed
    finalLeader := calculateLeader(failedViews, 5)
    // Unblock all nodes for final test
    for i := 0; i < failedViews; i++ {
        network.UnblockNode(calculateLeader(i, 5))
    }
    
    err = nodes[finalLeader].ProposeBlock([]byte("success_block"))
    assert.NoError(err)
}
```

#### 3.2 `TestRapidViewChanges`
**Objective**: Test rapid succession of view changes

**Test Approach**:
- Trigger view changes in quick succession
- Verify protocol handles rapid transitions
- Check no state corruption or race conditions
- Validate proper cleanup between views

### 4. Network Partition Tests (Priority 2)

#### 4.1 `TestPartitionDuringViewChange`
**Test Flow**:
```go
func TestPartitionDuringViewChange(t *testing.T) {
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Start normal consensus
    leader := calculateLeader(0, 5)
    nodes[leader].ProposeBlock([]byte("pre_partition_block"))
    time.Sleep(100 * time.Millisecond) // Let it start
    
    // Create partition: {0,1} vs {2,3,4}
    partition1 := []types.NodeID{0, 1}
    partition2 := []types.NodeID{2, 3, 4}
    network.CreatePartition(partition1, partition2)
    
    // Wait for timeout in majority partition
    time.Sleep(viewTimeout + 500*time.Millisecond)
    
    // Verify only majority can progress
    events := tracer.GetEvents()
    majorityEvents := filterEventsByNodes(events, partition2)
    minorityEvents := filterEventsByNodes(events, partition1)
    
    assert.Contains(majorityEvents, "view_change_started")
    assert.NotContains(minorityEvents, "view_change_started") // Minority stuck
    
    // Heal partition
    network.HealPartition()
    time.Sleep(1 * time.Second)
    
    // Verify minority catches up
    allEvents := tracer.GetEvents()
    assert.Contains(allEvents, "state_sync_requested")
    assert.Contains(allEvents, "state_sync_completed")
}
```

#### 4.2 `TestPartitionedLeader`
**Test Approach**:
- Isolate leader from all validators
- Verify validators detect timeout and elect new leader
- Check majority partition can continue
- Validate isolated leader cannot disrupt

#### 4.3 `TestSplitBrainRecovery`
**Test Approach**:
- Create 2 equal partitions
- Let each advance to different views
- Merge partitions
- Verify convergence to higher view

### 5. Concurrent Operations (Priority 2)

#### 5.1 `TestSimultaneousTimeouts`
**Test Flow**:
```go
func TestSimultaneousTimeouts(t *testing.T) {
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Block all message delivery to trigger simultaneous timeouts
    network.BlockAllMessages()
    
    // Wait for all nodes to timeout simultaneously
    time.Sleep(viewTimeout + 200*time.Millisecond)
    
    // Re-enable network
    network.UnblockAllMessages()
    
    events := tracer.GetEvents()
    
    // Verify all nodes detected timeout
    timeoutEvents := filterEventsByType(events, "view_timeout_detected")
    assert.Equal(len(timeoutEvents), 5) // All nodes timeout
    
    // Verify single view advancement (not multiple)
    viewChangeEvents := filterEventsByType(events, "view_change_started")
    assert.True(allEventsHaveSameView(viewChangeEvents)) // All advance to same view
}
```

#### 5.2 `TestTimeoutDuringDifferentPhases`
**Test Cases**:
- Timeout during Prepare phase (after proposal, before PrepareQC)
- Timeout during PreCommit phase (after PrepareQC, before PreCommitQC)
- Timeout during Commit phase (after PreCommitQC, before CommitQC)

#### 5.3 `TestViewChangeWithPendingVotes`
**Test Approach**:
- Send partial votes for current view
- Trigger view change
- Verify old votes are ignored in new view
- Validate clean start for new view

### 6. State Synchronization (Priority 3)

#### 6.1 `TestLaggingNodeCatchUp`
**Test Flow**:
```go
func TestLaggingNodeCatchUp(t *testing.T) {
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Disconnect node 4
    network.BlockNode(4)
    
    // Run several consensus rounds with 4 nodes
    for round := 0; round < 3; round++ {
        leader := calculateLeader(round, 4) // Only 4 active nodes
        if leader == 4 { continue } // Skip if disconnected node would be leader
        
        nodes[leader].ProposeBlock([]byte(fmt.Sprintf("block_%d", round)))
        time.Sleep(500 * time.Millisecond) // Let consensus complete
    }
    
    // Reconnect node 4
    network.UnblockNode(4)
    
    // Node 4 should detect it's behind and request sync
    time.Sleep(1 * time.Second)
    
    events := tracer.GetEvents()
    node4Events := filterEventsByNode(events, 4)
    
    assert.Contains(node4Events, "state_sync_requested")
    assert.Contains(node4Events, "state_sync_completed")
    
    // Verify node 4 is now at same view as others
    for nodeID, node := range nodes {
        if nodeID == 4 { continue }
        assert.Equal(nodes[4].GetCurrentView(), node.GetCurrentView())
    }
}
```

#### 6.2 `TestConflictingHighestQCs`
**Test Approach**:
- Create scenario where nodes have different highest QCs
- Trigger view change requiring QC selection
- Verify deterministic tie-breaking rules
- Validate all nodes converge to same QC

#### 6.3 `TestViewSyncValidation`
**Test Cases**:
- Valid sync with merkle proofs
- Invalid sync data rejection
- Corrupted sync response handling
- Byzantine sync response detection

### 7. Byzantine Behavior Tests (Priority 3, build tag: testing)

#### 7.1 `TestByzantineLeaderDuringViewChange`
**Test Flow**:
```go
//go:build testing

func TestByzantineLeaderDuringViewChange(t *testing.T) {
    nodes, network, err := integration.CreateTestNetwork(5, tracer, true) // Enable byzantine
    
    leader := calculateLeader(0, 5)
    
    // Enable byzantine behavior: conflicting NewView messages
    nodes[leader].EnableByzantineBehavior("conflicting_new_views")
    
    // Trigger view change
    network.BlockAllMessages() // Force timeout
    time.Sleep(viewTimeout + 500*time.Millisecond)
    network.UnblockAllMessages()
    
    events := tracer.GetEvents()
    
    // Verify byzantine behavior detected
    assert.Contains(events, "byzantine_detected")
    assert.Contains(events, "evidence_collected")
    assert.Contains(events, "evidence_broadcasted")
    
    // Verify honest nodes advance to next view with new leader
    assert.Contains(events, "view_change_started")
    newLeader := calculateLeader(1, 5)
    assert.NotEqual(leader, newLeader)
}
```

#### 7.2 `TestByzantineTimeoutSpam`
**Test Approach**:
- Byzantine node floods timeout messages
- Verify rate limiting and spam detection
- Check protocol continues correctly
- Validate honest nodes ignore spam

#### 7.3 `TestInvalidTimeoutSignatures`
**Test Approach**:
- Byzantine node sends timeouts with invalid signatures
- Verify signature validation rejects them
- Check invalid timeouts don't contribute to quorum
- Validate protocol safety maintained

#### 7.4 `TestEquivocationDuringViewChange`
**Test Approach**:
- Byzantine node sends different votes for same view
- Verify equivocation detection during view change
- Validate evidence collection and propagation
- Check honest nodes exclude equivocating node

### 8. Edge Cases and Corner Scenarios (Priority 3)

#### 8.1 `TestViewNumberOverflow`
**Test Approach**:
- Simulate consensus near maximum view number
- Test view number wraparound behavior
- Verify no protocol violations
- Check proper view arithmetic

#### 8.2 `TestTimeoutCertificateValidation`
**Test Cases**:
- Insufficient timeout certificates (< 2f+1)
- Invalid signatures in timeout certificates
- Timeout certificates from wrong view
- Mixed valid/invalid certificates

#### 8.3 `TestRaceConditions`
**Test Cases**:
- Timeout occurs just as proposal arrives
- NewView message arrives during timeout processing
- Vote arrives during view transition
- Multiple simultaneous view change triggers

#### 8.4 `TestMemoryPressureDuringViewChange`
**Test Approach**:
- Simulate high memory usage during view change
- Verify graceful degradation
- Check recovery after memory pressure relieved
- Validate no memory leaks

### 9. Performance Benchmarks (Priority 4)

#### 9.1 `BenchmarkViewChangeLatency`
**Metrics**:
- Time from timeout detection to new view start
- Latency across different network sizes (5, 10, 20 nodes)
- Memory allocation during view change
- CPU usage profiling

#### 9.2 `BenchmarkTimeoutDetection`
**Metrics**:
- Timeout detection accuracy (±100ms target)
- Timer precision under load
- False positive rate (premature timeouts)
- Resource usage of timer management

#### 9.3 `BenchmarkNewViewCollection`
**Metrics**:
- NewView message aggregation time
- Processing overhead per message
- Memory usage during collection
- Network bandwidth utilization

## 10. Branch Coverage Completeness Tests

This section contains tests specifically designed to achieve **100% branch coverage** of the view change protocol as specified in the data flow diagram. These tests target specific protocol branches that may not be covered by the general scenarios above.

### **Priority 1: Critical for 100% Coverage**

#### 10.1 `TestNewViewHighestQCTieBreakSameView`
**Objective**: Test NewView QC selection when multiple QCs have same view but different blocks (data flow lines 192-194)

**Test Flow**:
```go
func TestNewViewHighestQCTieBreakSameView(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Create scenario where validators have same-view QCs for different blocks
    // 1. Create two different blocks at same view
    block1 := createBlock(view=1, hash="0x1111")
    block2 := createBlock(view=1, hash="0x2222") 
    
    // 2. Give some validators QC for block1, others QC for block2
    nodes[1].SetHighestQC(createQC(block1)) // QC for block1
    nodes[2].SetHighestQC(createQC(block1))
    nodes[3].SetHighestQC(createQC(block2)) // QC for block2
    nodes[4].SetHighestQC(createQC(block2))
    
    // 3. Trigger view change to view 2
    triggerViewChange(nodes, view=2)
    
    // 4. Validate deterministic tie-breaking occurred
    events := tracer.GetEvents()
    
    // Must use deterministic rule: prefer higher block hash
    expectedWinner := block2 // 0x2222 > 0x1111
    assert.Contains(events, "highest_qc_updated")
    
    qcUpdateEvent := findEvent(events, "highest_qc_updated")
    assert.Equal(expectedWinner.Hash, qcUpdateEvent.Payload["selected_block_hash"])
    assert.Contains(qcUpdateEvent.Payload["reason"], "tie_break_higher_hash")
}
```

**Data Flow Reference**: Lines 192-194 (deterministic tie-breaking logic)

#### 10.2 `TestViewTimeoutBackoffCap`  
**Objective**: Validate timeout backoff formula and maximum cap (data flow line 513)

**Test Flow**:
```go
func TestViewTimeoutBackoffCap(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    baseTimeout := 1 * time.Second
    maxTimeout := 60 * time.Second
    
    // Test timeout progression: min(baseTimeout * 2^min(view, 10), maxTimeout)
    expectedTimeouts := []time.Duration{
        1 * time.Second,  // view 0: 1 * 2^0 = 1s
        2 * time.Second,  // view 1: 1 * 2^1 = 2s  
        4 * time.Second,  // view 2: 1 * 2^2 = 4s
        8 * time.Second,  // view 3: 1 * 2^3 = 8s
        16 * time.Second, // view 4: 1 * 2^4 = 16s
        32 * time.Second, // view 5: 1 * 2^5 = 32s
        60 * time.Second, // view 6: min(64s, 60s) = 60s (capped)
        60 * time.Second, // view 11: min(1*2^10, 60s) = 60s (capped)
    }
    
    for view, expectedTimeout := range expectedTimeouts {
        triggerViewTimeout(nodes, view)
        
        events := tracer.GetEvents()
        timerEvents := filterEventsByType(events, "view_timer_started")
        latestTimer := timerEvents[len(timerEvents)-1]
        
        actualTimeout := time.Duration(latestTimer.Payload["timeout_ms"].(int64)) * time.Millisecond
        assert.Equal(expectedTimeout, actualTimeout, 
            "View %d: expected %v, got %v", view, expectedTimeout, actualTimeout)
        
        if expectedTimeout >= maxTimeout {
            assert.Contains(latestTimer.Payload, "capped_at_max")
        }
    }
}
```

**Data Flow Reference**: Line 513 (timeout formula with cap)

#### 10.3 `TestInvalidNewViewSignaturesAndView`
**Objective**: Verify invalid NewView messages don't contribute to quorum

**Test Flow**:
```go
func TestInvalidNewViewSignaturesAndView(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    leader := calculateLeader(1, 5)
    
    // Send mix of valid and invalid NewView messages to leader
    validNewViews := 2   // Not enough for quorum (need 3)
    invalidNewViews := 2
    
    // Send valid NewView messages
    for i := 0; i < validNewViews; i++ {
        validMsg := createValidNewView(view=1, sender=nodes[i])
        nodes[leader].ProcessNewViewMessage(validMsg)
    }
    
    // Send invalid NewView messages (wrong signatures, wrong view)
    invalidMsg1 := createNewViewWithInvalidSignature(view=1, sender=nodes[3])
    invalidMsg2 := createNewViewWithWrongView(view=0, sender=nodes[4])
    
    nodes[leader].ProcessNewViewMessage(invalidMsg1)
    nodes[leader].ProcessNewViewMessage(invalidMsg2)
    
    // Wait to ensure no proposal is created (insufficient valid NewViews)
    time.Sleep(500 * time.Millisecond)
    
    events := tracer.GetEvents()
    
    // Verify invalid messages were rejected
    assert.Contains(events, "new_view_message_rejected")
    rejectionEvents := filterEventsByType(events, "new_view_message_rejected")
    assert.Equal(2, len(rejectionEvents)) // 2 invalid messages rejected
    
    // Verify no proposal was created (insufficient quorum)
    proposalEvents := filterEventsByType(events, "proposal_created")
    assert.Equal(0, len(proposalEvents))
    
    // Verify leader is still waiting for more NewViews
    leaderState := nodes[leader].GetState()
    assert.Equal("waiting_for_new_views", leaderState.Phase)
}
```

#### 10.4 `TestTimeoutMessageDeduplication`
**Objective**: Ensure duplicate timeout messages don't inflate quorum count

**Test Flow**:
```go
func TestTimeoutMessageDeduplication(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Node 1 sends duplicate timeout messages
    timeoutMsg := createTimeoutMessage(view=0, sender=nodes[1])
    
    // Send same timeout message 3 times
    for i := 0; i < 3; i++ {
        for _, node := range nodes {
            node.ProcessTimeoutMessage(timeoutMsg)
        }
    }
    
    // Send valid timeout from node 2 (should reach quorum: node1 + node2 = 2, need 2f+1=3)
    timeoutMsg2 := createTimeoutMessage(view=0, sender=nodes[2])
    for _, node := range nodes {
        node.ProcessTimeoutMessage(timeoutMsg2)
    }
    
    // Send valid timeout from node 3 (should complete quorum)
    timeoutMsg3 := createTimeoutMessage(view=0, sender=nodes[3])
    for _, node := range nodes {
        node.ProcessTimeoutMessage(timeoutMsg3)
    }
    
    events := tracer.GetEvents()
    
    // Verify deduplication occurred
    dedupEvents := filterEventsByType(events, "timeout_message_deduplicated")
    assert.Greater(len(dedupEvents), 0) // Duplicates were detected
    
    // Verify quorum was reached with exactly 3 unique validators
    quorumEvents := filterEventsByType(events, "timeout_quorum_reached")
    assert.Equal(1, len(quorumEvents))
    
    quorumEvent := quorumEvents[0]
    assert.Equal(3, quorumEvent.Payload["unique_timeout_count"])
    assert.Equal([]uint16{1, 2, 3}, quorumEvent.Payload["timeout_validators"])
}
```

#### 10.5 `TestViewSyncUpdatesLockedQC`
**Objective**: Test lockedQC update during slow path view sync (data flow line 534)

**Test Flow**:
```go
func TestViewSyncUpdatesLockedQC(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Setup: Node 0 has old lockedQC (view 1), other nodes have higher QC (view 3)
    oldQC := createQC(view=1, block="0x1111")
    newQC := createQC(view=3, block="0x3333")
    
    nodes[0].SetLockedQC(oldQC)
    for i := 1; i < 5; i++ {
        nodes[i].SetLockedQC(newQC)
    }
    
    // Disconnect node 0 temporarily to create sync scenario
    network.BlockNode(0)
    
    // Progress consensus to view 4 with other nodes
    progressToView(nodes[1:], view=4)
    
    // Reconnect node 0 - should trigger slow path sync
    network.UnblockNode(0)
    
    // Wait for sync completion
    time.Sleep(2 * time.Second)
    
    events := tracer.GetEvents()
    node0Events := filterEventsByNode(events, 0)
    
    // Verify sync occurred
    assert.Contains(node0Events, "state_sync_requested")
    assert.Contains(node0Events, "state_sync_completed")
    
    // Verify lockedQC was updated to higher QC during sync
    lockedQCEvents := filterEventsByType(node0Events, "locked_qc_updated")
    assert.Greater(len(lockedQCEvents), 0)
    
    updateEvent := lockedQCEvents[len(lockedQCEvents)-1]
    assert.Equal(uint64(3), updateEvent.Payload["new_locked_view"])
    assert.Equal("0x3333", updateEvent.Payload["new_locked_block"])
    assert.Equal("view_sync", updateEvent.Payload["update_reason"])
    
    // Verify final state
    finalLockedQC := nodes[0].GetLockedQC()
    assert.Equal(uint64(3), finalLockedQC.View)
}
```

**Data Flow Reference**: Line 534 (lockedQC update during sync)

### **Priority 2: Important for Completeness**

#### 10.6 `TestNewViewMisdirectedIgnored`
**Objective**: NewView messages sent to wrong leader are ignored

**Test Flow**:
```go
func TestNewViewMisdirectedIgnored(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    correctLeader := calculateLeader(1, 5) // e.g., node 1
    wrongLeader := (correctLeader + 1) % 5   // e.g., node 2
    
    // Validators mistakenly send NewView to wrong leader
    misdirectedCount := 3
    for i := 0; i < misdirectedCount; i++ {
        newViewMsg := createNewView(view=1, sender=nodes[i+2])
        nodes[wrongLeader].ProcessNewViewMessage(newViewMsg)
    }
    
    // Send correct NewViews to right leader
    correctCount := 3 // Quorum
    for i := 0; i < correctCount; i++ {
        newViewMsg := createNewView(view=1, sender=nodes[i+2])
        nodes[correctLeader].ProcessNewViewMessage(newViewMsg)
    }
    
    events := tracer.GetEvents()
    
    // Verify wrong leader ignored misdirected messages
    wrongLeaderEvents := filterEventsByNode(events, wrongLeader)
    ignoredEvents := filterEventsByType(wrongLeaderEvents, "new_view_message_ignored")
    assert.Equal(misdirectedCount, len(ignoredEvents))
    
    for _, event := range ignoredEvents {
        assert.Equal("not_leader_for_view", event.Payload["ignore_reason"])
    }
    
    // Verify correct leader processed messages and created proposal
    correctLeaderEvents := filterEventsByNode(events, correctLeader)
    assert.Contains(correctLeaderEvents, "new_view_quorum_reached")
    assert.Contains(correctLeaderEvents, "proposal_created")
}
```

#### 10.7 `TestPhaseTimerResetOnNewView`
**Objective**: Ensure phase timers reset properly on view change

**Test Flow**:
```go
func TestPhaseTimerResetOnNewView(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Start consensus in view 0
    leader0 := calculateLeader(0, 5)
    nodes[leader0].ProposeBlock([]byte("block_0"))
    
    // Let consensus progress partially, then trigger timeout
    time.Sleep(500 * time.Millisecond) // Partial progress
    network.BlockAllMessages() // Force timeout
    
    // Wait for timeout and view change
    time.Sleep(viewTimeout + 500*time.Millisecond)
    
    network.UnblockAllMessages()
    
    events := tracer.GetEvents()
    
    // Find timer events for both views
    timerEvents := filterEventsByType(events, "view_timer_started")
    assert.Greater(len(timerEvents), 1) // At least 2 timers (view 0 and view 1)
    
    // Verify old timer was cancelled
    cancelEvents := filterEventsByType(events, "view_timer_cancelled")
    assert.Greater(len(cancelEvents), 0)
    
    // Verify new timer started with fresh timeout
    view1TimerEvents := filterTimerEventsByView(timerEvents, 1)
    assert.Equal(1, len(view1TimerEvents))
    
    newTimerEvent := view1TimerEvents[0]
    expectedTimeout := calculateTimeout(view=1) // Fresh calculation for view 1
    actualTimeout := time.Duration(newTimerEvent.Payload["timeout_ms"].(int64)) * time.Millisecond
    assert.Equal(expectedTimeout, actualTimeout)
}
```

#### 10.8 `TestPartitionModeEvents`
**Objective**: Validate partition mode entry/exit events (data flow lines 556, 67)

**Test Flow**:
```go
func TestPartitionModeEvents(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Create partition that leaves nodes in minority
    partition1 := []types.NodeID{0, 1}     // Minority
    partition2 := []types.NodeID{2, 3, 4}  // Majority
    
    network.CreatePartition(partition1, partition2)
    
    // Wait for partition detection
    time.Sleep(partitionDetectionTimeout + 500*time.Millisecond)
    
    events := tracer.GetEvents()
    
    // Verify minority nodes entered partition mode
    minorityEvents := filterEventsByNodes(events, partition1)
    assert.Contains(minorityEvents, "partition_mode_entered")
    
    partitionEvents := filterEventsByType(minorityEvents, "partition_mode_entered")
    for _, event := range partitionEvents {
        assert.Contains([]uint16{0, 1}, event.NodeID)
        assert.Equal("insufficient_connectivity", event.Payload["reason"])
    }
    
    // Heal partition
    network.HealPartition()
    time.Sleep(1 * time.Second)
    
    // Verify minority nodes exited partition mode
    allEvents := tracer.GetEvents()
    exitEvents := filterEventsByType(allEvents, "partition_mode_exited")
    assert.Equal(2, len(exitEvents)) // Both minority nodes exit
    
    for _, event := range exitEvents {
        assert.Contains([]uint16{0, 1}, event.NodeID)
        assert.Equal("connectivity_restored", event.Payload["reason"])
    }
}
```

**Data Flow Reference**: Lines 556 (enter recovery mode), 67 (partition mode exited)

#### 10.9 `TestNoVotesDuringPartitionRecovery`
**Objective**: Ensure no votes are emitted during partition/recovery mode

**Test Flow**:
```go
func TestNoVotesDuringPartitionRecovery(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Create partition
    partition1 := []types.NodeID{0, 1}     
    partition2 := []types.NodeID{2, 3, 4}  
    network.CreatePartition(partition1, partition2)
    
    // Wait for partition detection
    time.Sleep(partitionDetectionTimeout + 500*time.Millisecond)
    
    // Try to send proposals to minority nodes (should be ignored)
    leader := calculateLeader(0, 5)
    if contains(partition1, leader) {
        nodes[leader].ProposeBlock([]byte("partition_test"))
    }
    
    // Also send proposals within minority partition
    for _, nodeID := range partition1 {
        nodes[nodeID].ProposeBlock([]byte("partition_internal_test"))
    }
    
    time.Sleep(1 * time.Second)
    
    events := tracer.GetEvents()
    minorityEvents := filterEventsByNodes(events, partition1)
    
    // Verify NO votes were sent by minority nodes
    voteEvents := filterEventsByType(minorityEvents, "vote_sent")
    assert.Equal(0, len(voteEvents), "No votes should be sent during partition mode")
    
    // Verify vote suppression was logged
    suppressionEvents := filterEventsByType(minorityEvents, "vote_suppressed")
    assert.Greater(len(suppressionEvents), 0)
    
    for _, event := range suppressionEvents {
        assert.Equal("partition_mode", event.Payload["suppression_reason"])
    }
}
```

#### 10.10 `TestLeaderElectionAndNewViewIdempotent`
**Objective**: Ensure exactly one leader election per view under concurrency

**Test Flow**:
```go
func TestLeaderElectionAndNewViewIdempotent(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Trigger simultaneous timeouts from all nodes
    network.BlockAllMessages()
    time.Sleep(viewTimeout + 200*time.Millisecond)
    network.UnblockAllMessages()
    
    // Wait for view change completion
    time.Sleep(2 * time.Second)
    
    events := tracer.GetEvents()
    
    // Group events by node
    for nodeID := uint16(0); nodeID < 5; nodeID++ {
        nodeEvents := filterEventsByNode(events, nodeID)
        
        // Each node should have exactly one leader_elected event for view 1
        leaderEvents := filterEventsByType(nodeEvents, "leader_elected")
        view1LeaderEvents := filterEventsByView(leaderEvents, 1)
        assert.Equal(1, len(view1LeaderEvents), 
            "Node %d should have exactly 1 leader_elected for view 1", nodeID)
        
        // Each node should have exactly one new_view_started event for view 1
        newViewEvents := filterEventsByType(nodeEvents, "new_view_started") 
        view1NewViewEvents := filterEventsByView(newViewEvents, 1)
        assert.Equal(1, len(view1NewViewEvents),
            "Node %d should have exactly 1 new_view_started for view 1", nodeID)
        
        // Verify order: leader_elected BEFORE new_view_started
        leaderTime := view1LeaderEvents[0].Timestamp
        newViewTime := view1NewViewEvents[0].Timestamp
        assert.True(leaderTime.Before(newViewTime) || leaderTime.Equal(newViewTime),
            "Node %d: leader_elected must occur before or with new_view_started", nodeID)
    }
    
    // Verify all nodes elected the same leader
    expectedLeader := calculateLeader(1, 5)
    for nodeID := uint16(0); nodeID < 5; nodeID++ {
        nodeEvents := filterEventsByNode(events, nodeID)
        leaderEvents := filterEventsByView(filterEventsByType(nodeEvents, "leader_elected"), 1)
        actualLeader := leaderEvents[0].Payload["elected_leader"].(uint16)
        assert.Equal(expectedLeader, actualLeader,
            "All nodes must elect same leader for view 1")
    }
}
```

### **Priority 3: Nice-to-Have Tests**

#### 10.11 `TestNewViewHighestQCRecencyWindow`
**Objective**: Validate NewView includes QC from last 3 views on fast path (line 527)

#### 10.12 `TestViewSyncQuorumMinPeers`
**Objective**: Validate f+1 peer threshold for ViewSync (lines 529-535)

#### 10.13 `TestViewSyncRetryAfterVerificationFailure`
**Objective**: Test retry mechanism after sync verification failure (line 608)

#### 10.14 `TestNewViewDeduplication`
**Objective**: Ensure duplicate NewView messages from same validator are deduped

#### 10.15 `TestLateNewViewMessageAfterQuorum`
**Objective**: Verify NewView messages arriving after quorum formation don't alter chosen highestQC

**Rationale**: Race condition where late-arriving NewView with higher QC might incorrectly influence already-formed proposal. Protocol must lock highestQC selection once quorum reached.

**Test Flow**:
```go
func TestLateNewViewMessageAfterQuorum(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    leader := calculateLeader(1, 5)
    
    // Create different QCs for the race condition
    standardQC := createQC(view=2, block="0x2222") // Lower QC
    higherQC := createQC(view=3, block="0x3333")   // Higher QC that arrives late
    
    // 1. Trigger view change to view 1
    triggerViewChange(nodes, view=1)
    
    // 2. Send exactly 2f+1 NewView messages to form quorum
    quorumSize := 3 // 2f+1 for 5 nodes
    for i := 0; i < quorumSize; i++ {
        newViewMsg := createNewView(view=1, sender=nodes[i+1], highestQC=standardQC)
        nodes[leader].ProcessNewViewMessage(newViewMsg)
    }
    
    // 3. Wait for quorum detection and QC selection
    time.Sleep(50 * time.Millisecond)
    
    events := tracer.GetEvents()
    
    // Verify quorum reached and QC selected
    assert.Contains(events, "new_view_quorum_reached")
    qcSelectedEvent := findEvent(events, "highest_qc_updated")
    selectedQC := qcSelectedEvent.Payload["selected_qc_hash"]
    assert.Equal(standardQC.Hash(), selectedQC) // Standard QC should be selected
    
    // 4. Inject delay before proposal creation (simulate processing time)
    time.Sleep(100 * time.Millisecond)
    
    // 5. Send late NewView with higher QC (after quorum already formed)
    lateNewViewMsg := createNewView(view=1, sender=nodes[4], highestQC=higherQC)
    nodes[leader].ProcessNewViewMessage(lateNewViewMsg)
    
    // 6. Validation: Late message should be ignored
    allEvents := tracer.GetEvents()
    
    // Verify late message was ignored
    lateEvents := filterEventsByType(allEvents, "new_view_message_ignored")
    assert.Greater(len(lateEvents), 0, "Late NewView message should be ignored")
    
    ignoreEvent := lateEvents[len(lateEvents)-1]
    assert.Equal("quorum_already_reached", ignoreEvent.Payload["ignore_reason"])
    
    // 7. Verify proposal uses original QC, not the late higher one
    proposalEvents := filterEventsByType(allEvents, "proposal_created")
    if len(proposalEvents) > 0 {
        proposalEvent := proposalEvents[len(proposalEvents)-1]
        usedQC := proposalEvent.Payload["justify_qc_hash"]
        assert.Equal(standardQC.Hash(), usedQC, 
            "Proposal must use QC from original quorum, not late arrival")
        assert.NotEqual(higherQC.Hash(), usedQC, 
            "Proposal must NOT use late-arriving higher QC")
    }
    
    // 8. Verify no additional highest_qc_updated events after late message
    qcUpdateEvents := filterEventsByType(allEvents, "highest_qc_updated")
    qcUpdatesAfterQuorum := filterEventsAfterTimestamp(qcUpdateEvents, 
        findEvent(allEvents, "new_view_quorum_reached").Timestamp)
    assert.Equal(0, len(qcUpdatesAfterQuorum), 
        "No QC updates should occur after quorum reached")
}
```

**Data Flow Reference**: Lines 192-194 (QC selection must be deterministic and locked after quorum)

#### 10.16 `TestValidatorJoiningMidViewChange`
**Objective**: Ensure validator rejoining during active view change can synchronize and participate correctly

**Rationale**: Tests robustness when state sync and view change mechanisms overlap. Node must handle simultaneous sync and view transition without causing protocol violations.

**Test Flow**:
```go
func TestValidatorJoiningMidViewChange(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    
    // Start with 4-node network (node 4 offline)
    nodes, network, err := integration.CreateTestNetwork(4, tracer, false)
    require.NoError(err)
    
    // Keep node 4 offline initially
    offlineNodeID := uint16(4)
    
    // 1. Progress the 4-node network through some consensus rounds
    for round := 0; round < 2; round++ {
        leader := calculateLeader(round, 4) // Only 4 active nodes
        nodes[leader].ProposeBlock([]byte(fmt.Sprintf("block_%d", round)))
        time.Sleep(500 * time.Millisecond) // Let consensus complete
    }
    
    // 2. Trigger view change on 4-node network (e.g., kill current leader)
    currentView := 2
    currentLeader := calculateLeader(currentView, 4)
    network.BlockNode(currentLeader) // Kill leader to trigger view change
    
    // 3. Let view change process start but not complete
    time.Sleep(viewTimeout/2) // Partial view change progress
    
    events := tracer.GetEvents()
    
    // Verify view change is in progress
    assert.Contains(events, "view_timeout_detected")
    assert.Contains(events, "timeout_message_sent")
    
    // 4. Bring node 4 online during active view change
    offlineNode, err := integration.CreateSingleNode(offlineNodeID, 
        make([]types.NodeID, 5), tracer, network)
    require.NoError(err)
    
    nodes[offlineNodeID] = offlineNode
    network.AddNode(offlineNodeID, offlineNode)
    
    t.Logf("Node %d joining during active view change", offlineNodeID)
    
    // 5. Give system time to handle rejoining node
    time.Sleep(2 * time.Second)
    
    allEvents := tracer.GetEvents()
    node4Events := filterEventsByNode(allEvents, offlineNodeID)
    
    // 6. Validation: Node 4 should sync and participate correctly
    
    // A. Initial state sync should occur
    assert.Contains(node4Events, "state_sync_requested")
    assert.Contains(node4Events, "state_sync_completed")
    
    // B. Node should detect ongoing view change
    viewChangeEvents := []string{
        "timeout_message_received", 
        "new_view_message_received", 
        "view_change_started"
    }
    foundViewChangeEvent := false
    for _, eventType := range viewChangeEvents {
        if containsEvent(node4Events, eventType) {
            foundViewChangeEvent = true
            break
        }
    }
    assert.True(foundViewChangeEvent, 
        "Node 4 must detect ongoing view change")
    
    // C. Node should participate in new view establishment
    assert.Contains(node4Events, "new_view_message_sent")
    assert.Contains(node4Events, "new_view_started")
    
    // D. Critical ordering: sync must complete before or with new view
    syncCompleteTime := findEvent(node4Events, "state_sync_completed").Timestamp
    newViewStartTime := findEvent(node4Events, "new_view_started").Timestamp
    
    assert.True(syncCompleteTime.Before(newViewStartTime) || 
                syncCompleteTime.Equal(newViewStartTime),
        "State sync must complete before or with new view start")
    
    // 7. Verify all nodes converged to same view
    finalView := currentView + 1 // Should advance from view change
    for nodeID, node := range nodes {
        actualView := node.GetCurrentView()
        assert.Equal(finalView, actualView, 
            "Node %d should be at final view %d", nodeID, finalView)
    }
    
    // 8. Verify new leader can propose and node 4 can participate
    newLeader := calculateLeader(finalView, 5) // Now 5 nodes total
    network.UnblockNode(currentLeader) // Unblock for test cleanup
    
    err = nodes[newLeader].ProposeBlock([]byte("post_rejoin_block"))
    assert.NoError(err, "New leader should be able to propose after node rejoin")
    
    time.Sleep(1 * time.Second)
    
    // Verify node 4 can vote (shows successful integration)
    finalEvents := tracer.GetEvents()
    node4FinalEvents := filterEventsByNode(finalEvents, offlineNodeID)
    
    // Look for participation events (proposal_received, vote_sent, etc.)
    participationEvents := []string{
        "proposal_received", 
        "vote_sent", 
        "block_added"
    }
    foundParticipation := false
    for _, eventType := range participationEvents {
        if containsEvent(node4FinalEvents, eventType) {
            foundParticipation = true
            t.Logf("Node 4 participated via: %s", eventType)
            break
        }
    }
    assert.True(foundParticipation, 
        "Node 4 must be able to participate in consensus after rejoin")
}
```

**Data Flow Reference**: Lines 528-535 (slow path sync), 556 (recovery mode), integration with NewView protocol

### 7. Byzantine Behavior Tests (Priority 3, build tag: testing)

**Implementation Note**: All tests in this section **must** be implemented using the Byzantine Strategy Injection pattern detailed in ADR-007. This involves creating test-only strategy implementations in the `pkg/consensus/testing` package and injecting them into the coordinator of malicious nodes.

#### 7.1 `TestByzantineLeaderDuringViewChange`
**Test Flow**:
```go
//go:build testing

func TestByzantineLeaderDuringViewChange(t *testing.T) {
    nodes, network, err := integration.CreateTestNetwork(5, tracer, true) // Enable byzantine
    
    leader := calculateLeader(0, 5)
    
    // Enable byzantine behavior: conflicting NewView messages
    nodes[leader].EnableByzantineBehavior("conflicting_new_views")
    
    // Trigger view change
    network.BlockAllMessages() // Force timeout
    time.Sleep(viewTimeout + 500*time.Millisecond)
    network.UnblockAllMessages()
    
    events := tracer.GetEvents()
    
    // Verify byzantine behavior detected
    assert.Contains(events, "byzantine_detected")
    assert.Contains(events, "evidence_collected")
    assert.Contains(events, "evidence_broadcasted")
    
    // Verify honest nodes advance to next view with new leader
    assert.Contains(events, "view_change_started")
    newLeader := calculateLeader(1, 5)
    assert.NotEqual(leader, newLeader)
}
```

#### 7.2 `TestByzantineTimeoutSpam`
**Test Approach**:
- Byzantine node floods timeout messages
- Verify rate limiting and spam detection
- Check protocol continues correctly
- Validate honest nodes ignore spam

#### 7.3 `TestInvalidTimeoutSignatures`
**Test Approach**:
- Byzantine node sends timeouts with invalid signatures
- Verify signature validation rejects them
- Check invalid timeouts don't contribute to quorum
- Validate protocol safety maintained

#### 7.4 `TestEquivocationDuringViewChange`
**Test Approach**:
- Byzantine node sends different votes for same view
- Verify equivocation detection during view change
- Validate evidence collection and propagation
- Check honest nodes exclude equivocating node

### 8. Edge Cases and Corner Scenarios (Priority 3)

#### 8.1 `TestViewNumberOverflow`
**Test Approach**:
- Simulate consensus near maximum view number
- Test view number wraparound behavior
- Verify no protocol violations
- Check proper view arithmetic

#### 8.2 `TestTimeoutCertificateValidation`
**Test Cases**:
- Insufficient timeout certificates (< 2f+1)
- Invalid signatures in timeout certificates
- Timeout certificates from wrong view
- Mixed valid/invalid certificates

#### 8.3 `TestRaceConditions`
**Test Cases**:
- Timeout occurs just as proposal arrives
- NewView message arrives during timeout processing
- Vote arrives during view transition
- Multiple simultaneous view change triggers

#### 8.4 `TestMemoryPressureDuringViewChange`
**Test Approach**:
- Simulate high memory usage during view change
- Verify graceful degradation
- Check recovery after memory pressure relieved
- Validate no memory leaks

### 9. Performance Benchmarks (Priority 4)

#### 9.1 `BenchmarkViewChangeLatency`
**Metrics**:
- Time from timeout detection to new view start
- Latency across different network sizes (5, 10, 20 nodes)
- Memory allocation during view change
- CPU usage profiling

#### 9.2 `BenchmarkTimeoutDetection`
**Metrics**:
- Timeout detection accuracy (±100ms target)
- Timer precision under load
- False positive rate (premature timeouts)
- Resource usage of timer management

#### 9.3 `BenchmarkNewViewCollection`
**Metrics**:
- NewView message aggregation time
- Processing overhead per message
- Memory usage during collection
- Network bandwidth utilization

## 10. Branch Coverage Completeness Tests

This section contains tests specifically designed to achieve **100% branch coverage** of the view change protocol as specified in the data flow diagram. These tests target specific protocol branches that may not be covered by the general scenarios above.

### **Priority 1: Critical for 100% Coverage**

#### 10.1 `TestNewViewHighestQCTieBreakSameView`
**Objective**: Test NewView QC selection when multiple QCs have same view but different blocks (data flow lines 192-194)

**Test Flow**:
```go
func TestNewViewHighestQCTieBreakSameView(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Create scenario where validators have same-view QCs for different blocks
    // 1. Create two different blocks at same view
    block1 := createBlock(view=1, hash="0x1111")
    block2 := createBlock(view=1, hash="0x2222") 
    
    // 2. Give some validators QC for block1, others QC for block2
    nodes[1].SetHighestQC(createQC(block1)) // QC for block1
    nodes[2].SetHighestQC(createQC(block1))
    nodes[3].SetHighestQC(createQC(block2)) // QC for block2
    nodes[4].SetHighestQC(createQC(block2))
    
    // 3. Trigger view change to view 2
    triggerViewChange(nodes, view=2)
    
    // 4. Validate deterministic tie-breaking occurred
    events := tracer.GetEvents()
    
    // Must use deterministic rule: prefer higher block hash
    expectedWinner := block2 // 0x2222 > 0x1111
    assert.Contains(events, "highest_qc_updated")
    
    qcUpdateEvent := findEvent(events, "highest_qc_updated")
    assert.Equal(expectedWinner.Hash, qcUpdateEvent.Payload["selected_block_hash"])
    assert.Contains(qcUpdateEvent.Payload["reason"], "tie_break_higher_hash")
}
```

**Data Flow Reference**: Lines 192-194 (deterministic tie-breaking logic)

#### 10.2 `TestViewTimeoutBackoffCap`  
**Objective**: Validate timeout backoff formula and maximum cap (data flow line 513)

**Test Flow**:
```go
func TestViewTimeoutBackoffCap(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    baseTimeout := 1 * time.Second
    maxTimeout := 60 * time.Second
    
    // Test timeout progression: min(baseTimeout * 2^min(view, 10), maxTimeout)
    expectedTimeouts := []time.Duration{
        1 * time.Second,  // view 0: 1 * 2^0 = 1s
        2 * time.Second,  // view 1: 1 * 2^1 = 2s  
        4 * time.Second,  // view 2: 1 * 2^2 = 4s
        8 * time.Second,  // view 3: 1 * 2^3 = 8s
        16 * time.Second, // view 4: 1 * 2^4 = 16s
        32 * time.Second, // view 5: 1 * 2^5 = 32s
        60 * time.Second, // view 6: min(64s, 60s) = 60s (capped)
        60 * time.Second, // view 11: min(1*2^10, 60s) = 60s (capped)
    }
    
    for view, expectedTimeout := range expectedTimeouts {
        triggerViewTimeout(nodes, view)
        
        events := tracer.GetEvents()
        timerEvents := filterEventsByType(events, "view_timer_started")
        latestTimer := timerEvents[len(timerEvents)-1]
        
        actualTimeout := time.Duration(latestTimer.Payload["timeout_ms"].(int64)) * time.Millisecond
        assert.Equal(expectedTimeout, actualTimeout, 
            "View %d: expected %v, got %v", view, expectedTimeout, actualTimeout)
        
        if expectedTimeout >= maxTimeout {
            assert.Contains(latestTimer.Payload, "capped_at_max")
        }
    }
}
```

**Data Flow Reference**: Line 513 (timeout formula with cap)

#### 10.3 `TestInvalidNewViewSignaturesAndView`
**Objective**: Verify invalid NewView messages don't contribute to quorum

**Test Flow**:
```go
func TestInvalidNewViewSignaturesAndView(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    leader := calculateLeader(1, 5)
    
    // Send mix of valid and invalid NewView messages to leader
    validNewViews := 2   // Not enough for quorum (need 3)
    invalidNewViews := 2
    
    // Send valid NewView messages
    for i := 0; i < validNewViews; i++ {
        validMsg := createValidNewView(view=1, sender=nodes[i])
        nodes[leader].ProcessNewViewMessage(validMsg)
    }
    
    // Send invalid NewView messages (wrong signatures, wrong view)
    invalidMsg1 := createNewViewWithInvalidSignature(view=1, sender=nodes[3])
    invalidMsg2 := createNewViewWithWrongView(view=0, sender=nodes[4])
    
    nodes[leader].ProcessNewViewMessage(invalidMsg1)
    nodes[leader].ProcessNewViewMessage(invalidMsg2)
    
    // Wait to ensure no proposal is created (insufficient valid NewViews)
    time.Sleep(500 * time.Millisecond)
    
    events := tracer.GetEvents()
    
    // Verify invalid messages were rejected
    assert.Contains(events, "new_view_message_rejected")
    rejectionEvents := filterEventsByType(events, "new_view_message_rejected")
    assert.Equal(2, len(rejectionEvents)) // 2 invalid messages rejected
    
    // Verify no proposal was created (insufficient quorum)
    proposalEvents := filterEventsByType(events, "proposal_created")
    assert.Equal(0, len(proposalEvents))
    
    // Verify leader is still waiting for more NewViews
    leaderState := nodes[leader].GetState()
    assert.Equal("waiting_for_new_views", leaderState.Phase)
}
```

#### 10.4 `TestTimeoutMessageDeduplication`
**Objective**: Ensure duplicate timeout messages don't inflate quorum count

**Test Flow**:
```go
func TestTimeoutMessageDeduplication(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Node 1 sends duplicate timeout messages
    timeoutMsg := createTimeoutMessage(view=0, sender=nodes[1])
    
    // Send same timeout message 3 times
    for i := 0; i < 3; i++ {
        for _, node := range nodes {
            node.ProcessTimeoutMessage(timeoutMsg)
        }
    }
    
    // Send valid timeout from node 2 (should reach quorum: node1 + node2 = 2, need 2f+1=3)
    timeoutMsg2 := createTimeoutMessage(view=0, sender=nodes[2])
    for _, node := range nodes {
        node.ProcessTimeoutMessage(timeoutMsg2)
    }
    
    // Send valid timeout from node 3 (should complete quorum)
    timeoutMsg3 := createTimeoutMessage(view=0, sender=nodes[3])
    for _, node := range nodes {
        node.ProcessTimeoutMessage(timeoutMsg3)
    }
    
    events := tracer.GetEvents()
    
    // Verify deduplication occurred
    dedupEvents := filterEventsByType(events, "timeout_message_deduplicated")
    assert.Greater(len(dedupEvents), 0) // Duplicates were detected
    
    // Verify quorum was reached with exactly 3 unique validators
    quorumEvents := filterEventsByType(events, "timeout_quorum_reached")
    assert.Equal(1, len(quorumEvents))
    
    quorumEvent := quorumEvents[0]
    assert.Equal(3, quorumEvent.Payload["unique_timeout_count"])
    assert.Equal([]uint16{1, 2, 3}, quorumEvent.Payload["timeout_validators"])
}
```

#### 10.5 `TestViewSyncUpdatesLockedQC`
**Objective**: Test lockedQC update during slow path view sync (data flow line 534)

**Test Flow**:
```go
func TestViewSyncUpdatesLockedQC(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Setup: Node 0 has old lockedQC (view 1), other nodes have higher QC (view 3)
    oldQC := createQC(view=1, block="0x1111")
    newQC := createQC(view=3, block="0x3333")
    
    nodes[0].SetLockedQC(oldQC)
    for i := 1; i < 5; i++ {
        nodes[i].SetLockedQC(newQC)
    }
    
    // Disconnect node 0 temporarily to create sync scenario
    network.BlockNode(0)
    
    // Progress consensus to view 4 with other nodes
    progressToView(nodes[1:], view=4)
    
    // Reconnect node 0 - should trigger slow path sync
    network.UnblockNode(0)
    
    // Wait for sync completion
    time.Sleep(2 * time.Second)
    
    events := tracer.GetEvents()
    node0Events := filterEventsByNode(events, 0)
    
    // Verify sync occurred
    assert.Contains(node0Events, "state_sync_requested")
    assert.Contains(node0Events, "state_sync_completed")
    
    // Verify lockedQC was updated to higher QC during sync
    lockedQCEvents := filterEventsByType(node0Events, "locked_qc_updated")
    assert.Greater(len(lockedQCEvents), 0)
    
    updateEvent := lockedQCEvents[len(lockedQCEvents)-1]
    assert.Equal(uint64(3), updateEvent.Payload["new_locked_view"])
    assert.Equal("0x3333", updateEvent.Payload["new_locked_block"])
    assert.Equal("view_sync", updateEvent.Payload["update_reason"])
    
    // Verify final state
    finalLockedQC := nodes[0].GetLockedQC()
    assert.Equal(uint64(3), finalLockedQC.View)
}
```

**Data Flow Reference**: Line 534 (lockedQC update during sync)

### **Priority 2: Important for Completeness**

#### 10.6 `TestNewViewMisdirectedIgnored`
**Objective**: NewView messages sent to wrong leader are ignored

**Test Flow**:
```go
func TestNewViewMisdirectedIgnored(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    correctLeader := calculateLeader(1, 5) // e.g., node 1
    wrongLeader := (correctLeader + 1) % 5   // e.g., node 2
    
    // Validators mistakenly send NewView to wrong leader
    misdirectedCount := 3
    for i := 0; i < misdirectedCount; i++ {
        newViewMsg := createNewView(view=1, sender=nodes[i+2])
        nodes[wrongLeader].ProcessNewViewMessage(newViewMsg)
    }
    
    // Send correct NewViews to right leader
    correctCount := 3 // Quorum
    for i := 0; i < correctCount; i++ {
        newViewMsg := createNewView(view=1, sender=nodes[i+2])
        nodes[correctLeader].ProcessNewViewMessage(newViewMsg)
    }
    
    events := tracer.GetEvents()
    
    // Verify wrong leader ignored misdirected messages
    wrongLeaderEvents := filterEventsByNode(events, wrongLeader)
    ignoredEvents := filterEventsByType(wrongLeaderEvents, "new_view_message_ignored")
    assert.Equal(misdirectedCount, len(ignoredEvents))
    
    for _, event := range ignoredEvents {
        assert.Equal("not_leader_for_view", event.Payload["ignore_reason"])
    }
    
    // Verify correct leader processed messages and created proposal
    correctLeaderEvents := filterEventsByNode(events, correctLeader)
    assert.Contains(correctLeaderEvents, "new_view_quorum_reached")
    assert.Contains(correctLeaderEvents, "proposal_created")
}
```

#### 10.7 `TestPhaseTimerResetOnNewView`
**Objective**: Ensure phase timers reset properly on view change

**Test Flow**:
```go
func TestPhaseTimerResetOnNewView(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Start consensus in view 0
    leader0 := calculateLeader(0, 5)
    nodes[leader0].ProposeBlock([]byte("block_0"))
    
    // Let consensus progress partially, then trigger timeout
    time.Sleep(500 * time.Millisecond) // Partial progress
    network.BlockAllMessages() // Force timeout
    
    // Wait for timeout and view change
    time.Sleep(viewTimeout + 500*time.Millisecond)
    
    network.UnblockAllMessages()
    
    events := tracer.GetEvents()
    
    // Find timer events for both views
    timerEvents := filterEventsByType(events, "view_timer_started")
    assert.Greater(len(timerEvents), 1) // At least 2 timers (view 0 and view 1)
    
    // Verify old timer was cancelled
    cancelEvents := filterEventsByType(events, "view_timer_cancelled")
    assert.Greater(len(cancelEvents), 0)
    
    // Verify new timer started with fresh timeout
    view1TimerEvents := filterTimerEventsByView(timerEvents, 1)
    assert.Equal(1, len(view1TimerEvents))
    
    newTimerEvent := view1TimerEvents[0]
    expectedTimeout := calculateTimeout(view=1) // Fresh calculation for view 1
    actualTimeout := time.Duration(newTimerEvent.Payload["timeout_ms"].(int64)) * time.Millisecond
    assert.Equal(expectedTimeout, actualTimeout)
}
```

#### 10.8 `TestPartitionModeEvents`
**Objective**: Validate partition mode entry/exit events (data flow lines 556, 67)

**Test Flow**:
```go
func TestPartitionModeEvents(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Create partition that leaves nodes in minority
    partition1 := []types.NodeID{0, 1}     // Minority
    partition2 := []types.NodeID{2, 3, 4}  // Majority
    
    network.CreatePartition(partition1, partition2)
    
    // Wait for partition detection
    time.Sleep(partitionDetectionTimeout + 500*time.Millisecond)
    
    events := tracer.GetEvents()
    
    // Verify minority nodes entered partition mode
    minorityEvents := filterEventsByNodes(events, partition1)
    assert.Contains(minorityEvents, "partition_mode_entered")
    
    partitionEvents := filterEventsByType(minorityEvents, "partition_mode_entered")
    for _, event := range partitionEvents {
        assert.Contains([]uint16{0, 1}, event.NodeID)
        assert.Equal("insufficient_connectivity", event.Payload["reason"])
    }
    
    // Heal partition
    network.HealPartition()
    time.Sleep(1 * time.Second)
    
    // Verify minority nodes exited partition mode
    allEvents := tracer.GetEvents()
    exitEvents := filterEventsByType(allEvents, "partition_mode_exited")
    assert.Equal(2, len(exitEvents)) // Both minority nodes exit
    
    for _, event := range exitEvents {
        assert.Contains([]uint16{0, 1}, event.NodeID)
        assert.Equal("connectivity_restored", event.Payload["reason"])
    }
}
```

**Data Flow Reference**: Lines 556 (enter recovery mode), 67 (partition mode exited)

#### 10.9 `TestNoVotesDuringPartitionRecovery`
**Objective**: Ensure no votes are emitted during partition/recovery mode

**Test Flow**:
```go
func TestNoVotesDuringPartitionRecovery(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Create partition
    partition1 := []types.NodeID{0, 1}     
    partition2 := []types.NodeID{2, 3, 4}  
    network.CreatePartition(partition1, partition2)
    
    // Wait for partition detection
    time.Sleep(partitionDetectionTimeout + 500*time.Millisecond)
    
    // Try to send proposals to minority nodes (should be ignored)
    leader := calculateLeader(0, 5)
    if contains(partition1, leader) {
        nodes[leader].ProposeBlock([]byte("partition_test"))
    }
    
    // Also send proposals within minority partition
    for _, nodeID := range partition1 {
        nodes[nodeID].ProposeBlock([]byte("partition_internal_test"))
    }
    
    time.Sleep(1 * time.Second)
    
    events := tracer.GetEvents()
    minorityEvents := filterEventsByNodes(events, partition1)
    
    // Verify NO votes were sent by minority nodes
    voteEvents := filterEventsByType(minorityEvents, "vote_sent")
    assert.Equal(0, len(voteEvents), "No votes should be sent during partition mode")
    
    // Verify vote suppression was logged
    suppressionEvents := filterEventsByType(minorityEvents, "vote_suppressed")
    assert.Greater(len(suppressionEvents), 0)
    
    for _, event := range suppressionEvents {
        assert.Equal("partition_mode", event.Payload["suppression_reason"])
    }
}
```

#### 10.10 `TestLeaderElectionAndNewViewIdempotent`
**Objective**: Ensure exactly one leader election per view under concurrency

**Test Flow**:
```go
func TestLeaderElectionAndNewViewIdempotent(t *testing.T) {
    tracer := mocks.NewConsensusEventTracer()
    nodes, network, err := integration.CreateTestNetwork(5, tracer, false)
    
    // Trigger simultaneous timeouts from all nodes
    network.BlockAllMessages()
    time.Sleep(viewTimeout + 200*time.Millisecond)
    network.UnblockAllMessages()
    
    // Wait for view change completion
    time.Sleep(2 * time.Second)
    
    events := tracer.GetEvents()
    
    // Group events by node
    for nodeID := uint16(0); nodeID < 5; nodeID++ {
        nodeEvents := filterEventsByNode(events, nodeID)
        
        // Each node should have exactly one leader_elected event for view 1
        leaderEvents := filterEventsByType(nodeEvents, "leader_elected")
        view1LeaderEvents := filterEventsByView(leaderEvents, 1)
        assert.Equal(1, len(view1LeaderEvents), 
            "Node %d should have exactly 1 leader_elected for view 1", nodeID)
        
        // Each node should have exactly one new_view_started event for view 1
        newViewEvents := filterEventsByType(nodeEvents, "new_view_started") 
        view1NewViewEvents := filterEventsByView(newViewEvents, 1)
        assert.Equal(1, len(view1NewViewEvents),
            "Node %d should have exactly 1 new_view_started for view 1", nodeID)
        
        // Verify order: leader_elected BEFORE new_view_started
        leaderTime := view1LeaderEvents[0].Timestamp
        newViewTime := view1NewViewEvents[0].Timestamp
        assert.True(leaderTime.Before(newViewTime) || leaderTime.Equal(newViewTime),
            "Node %d: leader_elected must occur before or with new_view_started", nodeID)
    }
    
    // Verify all nodes elected the same leader
    expectedLeader := calculateLeader(1, 5)
    for nodeID := uint16(0); nodeID < 5; nodeID++ {
        nodeEvents := filterEventsByNode(events, nodeID)
        leaderEvents := filterEventsByView(filterEventsByType(nodeEvents, "leader_elected"), 1)
        actualLeader := leaderEvents[0].Payload["elected_leader"].(uint16)
        assert.Equal(expectedLeader, actualLeader,
            "All nodes must elect same leader for view 1")
    }
}
```

### **Priority 3: Nice-to-Have Tests**

#### 10.11 `TestNewViewHighestQCRecencyWindow`
**Objective**: Validate NewView includes QC from last 3 views on fast path (line 527)

#### 10.12 `TestViewSyncQuorumMinPeers`
**Objective**: Validate f+1 peer threshold for ViewSync (lines 529-535)

#### 10.13 `TestViewSyncRetryAfterVerificationFailure`
**Objective**: Test retry mechanism after sync verification failure (line 608)

#### 10.14 `TestNewViewDeduplication`
**Objective**: Ensure duplicate NewView messages from same validator are deduped

## Test Infrastructure Requirements

## Testing Approach and Feasibility

Based on a detailed review of the `pkg/consensus/**` implementation, all functional test scenarios outlined in this document are achievable **without any modifications to the production `HotStuffCoordinator` logic**.

### Core Principles

- **Separation of Concerns**: The architecture exhibits a strong separation between the core consensus logic (`HotStuffCoordinator`) and its infrastructure dependencies (network, storage, crypto).
- **Deterministic State Machine**: The `HotStuffCoordinator` is implemented as a deterministic state machine that reacts to external stimuli: incoming messages and internal timers.
- **Mock-Driven Testing**: The provided mock infrastructure, particularly `MockNetwork`, is sufficiently powerful to control these external stimuli and drive the coordinator into any required state to test all protocol branches.

### Implementation Strategy by Test Type

To ensure both safety and comprehensive coverage, the following implementation strategies **must** be used:

#### 1. Mock Infrastructure Manipulation (Default Strategy)

This strategy applies to **all tests in Sections 1-6, 8, and 10**.

These tests **must** trigger protocol behavior by manipulating the mock infrastructure layers (`MockNetwork`, `MockStorage`, `MockCrypto`) from the outside. The production logic of `HotStuffCoordinator` **must not** be altered or instrumented.

- **Mechanism**: The test harness will use the public API of the mock layers to simulate real-world conditions.
- **Examples**:
    - To test timeouts (`TestViewTimeoutAndRecovery`), the test will use `MockNetwork.BlockNode()` to partition a leader, causing honest nodes to time out naturally.
    - To test race conditions (`TestLateNewViewMessageAfterQuorum`), the test will use `MockNetwork` to delay specific messages.
    - To test state-dependent logic (`TestNewViewHighestQCTieBreakSameView`), the test will use `MockStorage` to pre-populate nodes with the required state *before* the test begins.

#### 2. Byzantine Strategy Injection (For Malicious Behavior)

This strategy applies exclusively to **all tests in Section 7: Byzantine Behavior Tests**.

These tests **must** be implemented using the **Strategy-based Dependency Injection** pattern detailed in `ADR-007`. This ensures malicious logic is physically and logically separate from production code and can only be compiled in testing builds.

- **Mechanism**:
    1. A behavioral interface (e.g., `ProposalStrategy`) is defined in the production `integration` package.
    2. The `HotStuffCoordinator` is refactored to depend on this interface.
    3. A malicious implementation of the interface (e.g., `ByzantineProposalStrategy`) is created in the `pkg/consensus/testing` package, guarded by a `//go:build testing` tag.
    4. The test setup, running with the `testing` tag, will inject the `HonestProposalStrategy` into honest nodes and the `ByzantineProposalStrategy` into the designated malicious node.

This approach provides compiler-enforced safety, guaranteeing that Byzantine code cannot be included in a production build.

### Fault Injection Utilities

```go
// Network control functions
func BlockNode(network *MockNetwork, nodeID types.NodeID)
func UnblockNode(network *MockNetwork, nodeID types.NodeID)
func BlockAllMessages(network *MockNetwork)
func UnblockAllMessages(network *MockNetwork)
func DelayMessages(network *MockNetwork, delay time.Duration)
func DropMessages(network *MockNetwork, dropRate float64)
func CreatePartition(network *MockNetwork, partition1, partition2 []types.NodeID)
func HealPartition(network *MockNetwork)

// Node control functions
func KillNode(node *UnifiedNode)
func RestartNode(node *UnifiedNode)
func PauseNode(node *UnifiedNode, duration time.Duration)
```

### Event Validation Helpers

```go
// Protocol behavior validation
func ValidateViewChange(events []ConsensusEvent, fromView, toView uint64) error
func ValidateTimeoutQuorum(events []ConsensusEvent, nodeCount int) error
func ValidateLeaderElection(events []ConsensusEvent, view uint64, expectedLeader uint16) error
func ValidateStateConsistency(nodes []*UnifiedNode) error
func ValidateEventSequence(events []ConsensusEvent, expectedSequence []string) error

// Event filtering utilities
func FilterEventsByNode(events []ConsensusEvent, nodeID uint16) []ConsensusEvent
func FilterEventsByType(events []ConsensusEvent, eventType string) []ConsensusEvent
func FilterEventsByTimeWindow(events []ConsensusEvent, start, end time.Time) []ConsensusEvent
```

### Byzantine Behavior Injection (testing build tag only)

```go
//go:build testing

// Byzantine behaviors for testing
func EnableConflictingNewViews(node *UnifiedNode)
func EnableTimeoutSpamming(node *UnifiedNode)
func EnableInvalidSignatures(node *UnifiedNode)
func EnableViewManipulation(node *UnifiedNode)
func EnableEquivocation(node *UnifiedNode)
```

## Key Events to Validate

For each test scenario, the following protocol events must be verified:

### Timeout Phase Events
- `view_timer_started` - View timer initialization
- `view_timer_cancelled` - Previous timer cancelled during view change
- `view_timeout_detected` - Timeout detection
- `timeout_message_sent` - Timeout message broadcast
- `timeout_message_received` - Timeout message reception
- `timeout_message_validated` - Signature validation of timeouts
- `timeout_message_deduplicated` - Duplicate timeout messages detected
- `timeout_quorum_reached` - Sufficient timeout certificates collected

### View Change Phase Events
- `view_change_started` - Beginning of view transition
- `new_view_message_sent` - NewView message broadcast
- `new_view_message_received` - NewView message reception
- `new_view_message_validated` - NewView signature validation
- `new_view_message_rejected` - Invalid NewView message rejected
- `new_view_message_ignored` - Misdirected NewView message ignored
- `new_view_quorum_reached` - Sufficient NewView messages collected
- `highest_qc_updated` - QC selection from NewView messages
- `leader_elected` - New leader selection
- `new_view_started` - Completion of view transition

### State Management Events
- `state_sync_requested` - Node requests state synchronization
- `state_sync_completed` - State synchronization finished
- `view_synchronized` - Node synchronized to network view
- `locked_qc_updated` - LockedQC updated during sync or protocol progression

### Partition and Network Events
- `partition_mode_entered` - Node enters partition/recovery mode
- `partition_mode_exited` - Node exits partition mode
- `blame_message_sent` - Leader failure blame message broadcast
- `blame_message_received` - Blame message received from validator
- `vote_suppressed` - Vote suppressed during partition mode

### Byzantine Detection Events
- `byzantine_detected` - Malicious behavior identified
- `evidence_collected` - Evidence of misbehavior gathered
- `evidence_broadcasted` - Evidence shared with network
- `evidence_stored` - Evidence persisted for future reference
- `validator_excluded` - Byzantine validator excluded

### Branch Coverage Events (New for 100% Coverage)
- `new_view_highestqc_tie_break` - Tie-breaking applied for same-view QCs
- `viewsync_peer_count` - Number of peers queried during ViewSync
- `sync_verification_failed` - State sync verification failed, retrying
- `timeout_backoff_capped` - Timeout reached maximum cap
- `new_view_message_late` - NewView message arrived after quorum formed
- `node_rejoining_during_view_change` - Node coming online during active view change

## Performance Requirements

### Latency Targets
- **View change completion**: < 10 seconds (from timeout to new view active)
- **Timeout detection**: ±100ms accuracy
- **NewView collection**: < 2 seconds for quorum
- **State synchronization**: < 5 seconds for typical catch-up

### Resource Limits
- **Memory overhead**: < 50MB additional during view change
- **CPU usage**: < 80% single core during transition
- **Network bandwidth**: Efficient use of available bandwidth
- **Storage I/O**: Minimal disk access during view change

## Success Criteria

### Functional Requirements
- ✅ All view change flows from data flow diagram covered
- ✅ Protocol safety maintained under all failure scenarios
- ✅ Liveness guaranteed (progress eventually made)
- ✅ State consistency preserved across view changes
- ✅ Byzantine behaviors properly detected and contained
- ✅ No deadlocks or livelocks in any scenario

### Quality Requirements
- ✅ Tests pass consistently (< 1% flakiness rate)
- ✅ Performance targets met in all scenarios
- ✅ Complete event validation for all test cases
- ✅ Comprehensive edge case coverage
- ✅ Clean separation of production and testing code

### Coverage Goals
- **Protocol Path Coverage**: 100% of view change paths from data flow diagram
- **Branch Coverage**: 100% branch coverage achieved via Priority 1 tests in Section 10
- **Failure Scenario Coverage**: All identified failure modes tested  
- **Byzantine Behavior Coverage**: All attack vectors from ADR-007 covered
- **Performance Coverage**: Benchmarks for all critical operations
- **Event Coverage**: All events from "Key Events to Validate" section tested

## Implementation Notes

### Design Principles
- Tests work with actual `HotStuffCoordinator` implementation
- No test-specific protocol logic created
- Fault injection through network/node control only
- Event validation against actual emitted events
- Performance measured on real protocol execution
- Byzantine tests isolated with build tags

### Integration Points
- Uses existing test framework from `tests/consensus/happy_path_test.go`
- Extends `pkg/consensus/integration` for node management
- Leverages `pkg/consensus/testing` for event validation
- Utilizes `pkg/consensus/mocks` for controlled environments

### Data Flow Diagram Line References

**Critical Protocol Branches Tested**:
- **Lines 192-194**: Conflicting highestQC tie-breaking → `TestNewViewHighestQCTieBreakSameView`
- **Line 513**: Timeout backoff formula with cap → `TestViewTimeoutBackoffCap`
- **Line 534**: LockedQC update during view sync → `TestViewSyncUpdatesLockedQC`
- **Line 558**: Event ordering (leader_elected before new_view_started) → All view change tests
- **Lines 556, 67**: Partition mode entry/exit → `TestPartitionModeEvents`
- **Lines 527**: Fast path QC recency (3 views) → `TestNewViewHighestQCRecencyWindow`
- **Lines 529-535**: Slow path ViewSync with f+1 peers → `TestViewSyncQuorumMinPeers`
- **Lines 608**: Sync verification retry logic → `TestViewSyncRetryAfterVerificationFailure`

**Timeout Protocol Flow (Lines 510-546)**:
- Line 512: Timeout detection → `TestViewTimeoutAndRecovery`
- Line 517: Timeout message broadcast → `TestTimeoutMessageDeduplication`
- Lines 519-521: Timeout collection and validation → All timeout tests
- Line 523: View change initiation → All view change tests

**NewView Protocol Flow (Lines 538-558)**:
- Line 539: NewView message broadcast → `TestNewViewMisdirectedIgnored`
- Line 542: NewView collection (≥2f+1) → `TestInvalidNewViewSignaturesAndView`
- Line 544: Leader election → `TestLeaderElectionAndNewViewIdempotent`
- Line 558: New view establishment → All tests validate this final state

### Implementation Priorities

**Phase 1 (Priority 1)**: 100% Branch Coverage
1. Implement all tests in Section 10
2. Ensure each test targets specific data flow diagram branches
3. Validate all critical protocol decision points covered

**Phase 2 (Priority 2)**: Core Protocol Validation
1. Implement basic view change and leader failure tests (Sections 1-5)
2. Validate essential timeout and recovery mechanisms
3. Ensure proper event ordering and state management

**Phase 3 (Priority 3)**: Advanced Scenarios
1. Add Byzantine behavior and edge case tests (Sections 6-8)
2. Validate network partition handling
3. Test complex failure combinations

**Phase 4 (Priority 4)**: Performance Validation  
1. Implement benchmarks (Section 9)
2. Establish baseline performance metrics
3. Profile resource usage patterns

### Development Approach
- **Start with Priority 1 tests** to achieve immediate branch coverage
- Add incremental complexity following priority structure
- **Validate each test against specific data flow diagram lines**
- Cross-reference event emissions with protocol specification
- Benchmark performance after functional completion
- Document any protocol behavior discoveries or deviations

### Success Validation Matrix

| Test Category | Data Flow Lines | Coverage Goal | Success Criteria |
|---------------|----------------|---------------|------------------|
| Branch Coverage (P1) | 192-194, 513, 534, 558 | 100% branches | All critical decision points tested |
| Core Protocol (P2) | 510-558 (full flow) | Essential paths | Timeout → NewView → Leader election |
| Advanced Scenarios (P3) | 556, 608, edge cases | Error handling | Byzantine and partition scenarios |
| Performance (P4) | All flows | Baseline metrics | Latency and resource targets met |

This comprehensive plan ensures thorough testing of the view change mechanism while maintaining the principle that tests verify the implementation rather than reimplement the protocol specification. Each test directly maps to specific protocol behaviors defined in the HotStuff data flow diagram.

```go
// Network control functions
func BlockNode(network *MockNetwork, nodeID types.NodeID)
func UnblockNode(network *MockNetwork, nodeID types.NodeID)
func BlockAllMessages(network *MockNetwork)
func UnblockAllMessages(network *MockNetwork)
func DelayMessages(network *MockNetwork, delay time.Duration)
func DropMessages(network *MockNetwork, dropRate float64)
func CreatePartition(network *MockNetwork, partition1, partition2 []types.NodeID)
func HealPartition(network *MockNetwork)

// Node control functions
func KillNode(node *UnifiedNode)
func RestartNode(node *UnifiedNode)
func PauseNode(node *UnifiedNode, duration time.Duration)
```

### Event Validation Helpers

```go
// Protocol behavior validation
func ValidateViewChange(events []ConsensusEvent, fromView, toView uint64) error
func ValidateTimeoutQuorum(events []ConsensusEvent, nodeCount int) error
func ValidateLeaderElection(events []ConsensusEvent, view uint64, expectedLeader uint16) error
func ValidateStateConsistency(nodes []*UnifiedNode) error
func ValidateEventSequence(events []ConsensusEvent, expectedSequence []string) error

// Event filtering utilities
func FilterEventsByNode(events []ConsensusEvent, nodeID uint16) []ConsensusEvent
func FilterEventsByType(events []ConsensusEvent, eventType string) []ConsensusEvent
func FilterEventsByTimeWindow(events []ConsensusEvent, start, end time.Time) []ConsensusEvent
```

### Byzantine Behavior Injection (testing build tag only)

```go
//go:build testing

// Byzantine behaviors for testing
func EnableConflictingNewViews(node *UnifiedNode)
func EnableTimeoutSpamming(node *UnifiedNode)
func EnableInvalidSignatures(node *UnifiedNode)
func EnableViewManipulation(node *UnifiedNode)
func EnableEquivocation(node *UnifiedNode)
```

## Key Events to Validate

For each test scenario, the following protocol events must be verified:

### Timeout Phase Events
- `view_timer_started` - View timer initialization
- `view_timer_cancelled` - Previous timer cancelled during view change
- `view_timeout_detected` - Timeout detection
- `timeout_message_sent` - Timeout message broadcast
- `timeout_message_received` - Timeout message reception
- `timeout_message_validated` - Signature validation of timeouts
- `timeout_message_deduplicated` - Duplicate timeout messages detected
- `timeout_quorum_reached` - Sufficient timeout certificates collected

### View Change Phase Events
- `view_change_started` - Beginning of view transition
- `new_view_message_sent` - NewView message broadcast
- `new_view_message_received` - NewView message reception
- `new_view_message_validated` - NewView signature validation
- `new_view_message_rejected` - Invalid NewView message rejected
- `new_view_message_ignored` - Misdirected NewView message ignored
- `new_view_quorum_reached` - Sufficient NewView messages collected
- `highest_qc_updated` - QC selection from NewView messages
- `leader_elected` - New leader selection
- `new_view_started` - Completion of view transition

### State Management Events
- `state_sync_requested` - Node requests state synchronization
- `state_sync_completed` - State synchronization finished
- `view_synchronized` - Node synchronized to network view
- `locked_qc_updated` - LockedQC updated during sync or protocol progression

### Partition and Network Events
- `partition_mode_entered` - Node enters partition/recovery mode
- `partition_mode_exited` - Node exits partition mode
- `blame_message_sent` - Leader failure blame message broadcast
- `blame_message_received` - Blame message received from validator
- `vote_suppressed` - Vote suppressed during partition mode

### Byzantine Detection Events
- `byzantine_detected` - Malicious behavior identified
- `evidence_collected` - Evidence of misbehavior gathered
- `evidence_broadcasted` - Evidence shared with network
- `evidence_stored` - Evidence persisted for future reference
- `validator_excluded` - Byzantine validator excluded

### Branch Coverage Events (New for 100% Coverage)
- `new_view_highestqc_tie_break` - Tie-breaking applied for same-view QCs
- `viewsync_peer_count` - Number of peers queried during ViewSync
- `sync_verification_failed` - State sync verification failed, retrying
- `timeout_backoff_capped` - Timeout reached maximum cap
- `new_view_message_late` - NewView message arrived after quorum formed
- `node_rejoining_during_view_change` - Node coming online during active view change

## Performance Requirements

### Latency Targets
- **View change completion**: < 10 seconds (from timeout to new view active)
- **Timeout detection**: ±100ms accuracy
- **NewView collection**: < 2 seconds for quorum
- **State synchronization**: < 5 seconds for typical catch-up

### Resource Limits
- **Memory overhead**: < 50MB additional during view change
- **CPU usage**: < 80% single core during transition
- **Network bandwidth**: Efficient use of available bandwidth
- **Storage I/O**: Minimal disk access during view change

## Success Criteria

### Functional Requirements
- ✅ All view change flows from data flow diagram covered
- ✅ Protocol safety maintained under all failure scenarios
- ✅ Liveness guaranteed (progress eventually made)
- ✅ State consistency preserved across view changes
- ✅ Byzantine behaviors properly detected and contained
- ✅ No deadlocks or livelocks in any scenario

### Quality Requirements
- ✅ Tests pass consistently (< 1% flakiness rate)
- ✅ Performance targets met in all scenarios
- ✅ Complete event validation for all test cases
- ✅ Comprehensive edge case coverage
- ✅ Clean separation of production and testing code

### Coverage Goals
- **Protocol Path Coverage**: 100% of view change paths from data flow diagram
- **Branch Coverage**: 100% branch coverage achieved via Priority 1 tests in Section 10
- **Failure Scenario Coverage**: All identified failure modes tested  
- **Byzantine Behavior Coverage**: All attack vectors from ADR-007 covered
- **Performance Coverage**: Benchmarks for all critical operations
- **Event Coverage**: All events from "Key Events to Validate" section tested

## Implementation Notes

### Design Principles
- Tests work with actual `HotStuffCoordinator` implementation
- No test-specific protocol logic created
- Fault injection through network/node control only
- Event validation against actual emitted events
- Performance measured on real protocol execution
- Byzantine tests isolated with build tags

### Integration Points
- Uses existing test framework from `tests/consensus/happy_path_test.go`
- Extends `pkg/consensus/integration` for node management
- Leverages `pkg/consensus/testing` for event validation
- Utilizes `pkg/consensus/mocks` for controlled environments

### Data Flow Diagram Line References

**Critical Protocol Branches Tested**:
- **Lines 192-194**: Conflicting highestQC tie-breaking → `TestNewViewHighestQCTieBreakSameView`
- **Line 513**: Timeout backoff formula with cap → `TestViewTimeoutBackoffCap`
- **Line 534**: LockedQC update during view sync → `TestViewSyncUpdatesLockedQC`
- **Line 558**: Event ordering (leader_elected before new_view_started) → All view change tests
- **Lines 556, 67**: Partition mode entry/exit → `TestPartitionModeEvents`
- **Lines 527**: Fast path QC recency (3 views) → `TestNewViewHighestQCRecencyWindow`
- **Lines 529-535**: Slow path ViewSync with f+1 peers → `TestViewSyncQuorumMinPeers`
- **Lines 608**: Sync verification retry logic → `TestViewSyncRetryAfterVerificationFailure`

**Timeout Protocol Flow (Lines 510-546)**:
- Line 512: Timeout detection → `TestViewTimeoutAndRecovery`
- Line 517: Timeout message broadcast → `TestTimeoutMessageDeduplication`
- Lines 519-521: Timeout collection and validation → All timeout tests
- Line 523: View change initiation → All view change tests

**NewView Protocol Flow (Lines 538-558)**:
- Line 539: NewView message broadcast → `TestNewViewMisdirectedIgnored`
- Line 542: NewView collection (≥2f+1) → `TestInvalidNewViewSignaturesAndView`
- Line 544: Leader election → `TestLeaderElectionAndNewViewIdempotent`
- Line 558: New view establishment → All tests validate this final state

### Implementation Priorities

**Phase 1 (Priority 1)**: 100% Branch Coverage
1. Implement all tests in Section 10
2. Ensure each test targets specific data flow diagram branches
3. Validate all critical protocol decision points covered

**Phase 2 (Priority 2)**: Core Protocol Validation
1. Implement basic view change and leader failure tests (Sections 1-5)
2. Validate essential timeout and recovery mechanisms
3. Ensure proper event ordering and state management

**Phase 3 (Priority 3)**: Advanced Scenarios
1. Add Byzantine behavior and edge case tests (Sections 6-8)
2. Validate network partition handling
3. Test complex failure combinations

**Phase 4 (Priority 4)**: Performance Validation  
1. Implement benchmarks (Section 9)
2. Establish baseline performance metrics
3. Profile resource usage patterns

### Development Approach
- **Start with Priority 1 tests** to achieve immediate branch coverage
- Add incremental complexity following priority structure
- **Validate each test against specific data flow diagram lines**
- Cross-reference event emissions with protocol specification
- Benchmark performance after functional completion
- Document any protocol behavior discoveries or deviations

### Success Validation Matrix

| Test Category | Data Flow Lines | Coverage Goal | Success Criteria |
|---------------|----------------|---------------|------------------|
| Branch Coverage (P1) | 192-194, 513, 534, 558 | 100% branches | All critical decision points tested |
| Core Protocol (P2) | 510-558 (full flow) | Essential paths | Timeout → NewView → Leader election |
| Advanced Scenarios (P3) | 556, 608, edge cases | Error handling | Byzantine and partition scenarios |
| Performance (P4) | All flows | Baseline metrics | Latency and resource targets met |

This comprehensive plan ensures thorough testing of the view change mechanism while maintaining the principle that tests verify the implementation rather than reimplement the protocol specification. Each test directly maps to specific protocol behaviors defined in the HotStuff data flow diagram.
