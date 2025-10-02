# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Payment Estimation for Exchange Flows
- **PRD ID**: 05
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.1
- **Date**: 2025-10-01
- **Author(s)**: Claude Code, based on Architect's requirements
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-10-01
- **Previous PRD**: [04-exchange-flow-calculation.md](04-exchange-flow-calculation.md)
- **Related Documents**: [Exchange Flow Calculation PRD](04-exchange-flow-calculation.md), [Topology Collection Protocol](../../../architecture/vtcpd/protocols/topology-collection-protocol.md)

## Executive Summary
This PRD introduces payment estimation capabilities that enable users to calculate either required payment amounts or expected receive amounts based on previously computed optimal exchange flow paths. Building upon the Exchange Flow Calculation (PRD 04), this feature provides bidirectional estimation without triggering new topology collection or flow calculation.

### Current project state
- Exchange Flow Calculation (PRD 04) computes optimal cross-equivalent payment paths using OR-Tools
- `InitiateMaxFlowExchangeCalculationTransaction` produces `mOptimalPathResults` with detailed path information
- Optimal paths are sorted by efficiency but not persisted for reuse

### This iteration's focus
- Create `ExchangePathsManager` to store and manage optimal path results per (contractor, sender_equivalent, receiver_equivalent)
- Implement bidirectional estimation: payment amount ↔ receive amount
- Provide estimation based on cached optimal paths without recalculating topology
- Handle commissions, exchange limits, and capacity constraints during estimation

### Connection to overall vision
This enables efficient payment planning in the vTCP network by allowing users to estimate costs and outcomes before initiating actual payments, improving user experience and reducing unnecessary topology collection overhead.

## Iteration Context
### Previous Iterations Summary
- **PRD 01**: Database provider abstraction established multi-database support
- **PRD 03**: Exchange Rates Manager provides exchange rate storage with TTL management
- **PRD 04**: Exchange Flow Calculation implements cross-equivalent max flow computation using OR-Tools with optimal path enumeration and commission handling
- **Completed Features**: Topology collection, max flow calculation, optimal path computation with commissions

### Lessons Learned
- OR-Tools LP optimization provides detailed path information that can be reused
- Commission "charge once" semantics are critical for accurate flow simulation
- Path ordering by efficiency enables predictable estimation results
- Exchange rate limits must be validated during flow distribution

### Current State Analysis
- **What's working well**: Optimal path calculation produces comprehensive results
- **Pain points identified**: Path results are not persisted; each estimation would require full topology collection
- **Performance metrics**: OR-Tools optimization completes within acceptable timeframes for typical topologies

## Problem Statement
### Background
After computing maximum receivable flow using `InitiateMaxFlowExchangeCalculationTransaction`, users need to estimate:
1. How much they need to pay (in sender equivalent) to receive a specific amount (in receiver equivalent)
2. How much they will receive (in receiver equivalent) if they pay a specific amount (in sender equivalent)

Currently, this information exists in `mOptimalPathResults` within the transaction but is not accessible for subsequent queries, forcing users to either:
- Trigger new topology collection (expensive)
- Implement manual estimation (error-prone)

### Problem Description
**Who is affected**: All network participants planning cross-equivalent payments
**When and where**: During payment preparation and cost estimation phases
**Current limitations**:
- No access to computed optimal paths after max flow calculation completes
- No estimation API for payment/receive amount calculations
- Users cannot plan payment amounts effectively without triggering expensive operations

### Impact of not solving this problem
- Increased network overhead from redundant topology collections
- Poor user experience due to inability to estimate payment costs
- Potential for payment failures due to inaccurate manual estimations

### Success Metrics
**Primary KPIs**:
- Estimation accuracy matches actual flow simulation
- Zero additional topology collection overhead for estimation queries
- Estimation completes within 100ms for typical path sets

**Target Values**:
- 100% accuracy when optimal paths remain valid (within TTL)
- Support estimation for up to 100 cached optimal path results
- Handle edge cases (insufficient funds, no paths) with appropriate error codes

## Goals
The primary goals of this iteration are to provide efficient and accurate payment estimation capabilities.

*   **Goal 1: Enable bidirectional payment estimation.**
    *   **Description:** Allow users to estimate required payment amounts for desired receive amounts, and vice versa, using cached optimal paths.
    *   **Success Metric:** Estimation commands return accurate results matching flow simulation for 100% of valid cached paths within TTL.

*   **Goal 2: Minimize network overhead.**
    *   **Description:** Provide estimations using cached optimal paths without triggering topology collection or flow recalculation.
    *   **Success Metric:** Zero topology collection messages generated during estimation operations; all estimations served from cache.

*   **Goal 3: Handle edge cases gracefully.**
    *   **Description:** Detect and report situations where estimation cannot be performed (no cached data, insufficient capacity, amounts exceed limits).
    *   **Success Metric:** All error conditions return appropriate error codes (401, 412, 462) with clear semantics.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **ExchangePathsManager**: Component for storing and managing optimal path results indexed by (contractor, sender_equivalent, receiver_equivalent)
2. **Payment estimation command**: Calculate required payment amount for desired receive amount
3. **Receive estimation command**: Calculate expected receive amount for given payment amount
4. **Path-based simulation**: Accurate flow simulation accounting for commissions, exchanges, and capacity
5. **TTL management**: Automatic expiration of cached optimal paths after 600 seconds using boost::asio timer (similar to ExchangeRatesManager and TopologyTrustLinesManager patterns)

#### Technical Infrastructure
- New manager class `ExchangePathsManager` in `src/core/paths/`
- Data structures: `ExchangePath`, `OptimalPathResult` extracted to separate headers
- New command classes: `EstimatePaymentForReceiveAmountCommand`, `EstimateReceiveForPaymentAmountCommand`
- New transaction classes: `EstimatePaymentForReceiveAmountTransaction`, `EstimateReceiveForPaymentAmountTransaction`

#### Integration Points
- Extension of `InitiateMaxFlowExchangeCalculationTransaction` to populate `ExchangePathsManager` (splitting paths by sender equivalent)
- Reuse of commission lookup and exchange rate validation logic
- **Note**: Integration with `CoordinatorPaymentTransaction` for automatic path invalidation is deferred to future iteration when transaction supports multi-equivalent payments

