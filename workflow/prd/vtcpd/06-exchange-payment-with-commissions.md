# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Exchange Payment with Commissions and Multi-Equivalent Support
- **PRD ID**: 06
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.1
- **Date**: 2025-10-06
- **Author(s)**: Claude Code, based on Architect's requirements
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-10-06
- **Previous PRD**: [05-payment-estimation.md](05-payment-estimation.md)
- **Related Documents**:
  - [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
  - [Exchange Flow Calculation PRD](04-exchange-flow-calculation.md)
  - [Payment Estimation PRD](05-payment-estimation.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This PRD introduces multi-equivalent payment execution with exchange rate support and transit commissions. Building upon the Exchange Flow Calculation (PRD 04) and Payment Estimation (PRD 05), this feature enables actual payment execution across different equivalents using optimal exchange paths with proper commission handling.

### Current project state
- Exchange Flow Calculation (PRD 04) computes optimal cross-equivalent payment paths using OR-Tools with commission support
- Payment Estimation (PRD 05) provides bidirectional estimation using cached optimal paths
- Current payment transactions (BasePaymentTransaction, CoordinatorPaymentTransaction, ReceiverPaymentTransaction, IntermediateNodePaymentTransaction) operate only within single equivalent

### This iteration's focus
- Create exchange-aware payment transaction classes using EquivalentsSubsystemsRouter
- Extend reservation protocol to support multiple equivalents per payment
- Implement multi-equivalent reservation validation with exchange rates and commissions
- Support multiple payment receipts per trust line (one per equivalent)
- Integrate ExchangePathsManager for optimal path retrieval

### Connection to overall vision
This completes the multi-equivalent payment capability in the vTCP network, enabling users to pay in one equivalent while receivers accept another, with automatic exchange routing and commission handling.

## Iteration Context
### Previous Iterations Summary
- **PRD 03**: Exchange Rates Manager provides exchange rate storage with TTL management
- **PRD 04**: Exchange Flow Calculation implements cross-equivalent max flow computation using OR-Tools with optimal path enumeration and commission handling
- **PRD 05**: Payment Estimation provides bidirectional estimation using cached optimal paths
- **Completed Features**: Topology collection, max flow calculation, optimal path computation with commissions, payment estimation

### Lessons Learned
- EquivalentsSubsystemsRouter provides unified access to per-equivalent subsystems
- Commission "charge once" semantics are critical for accurate flow simulation
- Exchange rate limits must be validated during reservation
- Path-based reservation tracking enables multi-equivalent coordination

### Current State Analysis
- **What's working well**: Single-equivalent payment execution is stable and tested
- **Pain points identified**: No support for cross-equivalent payments; separate transaction classes per equivalent limit flexibility
- **Performance metrics**: Single-equivalent payments execute within acceptable timeframes

## Problem Statement
### Background
The current payment system supports only single-equivalent transactions. With exchange rates and optimal path calculation now available (PRD 03, 04), users need the ability to execute payments where:
1. Sender pays in one equivalent
2. Receiver accepts in another equivalent
3. Intermediate nodes may perform exchanges
4. Transit commissions are charged correctly

However, current payment transaction classes (CoordinatorPaymentTransaction, ReceiverPaymentTransaction, IntermediateNodePaymentTransaction) are designed for single-equivalent operations with direct manager access, making multi-equivalent support challenging.

### Problem Description
**Who is affected**: All network participants wanting to execute cross-equivalent payments

**When and where**: During payment execution phase after path calculation

**Current limitations**:
- Payment transactions use direct manager references (TrustLinesManager, TopologyCacheManager, etc.) instead of EquivalentsSubsystemsRouter
- Reservations lack equivalent information, preventing multi-equivalent tracking
- Message protocol doesn't include reservation equivalents
- Single receipt per trust line limits multi-equivalent payments between same nodes
- No exchange rate or commission validation during reservation

### Impact of not solving this problem
- Exchange rate and path calculation infrastructure (PRD 03, 04, 05) remains unused for actual payments
- Users cannot benefit from optimal exchange routing
- Network utility limited to single-equivalent operations

### Success Metrics
**Primary KPIs**:
- Successful multi-equivalent payment execution using optimal paths
- Correct exchange rate application during payment
- Accurate commission deduction with "charge once" semantics
- Proper validation of exchange limits and reservation directions

**Target Values**:
- 100% accuracy in multi-equivalent payment execution matching estimated flows
- Zero payment failures due to incorrect exchange or commission handling
- Support for up to 5 exchange equivalents per payment (as per PRD 04 limit)

## Goals
The primary goals of this iteration are to enable multi-equivalent payment execution with exchange and commission support.

*   **Goal 1: Enable multi-equivalent payment execution.**
    *   **Description:** Allow payment execution where sender and receiver use different equivalents, with automatic exchange routing via optimal paths.
    *   **Success Metric:** Payments execute successfully across different equivalents with correct amount delivery to receiver.

*   **Goal 2: Ensure accurate exchange and commission handling.**
    *   **Description:** Apply exchange rates and transit commissions correctly during payment, matching estimation results from PRD 05.
    *   **Success Metric:** Actual delivered amounts match estimated amounts within rounding precision; commissions charged exactly once per (node, equivalent) pair.

*   **Goal 3: Support multiple receipts per trust line.**
    *   **Description:** Enable multiple simultaneous reservations on same trust line in different equivalents.
    *   **Success Metric:** Nodes can maintain separate debt records per equivalent on same trust line.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **BaseExchangePaymentTransaction**: Base class using EquivalentsSubsystemsRouter instead of direct manager access
2. **CoordinatorExchangePaymentTransaction**: Multi-equivalent payment coordinator using ExchangePathsManager
3. **ReceiverExchangePaymentTransaction**: Receiver handling multi-equivalent reservations
4. **IntermediateNodeExchangePaymentTransaction**: Intermediate node with exchange and commission validation
5. **Multi-equivalent reservation protocol**: Extended messages with reservation equivalents
6. **Multiple receipt support**: Vector of (equivalent, signature) pairs in messages
7. **Exchange command**: New command accepting exchangeEquivalents parameter

#### Technical Infrastructure
- New transaction base class with EquivalentsSubsystemsRouter integration
- Enhanced reservation messages with equivalent tracking
- Extended AmountReservation with SerializedEquivalent field
- PathReservation structure for (PathID, Amount, Equivalent) tuples
- Enhanced ExchangePath with both ContractorID and BaseAddress vectors

#### Integration Points
- Integration with ExchangePathsManager for optimal path retrieval
- Usage of ExchangeRatesManager for exchange rate validation
- Usage of CommissionsManager for commission lookup
- Extension of existing message protocol with equivalent fields

### Explicitly Out of Scope
- Automatic path invalidation after payment (noted as future work in PRD 05)
- Performance optimization for large path sets
- Cryptographic validation of exchange rates
- UI/API for payment monitoring
- Migration of existing single-equivalent transactions (will be removed after testing)

### Dependencies from Previous Iterations
- **PRD 03**: ExchangeRatesManager for exchange rate validation
- **PRD 04**: ExchangePathsManager and optimal path calculation infrastructure
- **PRD 05**: ExchangePathsManager path caching and retrieval
- **Existing payment infrastructure**: Message protocol, reservation system, voting mechanism

### Future Roadmap Impact
This iteration establishes foundation for:
- **Automatic path invalidation**: Integration with ExchangePathsManager for cache updates after payments
- **Advanced payment strategies**: Multi-path payment optimization, dynamic routing
- **Payment analytics**: Cross-equivalent payment tracking and analysis
- **Exchange arbitrage detection**: Monitoring for beneficial exchange cycles

## User Stories & Requirements

### User Personas
#### Primary User: Payment Initiator
- **Role**: Node operator initiating cross-equivalent payments
- **Goals**: Execute payments in available equivalent while receiver gets desired equivalent
- **Pain Points**: Cannot execute cross-equivalent payments; must hold exact equivalent receiver wants
- **Technical Proficiency**: Advanced

#### Secondary User: Payment Receiver
- **Role**: Node receiving payments
- **Goals**: Receive payments in preferred equivalent regardless of sender's equivalent
- **Pain Points**: Limited to accepting equivalents sender has available
- **Technical Proficiency**: Intermediate

#### Tertiary User: Intermediate Node Operator
- **Role**: Node providing exchange and transit services
- **Goals**: Earn commissions for exchange services and payment routing
- **Pain Points**: No mechanism to charge for exchange or transit services
- **Technical Proficiency**: Advanced

### Functional Requirements
#### New Features for This Iteration

1. **BaseExchangePaymentTransaction**
   - **Description**: Base payment transaction class using EquivalentsSubsystemsRouter for multi-equivalent support
   - **User Story**: As a payment system, I need a base transaction class that can access per-equivalent managers for multi-equivalent operations
   - **Rationale**: Current BasePaymentTransaction uses direct manager references; EquivalentsSubsystemsRouter enables dynamic equivalent-specific access
   - **Builds Upon**: Existing BasePaymentTransaction structure
   - **Acceptance Criteria**:
     - Constructor accepts EquivalentsSubsystemsRouter instead of individual managers (iAmGateway, TrustLinesManager, TopologyCacheManager, MaxFlowCacheManager)
     - Methods accept SerializedEquivalent parameter where needed to retrieve correct manager
     - Located in same directory as BasePaymentTransaction
     - Inherits core payment logic while enabling multi-equivalent support
   - **Priority**: High
   - **Dependencies**: EquivalentsSubsystemsRouter

2. **CoordinatorExchangePaymentTransaction**
   - **Description**: Payment coordinator supporting multi-equivalent execution using ExchangePathsManager
   - **User Story**: As a payment coordinator, I need to execute payments using optimal exchange paths with proper commission handling
   - **Rationale**: Enables actual payment execution using infrastructure from PRD 04 and 05
   - **Builds Upon**: CoordinatorPaymentTransaction pattern, ExchangePathsManager
   - **Acceptance Criteria**:
     - Inherits from BaseExchangePaymentTransaction
     - Uses ExchangePathsManager instead of PathsManager
     - Retrieves optimal paths from ExchangePathsManager (no PathsResource usage)
     - Supports exchangeEquivalents from command (via CreditUsageExchangeCommand)
     - Implements path reservation with equivalent tracking
     - Handles multiple receipts per neighbor (one per equivalent)
     - Located alongside CoordinatorPaymentTransaction
   - **Priority**: High
   - **Dependencies**: BaseExchangePaymentTransaction, ExchangePathsManager, CreditUsageExchangeCommand

3. **ReceiverExchangePaymentTransaction and IntermediateNodeExchangePaymentTransaction**
   - **Description**: Receiver and intermediate node handling multi-equivalent reservations with exchange validation
   - **User Story**: As a receiver/intermediate node, I need to validate and process reservations across multiple equivalents with proper exchange rate checking
   - **Rationale**: Completes payment execution path with exchange and commission support
   - **Builds Upon**: ReceiverPaymentTransaction, IntermediateNodePaymentTransaction patterns
   - **Acceptance Criteria**:
     - Inherit from BaseExchangePaymentTransaction
     - Validate reservations with equivalent information
     - Implement checkReservationsDirections() with multi-equivalent logic:
       - All outgoing reservations in single equivalent
       - Incoming reservations in multiple equivalents allowed
       - Convert incoming amounts to outgoing equivalent using ExchangeRatesManager
       - Deduct commission once per incoming equivalent if same as outgoing (transit only, not exchange)
       - Return false if exchange rate not found or sums don't match
     - Support multiple receipts (one per equivalent)
     - Located alongside respective payment transactions
   - **Priority**: High
   - **Dependencies**: BaseExchangePaymentTransaction, ExchangeRatesManager, CommissionsManager

4. **Enhanced Reservation Protocol**
   - **Description**: Extend reservation messages and data structures with equivalent information
   - **User Story**: As a payment system, I need to track which equivalent each reservation uses for proper validation
   - **Rationale**: Enables multi-equivalent reservation tracking and validation
   - **Builds Upon**: Existing RequestMessageWithReservations and AmountReservation
   - **Acceptance Criteria**:
     - AmountReservation extended with SerializedEquivalent field, getter, and updated constructor
     - AmountReservationsHandler methods accept SerializedEquivalent parameter (reserve, updateReservation, free, totalReserved, getReservation)
     - RequestMessageWithReservations and inheritors serialize/deserialize reservation equivalents
     - Two constructors per message class: with equivalents (new) and without (deprecated, for compatibility)
     - BasePaymentTransaction methods use mEquivalent for all operations (no multi-equivalent in old transactions)
     - New transactions use appropriate equivalents per reservation
   - **Priority**: High
   - **Dependencies**: None (refactoring of existing classes)

5. **Multiple Receipt Support**
   - **Description**: Support multiple payment receipts per trust line (one per equivalent)
   - **User Story**: As a payment system, I need to track separate debt records per equivalent on same trust line
   - **Rationale**: Single receipt limitation prevents multi-equivalent payments between same nodes
   - **Builds Upon**: Existing receipt mechanism in FinalAmountsConfigurationMessage and TransactionPublicKeyHashMessage
   - **Acceptance Criteria**:
     - FinalAmountsConfigurationMessage: replace mIsReceiptContains (bool) and mSignature with vector<pair<SerializedEquivalent, Signature>>
     - TransactionPublicKeyHashMessage: replace mIsReceiptContains (bool) and mSignature with vector<pair<SerializedEquivalent, Signature>>
     - isReceiptContains() returns true if vector has at least one element
     - Old transactions create single-element vector with mEquivalent
     - New transactions populate vector with all relevant equivalents
     - Serialize/deserialize vector in both messages
   - **Priority**: High
   - **Dependencies**: Enhanced reservation protocol

6. **CreditUsageExchangeCommand**
   - **Description**: New command for initiating multi-equivalent payments
   - **User Story**: As a user, I need a command to specify which equivalents I can pay with and which equivalent receiver wants
   - **Rationale**: Extends CreditUsageCommand with exchangeEquivalents parameter
   - **Builds Upon**: CreditUsageCommand structure
   - **Acceptance Criteria**:
     - Analogous to CreditUsageCommand with added exchangeEquivalents parameter
     - Similar to InitiateMaxFlowExchangeCalculationCommand structure
     - Passed to CoordinatorExchangePaymentTransaction constructor
     - Located alongside CreditUsageCommand
   - **Priority**: High
   - **Dependencies**: None

#### Enhancements to Existing Features

1. **OptimalPathResult Enhancement**
   - **Current State**: Contains basic path information from OR-Tools optimization
   - **Proposed Changes**:
     - Add fields from PathStats: `mMaxPathFlow`, `mIsValid`, `mIntermediateNodesStates` (vector), `NodeState` enum
     - Add all methods from PathStats for path state management
     - Note: Do NOT add `mPath` (Path::Shared) field; existing `ExchangePath path` serves this role
   - **Impact Assessment**: Enables path state tracking during reservation phase
   - **Migration Strategy**: Direct field addition; no data migration needed

2. **ExchangePath Enhancement**
   - **Current State**: Contains `vector<ContractorID> nodes`
   - **Proposed Changes**:
     - Rename `nodes` field to `ids` (vector<ContractorID>)
     - Add `nodes` field (vector<BaseAddress::Shared>) - same as Path class
     - Add all methods from Path class
     - Conversion ContractorID → BaseAddress happens once during mPathsStats population in CoordinatorExchangePaymentTransaction via ContractorsManager
   - **Impact Assessment**: Enables compatibility with existing reservation code expecting BaseAddress
   - **Migration Strategy**: Field rename and addition; update path construction logic

3. **PathReservation Structure**
   - **Current State**: Using pair<PathID, ConstSharedTrustLineAmount> in mNodesFinalAmountsConfiguration
   - **Proposed Changes**:
     - Create PathReservation structure:
       ```cpp
       struct PathReservation {
           PathID pathID;
           ConstSharedTrustLineAmount amount;
           SerializedEquivalent equivalent;
       };
       ```
     - Replace all usages of pair<PathID, ConstSharedTrustLineAmount> with PathReservation
     - Update mNodesFinalAmountsConfiguration type
   - **Impact Assessment**: Improves code readability and supports multi-equivalent tracking
   - **Migration Strategy**: Structure introduction and type replacement

4. **BasePaymentTransaction Serialization**
   - **Current State**: serializeToBytes() and deserialization constructor don't include reservation equivalents
   - **Proposed Changes**:
     - Extend serializeToBytes() to serialize reservation equivalents
     - Update deserialization constructor to read reservation equivalents
     - Maintain backward compatibility (old transactions use mEquivalent for all reservations)
   - **Impact Assessment**: Enables transaction state persistence with multi-equivalent support
   - **Migration Strategy**: Extend serialization format; old data handled by deprecated constructors

5. **Equivalent-Aware Helper Methods**
   - **Current State**: totalReservedIncomingAmountToNode() and totalReservedAmount() don't accept equivalent parameter
   - **Proposed Changes**:
     - Add SerializedEquivalent parameter to both methods
     - Calculate totals only for specified equivalent
     - Update all call sites to provide equivalent
   - **Impact Assessment**: Enables per-equivalent reservation tracking
   - **Migration Strategy**: Method signature update; old transactions always pass mEquivalent

### Non-Functional Requirements
#### Performance
- Payment execution time increase < 20% compared to single-equivalent payments
- Exchange rate validation overhead < 50ms per payment
- Support for up to 10 exchange operations per payment path

#### Security
- Exchange rate validation uses current rates from ExchangeRatesManager
- Commission validation uses current values from CommissionsManager
- Receipt signatures per equivalent prevent cross-equivalent fraud

#### Scalability
- Support payments with up to 5 exchange equivalents (as per PRD 04 limit)
- Handle up to 50 optimal paths per payment
- Memory overhead < 100MB per concurrent multi-equivalent payment

#### Reliability
- Payment failures roll back all reservations correctly
- Exchange validation failures return appropriate error codes
- Commission calculation errors don't cause payment inconsistency

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Single-equivalent payment transactions with direct manager access
- **Proposed Changes**: Multi-equivalent payment transactions using EquivalentsSubsystemsRouter; ExchangePathsManager integration; enhanced reservation protocol
- **Backwards Compatibility**: Old transactions will be removed after testing; no migration needed
- **Migration Requirements**: None (new deployment only)

### Technology Stack Updates
#### New Technologies/Libraries
- No new external libraries required
- Reuses existing infrastructure: EquivalentsSubsystemsRouter, ExchangeRatesManager, CommissionsManager, ExchangePathsManager

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- CoordinatorExchangePaymentTransaction integrates with ExchangePathsManager
- All exchange transactions integrate with ExchangeRatesManager for validation
- Intermediate transactions integrate with CommissionsManager for commission lookup

#### Modified Integrations
- Payment messages extended with reservation equivalents
- AmountReservation and AmountReservationsHandler extended with equivalent tracking
- FinalAmountsConfigurationMessage and TransactionPublicKeyHashMessage support multiple receipts

### Data Requirements
#### Data Models

##### PathReservation Structure
**Purpose**: Represent reservation with equivalent information

**Fields**:
```cpp
struct PathReservation {
    PathID pathID;
    ConstSharedTrustLineAmount amount;
    SerializedEquivalent equivalent;
};
```

**Usage**: Replace pair<PathID, ConstSharedTrustLineAmount> in mNodesFinalAmountsConfiguration and related code

##### Enhanced AmountReservation
**Purpose**: Track reservation equivalent

**Changes**:
- Add field: `SerializedEquivalent mEquivalent`
- Add getter: `const SerializedEquivalent& equivalent() const`
- Update constructor: `AmountReservation(const TransactionUUID &transactionUUID, const TrustLineAmount &amount, const ReservationDirection direction, const SerializedEquivalent equivalent)`

##### Enhanced AmountReservationsHandler
**Purpose**: Manage reservations with equivalent awareness

**Changes**:
- Add SerializedEquivalent parameter to:
  - `reserve(..., SerializedEquivalent equivalent)`
  - `updateReservation(..., SerializedEquivalent equivalent)`
  - `free(..., SerializedEquivalent equivalent)`
  - `totalReserved(..., SerializedEquivalent equivalent)`
  - `getReservation(..., SerializedEquivalent equivalent)`
- All methods use equivalent when working with AmountReservation

##### Enhanced OptimalPathResult
**Purpose**: Track path state during reservation

**Added Fields** (from PathStats):
```cpp
TrustLineAmount mMaxPathFlow;
bool mIsValid;
vector<NodeState> mIntermediateNodesStates;
enum NodeState {
    ReservationRequestDoesntSent = 0,
    NeighbourReservationRequestSent,
    NeighbourReservationApproved,
    ReservationRequestSent,
    ReservationApproved,
    ReservationRejected
};
```

**Added Methods** (from PathStats):
- `void setNodeState(const SerializedPositionInPath positionInPath, const NodeState state)`
- `const TrustLineAmount &maxFlow() const`
- `void shortageMaxFlow(const TrustLineAmount &kAmount)`
- `const ExchangePath &path() const` (uses existing `path` field)
- `bool containsIntermediateNodes() const`
- `const pair<BaseAddress::Shared, SerializedPositionInPath> currentIntermediateNodeAndPos() const`
- `const pair<BaseAddress::Shared, SerializedPositionInPath> nextIntermediateNodeAndPos() const`
- `const bool reservationRequestSentToAllNodes() const`
- `const bool isNeighborAmountReserved() const`
- `const bool isWaitingForNeighborReservationResponse() const`
- `const bool isWaitingForNeighborReservationPropagationResponse() const`
- `const bool isWaitingForReservationResponse() const`
- `const bool isReadyToSendNextReservationRequest() const`
- `const bool isLastIntermediateNodeProcessed() const`
- `const bool isLastIntermediateNodeApproved() const`
- `const bool isValid() const`
- `void setUnusable()`

**Note**: Do NOT add `mPath` (Path::Shared) field; existing `ExchangePath path` replaces it

##### Enhanced ExchangePath
**Purpose**: Support both ContractorID and BaseAddress representations

**Changes**:
- Rename field: `nodes` → `ids` (vector<ContractorID>)
- Add field: `nodes` (vector<BaseAddress::Shared>)
- Add all methods from Path class
- Conversion: ContractorID → BaseAddress via ContractorsManager during mPathsStats population

##### Enhanced Message Classes

**FinalAmountsConfigurationMessage**:
- Remove: `bool mIsReceiptContains`, `sphincs::Signature::Shared mSignature`
- Add: `vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> mSignatures`
- Update: `bool isReceiptContains() const` returns `!mSignatures.empty()`
- Add: `const vector<pair<SerializedEquivalent, sphincs::Signature::Shared>>& signatures() const`
- Compatibility: Old transactions create single-element vector with `{mEquivalent, signature}`

**TransactionPublicKeyHashMessage**:
- Remove: `bool mIsReceiptContains`, `sphincs::Signature::Shared mSignature`
- Add: `vector<pair<SerializedEquivalent, sphincs::Signature::Shared>> mSignatures`
- Update: `bool isReceiptContains() const` returns `!mSignatures.empty()`
- Add: `const vector<pair<SerializedEquivalent, sphincs::Signature::Shared>>& signatures() const`
- Compatibility: Old transactions create single-element vector with `{mEquivalent, signature}`

**RequestMessageWithReservations and Inheritors**:
- Add reservation equivalents to serialization/deserialization
- Two constructor variants:
  - With equivalents: `RequestMessageWithReservations(..., const vector<PathReservation> &finalAmountsConfig)`
  - Without equivalents (deprecated): `RequestMessageWithReservations(..., const vector<pair<PathID, ConstSharedTrustLineAmount>> &finalAmountsConfig)` - uses transaction mEquivalent for all
- Both serialize/deserialize equivalents (deprecated uses mEquivalent for all)

#### Data Storage
- No new persistent storage requirements
- Transaction state serialization extended with reservation equivalents
- Receipt storage per equivalent (multiple receipts per trust line)

#### Data Migration
- No migration needed (new deployment; old transactions removed)

### Algorithm Specifications

#### runPathsResourceProcessingStage() in CoordinatorExchangePaymentTransaction
**Purpose**: Retrieve optimal paths from ExchangePathsManager and prepare for reservation

**Algorithm**:
```cpp
TransactionResult::SharedConst runPathsResourceProcessingStage() {
    // Step 1: Retrieve optimal paths from ExchangePathsManager
    // for each sender equivalent in exchangeEquivalents

    TrustLineAmount totalAddedFlow = TrustLineAmount(0);

    for (const auto& senderEquiv : mExchangeEquivalents) {
        PathCacheKey key{mContractorID, senderEquiv, mEquivalent};
        auto optimalPaths = mExchangePathsManager->retrievePaths(key);

        if (!optimalPaths) continue;

        // Step 2: Add paths until total flow >= payment amount
        for (const auto& pathResult : *optimalPaths) {
            if (totalAddedFlow >= mAmount) break;

            // Step 3: Add path to mPathsStats
            addPathForFurtherProcessing(pathResult);
            totalAddedFlow = totalAddedFlow + pathResult.received_amount;
        }

        if (totalAddedFlow >= mAmount) break;
    }

    // Step 4: Reduce timeout from maxNetworkDelay(10) to maxNetworkDelay(4)
    return resultWaitForMessageTypes(
        {Message::Payments_ReceiverInitPaymentResponse},
        maxNetworkDelay(4));
}
```

#### addPathForFurtherProcessing() Enhancement
**Purpose**: Add OptimalPathResult to mPathsStats with proper initialization

**Algorithm**:
```cpp
void addPathForFurtherProcessing(const OptimalPathResult& pathResult) {
    // Step 1: Initialize mIntermediateNodesStates
    size_t pathLength = pathResult.path.ids.size();
    pathResult.mIntermediateNodesStates.resize(pathLength, NodeState::ReservationRequestDoesntSent);

    // Step 2: Initialize nodes vector in ExchangePath
    pathResult.path.nodes.clear();
    for (const auto& contractorID : pathResult.path.ids) {
        auto contractor = mContractorsManager->contractor(contractorID);
        if (contractor) {
            pathResult.path.nodes.push_back(contractor->mainAddress());
        } else {
            throw ValueError("Contractor not found for ID: " + to_string(contractorID));
        }
    }

    // Step 3: Add to mPathsStats
    PathID pathID = generateNextPathID();
    mPathsStats[pathID] = make_unique<OptimalPathResult>(pathResult);
}
```

#### checkReservationsDirections() in Exchange Transactions
**Purpose**: Validate reservation direction balance with multi-equivalent support

**Algorithm**:
```cpp
bool checkReservationsDirections() const {
    // Step 1: Determine outgoing equivalent and validate uniformity
    SerializedEquivalent outgoingEquivalent;
    bool outgoingEquivalentSet = false;
    TrustLineAmount totalOutgoing = TrustLineAmount(0);

    for (const auto& [contractorID, reservations] : mReservations) {
        for (const auto& [pathID, reservation] : reservations) {
            if (reservation->direction() == AmountReservation::Outgoing) {
                if (!outgoingEquivalentSet) {
                    outgoingEquivalent = reservation->equivalent();
                    outgoingEquivalentSet = true;
                } else if (outgoingEquivalent != reservation->equivalent()) {
                    // All outgoing must be in same equivalent
                    return false;
                }
                totalOutgoing = totalOutgoing + reservation->amount();
            }
        }
    }

    if (!outgoingEquivalentSet) {
        // No outgoing reservations - invalid for intermediate/receiver
        return false;
    }

    // Step 2: Calculate total incoming converted to outgoing equivalent
    TrustLineAmount totalIncomingConverted = TrustLineAmount(0);
    set<SerializedEquivalent> processedIncomingEquivalents;

    for (const auto& [contractorID, reservations] : mReservations) {
        for (const auto& [pathID, reservation] : reservations) {
            if (reservation->direction() == AmountReservation::Incoming) {
                SerializedEquivalent incomingEquiv = reservation->equivalent();
                TrustLineAmount incomingAmount = reservation->amount();

                if (incomingEquiv == outgoingEquivalent) {
                    // Same equivalent - direct add
                    totalIncomingConverted = totalIncomingConverted + incomingAmount;

                    // Step 3: Deduct commission if same equivalent (transit only)
                    if (processedIncomingEquivalents.find(incomingEquiv) == processedIncomingEquivalents.end()) {
                        auto commission = mCommissionsManager->get(incomingEquiv);
                        if (commission) {
                            totalIncomingConverted = totalIncomingConverted - commission->amount();
                        }
                        processedIncomingEquivalents.insert(incomingEquiv);
                    }
                } else {
                    // Different equivalent - convert
                    auto rate = mExchangeRatesManager->get(incomingEquiv, outgoingEquivalent);
                    if (!rate) {
                        // No exchange rate found
                        return false;
                    }

                    try {
                        TrustLineAmount converted = mExchangeRatesManager->calculateConvertedAmount(
                            incomingEquiv, outgoingEquivalent, incomingAmount);
                        totalIncomingConverted = totalIncomingConverted + converted;
                    } catch (const Exception&) {
                        // Conversion failed (overflow or other error)
                        return false;
                    }

                    // No commission for exchange operations
                }
            }
        }
    }

    // Step 4: Compare totals
    return totalIncomingConverted == totalOutgoing;
}
```

**Key Points**:
- All **outgoing** reservations must be in **one** equivalent
- **Incoming** reservations can be in **multiple** equivalents
- Each incoming reservation converted to outgoing equivalent
- Commission deducted **once per incoming equivalent** only if **same as outgoing** (transit, not exchange)
- No exchange rate → validation fails
- Sums must match exactly

### Error Handling Specifications

#### Error Conditions
1. **No optimal paths in ExchangePathsManager**: Transaction fails with appropriate error code
2. **Insufficient paths to cover payment amount**: Transaction fails after attempting all available paths
3. **Exchange rate not found during validation**: checkReservationsDirections() returns false, transaction rejected
4. **Commission calculation error**: Transaction fails with error
5. **Reservation direction mismatch**: Validation fails, transaction rejected
6. **ContractorID → BaseAddress conversion failure**: Transaction fails with ValueError

## Implementation Plan
### This Iteration Timeline
- **Duration**: 6-8 weeks implementation + 2-3 weeks testing
- **Sprint Breakdown**:
  - Sprint 1 (Week 1-2): Data structure enhancements (OptimalPathResult, ExchangePath, PathReservation, AmountReservation)
  - Sprint 2 (Week 2-3): BaseExchangePaymentTransaction and message protocol extensions
  - Sprint 3 (Week 4-5): CoordinatorExchangePaymentTransaction with ExchangePathsManager integration
  - Sprint 4 (Week 5-6): ReceiverExchangePaymentTransaction and IntermediateNodeExchangePaymentTransaction
  - Sprint 5 (Week 7-8): Integration testing and validation
  - Sprint 6 (Week 9-10): Unit testing and edge case validation

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Data Structures Enhanced | Week 2 | OptimalPathResult, ExchangePath, PathReservation, AmountReservation updated | None | Low |
| BaseExchangePaymentTransaction Complete | Week 3 | Base class with EquivalentsSubsystemsRouter | Data structures | Medium |
| Message Protocol Extended | Week 3 | All messages support reservation equivalents and multiple receipts | Data structures | Medium |
| CoordinatorExchangePaymentTransaction Complete | Week 5 | Payment coordinator with ExchangePathsManager | All previous | High |
| Receiver/Intermediate Transactions Complete | Week 6 | Multi-equivalent validation implemented | All previous | High |
| Integration Testing Complete | Week 8 | End-to-end multi-equivalent payments validated | All previous | Medium |
| Unit Testing Complete | Week 10 | All components unit tested | All previous | Low |

### Dependencies on Other Teams/Projects
- No external team dependencies identified

### Integration Points with Previous Work
- Builds directly upon ExchangePathsManager (PRD 05)
- Uses ExchangeRatesManager (PRD 03) for validation
- Uses CommissionsManager (PRD 04) for commission lookup
- Extends existing payment protocol and message infrastructure

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++ and payment systems experience
- **Developers**: 2 developers for implementation support
- **QA Engineers**: 1 engineer for integration and unit testing

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Multi-equivalent reservation validation complexity | High | Medium | Thorough unit testing with diverse equivalent combinations; extensive logging |
| ExchangePath nodes initialization errors | Medium | Low | Validate ContractorsManager data availability; comprehensive error handling |
| Message serialization compatibility issues | Medium | Low | Extensive serialization/deserialization tests; validate deprecated constructor behavior |
| OptimalPathResult state management bugs | High | Medium | Port PathStats tests to OptimalPathResult; validate state transitions |
| checkReservationsDirections() logic errors | High | Medium | Comprehensive unit tests with all equivalent combinations and commission scenarios |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Payment execution differs from estimation | High | Low | Validate against PRD 05 estimation results; ensure identical logic |
| Commission handling discrepancies | Medium | Low | Match PRD 04 commission semantics exactly; cross-validate with flow calculation |

## Testing Strategy
### Testing Approach (Unit-Only)
- All testing is unit-only (no integration/E2E tests)
- Tests are built and executed exclusively in `build-tests`
- No Docker usage; mock all external dependencies
- Provide mock data for both SQLite and PostgreSQL where applicable

#### Unit Tests: New Components

**1. BaseExchangePaymentTransaction Tests**:
- Constructor with EquivalentsSubsystemsRouter correctly initializes
- Methods retrieve correct managers for different equivalents via router
- SerializedEquivalent parameter properly used in equivalent-aware methods
- Serialization/deserialization with reservation equivalents works correctly
- Backward compatibility with deprecated constructors (uses mEquivalent for all)

**2. OptimalPathResult Enhancement Tests**:
- All fields from PathStats correctly added (mMaxPathFlow, mIsValid, mIntermediateNodesStates)
- NodeState enum properly defined and used
- All PathStats methods correctly implemented:
  - setNodeState() updates state at correct position
  - maxFlow() returns correct value
  - shortageMaxFlow() reduces flow correctly
  - path() returns ExchangePath reference (not Path::Shared)
  - containsIntermediateNodes() validates path structure
  - currentIntermediateNodeAndPos() returns correct node and position
  - nextIntermediateNodeAndPos() returns correct next node
  - reservationRequestSentToAllNodes() checks all states
  - isNeighborAmountReserved() validates neighbor state
  - isWaitingForNeighborReservationResponse() checks waiting state
  - isWaitingForNeighborReservationPropagationResponse() checks propagation state
  - isWaitingForReservationResponse() validates response wait
  - isReadyToSendNextReservationRequest() checks readiness
  - isLastIntermediateNodeProcessed() validates completion
  - isLastIntermediateNodeApproved() checks approval
  - isValid() returns mIsValid
  - setUnusable() sets mIsValid to false
- mIntermediateNodesStates properly initialized and managed
- No mPath (Path::Shared) field present

**3. ExchangePath Enhancement Tests**:
- Field rename: `nodes` → `ids` correctly applied
- New `nodes` field (vector<BaseAddress::Shared>) properly added
- All methods from Path class correctly implemented
- ContractorID → BaseAddress conversion via ContractorsManager:
  - Valid contractors convert successfully
  - Invalid contractor ID throws ValueError
  - Empty ids vector results in empty nodes vector
  - Conversion preserves order
- Both ids and nodes vectors maintain consistency

**4. PathReservation Structure Tests**:
- Structure correctly defined with pathID, amount, equivalent fields
- Construction with all three parameters works
- Used correctly in mNodesFinalAmountsConfiguration
- Replaces pair<PathID, ConstSharedTrustLineAmount> in all locations
- Accessor methods work correctly

**5. CoordinatorExchangePaymentTransaction Tests**:
- Inherits from BaseExchangePaymentTransaction
- Uses ExchangePathsManager instead of PathsManager
- runPathsResourceProcessingStage() correctly:
  - Retrieves optimal paths from ExchangePathsManager for each sender equivalent
  - Iterates through paths until totalAddedFlow >= mAmount
  - Calls addPathForFurtherProcessing() for each path
  - Breaks when sufficient flow accumulated
  - Uses maxNetworkDelay(4) instead of maxNetworkDelay(10)
- addPathForFurtherProcessing() correctly:
  - Initializes mIntermediateNodesStates with ReservationRequestDoesntSent
  - Converts ids to nodes via ContractorsManager
  - Adds OptimalPathResult to mPathsStats
- Handles exchangeEquivalents from CreditUsageExchangeCommand
- mPathsStats uses OptimalPathResult instead of PathStats
- Supports multiple receipts per neighbor (one per equivalent)

**6. CreditUsageExchangeCommand Tests**:
- Command structure analogous to CreditUsageCommand
- exchangeEquivalents parameter correctly added (similar to InitiateMaxFlowExchangeCalculationCommand)
- Passed to CoordinatorExchangePaymentTransaction constructor
- Parsing from command string works correctly

**7. ReceiverExchangePaymentTransaction and IntermediateNodeExchangePaymentTransaction Tests**:
- Inherit from BaseExchangePaymentTransaction
- updateReservations() handles reservations with equivalents:
  - Correctly updates reservations for matching (pathID, equivalent) pairs
  - Validates equivalent matches during update
  - Rejects updates with mismatched equivalents
- checkReservationsDirections() validation with multi-equivalent logic:
  - **Test 1: All outgoing in single equivalent**
    - Setup: outgoing in eq 1 only
    - Expected: outgoingEquivalent = 1, validation proceeds
  - **Test 2: Outgoing in multiple equivalents → fail**
    - Setup: outgoing in eq 1 and eq 2
    - Expected: return false immediately
  - **Test 3: Incoming same as outgoing, no commission**
    - Setup: incoming eq 1 (100), outgoing eq 1 (100), no commission configured
    - Expected: totals match, return true
  - **Test 4: Incoming same as outgoing, with commission**
    - Setup: incoming eq 1 (110), outgoing eq 1 (100), commission eq 1 = 10
    - Expected: totalIncomingConverted = 110 - 10 = 100, return true
  - **Test 5: Incoming different from outgoing, convert**
    - Setup: incoming eq 1 (100), outgoing eq 2 (200), rate 1→2 = 2.0
    - Expected: converted = 100 * 2.0 = 200, return true
  - **Test 6: Multiple incoming equivalents**
    - Setup: incoming eq 1 (100), incoming eq 2 (50), outgoing eq 3 (250)
    - Rates: 1→3 = 2.0, 2→3 = 3.0
    - Expected: 100*2.0 + 50*3.0 = 200 + 150 = 250, return true
  - **Test 7: Commission only for transit (same equivalent)**
    - Setup: incoming eq 1 (110), incoming eq 2 (100), outgoing eq 1 (200), commission eq 1 = 10
    - Rate: 2→1 = 2.0
    - Expected: (110 - 10) + 100*2.0 = 100 + 200 = 300 ≠ 200, return false
    - Corrected setup: incoming eq 1 (110), incoming eq 2 (50), outgoing eq 1 (200), commission eq 1 = 10, rate 2→1 = 2.0
    - Expected: (110 - 10) + 50*2.0 = 100 + 100 = 200, return true
  - **Test 8: Commission charged once per incoming equivalent**
    - Setup: two incoming reservations in eq 1 (60 each), outgoing eq 1 (110), commission eq 1 = 10
    - Expected: (60 + 60) - 10 = 110, return true (commission deducted only once)
  - **Test 9: No exchange rate found → fail**
    - Setup: incoming eq 1 (100), outgoing eq 2 (200), no rate 1→2
    - Expected: return false
  - **Test 10: Overflow during conversion → fail**
    - Setup: incoming with MAX_AMOUNT, outgoing with high rate causing overflow
    - Expected: catch exception, return false
  - **Test 11: Sums don't match → fail**
    - Setup: incoming eq 1 (100), outgoing eq 2 (201), rate 1→2 = 2.0
    - Expected: converted = 200 ≠ 201, return false

**8. AmountReservation Enhancement Tests**:
- SerializedEquivalent field correctly added
- Constructor with equivalent parameter works
- equivalent() getter returns correct value
- Existing methods unchanged (amount(), transactionUUID(), direction())

**9. AmountReservationsHandler Enhancement Tests**:
- reserve() with SerializedEquivalent parameter creates reservation with correct equivalent
- updateReservation() with SerializedEquivalent parameter updates correct reservation
- free() with SerializedEquivalent parameter removes correct reservation
- totalReserved() with SerializedEquivalent parameter sums only reservations in that equivalent
- getReservation() with SerializedEquivalent parameter retrieves correct reservation
- Multiple reservations with different equivalents to same contractor handled correctly
- Edge case: same amount, same direction, different equivalents are distinct reservations

**10. RequestMessageWithReservations and Inheritors Tests**:
- Serialization with reservation equivalents:
  - PathReservation vector correctly serialized with (pathID, amount, equivalent) for each
  - Deserialization reconstructs PathReservation vector correctly
- Two constructors:
  - With equivalents: uses provided PathReservation vector
  - Without equivalents (deprecated): converts pair vector to PathReservation using mEquivalent
- Backward compatibility:
  - Deprecated constructor creates PathReservation with mEquivalent for all reservations
  - Both constructors produce same serialization format
  - Deserialization works for both cases
- All inheritor classes (IntermediateNodeReservationRequestMessage, CoordinatorReservationRequestMessage, etc.) correctly handle equivalents

**11. FinalAmountsConfigurationMessage Tests**:
- mIsReceiptContains and mSignature removed
- vector<pair<SerializedEquivalent, Signature>> mSignatures correctly added
- isReceiptContains() returns:
  - true if mSignatures.size() > 0
  - false if mSignatures.empty()
- signatures() getter returns correct vector
- Serialization/deserialization of vector<pair<SerializedEquivalent, Signature>>:
  - Empty vector serialized/deserialized correctly
  - Single element vector works (old transaction compatibility)
  - Multiple element vector works (new multi-equivalent payments)
  - Order preserved during serialization/deserialization
- Backward compatibility:
  - Old transaction creates single-element vector with {mEquivalent, signature}
  - Serialization format handles both cases

**12. TransactionPublicKeyHashMessage Tests**:
- mIsReceiptContains and mSignature removed
- vector<pair<SerializedEquivalent, Signature>> mSignatures correctly added
- isReceiptContains() returns:
  - true if mSignatures.size() > 0
  - false if mSignatures.empty()
- signatures() getter returns correct vector
- Serialization/deserialization of vector<pair<SerializedEquivalent, Signature>>:
  - Empty vector (no receipts) serialized/deserialized correctly
  - Single element vector works (old transaction compatibility)
  - Multiple element vector works (multi-equivalent payments)
  - Order preserved
- Backward compatibility:
  - Old transaction creates single-element vector with {mEquivalent, signature} if receipt present
  - Empty vector if no receipt

**13. BasePaymentTransaction Serialization Tests**:
- serializeToBytes() includes reservation equivalents:
  - All reservations serialized with their equivalents
  - Serialization format correct
- Deserialization constructor reads reservation equivalents:
  - Reservations correctly reconstructed with equivalents
  - Legacy format (without equivalents) handled by using mEquivalent for all
- Round-trip serialization/deserialization preserves all data

**14. Equivalent-Aware Helper Methods Tests**:
- totalReservedIncomingAmountToNode(contractorID, equivalent):
  - Sums only incoming reservations to contractorID in specified equivalent
  - Ignores reservations in other equivalents
  - Returns zero if no reservations match
- totalReservedAmount(direction, equivalent):
  - Sums only reservations with specified direction and equivalent
  - Ignores other equivalents
  - Returns zero if no match
- All call sites updated to provide equivalent parameter

#### Regression Testing (Unit)
- Scope: Ensure new classes don't break existing patterns
- Verify BasePaymentTransaction, CoordinatorPaymentTransaction, ReceiverPaymentTransaction, IntermediateNodePaymentTransaction remain unchanged
- Validate old transactions still work (will be removed later, but must work during transition)

#### Execution in CI/Locally
- Build tests in `build-tests` and run the produced binaries
- All unit tests must pass before PRD completion

### Quality Gates
- All unit tests pass in `build-tests`
- Multi-equivalent payment execution matches estimation results (PRD 05)
- Commission "charge once" semantics validated in all test scenarios
- Exchange rate validation correctly rejects invalid conversions
- checkReservationsDirections() passes all 11 test scenarios
- No memory leaks in multi-equivalent payment execution
- Serialization/deserialization preserves all data including equivalents

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Major feature addition (replaces single-equivalent payments)
- **Rollout Strategy**: Full deployment with new transaction types; old transactions removed after testing
- **Rollback Plan**: Revert to previous version if critical issues found; no partial rollback (all-or-nothing)

### Database Migrations
- No database migrations required (new deployment)
- Transaction state serialization updated to include equivalents

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: Node operator guidance on multi-equivalent payment capabilities
- **Documentation Updates**:
  - Payment protocol documentation extended with multi-equivalent details
  - API documentation for new CreditUsageExchangeCommand
  - Exchange and commission handling documentation

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**:
  - Successful multi-equivalent payment execution (target: 100% success rate for valid paths)
  - Payment amounts match estimation (PRD 05) within rounding precision
  - Commission charged exactly once per (node, equivalent) pair in all payments
- **Leading Indicators**: Successful unit test execution, optimal path retrieval from ExchangePathsManager
- **Baseline Values**: Single-equivalent payment success rates and execution times
- **Target Values**:
  - 100% test pass rate
  - Payment execution time increase < 20% vs single-equivalent
  - Zero discrepancies between estimation and execution

### Monitoring Plan
- **New Dashboards/Alerts**: Multi-equivalent payment success rates, exchange validation failures, commission calculation errors
- **Enhanced Monitoring**: Extended logging for multi-equivalent payment execution, exchange rate application, commission deduction
- **A/B Testing**: Not applicable

### Review Schedule
- **Daily**: Development progress and blocker identification
- **Weekly**: Unit test results and algorithm validation
- **Post-Implementation Review**: Payment execution accuracy analysis, performance benchmarking

## Appendices
### Glossary
- **Exchange Payment**: Payment where sender and receiver use different equivalents
- **Transit Commission**: Fixed fee charged by intermediate node for routing payment in same equivalent (not for exchange)
- **Reservation Equivalent**: Equivalent in which a specific reservation is denominated
- **PathReservation**: Structure combining PathID, amount, and equivalent
- **Multiple Receipts**: Vector of (equivalent, signature) pairs enabling multiple debt records per trust line

### References
- [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
- [Exchange Flow Calculation PRD](04-exchange-flow-calculation.md)
- [Payment Estimation PRD](05-payment-estimation.md)
- [Exchange Rates Manager PRD](03-exchange-rates-manager.md)
- [Existing Payment Transaction Implementation](../../../src/core/transactions/transactions/regular/payments/)

### Detailed Component Specifications

#### New Transaction Classes

##### BaseExchangePaymentTransaction
- **Location**: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.h`
- **Inheritance**: Analog of BasePaymentTransaction
- **Key Differences**:
  - Constructor accepts `EquivalentsSubsystemsRouter*` instead of individual managers (iAmGateway, TrustLinesManager, TopologyCacheManager, MaxFlowCacheManager)
  - Methods accept `SerializedEquivalent` parameter where needed to retrieve correct manager
  - Supports multi-equivalent reservation logic
- **Responsibilities**:
  - Manage reservations with equivalent tracking
  - Provide base voting and approval logic for exchange payments
  - Handle transaction serialization with reservation equivalents

##### CoordinatorExchangePaymentTransaction
- **Location**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`
- **Inheritance**: Extends BaseExchangePaymentTransaction
- **Input**: CreditUsageExchangeCommand (includes exchangeEquivalents)
- **Key Components**:
  - Uses `ExchangePathsManager*` instead of `PathsManager*`
  - `mPathsStats` map: PathID → OptimalPathResult (replaces PathStats)
  - `mNodesFinalAmountsConfiguration` uses PathReservation structure
- **Key Methods**:
  - `runPathsResourceProcessingStage()`: Retrieve optimal paths from ExchangePathsManager
  - `addPathForFurtherProcessing(OptimalPathResult)`: Initialize path state and add to mPathsStats
  - Reservation coordination with multiple equivalents

##### ReceiverExchangePaymentTransaction
- **Location**: `src/core/transactions/transactions/regular/payments/ReceiverExchangePaymentTransaction.h`
- **Inheritance**: Extends BaseExchangePaymentTransaction
- **Responsibilities**:
  - Validate multi-equivalent reservations
  - Implement checkReservationsDirections() with exchange rate conversion
  - Handle multiple receipts from coordinator

##### IntermediateNodeExchangePaymentTransaction
- **Location**: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h`
- **Inheritance**: Extends BaseExchangePaymentTransaction
- **Responsibilities**:
  - Process multi-equivalent reservation requests
  - Validate reservation directions with exchange and commission
  - Forward reservations with proper equivalent tracking

#### New Command Class

##### CreditUsageExchangeCommand
- **Location**: `src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.h`
- **Inheritance**: Analogous to CreditUsageCommand
- **Additional Field**: `vector<SerializedEquivalent> exchangeEquivalents` (similar to InitiateMaxFlowExchangeCalculationCommand)
- **Usage**: Input to CoordinatorExchangePaymentTransaction constructor

**Command Format**:
```
CREATE:contractors/transactions/exchange:<contractor_addresses_count>:<address_type>:<address>:<amount>:<receiver_equivalent>:<exchange_equivalent_1>[:<exchange_equivalent_2>:...][:<payload>]
```

**Parameters**:
- `contractor_addresses_count`: Number of contractor addresses (typically 1)
- `address_type`: Type of address (e.g., `12` for IPv4 with port)
- `address`: Contractor address (receiver)
- `amount`: Payment amount to send
- `receiver_equivalent`: Equivalent in which receiver accepts funds
- `exchange_equivalent_1...N`: List of equivalents in which sender can pay (maximum 5)
- `payload` (optional): Additional metadata for the payment

**Example 1** (single exchange equivalent):
```
CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1
```
*"Send payment to 127.0.0.1:2003 with amount 1000, receiver wants equivalent 2, sender can pay with equivalent 1"*

**Example 2** (multiple exchange equivalents):
```
CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2004:500:3:1:2:5
```
*"Send payment to 127.0.0.1:2004 with amount 500, receiver wants equivalent 3, sender can pay with equivalents 1, 2, or 5"*

**Example 3** (with payload):
```
CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2005:2000:1:2:Invoice-12345
```
*"Send payment to 127.0.0.1:2005 with amount 2000, receiver wants equivalent 1, sender can pay with equivalent 2, with payload 'Invoice-12345'"*

**Response Format** (success):
```
201
<transaction_uuid>
```

**Response Format** (error):
```
<error_code>
```

**Error Codes**:
- `401`: Protocol error (invalid command format or parameters)
- `402`: Receiver address invalid or unreachable
- `404`: No optimal paths found for any exchange equivalent
- `462`: No cached paths available (must run max flow calculation first)

**Validation Rules**:
- `exchangeEquivalents` must contain at least 1 element
- `exchangeEquivalents` must not exceed 5 elements (error 401 if violated)
- `amount` must be positive
- `receiver_equivalent` must be valid SerializedEquivalent
- All `exchange_equivalent_N` must be valid SerializedEquivalent values
- `payload` length must not exceed 65535 characters

**Implementation Notes**:
- Parsing follows InitiateMaxFlowExchangeCalculationCommand pattern for exchangeEquivalents
- Limit validation (5 elements) happens after parsing, same as InitiateMaxFlowExchangeCalculationCommand
- responseOK() returns status code 201 with transactionUUID (same as CreditUsageCommand)

#### Data Structure Files

##### PathReservation
- **Location**: `src/core/transactions/transactions/regular/payments/base/PathReservation.h` (new file)
- **Definition**:
```cpp
struct PathReservation {
    PathID pathID;
    ConstSharedTrustLineAmount amount;
    SerializedEquivalent equivalent;
};
```

### Command Response Specification

#### CreditUsageExchangeCommand Response via ResultsInterface

**Success Response Format**:
```
201
<transaction_uuid>
```

**Components**:
- **Status Code**: `201` (Created)
- **Body**: Transaction UUID as string (e.g., `"550e8400-e29b-41d4-a716-446655440000"`)

**Example Success Response**:
```
201
550e8400-e29b-41d4-a716-446655440000
```

**Error Response Format**:
```
<error_code>
```

**Error Codes and Meanings**:

| Code | Meaning | Description | When Occurs |
|------|---------|-------------|-------------|
| `401` | Protocol Error | Invalid command format, parsing error, or constraint violation | - exchangeEquivalents > 5 elements<br>- Invalid address format<br>- Invalid equivalent value<br>- Payload too long (> 65535 chars) |
| `402` | Receiver Unreachable | Cannot establish connection to receiver | - Receiver address not reachable<br>- Network error during initialization |
| `404` | No Paths Found | No optimal paths available for payment | - ExchangePathsManager has no cached paths for any (contractor, exchange_equiv, receiver_equiv) combination<br>- All paths expired (TTL exceeded) |
| `412` | Insufficient Capacity | Paths exist but cannot deliver requested amount | - Total capacity of all paths < payment amount<br>- Reservations failed due to insufficient balance |
| `462` | No Cached Paths | Must run max flow calculation first | - ExchangePathsManager has no entry for requested contractor and equivalent combinations<br>- User must execute GET:contractors/transactions/max/exchange first |

**Implementation Details**:
- Response generated by `CoordinatorExchangePaymentTransaction::responseOK(string &transactionUUID)` method
- Follows same pattern as `CreditUsageCommand::responseOK()`
- Returns `CommandResult::SharedConst` with:
  - Command identifier: `"CREATE:contractors/transactions/exchange"`
  - Command UUID: from original command
  - Status code: `201` for success
  - Body: transactionUUID string

**Response Flow**:
```
User → CreditUsageExchangeCommand
      ↓
CoordinatorExchangePaymentTransaction created
      ↓
[Transaction execution with reservations, exchange, commissions]
      ↓
Success: responseOK(transactionUUID) → ResultsInterface → User
Error: resultProtocolError/resultForbidden/etc → ResultsInterface → User
```

**Example Usage Session**:

```bash
# Step 1: Calculate max flow with exchange
→ GET:contractors/transactions/max/exchange:1:12:127.0.0.1:2003:2:1
← 200
  150000

# Step 2: Estimate payment amount needed
→ GET:contractors/transactions/estimate/payment:12:127.0.0.1:2003:100000:2:1
← 200
  50000

# Step 3: Execute payment
→ CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:50000:2:1
← 201
  550e8400-e29b-41d4-a716-446655440000

# Payment successfully created with UUID: 550e8400-e29b-41d4-a716-446655440000
```

**Error Example**:
```bash
# Attempt payment without prior max flow calculation
→ CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1
← 462

# Error 462: No cached paths available, must run max flow calculation first
```

### API Documentation
Detailed API specifications will be provided in task-level documentation for all components listed above.

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-10-06 | Claude Code | Initial draft for exchange payment with commissions | Phase 1 |
| 1.1 | 2025-10-06 | Claude Code | Added CreditUsageExchangeCommand format, examples, and ResultsInterface response specification | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [05-payment-estimation.md](05-payment-estimation.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)
- **Payment Protocol**: [payment-protocol.md](../../../architecture/vtcpd/protocols/payment-protocol.md)
