# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Path Rebuilding on Inaccessible Nodes and Rejected Trust Lines
- **PRD ID**: 11
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.0
- **Date**: 2025-11-06
- **Author(s)**: Claude Code, based on Architect's requirements
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-11-06
- **Previous PRD**: [10-allowable-payment-amount-control.md](10-allowable-payment-amount-control.md)
- **Related Documents**:
  - [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
  - [Exchange Payment Topology Collection PRD](07-exchange-payment-topology-collection.md)
  - [Path Capacity Adjustment PRD](08-path-capacity-adjustment.md)
  - [Exchange Rate and Commission Change Handling PRD](09-exchange-rate-commission-change-handling.md)
  - [Allowable Payment Amount Control PRD](10-allowable-payment-amount-control.md)
  - [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
  - [Topology Collection Protocol](../../../architecture/vtcpd/protocols/topology-collection-protocol.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This PRD introduces automatic path rebuilding mechanism for multi-equivalent exchange payments when nodes become inaccessible or trust lines are rejected during reservation phase. Currently, when coordinator cannot reserve required amount and has no more unprocessed paths, the transaction fails with code 412 (Insufficient Funds). However, if some nodes were inaccessible (`mInaccessibleNodes`) or some trust lines were rejected (`mRejectedTrustLines`), the system should attempt to rebuild paths excluding these problematic elements, potentially finding alternative routes to complete the payment.

### Current project state
- Exchange Payment with Commissions (PRD 06) implements multi-equivalent payment execution with OR-Tools-based path calculation
- Exchange Payment Topology Collection (PRD 07) adds automatic topology collection for path building
- Path Capacity Adjustment (PRD 08) handles dynamic capacity changes during reservation
- Exchange Rate and Commission Change Handling (PRD 09) adapts to condition changes during payment
- Allowable Payment Amount Control (PRD 10) adds user spending limits
- Single-equivalent payments already have path rebuilding logic in `CoordinatorPaymentTransaction`
- Current implementation: Multi-equivalent payments fail when paths exhausted, even if alternative paths might exist

### This iteration's focus
- Implement `buildPathsAgain()` method for `CoordinatorExchangePaymentTransaction` using OR-Tools
- Update topology in `EquivalentsSubsystemsRouter` and `ExchangeRatesManager` to exclude inaccessible nodes and rejected trust lines
- Account for already reserved amounts in topology to ensure accurate capacity information
- Rebuild paths using `calculateMaxFlow()` with updated topology
- Validate that rebuilt paths have sufficient capacity for remaining required amount
- Add rebuilt paths to `mPathsStats` and continue reservation process
- Ensure sufficient capacity validation after path rebuilding in `tryProcessNextPath()`

### Connection to overall vision
This completes the resilience layer for multi-equivalent exchange payments, enabling automatic recovery from temporary network issues (offline nodes, temporarily unavailable trust lines). The system now maximizes payment success rate by dynamically adapting to network availability changes during payment execution, mirroring the proven approach from single-equivalent payments.

## Iteration Context
### Previous Iterations Summary
- **PRD 04**: Exchange Flow Calculation implements topology collection and OR-Tools-based max flow computation
- **PRD 05**: Payment Estimation provides bidirectional estimation using cached paths
- **PRD 06**: Exchange Payment with Commissions implements multi-equivalent payment execution with OR-Tools
- **PRD 07**: Exchange Payment Topology Collection adds automatic path collection
- **PRD 08**: Path Capacity Adjustment handles dynamic capacity changes during reservation
- **PRD 09**: Exchange Rate and Commission Change Handling adapts to condition changes during payment
- **PRD 10**: Allowable Payment Amount Control adds user spending limits
- **Completed Features**: Multi-equivalent payments, topology collection, OR-Tools path calculation, capacity adjustment, condition change handling, spending limits

### Lessons Learned
- Single-equivalent payments successfully use path rebuilding to recover from inaccessible nodes
- Network topology is dynamic - nodes may go offline or trust lines may become unavailable during payment
- Rebuilding paths without problematic elements often finds alternative routes
- OR-Tools `calculateMaxFlow()` requires current topology state in `EquivalentsSubsystemsRouter` and `ExchangeRatesManager`
- Already reserved amounts must be reflected in topology for accurate path calculation

### Current State Analysis
- **What's working well**: Multi-equivalent payments execute successfully when all nodes accessible
- **Pain points identified**:
  - Payments fail when nodes temporarily offline, even if alternative paths exist
  - Coordinator doesn't attempt path rebuilding after identifying inaccessible nodes
  - No mechanism to exclude rejected trust lines from subsequent path attempts
  - Already reserved amounts not considered when rebuilding paths
  - `buildPathsAgain()` method exists but is empty (skeleton from single-equivalent code)
- **Performance metrics**: Payment success rate lower than optimal due to lack of path rebuilding

## Problem Statement
### Background
During multi-equivalent exchange payment execution, coordinator reserves amounts on paths in sequence. During this process:

1. **Nodes become inaccessible**: Some intermediate nodes may not respond (offline, network issues), tracked in `mInaccessibleNodes`
2. **Trust lines rejected**: Some trust lines may reject reservation (insufficient capacity, state change), tracked in `mRejectedTrustLines`
3. **Paths exhausted**: Coordinator processes all paths from `mPathsStats`, but reserved amount still insufficient
4. **Current behavior**: Transaction fails with code 412 (Insufficient Funds)

However, the coordinator has valuable information about network problems:
- List of inaccessible nodes (`mInaccessibleNodes`)
- List of rejected trust lines (`mRejectedTrustLines`)
- Amount already reserved on successful paths

This information can be used to rebuild paths excluding problematic elements, potentially finding alternative routes to complete the payment.

### Problem Description
**Who is affected**: Users initiating exchange payments through networks with temporary availability issues

**When and where**:
- During reservation stage: when coordinator exhausts paths but hasn't reserved sufficient amount
- In `tryProcessNextPath()`: when `NotFoundError` thrown (no more unprocessed paths)
- Before returning `resultInsufficientFundsError()`: when should attempt path rebuilding

**Current limitations**:
- `buildPathsAgain()` method exists but is empty (no implementation)
- No mechanism to update topology with inaccessible nodes and rejected trust lines
- No mechanism to reflect already reserved amounts in topology
- No validation that rebuilt paths have sufficient capacity
- `tryProcessNextPath()` doesn't check if rebuilt paths are sufficient

### Impact of not solving this problem
- Lower payment success rate due to temporary network issues
- Users unable to complete valid payments when alternative paths exist
- Wasted topology collection and path calculation effort from earlier stages
- Negative user experience when payments fail unnecessarily

### Success Metrics
**Primary KPIs**:
- Path rebuilding success rate (target: >50% when inaccessible nodes/rejected trust lines exist)
- Payment success rate improvement (target: +10-20% in networks with availability issues)
- Correct topology update (target: 100% accurate exclusion of problematic elements)
- Correct capacity accounting (target: 100% accurate reflection of reserved amounts)

**Target Values**:
- 100% correct topology modification (inaccessible nodes and rejected trust lines excluded)
- 100% correct capacity reduction for already reserved amounts
- >50% of rebuilding attempts produce new valid paths
- 0 incorrect path calculations due to topology inconsistencies

## Goals
The primary goals of this iteration are to enable automatic path rebuilding for multi-equivalent exchange payments when network elements become unavailable.

*   **Goal 1: Implement topology update to exclude inaccessible nodes and rejected trust lines.**
    *   **Description:** Modify topology in `EquivalentsSubsystemsRouter` to remove inaccessible nodes (and their trust lines) and to remove specific rejected trust lines before rebuilding paths.
    *   **Success Metric:** Topology correctly reflects unavailable network elements; subsequent path calculations exclude these elements.

*   **Goal 2: Account for already reserved amounts in topology.**
    *   **Description:** Reduce trust line capacities in topology by amounts already reserved for current payment to ensure accurate available capacity for new paths.
    *   **Success Metric:** Path capacity calculations accurate; new paths don't conflict with existing reservations.

*   **Goal 3: Implement `buildPathsAgain()` method using OR-Tools.**
    *   **Description:** Complete implementation of `buildPathsAgain()` that updates topology, calls `calculateMaxFlow()`, and adds resulting paths to `mPathsStats`.
    *   **Success Metric:** Method successfully rebuilds paths using OR-Tools; new paths added to `mPathsStats` for processing.

*   **Goal 4: Validate sufficient capacity after path rebuilding.**
    *   **Description:** After rebuilding paths, verify that total capacity of all paths (existing + new) is sufficient to reserve remaining required amount.
    *   **Success Metric:** Transaction only continues if sufficient capacity exists; fails with clear error if capacity insufficient.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **Topology update mechanism for inaccessible nodes**: Remove nodes from topology and all associated trust lines
2. **Topology update mechanism for rejected trust lines**: Remove specific trust lines from topology
3. **Reserved amounts reflection in topology**: Reduce trust line capacities by already reserved amounts
4. **Complete `buildPathsAgain()` implementation**: Update topology, call `calculateMaxFlow()`, add paths to `mPathsStats`
5. **Sufficient capacity validation**: Check if rebuilt paths provide enough capacity for remaining amount
6. **Integration in `tryProcessNextPath()`**: Add capacity check after path rebuilding

#### Technical Infrastructure
- Topology modification methods in `EquivalentsSubsystemsRouter`
- Trust line capacity reduction based on existing reservations
- OR-Tools `calculateMaxFlow()` invocation with updated topology
- Path validation and addition to `mPathsStats`

#### Integration Points
- Integration with existing topology management in `EquivalentsSubsystemsRouter`
- Integration with `ExchangeRatesManager` for exchange rate data
- Integration with `ExchangePathsManager` for `calculateMaxFlow()`
- Integration with existing reservation tracking
- Integration with `tryProcessNextPath()` flow

### Explicitly Out of Scope
- Path rebuilding for single-equivalent payments (already implemented)
- Predictive path pre-calculation based on historical failures
- Machine learning for path reliability scoring
- Multi-attempt rebuilding with different strategies
- Path rebuilding at other transaction stages (only during reservation exhaustion)

### Dependencies from Previous Iterations
- **PRD 06**: `CoordinatorExchangePaymentTransaction` implementation, `mPathsStats` structure
- **PRD 07**: Topology collection mechanisms, `fillTopology()` and `fillRates()` methods
- **PRD 04**: `calculateMaxFlow()` implementation in `ExchangePathsManager`
- Existing `mInaccessibleNodes` and `mRejectedTrustLines` tracking

### Future Roadmap Impact
This iteration establishes foundation for:
- **Multi-strategy rebuilding**: Try multiple rebuilding strategies with different parameters
- **Predictive exclusions**: Exclude historically unreliable nodes proactively
- **Partial path reuse**: Reuse successful path segments when rebuilding
- **Learning mechanisms**: Track rebuilding success patterns for optimization

## User Stories & Requirements

### User Personas
#### Primary User: Payment Initiator in Dynamic Network
- **Role**: User initiating exchange payment in network with varying node availability
- **Goals**: Complete payment successfully even when some network elements unavailable; maximize payment success rate
- **Pain Points**: Payments fail when temporary issues affect individual nodes; no automatic recovery from network problems
- **Technical Proficiency**: Intermediate

#### Secondary User: Payment Receiver
- **Role**: User receiving exchange payment
- **Goals**: Receive payments reliably; minimize payment failures
- **Pain Points**: Payments fail unnecessarily when alternative paths exist but aren't explored
- **Technical Proficiency**: Intermediate

### Functional Requirements
#### New Features for This Iteration

1. **Topology Update: Exclude Inaccessible Nodes**
   - **Description**: Remove inaccessible nodes and all their trust lines from topology before rebuilding paths
   - **User Story**: As a coordinator, I need to exclude offline nodes from path calculation to find alternative routes
   - **Rationale**: Inaccessible nodes cannot participate in payment, so including them in topology wastes computation and produces invalid paths
   - **Builds Upon**: Existing topology management in `EquivalentsSubsystemsRouter`
   - **Acceptance Criteria**:
     - Iterate through `mInaccessibleNodes` set
     - For each inaccessible node:
       - Get ContractorID from node address via `mEquivalentsSubsystemsRouter->getOrCreateParticipantID(address)`
       - For each equivalent in `mCommand->exchangeEquivalents()`:
         - Get topology manager: `mEquivalentsSubsystemsRouter->topologyTrustLineManager(equivalent)`
         - Remove all trust lines where node is source: iterate outgoing trust lines and call `removeTrustLine(nodeID, targetID)`
         - Remove all trust lines where node is target: iterate incoming trust lines and call `removeTrustLine(sourceID, nodeID)`
     - Log each removed node and count of removed trust lines
     - Implementation in `buildPathsAgain()` method
   - **Priority**: High
   - **Dependencies**: `EquivalentsSubsystemsRouter` topology management APIs

2. **Topology Update: Exclude Rejected Trust Lines**
   - **Description**: Remove specific rejected trust lines from topology before rebuilding paths
   - **User Story**: As a coordinator, I need to exclude rejected trust lines from path calculation to avoid repeatedly attempting failed reservations
   - **Rationale**: Rejected trust lines indicate current unavailability; retrying same trust lines wastes time and resources
   - **Builds Upon**: Existing topology management in `EquivalentsSubsystemsRouter`
   - **Acceptance Criteria**:
     - Iterate through `mRejectedTrustLines` set (each element is `pair<BaseAddress::Shared, BaseAddress::Shared>` representing source and target)
     - For each rejected trust line:
       - Get ContractorIDs for source and target via `mEquivalentsSubsystemsRouter->getOrCreateParticipantID()`
       - For each equivalent in `mCommand->exchangeEquivalents()`:
         - Get topology manager: `mEquivalentsSubsystemsRouter->topologyTrustLineManager(equivalent)`
         - Call `removeTrustLine(sourceID, targetID)` to remove the specific trust line
     - Log each removed trust line (source -> target)
     - Implementation in `buildPathsAgain()` method
   - **Priority**: High
   - **Dependencies**: `EquivalentsSubsystemsRouter` topology management APIs

3. **Topology Update: Reflect Already Reserved Amounts**
   - **Description**: Відобразити у топології вже зафіксовані резерви, щоб нові шляхи не перевикористовували ту саму пропускну здатність
   - **User Story**: Як координатор, я хочу, щоб резерви, зроблені на попередніх шляхах, зменшували доступний потік при перебудові
   - **Rationale**: `mReservations` містить лише резерви координатора, тож для коректної картини потрібно покладатися на фінальну конфігурацію резервів усіх нод
   - **Builds Upon**: Наявна структура `mNodesFinalAmountsConfiguration` у `CoordinatorExchangePaymentTransaction`
   - **Acceptance Criteria**:
     - Під час `buildPathsAgain()` зчитати `mNodesFinalAmountsConfiguration` та скласти резерви лише за напрямком `Incoming`
     - Для кожного елементу отримати ідентифікатори відправника та отримувача через `mEquivalentsSubsystemsRouter->getOrCreateParticipantID`
     - Використати `TopologyTrustLinesManager::addUsedAmount()` для часткових резервів та `TopologyTrustLinesManager::makeFullyUsed()` якщо заблоковано всю місткість
     - Працювати окремо для кожного еквівалента з `mCommand->exchangeEquivalents()` і еквівалента отримувача
     - Журнальнувати кількість застосованих резервів і загальну суму скорочення потоку
   - **Priority**: High
   - **Dependencies**: `mNodesFinalAmountsConfiguration`, API `TopologyTrustLinesManager`

4. **Topology Manager / Router API: розширення для модифікації топології**
   - **Description**: Додати в `TopologyTrustLinesManager` метод `removeTrustLine(sourceID, targetID)` та надати у `EquivalentsSubsystemsRouter` можливість перерахувати всіх відомих учасників (наприклад, `participantsIDs()`)
   - **User Story**: Як координатор, я хочу фізично прибрати лінії довіри з топології, щоб OR-Tools не будував по ним шляхи
   - **Rationale**: чинний менеджер дозволяє лише змінювати використані обсяги; для повного виключення потрібно видалення
   - **Acceptance Criteria**:
     - `removeTrustLine(sourceID, targetID)` видаляє запис у `msTrustLines` для вказаної пари та прибирає відповідні покажчики
     - При видаленні останньої лінії для ноди структура очищується без витоків пам’яті
     - `participantsIDs()` (або аналогічний метод) повертає усі ContractorID, що вже зареєстровані в роутері, без побічних ефектів
     - Обидві функції безпечні при повторних викликах і використовуються в `buildPathsAgain()`
   - **Priority**: High
   - **Dependencies**: Існуюча структура зберігання `TopologyTrustLineWithPtr`

5. **Complete `buildPathsAgain()` Implementation**
   - **Description**: Implement complete path rebuilding logic using OR-Tools-based `calculateMaxFlow()`
   - **User Story**: As a coordinator, I need to automatically rebuild paths when initial paths insufficient due to network issues
   - **Rationale**: Enables payment completion when alternative paths exist after excluding problematic elements
   - **Builds Upon**: Empty `buildPathsAgain()` skeleton, existing `calculateMaxFlow()` infrastructure
   - **Acceptance Criteria**:
     - Method signature: `void CoordinatorExchangePaymentTransaction::buildPathsAgain()`
     - Implementation steps:
       1. Log start: `debug() << "buildPathsAgain: attempting path rebuilding";`
       2. Update topology: exclude inaccessible nodes (Requirement 1)
       3. Update topology: exclude rejected trust lines (Requirement 2)
       4. Update topology: reflect reserved amounts (Requirement 3)
      5. Виклик `calculateMaxFlow()` із використанням ContractorID:
          ```cpp
          auto maxFlowResult = mExchangePathsManager->calculateMaxFlow(
              mContractorID,
              mCommand->equivalent(),
              mCommand->exchangeEquivalents(),
              TopologyTrustLinesManager::kCurrentNodeID,
              kMaxPathLength);
          ```
      6. Log result: `info() << "Rebuilt " << maxFlowResult.optimalPaths.size() << " new paths, max flow: " << maxFlowResult.maxFlow;`
      7. For each path in `maxFlowResult.optimalPaths`:
          - Generate new PathID: `PathID newPathID = generateNextPathID();`
          - Add to `mPathsStats`: `mPathsStats[newPathID] = std::make_unique<OptimalPathResult>(path);`
          - Log: `debug() << "Added rebuilt path " << newPathID << " with capacity " << path.received_amount;`
       8. If no new paths: `warning() << "Path rebuilding produced no new paths";`
     - Method doesn't return value (void)
     - Method modifies topology (side effect)
     - Method adds paths to `mPathsStats` (side effect)
     - Located in: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`
   - **Priority**: High
   - **Dependencies**: Requirements 1-4, `ExchangePathsManager::calculateMaxFlow()`, topology management

6. **Calculate Total Path Capacity**
   - **Description**: Helper method to calculate total capacity of all paths in `mPathsStats` for remaining receive amount
   - **User Story**: As a coordinator, I need to know if rebuilt paths provide sufficient capacity to complete payment
   - **Rationale**: Prevents wasting time on reservation attempts when capacity mathematically insufficient
   - **Builds Upon**: Existing `mPathsStats` structure
   - **Acceptance Criteria**:
     - Method signature: `TrustLineAmount calculateTotalPathCapacityForReceive() const;` (або еквівалентна, якщо буде потрібен доступ до неконстантних полів через helper)
     - Обчислює `remainingReceive = mAmount - calculateTotalReservedAmount()` (передбачає появу const-безпечного доступу до зарезервованої суми)
     - Сумує `received_amount` лише для шляхів, де `pathID > mCurrentAmountReservingPathIdentifier`, `path->isValid()` і `!path->isLastIntermediateNodeProcessed()`
     - Журнальнування показує окремо пропущені шляхи та підсумкову суму
     - Повертає загальну доступну пропускну здатність для наступних резервів
   - **Priority**: High
   - **Dependencies**: Існуючі структури `mPathsStats`, механізм обчислення зарезервованих сум

7. **Sufficient Capacity Validation After Rebuilding**
   - **Description**: Validate that total path capacity (existing + rebuilt) is sufficient for remaining required amount
   - **User Story**: As a coordinator, I need to fail fast with clear error when rebuilt paths still insufficient
   - **Rationale**: Avoids wasting time on reservation attempts when capacity mathematically insufficient
   - **Builds Upon**: Requirement 6 (capacity calculation method)
   - **Acceptance Criteria**:
     - Location: In `tryProcessNextPath()`, in `catch (NotFoundError &e)` block, after `buildPathsAgain()` call
     - After `buildPathsAgain()` completes and `mPathsStats.size() > countPathsBeforeBuilding`:
       ```cpp
       // Check if rebuilt paths provide sufficient capacity
       TrustLineAmount remainingReceive = mAmount - calculateTotalReservedAmount();
       TrustLineAmount totalCapacity = calculateTotalPathCapacityForReceive();

       if (totalCapacity < remainingReceive) {
           warning() << "Rebuilt paths insufficient: need " << remainingReceive
                     << " but total capacity only " << totalCapacity;
           reject("Rebuilt paths have insufficient capacity");
           return resultInsufficientFundsError();
       }

       info() << "Rebuilt paths have sufficient capacity: " << totalCapacity
              << " for remaining " << remainingReceive;
       ```
     - Validation happens before continuing to reservation
     - Clear logging of capacity vs. requirement
     - Transaction fails cleanly if capacity insufficient
   - **Priority**: High
   - **Dependencies**: Requirement 5, existing `tryProcessNextPath()` flow

#### Enhancements to Existing Features

1. **`tryProcessNextPath()` Exception Handling**
   - **Current State**: Has try-catch with `buildPathsAgain()` call but no capacity validation
   - **Proposed Changes**: Add capacity validation after path rebuilding (Requirement 6)
   - **Impact Assessment**: Prevents unnecessary reservation attempts; improves error clarity
   - **Migration Strategy**: Pure addition after existing `buildPathsAgain()` call

2. **Topology State Management**
   - **Current State**: Topology populated during collection phase and remains static
   - **Proposed Changes**: Dynamic modification to reflect network availability and reservations
   - **Impact Assessment**: Enables accurate path recalculation; topology reflects current network state
   - **Migration Strategy**: Modifications local to `buildPathsAgain()`, doesn't affect other uses

### Non-Functional Requirements
#### Performance
- Topology update overhead < 100ms for typical network size (< 100 nodes)
- `buildPathsAgain()` execution < 5 seconds for typical networks
- Path capacity calculation < 50ms
- No significant memory overhead (topology modifications in-place where possible)

#### Security
- No security implications (internal optimization)
- No exposure of network topology to external parties
- Existing security mechanisms unaffected

#### Scalability
- Support networks up to 500 nodes
- Support up to 50 inaccessible nodes
- Support up to 100 rejected trust lines
- Efficient topology iteration (O(n) where n = node count)

#### Reliability
- Topology modifications never crash transaction (graceful degradation)
- Path rebuilding failures don't corrupt transaction state
- Transaction state consistent after rebuilding attempt
- Proper error reporting on rebuilding failures

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Multi-equivalent payments fail when paths exhausted, no automatic recovery
- **Proposed Changes**: Add topology update mechanisms and path rebuilding using OR-Tools; validate capacity before continuing
- **Backwards Compatibility**: Fully backward compatible (enhancement to existing flow)
- **Migration Requirements**: None (pure addition to existing transaction)

### Technology Stack Updates
#### New Technologies/Libraries
- No new external libraries (uses existing OR-Tools, Boost)
- Reuses existing infrastructure: topology management, path calculation, reservation tracking

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- Topology modification integrated into `buildPathsAgain()`
- Capacity calculation integrated into `tryProcessNextPath()`
- Path rebuilding integrated into reservation failure recovery

#### Modified Integrations
- `CoordinatorExchangePaymentTransaction::buildPathsAgain()` implementation completed
- `CoordinatorExchangePaymentTransaction::tryProcessNextPath()` capacity validation added
- `EquivalentsSubsystemsRouter` topology used for dynamic modifications

### Data Requirements
#### Data Models

No new data models. Enhanced usage of existing structures:

##### Existing `mInaccessibleNodes`
**Purpose**: Track nodes that didn't respond during reservation

**Type**: `set<BaseAddress::Shared>`

**Usage in this iteration**:
- Source of nodes to exclude from topology
- Iterated during topology update in `buildPathsAgain()`

##### Existing `mRejectedTrustLines`
**Purpose**: Track trust lines that rejected reservation

**Type**: `set<pair<BaseAddress::Shared, BaseAddress::Shared>>`

**Usage in this iteration**:
- Source of trust lines to exclude from topology
- Iterated during topology update in `buildPathsAgain()`

##### Existing `mNodesFinalAmountsConfiguration`
**Purpose**: Зберігає фінальну конфігурацію резервів для кожного учасника (Incoming/Outgoing) по шляху

**Type**: `map<string /*node address*/, vector<PathReservation>>`

**Usage in this iteration**:
- Джерело даних для застосування існуючих резервів у топології (беремо лише `Incoming`)
- Використовується разом із `mPathsStats` для визначення напрямку ребра

##### Existing `mPathsStats`
**Purpose**: Store all paths available for reservation

**Type**: `map<PathID, unique_ptr<OptimalPathResult>>`

**Usage in this iteration**:
- Destination for newly rebuilt paths
- Source for capacity calculation (`received_amount`, валідність, позиція вузлів)
- Modified by `buildPathsAgain()` (new paths added)

#### Data Storage
- No persistent storage changes
- All data modifications runtime-only
- Topology modifications transient (local to transaction)

#### Data Migration
- No migration needed (uses existing structures)

### Algorithm Specifications

#### Algorithm 1: Topology Update - Exclude Inaccessible Nodes

**Purpose**: Remove inaccessible nodes and their trust lines from topology

**Location**: `buildPathsAgain()` in `CoordinatorExchangePaymentTransaction`

**Algorithm**:
```cpp
// Step 1: Exclude inaccessible nodes
if (!mInaccessibleNodes.empty()) {
    debug() << "Excluding " << mInaccessibleNodes.size() << " inaccessible nodes from topology";

    vector<SerializedEquivalent> equivalentsToProcess = mCommand->exchangeEquivalents();
    equivalentsToProcess.push_back(mEquivalent);

    for (const auto &address : mInaccessibleNodes) {
        const auto nodeID = mEquivalentsSubsystemsRouter->getOrCreateParticipantID(address);

        for (const auto &equivalent : equivalentsToProcess) {
            auto topologyManager = mEquivalentsSubsystemsRouter->topologyTrustLineManager(equivalent);

            // Remove всі вихідні ребра
            size_t removedOutgoing = 0;
            for (auto *linePtr : topologyManager->trustLinePtrsSet(nodeID)) {
                topologyManager->removeTrustLine(nodeID, linePtr->topologyTrustLine()->targetID());
                ++removedOutgoing;
            }

            // Видалити входи: проходимо по всіх учасниках з кешу маршрутизатора
            size_t removedIncoming = 0;
            for (const auto participantID : mEquivalentsSubsystemsRouter->participantsIDs()) {
                if (participantID == nodeID) {
                    continue;
                }
                for (auto *linePtr : topologyManager->trustLinePtrsSet(participantID)) {
                    if (linePtr->topologyTrustLine()->targetID() == nodeID) {
                        topologyManager->removeTrustLine(participantID, nodeID);
                        ++removedIncoming;
                        break;
                    }
                }
            }

            debug() << "Node " << nodeID << " removed from equivalent " << equivalent
                    << ": " << removedOutgoing << " outgoing, " << removedIncoming
                    << " incoming trust lines";
        }
    }
}
```

**Key Points**:
- Removes node completely from topology by removing all connected trust lines
- Processes all exchange equivalents
- Logs removal for debugging
- Handles both outgoing and incoming trust lines

#### Algorithm 2: Topology Update - Exclude Rejected Trust Lines

**Purpose**: Remove specific rejected trust lines from topology

**Location**: `buildPathsAgain()` in `CoordinatorExchangePaymentTransaction`

**Algorithm**:
```cpp
// Step 2: Exclude rejected trust lines
if (!mRejectedTrustLines.empty()) {
    debug() << "Excluding " << mRejectedTrustLines.size() << " rejected trust lines";

    for (const auto &line : mRejectedTrustLines) {
        const auto sourceID = mEquivalentsSubsystemsRouter->getOrCreateParticipantID(line.first);
        const auto targetID = mEquivalentsSubsystemsRouter->getOrCreateParticipantID(line.second);

        for (const auto &equivalent : equivalentsToProcess) {
            auto topologyManager = mEquivalentsSubsystemsRouter->topologyTrustLineManager(equivalent);

            topologyManager->removeTrustLine(sourceID, targetID);
            debug() << "Rejected trust line " << sourceID << " -> " << targetID
                    << " removed from equivalent " << equivalent;
        }
    }
}
```

**Key Points**:
- Removes specific trust lines (directional: source -> target)
- Processes all exchange equivalents
- Checks existence before removal
- Logs each removal

#### Algorithm 3: Topology Update - Reflect Reserved Amounts

**Purpose**: Reduce trust line capacities by already reserved amounts

**Location**: `buildPathsAgain()` in `CoordinatorExchangePaymentTransaction`

**Algorithm**:
```cpp
// Step 3: Apply already established reservations (incoming only)
debug() << "Applying reservations from mNodesFinalAmountsConfiguration";
size_t processedReservations = 0;

for (const auto &nodeEntry : mNodesFinalAmountsConfiguration) {
    const auto &nodeAddressKey = nodeEntry.first;
    const auto targetPaymentNode = mPaymentNodesIds.at(nodeAddressKey);
    const auto targetContractor = mPaymentParticipants.at(targetPaymentNode);
    const auto targetID = mEquivalentsSubsystemsRouter->getOrCreateParticipantID(
        targetContractor->mainAddress());

    for (const auto &reservation : nodeEntry.second) {
        if (reservation.direction != PathReservation::Incoming) {
            continue;
        }

        auto pathIt = mPathsStats.find(reservation.pathID);
        if (pathIt == mPathsStats.end()) {
            continue;
        }

        auto *pathStats = pathIt->second.get();
        const auto position = pathStats->path().positionOfNode(targetContractor->mainAddress());
        if (position <= 0) {
            continue; // немає попереднього вузла
        }

        const auto &previousNodeAddress = pathStats->path().nodes[position - 1];
        const auto sourceID = mEquivalentsSubsystemsRouter->getOrCreateParticipantID(previousNodeAddress);

        auto topologyManager = mEquivalentsSubsystemsRouter->topologyTrustLineManager(reservation.equivalent);

        const auto reservedAmount = *reservation.amount;
        const auto maxCapacity = pathStats->flows[position].first; // amount, який очікує вузол на своєму вході

        if (reservedAmount >= maxCapacity) {
            topologyManager->makeFullyUsed(sourceID, targetID);
        } else {
            topologyManager->addUsedAmount(sourceID, targetID, reservedAmount);
        }

        ++processedReservations;
    }
}

info() << "Reservations applied: " << processedReservations;
```

**Key Points**:
- Iterates through all existing reservations
- Reduces capacity or removes trust lines
- Handles full consumption (removal) vs. partial consumption (capacity update)
- Only processes outgoing reservations (coordinator's perspective)
- Logs each capacity change

#### Algorithm 4: Complete Path Rebuilding

**Purpose**: Orchestrate topology update and path recalculation

**Location**: `buildPathsAgain()` in `CoordinatorExchangePaymentTransaction`

**Algorithm**:
```cpp
void CoordinatorExchangePaymentTransaction::buildPathsAgain()
{
    debug() << "buildPathsAgain: attempting path rebuilding";
    auto startTime = utc_now();

    // Step 1: Exclude inaccessible nodes (Algorithm 1)
    // [Implementation from Algorithm 1]

    // Step 2: Exclude rejected trust lines (Algorithm 2)
    // [Implementation from Algorithm 2]

    // Step 3: Reflect reserved amounts (Algorithm 3)
    // [Implementation from Algorithm 3]

    // Step 4: Call calculateMaxFlow with updated topology
    try {
        info() << "Calling calculateMaxFlow with updated topology";

        auto maxFlowResult = mExchangePathsManager->calculateMaxFlow(
            mContractor->mainAddress(),
            mCommand->equivalent(),
            mCommand->exchangeEquivalents(),
            mContractorsManager->selfContractor()->ownAddresses().at(0)->uuid(),
            kMaxPathLength);

        info() << "Rebuilt " << maxFlowResult.optimalPaths.size()
               << " new paths, max flow: " << maxFlowResult.maxFlow;

        // Step 5: Add rebuilt paths to mPathsStats
        for (auto& rebuiltPath : maxFlowResult.optimalPaths) {
            PathID newPathID = generateNextPathID();
            mPathsStats[newPathID] = make_unique<OptimalPathResult>(std::move(rebuiltPath));

            debug() << "Added rebuilt path " << newPathID
                    << " with capacity " << mPathsStats[newPathID]->path().maxFlow;
        }

        if (maxFlowResult.optimalPaths.empty()) {
            warning() << "Path rebuilding produced no new paths";
        }

    } catch (const exception& e) {
        warning() << "Path rebuilding failed: " << e.what();
    }

    debug() << "buildPathsAgain method time: " << utc_now() - startTime;
}
```

**Key Points**:
- Orchestrates all topology updates sequentially
- Calls `calculateMaxFlow()` with modified topology
- Adds new paths to `mPathsStats` for processing
- Exception-safe (catches and logs failures)
- Measures execution time

#### Algorithm 5: Total Path Capacity Calculation

**Purpose**: Calculate total available capacity for remaining receive amount

**Location**: New method in `CoordinatorExchangePaymentTransaction`

**Algorithm**:
```cpp
TrustLineAmount CoordinatorExchangePaymentTransaction::calculateTotalPathCapacityForReceive() const
{
    const TrustLineAmount alreadyReserved = calculateTotalReservedAmount();
    const TrustLineAmount remainingNeeded = mAmount - alreadyReserved;

    debug() << "Calculating capacity: reserved=" << alreadyReserved
            << ", remaining=" << remainingNeeded
            << ", processedPathID=" << mCurrentAmountReservingPathIdentifier;

    TrustLineAmount totalCapacity = TrustLineAmount(0);

    for (const auto &entry : mPathsStats) {
        const PathID pathID = entry.first;
        const auto *pathStats = entry.second.get();

        if (pathID <= mCurrentAmountReservingPathIdentifier) {
            continue; // шлях уже оброблено або в роботі
        }

        if (!pathStats->isValid()) {
            continue;
        }

        if (pathStats->isLastIntermediateNodeProcessed()) {
            continue; // шлях уже вичерпано під час резервів
        }

        totalCapacity = totalCapacity + pathStats->received_amount;
        debug() << "Path " << pathID << " adds capacity " << pathStats->received_amount;
    }

    info() << "Total new capacity: " << totalCapacity
           << " against remaining " << remainingNeeded;

    return totalCapacity;
}
```

**Key Points**:
- Використовує `mCurrentAmountReservingPathIdentifier` як межу між вже опрацьованими й новими шляхами
- Ігнорує невалідні та повністю пройдені шляхи (`isValid()`, `isLastIntermediateNodeProcessed()`)
- Сумує `received_amount` як доступну пропускну здатність для отримувача
- Повідомляє в журналах про пропущені шляхи та підсумкову суму

#### Algorithm 6: Capacity Validation in `tryProcessNextPath()`

**Purpose**: Validate sufficient capacity after path rebuilding

**Location**: `tryProcessNextPath()`, in `catch (NotFoundError &e)` block

**Integration Point**: After existing code at line 2650-2663 in CoordinatorExchangePaymentTransaction.cpp

**Algorithm**:
```cpp
// In tryProcessNextPath(), in catch (NotFoundError &e) block:

if (mInaccessibleNodes.size() != mPreviousInaccessibleNodesCount ||
        mRejectedTrustLines.size() != mPreviousRejectedTrustLinesCount) {
    auto countPathsBeforeBuilding = mPathsStats.size();
    buildPathsAgain();

    if (mPathsStats.size() > countPathsBeforeBuilding) {
        debug() << "New paths was built " << to_string(mPathsStats.size() - countPathsBeforeBuilding);

        // NEW: Validate sufficient capacity
        TrustLineAmount remainingReceive = mAmount - calculateTotalReservedAmount();
        TrustLineAmount totalCapacity = calculateTotalPathCapacityForReceive();

        if (totalCapacity < remainingReceive) {
            warning() << "Rebuilt paths insufficient: need " << remainingReceive
                      << " but total capacity only " << totalCapacity;
            reject("Rebuilt paths have insufficient capacity");
            return resultInsufficientFundsError();
        }

        info() << "Rebuilt paths have sufficient capacity: " << totalCapacity
               << " for remaining " << remainingReceive;

        // Existing code continues...
        mPreviousInaccessibleNodesCount = mInaccessibleNodes.size();
        mPreviousRejectedTrustLinesCount = mRejectedTrustLines.size();
        mDirectPathIsAlreadyProcessed = false;
        initAmountsReservationOnNextPath();
        mIsAuditPendingPathsOccurred = false;
        return runAmountReservationStage();
    }
    debug() << "New paths was not built";
}
```

**Key Points**:
- Inserted after `buildPathsAgain()` and path count check
- Calculates remaining needed and total capacity
- Fails transaction if capacity insufficient
- Clear logging of capacity vs. requirement
- Only continues if capacity sufficient

### Error Handling Specifications

#### Error Conditions

1. **No new paths after rebuilding**:
   - Log: `warning() << "Path rebuilding produced no new paths"`
   - Action: Continue to existing logic (check audit pending paths or fail)
   - Cleanup: None needed (topology modifications local to rebuilding attempt)
   - Result: Eventually returns `resultInsufficientFundsError()`

2. **Insufficient capacity after rebuilding**:
   - Log: `warning() << "Rebuilt paths insufficient: need X but total capacity only Y"`
   - Action: Call `reject("Rebuilt paths have insufficient capacity")` and return `resultInsufficientFundsError()`
   - Cleanup: All reservations dropped via `rollBack()`
   - Result: Transaction fails with code 412

3. **Exception during `calculateMaxFlow()`**:
   - Log: `warning() << "Path rebuilding failed: " << e.what()`
   - Action: Catch exception in `buildPathsAgain()`, log, and return
   - Cleanup: No paths added to `mPathsStats`
   - Result: Transaction continues to existing logic (likely fails with insufficient funds)

4. **Topology modification errors**:
   - Log: Depends on specific operation failing
   - Action: Skip problematic modification, continue with partial updates
   - Cleanup: Best-effort topology modification
   - Result: Path rebuilding may produce suboptimal but valid results

5. **Capacity calculation errors**:
   - Unlikely (calculations on existing data)
   - If occurs: Method should not throw, return conservative value (0 or sum of valid entries)
   - Result: Transaction may fail conservatively (false negative acceptable)

## Implementation Plan
### Milestone Flow
- **Milestone 1 – Topology Sanitisation Ready**: Requirements 1-4 реалізовані (виключення нод/ліній, застосування резервів, нові API). Вихід: оновлена топологія без некоректних ребер.
- **Milestone 2 – Path Rebuilder Integrated**: `buildPathsAgain()` (Requirement 5) перебудовує шляхи на основі очищеної топології та кладе їх у `mPathsStats`. Залежить від Milestone 1.
- **Milestone 3 – Capacity Guard**: `calculateTotalPathCapacityForReceive()` (Requirement 6) та перевірка у `tryProcessNextPath()` (Requirement 7) гарантують достатню пропускну здатність перед переходом до резерву. Залежить від Milestone 2.
- **Milestone 4 – Validation & Tests**: Усі юніт-тести з розділу “Testing Strategy” проходять, журнали підтверджують відсутність деградацій. Залежить від Milestones 1-3.

### Iteration Milestones
| Milestone | Exit Criteria | Dependencies | Risk Level |
|-----------|---------------|--------------|------------|
| API Extensions Ready | `removeTrustLine` та `participantsIDs()` доступні й покриті тестами | Milestone 1 | Medium |
| Path Rebuilder Complete | `buildPathsAgain()` повертає позитивний результат на зміненій топології | API Extensions Ready | High |
| Capacity Guard Active | Валідація пропускної здатності блокує дефіцитні сценарії | Path Rebuilder Complete | Medium |
| Test Suite Passing | Усі нові/оновлені тести зелені | Попередні мілстоуни | Low |

### Dependencies on Other Teams/Projects
- No external team dependencies identified

### Integration Points with Previous Work
- Builds upon `EquivalentsSubsystemsRouter` topology management (PRD 07)
- Uses `ExchangePathsManager::calculateMaxFlow()` (PRD 04)
- Uses existing `mInaccessibleNodes` and `mRejectedTrustLines` tracking
- Uses existing reservation tracking infrastructure

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++, OR-Tools, and payment transaction experience
- **Developers**: 1 developer for implementation support
- **QA Engineers**: 1 engineer for unit testing

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Topology modification corruption | High | Low | Careful implementation with thorough testing; topology modifications isolated to `buildPathsAgain()` |
| `calculateMaxFlow()` failure with modified topology | High | Medium | Exception handling in `buildPathsAgain()`; validation of topology state |
| Incorrect capacity calculation | Medium | Low | Thorough unit testing with diverse scenarios; cross-validation with manual calculations |
| Performance degradation on large networks | Medium | Low | Performance benchmarking during testing; optimization if needed |
| Race conditions in topology modification | Low | Very Low | Single-threaded transaction execution; no concurrent access |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Path rebuilding doesn't improve success rate | Medium | Low | Evaluate with realistic test scenarios; worst case: no regression (feature optional) |

## Testing Strategy
### Testing Approach (Unit-Only)
- All testing is unit-only (no integration/E2E tests)
- Tests are built and executed exclusively in `build-tests`
- Use real objects following existing test patterns
- Test files located in `tests/unit/transactions/` directory

### Testing Best Practices
- **Real Objects Over Mocks**: Use real instances where possible
- **Exception Testing**: Test error paths using EXPECT_THROW
- **Edge Case Coverage**: Test boundary values, empty sets, large networks
- **Scenario Tests**: Test realistic payment scenarios with node failures

#### Unit Tests: New/Modified Components

**Test File**: `tests/unit/transactions/CoordinatorExchangePaymentPathRebuildingTest.cpp`

**1. Topology Update Tests**

*Test 1: Outgoing links removed for inaccessible node*
- **Setup**: зібрати топологію `A -> B -> C`; додати B до `mInaccessibleNodes`; змокати `participantsIDs()` так, щоб повертала {A,B,C}.
- **Execution**: виклик `buildPathsAgain()` після підготовки.
- **Expected**: `trustLinePtrsSet(A)` не містить ребра до B; `trustLinePtrsSet(B)` порожнє; `removeTrustLine` викликано саме двічі.

*Test 2: Incoming лінії до inaccessible node видаляються*
- **Setup**: топологія `A -> B`, `D -> B`; додаємо B до `mInaccessibleNodes`; `participantsIDs()` повертає {A,B,D}.
- **Expected**: після `buildPathsAgain()` `trustLinePtrsSet(A)` і `trustLinePtrsSet(D)` не містять цільових ребер на B.

*Test 3: Rejected trust line виключається по всіх еквівалентах*
- **Setup**: топологія з двома еквівалентами; пара (B,C) у `mRejectedTrustLines`.
- **Expected**: для кожного еквівалента `trustLinePtrsSet(B)` не містить ребра до C; інші ребра зберігаються.

*Test 4: Комбінований сценарій (inaccessible + rejected)*
- **Setup**: топологія на 4 вузли, один вузол у `mInaccessibleNodes`, одна пара у `mRejectedTrustLines`.
- **Expected**: `removeTrustLine` викликано для всіх відповідних ребер, інші ребра недоторкані.

**1.1. Topology Manager / Router API**

*Test 4a: `removeTrustLine` видаляє усі внутрішні структури*
- **Setup**: створити менеджер, додати ребро `A -> B`.
- **Execution**: викликати `removeTrustLine(A,B)`.
- **Expected**: `trustLinePtrsSet(A)` більше не містить ребра, пам’ять не витікає.

*Test 4b: `participantsIDs()` повертає унікальні ContractorID*
- **Setup**: зареєструвати три адреси через `getOrCreateParticipantID`.
- **Execution**: викликати `participantsIDs()`.
- **Expected**: набір містить саме три ідентифікатори без дублювань; порядок не критичний.

**2. Резерви через `mNodesFinalAmountsConfiguration`**

*Test 5: Частковий резерв мітить ребро як використане*
- **Setup**: створити шлях `A -> B -> C`, додати до `mNodesFinalAmountsConfiguration[B]` Incoming резервацію на pathID=1 із сумою 400; `mPathsStats[1]` містить відповідний шлях; встановити `mPaymentNodesIds` та `mPaymentParticipants`.
- **Execution**: `buildPathsAgain()`.
- **Expected**: `TopologyTrustLinesManager::addUsedAmount(A,B,400)` викликано; `makeFullyUsed` не викликано.

*Test 6: Повний резерв вмикає `makeFullyUsed`*
- **Setup**: аналог тесту 5, але сума дорівнює максимальному потоку для вузла.
- **Expected**: `makeFullyUsed(A,B)` викликано один раз, `addUsedAmount` не викликано для цієї пари.

*Test 7: Резерви для кількох шляхів не дублюються*
- **Setup**: два різні `pathID`, вхідні резерви для різних пар вузлів; у конфігурації також наявні outgoing записи.
- **Expected**: застосовані лише Incoming; outgoing ігноруються; загальна кількість викликів відповідає кількості унікальних Incoming записів.

*Test 8: Конфігурація без попереднього вузла пропускається без падіння*
- **Setup**: додати Incoming резерв для вузла на позиції 0 (початок шляху).
- **Expected**: `addUsedAmount`/`makeFullyUsed` не викликаються, метод завершується без винятків.

**3. Path Rebuilding Tests**

*Test 9: Rebuild paths after excluding inaccessible node*
- **Setup**:
  - Initial topology: A -> B -> C (primary path) and A -> D -> C (alternative)
  - Node B added to `mInaccessibleNodes`
  - Initial paths include A -> B -> C
- **Execution**: Call `buildPathsAgain()`
- **Expected**:
  - New paths calculated
  - New paths use alternative route A -> D -> C
  - New paths added to `mPathsStats`
- **Assertions**:
  - `mPathsStats.size()` increased
  - New paths don't include node B
  - Alternative path A -> D -> C present in results

*Test 10: Rebuild paths after excluding rejected trust line*
- **Setup**:
  - Topology with multiple alternative paths
  - One trust line rejected and added to `mRejectedTrustLines`
- **Execution**: Call `buildPathsAgain()`
- **Expected**:
  - New paths calculated avoiding rejected trust line
  - New paths added to `mPathsStats`
- **Assertions**:
  - New paths don't use rejected trust line
  - Alternative paths found

*Test 11: No new paths available after topology update*
- **Setup**:
  - Simple topology with single path A -> B -> C
  - Node B is only intermediary, added to `mInaccessibleNodes`
- **Execution**: Call `buildPathsAgain()`
- **Expected**:
  - No new paths calculated (no alternatives exist)
  - `mPathsStats` size unchanged
  - Warning logged
- **Assertions**:
  - `mPathsStats.size()` unchanged
  - No crash or exception
  - Method completes successfully

*Test 12: Applied reservations впливають на `calculateMaxFlow()`*
- **Setup**: Створити два паралельні шляхи; застосувати Incoming резерв 300 на одному з ребер через `mNodesFinalAmountsConfiguration` (тест 5 підготовлений).
- **Execution**: Виклик `buildPathsAgain()`.
- **Expected**: OR-Tools повертає шлях із зменшеним `received_amount` (оригінал - 300).
- **Assertions**: новий `OptimalPathResult.received_amount` менший на величину резерву; лог містить запис про застосовані резерви.

**3. Capacity Calculation Tests**

*Test 13: Capacity sums only paths з ID > current*
- **Setup**: `mCurrentAmountReservingPathIdentifier = 1`; `mPathsStats` містить шляхи з ID 1 (500) та 2 (700).
- **Expected**: метод повертає 700 (лише шлях 2).
- **Assertions**: результат 700; debug лог містить запис про пропуск pathID=1.

*Test 14: Невалідні шляхи ігноруються*
- **Setup**: додати шлях ID=3 із `isValid() == false`, шлях ID=4 дійсний із `received_amount = 400`.
- **Expected**: у суму потрапляє тільки шлях 4.
- **Assertions**: результат 400; журнал містить запис про пропуск невалідного шляху.

*Test 15: Оброблені шляхи (isLastIntermediateNodeProcessed) не враховуються*
- **Setup**: шлях ID=5 із прапорцем `isLastIntermediateNodeProcessed() == true`, шлях ID=6 ще не оброблено.
- **Expected**: підсумок включає лише шлях 6.
- **Assertions**: результат відповідає `received_amount` шляху 6.

*Test 16: Відсутність шляхів повертає 0*
- **Setup**: `mPathsStats` порожній.
- **Expected**: метод повертає 0 без журналів warn/error.
- **Assertions**: результат 0, лог лише з інформаційним повідомленням про суму.

**4. Capacity Validation Tests**

*Test 17: Validation passes коли `totalCapacity` >= потреби*
- **Setup**: після перебудови `calculateTotalPathCapacityForReceive()` повертає 700; `remainingReceive = 500`.
- **Expected**: `tryProcessNextPath()` продовжує маршурування, лог містить info про успішну перевірку.

*Test 18: Validation завершує транзакцію при дефіциті*
- **Setup**: `totalCapacity = 500`, `remainingReceive = 800`.
- **Expected**: метод логінгить warning та повертає `resultInsufficientFundsError()`; викликається `reject("Rebuilt paths have insufficient capacity")`.

*Test 19: Граничний випадок із точним збігом*
- **Setup**: `totalCapacity = remainingReceive = 1000`.
- **Expected**: перевірка проходить, транзакція переходить до `runAmountReservationStage()` без помилок.

**5. End-to-End Scenario Tests**

*Test 20: Complete path rebuilding scenario with successful recovery*
- **Setup**:
  - Payment for 1000, primary path through nodes A -> B -> C
  - Node B becomes inaccessible after some reservations
  - Alternative path A -> D -> C exists
  - Already reserved 300 on other paths
- **Execution**:
  - Process paths until B inaccessible
  - Trigger `tryProcessNextPath()` with no more paths
  - `buildPathsAgain()` called
  - Capacity validation runs
- **Expected**:
  - Paths rebuilt excluding node B
  - Alternative path A -> D -> C found
  - Capacity validation passes (700 remaining, alternative has capacity)
  - Transaction continues to complete payment
- **Assertions**:
  - New paths added to `mPathsStats`
  - Alternative path used
  - Payment eventually succeeds

*Test 21: Complete path rebuilding scenario with insufficient capacity*
- **Setup**:
  - Payment for 1000, only path through A -> B -> C
  - Node B becomes inaccessible
  - No alternative paths exist
- **Execution**:
  - Process paths until B inaccessible
  - Trigger `tryProcessNextPath()` with no more paths
  - `buildPathsAgain()` called
  - No new paths found
- **Expected**:
  - `buildPathsAgain()` produces no new paths
  - Capacity validation not reached (no new paths)
  - Transaction fails with `resultInsufficientFundsError()`
- **Assertions**:
  - Result code == 412
  - Error message indicates insufficient funds

*Test 22: Multiple rebuilding attempts with different failures*
- **Setup**:
  - Complex network with multiple alternative paths
  - First attempt: node B inaccessible
  - Second attempt (after rebuilding): trust line D->E rejected
- **Execution**:
  - First rebuilding after B inaccessible
  - Reservation on alternative path
  - D->E rejects
  - Second rebuilding attempt
- **Expected**:
  - First rebuilding: finds alternative excluding B
  - Second rebuilding: finds alternative excluding B and D->E
  - Multiple rebuilding attempts tracked by `mRebuildingAttemptsCount`
  - Eventually either succeeds or reaches max attempts
- **Assertions**:
  - Both inaccessible node and rejected trust line excluded in topology
  - Paths progressively avoid more problematic elements
  - Proper attempt counting

#### Regression Testing (Unit)
- Scope: Ensure changes don't break existing exchange payment execution
- Verify payments without inaccessible nodes work unchanged
- Validate single-equivalent payments remain unaffected (use different code path)
- Confirm existing path processing logic unaffected

#### Execution in CI/Locally
- Build tests in `build-tests` and run the produced binaries
- All unit tests must pass before PRD completion
- Performance benchmarks for topology update and path rebuilding

### Quality Gates
- All unit tests pass in `build-tests`
- Topology updates correctly exclude all problematic elements (100% accuracy)
- Path rebuilding produces valid alternative paths when they exist
- Capacity validation correctly detects sufficient/insufficient capacity (100% accuracy)
- No regressions in existing exchange payment behavior
- Performance acceptable: `buildPathsAgain()` < 5 seconds on typical networks

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Feature addition (enhancement to existing transaction)
- **Rollout Strategy**: Direct deployment; enhancement is internal optimization
- **Rollback Plan**: Revert to previous version if critical issues found; no data migration concerns

### Database Migrations
- No database migrations required (runtime logic only)

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: Release notes mentioning improved payment resilience
- **Documentation Updates**:
  - Internal documentation on path rebuilding mechanism
  - Notes on topology state management during rebuilding

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**:
  - Path rebuilding success rate (target: >50% when alternatives exist)
  - Topology update accuracy (target: 100%)
  - Capacity validation accuracy (target: 100%)
  - Payment success rate improvement (target: measurable increase)
- **Leading Indicators**: Unit test pass rate, topology update correctness
- **Baseline Values**: Current payment success rate without rebuilding
- **Target Values**:
  - 100% unit test pass rate
  - 100% topology update accuracy
  - >50% path rebuilding success when alternatives exist
  - +10-20% payment success rate in networks with availability issues

### Monitoring Plan
- **New Dashboards/Alerts**: Not applicable (internal logic enhancement)
- **Enhanced Monitoring**: Extended logging for path rebuilding attempts, success/failure rates
- **A/B Testing**: Not applicable

### Review Schedule
- **Daily**: Development progress and unit test status
- **Weekly**: Code review and integration testing results
- **Post-Implementation Review**: Evaluation of payment success rate improvement

## Appendices
### Glossary
- **Inaccessible Node**: Node that didn't respond during reservation, possibly offline
- **Rejected Trust Line**: Trust line that rejected reservation request
- **Path Rebuilding**: Process of recalculating paths excluding problematic network elements
- **Topology Update**: Modification of network topology to reflect current availability
- **Capacity Reduction**: Decreasing trust line capacity to reflect already reserved amounts
- **Path Capacity**: Maximum amount that can flow through a path to receiver

### References
- [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
- [Exchange Payment Topology Collection PRD](07-exchange-payment-topology-collection.md)
- [Path Capacity Adjustment PRD](08-path-capacity-adjustment.md)
- [Exchange Rate and Commission Change Handling PRD](09-exchange-rate-commission-change-handling.md)
- [Allowable Payment Amount Control PRD](10-allowable-payment-amount-control.md)
- [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
- [Topology Collection Protocol](../../../architecture/vtcpd/protocols/topology-collection-protocol.md)
- [CoordinatorExchangePaymentTransaction Implementation](../../../src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h)
- [CoordinatorPaymentTransaction Implementation](../../../src/core/transactions/transactions/regular/payments/CoordinatorPaymentTransaction.h) (reference for single-equivalent path rebuilding)
- [ExchangePathsManager Implementation](../../../src/core/paths/ExchangePathsManager.h)
- [EquivalentsSubsystemsRouter Implementation](../../../src/core/equivalents/EquivalentsSubsystemsRouter.h)

### Detailed Component Specifications

#### Modified Classes

##### CoordinatorExchangePaymentTransaction
- **Location**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h/.cpp`
- **Modified Methods**:
  - `buildPathsAgain()`: Complete implementation (currently empty skeleton)
  - `tryProcessNextPath()`: Add capacity validation after path rebuilding
- **New Methods**:
  ```cpp
  TrustLineAmount calculateTotalPathCapacityForReceive() const;
  ```
- **Implementation Focus**:
  - Topology update in `buildPathsAgain()`
  - OR-Tools `calculateMaxFlow()` invocation
  - Path addition to `mPathsStats`
  - Capacity calculation and validation

#### Algorithm Implementation Examples

**Topology Modification Pattern**:
```cpp
auto topologyManager = mEquivalentsSubsystemsRouter->topologyTrustLineManager(equivalent);
for (auto *linePtr : topologyManager->trustLinePtrsSet(sourceID)) {
    if (linePtr->topologyTrustLine()->targetID() == targetID) {
        topologyManager->removeTrustLine(sourceID, targetID);
        break;
    }
}
```

**Reservation Application Pattern**:
```cpp
auto topologyManager = mEquivalentsSubsystemsRouter->topologyTrustLineManager(equivalent);

if (reservedAmount >= maxHopCapacity) {
    topologyManager->makeFullyUsed(sourceID, targetID);
} else {
    topologyManager->addUsedAmount(sourceID, targetID, reservedAmount);
}
```

**Path Addition Pattern**:
```cpp
// Pattern for adding rebuilt paths
for (auto& rebuiltPath : maxFlowResult.optimalPaths) {
    PathID newPathID = generateNextPathID();
    mPathsStats[newPathID] = make_unique<OptimalPathResult>(std::move(rebuiltPath));
}
```

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-11-06 | Claude Code | Initial draft for path rebuilding mechanism | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [10-allowable-payment-amount-control.md](10-allowable-payment-amount-control.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)
- **Payment Protocol**: [payment-protocol.md](../../../architecture/vtcpd/protocols/payment-protocol.md)
- **Topology Collection Protocol**: [topology-collection-protocol.md](../../../architecture/vtcpd/protocols/topology-collection-protocol.md)