### Explicitly Out of Scope
- Real-time path updates based on topology changes (handled by TTL expiration)
- Cryptographic validation of cached path data
- Estimation for multiple sender equivalents simultaneously (single sender equivalent per query)
- UI/API for browsing cached paths
- Integration with `CoordinatorPaymentTransaction` for automatic path invalidation (deferred until transaction supports multi-equivalent payments)

### Dependencies from Previous Iterations
- **PRD 04**: `InitiateMaxFlowExchangeCalculationTransaction` and optimal path calculation infrastructure
- **PRD 03**: `ExchangeRatesManager` for exchange rate validation
- **Existing topology infrastructure**: Commission lookup and trust line management

### Future Roadmap Impact
This iteration establishes foundation for:
- **Real-time payment planning**: Users can quickly evaluate multiple payment scenarios
- **Path analytics**: Historical path usage and efficiency tracking
- **Predictive estimation**: ML-based estimation improvements using cached path history
- **Alternative path suggestions**: Recommendation system for optimal payment routes
- **Multi-sender-equivalent estimation**: Future extension to estimate across multiple sender equivalents
- **Automatic path invalidation**: Integration with multi-equivalent `CoordinatorPaymentTransaction` for automatic cache invalidation after payments

## User Stories & Requirements

### User Personas
#### Primary User: Payment Initiator
- **Role**: Node operator initiating cross-equivalent payments
- **Goals**: Accurately estimate payment costs before committing funds
- **Pain Points**: Cannot estimate costs without triggering expensive topology collection
- **Technical Proficiency**: Intermediate to Advanced

#### Secondary User: Payment Receiver
- **Role**: Node receiving cross-equivalent payments
- **Goals**: Understand how much will be received for various sender payment amounts
- **Pain Points**: No visibility into conversion costs and path efficiency
- **Technical Proficiency**: Intermediate

### Functional Requirements
#### New Features for This Iteration

1. **ExchangePathsManager**
   - **Description**: Central component for storing and managing optimal path results from max flow calculations, indexed by (contractor, sender_equivalent, receiver_equivalent) triplet
   - **User Story**: As a payment system, I want to cache computed optimal paths per sender-receiver equivalent pair so that subsequent estimation queries don't require recalculation
   - **Rationale**: Eliminates redundant topology collection and flow calculation overhead; enables efficient lookups by equivalent pairs
   - **Builds Upon**: `InitiateMaxFlowExchangeCalculationTransaction` from PRD 04
   - **Acceptance Criteria**:
     - Stores `OptimalPathResult` structures per (contractor, sender_equivalent, receiver_equivalent) key
     - Implements 600-second TTL with automatic cleanup
     - Provides thread-safe access to cached paths
     - Supports invalidation by contractor, by equivalent, or by specific key
   - **Priority**: High
   - **Dependencies**: None

2. **Payment Amount Estimation (Receive → Payment)**
   - **Description**: Calculate required payment amount in sender equivalent to achieve desired receive amount in receiver equivalent
   - **User Story**: As a payment initiator, I want to know how much I need to pay in my equivalent to ensure the receiver gets a specific amount in their equivalent
   - **Rationale**: Enables accurate cost planning for fixed-receive-amount scenarios
   - **Builds Upon**: Cached optimal paths from `ExchangePathsManager`
   - **Acceptance Criteria**:
     - Command: `GET:contractors/transactions/estimate/payment`
     - Looks up paths for (contractor, sender_equivalent, receiver_equivalent)
     - Iterates through ordered optimal paths until desired receive amount is achieved
     - Accounts for commissions with "charge once" semantics
     - Validates exchange min/max limits
     - Returns error 412 if paths cannot deliver desired amount
     - Returns error 462 if no cached paths exist for the key
   - **Priority**: High
   - **Dependencies**: ExchangePathsManager

3. **Receive Amount Estimation (Payment → Receive)**
   - **Description**: Calculate expected receive amount in receiver equivalent for given payment amount in sender equivalent
   - **User Story**: As a payment initiator, I want to know how much the receiver will get in their equivalent if I pay a specific amount in my equivalent
   - **Rationale**: Enables planning for fixed-payment-amount scenarios
   - **Builds Upon**: Cached optimal paths from `ExchangePathsManager`
   - **Acceptance Criteria**:
     - Command: `GET:contractors/transactions/estimate/receive`
     - Looks up paths for (contractor, sender_equivalent, receiver_equivalent)
     - Iterates through ordered optimal paths distributing payment amount
     - Accounts for commissions with "charge once" semantics
     - Validates exchange min/max limits
     - Returns delivered amount (may be less than optimal flow if payment insufficient)
     - Returns error 462 if no cached paths exist for the key
   - **Priority**: High
   - **Dependencies**: ExchangePathsManager

4. **Path Flow Simulation**
   - **Description**: Accurate simulation of flow through exchange paths accounting for all constraints
   - **User Story**: As an estimation system, I need to simulate flow through paths exactly as actual payments would execute
   - **Rationale**: Ensures estimation accuracy matches runtime behavior
   - **Builds Upon**: Commission and exchange rate logic from PRD 04
   - **Acceptance Criteria**:
     - Simulates flow through path segments applying cumulative exchange rates
     - Applies commissions only once per (ContractorID, SerializedEquivalent) pair
     - Validates exchange amounts against min/max limits
     - Respects edge capacity constraints
     - Returns accurate delivered amount at receiver
   - **Priority**: High
   - **Dependencies**: ExchangePathsManager, existing commission/exchange infrastructure

