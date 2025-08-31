# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Exchange Flow Calculation
- **PRD ID**: 04
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.0
- **Date**: 2025-08-26
- **Author(s)**: Mykola Ilashchuk, Dima Chizhevsky
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevskys
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-08-26
- **Previous PRD**: [03-exchange-rates-manager.md](03-exchange-rates-manager.md)
- **Related Documents**: [Exchange Rates Manager PRD](03-exchange-rates-manager.md), [Database Provider Abstraction](01-database-provider-abstraction.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This iteration extends the existing payment path finding system to support multi-equivalent exchanges and maximum flow calculations. Building upon the Exchange Rates Manager (PRD 03), this PRD implements topology collection with exchange rates and optimal flow calculation using OR-Tools library.

### Current project state
- Exchange Rates Manager provides local exchange rate storage and management
- Single-equivalent path finding works with existing topology collection
- Trust line topology collection infrastructure exists

### This iteration's focus
- Multi-equivalent topology collection with exchange rates
- Integration with OR-Tools library for maximum flow calculation
- New transaction types for exchange-based flow calculations
- Unified contractor ID management across equivalents

### Connection to overall vision
This enables efficient cross-equivalent payments in the decentralized vTCP network, maximizing payment capacity while maintaining the distributed architecture.

## Iteration Context
### Previous Iterations Summary
- **PRD 01**: Database provider abstraction established multi-database support
- **PRD 03**: Exchange Rates Manager provides local exchange rate storage with TTL management
- **Completed Features**: Basic topology collection, single-equivalent max flow calculation, exchange rate storage

### Lessons Learned
- Topology collection requires efficient message distribution
- TTL management is critical for exchange rate accuracy
- Unified ID management needed for multi-equivalent operations

### Current State Analysis
- **What's working well**: Existing topology collection infrastructure, exchange rate storage
- **Pain points identified**: Separate ID spaces per equivalent limit cross-equivalent operations
- **Performance metrics**: Current single-equivalent flow calculation completes within acceptable timeframes

## Problem Statement
### Background
The current vTCP network supports single-equivalent payment path finding, but users often need to pay in one equivalent while receivers want to receive in another. Exchange-capable nodes can facilitate these conversions, but the system lacks the ability to:
1. Collect and distribute exchange rate information across the network
2. Calculate optimal multi-equivalent payment flows
3. Determine maximum receivable amounts considering all possible exchange paths

### Problem Description
**Who is affected**: All network participants wanting to perform cross-equivalent payments
**When and where**: During payment path discovery and flow calculation phases
**Current limitations**:
- No exchange rate topology collection
- No cross-equivalent flow optimization
- Inconsistent node identification across equivalents prevents unified flow calculation

### Impact of not solving this problem
- Reduced network utility for cross-equivalent payments
- Suboptimal exchange rate utilization
- Manual coordination required for multi-equivalent transactions

### Success Metrics
**Primary KPIs**:
- Successful multi-equivalent topology collection with exchange rates
- Accurate maximum flow calculation using OR-Tools
- Maintained backward compatibility with existing single-equivalent operations

**Target Values**:
- 100% compatibility with existing single-equivalent transactions
- Exchange rate topology collection within existing timeout windows
- Maximum flow calculation completion for typical network topologies

## Goals
The primary goals of this iteration are to enhance network connectivity and provide optimal value to users during transactions.

*   **Goal 1: Enable universal transaction capability.**
    *   **Description:** Allow any user to send a payment to any other user within the network, provided a path of intermediaries exists between them.
    *   **Success Metric:** The system successfully finds a path and calculates a valid exchange rate for at least 99% of payment requests between connected nodes in a simulated complex network topology.

*   **Goal 2: Ensure the most cost-effective transaction routing.**
    *   **Description:** When multiple payment paths exist, the system must automatically calculate and select the path that results in the best possible exchange rate for the end-user, minimizing fees and slippage.
    *   **Success Metric:** The implemented algorithm consistently identifies the path with the lowest overall transaction cost (including intermediary fees) in all defined test scenarios.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **Multi-equivalent topology collection**: Extended message protocol to include exchange rate information
2. **Exchange-aware transaction types**: New transaction classes for multi-equivalent flow calculation
3. **OR-Tools integration**: Library integration for optimal flow calculation
4. **Unified contractor management**: Consistent node identification across equivalents
5. **External exchange rate storage**: Management of exchange rates from network participants

#### Technical Infrastructure
- New message types for exchange rate distribution
- Extended topology collection with exchange rate awareness
- OR-Tools library integration in CMake build system
- Enhanced TailManager for exchange rate message queuing

#### Integration Points
- Extension of existing topology collection classes
- Enhancement of existing message types with exchange rate parameters
- Integration with existing ExchangeRatesManager from PRD 03

### Explicitly Out of Scope
- Exchange rate validation and security mechanisms
- Performance optimization for large topologies
- Advanced OR-Tools solver configuration
- User interface for multi-equivalent operations

### Dependencies from Previous Iterations
- **PRD 03**: ExchangeRatesManager for local exchange rate storage
- **Existing topology collection**: BaseCollectTopologyTransaction and related infrastructure
- **Message infrastructure**: Existing max flow calculation message types

### Future Roadmap Impact
This iteration establishes the foundation for:
- **Advanced cross-equivalent payment strategies**: Path-based optimization enables sophisticated payment routing
- **Multi-hop exchange optimization**: Complete path information supports complex exchange chain analysis  
- **Network-wide liquidity analysis**: Comprehensive path data enables liquidity mapping and bottleneck identification
- **Path-based analytics and monitoring**: Direct access to optimal paths enables:
  - Path performance tracking and ranking
  - Exchange rate sensitivity analysis
  - Alternative path discovery for redundancy planning
  - Real-time path efficiency monitoring
- **Dynamic path selection**: Foundation for adaptive routing based on current network conditions
- **Multi-objective optimization**: Infrastructure ready for incorporating path preferences, fees, latency considerations
- **Predictive path analysis**: Historical path data enables prediction of optimal routing strategies

## User Stories & Requirements

### User Personas
#### Primary User: Network Node Operator
- **Role**: Operates a vTCP network node
- **Goals**: Enable cross-equivalent payments with optimal exchange rates
- **Pain Points**: Cannot currently facilitate or optimize multi-equivalent transactions
- **Technical Proficiency**: Advanced

#### Secondary User: Payment Initiator
- **Role**: Initiates payments in the vTCP network
- **Goals**: Find optimal paths for cross-equivalent payments
- **Pain Points**: Limited to single-equivalent payments
- **Technical Proficiency**: Intermediate

### Functional Requirements
#### New Features for This Iteration

1. **Multi-Equivalent Topology Collection**
   - **Description**: Collect network topology including exchange rate information from exchange-capable nodes
   - **User Story**: As a network node, I want to collect exchange rate information during topology discovery so that I can calculate optimal cross-equivalent payment flows
   - **Rationale**: Required foundation for multi-equivalent flow calculation
   - **Builds Upon**: Existing topology collection infrastructure
   - **Acceptance Criteria**: 
     - Exchange rates collected from all exchange-capable nodes in topology
     - Exchange rates stored with appropriate TTL management
     - Backward compatibility maintained with single-equivalent topology collection
   - **Priority**: High
   - **Dependencies**: ExchangeRatesManager (PRD 03)

2. **OR-Tools Maximum Flow Integration**
   - **Description**: Integrate OR-Tools library for optimal multi-equivalent flow calculation
   - **User Story**: As a payment initiator, I want to determine the maximum possible payment amount considering all available exchange paths
   - **Rationale**: Provides mathematically optimal flow calculation
   - **Builds Upon**: Collected topology and exchange rate data
   - **Acceptance Criteria**:
     - OR-Tools successfully integrated in CMake build
     - Maximum flow calculation produces correct results
     - Flow decomposition provides detailed path information
   - **Priority**: High
   - **Dependencies**: Multi-equivalent topology collection

3. **Exchange-Aware Transaction Types**
   - **Description**: New transaction classes for initiating and managing multi-equivalent flow calculations
   - **User Story**: As a network node, I want specialized transaction types for handling multi-equivalent operations
   - **Rationale**: Separation of concerns from single-equivalent operations
   - **Builds Upon**: Existing transaction infrastructure
   - **Acceptance Criteria**:
     - InitiateMaxFlowExchangeCalculationTransaction implemented
     - CollectTopologyForExchangeTransaction implemented
     - Proper inheritance from base transaction classes
     - exchangeEquivalents limited to maximum 5 elements with error code 401 (responseProtocolError) for violations
   - **Priority**: High
   - **Dependencies**: Unified contractor management

4. **Unified Contractor ID Management**
   - **Description**: Move contractor ID management from TopologyTrustLinesManager to EquivalentsSubsystemsRouter
   - **User Story**: As a system, I want consistent node identification across all equivalents for proper multi-equivalent flow calculation
   - **Rationale**: Enables unified flow calculation across different equivalents
   - **Builds Upon**: Existing EquivalentsSubsystemsRouter infrastructure
   - **Acceptance Criteria**:
     - mParticipantsAddresses moved to EquivalentsSubsystemsRouter
     - Consistent ID assignment across equivalents
     - All topology managers use unified ID system
   - **Priority**: High
   - **Dependencies**: None

5. **External Exchange Rate Storage**
   - **Description**: Storage and management of exchange rates received from network participants
   - **User Story**: As a network node, I want to store and manage exchange rates from other nodes with proper expiration handling
   - **Rationale**: Required for multi-equivalent flow calculations
   - **Builds Upon**: ExchangeRatesManager from PRD 03
   - **Acceptance Criteria**:
     - mExternalExchangeRates map implemented in ExchangeRatesManager
     - Automatic cleanup of expired external exchange rates
     - Integration with TailManager for message processing
   - **Priority**: High
   - **Dependencies**: ExchangeRatesManager (PRD 03)

#### Enhancements to Existing Features

1. **Extended Message Protocol**
   - **Current State**: Max flow calculation messages support single equivalent only
   - **Proposed Changes**: Add vector<SerializedEquivalent> exchangeEquivalents parameter to all max flow messages
   - **Impact Assessment**: Backward compatible - empty vector maintains existing behavior
   - **Migration Strategy**: Gradual rollout with version compatibility

2. **Enhanced TailManager**
   - **Current State**: TailManager handles specific message types in dedicated queues
   - **Proposed Changes**: Add mExchangeRatesTail for ExchangeRatesMessage handling
   - **Impact Assessment**: Pure addition, no impact on existing functionality
   - **Migration Strategy**: Direct implementation in IncomingRemoteNode::tryCollectNextPacket()

### Non-Functional Requirements
#### Performance
- Topology collection with exchange rates completes within existing timeout windows
- OR-Tools flow calculation completes for typical network sizes (< 1000 nodes)
- Memory usage increase < 50MB for exchange rate storage

#### Security
- No validation of exchange rate authenticity (explicitly out of scope)

#### Scalability
- Support for up to 10 different equivalents in exchange calculations
- Handle exchange rate information from up to 100 exchange-capable nodes

#### Reliability
- Node startup failure when OR-Tools library is unavailable or doesn't meet minimum version requirements
- Automatic cleanup of expired exchange rate information
- Fallback error reporting for failed multi-equivalent calculations

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Single-equivalent topology collection with separate ID spaces per equivalent
- **Proposed Changes**: Unified contractor ID management and multi-equivalent aware topology collection
- **Backwards Compatibility**: New transaction types maintain separation from existing single-equivalent operations
- **Migration Requirements**: TopologyTrustLinesManager ID management migration to EquivalentsSubsystemsRouter

### Technology Stack Updates
#### New Technologies/Libraries
- **OR-Tools**: Mathematical optimization library for maximum flow calculation
  - Purpose: Optimal multi-equivalent flow calculation
  - Integration approach: CMake find_package integration similar to SQLite/PostgreSQL
  - Version: >= 9.0 (latest stable version)
  - Components: `ortools::graph` for maximum flow algorithms

#### Version Updates
- No version updates required for existing dependencies

### Integration Requirements
#### New Integrations
- **OR-Tools Library**: Integration via CMake for flow optimization calculations

#### Modified Integrations
- **ExchangeRatesManager**: Extended with external exchange rate storage capabilities
- **TailManager**: Enhanced with exchange rate message queue support

### Protocol Semantics
- **Terminology**:
  - `mEquivalent`: equivalent in which the receiver accepts funds.
  - `exchangeEquivalents`: list of payer equivalents in which the sender can pay.

- **CollectTopologyForExchangeTransaction (payer side)**:
  - For each `exchangeEquivalents[i]` send `MaxFlowCalculationSourceFstLevelMessage` with:
    - `equivalent = exchangeEquivalents[i]`
    - `exchangeEquivalents = { mEquivalent }` (single-element vector containing receiver equivalent)
  - `InitiateMaxFlowForExchangeCalculationMessage` replaces `InitiateMaxFlowCalculationMessage` and carries full `exchangeEquivalents` of the payer and `mEquivalent` of the receiver.

- **ReceiveMaxFlowCalculationForExchangeOnTargetTransaction (receiver side)**:
  - Collects topology only in `mEquivalent` and sends `MaxFlowCalculationTargetFstLevelMessage` with:
    - `equivalent = mEquivalent`
    - `exchangeEquivalents = <payer-provided set>`

- **Forwarding behavior on intermediate transactions**:
  - Source side (`MaxFlowCalculationSourceFstLevelTransaction`/`SndLevel`): forward messages preserving `exchangeEquivalents` as received; when present, attempt to find and send exchange rates for pairs `mEquivalent / exchangeEquivalents[i]` back to payer via `ExchangeRatesMessage` alongside topology.
  - Target side (`MaxFlowCalculationTargetFstLevelTransaction`/`SndLevel`): forward messages preserving `exchangeEquivalents` as received; when present, attempt to find and send exchange rates for pairs `exchangeEquivalents[i] / mEquivalent` back to payer via `ExchangeRatesMessage` alongside topology.
  - For single-equivalent collection (legacy flows), `exchangeEquivalents` MUST be an empty vector to preserve backward compatibility.

### Data Requirements
#### Data Models
- **ExchangeRatesMessage**: New message type containing vector of ExchangeRate objects
- **External Exchange Rates Storage**: Map structure in ExchangeRatesManager for network-sourced exchange rates
- **Enhanced Message Types**: All max flow calculation messages extended with exchangeEquivalents parameter

##### ExchangeRate (per `src/core/rates/ExchangeRate.h`)
- **Fields**:
  - `SerializedEquivalent equivalentFrom`
  - `SerializedEquivalent equivalentTo`
  - `TrustLineAmount exchangeRate`
  - `int16_t exchangeRateShift`
  - `DateTime expiresAt`
  - `TrustLineAmount minExchangeAmount`
  - `TrustLineAmount maxExchangeAmount`
- **Notes**:
  - `expiresAt` is used by TTL cleanup logic.
  - Shift and min/max amounts must be respected when composing feasible exchange paths.

##### ExchangeRatesMessage
- Payload: `vector<ExchangeRate>`
- Usage: sent by source/target level transactions when relevant exchange rates are available.
- Backward compatibility: absence of this message implies no exchange rates provided.

#### Data Storage
- Exchange rate information stored in memory with TTL expiration
- Unified contractor ID mapping in ContractorsManager
- No persistent storage requirements for this iteration

##### ExchangeRatesManager: external rates storage and TTL
- `mExternalExchangeRates`: `map<pair<SerializedEquivalent, SerializedEquivalent>, vector<pair<ContractorID, ExchangeRate>>>`
- TTL cleanup timer behavior:
  - Computes the nearest `expiresAt` across all stored external rates and sleeps until that time.
  - Upon wake-up, removes expired entries from vectors and erases empty vectors from the map.
  - On insertion of a new external `ExchangeRate`, recompute the nearest expiration and reschedule the timer if necessary.
  - Each external rate is stored together with the `ContractorID` of the sender (obtained via `EquivalentsSubsystemsRouter::getOrCreateParticipantID`).
  - Behavior mirrors the internal `mExchangeRates` TTL maintenance.

#### Data Migration
- Migration of mParticipantsAddresses from TopologyTrustLinesManager to EquivalentsSubsystemsRouter
- No data format changes requiring migration

### OR-Tools Integration Specifications

#### Problem Type
**Linear Programming Optimization Problem**: Maximize the total amount received by the target in the target equivalent, considering exchange rates and capacity constraints. This is NOT a pure maximum flow problem, but rather a multi-objective optimization where exchange rates directly affect the objective function.

#### Mathematical Model for Linear Programming

**Objective Function**:
```
Maximize: Σ(flow_path_i × effective_exchange_rate_i)
```
Where each path contributes to the total received amount based on its flow and the exchange rate along that path.

**Decision Variables**:
- `flow[path_i]`: Amount of flow through path i
- `exchange_amount[node_j, equiv_from, equiv_to]`: Amount exchanged at node j from equiv_from to equiv_to

**Constraints**:
1. **Capacity Constraints**: `flow[path_i] ≤ min_capacity_along_path_i`
2. **Exchange Limits**: `min_exchange ≤ exchange_amount[j, from, to] ≤ max_exchange`
3. **Balance Constraints**: Input flows = Output flows at each node
4. **Sender Balance**: Total outflow ≤ available balance per equivalent
5. **Non-negativity**: All flows ≥ 0

#### OR-Tools API Usage

##### Core Classes and Methods
```cpp
#include "ortools/linear_solver/linear_solver.h"
using operations_research::MPSolver;
using operations_research::MPVariable;
using operations_research::MPConstraint;
using operations_research::MPObjective;

// Primary solver creation
std::unique_ptr<MPSolver> solver = MPSolver::CreateSolver("GLOP");

// Variable creation for flows
MPVariable* flow_var = solver->MakeNumVar(0.0, capacity, "flow_path_i");

// Constraint creation  
MPConstraint* constraint = solver->MakeRowConstraint(min_val, max_val, "constraint_name");

// Objective function setup
MPObjective* objective = solver->MutableObjective();
objective->SetCoefficient(flow_var, exchange_rate);
objective->SetMaximization();

// Solving
MPSolver::ResultStatus result_status = solver->Solve();
```

##### Solution Interpretation and Path Extraction
**Linear Programming Advantage**: Unlike max flow, LP directly provides optimal variable values, making path interpretation straightforward without decomposition algorithms.

**Two Modeling Approaches for Path Information**:

**1. Path-Based LP Model (RECOMMENDED)**:
- **One LP variable per path**: Each feasible path gets its own flow variable
- **Direct path results**: `path_flow_var->solution_value()` immediately gives flow through specific path
- **Complete path information**: Full path details known at variable creation time
- **Advantages**: 
  - No decomposition needed
  - Direct path-flow mapping 
  - Easier result interpretation
  - Better for future path-based analytics

**2. Arc-Based LP Model (Alternative)**:
- Traditional network flow model with arc variables
- Requires post-solution path decomposition (similar to SimpleMaxFlow)
- More memory efficient for very large networks
- Standard approach but requires additional algorithmic work

**Recommended Path-Based Implementation Strategy**:
1. **Path Enumeration Phase**: Pre-compute all feasible paths during LP setup
2. **Path Variable Creation**: One variable per enumerated path with capacity constraints
3. **Exchange Rate Integration**: Objective coefficients reflect end-to-end exchange rates per path
4. **Direct Path Results**: Solution variables directly map to path flows without decomposition

**Path Enumeration Algorithm**:
```cpp
vector<ExchangePath> enumerateAllFeasiblePaths(
    const vector<SerializedEquivalent>& sender_equivalents,
    SerializedEquivalent target_equivalent,
    ContractorID target_contractor,
    int max_path_length = 6) 
{
    vector<ExchangePath> all_paths;
    
    for (auto sender_equiv : sender_equivalents) {
        // DFS from current node in sender_equiv to target in target_equivalent
        vector<ExchangePath> paths_from_equiv = dfsEnumeratePaths(
            TopologyTrustLinesManager::kCurrentNodeID, sender_equiv,
            target_contractor, target_equivalent, 
            max_path_length);
        
        all_paths.insert(all_paths.end(), paths_from_equiv.begin(), paths_from_equiv.end());
    }
    
    return all_paths;
}
```

**Benefits for Future Tasks**:
- **Path Analytics**: Direct access to optimal path set for analysis
- **Path Ranking**: Easy identification of most/least used paths  
- **Alternative Path Discovery**: All feasible paths known, not just optimal ones
- **Sensitivity Analysis**: Understand impact of capacity/rate changes on specific paths
- **Multi-Objective Optimization**: Future extensions can easily incorporate path preferences

#### Implementation Workflow in `InitiateMaxFlowExchangeCalculationTransaction::applyCustomLogic()`

**Step 1: Path Enumeration and LP Setup**
```cpp
// Create Linear Programming solver
auto solver = MPSolver::CreateSolver("GLOP");
if (!solver) {
    throw RuntimeError("GLOP solver unavailable");
}

// Enumerate all feasible paths from sender equivalents to receiver
vector<ExchangePath> feasible_paths = enumeratePathsToContractor(
    contractorID, mExchangeEquivalents, mEquivalent);

// Create LP variables for each path
vector<MPVariable*> path_flow_vars;
for (const auto& path : feasible_paths) {
    double max_capacity = path.calculateMaxCapacity();
    auto* flow_var = solver->MakeNumVar(0.0, max_capacity, 
                                       "flow_path_" + std::to_string(path.id()));
    path_flow_vars.push_back(flow_var);
}
```

**Step 2: LP Problem Construction**
```cpp
// Objective: maximize total received amount
MPObjective* objective = solver->MutableObjective();
for (size_t i = 0; i < feasible_paths.size(); i++) {
    double effective_rate = feasible_paths[i].calculateEffectiveExchangeRate();
    objective->SetCoefficient(path_flow_vars[i], effective_rate);
}
objective->SetMaximization();

// Constraints: sender balance limits per equivalent
for (const auto& equiv : mExchangeEquivalents) {
    auto* balance_constraint = solver->MakeRowConstraint(
        0.0, getSenderBalance(equiv), "balance_" + std::to_string(equiv));
    
    for (size_t i = 0; i < feasible_paths.size(); i++) {
        if (feasible_paths[i].startsWithEquivalent(equiv)) {
            balance_constraint->SetCoefficient(path_flow_vars[i], 1.0);
        }
    }
}
```

**Step 3: Optimization**
```cpp  
MPSolver::ResultStatus status = solver->Solve();
if (status == MPSolver::OPTIMAL) {
    double max_receivable = objective->Value();
    mMaxFlows[contractorID] = TrustLineAmount(max_receivable);
} else {
    warning() << "LP optimization failed for contractor " << contractorID;
    mMaxFlows[contractorID] = TrustLineAmount::kZeroAmount();
}
```

**Step 4: Solution Extraction and Path Information Storage**
```cpp
// Container for storing optimal path results for future use
vector<OptimalPathResult> optimal_paths;

// Extract optimal flows and build comprehensive path information
for (size_t i = 0; i < feasible_paths.size(); i++) {
    double optimal_flow = path_flow_vars[i]->solution_value();
    
    if (optimal_flow > 1e-6) {  // Ignore negligible flows
        const auto& path = feasible_paths[i];
        double received_amount = optimal_flow * path.calculateEffectiveExchangeRate();
        
        // Store detailed path result for future analytics
        OptimalPathResult path_result;
        path_result.path = path;
        path_result.optimal_flow = TrustLineAmount(optimal_flow);
        path_result.received_amount = TrustLineAmount(received_amount);
        path_result.effective_exchange_rate = path.calculateEffectiveExchangeRate();
        path_result.path_efficiency = received_amount / optimal_flow;
        optimal_paths.push_back(path_result);
        
        // Log with complete path information
        info() << "Optimal flow path: " << formatDetailedPath(path_result);
    }
}

// Store optimal paths for potential future use (caching, analytics, alternative solutions)
mOptimalPathResults[contractorID] = optimal_paths;

info() << "Total optimal receivable amount for contractor " << contractorID 
       << ": " << objective->Value() << " in equivalent " << mEquivalent
       << " achieved through " << optimal_paths.size() << " paths";
```

**Step 5: Error Handling**
```cpp
switch (status) {
    case MPSolver::INFEASIBLE:
        warning() << "No feasible solution exists to contractor " << contractorID;
        mMaxFlows[contractorID] = TrustLineAmount::kZeroAmount();
        break;
    case MPSolver::UNBOUNDED:
        throw RuntimeError("LP problem is unbounded - check constraints");
        break;
    case MPSolver::ABNORMAL:
        throw RuntimeError("LP solver encountered numerical issues");
        break;
    default:
        throw RuntimeError("LP solver failed with status: " + std::to_string(status));
}
```

#### Path Logging Format Specification
**Format**: Each path logged as single line with detailed node and flow information:
- **Flow Element (F)**: `F(nodes: [address1 -> address2]; ids [id1 -> id2]; flow: amount; eq: equivalent)`
- **Exchange Element (E)**: `E(node: address; id: nodeId; eqs [fromEq->toEq]; flows:[inputFlow->outputFlow])`
- **Path Separator**: ` -> ` (space-arrow-space)

**Examples**:
```
Flow path: F(nodes: [127.0.0.1:2002 -> 127.0.0.1:2003]; ids [0 -> 1]; flow: 100; eq: 1) -> F(nodes: [127.0.0.1:2003 -> 127.0.0.1:2004]; ids [1 -> 2]; flow: 100; eq: 1) -> E(node: 127.0.0.1:2004; id:2; eqs [1->2]; flows:[100->85]) -> F(nodes:[127.0.0.1:2004->127.0.0.1:2005]; ids:[2->3]; flow:85; eq:2)
```

Where:
- **F (Flow)**: Represents flow between two nodes in same equivalent
  - `nodes`: IP addresses of source and destination nodes
  - `ids`: ContractorIDs of source and destination nodes
  - `flow`: Amount of flow between nodes
  - `eq`: Equivalent of the flow
- **E (Exchange)**: Represents exchange operation at a single node
  - `node`: IP address of exchange node
  - `id`: ContractorID of exchange node
  - `eqs`: Exchange direction (fromEquivalent->toEquivalent)
  - `flows`: Flow transformation (inputAmount->outputAmount)

#### CMake Integration Pattern
```cmake
# Similar to SQLite/PostgreSQL integration
find_package(ortools REQUIRED)

target_link_libraries(vtcpd_core
    ortools::ortools
    # existing libraries...
)

# Conditional compilation support
if(ortools_FOUND)
    target_compile_definitions(vtcpd_core PRIVATE WITH_ORTOOLS)
endif()
```

#### Startup Requirements and Fallback Strategy
**OR-Tools Availability Check**: Node must verify OR-Tools library availability and minimum version (>= 9.0) during startup. If OR-Tools is unavailable or version is insufficient, node must terminate with appropriate error message.

**Runtime Fallback**: For individual calculation failures, return error through existing result interface mechanisms. No fallback algorithm implementation - multi-equivalent exchange rate optimization requires sophisticated mathematical optimization that cannot be reasonably implemented without specialized LP solvers.

## Implementation Plan
### This Iteration Timeline
- **Duration**: 4-6 weeks implementation + 2 weeks testing
- **Sprint Breakdown**: 
  - Sprint 1: Foundation and unified ID management
  - Sprint 2: Message protocol extensions and topology collection
  - Sprint 3: OR-Tools integration and flow calculation
  - Sprint 4: Testing and integration

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Unified ID Management | Week 2 | ContractorsManager enhancement complete | None | Low |
| Extended Message Protocol | Week 3 | All message types support exchangeEquivalents | Unified ID | Medium |
| OR-Tools Integration | Week 5 | Library integrated and flow calculation working | Extended messages | High |
| Complete Integration | Week 6 | All components integrated and tested | All previous | Medium |

### Dependencies on Other Teams/Projects
- No external team dependencies identified

### Integration Points with Previous Work
- Builds directly upon ExchangeRatesManager (PRD 03)
- Extends existing topology collection infrastructure
- Utilizes existing message passing and transaction systems

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++ and mathematical optimization experience
- **Developers**: 1-2 developers for implementation support
- **QA Engineers**: 1 engineer for integration testing

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| OR-Tools integration complexity | High | Medium | Start with simple integration, use CMake find_package pattern |
| Message protocol changes breaking compatibility | High | Low | Thorough testing with empty exchangeEquivalents vectors |
| Performance degradation with exchange rate data | Medium | Medium | Monitor memory usage and processing time during implementation |
| Unified ID migration complexity | Medium | Low | Careful step-by-step migration with validation |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Feature not meeting user expectations | Medium | Low | Clear scope definition and regular stakeholder feedback |

## Testing Strategy
### Testing Approach (Unit-Only)
- All testing is unit-only (no integration/E2E tests).
- Tests are built and executed exclusively in `build-tests`.
- No Docker usage. Mock all external dependencies (network, timers, storage, OR-Tools solver entry points).
- Provide mock data for both SQLite and PostgreSQL where applicable.

#### Unit Tests: New/Changed Components
- TailManager
  - Enqueue/dequeue path for `mExchangeRatesTail` is isolated from other tails.
  - `IncomingRemoteNode::tryCollectNextPacket` correctly routes `ExchangeRatesMessage` to `mExchangeRatesTail`.
  - Backward compatibility: existing tails (`mFlowTail`, `mCyclesFiveTail`, `mCyclesSixTail`, `mRoutingTableTail`) remain unaffected.
- ExchangeRatesManager (external rates)
  - Insert/update into `mExternalExchangeRates` with `ExchangeRate` objects (see `src/core/rates/ExchangeRate.h`).
  - TTL cleanup timer: computes nearest expiration, sleeps until due, removes expired entries, deletes empty vectors, and reschedules on insertions.
  - Concurrency safety for reads/writes (if applicable in implementation contract).
- Message protocol (serialization/deserialization)
  - `exchangeEquivalents` field is serialized/deserialized correctly; empty vector preserves backward-compatible behavior.
- Transactions (behavioral unit tests with mocks)
  - `CollectTopologyForExchangeTransaction`: sends `MaxFlowCalculationSourceFstLevelMessage` per payer equivalent with `equivalent = exchangeEquivalents[i]` and `exchangeEquivalents = { mEquivalent }`.
  - `ReceiveMaxFlowCalculationForExchangeOnTargetTransaction`: sends `MaxFlowCalculationTargetFstLevelMessage` with `equivalent = mEquivalent` and `exchangeEquivalents` equal to payer-provided set.

#### Regression Testing (Unit)
- Scope: existing single-equivalent topology collection and flow calculation.
- Ensure empty `exchangeEquivalents` vectors keep legacy behavior unchanged.

#### Execution in CI/Locally
- Build tests in `build-tests` and run the produced binaries (e.g., `unit_tests`, `postgresql_integration_tests` if present but only unit tests are mandatory for this PRD).

### Quality Gates
- All unit tests pass in `build-tests`.
- Legacy single-equivalent behavior preserved with empty `exchangeEquivalents`.
- Exchange rate TTL management (including external rates) verified by unit tests.

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Major feature addition (compatible with existing nodes)
- **Rollout Strategy**: Gradual deployment with feature flags for multi-equivalent operations
- **Rollback Plan**: Disable multi-equivalent transaction types, revert to single-equivalent only

### Database Migrations
- No database migrations required (in-memory storage only)

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: Node operator guidance for enabling multi-equivalent features
- **Documentation Updates**: API documentation for new transaction types

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**: 
  - Successful multi-equivalent topology collection (target: 100% success rate)
  - Accurate maximum flow calculation results (validated against known test cases)
  - Zero regression in existing single-equivalent functionality
- **Leading Indicators**: Successful OR-Tools integration, exchange rate message processing
- **Baseline Values**: Current single-equivalent operation success rates and performance
- **Target Values**: Maintain existing performance while adding multi-equivalent capabilities

### Monitoring Plan
- **New Dashboards/Alerts**: Exchange rate collection success rates, OR-Tools calculation performance
- **Enhanced Monitoring**: Extended logging for multi-equivalent flow paths
- **A/B Testing**: Not applicable for this iteration

### Review Schedule
- **Daily**: Development progress and blocker identification
- **Weekly**: Integration testing results and performance metrics
- **Post-Release Review**: Feature adoption and system stability analysis

## Appendices
### Glossary
- **Exchange Equivalent**: A currency/asset type that can be exchanged for another
- **Multi-Equivalent Flow**: Payment flow that involves exchange between different equivalents
- **Topology Collection**: Process of gathering network structure information
- **Maximum Flow**: Mathematically optimal flow through a network given capacity constraints

### References
- [Exchange Rates Manager PRD](03-exchange-rates-manager.md)
- [OR-Tools Documentation](https://developers.google.com/optimization)
- [Existing Topology Collection Implementation](../../../src/core/transactions/transactions/max_flow_calculation/)

### Detailed Component Specifications

#### New Transaction Classes

##### InitiateMaxFlowExchangeCalculationTransaction
- **Inheritance**: Extends `BaseCollectTopologyForExchangeTransaction`
- **Purpose**: Entry point for multi-equivalent flow calculation with comprehensive path information storage
- **Input Parameters**:
  - `InitiateMaxFlowExchangeCalculationCommand`: Command containing exchange equivalents list
  - `ExchangeRatesManager*`: Access to exchange rate storage
- **Key Methods**:
  - `sendRequestForCollectingTopology()`: Initiates topology collection process
  - `applyCustomLogic()`: Performs OR-Tools Linear Programming optimization
- **Output Storage**:
  - `mMaxFlows` map (ContractorID → maximum receivable amount): Primary optimization results
  - `mOptimalPathResults` map (ContractorID → vector<OptimalPathResult>): Detailed path information for future use
  - Complete path enumeration and optimization data preserved for analytics
- **Result Interface**: Returns results through existing result interface pattern
- **Future Extensions**: Stored path information enables advanced analytics, alternative solutions, path ranking

##### CollectTopologyForExchangeTransaction  
- **Inheritance**: Extends existing topology collection infrastructure
- **Purpose**: Collects topology and exchange rate information across network
- **Input Parameters**:
  - `ExchangeRatesManager*`: For accessing local exchange rates
  - `vector<SerializedEquivalent> exchangeEquivalents`: Sender's payment equivalents
  - `SerializedEquivalent mEquivalent`: Receiver's target equivalent (inherited)
- **Key Behaviors**:
  - Sends `InitiateMaxFlowForExchangeCalculationMessage` to all contractors
  - Sends `MaxFlowCalculationSourceFstLevelMessage` for each exchange equivalent
  - Uses `EquivalentsSubsystemsRouter` to access per-equivalent managers
- **Integration**: Uses `fillTopology()` and new `fillRates()` methods

##### BaseCollectTopologyForExchangeTransaction
- **Purpose**: Base class for exchange-aware topology collection
- **Inheritance**: Analog of `BaseCollectTopologyTransaction`
- **Architecture**: Uses `mEquivalentsSubsystemsRouter` instead of direct manager access
- **Key Methods**:
  - `fillTopology()`: Processes topology messages (inherited pattern)
  - `fillRates()`: NEW - processes `ExchangeRatesMessage` from `mExchangeRatesTail`

#### New Command Classes

##### InitiateMaxFlowExchangeCalculationCommand
- **Inheritance**: Analog of `InitiateMaxFlowCalculationCommand`
- **Key Fields**:
  - `vector<SerializedEquivalent> exchangeEquivalents`: Sender's payment equivalents (maximum 5 elements)
  - `SerializedEquivalent mEquivalent`: Receiver's target equivalent (reinterpreted)
- **Validation**: Must reject commands with more than 5 exchangeEquivalents with error code 401 (responseProtocolError)
- **Usage**: Input to `InitiateMaxFlowExchangeCalculationTransaction`

#### New Message Classes

##### InitiateMaxFlowForExchangeCalculationMessage
- **Inheritance**: Analog of `InitiateMaxFlowCalculationMessage`
- **Purpose**: Initiates exchange-aware flow calculation at target
- **Additional Fields**:
  - `vector<SerializedEquivalent> exchangeEquivalents`: Sender's payment equivalents
- **Handler**: `ReceiveMaxFlowCalculationForExchangeOnTargetTransaction`

##### ReceiveMaxFlowCalculationForExchangeOnTargetTransaction
- **Inheritance**: Analog of `ReceiveMaxFlowCalculationOnTargetTransaction`
- **Input**: `InitiateMaxFlowForExchangeCalculationMessage` (instead of base message)
- **Behavior**: Identical to base class but includes exchange equivalents in outgoing messages

##### ExchangeRatesMessage  
- **Inheritance**: Extends `SenderMessage`
- **Location**: `src/core/network/messages/max_flow_calculation/`
- **Payload**: `vector<ExchangeRate>` 
- **Equivalent Parameter**: Uses `mEquivalent` from creating transaction
- **Purpose**: Carries exchange rate information from network nodes to payer

##### OptimalPathResult (New Data Structure)
- **Purpose**: Comprehensive storage of optimal path solution information for future analytics
- **Fields**:
  - `ExchangePath path`: Complete path details (nodes, equivalents, exchange points)
  - `TrustLineAmount optimal_flow`: LP-optimized flow through this path
  - `TrustLineAmount received_amount`: Amount received at target after exchanges
  - `double effective_exchange_rate`: End-to-end exchange rate for this path
  - `double path_efficiency`: Ratio of received/sent (higher = better path)
  - `vector<ExchangeStep> exchange_steps`: Detailed exchange information per node
- **Usage**: Enable path-based analytics, alternative solution discovery, sensitivity analysis

#### Enhanced Existing Classes

##### Enhanced Max Flow Messages
All existing max flow calculation message classes gain:
- **New Field**: `vector<SerializedEquivalent> exchangeEquivalents`
- **Backward Compatibility**: Empty vector preserves existing behavior
- **Affected Classes**:
  - `MaxFlowCalculationSourceFstLevelMessage`
  - `MaxFlowCalculationSourceSndLevelMessage` 
  - `MaxFlowCalculationTargetFstLevelMessage`
  - `MaxFlowCalculationTargetSndLevelMessage`
  - Base class: `MaxFlowCalculationMessage`

##### Enhanced Transaction Behaviors
**Source-side transactions** (`MaxFlowCalculationSourceFstLevelTransaction`, `MaxFlowCalculationSourceSndLevelTransaction`):
- **Exchange Rate Logic**: When `exchangeEquivalents` non-empty, search local `ExchangeRatesManager` for rates matching pairs `mEquivalent/exchangeEquivalents[i]`
- **Response**: Send found rates via `ExchangeRatesMessage` alongside topology data

**Target-side transactions** (`MaxFlowCalculationTargetFstLevelTransaction`, `MaxFlowCalculationTargetSndLevelTransaction`):
- **Exchange Rate Logic**: When `exchangeEquivalents` non-empty, search local `ExchangeRatesManager` for rates matching pairs `exchangeEquivalents[i]/mEquivalent` 
- **Response**: Send found rates via `ExchangeRatesMessage` alongside topology data

**Legacy transactions** (`CollectTopologyTransaction`, `InitiateMaxFlowCalculationTransaction`, `ReceiveMaxFlowCalculationOnTargetTransaction`):
- **Compatibility**: Always pass empty `exchangeEquivalents` vector in outgoing messages

### API Documentation
Detailed API specifications will be provided in task-level documentation for all components listed above.

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-01-27 | Architect | Initial draft for multi-equivalent exchange flow calculation | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [03-exchange-rates-manager.md](03-exchange-rates-manager.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)