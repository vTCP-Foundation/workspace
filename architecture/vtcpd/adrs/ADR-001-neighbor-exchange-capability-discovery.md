# ADR-001: Neighbor Exchange Capability Discovery Protocol

## Status
Proposed

## Date
2025-09-02

## Context

The vTCP Topology Collection Protocol as described in the topology-collection-protocol.md enables nodes to collect network topology and exchange rate data for optimal payment path calculation. However, a critical limitation exists when dealing with 7-node paths where the exchange node (E) is positioned in the middle (4th position).

### Problem Description

In a 7-node payment path: `S → S1 → S2 → E → R2 → R1 → R`, where:
- S (1st node): Sender collecting topology
- E (4th node): Exchange node providing currency conversion
- R (7th node): Receiver

The current protocol has a gap: node S cannot discover that E is an exchange node because:
1. E does not execute any of the topology collection transactions:
   - `MaxFlowCalculationSourceFstLevelTransaction`
   - `MaxFlowCalculationSourceSndLevelTransaction` 
   - `MaxFlowCalculationTargetFstLevelTransaction`
   - `MaxFlowCalculationTargetSndLevelTransaction`
2. Only these transactions contain logic for sending exchange rates to S
3. Without exchange rate information from E, S cannot construct the complete path to R

### Current Architecture Limitations

The existing `ExchangeRatesManager` contains:
- `mExchangeRates`: Local exchange rates configured on the node
- `mExternalExchangeRates`: Exchange rates received from other nodes during topology collection

However, nodes lack awareness of which neighbors are exchange-capable and for which currency pairs, preventing effective path discovery through intermediate exchange nodes.

## Decision

We will implement a **Neighbor Exchange Capability Discovery Protocol** that enables nodes to maintain awareness of their neighbors' exchange capabilities and request exchange rates from intermediate exchange nodes during topology collection.

### Core Components

#### 1. Exchange Capability Notification System
When a node's exchange rates are updated via `addOrUpdate()` (but NOT via `addOrUpdateExternal()`), the node will:
1. Identify all neighbors with trust lines in any equivalent involved in current exchange pairs
2. Send exchange capability notifications to these neighbors
3. Include information about ALL current exchange pairs, not just the updated pair

#### 2. New Message Types

##### ExchangeCapabilityNotificationMessage
```cpp
class ExchangeCapabilityNotificationMessage : public SenderMessage {
public:
    struct ExchangePair {
        SerializedEquivalent equivalentFrom;
        SerializedEquivalent equivalentTo;
    };
    
    vector<ExchangePair> exchangePairs;
    // Inherited: senderAddresses, equivalent
};
```

##### ExchangeRateRequestMessage
```cpp
class ExchangeRateRequestMessage : public SenderMessage {
public:
    SerializedEquivalent equivalentFrom;
    SerializedEquivalent equivalentTo;
    vector<BaseAddress::Shared> finalReceiverAddresses; // Address of S (topology collector)
    // Inherited: senderAddresses, equivalent
};
```

#### 3. Enhanced ExchangeRatesManager Data Structure
Add new data structure to track neighbor exchange capabilities (separate from `mExternalExchangeRates`):
```cpp
class ExchangeRatesManager {
private:
    // New structure for neighbor capabilities
    map<ContractorID, set<pair<SerializedEquivalent, SerializedEquivalent>>> mNeighborExchangeCapabilities;
    map<ContractorID, DateTime> mNeighborCapabilityLastUpdate;

public:
    void addNeighborExchangeCapability(
        const ContractorID neighborID,
        const vector<pair<SerializedEquivalent, SerializedEquivalent>>& exchangePairs);
    
    set<pair<SerializedEquivalent, SerializedEquivalent>> getNeighborExchangeCapabilities(
        const ContractorID neighborID) const;
    
    vector<ContractorID> getExchangeCapableNeighbors(
        const SerializedEquivalent equivFrom,
        const SerializedEquivalent equivTo) const;
    
    void removeNeighborCapabilities(const ContractorID neighborID);
};
```

#### 4. Enhanced Topology Collection Logic
Nodes executing `MaxFlowCalculationSourceSndLevelTransaction` (S2) or `MaxFlowCalculationTargetFstLevelTransaction` (R1) will:
1. Check their `mNeighborExchangeCapabilities` for relevant exchange nodes
2. Send `ExchangeRateRequestMessage` to identified exchange-capable neighbors
3. Exchange nodes will respond by sending exchange rates directly to the topology collector (S)

### Transaction Architecture

#### 1. Exchange Capability Notification Handling
**ReceiveExchangeCapabilityNotificationTransaction** (new transaction for node B in the example):
- Receives `ExchangeCapabilityNotificationMessage`
- Processes exchange capability information
- Stores data in `ExchangeRatesManager.mNeighborExchangeCapabilities`