5. **Automatic Path Invalidation (Future Work)**
   - **Description**: Invalidate cached paths when multi-equivalent payments execute
   - **User Story**: As a payment system, I want to ensure estimations reflect current network state by invalidating stale paths after payments
   - **Rationale**: Prevents estimation based on outdated capacity information after network topology changes
   - **Builds Upon**: Multi-equivalent payment execution infrastructure
   - **Acceptance Criteria**:
     - **Note**: Implementation deferred until `CoordinatorPaymentTransaction` supports multi-equivalent payments
     - `CoordinatorPaymentTransaction` will notify `ExchangePathsManager` upon execution
     - Invalidation uses contractor ID and payment equivalents from transaction
     - Paths invalidated for affected (contractor, sender_equivalent, receiver_equivalent) keys
     - Invalidation occurs regardless of payment success/failure
   - **Priority**: Low (deferred to future iteration)
   - **Dependencies**: Multi-equivalent `CoordinatorPaymentTransaction` (not yet implemented)
   - **Implementation Note**: PRD documents the intended design; actual integration will occur when `CoordinatorPaymentTransaction` is refactored for multi-equivalent support

#### Enhancements to Existing Features

1. **InitiateMaxFlowExchangeCalculationTransaction Enhancement**
   - **Current State**: Computes optimal paths and stores in local `mOptimalPathResults`
   - **Proposed Changes**:
     - After OR-Tools optimization, split `mOptimalPathResults` by sender equivalent
     - For each sender equivalent, transfer paths starting with that equivalent to `ExchangePathsManager` with key (contractor, sender_equivalent, receiver_equivalent)
     - Associate paths with timestamp for TTL management
   - **Impact Assessment**: Minimal - adds path splitting and storage logic after optimization completes
   - **Migration Strategy**: No migration needed; purely additive change

2. **Data Structure Extraction**
   - **Current State**: `ExchangePath` and `OptimalPathResult` defined inline in transaction implementation
   - **Proposed Changes**:
     - Extract to `src/core/paths/lib/ExchangePath.h`
     - Extract to `src/core/paths/lib/OptimalPathResult.h`
     - Make structures reusable across transactions and manager
   - **Impact Assessment**: Refactoring only; no behavioral changes
   - **Migration Strategy**: Direct code extraction with header includes

### Non-Functional Requirements
#### Performance
- Estimation query completes within 100ms for typical path sets (≤50 paths)
- Path storage memory overhead < 10MB for 100 cached contractor-equivalent pairs
- TTL cleanup runs efficiently without blocking estimation queries

#### Security
- Cached paths contain no sensitive cryptographic material
- Access to estimation API follows existing authentication patterns
- No validation of path authenticity (trust based on TTL freshness)

#### Scalability
- Support up to 100 cached (contractor, sender_eq, receiver_eq) keys simultaneously
- Handle path sets up to 100 paths per key
- TTL cleanup scales linearly with number of cached entries

#### Reliability
- Estimation failures return appropriate error codes (never crash)
- TTL expiration ensures stale data automatically purged
- Thread-safe access to cached paths prevents race conditions

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: `InitiateMaxFlowExchangeCalculationTransaction` computes and discards optimal paths
- **Proposed Changes**: Introduce `ExchangePathsManager` as central cache for optimal paths indexed by (contractor, sender_eq, receiver_eq), enable estimation transactions to query cache by specific equivalent pairs
- **Backwards Compatibility**: New components; no changes to existing max flow calculation behavior
- **Migration Requirements**: None; purely additive

### Technology Stack Updates
#### New Technologies/Libraries
- No new external libraries required
- Reuses existing OR-Tools, commission, and exchange rate infrastructure

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- `ExchangePathsManager` integrates with `InitiateMaxFlowExchangeCalculationTransaction`
- Estimation transactions integrate with `ExchangePathsManager`
- Payment transactions integrate with `ExchangePathsManager` for invalidation

#### Modified Integrations
- None; all integrations are new

### Command Format Specifications

#### EstimatePaymentForReceiveAmount Command
**Identifier**: `GET:contractors/transactions/estimate/payment`

**Format**:
```
GET:contractors/transactions/estimate/payment:<address_type>:<address>:<receive_amount>:<receiver_equivalent>:<sender_equivalent>
```

**Parameters**:
- `address_type`: Type of contractor address (e.g., `12` for IPv4)
- `address`: Address of target contractor (e.g., `127.0.0.1:2003`)
- `receive_amount`: Desired amount to be received (in receiver equivalent)
- `receiver_equivalent`: Equivalent in which receiver accepts funds
- `sender_equivalent`: Equivalent in which sender will pay

**Example**:
```
GET:contractors/transactions/estimate/payment:12:127.0.0.1:2003:1000:2:1
```
(Estimate payment needed in equivalent 1 to deliver 1000 units in equivalent 2)

**Response Format** (success):
```
200:<total_payment_amount>
```

**Response Format** (error):
```
<error_code>
```

**Error Codes**:
- `401`: Unexpected error (protocol error)
- `412`: Insufficient paths to deliver requested receive amount
- `462`: No cached optimal paths for specified (contractor, sender_equivalent, receiver_equivalent)

#### EstimateReceiveForPaymentAmount Command
**Identifier**: `GET:contractors/transactions/estimate/receive`

**Format**:
```
GET:contractors/transactions/estimate/receive:<address_type>:<address>:<payment_amount>:<sender_equivalent>:<receiver_equivalent>
```

**Parameters**:
- `address_type`: Type of contractor address (e.g., `12` for IPv4)
- `address`: Address of target contractor (e.g., `127.0.0.1:2003`)
- `payment_amount`: Amount to be paid (in sender equivalent)
- `sender_equivalent`: Equivalent in which sender pays
- `receiver_equivalent`: Equivalent in which receiver accepts funds

**Example**:
```
GET:contractors/transactions/estimate/receive:12:127.0.0.1:2003:500:1:2
```
(Estimate receive amount in equivalent 2 if paying 500 units in equivalent 1)

**Response Format** (success):
```
200:<total_receive_amount>
```

**Response Format** (error):
```
<error_code>
```

**Error Codes**:
- `401`: Unexpected error (protocol error)
- `462`: No cached optimal paths for specified (contractor, sender_equivalent, receiver_equivalent)

### Data Requirements
#### Data Models

##### ExchangePath (extracted to ExchangePath.h)
**Purpose**: Represents a single feasible payment path through the network

**Fields**:
- `vector<ContractorID> nodes`: Sequence of node IDs along the path
- `vector<SerializedEquivalent> equivalents`: Equivalent at each node (parallel to nodes)
- `vector<ExchangeStep> exchangeSteps`: Exchange operations performed along path
- `TrustLineAmount minCapacity`: Maximum flow this path can handle (accounting for commissions)
- `double effectiveExchangeRate`: Cumulative exchange rate (product of all exchanges)
- `TrustLineAmount totalCommissions`: Sum of fixed commissions along path (for reference)

