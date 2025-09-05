# ADR-002: Scalable Multi-Level Topology Collection Protocol

## Status
Proposed

## Date
2025-09-03

## Context

The current vTCP Topology Collection Protocol has a fixed path length limitation of 7 nodes maximum (including sender S and receiver R), which constrains payment path discovery in larger network topologies. The existing architecture uses separate transaction classes for each distance level:

- `MaxFlowCalculationSourceFstLevelTransaction` - executes on first-level nodes (S1) relative to sender (S)
- `MaxFlowCalculationSourceSndLevelTransaction` - executes on second-level nodes (S2) relative to sender (S)
- `MaxFlowCalculationTargetFstLevelTransaction` - executes on first-level nodes (R1) relative to receiver (R)
- `MaxFlowCalculationTargetSndLevelTransaction` - executes on second-level nodes (R2) relative to receiver (R)

This results in a topology structure: `S → S1 → S2 → X → R2 → R1 → R` where X is the central exchange/bridge node discovered from the intersection of S2 and R2 topologies.

### Current Architecture Limitations

1. **Fixed Path Length**: Maximum path length is hardcoded to 7 nodes, limiting payment path discovery in larger networks
2. **Rigid Transaction Structure**: Separate transaction classes for each level create maintenance overhead and prevent flexible distance configuration
3. **Topology Size Explosion**: Extending beyond 2 levels would cause combinatorial explosion in topology data volume
4. **Performance Constraints**: Large topology datasets become computationally expensive for path finding algorithms

### Business Requirements

Network growth requires support for longer payment paths while maintaining:
- Reasonable topology collection performance
- Manageable data volumes for path calculation
- Efficient resource utilization
- Flexible distance configuration per payment scenario

## Decision

We will implement a **Scalable Multi-Level Topology Collection Protocol** that addresses both path length limitations and topology size management through two complementary approaches:

### Phase 1: Unified Distance-Configurable Transactions

Replace the existing four separate transaction classes with two unified transactions that accept configurable distance levels:

#### 1.1 New Transaction Architecture

**MaxFlowCalculationSourceLevelTransaction** (replaces both Fst and Snd level transactions):
```cpp
class MaxFlowCalculationSourceLevelTransaction : public BaseTransaction {
public:
    MaxFlowCalculationSourceLevelTransaction(
        MaxFlowCalculationSourceLevelMessage::Shared message,
        ContractorsManager *contractorsManager,
        EquivalentsSubsystemsRouter *equivalentsSubsystemsRouter,
        ExchangeRatesManager *exchangeRatesManager,
        Logger &logger,
        bool iAmGateway,
        uint8_t currentDistanceLevel,  // New parameter
        uint8_t maxDistanceLevel);     // New parameter
        
private:
    uint8_t mCurrentDistanceLevel;
    uint8_t mMaxDistanceLevel;
    
    void processDistanceLevel();
    bool shouldPropagateToNextLevel() const;
    void sendToNextLevel();
};
```

**MaxFlowCalculationTargetLevelTransaction** (replaces both Fst and Snd level transactions):
```cpp
class MaxFlowCalculationTargetLevelTransaction : public BaseTransaction {
public:
    MaxFlowCalculationTargetLevelTransaction(
        MaxFlowCalculationTargetLevelMessage::Shared message,
        ContractorsManager *contractorsManager,
        EquivalentsSubsystemsRouter *equivalentsSubsystemsRouter,
        ExchangeRatesManager *exchangeRatesManager,
        Logger &logger,
        bool iAmGateway,
        uint8_t currentDistanceLevel,  // New parameter
        uint8_t maxDistanceLevel);     // New parameter
        
private:
    uint8_t mCurrentDistanceLevel;
    uint8_t mMaxDistanceLevel;
    
    void processDistanceLevel();
    bool shouldPropagateToNextLevel() const;
    void sendToNextLevel();
};
```

#### 1.2 Enhanced Message Structure

**MaxFlowCalculationSourceLevelMessage** (replaces both Fst and Snd level messages):
```cpp
class MaxFlowCalculationSourceLevelMessage : public SenderMessage {
public:
    MaxFlowCalculationSourceLevelMessage(
        const SerializedEquivalent equivalent,
        ContractorID idOnReceiverSide,
        HopsCount_t hopsCount,
        uint8_t currentDistanceLevel,     // New field
        uint8_t maxDistanceLevel,         // New field
        vector<SerializedEquivalent> exchangeEquivalents = {});

    uint8_t getCurrentDistanceLevel() const;
    uint8_t getMaxDistanceLevel() const;
    
private:
    uint8_t mCurrentDistanceLevel;
    uint8_t mMaxDistanceLevel;
};
```