#### 2. Exchange Rate Request Handling
**ReceiveExchangeRateRequestTransaction** (new transaction in max_flow_calculation/ directory):
- Receives `ExchangeRateRequestMessage` from topology collection nodes
- Retrieves requested exchange rate from local `mExchangeRates`
- Sends `ExchangeRatesMessage` directly to the final receiver (S) specified in the request

#### 3. Exchange Rate Setting Enhancement
The transaction responsible for setting exchange rates (via `addOrUpdate()`) must:
- Have access to `EquivalentsSubsystemsRouter` to analyze neighbor trust lines
- Implement notification logic to send `ExchangeCapabilityNotificationMessage` to relevant neighbors
- Determine which neighbors have trust lines in equivalents involved in exchange pairs

#### 4. Enhanced Topology Collection Transactions
**MaxFlowCalculationSourceSndLevelTransaction** and **MaxFlowCalculationTargetFstLevelTransaction** enhancement:
- Add logic to check `mNeighborExchangeCapabilities` for relevant exchange nodes
- Send `ExchangeRateRequestMessage` to identified exchange-capable neighbors
- Continue with normal topology collection flow

### Implementation Strategy

#### Phase 1: Message Infrastructure
1. Implement `ExchangeCapabilityNotificationMessage` class
2. Implement `ExchangeRateRequestMessage` class
3. Add message routing in `TailManager` and `IncomingRemoteNode`
4. Create `ReceiveExchangeCapabilityNotificationTransaction`
5. Create `ReceiveExchangeRateRequestTransaction` in max_flow_calculation/ directory

#### Phase 2: ExchangeRatesManager Enhancement
1. Add `mNeighborExchangeCapabilities` data structure
2. Implement neighbor capability management methods
3. Add cleanup logic for disconnected neighbors

#### Phase 3: Exchange Rate Setting Integration
1. Enhance exchange rate setting transaction with `EquivalentsSubsystemsRouter` access
2. Implement neighbor discovery based on trust line relationships
3. Add notification logic for all current exchange pairs

#### Phase 4: Topology Collection Integration
1. Enhance `MaxFlowCalculationSourceSndLevelTransaction` and `MaxFlowCalculationTargetFstLevelTransaction`
2. Add neighbor capability checking logic
3. Implement exchange rate request logic for discovered exchange nodes

### Example Scenario

Given the example from the problem description:
1. **Node A** has trust lines with **Node B** in equivalents 1 and 3
2. **Node A** has exchange rates for pairs: 1→2, 2→1, 1→4, 3→4, 5→4
3. When **Node A** receives an update for pair 2→1 via `addOrUpdate()`:
   - Exchange rate setting transaction identifies that Node A shares trust lines with Node B in equivalents 1 and 3
   - Node A sends `ExchangeCapabilityNotificationMessage` to Node B containing pairs: 1→2, 2→1, 1→4, 3→4 (excluding 5→4 as no trust line exists in equivalent 5)
   - Node B receives the message via `ReceiveExchangeCapabilityNotificationTransaction`
   - Node B stores this information: `mNeighborExchangeCapabilities[NodeA] = {(1,2), (2,1), (1,4), (3,4)}`

Later, during topology collection:
- If **Node B** executes `MaxFlowCalculationSourceSndLevelTransaction` or `MaxFlowCalculationTargetFstLevelTransaction`
- Node B checks its `mNeighborExchangeCapabilities` and finds that Node A supports required exchange pairs
- Node B sends `ExchangeRateRequestMessage` to Node A with specific pair and S's address
- Node A receives the request via `ReceiveExchangeRateRequestTransaction`
- Node A sends `ExchangeRatesMessage` directly to **S** (topology collector)
- This enables complete path discovery through Node A as an intermediate exchange node

## Consequences

### Positive
- **Complete Path Discovery**: Enables discovery of payment paths through intermediate exchange nodes
- **Distributed Knowledge**: Each node maintains local knowledge of neighbor capabilities without central coordination
- **Efficient Topology Collection**: Reduces unnecessary exchange rate requests by targeting known exchange-capable neighbors
- **Backward Compatibility**: Does not break existing topology collection for direct exchange scenarios
- **Scalable Architecture**: Notification overhead is proportional to actual exchange rate updates
- **Separation of Concerns**: Local exchange rates (`addOrUpdate()`) vs external rates (`addOrUpdateExternal()`) handled separately

### Negative
- **Increased Message Overhead**: Additional messages sent when exchange rates are updated and during topology collection
- **Memory Overhead**: Each node must store neighbor exchange capability information
- **Complexity**: Additional logic in multiple transaction types
- **Potential Staleness**: Neighbor capability information may become stale if nodes disconnect without notification