**Methods**:
- `TrustLineAmount calculateMaxCapacity() const`: Returns minCapacity
- `double calculateEffectiveExchangeRate() const`: Returns effectiveExchangeRate
- `bool startsWithEquivalent(SerializedEquivalent eq) const`: Checks if path starts in given equivalent
- `bool isValid() const`: Validates path structure consistency

##### ExchangeStep (within ExchangePath.h)
**Purpose**: Describes a single exchange operation at a node

**Fields**:
- `ContractorID nodeID`: Node performing the exchange
- `SerializedEquivalent fromEquivalent`: Source equivalent
- `SerializedEquivalent toEquivalent`: Target equivalent
- `TrustLineAmount exchangeRate`: Exchange rate numerator
- `int16_t exchangeRateShift`: Decimal shift for rate precision
- `TrustLineAmount minExchangeAmount`: Minimum exchange amount
- `TrustLineAmount maxExchangeAmount`: Maximum exchange amount
- `TrustLineAmount commission`: Exchange commission (currently unused, always 0)

##### OptimalPathResult (extracted to OptimalPathResult.h)
**Purpose**: Stores result of OR-Tools optimization for a single path

**Fields**:
- `ExchangePath path`: The path structure
- `TrustLineAmount optimal_flow`: LP solver output (raw flow before commission simulation)
- `TrustLineAmount received_amount`: Amount delivered to receiver after all deductions
- `double effective_exchange_rate`: End-to-end exchange rate
- `double path_efficiency`: Ratio of received/sent (accounting for commissions)

**Notes**:
- `optimal_flow` may differ from actual usable flow due to commission deduplication during post-processing
- `received_amount` reflects actual delivered amount after "charge once" commission semantics

##### EdgeKey (helper structure for estimation algorithms)
**Purpose**: Unique identifier for a directed edge in the network topology

**Fields**:
```cpp
struct EdgeKey {
    ContractorID from;
    ContractorID to;
    SerializedEquivalent equivalent;

    bool operator<(const EdgeKey& other) const noexcept {
        if (from != other.from) return from < other.from;
        if (to != other.to) return to < other.to;
        return equivalent < other.equivalent;
    }
};
```

**Usage**:
- Used as map key to track remaining edge capacity during flow simulation
- Distinguishes between parallel edges in different equivalents (e.g., A→B in eq 1 vs A→B in eq 2)
- Distinguishes between bidirectional edges (A→B vs B→A in same equivalent)
- Enables accurate capacity tracking when multiple paths share the same physical edge

**Example**:
```cpp
// Track capacity consumption across paths
map<EdgeKey, double> edgeRemainingCapacity;

// Path 1 uses A→B in equivalent 1
EdgeKey key1{nodeA, nodeB, equivalent1};
edgeRemainingCapacity[key1] = initialCapacity - consumedByPath1;

// Path 2 uses A→B in equivalent 2 (different edge, same physical link)
EdgeKey key2{nodeA, nodeB, equivalent2};
edgeRemainingCapacity[key2] = initialCapacity - consumedByPath2;

// Reverse direction B→A in equivalent 1 (different edge)
EdgeKey key3{nodeB, nodeA, equivalent1};
edgeRemainingCapacity[key3] = reverseCapacity;
```

##### PathCacheKey (within ExchangePathsManager)
**Purpose**: Unique identifier for cached path set

**Fields**:
```cpp
struct PathCacheKey {
    ContractorID contractor;
    SerializedEquivalent senderEquivalent;
    SerializedEquivalent receiverEquivalent;

    bool operator<(const PathCacheKey& other) const {
        if (contractor != other.contractor) return contractor < other.contractor;
        if (senderEquivalent != other.senderEquivalent)
            return senderEquivalent < other.senderEquivalent;
        return receiverEquivalent < other.receiverEquivalent;
    }
};
```

**Notes**:
- Used as map key in `ExchangePathsManager`
- Supports same-equivalent scenarios (senderEquivalent == receiverEquivalent)
- Enables efficient lookup by specific equivalent pair

##### ExchangePathsManager
**Purpose**: Central cache for optimal path results from max flow calculations

**Storage**:
```cpp
struct CachedPathResult {
    vector<OptimalPathResult> paths;
    DateTime computedAt;
};

map<PathCacheKey, CachedPathResult> mCachedPaths;
```

**Constants**:
- `kPathResultsTTLSeconds = 600` (5 minutes)

**Key Methods**:
- `void storePaths(const PathCacheKey& key, const vector<OptimalPathResult>& paths)`: Store optimal paths for specific key
- `optional<vector<OptimalPathResult>> retrievePaths(const PathCacheKey& key)`: Retrieve cached paths if valid (within TTL)
- `void invalidatePaths(const PathCacheKey& key)`: Remove cached paths for specific key
- `void invalidatePathsForContractor(ContractorID contractor)`: Remove all cached paths for contractor
- `void invalidatePathsForEquivalent(SerializedEquivalent equivalent)`: Remove all cached paths involving equivalent (as sender or receiver)

**Private Methods (TTL Management)**:
- `void scheduleExpiryTimer()`: Schedule next expiry timer based on earliest expiry time
- `void onExpiryTimer(const boost::system::error_code& error)`: Timer callback to trigger cleanup
- `DateTime earliestExpiryTime() const`: Find earliest expiry time among cached paths
- `void removeExpiredPaths()`: Remove all paths exceeding TTL

**TTL Timer Implementation**:
- Uses `boost::asio::steady_timer` for automatic expiry scheduling
- Timer pattern follows `ExchangeRatesManager` (mExpiryTimer) and `TopologyTrustLinesManager` (mCommissionExpiryTimer)
- Timer reschedules itself after each cleanup based on next earliest expiry
- Requires `boost::asio::io_context` reference passed to constructor

**Data Members**:
```cpp
map<PathCacheKey, CachedPathResult> mCachedPaths;
mutex mCacheMutex;
Logger &mLog;
as::io_context &mIOContext;
unique_ptr<as::steady_timer> mExpiryTimer;
```