**MaxFlowCalculationTargetLevelMessage** (replaces both Fst and Snd level messages):
```cpp
class MaxFlowCalculationTargetLevelMessage : public MaxFlowCalculationMessage {
public:
    MaxFlowCalculationTargetLevelMessage(
        const SerializedEquivalent equivalent,
        ContractorID idOnReceiverSide,
        vector<BaseAddress::Shared> targetAddresses,
        bool isTargetGateway,
        HopsCount_t hopsCount,
        uint8_t currentDistanceLevel,     // New field
        uint8_t maxDistanceLevel,         // New field
        vector<SerializedEquivalent> exchangeEquivalents = {});

    uint8_t getCurrentDistanceLevel() const;
    uint8_t getMaxDistanceLevel() const;
    
private:
    uint8_t mCurrentDistanceLevel;
    uint8_t mMaxDistanceLevel;
};
```

#### 1.3 Transaction Logic Flow

The unified transactions will operate based on distance levels:

1. **Initiator Configuration**: 
   - Sender (S) configures `maxDistanceLevel` via constructor parameter
   - Receiver (R) receives `maxDistanceLevel` via initiation message
   
2. **Message Propagation**:
   - Each node receives `currentDistanceLevel` and `maxDistanceLevel` via message
   - Transaction logic determines whether to propagate to next level: `currentDistanceLevel < maxDistanceLevel`
   
3. **Dynamic Behavior**:
   - Level 1: Always sends topology to initiator, conditionally propagates to level 2
   - Level 2+: Sends topology to initiator, conditionally propagates to next level
   - Final Level: Only sends topology to initiator, no further propagation

### Phase 2: Selective Hub/Exchange-Based Topology Collection

To prevent combinatorial explosion in topology size while enabling longer paths, implement selective propagation based on node capabilities:

#### 2.1 Selective Propagation Logic

Beyond level 2 (S2 and R2), topology collection requests will be sent only to:
- **Hub Nodes**: Nodes with `mIAmGateway = true`
- **Exchange Nodes**: Nodes with available exchange rates (`ExchangeRatesManager` has non-empty rates)

#### 2.2 Enhanced Node Selection Criteria

```cpp
class MaxFlowCalculationSourceLevelTransaction : public BaseTransaction {
private:
    vector<ContractorID> getSelectiveNeighborsForPropagation(uint8_t distanceLevel) const {
        if (distanceLevel <= 2) {
            // Use existing logic - all neighbors with outgoing flows
            return mEquivalentsSubsystemsRouter->trustLinesManager(mEquivalent)
                ->firstLevelNeighborsWithOutgoingFlow().first;
        } else {
            // Selective propagation - only hubs and exchange nodes
            vector<ContractorID> selectiveNeighbors;
            auto allNeighbors = mEquivalentsSubsystemsRouter->trustLinesManager(mEquivalent)
                ->firstLevelNeighborsWithOutgoingFlow().first;
                
            for (const auto& neighborID : allNeighbors) {
                if (isHubNode(neighborID) || isExchangeCapableNode(neighborID)) {
                    selectiveNeighbors.push_back(neighborID);
                }
            }
            return selectiveNeighbors;
        }
    }
    
    bool isHubNode(ContractorID nodeID) const {
        // Check if neighbor node is a hub/gateway
        // Implementation depends on hub discovery mechanism
        return mEquivalentsSubsystemsRouter->isNeighborGateway(mEquivalent, nodeID);
    }
    
    bool isExchangeCapableNode(ContractorID nodeID) const {
        // Check if neighbor has exchange capabilities
        return mExchangeRatesManager->hasNeighborExchangeCapabilities(nodeID);
    }
};
```

#### 2.3 Hub/Exchange Discovery Integration

The selective propagation leverages existing capabilities:
- **Hub Identification**: Uses existing `mIAmGateway` flag and gateway neighbor detection
- **Exchange Capability**: Utilizes `ExchangeRatesManager` to identify exchange-capable neighbors
- **Topology Efficiency**: Significantly reduces topology collection scope while maintaining path discovery through high-connectivity nodes

### Implementation Strategy

#### Phase 1: Transaction Unification (Weeks 1-2)
1. Create unified transaction classes with distance level parameters
2. Create unified message classes with distance level fields
3. Update message routing and transaction scheduling
4. Implement backward-compatible message type handling
5. Remove deprecated transaction classes

