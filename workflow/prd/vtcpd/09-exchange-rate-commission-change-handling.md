# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Exchange Rate and Commission Change Handling During Payment Execution
- **PRD ID**: 09
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.0
- **Date**: 2025-10-23
- **Author(s)**: Claude Code, based on Architect's requirements
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-10-23
- **Previous PRD**: [08-path-capacity-adjustment.md](08-path-capacity-adjustment.md)
- **Related Documents**:
  - [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
  - [Exchange Payment Topology Collection PRD](07-exchange-payment-topology-collection.md)
  - [Path Capacity Adjustment PRD](08-path-capacity-adjustment.md)
  - [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This PRD implements handling of exchange rate and commission changes that occur during payment execution on exchange payment paths. When a coordinator requests a reservation from an intermediate node, the node may have different exchange rates or commissions than what the coordinator expected based on cached path data. This feature enables the coordinator to detect these changes, invalidate affected paths, create new paths with updated conditions, and update the ExchangeRatesManager and CommissionsManager with the new values.

### Current project state
- Exchange Payment with Commissions (PRD 06) implements multi-equivalent payment execution with commission handling
- Exchange Payment Topology Collection (PRD 07) adds automatic path availability checking and collection triggering
- Path Capacity Adjustment (PRD 08) implements dynamic path capacity adjustment when intermediate nodes return reduced reservation amounts
- Current implementation assumes exchange rates and commissions remain constant during payment execution

### This iteration's focus
- Add exchange rate and commission validation in intermediate node reservation processing
- Implement RejectedDueConditionsChanged response type for condition mismatch
- Add coordinator logic to detect condition changes and invalidate affected paths
- Create new paths with updated exchange rates/commissions
- Update ExchangeRatesManager and CommissionsManager with changed conditions
- Store exchange rates and commissions in intermediate node transaction context
- Implement commission "charge once per payment" semantics for multi-path payments

### Connection to overall vision
This completes the robustness of exchange payment execution by handling dynamic changes in network conditions (exchange rates and commissions) during payment processing, ensuring payments can adapt to real-time changes in intermediate node pricing.

## Iteration Context
### Previous Iterations Summary
- **PRD 04**: Exchange Flow Calculation implements topology collection and OR-Tools-based max flow computation
- **PRD 05**: Payment Estimation provides bidirectional estimation using cached paths
- **PRD 06**: Exchange Payment with Commissions implements multi-equivalent payment execution with proper commission handling
- **PRD 07**: Exchange Payment Topology Collection adds automatic path collection triggering
- **PRD 08**: Path Capacity Adjustment implements dynamic capacity adjustment during reservation
- **Completed Features**: Exchange rate management, optimal path calculation, exchange payment execution, automatic topology collection, path capacity adjustment

### Lessons Learned
- Exchange rates and commissions can change between path calculation and payment execution
- Intermediate nodes are authoritative for their current exchange rates and commissions
- Coordinator must adapt to condition changes to successfully complete payments
- Commission "charge once" semantics are critical for accurate flow simulation (from PRD 06)

### Current State Analysis
- **What's working well**: Exchange payments execute successfully when conditions remain constant
- **Pain points identified**:
  - No handling of exchange rate changes during payment execution
  - No handling of commission changes during payment execution
  - Coordinator has no mechanism to detect and react to condition changes
  - Intermediate nodes don't validate reservation requests against current conditions
- **Performance metrics**: Exchange payments succeed when conditions match cached path data

## Problem Statement
### Background
During exchange payment execution, the coordinator builds paths based on cached topology data that includes exchange rates and commissions from ExchangeRatesManager and CommissionsManager. However, these conditions can change between path calculation and reservation:

1. **Exchange rates change**: An intermediate node updates its exchange rate for a currency pair
2. **Commissions change**: An intermediate node updates its commission for a specific equivalent
3. **Commissions introduced/removed**: A node starts charging or stops charging a commission

When these changes occur, the coordinator's reservation requests will contain amounts calculated based on outdated conditions. Intermediate nodes need to detect this mismatch and reject the request, allowing the coordinator to adapt.

Additionally, for multi-path payments where an intermediate node participates in multiple paths, the commission must be charged only once per payment (not per path), as specified in PRD 06.

### Problem Description
**Who is affected**: Coordinators and intermediate nodes processing exchange payments with changing conditions

**When and where**:
- Coordinator: In reservation request processing when receiving RejectedDueConditionsChanged responses
- Intermediate node: In `runCoordinatorRequestProcessingStage()` when validating reservation requests against current conditions

**Current limitations**:
- CoordinatorReservationRequestMessage doesn't include exchange rate or commission information
- Intermediate nodes don't validate reservation requests against current exchange rates/commissions
- No response type for condition change rejection
- Coordinator has no logic to handle condition changes
- No storage of exchange rates/commissions in intermediate node transaction context
- No mechanism to enforce "charge commission once per payment" for multi-path scenarios

### Impact of not solving this problem
- Payments fail when exchange rates or commissions change during execution
- No graceful recovery from condition changes
- Intermediate nodes may process reservations based on incorrect assumptions
- Multi-path payments may incorrectly charge commissions multiple times
- Network appears unreliable due to failures from outdated cached data

### Success Metrics
**Primary KPIs**:
- Successful payment adaptation when exchange rates change (target: 100% detection and handling)
- Successful payment adaptation when commissions change (target: 100% detection and handling)
- Correct commission charging in multi-path payments (target: 100% charge once per payment)
- Path invalidation and recreation accuracy (target: 100% correct flow recalculation)

**Target Values**:
- 100% detection of exchange rate mismatches at intermediate nodes
- 100% detection of commission mismatches at intermediate nodes
- 100% correct path invalidation and recreation by coordinator
- 100% correct ExchangeRatesManager/CommissionsManager updates
- Zero payment failures due to unhandled condition changes
- 100% correct commission application (once per payment per equivalent)

## Goals
The primary goals of this iteration are to enable robust handling of exchange rate and commission changes during payment execution.

*   **Goal 1: Enable exchange rate and commission validation at intermediate nodes.**
    *   **Description:** Intermediate nodes validate incoming reservation requests against current exchange rates and commissions, rejecting requests with outdated conditions.
    *   **Success Metric:** Intermediate nodes correctly detect 100% of exchange rate and commission mismatches.

*   **Goal 2: Implement coordinator adaptation to condition changes.**
    *   **Description:** Coordinators detect condition changes from intermediate node responses, invalidate affected paths, create new paths with updated conditions, and update managers.
    *   **Success Metric:** Coordinators successfully adapt to condition changes 100% of the time, creating valid replacement paths.

*   **Goal 3: Ensure commission "charge once per payment" semantics.**
    *   **Description:** Intermediate nodes charge commissions only once per payment per equivalent, regardless of how many paths they participate in.
    *   **Success Metric:** Commissions charged exactly once per payment per equivalent in 100% of multi-path payments.

*   **Goal 4: Implement transaction context for intermediate nodes.**
    *   **Description:** Intermediate nodes store exchange rates and commissions in transaction context for consistent validation across multiple paths.
    *   **Success Metric:** Transaction context correctly maintains exchange rates and commissions throughout payment lifecycle.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **Exchange Rate and Commission Validation in Intermediate Nodes**: Validate reservation requests against current conditions
2. **RejectedDueConditionsChanged Response Type**: New response type for condition mismatch
3. **Coordinator Condition Change Handling**: Detect condition changes, invalidate paths, create new paths, update managers
4. **Intermediate Node Transaction Context**: Store exchange rates and commissions for consistent validation
5. **Commission "Charge Once Per Payment" Logic**: Ensure commissions charged only once regardless of path count
6. **Path Recalculation with Updated Conditions**: Recalculate flows and received amounts for affected paths
7. **Outgoing/Incoming Reservation Validation**: Validate outgoing reservation matches incoming after applying conditions

#### Technical Infrastructure
- Enhanced CoordinatorReservationRequestMessage with exchange rate and commission fields
- New ResponseMessage::OperationState value: RejectedDueConditionsChanged
- Enhanced CoordinatorReservationResponseMessage with exchange rate and commission fields
- Transaction context storage in IntermediateNodeExchangePaymentTransaction
- Path invalidation and recreation logic in CoordinatorExchangePaymentTransaction
- Manager update logic for ExchangeRatesManager and CommissionsManager

#### Integration Points
- Integration with ExchangeRatesManager for rate validation and updates
- Integration with CommissionsManager for commission validation and updates
- Extension of message protocol with condition information
- Enhancement of path processing logic in coordinator

### Explicitly Out of Scope
- Automatic detection of condition changes outside payment context
- Proactive notification of condition changes to cached paths
- Historical tracking of condition changes
- UI/API for condition change monitoring
- Predictive path recalculation based on condition change patterns

### Dependencies from Previous Iterations
- **PRD 06**: CoordinatorExchangePaymentTransaction and IntermediateNodeExchangePaymentTransaction implementations
- **PRD 06**: ExchangeRatesManager and CommissionsManager infrastructure
- **PRD 06**: OptimalPathResult structure with flow calculation capabilities
- **PRD 08**: Path capacity adjustment infrastructure (shortageReservationsOnPath, path invalidation)

### Future Roadmap Impact
This iteration establishes foundation for:
- **Proactive condition change notification**: Nodes broadcasting rate/commission changes to network
- **Adaptive path caching**: ExchangePathsManager invalidating paths when conditions change
- **Condition change analytics**: Monitoring and analysis of rate/commission volatility
- **Smart path selection**: Preferring paths with stable conditions

## User Stories & Requirements

### User Personas
#### Primary User: Exchange Payment Coordinator
- **Role**: Node operator coordinating multi-equivalent payments
- **Goals**: Successfully complete payments even when exchange rates or commissions change during execution
- **Pain Points**: Payments fail due to outdated cached conditions; no visibility into condition changes
- **Technical Proficiency**: Advanced

#### Secondary User: Intermediate Node Operator
- **Role**: Network participant providing exchange and routing services
- **Goals**: Enforce current exchange rates and commissions; avoid processing reservations based on incorrect assumptions
- **Pain Points**: No mechanism to reject reservations with outdated conditions; commission logic unclear for multi-path payments
- **Technical Proficiency**: Advanced

### Functional Requirements
#### New Features for This Iteration

1. **Enhanced CoordinatorReservationRequestMessage with Conditions**
   - **Description**: Add exchange rate and commission fields to reservation request message
   - **User Story**: As a coordinator, I need to send my expected exchange rate or commission with reservation requests so intermediate nodes can validate
   - **Rationale**: Enables intermediate nodes to detect condition mismatches
   - **Builds Upon**: Existing CoordinatorReservationRequestMessage structure
   - **Acceptance Criteria**:
     - Add optional fields to CoordinatorReservationRequestMessage:
       - `optional<pair<TrustLineAmount, int16_t>> mExpectedExchangeRate` (stores only rate and shift)
       - `optional<TrustLineAmount> mExpectedCommission`
     - Add two new constructors (in addition to default):
       - Constructor with expected exchange rate: takes rate and shift as parameters
       - Constructor with expected commission: takes commission amount as parameter
     - Coordinator creates message based on cached path data:
       - If node is exchanger on path:
         - Retrieve full ExchangeRate from ExchangeRatesManager/path data
         - Use constructor with exchange rate parameters
       - If node has commission on path:
         - Retrieve commission amount from CommissionsManager/path data
         - Use constructor with commission parameter
       - Otherwise: use default constructor (both fields empty/nullopt)
     - Fields cannot both be populated (exchange and commission are mutually exclusive)
     - **Intermediate node validation**: If both fields are populated in received message, intermediate node must reject with Rejected status (protocol violation)
     - Message serialization/deserialization handles optional fields as pair<rate, shift>
     - Methods `askRemoteNodeToApproveReservation()` and `askNeighborToApproveFurtherNodeReservation()` populate fields from path data
   - **Priority**: High
   - **Dependencies**: ExchangePath structure with exchange and commission information

2. **Intermediate Node Condition Validation**
   - **Description**: Validate reservation requests against current exchange rates and commissions
   - **User Story**: As an intermediate node, I need to validate reservation requests against my current exchange rates and commissions to ensure accurate processing
   - **Rationale**: Prevents processing reservations based on incorrect assumptions
   - **Builds Upon**: IntermediateNodeExchangePaymentTransaction reservation processing
   - **Acceptance Criteria**:
     - In `runCoordinatorRequestProcessingStage()`:
       - **First validation**: Check if both expectedExchangeRate and expectedCommission are populated
         - If both present: reject with Rejected status (protocol violation by coordinator)
         - Log error: "Protocol violation: both exchange rate and commission provided in request"
         - Return immediately without further processing
       - Extract expected exchange rate or commission from request message (as pair<rate, shift>)
       - If expected exchange rate provided:
         - **Determine equivalent pair**:
           - equivalentFrom = equivalent of incoming reservation on this pathID (already created in transaction)
           - equivalentTo = equivalent of outgoing reservation being requested (from message/reservation context)
         - Look up current exchange rate from ExchangeRatesManager using (equivalentFrom, equivalentTo)
         - Compare received pair<rate, shift> with ExchangeRate.exchangeRate() and ExchangeRate.exchangeRateShift()
         - If current rate doesn't match expected: reject with RejectedDueConditionsChanged
         - Create rejection response using constructor with actual rate and shift parameters
       - If expected commission provided:
         - Look up current commission from CommissionsManager for equivalent
         - If current commission doesn't match expected: reject with RejectedDueConditionsChanged
         - Include current commission in rejection response
       - If neither provided but current node has commission/exchange: include in rejection response
       - If request had rate/commission but node no longer has it: send response without rate/commission
       - After validation passes: store rate/commission in transaction context (first time only)
       - Drop incoming reservation on path after rejection
   - **Priority**: High
   - **Dependencies**: ExchangeRatesManager, CommissionsManager, transaction context storage

3. **RejectedDueConditionsChanged Response Type**
   - **Description**: New rejection reason for exchange rate or commission mismatch
   - **User Story**: As an intermediate node, I need a specific rejection type for condition changes to inform the coordinator
   - **Rationale**: Distinguishes condition changes from other rejection reasons
   - **Builds Upon**: ResponseMessage::OperationState enum
   - **Acceptance Criteria**:
     - Add `RejectedDueConditionsChanged` to ResponseMessage::OperationState enum
     - IntermediateNode sends this response when exchange rate or commission validation fails
     - Coordinator recognizes this response type and triggers condition change handling
   - **Priority**: High
   - **Dependencies**: None (enum extension)

4. **Enhanced CoordinatorReservationResponseMessage with Updated Conditions**
   - **Description**: Include updated exchange rate or commission in rejection response
   - **User Story**: As a coordinator, I need to know the new exchange rate or commission when a reservation is rejected due to condition changes
   - **Rationale**: Enables coordinator to update managers and recalculate paths
   - **Builds Upon**: Existing CoordinatorReservationResponseMessage structure
   - **Acceptance Criteria**:
     - Add optional fields to CoordinatorReservationResponseMessage:
       - `optional<pair<TrustLineAmount, int16_t>> mActualExchangeRate` (stores only rate and shift)
       - `optional<TrustLineAmount> mActualCommission`
     - Add two new constructors (in addition to default):
       - Constructor with actual exchange rate: takes reservedAmount, rate and shift as parameters
       - Constructor with actual commission: takes reservedAmount and commission as parameters
     - IntermediateNode creates response for RejectedDueConditionsChanged:
       - Retrieves full ExchangeRate from ExchangeRatesManager
       - Creates response using constructor with rate and shift parameters
       - Or retrieves commission and creates response using constructor with commission parameter
     - Both fields can be empty if condition was removed (use default constructor)
     - Message serialization/deserialization handles optional fields as pair<rate, shift>
     - Coordinator receives pair<rate, shift> and uses path context to determine equivalents for manager updates
   - **Priority**: High
   - **Dependencies**: RejectedDueConditionsChanged response type

5. **Coordinator Condition Change Handling**
   - **Description**: Detect condition changes, invalidate paths, create new paths, update managers
   - **User Story**: As a coordinator, I need to adapt when intermediate nodes report condition changes
   - **Rationale**: Enables payment to succeed despite condition changes
   - **Builds Upon**: Existing reservation response processing in CoordinatorExchangePaymentTransaction
   - **Acceptance Criteria**:
     - In response processing (processRemoteNodeResponse, processNeighborFurtherReservationResponse):
       - Detect RejectedDueConditionsChanged response
       - Extract actual exchange rate or commission from response
       - Call dropReservationsOnPath() to drop coordinator's reservation on path (sendToLastProcessedNode = false)
       - Send FinalPathExchangeConfigurationMessage to drop reservations on intermediate nodes of this path
       - Mark path unusable: `pathStats->setUnusable()`
       - Update ExchangeRatesManager (if exchange rate changed) or CommissionsManager (if commission changed) with new values
       - Create new path with updated conditions:
         - Set `optimal_flow = mMaxPathFlow` from invalidated path
         - Recalculate `received_amount` using new exchange rate or commission
         - Recalculate `flows` via `calculateFlows(optimal_flow)` with updated conditions
         - Insert new path into mPathsStats with next PathID
         - Shift all subsequent PathIDs by 1
       - Update all subsequent paths containing the affected node:
         - Recalculate `received_amount` for each path with new conditions
         - Recalculate `flows` for each path
         - **TODO**: Update ExchangePath mPath fields (exchangeSteps, minCapacity, effectiveExchangeRate, totalCommissions)
       - When new path taken into processing via switchToNextPath():
         - Apply existing path truncation logic if needed (from PRD 08)
   - **Priority**: High
   - **Dependencies**: Path invalidation logic, ExchangeRatesManager, CommissionsManager, path flow recalculation

6. **Transaction Context for Exchange Rates and Commissions**
   - **Description**: Store exchange rates and commissions in intermediate node transaction context
   - **User Story**: As an intermediate node, I need to store exchange rates and commissions I'm using for this payment to ensure consistent validation
   - **Rationale**: Ensures all paths in same payment use consistent conditions; enables transaction completion logic
   - **Builds Upon**: IntermediateNodeExchangePaymentTransaction structure
   - **Acceptance Criteria**:
     - Add context storage fields to IntermediateNodeExchangePaymentTransaction:
       - `map<pair<SerializedEquivalent, SerializedEquivalent>, ExchangeRate> mContextExchangeRates`
       - `map<SerializedEquivalent, Commission> mContextCommissions`
     - When processing CoordinatorReservationRequestMessage with exchange rate:
       - Check if rate already in context for this pair
       - If yes: use context rate for validation
       - If no: get rate from ExchangeRatesManager, store in context, use for validation
     - When processing CoordinatorReservationRequestMessage with commission:
       - Check if commission already in context for this equivalent
       - If yes: use context commission for validation
       - If no: get commission from CommissionsManager, store in context, use for validation
     - Context written only once per equivalent/pair (first path)
     - All subsequent validations use context values
     - If checkReservationsDirections() needs rate/commission not in context: validation fails (log error)
     - Transaction completion logic updated:
       - Can complete only if: no reservations AND no context rates AND no context commissions
       - If any of these exist: transaction must continue
   - **Priority**: High
   - **Dependencies**: None (new context storage)

7. **Commission "Charge Once Per Payment" Logic**
   - **Description**: Ensure commissions charged only once per payment per equivalent, regardless of path count
   - **User Story**: As an intermediate node, I should charge my commission only once per payment even if I participate in multiple paths
   - **Rationale**: Matches commission semantics from PRD 06; prevents overcharging multi-path payments
   - **Builds Upon**: Transaction context storage
   - **Acceptance Criteria**:
     - Commission charged on first path where node participates in given equivalent
     - Commission stored in mContextCommissions upon first charge
     - Subsequent paths in same equivalent don't charge commission (already in context)
     - Commission deducted from incoming amount only once per payment
     - checkReservationsDirections() accounts for commission based on context (not per-path)
   - **Priority**: High
   - **Dependencies**: Transaction context storage

8. **Outgoing/Incoming Reservation Validation**
   - **Description**: Validate that outgoing reservation matches incoming reservation after applying conditions
   - **User Story**: As an intermediate node, I need to validate that the coordinator's requested outgoing amount is correct based on my incoming reservation and current conditions
   - **Rationale**: Ensures coordinator calculated amounts correctly; detects arithmetic errors
   - **Builds Upon**: Existing reservation validation logic
   - **Acceptance Criteria**:
     - Validation occurs after exchange rate/commission validation passes
     - Find incoming reservation by same PathID
     - If no exchange rate and no commission:
       - Outgoing amount must equal incoming amount
       - Equivalents must match
       - Equivalent must match commission equivalent (if any)
     - If commission exists (same equivalent):
       - Outgoing amount must equal incoming amount minus commission
       - Equivalents must match
     - If exchange rate exists (different equivalents):
       - Outgoing amount must equal incoming amount after applying exchange rate
       - Equivalents must match exchange rate pair
     - If validation fails: send CoordinatorReservationResponseMessage with Rejected status
     - Method: add validation logic in `runCoordinatorRequestProcessingStage()` after condition validation
   - **Priority**: High
   - **Dependencies**: Transaction context with exchange rates and commissions

#### Enhancements to Existing Features

1. **ExchangePath Flow Recalculation**
   - **Current State**: flows calculated once during path building
   - **Proposed Changes**: Recalculate flows when exchange rate or commission changes
   - **Impact Assessment**: Enables path adaptation to condition changes
   - **Migration Strategy**: Direct recalculation when conditions change

2. **Path Invalidation Extension**
   - **Current State**: setUnusable() marks path invalid
   - **Proposed Changes**: Invalidate path and create replacement with updated conditions
   - **Impact Assessment**: Enables graceful recovery from condition changes
   - **Migration Strategy**: Extend existing invalidation logic

3. **Transaction Completion Logic in Intermediate Node**
   - **Current State**: Transaction completes when no reservations remain
   - **Proposed Changes**: Transaction completes only when no reservations AND no context rates/commissions
   - **Impact Assessment**: Ensures transaction stays alive for potential subsequent paths
   - **Migration Strategy**: Add context checks to completion condition

### Non-Functional Requirements
#### Performance
- Condition validation overhead < 5ms per reservation request
- Path recalculation overhead < 20ms per path
- Manager update overhead < 10ms per update
- Total overhead for condition change handling < 100ms

#### Security
- No security implications (internal validation and adaptation)
- Ensures nodes enforce their current pricing
- Prevents payments based on outdated conditions

#### Scalability
- Support condition changes on up to 50 paths per payment
- Handle up to 10 condition changes per payment
- Support up to 100 concurrent payments with condition changes

#### Reliability
- Condition change handling failures don't crash transaction
- Manager update failures logged and continue payment
- Path recalculation errors handled gracefully
- Transaction state recovery after condition change handling

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Exchange payments assume static conditions during execution
- **Proposed Changes**: Add condition validation, change detection, path adaptation, and manager updates
- **Backwards Compatibility**: No protocol changes to existing messages; extensions are backward compatible
- **Migration Requirements**: None (runtime behavior enhancement)

### Technology Stack Updates
#### New Technologies/Libraries
- No new external libraries required
- Reuses existing infrastructure: ExchangeRatesManager, CommissionsManager, OptimalPathResult

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- IntermediateNode integrates with ExchangeRatesManager for validation
- IntermediateNode integrates with CommissionsManager for validation
- Coordinator integrates with managers for updates

#### Modified Integrations
- CoordinatorReservationRequestMessage extended with condition fields
- CoordinatorReservationResponseMessage extended with actual condition fields
- ResponseMessage extended with RejectedDueConditionsChanged

### Data Requirements
#### Data Models

##### Enhanced CoordinatorReservationRequestMessage
**Purpose**: Include expected exchange rate or commission for validation

**Changes**:
```cpp
class CoordinatorReservationRequestMessage : public RequestMessage {
    // Existing fields...

    // New constructors:
    // Default constructor (no exchange rate, no commission) - existing signature
    CoordinatorReservationRequestMessage(...);

    // Constructor with expected exchange rate
    CoordinatorReservationRequestMessage(
        ..., // existing parameters
        const TrustLineAmount &expectedExchangeRate,
        const int16_t expectedExchangeRateShift);

    // Constructor with expected commission
    CoordinatorReservationRequestMessage(
        ..., // existing parameters
        const TrustLineAmount &expectedCommission);

    // New getters:
    const optional<pair<TrustLineAmount, int16_t>>& expectedExchangeRate() const;
    const optional<TrustLineAmount>& expectedCommission() const;

    // New fields:
    // Exchange rate stored as pair<rate, shift> - equivalents derived from context
    optional<pair<TrustLineAmount, int16_t>> mExpectedExchangeRate;
    optional<TrustLineAmount> mExpectedCommission;
};
```

**Serialization Details**:
- For `optional<ExchangeRate>`: Serialize only `exchangeRate` (TrustLineAmount) and `exchangeRateShift` (int16_t)
  - `equivalentFrom` and `equivalentTo` can be derived from context:
    - `equivalentTo` = equivalent of the reservation being requested (from message)
    - `equivalentFrom` = equivalent of the incoming reservation on the same path
  - `expiresAt`, `minExchangeAmount`, `maxExchangeAmount` are not needed for basic validation
- For `optional<TrustLineAmount>`: Serialize the commission amount directly
- Use byte_t flags (0/1) to indicate presence of optional fields

**Context Reconstruction at Intermediate Node**:
- Message contains only `pair<rate, shift>`, but intermediate node needs to identify which ExchangeRate this refers to
- **equivalentFrom**: obtained from the incoming reservation already created on this path (from transaction reservations)
- **equivalentTo**: obtained from the outgoing reservation being requested (from message/reservation context)
- With the pair (equivalentFrom, equivalentTo), intermediate node:
  1. Looks up the full ExchangeRate object from ExchangeRatesManager
  2. Compares the received (rate, shift) with current ExchangeRate.exchangeRate() and ExchangeRate.exchangeRateShift()
  3. If validation passes, stores the **complete ExchangeRate object** (with all fields) in transaction context
- This ensures consistent validation across multiple paths using the same rate

**Population Logic**:
- Coordinator determines from path data if node is exchanger or charges commission
- If node is exchanger:
  - Retrieves full ExchangeRate from ExchangeRatesManager
  - Extracts only rate and shift: `make_pair(exchangeRate.exchangeRate(), exchangeRate.exchangeRateShift())`
  - Sets mExpectedExchangeRate with this pair
- If node charges commission:
  - Retrieves commission from CommissionsManager
  - Sets mExpectedCommission with the amount
- Both fields empty if node neither exchanges nor charges commission

##### Enhanced CoordinatorReservationResponseMessage
**Purpose**: Include actual exchange rate or commission in rejection response

**Changes**:
```cpp
class CoordinatorReservationResponseMessage : public ResponseMessage {
    // Existing fields...

    // New constructors:
    // Default constructor (no exchange rate, no commission) - existing signature
    CoordinatorReservationResponseMessage(...);

    // Constructor with actual exchange rate
    CoordinatorReservationResponseMessage(
        ..., // existing parameters including reservedAmount
        const TrustLineAmount &actualExchangeRate,
        const int16_t actualExchangeRateShift);

    // Constructor with actual commission
    CoordinatorReservationResponseMessage(
        ..., // existing parameters including reservedAmount
        const TrustLineAmount &actualCommission);

    // New getters:
    const optional<pair<TrustLineAmount, int16_t>>& actualExchangeRate() const;
    const optional<TrustLineAmount>& actualCommission() const;

    // New fields:
    // Exchange rate stored as pair<rate, shift> - equivalents derived from context
    optional<pair<TrustLineAmount, int16_t>> mActualExchangeRate;
    optional<TrustLineAmount> mActualCommission;
};
```

**Serialization Details**:
- Same serialization approach as CoordinatorReservationRequestMessage
- For `optional<ExchangeRate>`: Serialize only `exchangeRate` (TrustLineAmount) and `exchangeRateShift` (int16_t)
- For `optional<TrustLineAmount>`: Serialize the commission amount directly
- Use byte_t flags (0/1) to indicate presence of optional fields

**Population Logic**:
- IntermediateNode creates response when sending RejectedDueConditionsChanged
- Extracts rate and shift from current ExchangeRate object in ExchangeRatesManager
- Creates response using constructor with actual exchange rate: passes rate and shift as parameters
- If condition was removed: uses default constructor (optional fields remain nullopt)

**Context Reconstruction at Coordinator**:
- Coordinator receives only `pair<rate, shift>` in the response
- **equivalentFrom/To**: coordinator already knows these from the path being processed
- Coordinator uses these equivalents to update ExchangeRatesManager with new rate/shift values
- Full ExchangeRate object reconstruction happens at ExchangeRatesManager level

##### ResponseMessage::OperationState Extension
**Purpose**: Add rejection type for condition changes

**Changes**:
```cpp
enum OperationState {
    // Existing values...
    RejectedDueConditionsChanged = 12,  // New value
};
```

##### IntermediateNode Transaction Context
**Purpose**: Store exchange rates and commissions for consistent validation

**Structure**:
```cpp
class IntermediateNodeExchangePaymentTransaction {
    // Existing fields...

    // New context storage:
    map<pair<SerializedEquivalent, SerializedEquivalent>, ExchangeRate> mContextExchangeRates;
    map<SerializedEquivalent, Commission> mContextCommissions;
};
```

**Usage**:
- Written once per equivalent/pair on first path
- Used for all subsequent validations
- Checked in transaction completion logic

**Context Storage Details**:
- When storing ExchangeRate in context, store the **complete ExchangeRate object** with all fields from ExchangeRatesManager
- This includes: equivalentFrom, equivalentTo, exchangeRate, exchangeRateShift, expiresAt, minExchangeAmount, maxExchangeAmount
- Even though only exchangeRate and exchangeRateShift are transmitted in messages, the full object is maintained in context for potential future use

#### Data Storage
- No persistent storage changes
- Runtime-only context in transaction state
- Manager updates reflected in ExchangeRatesManager/CommissionsManager storage

#### Data Migration
- No migration needed (runtime enhancement)

### Algorithm Specifications

#### Intermediate Node Condition Validation in runCoordinatorRequestProcessingStage()
**Purpose**: Validate reservation request against current exchange rates and commissions

**Algorithm**:
```cpp
TransactionResult::SharedConst
IntermediateNodeExchangePaymentTransaction::runCoordinatorRequestProcessingStage() {
    // ... [existing initialization and setup] ...

    // Step 1: Extract expected conditions from request
    auto request = /* get CoordinatorReservationRequestMessage */;
    auto expectedExchangeRate = request->expectedExchangeRate();
    auto expectedCommission = request->expectedCommission();

    // Step 1.1: Validate mutual exclusivity (protocol violation check)
    if (expectedExchangeRate.has_value() && expectedCommission.has_value()) {
        error() << "Protocol violation: both exchange rate and commission provided in request";
        return sendErrorMessageOnCoordinatorRequest(ResponseMessage::Rejected);
    }

    // Step 2: Validate exchange rate if provided
    if (expectedExchangeRate.has_value()) {
        // Get equivalent pair from reservation context
        auto incomingEquiv = /* incoming reservation equivalent */;
        auto outgoingEquiv = /* outgoing reservation equivalent */;
        auto ratePair = make_pair(incomingEquiv, outgoingEquiv);

        // Check if already in context
        ExchangeRate actualRate;
        auto contextIt = mContextExchangeRates.find(ratePair);

        if (contextIt != mContextExchangeRates.end()) {
            // Use rate from context
            actualRate = contextIt->second;
        } else {
            // Get current rate from manager
            auto rateOpt = mExchangeRatesManager->get(incomingEquiv, outgoingEquiv);

            if (!rateOpt) {
                // Rate not found but expected - reject
                warning() << "Exchange rate not found for pair ("
                          << incomingEquiv << ", " << outgoingEquiv << ")";
                return sendRejectedDueConditionsChanged(nullopt, nullopt);
            }

            actualRate = *rateOpt;
            // Store in context for future validations
            mContextExchangeRates[ratePair] = actualRate;
        }

        // Compare rates
        if (actualRate != *expectedExchangeRate) {
            info() << "Exchange rate mismatch: expected " << *expectedExchangeRate
                   << ", actual " << actualRate;

            // Drop incoming reservation on this path
            dropIncomingReservationOnPath(pathID);

            return sendRejectedDueConditionsChanged(actualRate, nullopt);
        }
    }

    // Step 3: Validate commission if provided
    if (expectedCommission.has_value()) {
        auto equivalent = /* get equivalent from reservation */;

        // Check if already in context
        Commission actualCommission;
        auto contextIt = mContextCommissions.find(equivalent);

        if (contextIt != mContextCommissions.end()) {
            // Use commission from context
            actualCommission = contextIt->second;
        } else {
            // Get current commission from manager
            auto commissionOpt = mCommissionsManager->get(equivalent);

            if (!commissionOpt) {
                // Commission not found but expected - reject with empty commission
                info() << "Commission no longer exists for equivalent " << equivalent;

                dropIncomingReservationOnPath(pathID);

                return sendRejectedDueConditionsChanged(nullopt, TrustLineAmount(0));
            }

            actualCommission = *commissionOpt;
            // Store in context for future validations (first path only)
            mContextCommissions[equivalent] = actualCommission;
        }

        // Compare commissions
        if (actualCommission.amount() != *expectedCommission) {
            info() << "Commission mismatch: expected " << *expectedCommission
                   << ", actual " << actualCommission.amount();

            dropIncomingReservationOnPath(pathID);

            return sendRejectedDueConditionsChanged(nullopt, actualCommission.amount());
        }
    }

    // Step 4: Handle case where coordinator didn't send condition but node has one
    if (!expectedExchangeRate.has_value() && !expectedCommission.has_value()) {
        // Check if node has exchange rate or commission that coordinator didn't know about
        auto incomingEquiv = /* incoming reservation equivalent */;
        auto outgoingEquiv = /* outgoing reservation equivalent */;

        if (incomingEquiv != outgoingEquiv) {
            // Different equivalents - should have exchange rate
            auto rateOpt = mExchangeRatesManager->get(incomingEquiv, outgoingEquiv);
            if (rateOpt) {
                info() << "Exchange rate exists but not provided in request";
                dropIncomingReservationOnPath(pathID);
                return sendRejectedDueConditionsChanged(*rateOpt, nullopt);
            }
        } else {
            // Same equivalent - check commission
            auto commissionOpt = mCommissionsManager->get(incomingEquiv);
            if (commissionOpt) {
                info() << "Commission exists but not provided in request";
                dropIncomingReservationOnPath(pathID);
                return sendRejectedDueConditionsChanged(nullopt, commissionOpt->amount());
            }
        }
    }

    // Step 5: Validation passed - continue with outgoing/incoming reservation validation
    // ... [existing reservation processing logic] ...
}
```

**Helper Method**:
```cpp
TransactionResult::SharedConst sendRejectedDueConditionsChanged(
    optional<ExchangeRate> actualRate,
    optional<TrustLineAmount> actualCommission)
{
    // Create rejection response using appropriate constructor
    shared_ptr<CoordinatorReservationResponseMessage> response;

    if (actualRate && actualCommission) {
        // Both rate and commission changed - use combined constructor
        response = make_shared<CoordinatorReservationResponseMessage>(
            equivalent(), senderAddresses, currentTransactionUUID(), pathID,
            ResponseMessage::RejectedDueConditionsChanged,
            TrustLineAmount(0), actualRate, actualCommission);
    } else if (actualRate) {
        // Only rate changed
        response = make_shared<CoordinatorReservationResponseMessage>(
            equivalent(), senderAddresses, currentTransactionUUID(), pathID,
            ResponseMessage::RejectedDueConditionsChanged,
            TrustLineAmount(0), actualRate->first, actualRate->second);
    } else if (actualCommission) {
        // Only commission changed
        response = make_shared<CoordinatorReservationResponseMessage>(
            equivalent(), senderAddresses, currentTransactionUUID(), pathID,
            ResponseMessage::RejectedDueConditionsChanged,
            TrustLineAmount(0), *actualCommission);
    } else {
        // Condition removed - use default constructor
        response = make_shared<CoordinatorReservationResponseMessage>(
            equivalent(), senderAddresses, currentTransactionUUID(), pathID,
            ResponseMessage::RejectedDueConditionsChanged,
            TrustLineAmount(0));
    }

    sendMessage(response, coordinatorAddress());
    return resultDone();
}
```

**Key Points**:
- Validates exchange rate if provided in request
- Validates commission if provided in request
- Uses context values if already stored
- Stores in context on first validation
- Drops incoming reservation on path if validation fails
- Returns RejectedDueConditionsChanged with actual values
- Handles case where condition exists but wasn't provided

#### Outgoing/Incoming Reservation Validation
**Purpose**: Validate outgoing reservation matches incoming after applying conditions

**Algorithm**:
```cpp
// In runCoordinatorRequestProcessingStage(), after condition validation passes

// Step 1: Find incoming reservation by PathID
AmountReservation::ConstShared incomingReservation = nullptr;
SerializedEquivalent incomingEquiv;

for (const auto &[contractorID, reservationsMap] : mReservations) {
    for (const auto &[resPathID, reservation] : reservationsMap) {
        if (resPathID == pathID &&
            reservation->direction() == AmountReservation::Incoming) {
            incomingReservation = reservation;
            incomingEquiv = reservation->equivalent();
            break;
        }
    }
    if (incomingReservation) break;
}

if (!incomingReservation) {
    error() << "Incoming reservation not found for pathID=" << pathID;
    return sendErrorMessageOnCoordinatorRequest(ResponseMessage::Rejected);
}

// Step 2: Get outgoing reservation details from request
TrustLineAmount requestedOutgoingAmount = request->amount();
SerializedEquivalent outgoingEquiv = /* from request or reservation context */;

// Step 3: Validate based on conditions
bool validationPassed = false;

if (incomingEquiv == outgoingEquiv) {
    // Same equivalent - check commission
    auto commissionIt = mContextCommissions.find(incomingEquiv);

    if (commissionIt != mContextCommissions.end()) {
        // Commission exists
        TrustLineAmount expectedOutgoing =
            incomingReservation->amount() - commissionIt->second.amount();

        validationPassed = (requestedOutgoingAmount == expectedOutgoing);

        if (!validationPassed) {
            warning() << "Outgoing amount mismatch with commission: "
                      << "expected " << expectedOutgoing
                      << ", got " << requestedOutgoingAmount;
        }
    } else {
        // No commission - amounts should be equal
        validationPassed = (requestedOutgoingAmount == incomingReservation->amount());

        if (!validationPassed) {
            warning() << "Outgoing amount mismatch without commission: "
                      << "expected " << incomingReservation->amount()
                      << ", got " << requestedOutgoingAmount;
        }
    }
} else {
    // Different equivalents - check exchange rate
    auto ratePair = make_pair(incomingEquiv, outgoingEquiv);
    auto rateIt = mContextExchangeRates.find(ratePair);

    if (rateIt != mContextExchangeRates.end()) {
        // Apply exchange rate
        TrustLineAmount expectedOutgoing = applyExchangeRate(
            incomingReservation->amount(),
            rateIt->second);

        validationPassed = (requestedOutgoingAmount == expectedOutgoing);

        if (!validationPassed) {
            warning() << "Outgoing amount mismatch with exchange: "
                      << "expected " << expectedOutgoing
                      << ", got " << requestedOutgoingAmount;
        }
    } else {
        error() << "Exchange rate not found in context for validation";
        validationPassed = false;
    }
}

// Step 4: Send rejection if validation failed
if (!validationPassed) {
    return sendErrorMessageOnCoordinatorRequest(ResponseMessage::Rejected);
}

// Step 5: Validation passed - continue processing
// ... [existing outgoing reservation logic] ...
```

**Key Points**:
- Validates after condition validation passes
- Uses context rates/commissions for validation
- Checks different logic for same vs different equivalents
- Rejects with Rejected status (not RejectedDueConditionsChanged) if mismatch
- Uses existing reservation amounts for comparison

#### Coordinator Condition Change Handling
**Purpose**: Detect condition changes, invalidate paths, create new paths, update managers

**Algorithm**:
```cpp
// In processRemoteNodeResponse() or processNeighborFurtherReservationResponse()

if (response->state() == ResponseMessage::RejectedDueConditionsChanged) {
    info() << "Received RejectedDueConditionsChanged from node on pathID=" << pathID;

    // Step 1: Extract actual conditions from response
    auto actualExchangeRate = response->actualExchangeRate();
    auto actualCommission = response->actualCommission();

    // Step 2: Get path stats
    auto pathStatsIt = mPathsStats.find(pathID);
    if (pathStatsIt == mPathsStats.end()) {
        warning() << "Path not found for pathID=" << pathID;
        return tryProcessNextPath();
    }

    OptimalPathResult *pathStats = pathStatsIt->second.get();
    const auto &path = pathStats->path();

    // Step 3: Drop reservations on this path
    dropReservationsOnPath(pathStats, pathID, /* sendToLastProcessedNode */ false);

    // Step 4: Mark path unusable
    pathStats->setUnusable();

    // Step 5: Update manager with new conditions
    if (actualExchangeRate) {
        // Update ExchangeRatesManager
        auto incomingEquiv = /* determine from path and node position */;
        auto outgoingEquiv = /* determine from path and node position */;

        mExchangeRatesManager->set(incomingEquiv, outgoingEquiv, *actualExchangeRate);

        info() << "Updated exchange rate in manager: "
               << incomingEquiv << "->" << outgoingEquiv
               << " = " << *actualExchangeRate;
    }

    if (actualCommission) {
        // Update CommissionsManager
        auto equivalent = /* determine from path and node position */;

        if (*actualCommission > TrustLineAmount(0)) {
            mCommissionsManager->set(equivalent, *actualCommission);
            info() << "Updated commission in manager: eq=" << equivalent
                   << " commission=" << *actualCommission;
        } else {
            // Commission removed
            mCommissionsManager->remove(equivalent);
            info() << "Removed commission from manager: eq=" << equivalent;
        }
    }

    // Step 6: Create new path with updated conditions
    TrustLineAmount oldOptimalFlow = pathStats->mMaxPathFlow;

    // Calculate new received amount with updated conditions
    TrustLineAmount newReceivedAmount = calculateReceivedAmountWithUpdatedConditions(
        pathStats,
        oldOptimalFlow,
        actualExchangeRate,
        actualCommission);

    // Create new OptimalPathResult
    auto newPathStats = make_unique<OptimalPathResult>();
    newPathStats->path = pathStats->path;  // Copy path structure
    newPathStats->optimal_flow = oldOptimalFlow;
    newPathStats->mMaxPathFlow = oldOptimalFlow;
    newPathStats->received_amount = newReceivedAmount;

    // Recalculate flows
    try {
        newPathStats->calculateFlows(oldOptimalFlow);
    } catch (const exception &e) {
        error() << "Error calculating flows for new path: " << e.what();
        return tryProcessNextPath();
    }

    // Insert new path and shift subsequent paths
    PathID newPathID = generateNextPathID();
    mPathsStats[newPathID] = std::move(newPathStats);
    mPathIDs.insert(mPathIDs.begin() + currentPathIndex + 1, newPathID);

    info() << "Created new path with ID=" << newPathID
           << ", optimalFlow=" << oldOptimalFlow
           << ", receivedAmount=" << newReceivedAmount;

    // Step 7: Update all subsequent paths containing affected node
    BaseAddress::Shared affectedNode = /* node that sent rejection */;

    for (size_t idx = currentPathIndex + 2; idx < mPathIDs.size(); ++idx) {
        PathID subsequentPathID = mPathIDs[idx];
        auto &subsequentPathStats = mPathsStats[subsequentPathID];

        // Check if path contains affected node
        bool containsNode = false;
        for (const auto &node : subsequentPathStats->path().nodes) {
            if (node == affectedNode) {
                containsNode = true;
                break;
            }
        }

        if (containsNode) {
            // Recalculate received_amount and flows
            TrustLineAmount subsequentFlow = subsequentPathStats->optimal_flow;

            TrustLineAmount updatedReceived = calculateReceivedAmountWithUpdatedConditions(
                subsequentPathStats.get(),
                subsequentFlow,
                actualExchangeRate,
                actualCommission);

            subsequentPathStats->received_amount = updatedReceived;

            try {
                subsequentPathStats->calculateFlows(subsequentFlow);
            } catch (const exception &e) {
                warning() << "Error recalculating flows for path " << subsequentPathID
                          << ": " << e.what();
            }

            // TODO: Update ExchangePath mPath fields:
            // - exchangeSteps
            // - minCapacity
            // - effectiveExchangeRate
            // - totalCommissions

            info() << "Updated subsequent path " << subsequentPathID
                   << " with new receivedAmount=" << updatedReceived;
        }
    }

    // Step 8: Continue with next path
    return tryProcessNextPath();
}
```

**Helper Method**:
```cpp
TrustLineAmount calculateReceivedAmountWithUpdatedConditions(
    OptimalPathResult *pathStats,
    const TrustLineAmount &inputFlow,
    const optional<ExchangeRate> &updatedRate,
    const optional<TrustLineAmount> &updatedCommission)
{
    // Forward simulate through path with updated conditions
    // Similar to shortageReservationsOnPath from PRD 08
    // but uses updated rate/commission instead of stored ones

    TrustLineAmount currentAmount = inputFlow;
    const auto &path = pathStats->path();

    for (size_t idx = 0; idx + 1 < path.ids.size(); ++idx) {
        const ContractorID fromNode = path.ids[idx];
        const ContractorID toNode = path.ids[idx + 1];
        const SerializedEquivalent currentEquiv = path.equivalents[idx];
        const SerializedEquivalent nextEquiv = path.equivalents[idx + 1];

        // Check for exchange
        if (fromNode == toNode && currentEquiv != nextEquiv) {
            const auto *exchangeStep = findExchangeStep(path, fromNode, currentEquiv, nextEquiv);

            if (!exchangeStep) {
                throw ValueError("Exchange step not found");
            }

            // Use updated rate if this is the affected exchange
            if (updatedRate &&
                exchangeStep->sourceEquivalent == currentEquiv &&
                exchangeStep->targetEquivalent == nextEquiv) {
                currentAmount = applyExchangeForward(currentAmount, *updatedRate);
            } else {
                currentAmount = applyExchangeForward(currentAmount, exchangeStep->exchangeRate);
            }
            continue;
        }

        // Check for commission
        if (idx + 1 < path.ids.size() - 1) {
            const auto *commissionStep = findExchangeStep(path, toNode, nextEquiv, nextEquiv);

            if (commissionStep && commissionStep->commission > TrustLineAmount(0)) {
                // Use updated commission if this is the affected node
                TrustLineAmount commissionToApply;

                if (updatedCommission && toNode == /* affected node ID */) {
                    commissionToApply = *updatedCommission;
                } else {
                    commissionToApply = commissionStep->commission;
                }

                if (currentAmount < commissionToApply) {
                    throw ValueError("Amount exhausted by commission");
                }

                currentAmount = currentAmount - commissionToApply;
            }
        }
    }

    return currentAmount;
}
```

**Key Points**:
- Detects RejectedDueConditionsChanged response
- Drops reservations on invalidated path
- Marks path unusable
- Updates ExchangeRatesManager or CommissionsManager
- Creates new path with same optimal_flow but recalculated received_amount
- Inserts new path at next PathID position
- Updates all subsequent paths containing affected node
- Recalculates flows for all affected paths
- Continues processing with next path

#### Transaction Completion Logic Update
**Purpose**: Prevent premature transaction completion when context has stored conditions

**Algorithm**:
```cpp
// In IntermediateNodeExchangePaymentTransaction, check before completing transaction

bool canCompleteTransaction() const {
    // Existing check: no reservations
    bool hasReservations = !mReservations.empty();

    // New checks: no context rates or commissions
    bool hasContextRates = !mContextExchangeRates.empty();
    bool hasContextCommissions = !mContextCommissions.empty();

    bool canComplete = !hasReservations && !hasContextRates && !hasContextCommissions;

    if (!canComplete) {
        debug() << "Cannot complete transaction: "
                << "hasReservations=" << hasReservations
                << ", hasContextRates=" << hasContextRates
                << ", hasContextCommissions=" << hasContextCommissions;
    }

    return canComplete;
}
```

**Key Points**:
- Checks for reservations (existing logic)
- Checks for context exchange rates (new)
- Checks for context commissions (new)
- Transaction completes only when all three are empty
- Ensures transaction stays alive for subsequent paths

#### Enhanced Condition Change Handling with Maximum Receiver Capacity
**Purpose**: Optimize path recalculation by prioritizing receiver-side capacity preservation when conditions change

**Background**:
The original implementation (task 09-03) recalculates paths when conditions change by preserving sender-side flow (`mMaxPathFlow`) and recalculating receiver amount. However, this approach doesn't maximize payment throughput when conditions improve (e.g., exchange rate increases or commission decreases).

To address this, we introduce `mMaxPathReceivedAmount` - the maximum receiver-side capacity that can be delivered through a path. When conditions change, we attempt to preserve this receiver capacity by recalculating the required sender-side flow. Only if the new required flow exceeds sender capacity do we fall back to constraining by sender capacity.

**New Field in OptimalPathResult**:
```cpp
class OptimalPathResult {
    // Existing fields
    TrustLineAmount mMaxPathFlow;        // Maximum sender-side flow
    TrustLineAmount optimal_flow;        // Current sender-side flow
    TrustLineAmount received_amount;     // Current receiver-side amount

    // NEW: Maximum receiver-side capacity
    TrustLineAmount mMaxPathReceivedAmount;  // Maximum deliverable amount to receiver
};
```

**Field Initialization**:
- In `addPathForFurtherProcessing()`:
  ```cpp
  pathCopy->mMaxPathFlow = pathResult.optimal_flow;
  pathCopy->mMaxPathReceivedAmount = pathResult.received_amount;  // NEW
  ```
- In `shortageReservationsOnPath()`: Both fields updated when capacity reduced
- In `handleConditionChange()`: Both fields preserved from original path (see algorithm below)

**Enhanced Algorithm**:
```cpp
// In handleConditionChange(), after dropping reservations and before creating new path:

// Step 1: Save critical parameters BEFORE dropReservationsOnPath/setUnusable
// (these methods reset mMaxPathFlow to 0)
TrustLineAmount savedMaxPathFlow = pathStats->mMaxPathFlow;
TrustLineAmount savedMaxPathReceivedAmount = pathStats->mMaxPathReceivedAmount;
TrustLineAmount savedPaymentFlow = pathStats->paymentFlow;

// Step 2: Drop reservations and mark unusable
dropReservationsOnPath(pathStats, pathID, false);
pathStats->setUnusable();

// Step 3: Calculate required optimal_flow to deliver mMaxPathReceivedAmount
TrustLineAmount newOptimalFlow;
try {
    newOptimalFlow = calculateOptimalFlowWithUpdatedConditions(
        pathStats,
        savedMaxPathReceivedAmount,  // Target receiver amount
        affectedPositionInPath,
        actualExchangeRate,
        actualCommission);
} catch (const exception &e) {
    error() << "Error calculating optimal flow: " << e.what();
    return tryProcessNextPath();
}

// Step 4: Determine final flow and received amount based on sender capacity
TrustLineAmount finalOptimalFlow;
TrustLineAmount finalReceivedAmount;

if (newOptimalFlow <= savedMaxPathFlow) {
    // New conditions acceptable: can deliver full receiver capacity
    finalOptimalFlow = newOptimalFlow;
    finalReceivedAmount = savedMaxPathReceivedAmount;

    info() << "Conditions improved/acceptable: using full receiver capacity "
           << savedMaxPathReceivedAmount;
} else {
    // New conditions require more resources: constrain by sender capacity
    finalOptimalFlow = savedMaxPathFlow;

    finalReceivedAmount = calculateReceivedAmountWithUpdatedConditions(
        pathStats,
        savedMaxPathFlow,  // Constrain by sender capacity
        affectedPositionInPath,
        actualExchangeRate,
        actualCommission);

    info() << "Conditions worsened: constraining to sender capacity, "
           << "receivedAmount=" << finalReceivedAmount;
}

// Step 5: Create new path with calculated values
auto newPathStats = make_unique<OptimalPathResult>();
newPathStats->mPath = updatedPath;
newPathStats->optimal_flow = finalOptimalFlow;
newPathStats->received_amount = finalReceivedAmount;

// Preserve original capacity bounds (independent of current flow)
newPathStats->mMaxPathFlow = savedMaxPathFlow;
newPathStats->mMaxPathReceivedAmount = savedMaxPathReceivedAmount;

// Step 6: Recalculate flows
newPathStats->calculateFlows(finalOptimalFlow);
```

**New Helper Method - Backward Simulation**:
```cpp
TrustLineAmount calculateOptimalFlowWithUpdatedConditions(
    OptimalPathResult *pathStats,
    const TrustLineAmount &desiredReceivedAmount,
    const SerializedPositionInPath affectedNodePosition,
    const optional<pair<TrustLineAmount, int16_t>> &updatedRate,
    const optional<TrustLineAmount> &updatedCommission)
{
    // Backward simulation: receiver -> sender
    TrustLineAmount requiredAmount = desiredReceivedAmount;
    const auto &path = pathStats->path();

    // Iterate backward through path
    for (size_t idx = path.ids.size() - 1; idx > 0; --idx) {
        // Check for exchange at current position
        if (previousNode == currentNode && previousEquiv != currentEquiv) {
            // Invert exchange rate
            if (idx - 1 == affectedNodePosition && updatedRate) {
                // Use updated rate (inverted)
                requiredAmount = invertExchangeForRequiredInput(updatedRate, requiredAmount);
            } else {
                // Use original rate (inverted)
                requiredAmount = invertExchangeForRequiredInput(originalRate, requiredAmount);
            }
            continue;
        }

        // Check for commission at intermediate node
        if (idx < path.ids.size() - 1 && idx > 0) {
            TrustLineAmount commissionToAdd;
            if (idx == affectedNodePosition && updatedCommission) {
                commissionToAdd = *updatedCommission;
            } else {
                commissionToAdd = findOriginalCommission(path, currentNode);
            }
            // Add commission back (going backward)
            requiredAmount = requiredAmount + commissionToAdd;
        }
    }

    return requiredAmount;  // Required sender-side flow
}
```

**Key Benefits**:
1. **Better throughput when conditions improve**: If exchange rate increases or commission decreases, the new path can deliver more to receiver without increasing sender flow
2. **Correct behavior when conditions worsen**: Falls back to sender capacity constraint, same as original implementation
3. **Preserves capacity bounds**: `mMaxPathFlow` and `mMaxPathReceivedAmount` remain constant as upper bounds, independent of current flow adjustments
4. **Distinction from shortage**: Unlike `shortageReservationsOnPath` which updates both bounds (real capacity reduction), `handleConditionChange` preserves bounds (pricing change only)

**Example Scenario**:
- Initial path: `optimal_flow=2990`, `mMaxPathFlow=2990`, `received_amount=146`, `mMaxPathReceivedAmount=146`
- After partial payment: `optimal_flow=2070`, `mMaxPathFlow=2990`, `received_amount=100`, `mMaxPathReceivedAmount=146`
- Exchange rate changes: 0.05 → 0.04 (improves by 20%)
- New calculation:
  - Required flow for `mMaxPathReceivedAmount=146` → 3735
  - 3735 > 2990 → constrain to `mMaxPathFlow=2990`
  - New `received_amount` = 116 (was 100 before improvement)
  - Payment can deliver more despite using same sender capacity

**Implementation Notes**:
- `calculateOptimalFlowWithUpdatedConditions` performs backward simulation (receiver to sender)
- `calculateReceivedAmountWithUpdatedConditions` performs forward simulation (sender to receiver) - existing method
- Both methods apply updated conditions at affected position and original conditions elsewhere
- Saved values critical because `setUnusable()` resets `mMaxPathFlow` to 0

### Error Handling Specifications

#### Error Conditions
1. **Protocol violation: both exchange rate and commission in request (Intermediate Node)**:
   - Log error: `error() << "Protocol violation: both exchange rate and commission provided in request"`
   - Send CoordinatorReservationResponseMessage with Rejected status
   - Do not drop reservations (protocol error, not condition mismatch)
   - Continue transaction

2. **Exchange rate not found during validation (Intermediate Node)**:
   - Log warning: `warning() << "Exchange rate not found for pair (" << incomingEquiv << ", " << outgoingEquiv << ")"`
   - Send RejectedDueConditionsChanged with empty actualExchangeRate
   - Drop incoming reservation on path
   - Continue transaction

3. **Commission not found during validation (Intermediate Node)**:
   - Log info: `info() << "Commission no longer exists for equivalent " << equivalent`
   - Send RejectedDueConditionsChanged with actualCommission = TrustLineAmount(0)
   - Drop incoming reservation on path
   - Continue transaction

4. **Exchange rate mismatch (Intermediate Node)**:
   - Log info with expected and actual rates
   - Drop incoming reservation on path
   - Send RejectedDueConditionsChanged with actual rate
   - Continue transaction

5. **Commission mismatch (Intermediate Node)**:
   - Log info with expected and actual commission
   - Drop incoming reservation on path
   - Send RejectedDueConditionsChanged with actual commission
   - Continue transaction

6. **Outgoing/incoming reservation validation failure (Intermediate Node)**:
   - Log warning with expected and actual amounts
   - Send Rejected (not RejectedDueConditionsChanged)
   - Continue transaction

7. **Path not found during condition change handling (Coordinator)**:
   - Log warning: `warning() << "Path not found for pathID=" << pathID`
   - Call tryProcessNextPath()
   - Continue payment

8. **Flow recalculation error during new path creation (Coordinator)**:
   - Log error: `error() << "Error calculating flows for new path: " << e.what()`
   - Skip new path creation
   - Call tryProcessNextPath()
   - Continue payment

9. **Manager update failure (Coordinator)**:
   - Log error but continue
   - Path recalculation may use stale data but won't crash
   - Subsequent payments will get updated data

10. **checkReservationsDirections() missing context rate/commission**:
   - Log error: `error() << "Exchange rate/commission not found in context for validation"`
   - Return false from checkReservationsDirections()
   - Transaction fails validation
   - Payment rolls back

## Implementation Plan
### This Iteration Timeline
- **Duration**: 3-4 weeks implementation + 1-2 weeks testing
- **Sprint Breakdown**:
  - Sprint 1 (Week 1): Message protocol extensions (CoordinatorReservationRequestMessage, CoordinatorReservationResponseMessage, RejectedDueConditionsChanged)
  - Sprint 2 (Week 2): Intermediate node validation logic and transaction context
  - Sprint 3 (Week 2-3): Coordinator condition change handling and path recalculation
  - Sprint 4 (Week 3-4): Commission "charge once" logic and outgoing/incoming validation
  - Sprint 5 (Week 4): Integration testing and edge case validation

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Message Protocol Extended | Week 1 | All message extensions implemented | None | Low |
| Intermediate Node Validation Complete | Week 2 | Condition validation and context storage working | Message protocol | Medium |
| Coordinator Adaptation Complete | Week 3 | Path invalidation, creation, and manager updates working | Intermediate validation | High |
| Commission Logic Complete | Week 4 | Charge once per payment implemented | Context storage | Medium |
| Integration Testing Complete | Week 4 | End-to-end condition change handling validated | All previous | Medium |

### Dependencies on Other Teams/Projects
- No external team dependencies identified

### Integration Points with Previous Work
- Builds upon CoordinatorExchangePaymentTransaction (PRD 06)
- Builds upon IntermediateNodeExchangePaymentTransaction (PRD 06)
- Extends path invalidation from PRD 08
- Uses ExchangeRatesManager and CommissionsManager (PRD 03-04)

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++ and payment transaction experience
- **Developers**: 1 developer for implementation support
- **QA Engineers**: 1 engineer for unit testing

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Path recalculation errors with updated conditions | High | Medium | Thorough unit testing with diverse condition combinations; extensive logging |
| Manager update race conditions | Medium | Low | Use existing manager thread safety; validate updates in tests |
| Context storage memory issues | Low | Low | Context cleared on transaction completion; bounded by number of equivalents |
| Commission "charge once" logic errors | High | Medium | Comprehensive testing with multi-path scenarios; clear context storage semantics |
| Flow recalculation arithmetic errors | High | Medium | Unit tests with known input/output examples; validate against manual calculations |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Condition changes causing payment failures | Medium | Low | Graceful handling with path recreation; extensive error logging |

## Testing Strategy
### Testing Approach (Unit-Only)
- All testing is unit-only (no integration/E2E tests)
- Tests are built and executed exclusively in `build-tests`
- Use real objects following ExchangePathsManagerTest pattern
- Use TestEnvironment helper classes for consistent test setup
- Test files located in `tests/unit/transactions/` subdirectory

### Testing Best Practices
- **Real Objects Over Mocks**: Use real instances of ExchangeRatesManager, CommissionsManager, etc.
- **TestEnvironment Helpers**: Create helper classes for test initialization
- **Exception Testing**: Test that methods throw correct exceptions using EXPECT_THROW
- **Edge Case Coverage**: Test boundary values, missing data, condition removal scenarios
- **Parameterized Tests**: Use for testing different condition combinations

#### Unit Tests: New/Modified Components

**1. CoordinatorReservationRequestMessage Tests**:
- Message correctly serializes/deserializes with expectedExchangeRate
- Message correctly serializes/deserializes with expectedCommission
- Message correctly serializes/deserializes with both fields empty
- Cannot have both fields populated simultaneously (coordinator-side validation)
- Backward compatibility with messages without optional fields

**1a. IntermediateNode Protocol Violation Tests**:
- **Test 0: Both exchange rate and commission in request - protocol violation**
  - Setup: CoordinatorReservationRequestMessage with both expectedExchangeRate and expectedCommission populated
  - Expected: Intermediate node rejects with Rejected status (not RejectedDueConditionsChanged)
  - Error log: "Protocol violation: both exchange rate and commission provided in request"
  - No further processing occurs

**2. CoordinatorReservationResponseMessage Tests**:
- Message correctly serializes/deserializes with actualExchangeRate
- Message correctly serializes/deserializes with actualCommission
- Message correctly serializes/deserializes with both fields empty
- Backward compatibility with messages without optional fields

**3. IntermediateNode Exchange Rate Validation Tests**:
- **Test 1: Exchange rate matches - validation passes**
  - Setup: Expected rate = actual rate
  - Expected: Validation passes, rate stored in context
- **Test 2: Exchange rate mismatch - rejection**
  - Setup: Expected rate ≠ actual rate
  - Expected: RejectedDueConditionsChanged sent, actual rate in response, incoming reservation dropped
- **Test 3: Exchange rate from context - validation**
  - Setup: Rate already in context (second path)
  - Expected: Uses context rate for validation
- **Test 4: Exchange rate not found - rejection**
  - Setup: Expected rate provided but not in ExchangeRatesManager
  - Expected: RejectedDueConditionsChanged sent with empty rate
- **Test 5: Exchange rate exists but not expected - rejection**
  - Setup: Node has rate but coordinator didn't send it
  - Expected: RejectedDueConditionsChanged sent with actual rate

**4. IntermediateNode Commission Validation Tests**:
- **Test 6: Commission matches - validation passes**
  - Setup: Expected commission = actual commission
  - Expected: Validation passes, commission stored in context
- **Test 7: Commission mismatch - rejection**
  - Setup: Expected commission ≠ actual commission
  - Expected: RejectedDueConditionsChanged sent, actual commission in response, incoming reservation dropped
- **Test 8: Commission from context - validation**
  - Setup: Commission already in context (second path)
  - Expected: Uses context commission for validation
- **Test 9: Commission removed - rejection**
  - Setup: Expected commission but node no longer has it
  - Expected: RejectedDueConditionsChanged sent with commission = 0
- **Test 10: Commission exists but not expected - rejection**
  - Setup: Node has commission but coordinator didn't send it
  - Expected: RejectedDueConditionsChanged sent with actual commission

**5. IntermediateNode Context Storage Tests**:
- **Test 11: Exchange rate stored on first path**
  - Setup: First CoordinatorReservationRequestMessage with rate
  - Expected: Rate stored in mContextExchangeRates
- **Test 12: Commission stored on first path**
  - Setup: First CoordinatorReservationRequestMessage with commission
  - Expected: Commission stored in mContextCommissions
- **Test 13: Context not overwritten on subsequent paths**
  - Setup: Second path with different rate/commission
  - Expected: Context unchanged, validation uses original context values
- **Test 14: Transaction completion blocked by context**
  - Setup: No reservations but context has rates/commissions
  - Expected: Transaction does not complete
- **Test 15: Transaction completes when all clear**
  - Setup: No reservations, no context rates, no context commissions
  - Expected: Transaction completes

**6. IntermediateNode Outgoing/Incoming Reservation Validation Tests**:
- **Test 16: Same equivalent without commission - validation**
  - Setup: Incoming = outgoing = 100, no commission
  - Expected: Validation passes
- **Test 17: Same equivalent with commission - validation**
  - Setup: Incoming = 110, outgoing = 100, commission = 10
  - Expected: Validation passes
- **Test 18: Same equivalent with commission - mismatch**
  - Setup: Incoming = 110, outgoing = 105, commission = 10
  - Expected: Validation fails, Rejected sent
- **Test 19: Different equivalents with exchange - validation**
  - Setup: Incoming = 100 (eq 1), outgoing = 200 (eq 2), rate = 2.0
  - Expected: Validation passes
- **Test 20: Different equivalents with exchange - mismatch**
  - Setup: Incoming = 100 (eq 1), outgoing = 201 (eq 2), rate = 2.0
  - Expected: Validation fails, Rejected sent

**7. Coordinator Exchange Rate Change Handling Tests**:
- **Test 21: Receive RejectedDueConditionsChanged with new rate (higher)**
  - Setup: Path with rate 0.5, node returns rate 0.6
  - Expected: Path invalidated, ExchangeRatesManager updated, new path created with correct received_amount
- **Test 22: Receive RejectedDueConditionsChanged with new rate (lower)**
  - Setup: Path with rate 0.5, node returns rate 0.4
  - Expected: Path invalidated, ExchangeRatesManager updated, new path created with correct received_amount
- **Test 23: New path flows recalculated correctly**
  - Setup: Condition change on path
  - Expected: New path has correct flows vector from calculateFlows()
- **Test 24: Subsequent paths updated with new rate**
  - Setup: Multiple paths containing same exchanger node
  - Expected: All subsequent paths recalculated with new rate
- **Test 25: Reservations dropped on invalidated path**
  - Setup: Path with existing reservations gets invalidated
  - Expected: dropReservationsOnPath() called, FinalPathExchangeConfigurationMessage sent

**8. Coordinator Commission Change Handling Tests**:
- **Test 26: Receive RejectedDueConditionsChanged with new commission (higher)**
  - Setup: Path with commission 5, node returns commission 7
  - Expected: Path invalidated, CommissionsManager updated, new path created
- **Test 27: Receive RejectedDueConditionsChanged with commission removed**
  - Setup: Path with commission 5, node returns commission 0
  - Expected: Path invalidated, CommissionsManager removes commission, new path created
- **Test 28: Subsequent paths updated with new commission**
  - Setup: Multiple paths containing same commission node
  - Expected: All subsequent paths recalculated with new commission

**9. Commission "Charge Once Per Payment" Tests**:
- **Test 29: Commission charged on first path**
  - Setup: Node participates in first path with commission
  - Expected: Commission stored in context, deducted from amount
- **Test 30: Commission not charged on second path**
  - Setup: Same node participates in second path, commission already in context
  - Expected: Commission not deducted again
- **Test 31: Different equivalent commission charged separately**
  - Setup: Node charges commission in eq 1 and eq 2 in different paths
  - Expected: Both commissions charged (different equivalents)

**10. Path Recalculation Tests**:
- **Test 32: calculateReceivedAmountWithUpdatedConditions - exchange rate**
  - Setup: Path with old rate, calculate with new rate
  - Expected: Correct received_amount calculated
- **Test 33: calculateReceivedAmountWithUpdatedConditions - commission**
  - Setup: Path with old commission, calculate with new commission
  - Expected: Correct received_amount calculated
- **Test 34: Flow recalculation via calculateFlows()**
  - Setup: New path with updated conditions
  - Expected: flows vector correctly populated

#### Regression Testing (Unit)
- Scope: Ensure changes don't break existing exchange payment execution
- Verify payments without condition changes work unchanged
- Validate single-equivalent payments remain unaffected
- Confirm path capacity adjustment (PRD 08) still works

#### Execution in CI/Locally
- Build tests in `build-tests` and run the produced binaries
- All unit tests must pass before PRD completion

### Quality Gates
- All unit tests pass in `build-tests`
- Condition change detection 100% accurate in test scenarios
- Path recalculation produces correct flows in all test cases
- Commission "charge once" logic correct in multi-path tests
- No regressions in existing exchange payment behavior
- Manager updates correctly reflected in test validation

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Feature addition (backward compatible enhancement)
- **Rollout Strategy**: Direct deployment; internal logic enhancement, protocol extensions backward compatible
- **Rollback Plan**: Revert to previous version if critical issues found; no data migration concerns

### Database Migrations
- No database migrations required (runtime logic only)

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: No external communication needed (internal enhancement)
- **Documentation Updates**: Update payment flow documentation with condition change handling

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**:
  - Condition change detection accuracy (target: 100% in unit tests)
  - Path recalculation correctness (target: 100% accurate flows)
  - Commission charging accuracy (target: 100% charge once per payment)
  - Payment success rate with condition changes (target: +20% vs no handling)
- **Leading Indicators**: Unit test pass rate, condition mismatch detection count
- **Baseline Values**: Current payment failure rate when conditions change
- **Target Values**:
  - 100% unit test pass rate
  - Zero payment failures due to unhandled condition changes
  - 100% correct manager updates
  - 100% correct commission charging in multi-path scenarios

### Monitoring Plan
- **New Dashboards/Alerts**: Not applicable (internal logic enhancement)
- **Enhanced Monitoring**: Extended logging for condition validation, path recalculation, manager updates
- **A/B Testing**: Not applicable

### Review Schedule
- **Daily**: Development progress and unit test status
- **Weekly**: Integration validation with manual scenarios
- **Post-Implementation Review**: Payment success rate analysis with condition changes

## Appendices
### Glossary
- **Condition Change**: Modification of exchange rate or commission during payment execution
- **Transaction Context**: Storage of exchange rates and commissions within intermediate node transaction for consistent validation
- **Charge Once Per Payment**: Commission semantics where a node charges its commission only once per payment, regardless of path count
- **Path Invalidation**: Marking a path as unusable due to condition changes
- **Path Recalculation**: Computing new flows and received_amount with updated exchange rates or commissions

### References
- [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
- [Exchange Payment Topology Collection PRD](07-exchange-payment-topology-collection.md)
- [Path Capacity Adjustment PRD](08-path-capacity-adjustment.md)
- [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
- [CoordinatorExchangePaymentTransaction Implementation](../../../src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h)
- [IntermediateNodeExchangePaymentTransaction Implementation](../../../src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h)

### Detailed Component Specifications

#### Modified Message Classes

##### CoordinatorReservationRequestMessage
- **Location**: `src/core/network/messages/payments/CoordinatorReservationRequestMessage.h/.cpp`
- **New Fields**:
  ```cpp
  optional<ExchangeRate> mExpectedExchangeRate;
  optional<TrustLineAmount> mExpectedCommission;
  ```
- **New Methods**:
  ```cpp
  const optional<ExchangeRate>& expectedExchangeRate() const;
  const optional<TrustLineAmount>& expectedCommission() const;
  ```
- **Serialization**: Include optional fields in message buffer
- **Backward Compatibility**: Messages without fields handled gracefully

##### CoordinatorReservationResponseMessage
- **Location**: `src/core/network/messages/payments/CoordinatorReservationResponseMessage.h/.cpp`
- **New Fields**:
  ```cpp
  optional<ExchangeRate> mActualExchangeRate;
  optional<TrustLineAmount> mActualCommission;
  ```
- **New Methods**:
  ```cpp
  const optional<pair<TrustLineAmount, int16_t>>& actualExchangeRate() const;
  const optional<TrustLineAmount>& actualCommission() const;
  ```
- **Serialization**: Include optional fields in message buffer

##### ResponseMessage
- **Location**: `src/core/network/messages/payments/base/ResponseMessage.h`
- **New Enum Value**:
  ```cpp
  enum OperationState {
      // Existing values...
      RejectedDueConditionsChanged = 12,
  };
  ```

#### Modified Transaction Classes

##### IntermediateNodeExchangePaymentTransaction
- **Location**: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h/.cpp`
- **New Fields**:
  ```cpp
  map<pair<SerializedEquivalent, SerializedEquivalent>, ExchangeRate> mContextExchangeRates;
  map<SerializedEquivalent, Commission> mContextCommissions;
  ```
- **Modified Methods**:
  - `runCoordinatorRequestProcessingStage()`: Add condition validation logic
  - `canCompleteTransaction()`: Add context checks
- **New Methods**:
  ```cpp
  TransactionResult::SharedConst sendRejectedDueConditionsChanged(
      optional<ExchangeRate> actualRate,
      optional<TrustLineAmount> actualCommission);

  void dropIncomingReservationOnPath(const PathID &pathID);
  ```

##### CoordinatorExchangePaymentTransaction
- **Location**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h/.cpp`
- **Modified Methods**:
  - `askRemoteNodeToApproveReservation()`: Populate expected exchange rate/commission in request
  - `askNeighborToApproveFurtherNodeReservation()`: Populate expected exchange rate/commission in request
  - `processRemoteNodeResponse()`: Add RejectedDueConditionsChanged handling
  - `processNeighborFurtherReservationResponse()`: Add RejectedDueConditionsChanged handling
- **New Methods**:
  ```cpp
  TrustLineAmount calculateReceivedAmountWithUpdatedConditions(
      OptimalPathResult *pathStats,
      const TrustLineAmount &inputFlow,
      const optional<ExchangeRate> &updatedRate,
      const optional<TrustLineAmount> &updatedCommission);
  ```

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-10-23 | Claude Code | Initial draft for exchange rate and commission change handling | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [08-path-capacity-adjustment.md](08-path-capacity-adjustment.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)
- **Payment Protocol**: [payment-protocol.md](../../../architecture/vtcpd/protocols/payment-protocol.md)