**Thread Safety**:
- Uses mutex for concurrent access protection
- Read operations lock only for duration of map access
- Timer callbacks execute on IO context thread, acquire mutex for cleanup

#### Data Storage
- All path caching is in-memory only
- No persistent storage for optimal paths
- TTL-based expiration ensures bounded memory usage

#### Data Migration
- No migration required; purely new functionality

### Estimation Algorithm Specifications

#### Payment Estimation Algorithm (Receive → Payment)
**Goal**: Determine minimum payment amount in sender equivalent needed to deliver `targetReceiveAmount` in receiver equivalent.

**Input**:
- `contractorID`: Target contractor
- `targetReceiveAmount`: Desired receive amount
- `receiverEquivalent`: Equivalent in which to receive
- `senderEquivalent`: Equivalent in which to pay

**Algorithm**:
```cpp
TrustLineAmount estimatePaymentForReceive(
    ContractorID contractorID,
    TrustLineAmount targetReceiveAmount,
    SerializedEquivalent receiverEquivalent,
    SerializedEquivalent senderEquivalent)
{
    // Step 1: Retrieve cached paths for specific key
    PathCacheKey key{contractorID, senderEquivalent, receiverEquivalent};
    auto cachedPaths = mExchangePathsManager->retrievePaths(key);
    if (!cachedPaths) {
        throw NoPathsError(462); // No cached paths for this key
    }

    // Step 2: Paths are already sorted by efficiency (from max flow calculation)
    // Iterate through paths, simulating flow distribution

    TrustLineAmount remainingReceive = targetReceiveAmount;
    TrustLineAmount totalPayment = TrustLineAmount(0);

    set<pair<ContractorID, SerializedEquivalent>> appliedCommissions;
    map<EdgeKey, double> edgeRemainingCapacity; // Track consumed capacity

    for (const auto& pathResult : *cachedPaths) {
        if (remainingReceive == TrustLineAmount(0)) break;

        // Simulate: how much payment needed on this path to deliver remainingReceive?
        // This requires inverse simulation accounting for exchanges and commissions

        double targetForPath = std::min(
            remainingReceive.convert_to<double>(),
            pathResult.received_amount.convert_to<double>()
        );

        // Inverse transform through path (work backwards from receiver to sender)
        double requiredPayment = inverseSimulatePath(
            pathResult.path,
            targetForPath,
            appliedCommissions,
            edgeRemainingCapacity
        );

        if (requiredPayment <= 0.0) continue; // Path exhausted

        totalPayment = totalPayment + TrustLineAmount(static_cast<uint64_t>(requiredPayment));
        remainingReceive = remainingReceive - TrustLineAmount(static_cast<uint64_t>(targetForPath));
    }

    // Step 3: Check if target achieved
    if (remainingReceive > TrustLineAmount(0)) {
        throw InsufficientPathsError(412); // Cannot deliver requested amount
    }

    return totalPayment;
}
```

**Key Helper**: `inverseSimulatePath`
- Works backwards through path from receiver to sender
- Applies inverse exchange rates: `inputAmount = outputAmount / exchangeRate`
- Accounts for commissions: if commission was deducted on forward pass, add it back to compute required input
- Validates edge capacity: ensure required input doesn't exceed available capacity
- Returns required payment amount at sender to achieve target output at receiver

#### Receive Estimation Algorithm (Payment → Receive)
**Goal**: Determine receive amount in receiver equivalent when paying `paymentAmount` in sender equivalent.

**Input**:
- `contractorID`: Target contractor
- `paymentAmount`: Amount to pay
- `senderEquivalent`: Equivalent in which to pay
- `receiverEquivalent`: Equivalent in which to receive

**Algorithm**:
```cpp
TrustLineAmount estimateReceiveForPayment(
    ContractorID contractorID,
    TrustLineAmount paymentAmount,
    SerializedEquivalent senderEquivalent,
    SerializedEquivalent receiverEquivalent)
{
    // Step 1: Retrieve cached paths for specific key
    PathCacheKey key{contractorID, senderEquivalent, receiverEquivalent};
    auto cachedPaths = mExchangePathsManager->retrievePaths(key);
    if (!cachedPaths) {
        throw NoPathsError(462); // No cached paths for this key
    }

    // Step 2: Paths are already sorted by efficiency
    // Distribute payment across paths, simulating flow

    TrustLineAmount remainingPayment = paymentAmount;
    TrustLineAmount totalReceive = TrustLineAmount(0);

    set<pair<ContractorID, SerializedEquivalent>> appliedCommissions;
    map<EdgeKey, double> edgeRemainingCapacity;

    for (const auto& pathResult : *cachedPaths) {
        if (remainingPayment == TrustLineAmount(0)) break;

        // Determine how much of remainingPayment can flow through this path
        double maxPathInput = pathResult.optimal_flow.convert_to<double>();
        double pathInput = std::min(
            remainingPayment.convert_to<double>(),
            maxPathInput
        );

        // Forward simulate to get output amount
        double pathOutput = forwardSimulatePath(
            pathResult.path,
            pathInput,
            appliedCommissions,
            edgeRemainingCapacity
        );

        if (pathOutput <= 0.0) continue; // Path produced no output

        totalReceive = totalReceive + TrustLineAmount(static_cast<uint64_t>(pathOutput));
        remainingPayment = remainingPayment - TrustLineAmount(static_cast<uint64_t>(pathInput));
    }

    // No error if remainingPayment > 0; just return what was delivered
    return totalReceive;
}
```

**Key Helper**: `forwardSimulatePath`
- Reuses logic from `InitiateMaxFlowExchangeCalculationTransaction::simulatePathNetAmount`
- Applies exchange rates: `outputAmount = inputAmount * exchangeRate`
- Deducts commissions with "charge once" semantics (checks `appliedCommissions` set)
- Validates edge capacity: reduces flow if edge insufficient
- Returns delivered amount at receiver