#### Phase 2: Selective Collection (Weeks 3-4)
1. Implement selective neighbor identification logic
2. Add hub/exchange node detection mechanisms
3. Update topology collection algorithms for selective propagation
4. Integrate with existing caching mechanisms
5. Performance testing and optimization

### Configuration Parameters

```cpp
// New configuration options for topology collection
struct TopologyCollectionConfig {
    uint8_t maxSourceDistance = 3;      // Configurable sender-side distance
    uint8_t maxTargetDistance = 3;      // Configurable receiver-side distance
    bool enableSelectiveCollection = true; // Enable hub/exchange-based selection
    uint8_t selectiveCollectionThreshold = 2; // Distance level to start selective collection
};
```

### Example Scenarios

#### Scenario 1: Extended Path with Selective Collection
```
Configuration: maxSourceDistance = 4, maxTargetDistance = 3, selective = true

Path Structure: S → S1 → S2 → S3(hub) → S4(exchange) → R3 → R2 → R1 → R
Total Length: 9 nodes (8 hops)

Topology Collection:
- S → S1: All neighbors (level 1)
- S1 → S2: All neighbors (level 2)  
- S2 → S3: Only hubs/exchanges (level 3, selective)
- S3 → S4: Only hubs/exchanges (level 4, selective)
- R → R1: All neighbors (level 1)
- R1 → R2: All neighbors (level 2)
- R2 → R3: Only hubs/exchanges (level 3, selective)
```

#### Scenario 2: Asymmetric Distance Configuration
```
Configuration: maxSourceDistance = 2, maxTargetDistance = 4, selective = true

Path Structure: S → S1 → S2 → X → R4 → R3 → R2 → R1 → R
Total Length: 9 nodes (8 hops)

Benefits:
- Sender-side topology remains manageable (2 levels)
- Receiver-side extends deeper for better path discovery
- Selective collection prevents topology explosion
```

## Consequences

### Positive
- **Flexible Path Length**: Configurable distance levels enable paths longer than 7 nodes
- **Reduced Code Complexity**: Single transaction class per side instead of multiple level-specific classes
- **Topology Size Control**: Selective collection prevents combinatorial explosion
- **Performance Optimization**: Hub/exchange-based selection focuses on high-value nodes
- **Configuration Flexibility**: Different distance levels for sender and receiver sides
- **Maintainability**: Unified codebase is easier to maintain and extend
- **Scalability**: Protocol can handle larger networks efficiently

### Negative
- **Implementation Complexity**: More sophisticated logic for distance level handling and selective collection
- **Migration Effort**: Requires replacing existing transaction classes and updating message routing
- **Testing Complexity**: More scenarios to test with different distance configurations
- **Memory Overhead**: Additional fields in messages and transaction state
- **Hub Discovery Dependency**: Selective collection effectiveness depends on accurate hub/exchange identification

### Risk Mitigation
- **Gradual Migration**: Implement Phase 1 first, then Phase 2 to minimize integration risk
- **Configuration Validation**: Add bounds checking for distance level parameters
- **Performance Monitoring**: Implement metrics to track topology collection efficiency
- **Fallback Mechanisms**: Provide option to disable selective collection if needed
- **Comprehensive Testing**: Test all distance level combinations and selective collection scenarios

## Implementation Details

### Transaction Lifecycle

```cpp
// Unified transaction execution flow
TransactionResult::SharedConst MaxFlowCalculationSourceLevelTransaction::run() {
    // Send exchange rates if needed (existing logic)
    sendExchangeRatesIfNeeded();
    
    // Send topology to initiator
    if (mIAmGateway) {
        sendGatewayResultToInitiator(mEquivalent);
    } else {
        sendResultToInitiator(mEquivalent);
    }
    
    // Determine if should propagate to next level
    if (shouldPropagateToNextLevel()) {
        sendToNextLevel();
    }
    
    return resultDone();
}

bool MaxFlowCalculationSourceLevelTransaction::shouldPropagateToNextLevel() const {
    return mCurrentDistanceLevel < mMaxDistanceLevel;
}

void MaxFlowCalculationSourceLevelTransaction::sendToNextLevel() {
    auto neighbors = getSelectiveNeighborsForPropagation(mCurrentDistanceLevel + 1);
    
    for (const auto& neighborID : neighbors) {
        if (neighborID == mMessage->idOnReceiverSide) {
            continue; // Skip sender
        }
        
        sendMessage<MaxFlowCalculationSourceLevelMessage>(
            mContractorsManager->contractorMainAddress(neighborID),
            mEquivalent,
            mContractorsManager->idOnContractorSide(neighborID),
            mContractorsManager->contractorAddresses(mMessage->idOnReceiverSide),
            mMessage->exchangeEquivalents(),
            mCurrentDistanceLevel + 1,  // Increment distance level
            mMaxDistanceLevel);
    }
}
```