### Risk Mitigation
- **Message Deduplication**: Implement mechanisms to avoid duplicate notifications and requests
- **Capability Expiry**: Add TTL to neighbor capability information with periodic refresh
- **Connection State Integration**: Clean up neighbor capabilities when connections are lost
- **Rate Limiting**: Implement rate limiting for capability notifications and requests to prevent abuse

## Implementation Details

### Transaction Classes

#### ReceiveExchangeCapabilityNotificationTransaction
```cpp
class ReceiveExchangeCapabilityNotificationTransaction : public BaseTransaction {
public:
    ReceiveExchangeCapabilityNotificationTransaction(
        ExchangeCapabilityNotificationMessage::Shared message,
        ExchangeRatesManager *exchangeRatesManager);
    
protected:
    TransactionResult::SharedConst run() override;

private:
    ExchangeCapabilityNotificationMessage::Shared mMessage;
    ExchangeRatesManager *mExchangeRatesManager;
};
```

#### ReceiveExchangeRateRequestTransaction
```cpp
class ReceiveExchangeRateRequestTransaction : public BaseTransaction {
public:
    ReceiveExchangeRateRequestTransaction(
        ExchangeRateRequestMessage::Shared message,
        ExchangeRatesManager *exchangeRatesManager,
        // other dependencies for sending messages...
    );
    
protected:
    TransactionResult::SharedConst run() override;

private:
    ExchangeRateRequestMessage::Shared mMessage;
    ExchangeRatesManager *mExchangeRatesManager;
    // other dependencies...
};
```

#### Enhanced Exchange Rate Setting Transaction
The existing transaction that handles `addOrUpdate()` must be enhanced with:
```cpp
class SetExchangeRateTransaction : public BaseTransaction {
public:
    SetExchangeRateTransaction(
        // existing parameters...
        EquivalentsSubsystemsRouter *equivalentsSubsystemsRouter);

private:
    void notifyNeighborsOfExchangeCapabilities(
        const SerializedEquivalent equivFrom,
        const SerializedEquivalent equivTo);
    
    vector<ContractorID> getNeighborsWithTrustLines(
        const SerializedEquivalent equivalent) const;

private:
    EquivalentsSubsystemsRouter *mEquivalentsSubsystemsRouter;
    // existing members...
};
```

### Message Processing Integration
```cpp
// In TailManager or similar message routing component
void routeExchangeCapabilityMessage(ExchangeCapabilityNotificationMessage::Shared message) {
    auto transaction = make_shared<ReceiveExchangeCapabilityNotificationTransaction>(
        message, mExchangeRatesManager);
    mScheduler->scheduleTransaction(transaction);
}

void routeExchangeRateRequestMessage(ExchangeRateRequestMessage::Shared message) {
    auto transaction = make_shared<ReceiveExchangeRateRequestTransaction>(
        message, mExchangeRatesManager, /* other dependencies */);
    mScheduler->scheduleTransaction(transaction);
}
```

## Testing Strategy

### Unit Tests
1. Test `ExchangeRatesManager` neighbor capability tracking
2. Test message serialization/deserialization for both new message types
3. Test notification triggering logic in exchange rate setting transaction
4. Test exchange rate request logic in topology collection transactions

### Integration Tests
1. Test end-to-end capability discovery between connected nodes
2. Test topology collection with intermediate exchange nodes using `ExchangeRateRequestMessage`
3. Test cleanup of stale neighbor information
4. Test that `addOrUpdateExternal()` does not trigger notifications

### Performance Tests
1. Measure notification overhead under various network conditions
2. Test memory usage with large numbers of neighbors and exchange pairs
3. Validate path discovery performance improvements
4. Measure exchange rate request overhead during topology collection

## Future Enhancements

### Dynamic Capability Updates
- Implement periodic capability refresh to handle stale information
- Add capability versioning for conflict resolution

### Advanced Path Optimization
- Use neighbor capability information for proactive path pre-calculation
- Implement capability-aware load balancing across multiple exchange nodes

### Security Considerations
- Add signature verification for capability notifications and requests
- Implement capability announcement rate limiting
- Consider privacy implications of capability disclosure

## Related Decisions
- Topology Collection Protocol (topology-collection-protocol.md)
- Exchange Rate Management Architecture
- Network Message Routing Architecture

## Validation Criteria

This decision will be considered successful when:
1. Nodes can successfully discover payment paths through intermediate exchange nodes in 7-node scenarios
2. Topology collection includes exchange rates from nodes not directly participating in collection transactions via `ExchangeRateRequestMessage`
3. Path calculation performance improves due to more complete topology information
4. System maintains backward compatibility with existing single-equivalent and direct exchange scenarios
5. Message overhead remains within acceptable bounds (< 15% increase in typical scenarios)
6. Exchange rate notifications are triggered only by `addOrUpdate()` and not by `addOrUpdateExternal()`