#### Commission Handling Details
**"Charge Once" Semantics**:
- Maintain `set<pair<ContractorID, SerializedEquivalent>> appliedCommissions` across all paths in estimation
- When simulating flow through a path segment:
  - Check if `(nodeID, equivalent)` is in `appliedCommissions`
  - If yes: skip commission deduction (already charged on previous path)
  - If no: deduct commission and add `(nodeID, equivalent)` to set

**Commission Refresh**:
- Estimations always fetch current commission values from `TopologyTrustLinesManager`
- If commissions changed since max flow calculation, new values apply
- This ensures estimation reflects current network state within cache TTL

#### Exchange Limit Validation
**Per-Step Validation**:
- When simulating flow through exchange step:
  - Extract `minExchangeAmount` and `maxExchangeAmount` from `ExchangeStep`
  - Check if flow amount falls within `[min, max]` range
  - If below min: skip path (cannot use exchange)
  - If above max: cap flow at max
  - If within range: proceed normally

**Multi-Step Exchanges**:
- Each exchange step validated independently
- Cumulative flow adjusted based on tightest constraint along path

### Error Handling Specifications

#### Error Code 401 (Unexpected Error)
**Trigger Conditions**:
- Exception thrown during estimation logic
- Invalid data structures encountered
- Memory allocation failures

**Response**:
```
401
```

**Logging**:
```cpp
error() << "Estimation failed with unexpected error: " << exception.what();
```

#### Error Code 412 (Insufficient Paths)
**Trigger Conditions**:
- Payment estimation: Cannot deliver requested `targetReceiveAmount` with available paths
- All paths exhausted but `remainingReceive > 0`

**Response**:
```
412
```

**Logging**:
```cpp
warning() << "Insufficient paths to deliver " << targetReceiveAmount
          << " to contractor " << contractorID
          << "; delivered only " << (targetReceiveAmount - remainingReceive);
```

#### Error Code 462 (No Cached Paths)
**Trigger Conditions**:
- No cached `OptimalPathResult` for specified (contractor, sender_equivalent, receiver_equivalent) key
- Cached paths expired (TTL exceeded)

**Response**:
```
462
```

**Logging**:
```cpp
warning() << "No cached optimal paths for contractor " << contractorID
          << " with sender equivalent " << senderEquivalent
          << " and receiver equivalent " << receiverEquivalent;
```

**User Action**:
- Trigger `InitiateMaxFlowExchangeCalculationTransaction` to compute paths
- Retry estimation after max flow calculation completes

## Implementation Plan
### This Iteration Timeline
- **Duration**: 3-4 weeks implementation + 1-2 weeks testing
- **Sprint Breakdown**:
  - Sprint 1 (Week 1): Data structure extraction and `ExchangePathsManager` implementation
  - Sprint 2 (Week 2): Estimation commands and transactions
  - Sprint 3 (Week 3): Flow simulation algorithms and commission handling
  - Sprint 4 (Week 4): Integration, testing, and validation

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Data Structure Extraction | Week 1 | `ExchangePath` and `OptimalPathResult` in separate headers | None | Low |
| ExchangePathsManager Complete | Week 1 | Manager with PathCacheKey and TTL | Data structures | Medium |
| Commands and Transactions Skeleton | Week 2 | Command parsing and transaction stubs | Manager | Low |
| Flow Simulation Algorithms | Week 3 | Forward and inverse path simulation | All previous | High |
| Integration and Testing | Week 4 | Full integration with unit tests | All previous | Medium |

### Dependencies on Other Teams/Projects
- No external team dependencies identified

### Integration Points with Previous Work
- Builds directly upon `InitiateMaxFlowExchangeCalculationTransaction` (PRD 04)
- Reuses commission lookup from `TopologyTrustLinesManager`
- Leverages exchange rate structures from PRD 03

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++ experience
- **Developers**: 1 developer for implementation support
- **QA Engineers**: 1 engineer for unit test validation

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Inverse path simulation complexity | High | Medium | Thorough unit testing with known topologies; leverage existing forward simulation as reference |
| Commission "charge once" bugs | High | Medium | Extract commission logic into testable helper functions; extensive edge case testing |
| Thread safety issues in cache | Medium | Low | Use proven locking patterns; stress test with concurrent access |
| TTL cleanup performance | Low | Low | Implement efficient timer-based cleanup; monitor memory usage in tests |
| Path splitting logic errors | Medium | Low | Validate split paths against original mOptimalPathResults; unit test coverage |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Estimation inaccuracy vs. actual payments | High | Low | Validate estimation algorithm against actual flow simulation; iterate based on discrepancies |

## Testing Strategy
### Testing Approach (Unit-Only)
- All testing is unit-only (no integration/E2E tests)
- Tests are built and executed exclusively in `build-tests`
- No Docker usage; mock all external dependencies
- Provide mock data for both SQLite and PostgreSQL where applicable

#### Unit Tests: New Components

**ExchangePathsManager Tests**:
- Store and retrieve paths with various PathCacheKey combinations
- TTL expiration: store paths, wait, verify auto-cleanup
- Invalidation by key: store multiple keys, invalidate one, verify others remain
- Invalidation by contractor: store paths for same contractor with different equivalents, invalidate contractor, verify all removed
- Invalidation by equivalent: store paths with mixed equivalents, invalidate one equivalent, verify correct subset removed
- Thread safety: concurrent store/retrieve operations
- Edge cases: retrieve non-existent key, store empty path list, same-equivalent scenarios (sender_eq == receiver_eq)

**Path Splitting Tests** (in InitiateMaxFlowExchangeCalculationTransaction):
- Mock `mOptimalPathResults` with paths starting from different sender equivalents
- Verify paths correctly split by sender equivalent
- Verify each split stored with correct PathCacheKey
- Test same-equivalent scenario (all paths use same sender_eq == receiver_eq)

**EstimatePaymentForReceiveAmountTransaction Tests**:

*Test Topology 1: Single-Equivalent Path (No Exchange)*
- Topology: A → B → C in equivalent 1 (sender_eq = receiver_eq = 1)
- Capacities: A→B = 1000, B→C = 800
- Commission: B has 10 unit commission in equivalent 1
- Test: Estimate payment for receive amount 500
  - Expected: ~510 (500 + 10 commission)
- Test: Estimate payment for receive amount 900
  - Expected: Error 412 (insufficient; max deliverable ~790)