### Message Serialization Enhancement

```cpp
pair<BytesShared, size_t> MaxFlowCalculationSourceLevelMessage::serializeToBytes() const {
    // Existing serialization + new distance level fields
    size_t totalSize = SenderMessage::kOffsetToInheritedBytes() +
                      sizeof(HopsCount_t) +
                      sizeof(uint8_t) + // exchangeEquivalents count
                      mExchangeEquivalents.size() * sizeof(SerializedEquivalent) +
                      sizeof(uint8_t) + // currentDistanceLevel
                      sizeof(uint8_t);  // maxDistanceLevel
    
    // Implementation continues with serialization logic...
}
```

### Configuration Integration

```cpp
// Integration with existing configuration system
class CollectTopologyTransaction : public BaseTransaction {
private:
    TopologyCollectionConfig mConfig;
    
public:
    CollectTopologyTransaction(
        // existing parameters...
        const TopologyCollectionConfig& config = TopologyCollectionConfig{}) :
        // existing initialization...
        mConfig(config) {}
        
    void initiateSourceSideCollection() {
        // Send with configured distance levels
        sendMessage<MaxFlowCalculationSourceLevelMessage>(
            // existing parameters...
            1,  // currentDistanceLevel starts at 1
            mConfig.maxSourceDistance);
    }
};
```

## Testing Strategy

### Unit Tests
1. Test distance level parameter handling in unified transactions
2. Test selective neighbor identification logic
3. Test message serialization/deserialization with new fields
4. Test configuration validation and bounds checking

### Integration Tests
1. Test end-to-end topology collection with various distance configurations
2. Test selective collection with hub and exchange nodes
3. Test backward compatibility during migration period
4. Test performance with extended path lengths

### Performance Tests
1. Measure topology collection time vs. distance levels
2. Compare topology data volume with and without selective collection
3. Test path calculation performance with extended topologies
4. Validate memory usage with different configurations

## Migration Plan

### Phase 1: Unified Transaction Implementation
1. **Week 1**: Implement unified transaction classes with distance level support
2. **Week 2**: Create unified message classes and update message routing
3. **Week 3**: Update transaction scheduling and message handling
4. **Week 4**: Integration testing and bug fixes

### Phase 2: Selective Collection Implementation
1. **Week 5**: Implement selective neighbor identification logic
2. **Week 6**: Add hub/exchange detection mechanisms
3. **Week 7**: Integration with topology collection flow
4. **Week 8**: Performance testing and optimization

### Phase 3: Migration and Cleanup
1. **Week 9**: Remove deprecated transaction classes
2. **Week 10**: Update documentation and configuration examples
3. **Week 11**: Final testing and validation
4. **Week 12**: Production readiness verification

## Future Enhancements

### Dynamic Distance Adjustment
- Implement adaptive distance level adjustment based on network conditions
- Add machine learning-based optimization for distance configuration

### Advanced Selective Criteria
- Extend selective collection beyond hubs and exchanges
- Add node reputation and performance-based selection criteria

### Cross-Equivalent Path Optimization
- Optimize selective collection for cross-equivalent payments
- Implement exchange-rate-aware selective propagation

## Related Decisions
- [ADR-001: Neighbor Exchange Capability Discovery Protocol](ADR-001-neighbor-exchange-capability-discovery.md)
- [Topology Collection Protocol](../protocols/topology-collection-protocol.md)
- Exchange Rate Management Architecture

## Validation Criteria

This decision will be considered successful when:
1. Payment paths longer than 7 nodes can be successfully discovered and utilized
2. Topology collection performance remains acceptable (< 50% increase in collection time) with extended distance levels
3. Selective collection reduces topology data volume by at least 60% compared to full collection beyond level 2
4. Hub and exchange-based selective collection maintains > 90% path discovery effectiveness
5. Configuration flexibility allows different distance levels for sender and receiver sides
6. Memory usage increase is proportional to configured distance levels (linear growth, not exponential)
7. Integration with existing exchange rate and caching mechanisms works seamlessly
8. Migration from existing transaction classes completes without breaking existing functionality