*Test Topology 2: Cross-Equivalent Path with Exchange*
- Topology: A (eq 1) → B (eq 1) → X [1→2, rate=2.0] → C (eq 2)
- Capacities: A→B = 1000, B→X = 800, X→C = 1500
- Key: (C, sender_eq=1, receiver_eq=2)
- Test: Estimate payment for receive amount 400 in eq 2
  - Expected: ~200 in eq 1 (400 / 2.0 exchange rate)
- Test: Estimate payment for receive amount 2000 in eq 2
  - Expected: Error 412 (max deliverable ~800 * 2.0 = 1600)

*Test Topology 3: Multi-Path with Commissions*
- Topology (same as PRD 04 commission example):
  - Path 1: A → B → F (2 hops)
  - Path 2: A → B → C → F (3 hops)
  - Path 3: A → B → D → E → F (4 hops)
  - Commission on B: 10 units
- Capacities: A→B=1000, B→F=500, B→C=800, C→F=200, B→D=700, D→E=300, E→F=900
- All in equivalent 2002 (sender_eq = receiver_eq = 2002)
- Key: (F, sender_eq=2002, receiver_eq=2002)
- Test: Estimate payment for receive amount 990
  - Expected: ~1000 (simulation should match optimal flow calculation)
  - Verify commission charged only once across all paths

*Test Topology 4: Exchange Min/Max Limits*
- Topology: A (eq 1) → X [1→2, rate=1.5, min=100, max=500] → B (eq 2)
- Key: (B, sender_eq=1, receiver_eq=2)
- Test: Estimate payment for receive amount 50 in eq 2
  - Expected: Error 412 (below exchange minimum: 50/1.5 ≈ 33 < 100)
- Test: Estimate payment for receive amount 600 in eq 2
  - Expected: ~400 in eq 1 (max exchange output 500*1.5=750, so 600 deliverable)

*Error Case Tests*:
- No cached paths for key: Error 462
- Cached paths expired (mock TTL): Error 462
- Requested receive amount exceeds all path capacity: Error 412

**EstimateReceiveForPaymentAmountTransaction Tests**:

*Test Topology 1: Single-Equivalent Path (No Exchange)*
- Same topology as payment estimation test 1
- Test: Estimate receive for payment amount 510 in eq 1
  - Expected: ~500 (510 - 10 commission)
- Test: Estimate receive for payment amount 2000 in eq 1
  - Expected: ~790 (limited by path capacity, not payment amount)

*Test Topology 2: Cross-Equivalent Path with Exchange*
- Same topology as payment estimation test 2
- Test: Estimate receive for payment amount 200 in eq 1
  - Expected: ~400 in eq 2 (200 * 2.0 exchange rate)
- Test: Estimate receive for payment amount 1000 in eq 1
  - Expected: ~1600 in eq 2 (limited by B→X capacity of 800)

*Test Topology 3: Multi-Path with Commissions*
- Same topology as payment estimation test 3
- Test: Estimate receive for payment amount 1000
  - Expected: ~990 (matches optimal flow calculation)
  - Verify commission charged only once

*Test Topology 4: Exchange Min/Max Limits*
- Same topology as payment estimation test 4
- Test: Estimate receive for payment amount 50 in eq 1
  - Expected: 0 (below exchange minimum)
- Test: Estimate receive for payment amount 600 in eq 1
  - Expected: ~750 in eq 2 (capped at max exchange: 500 * 1.5)

*Error Case Tests*:
- No cached paths for key: Error 462
- Cached paths expired (mock TTL): Error 462

**Flow Simulation Algorithm Tests**:
- `forwardSimulatePath`: Verify exchange rate application, commission deduction, capacity limits
- `inverseSimulatePath`: Verify inverse exchange rates, commission addition, capacity validation
- Commission "charge once": Multiple paths sharing intermediate node, verify commission deducted only once
- Edge capacity tracking: Simulate multiple flows through shared edge, verify capacity consumed correctly

#### Regression Testing (Unit)
- Scope: `InitiateMaxFlowExchangeCalculationTransaction` behavior unchanged
- Verify max flow calculation still produces identical results
- Verify optimal path ordering remains consistent

#### Execution in CI/Locally
- Build tests in `build-tests` and run the produced binaries
- All unit tests must pass before PRD completion

### Quality Gates
- All unit tests pass in `build-tests`
- Estimation results match manual calculations for all test topologies
- Error codes 401, 412, 462 triggered correctly in respective scenarios
- No memory leaks detected in TTL cleanup tests
- Thread safety validated with concurrent access tests
- Path splitting logic validated against original mOptimalPathResults

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Feature addition (compatible with existing nodes)
- **Rollout Strategy**: Direct deployment; commands available immediately after node restart
- **Rollback Plan**: Disable estimation commands; no impact on existing max flow calculation

### Database Migrations
- No database migrations required (in-memory storage only)

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: Node operator guidance on estimation command usage
- **Documentation Updates**: Command reference documentation for estimation APIs

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**:
  - Estimation accuracy: 100% match with flow simulation for all test cases
  - Performance: Estimation completes within 100ms for typical path sets
  - Error handling: All error conditions return correct error codes
- **Leading Indicators**: Successful unit test execution, cache hit rate in manual testing
- **Baseline Values**: N/A (new feature)
- **Target Values**:
  - 100% test pass rate
  - Zero topology collection overhead during estimation
  - All error codes validated in tests

### Monitoring Plan
- **New Dashboards/Alerts**: N/A (no production monitoring for this iteration)
- **Enhanced Monitoring**: Log estimation query frequency and cache hit/miss rates
- **A/B Testing**: Not applicable

### Review Schedule
- **Daily**: Development progress and blocker identification
- **Weekly**: Unit test results and algorithm validation
- **Post-Implementation Review**: Test coverage analysis and edge case validation

## Appendices
### Glossary
- **Optimal Path**: A feasible payment path computed by OR-Tools LP optimization
- **Path Estimation**: Calculation of payment/receive amounts using cached optimal paths without recalculating topology
- **Forward Simulation**: Simulating flow from sender to receiver (payment → receive)
- **Inverse Simulation**: Calculating required input from desired output (receive → payment)
- **Charge Once**: Commission deduction semantics ensuring each intermediate node charges commission only once across all paths
- **PathCacheKey**: Unique identifier combining (contractor, sender_equivalent, receiver_equivalent)

### References
- [Exchange Flow Calculation PRD](04-exchange-flow-calculation.md)
- [Topology Collection Protocol](../../../architecture/vtcpd/protocols/topology-collection-protocol.md)
- [InitiateMaxFlowExchangeCalculationTransaction Implementation](../../../src/core/transactions/transactions/max_flow_calculation/InitiateMaxFlowExchangeCalculationTransaction.cpp)

### Detailed Component Specifications

#### New Command Classes

##### EstimatePaymentForReceiveAmountCommand
- **Inheritance**: Extends `BaseUserCommand`
- **Identifier**: `GET:contractors/transactions/estimate/payment`
- **Input Parameters**:
  - `address_type`: Type of contractor address
  - `address`: Contractor address
  - `receive_amount`: Desired receive amount
  - `receiver_equivalent`: Receiver's equivalent
  - `sender_equivalent`: Sender's equivalent
- **Validation**:
  - Contractor address must be valid and parseable
  - Amounts must be positive
  - Equivalents must be valid SerializedEquivalent values
- **Output**: `EstimatePaymentForReceiveAmountTransaction` initiated

##### EstimateReceiveForPaymentAmountCommand
- **Inheritance**: Extends `BaseUserCommand`
- **Identifier**: `GET:contractors/transactions/estimate/receive`
- **Input Parameters**:
  - `address_type`: Type of contractor address
  - `address`: Contractor address
  - `payment_amount`: Payment amount
  - `sender_equivalent`: Sender's equivalent
  - `receiver_equivalent`: Receiver's equivalent
- **Validation**:
  - Contractor address must be valid and parseable
  - Amounts must be positive
  - Equivalents must be valid SerializedEquivalent values
- **Output**: `EstimateReceiveForPaymentAmountTransaction` initiated

#### New Transaction Classes

##### EstimatePaymentForReceiveAmountTransaction
- **Inheritance**: Extends `BaseTransaction`
- **Purpose**: Execute payment estimation for given receive amount
- **Input**: `EstimatePaymentForReceiveAmountCommand`
- **Key Methods**:
  - `run()`: Main transaction execution
  - `estimatePayment()`: Core estimation algorithm
  - `resultOk(TrustLineAmount payment)`: Success result
  - `resultError(int errorCode)`: Error result
- **Dependencies**:
  - `ExchangePathsManager`: Retrieve cached paths by PathCacheKey
  - `TopologyTrustLinesManager`: Fetch current commissions
  - `EquivalentsSubsystemsRouter`: Access equivalent-specific managers

##### EstimateReceiveForPaymentAmountTransaction
- **Inheritance**: Extends `BaseTransaction`
- **Purpose**: Execute receive estimation for given payment amount
- **Input**: `EstimateReceiveForPaymentAmountCommand`
- **Key Methods**:
  - `run()`: Main transaction execution
  - `estimateReceive()`: Core estimation algorithm
  - `resultOk(TrustLineAmount receive)`: Success result
  - `resultError(int errorCode)`: Error result
- **Dependencies**:
  - `ExchangePathsManager`: Retrieve cached paths by PathCacheKey
  - `TopologyTrustLinesManager`: Fetch current commissions
  - `EquivalentsSubsystemsRouter`: Access equivalent-specific managers

#### New Manager Class

##### ExchangePathsManager
- **Location**: `src/core/paths/ExchangePathsManager.h/cpp`
- **Purpose**: Centralized cache for optimal path results indexed by (contractor, sender_equivalent, receiver_equivalent)
- **Inheritance**: None (standalone manager)
- **Constructor**:
  ```cpp
  ExchangePathsManager(
      as::io_context &ioContext,
      Logger &logger)
  ```
- **Key Methods**:
  ```cpp
  void storePaths(
      const PathCacheKey& key,
      const vector<OptimalPathResult>& paths);

  optional<vector<OptimalPathResult>> retrievePaths(
      const PathCacheKey& key);

  void invalidatePaths(const PathCacheKey& key);

  void invalidatePathsForContractor(ContractorID contractor);

  void invalidatePathsForEquivalent(SerializedEquivalent equivalent);

  void cleanupExpiredPaths();
  ```
- **Data Members**: See "Data Models" section above for complete `PathCacheKey`, `CachedPathResult`, and TTL timer details
- **Thread Safety**: All public methods acquire `mCacheMutex` before accessing `mCachedPaths`

#### Modified Classes

##### InitiateMaxFlowExchangeCalculationTransaction (Enhanced)
- **New Dependency**: `ExchangePathsManager* mExchangePathsManager`
- **Constructor Change**: Add `ExchangePathsManager*` parameter
- **New Method Call** (in `applyCustomLogic()` after OR-Tools optimization):
  ```cpp
  // After calculating mOptimalPathResults[contractorID]
  // Split paths by sender equivalent and store each group separately
  map<SerializedEquivalent, vector<OptimalPathResult>> pathsBySenderEq;
  for (const auto& pathResult : mOptimalPathResults[contractorID]) {
      SerializedEquivalent senderEq = pathResult.path.equivalents.front();
      pathsBySenderEq[senderEq].push_back(pathResult);
  }

  // Store each sender equivalent group with appropriate key
  for (const auto& [senderEq, paths] : pathsBySenderEq) {
      PathCacheKey key{contractorID, senderEq, mEquivalent};
      mExchangePathsManager->storePaths(key, paths);
  }
  ```

### API Documentation
Detailed API specifications will be provided in task-level documentation for all components listed above.

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-10-01 | Claude Code | Initial draft for payment estimation feature | Phase 1 |
| 1.1 | 2025-10-01 | Claude Code | Simplified to single sender_equivalent per query; introduced PathCacheKey structure | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [04-exchange-flow-calculation.md](04-exchange-flow-calculation.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)
- **Topology Collection Protocol**: [topology-collection-protocol.md](../../../architecture/vtcpd/protocols/topology-collection-protocol.md)
