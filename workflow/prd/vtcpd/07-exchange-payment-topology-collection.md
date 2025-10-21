# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Exchange Payment Topology Collection Integration
- **PRD ID**: 07
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.1
- **Date**: 2025-10-18
- **Author(s)**: Claude Code, based on Architect's requirements
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-10-18
- **Previous PRD**: [06-exchange-payment-with-commissions.md](06-exchange-payment-with-commissions.md)
- **Related Documents**:
  - [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
  - [Exchange Flow Calculation PRD](04-exchange-flow-calculation.md)
  - [Payment Estimation PRD](05-payment-estimation.md)
  - [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This PRD introduces automatic topology collection and path building into the exchange payment flow. Currently, `CoordinatorExchangePaymentTransaction` (PRD 06) assumes that optimal exchange paths are already cached in `ExchangePathsManager`. This iteration implements automatic detection of missing or expired paths and triggers path collection through the resource management system, similar to how single-equivalent payments work in `CoordinatorPaymentTransaction`.

### Current project state
- Exchange Payment with Commissions (PRD 06) implements multi-equivalent payment execution
- Exchange Flow Calculation (PRD 04) provides topology collection and max flow calculation using OR-Tools
- Payment Estimation (PRD 05) enables bidirectional payment estimation using cached paths
- `CoordinatorPaymentTransaction` demonstrates the pattern of checking cached paths and requesting topology collection when needed

### This iteration's focus
- Integrate automatic path availability checking in `CoordinatorExchangePaymentTransaction`
- Implement resource-based path collection triggering for exchange payments
- Create `FindPathsByMaxFlowExchangeTransaction` for exchange path topology collection
- Add `RequestExchangePathsResourceSignal` to `ResourcesManager`
- Implement `ExchangePathsResource` for resource-based communication
- Move exchange amount calculation and validation from initialization to path processing stage

### Connection to overall vision
This completes the automation of exchange payment path management, enabling seamless multi-equivalent payments without manual topology collection steps. The system automatically detects when paths are needed and initiates collection, matching the user experience of single-equivalent payments.

## Iteration Context
### Previous Iterations Summary
- **PRD 03**: Exchange Rates Manager provides exchange rate storage with TTL management
- **PRD 04**: Exchange Flow Calculation implements topology collection and OR-Tools-based max flow calculation for exchange payments
- **PRD 05**: Payment Estimation provides bidirectional estimation using cached optimal paths
- **PRD 06**: Exchange Payment with Commissions implements multi-equivalent payment execution with proper commission handling
- **Completed Features**: Exchange rate management, topology collection infrastructure, OR-Tools integration, optimal path calculation with commissions, exchange payment execution

### Lessons Learned
- Resource-based communication pattern (from `CoordinatorPaymentTransaction`) enables clean separation between path collection and payment execution
- TTL-based path caching reduces redundant topology collection while maintaining data freshness
- Signal-based architecture in `ResourcesManager` allows flexible transaction triggering

### Current State Analysis
- **What's working well**: Exchange payment execution (PRD 06) successfully processes payments when paths are available
- **Pain points identified**: Manual topology collection required before exchange payments; no automatic path availability checking
- **Performance metrics**: Exchange path collection and calculation from PRD 04 complete within acceptable timeframes

## Problem Statement
### Background
The current exchange payment implementation (`CoordinatorExchangePaymentTransaction` from PRD 06) requires paths to be pre-cached in `ExchangePathsManager`. Users must manually initiate topology collection via `InitiateMaxFlowExchangeCalculationTransaction` before attempting payment. This creates a poor user experience and differs from single-equivalent payments, which automatically trigger path collection when needed.

`CoordinatorPaymentTransaction` demonstrates the correct pattern:
1. Check if paths exist in cache (`PathsManager`)
2. If missing or expired, request paths via `ResourcesManager`
3. Wait for `PathsResource` response
4. Proceed with path processing

This pattern needs to be adapted for exchange payments with multi-equivalent awareness.

### Problem Description
**Who is affected**: Users attempting exchange payments without pre-cached paths

**When and where**: During exchange payment initialization in `CoordinatorExchangePaymentTransaction::runPaymentInitializationStage()`

**Current limitations**:
- No automatic detection of missing or expired exchange paths
- No integration with resource management system for exchange path requests
- Users must manually check path availability and trigger collection
- Exchange amount calculation (`mExchangeAmount`) happens before path availability verification
- Inconsistent behavior between single-equivalent and exchange payments

### Impact of not solving this problem
- Poor user experience requiring manual multi-step workflows
- Higher failure rate for exchange payments due to missing paths
- Confusion about why exchange payments fail when single-equivalent payments succeed automatically
- Underutilization of exchange payment capabilities due to complexity

### Success Metrics
**Primary KPIs**:
- Automatic path collection trigger rate (target: 100% when paths missing/expired)
- Exchange payment success rate improvement (target: match single-equivalent payment success rate)
- Reduced user intervention for path management (target: zero manual topology collection commands)

**Target Values**:
- 100% automatic path availability checking before payment attempts
- Zero exchange payment failures due to uncached paths
- Path collection latency < 5 seconds for typical network topologies
- Cache hit rate > 80% after initial path collection (due to 150s TTL)

## Goals
The primary goals of this iteration are to automate exchange payment path management and improve user experience.

*   **Goal 1: Enable automatic exchange path availability detection.**
    *   **Description:** Automatically detect when exchange paths are missing or expired for requested equivalents, similar to single-equivalent payment behavior.
    *   **Success Metric:** Exchange payments automatically trigger path collection 100% of the time when paths are unavailable.

*   **Goal 2: Implement resource-based exchange path collection.**
    *   **Description:** Integrate exchange payment coordinator with resource management system to request and receive paths asynchronously.
    *   **Success Metric:** `CoordinatorExchangePaymentTransaction` successfully receives and processes `ExchangePathsResource` from topology collection.

*   **Goal 3: Create dedicated exchange path collection transaction.**
    *   **Description:** Implement `FindPathsByMaxFlowExchangeTransaction` to handle topology collection specifically for exchange payment paths.
    *   **Success Metric:** Transaction successfully collects topology and builds paths using OR-Tools, caching results in `ExchangePathsManager`.

*   **Goal 4: Ensure correct timing of exchange amount calculation.**
    *   **Description:** Move exchange amount calculation and validation from initialization stage to path processing stage, after paths are guaranteed to be available.
    *   **Success Metric:** Exchange amount calculation always has access to valid cached paths; no calculation failures due to missing paths.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **Exchange Path Availability Checking**: Automatic detection of missing or expired paths in `CoordinatorExchangePaymentTransaction`
2. **FindPathsByMaxFlowExchangeTransaction**: New transaction for exchange-specific topology collection
3. **RequestExchangePathsResourceSignal**: New signal in `ResourcesManager` for exchange path requests
4. **ExchangePathsResource**: New resource type for communicating path collection completion
5. **Exchange Amount Calculation Relocation**: Move `mExchangeAmount` calculation from initialization to path processing stage

#### Technical Infrastructure
- Signal connection in `Core::connectResourcesManagerSignals()` linking new signal to transaction launcher
- Resource handling in `CoordinatorExchangePaymentTransaction` for `ExchangePathsResource`
- Path expiry checking logic in coordinator transaction
- Integration with existing `ExchangePathsManager` caching infrastructure

#### Integration Points
- Extension of `ResourcesManager` signal architecture
- Integration with `Core` transaction launching mechanism
- Usage of `ExchangePathsManager` for path caching and retrieval
- Connection with existing `BaseCollectTopologyForExchangeTransaction` infrastructure

### Explicitly Out of Scope
- Modification of topology collection protocol or messages (already implemented in PRD 04)
- Changes to OR-Tools integration or max flow calculation algorithms
- Path invalidation strategies or cache management policies
- Performance optimization for large-scale topology collection
- UI/API changes for user-facing commands

### Dependencies from Previous Iterations
- **PRD 04**: Exchange topology collection infrastructure and OR-Tools integration
- **PRD 05**: `ExchangePathsManager` path caching functionality
- **PRD 06**: `CoordinatorExchangePaymentTransaction` basic implementation
- **Existing infrastructure**: `ResourcesManager`, `PathsResource` pattern, `Core` signal management

### Future Roadmap Impact
This iteration establishes foundation for:
- **Automatic path refresh strategies**: Time-based or event-driven path invalidation
- **Predictive path collection**: Pre-emptive topology collection based on usage patterns
- **Path quality metrics**: Monitoring and optimization of path collection effectiveness
- **Multi-transaction path sharing**: Reusing collected paths across concurrent exchange payments

## User Stories & Requirements

### User Personas
#### Primary User: Exchange Payment Initiator
- **Role**: Node operator initiating cross-equivalent payments
- **Goals**: Successfully execute exchange payments without manual topology management
- **Pain Points**: Must remember to collect topology before payments; unclear when paths expire
- **Technical Proficiency**: Intermediate

#### Secondary User: Node Operator
- **Role**: Network participant running vTCP node
- **Goals**: Provide reliable payment services with minimal manual intervention
- **Pain Points**: Complex multi-step workflows; inconsistent behavior between payment types
- **Technical Proficiency**: Advanced

### Functional Requirements
#### New Features for This Iteration

1. **Exchange Path Availability Checking in runPaymentInitializationStage()**
   - **Description**: Verify cached path availability and freshness for all required exchange equivalents
   - **User Story**: As a payment coordinator, I need to know if paths exist before attempting payment
   - **Rationale**: Prevents payment failures due to missing topology data
   - **Builds Upon**: Existing `ExchangePathsManager` cache structure
   - **Acceptance Criteria**:
     - For each `mExchangeEquivalent`, check if paths exist for `PathCacheKey{mContractorID, exchangeEquiv, mEquivalent}`
     - Call `mExchangePathsManager->retrievePaths(key, kExchangePathsCacheTTLSeconds)` with custom 150s TTL
     - `retrievePaths()` returns `nullopt` if paths missing OR expired (age >= 150s)
     - Collect list of equivalents where `retrievePaths()` returned `nullopt`
     - If all equivalents have valid paths, proceed to exchange amount calculation
     - If any equivalents missing/expired, trigger path collection request
     - Implementation matches pattern from `CoordinatorPaymentTransaction::runPaymentInitializationStage()`
   - **Priority**: High
   - **Dependencies**: `ExchangePathsManager::retrievePaths()` with customTTL parameter

2. **Resource Request for Missing Exchange Paths**
   - **Description**: Trigger path collection via `ResourcesManager` when paths unavailable
   - **User Story**: As a payment coordinator, I need to request topology collection for missing equivalents
   - **Rationale**: Automates path collection without user intervention
   - **Builds Upon**: `ResourcesManager` signal architecture
   - **Acceptance Criteria**:
     - Call `mResourcesManager->requestExchangePaths()` with transaction UUID, contractor address, missing equivalents, and receiver equivalent
     - Wait for `ExchangePathsResource` using appropriate timeout (similar to `PathsResource` pattern)
     - Handle resource arrival in dedicated handler method
     - Transition to `runPathsResourceProcessingStage()` upon resource receipt
     - Do NOT proceed to exchange amount calculation until paths are available
   - **Priority**: High
   - **Dependencies**: `RequestExchangePathsResourceSignal`, `ExchangePathsResource`

3. **FindPathsByMaxFlowExchangeTransaction**
   - **Description**: Transaction for collecting topology and building exchange paths
   - **User Story**: As a resource manager, I need a transaction to build exchange paths on demand
   - **Rationale**: Separates path collection logic from payment execution
   - **Builds Upon**: `BaseCollectTopologyForExchangeTransaction`, `InitiateMaxFlowExchangeCalculationTransaction` pattern
   - **Acceptance Criteria**:
     - Inherits from `BaseCollectTopologyForExchangeTransaction`
     - Constructor parameters: `BaseAddress::Shared contractorAddress`, `TransactionUUID requestedTransactionUUID`, `vector<SerializedEquivalent> exchangeEquivalents`, `SerializedEquivalent receiverEquivalent`, and all components from `InitiateMaxFlowExchangeCalculationTransaction` (except command)
     - Implements `sendRequestForCollectingTopology()`: initiates topology collection for specified equivalents
     - Implements `processCollectingTopology()`: builds paths using OR-Tools (reuses logic from `InitiateMaxFlowExchangeCalculationTransaction::applyCustomLogic()`)
     - Caches built paths in `ExchangePathsManager` for each `PathCacheKey{contractorID, exchangeEquiv, receiverEquiv}`
     - Returns `ExchangePathsResource` via `mResourcesManager->putResource()`
     - Location: `src/core/transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.h/.cpp`
     - Does NOT require paths to be built successfully - returns resource even if no paths found (coordinator handles this)
   - **Priority**: High
   - **Dependencies**: `ExchangePathsManager`, OR-Tools integration, topology collection infrastructure

4. **RequestExchangePathsResourceSignal in ResourcesManager**
   - **Description**: New signal for triggering exchange path collection
   - **User Story**: As a coordinator transaction, I need a signal to request exchange paths
   - **Rationale**: Extends resource management pattern to exchange payments
   - **Builds Upon**: Existing `RequestPathsResourcesSignal` pattern
   - **Acceptance Criteria**:
     - Signal signature: `void(const TransactionUUID&, BaseAddress::Shared, const vector<SerializedEquivalent>&, const SerializedEquivalent)`
     - Parameters: transaction UUID, contractor address, exchange equivalents list (sender equivalents), receiver equivalent
     - Method `requestExchangePaths()` in `ResourcesManager` to trigger signal
     - Connected in `Core::connectResourcesManagerSignals()` to launch `FindPathsByMaxFlowExchangeTransaction`
     - Location: `src/core/resources/manager/ResourcesManager.h/.cpp`
   - **Priority**: High
   - **Dependencies**: None (new signal definition)

5. **ExchangePathsResource**
   - **Description**: Resource object communicating path collection completion
   - **User Story**: As a path collection transaction, I need to notify coordinator when paths are ready
   - **Rationale**: Enables asynchronous communication between transactions
   - **Builds Upon**: `BaseResource` pattern, similar to `PathsResource`
   - **Acceptance Criteria**:
     - Inherits from `BaseResource`
     - Contains transaction UUID for routing to correct coordinator
     - No additional fields needed (paths already in `ExchangePathsManager` cache)
     - Resource arrival triggers transition to `runPathsResourceProcessingStage()` in coordinator
     - Location: `src/core/resources/resources/ExchangePathsResource.h`
   - **Priority**: High
   - **Dependencies**: `BaseResource`

6. **Exchange Amount Calculation Relocation**
   - **Description**: Move `mExchangeAmount` calculation and validation to `runPathsResourceProcessingStage()`
   - **User Story**: As a payment coordinator, I need to calculate exchange amount only after paths are available
   - **Rationale**: Ensures calculation always has access to valid cached paths
   - **Builds Upon**: Existing calculation logic from PRD 06
   - **Acceptance Criteria**:
     - Remove `mExchangeAmount` calculation from `runPaymentInitializationStage()`
     - Remove `kTotalOutgoingPossibilities` check from initialization stage (lines 383-400 in current implementation)
     - Add `mExchangeAmount` calculation at start of `runPathsResourceProcessingStage()` (before "Step 1: Initialize total flow counter")
     - Add `kTotalOutgoingPossibilities` validation in `runPathsResourceProcessingStage()` after calculation
     - Calculation uses same algorithm as described in PRD 06
     - Return appropriate error if calculation fails or validation fails
   - **Priority**: High
   - **Dependencies**: Path availability checking

#### Enhancements to Existing Features

1. **ExchangePathsManager::retrievePaths() Extension**
   - **Current State**: Returns cached paths if found and not expired (uses default 600s TTL)
   - **Proposed Changes**:
     - Add `optional<uint32_t> customTTL` parameter (default = nullopt → 600s)
     - Use customTTL when provided for expiry checking
     - TTL semantics: `age >= TTL` means expired (currently `expiresAt <= now`, equivalent)
     - Backward compatible: existing calls without customTTL use default behavior
   - **Impact Assessment**: Pure extension, no breaking changes to existing functionality
   - **Migration Strategy**: Parameter has default value, no changes needed in existing code

2. **CoordinatorExchangePaymentTransaction::runPaymentInitializationStage()**
   - **Current State**: Assumes paths are already cached; calculates `mExchangeAmount` immediately
   - **Proposed Changes**:
     - Add path availability checking using `retrievePaths(key, kExchangePathsCacheTTLSeconds)`
     - Remove `mExchangeAmount` calculation (moved to path processing stage)
     - Remove `kTotalOutgoingPossibilities` check (moved to path processing stage)
     - Add resource request logic for missing equivalents
     - Add wait state for `ExchangePathsResource`
   - **Impact Assessment**: Changes payment initialization flow but maintains overall transaction structure
   - **Migration Strategy**: Direct modification of existing method

3. **CoordinatorExchangePaymentTransaction::runPathsResourceProcessingStage()**
   - **Current State**: Retrieves paths from `ExchangePathsManager` and processes them
   - **Proposed Changes**:
     - Add `mExchangeAmount` calculation at method start (before path processing)
     - Add `kTotalOutgoingPossibilities` validation after calculation
     - Existing path retrieval and processing logic unchanged
   - **Impact Assessment**: Adds validation before processing, no impact on processing logic
   - **Migration Strategy**: Insert calculation and validation at method start

4. **Core::connectResourcesManagerSignals()**
   - **Current State**: Connects `RequestPathsResourcesSignal` to single-equivalent path collection
   - **Proposed Changes**: Add connection for `RequestExchangePathsResourceSignal` to launch `FindPathsByMaxFlowExchangeTransaction`
   - **Impact Assessment**: Pure addition, no impact on existing signal handling
   - **Migration Strategy**: Add new signal connection alongside existing ones

### Non-Functional Requirements
#### Performance
- Path availability checking overhead < 10ms per exchange equivalent
- Path collection initiation latency < 50ms
- Resource signal processing latency < 20ms
- Total overhead for path checking and request < 100ms

#### Security
- No security implications (reuses existing secure topology collection)
- Transaction UUID prevents resource hijacking
- Path cache access remains thread-safe

#### Scalability
- Support checking up to 5 exchange equivalents (as per PRD 06 limit)
- Handle concurrent path collection requests from multiple payment transactions
- Path cache scales with existing `ExchangePathsManager` capacity

#### Reliability
- Path collection failures do not crash coordinator transaction
- Resource timeout handling prevents indefinite waiting
- Empty path results handled gracefully (payment fails with appropriate error)
- Transaction state recovery after path collection timeout

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Exchange payments assume pre-cached paths; single-equivalent payments use resource-based path collection
- **Proposed Changes**: Extend resource-based pattern to exchange payments with multi-equivalent awareness
- **Backwards Compatibility**: Existing exchange payment behavior preserved (if paths already cached, no collection triggered)
- **Migration Requirements**: None (new behavior is enhancement, not breaking change)

### Technology Stack Updates
#### New Technologies/Libraries
- No new external libraries required
- Reuses existing infrastructure: `ResourcesManager`, `BaseResource`, `ExchangePathsManager`, OR-Tools integration

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- `FindPathsByMaxFlowExchangeTransaction` integrates with `ExchangePathsManager` for path caching
- `Core` integrates new signal to transaction launcher
- `CoordinatorExchangePaymentTransaction` integrates with resource request system

#### Modified Integrations
- `ResourcesManager` extended with exchange-specific signal
- `Core::connectResourcesManagerSignals()` adds new signal connection

### Data Requirements
#### Data Models

##### ExchangePathsResource
**Purpose**: Signal completion of exchange path collection to coordinator transaction

**Fields**:
```cpp
class ExchangePathsResource : public BaseResource {
public:
    ExchangePathsResource(const TransactionUUID &transactionUUID);

    const TransactionUUID& transactionUUID() const;

private:
    TransactionUUID mTransactionUUID;
};
```

**Usage**: Returned by `FindPathsByMaxFlowExchangeTransaction`, received by `CoordinatorExchangePaymentTransaction`

##### RequestExchangePathsResourceSignal
**Purpose**: Signal for requesting exchange path collection

**Signature**:
```cpp
typedef signals::signal<void(
    const TransactionUUID&,                    // Requesting transaction UUID
    BaseAddress::Shared,                       // Contractor address
    const vector<SerializedEquivalent>&,       // Exchange equivalents (sender)
    const SerializedEquivalent)>               // Receiver equivalent
RequestExchangePathsResourceSignal;
```

**Usage**: Emitted by `CoordinatorExchangePaymentTransaction`, handled by `Core` to launch `FindPathsByMaxFlowExchangeTransaction`

##### ExchangePathsManager::retrievePaths() Extension
**Purpose**: Retrieve cached paths with custom TTL support for different use cases

**Enhanced Signature**:
```cpp
optional<vector<OptimalPathResult>> retrievePaths(
    const PathCacheKey &key,
    optional<uint32_t> customTTL = nullopt);  // nullopt = use default kPathResultsTTLSeconds (600s)
```

**Enhanced Algorithm**:
```cpp
optional<vector<OptimalPathResult>> ExchangePathsManager::retrievePaths(
    const PathCacheKey &key,
    optional<uint32_t> customTTL)
{
    lock_guard<mutex> lock(mCacheMutex);

    auto it = mCachedPaths.find(key);
    if (it == mCachedPaths.end()) {
        return nullopt;  // Paths not found
    }

    // Use custom TTL if provided, otherwise use default
    uint32_t ttlToUse = customTTL.value_or(kPathResultsTTLSeconds);

    auto now = utc_now();
    auto age = now - it->second.computedAt;

    // TTL Semantics: age >= TTL means expired
    if (age.total_seconds() >= ttlToUse) {
        debug() << "Cached paths expired (age=" << age.total_seconds()
                << "s, TTL=" << ttlToUse << "s), removing";
        mCachedPaths.erase(it);
        return nullopt;  // Paths expired
    }

    return it->second.paths;  // Paths valid
}
```

**Key Changes from Current Implementation**:
1. Added `optional<uint32_t> customTTL` parameter for flexible TTL per use case
2. Changed expiry semantics from `expiresAt <= now` to `age >= TTL` (equivalent but clearer)
3. Backward compatible: without customTTL, uses default 600s for estimation (PRD 05)
4. With customTTL=150s: used by payment coordinator for fresher paths (PRD 06/07)

**Location**: `src/core/paths/ExchangePathsManager.h/.cpp`

**Constants**:
```cpp
class CoordinatorExchangePaymentTransaction {
    static const uint32_t kExchangePathsCacheTTLSeconds = 150;
};
```

#### Data Storage
- No new persistent storage requirements
- In-memory path cache in `ExchangePathsManager` (existing)
- Transaction state includes path collection waiting state

#### Data Migration
- No migration required (new functionality)

### Algorithm Specifications

#### Path Availability Checking in runPaymentInitializationStage()
**Purpose**: Determine which exchange equivalents need topology collection

**Algorithm**:
```cpp
TransactionResult::SharedConst runPaymentInitializationStage() {
    // ... [existing self-contractor check and initialization] ...

    // Step 1: Check path availability for all exchange equivalents
    vector<SerializedEquivalent> missingEquivalents;

    for (const auto& exchangeEquiv : mExchangeEquivalents) {
        PathCacheKey key{mContractorID, exchangeEquiv, mEquivalent};

        // Check if paths exist and are fresh using custom TTL (150s)
        // retrievePaths() returns nullopt if paths missing OR expired
        auto cachedPaths = mExchangePathsManager->retrievePaths(
            key,
            kExchangePathsCacheTTLSeconds);

        if (!cachedPaths) {
            // Paths not found or expired - need collection
            missingEquivalents.push_back(exchangeEquiv);
        }
    }

    // Step 2: If any equivalents missing, request path collection
    if (!missingEquivalents.empty()) {
        info() << "Exchange paths missing or expired for " << missingEquivalents.size()
               << " equivalents, requesting collection";

        mResourcesManager->requestExchangePaths(
            currentTransactionUUID(),
            mContractorAddresses[0], // Main contractor address
            missingEquivalents,
            mEquivalent);

        // Wait for ExchangePathsResource
        return resultWaitForResourceTypes(
            {BaseResource::ExchangePaths},
            maxNetworkDelay(4));
    }

    // Step 3: All paths available, proceed to path processing
    // Note: mExchangeAmount calculation moved to runPathsResourceProcessingStage()
    mStep = Coordinator_PathsResourceProcessing;
    return runPathsResourceProcessingStage();
}
```

**Key Points**:
- Checks each `mExchangeEquivalent` for path availability and freshness
- Collects only missing or expired equivalents for request
- Waits for resource if collection needed
- Proceeds directly to path processing if all paths valid

#### Exchange Amount Calculation in runPathsResourceProcessingStage()
**Purpose**: Calculate required sender payment amount after paths are guaranteed available

**Algorithm**:
```cpp
TransactionResult::SharedConst runPathsResourceProcessingStage() {
    // Step 0: Calculate mExchangeAmount (moved from runPaymentInitializationStage)
    try {
        TrustLineAmount remainingReceive = mAmount;  // Receiver amount
        TrustLineAmount totalPayment = TrustLineAmount(0);

        // Calculate required payment amount using cached paths
        for (const auto& exchangeEquiv : mExchangeEquivalents) {
            if (remainingReceive == TrustLineAmount(0)) {
                break;
            }

            PathCacheKey key{mContractorID, exchangeEquiv, mEquivalent};
            auto cachedPaths = mExchangePathsManager->retrievePaths(key);

            if (!cachedPaths) {
                // This should not happen as we verified availability in initialization
                warning() << "Paths disappeared between initialization and processing";
                return resultNoPathsError();
            }

            for (const auto &pathResult : *cachedPaths) {
                if (remainingReceive == TrustLineAmount(0)) {
                    break;
                }

                TrustLineAmount deliveredAmount = min(remainingReceive, pathResult.received_amount);

                // Calculate required payment for this delivered amount
                double ratio = deliveredAmount.convert_to<double>() /
                               pathResult.received_amount.convert_to<double>();
                TrustLineAmount requiredPayment(
                    static_cast<uint64_t>(pathResult.optimal_flow.convert_to<double>() * ratio));

                totalPayment = totalPayment + requiredPayment;
                remainingReceive = remainingReceive - deliveredAmount;
            }
        }

        if (remainingReceive > TrustLineAmount(0)) {
            warning() << "Insufficient paths to deliver " << mAmount;
            return resultInsufficientFundsError();
        }

        mExchangeAmount = totalPayment;
        info() << "Calculated exchange amount: " << mExchangeAmount
               << " (sender eq=" << mExchangeEquivalent << ") "
               << "to deliver " << mAmount
               << " (receiver eq=" << mEquivalent << ")";

    } catch (const exception &e) {
        error() << "Error calculating exchange amount: " << e.what();
        return resultProtocolError();
    }

    // Step 0.5: Check kTotalOutgoingPossibilities (moved from runPaymentInitializationStage)
    // This is the check from lines 383-400 of current implementation
    TrustLineAmount totalOutgoingAmount = TrustLineAmount(0);
    for (const auto& exchangeEquiv : mExchangeEquivalents) {
        auto manager = mEquivalentsSubsystemsRouter->trustLinesManager(exchangeEquiv);
        totalOutgoingAmount = totalOutgoingAmount + manager->totalOutgoingAmount();
    }

    if (totalOutgoingAmount < mExchangeAmount) {
        warning() << "Insufficient outgoing capacity: have " << totalOutgoingAmount
                  << ", need " << mExchangeAmount;
        return resultInsufficientFundsError();
    }

    // Step 1: Initialize total flow counter (existing code continues...)
    TrustLineAmount totalAddedFlow = TrustLineAmount(0);

    // ... [rest of existing path processing logic] ...
}
```

**Key Points**:
- Calculation now happens AFTER path availability is guaranteed
- Uses same algorithm as PRD 06 description
- Includes outgoing capacity validation previously in initialization stage
- Proceeds to existing path processing after validation

#### FindPathsByMaxFlowExchangeTransaction::processCollectingTopology()
**Purpose**: Build exchange paths using OR-Tools and cache results

**Algorithm**:
```cpp
TransactionResult::SharedConst processCollectingTopology() {
    info() << "Building exchange paths for contractor " << mContractorID;

    // Reuse logic from InitiateMaxFlowExchangeCalculationTransaction::applyCustomLogic()
    // This includes:
    // 1. Path enumeration
    // 2. OR-Tools LP problem construction
    // 3. Optimization
    // 4. Solution extraction
    // 5. Path caching

    try {
        auto result = mExchangePathsManager->calculateMaxFlow(
            mContractorID,
            mReceiverEquivalent,
            mExchangeEquivalents,
            TopologyTrustLinesManager::kCurrentNodeID,
            mHopsCount);

        // Paths are automatically cached by ExchangePathsManager::calculateMaxFlow()
        // for each PathCacheKey{mContractorID, exchangeEquiv, mReceiverEquivalent}

        info() << "Exchange path building complete, found " << result.optimalPaths.size()
               << " optimal paths with max flow " << result.maxFlow;

    } catch (const exception &e) {
        warning() << "Error building exchange paths: " << e.what();
        // Continue - return resource even if no paths found
    }

    // Return ExchangePathsResource to notify coordinator
    auto resource = make_shared<ExchangePathsResource>(mRequestedTransactionUUID);
    mResourcesManager->putResource(resource);

    return resultDone();
}
```

**Key Points**:
- Reuses OR-Tools integration from `InitiateMaxFlowExchangeCalculationTransaction`
- Paths automatically cached by `ExchangePathsManager::calculateMaxFlow()`
- Returns resource even if path building fails (coordinator handles empty cache)
- Clean separation between path building and payment execution

### Sequence Diagram

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│ User/Client │         │ Coordinator  │         │  Resources  │         │     Core     │         │ FindPaths   │
│             │         │ ExchangePmt  │         │   Manager   │         │              │         │ Transaction │
└──────┬──────┘         └──────┬───────┘         └──────┬──────┘         └──────┬───────┘         └──────┬──────┘
       │                       │                        │                       │                        │
       │ CreditUsageExchange   │                        │                       │                        │
       │ Command               │                        │                       │                        │
       │──────────────────────>│                        │                       │                        │
       │                       │                        │                       │                        │
       │                       │ runPaymentInit         │                       │                        │
       │                       │ Stage()                │                       │                        │
       │                       │───┐                    │                       │                        │
       │                       │   │                    │                       │                        │
       │                       │   │ Check path cache   │                       │                        │
       │                       │   │ for each           │                       │                        │
       │                       │   │ mExchangeEquivalent│                       │                        │
       │                       │<──┘                    │                       │                        │
       │                       │                        │                       │                        │
       │                       │ Are paths missing/     │                       │                        │
       │                       │ expired?               │                       │                        │
       │                       │───┐                    │                       │                        │
       │                       │   │ YES: collect       │                       │                        │
       │                       │   │ missing equivalents│                       │                        │
       │                       │<──┘                    │                       │                        │
       │                       │                        │                       │                        │
       │                       │ requestExchangePaths() │                       │                        │
       │                       │ (txUUID, contractor,   │                       │                        │
       │                       │  missingEquivs,        │                       │                        │
       │                       │  receiverEquiv)        │                       │                        │
       │                       │───────────────────────>│                       │                        │
       │                       │                        │                       │                        │
       │                       │                        │ RequestExchangePaths  │                        │
       │                       │                        │ ResourceSignal        │                        │
       │                       │                        │──────────────────────>│                        │
       │                       │                        │                       │                        │
       │                       │                        │                       │ Launch                 │
       │                       │                        │                       │ FindPathsByMaxFlow     │
       │                       │                        │                       │ ExchangeTransaction    │
       │                       │                        │                       │───────────────────────>│
       │                       │                        │                       │                        │
       │                       │ Wait for               │                       │                        │
       │                       │ ExchangePathsResource  │                       │                        │
       │                       │───┐                    │                       │                        │
       │                       │   │                    │                       │                        │
       │                       │<──┘                    │                       │                        │
       │                       │                        │                       │                        │
       │                       │                        │                       │ sendRequestForCollecting│
       │                       │                        │                       │ Topology()             │
       │                       │                        │                       │───┐                    │
       │                       │                        │                       │   │                    │
       │                       │                        │                       │<──┘                    │
       │                       │                        │                       │                        │
       │                       │                        │                       │ [Topology collection]  │
       │                       │                        │                       │ (PRD 04 protocol)      │
       │                       │                        │                       │───┐                    │
       │                       │                        │                       │   │                    │
       │                       │                        │                       │<──┘                    │
       │                       │                        │                       │                        │
       │                       │                        │                       │ processCollectingTopology│
       │                       │                        │                       │ ()                     │
       │                       │                        │                       │───┐                    │
       │                       │                        │                       │   │ Build paths via    │
       │                       │                        │                       │   │ OR-Tools           │
       │                       │                        │                       │   │ Cache in           │
       │                       │                        │                       │   │ ExchangePathsMgr   │
       │                       │                        │                       │<──┘                    │
       │                       │                        │                       │                        │
       │                       │                        │ putResource(          │                        │
       │                       │                        │ ExchangePathsResource)│                        │
       │                       │                        │<──────────────────────────────────────────────│
       │                       │                        │                       │                        │
       │                       │ AttachResourceSignal   │                       │                        │
       │                       │ (ExchangePathsResource)│                       │                        │
       │                       │<───────────────────────│                       │                        │
       │                       │                        │                       │                        │
       │                       │ runPathsResource       │                       │                        │
       │                       │ ProcessingStage()      │                       │                        │
       │                       │───┐                    │                       │                        │
       │                       │   │                    │                       │                        │
       │                       │   │ Calculate          │                       │                        │
       │                       │   │ mExchangeAmount    │                       │                        │
       │                       │   │                    │                       │                        │
       │                       │   │ Validate outgoing  │                       │                        │
       │                       │   │ possibilities      │                       │                        │
       │                       │   │                    │                       │                        │
       │                       │   │ Retrieve paths from│                       │                        │
       │                       │   │ ExchangePathsMgr   │                       │                        │
       │                       │   │                    │                       │                        │
       │                       │   │ Process paths      │                       │                        │
       │                       │<──┘                    │                       │                        │
       │                       │                        │                       │                        │
       │                       │ [Continue payment      │                       │                        │
       │                       │  execution...]         │                       │                        │
       │                       │                        │                       │                        │
       │<──────────────────────│                        │                       │                        │
       │ Payment Result        │                        │                       │                        │
       │                       │                        │                       │                        │
```

**Legend**:
- Vertical lines: Timeline for each component
- Horizontal arrows: Method calls or messages
- Boxes with `───┐`: Internal processing
- `[...]`: Existing complex processes (not detailed here)

**Flow Description**:
1. User initiates exchange payment via `CreditUsageExchangeCommand`
2. Coordinator checks path cache for all exchange equivalents
3. If paths missing/expired, requests collection via `ResourcesManager`
4. Signal triggers `Core` to launch `FindPathsByMaxFlowExchangeTransaction`
5. Path collection transaction builds topology and paths (PRD 04 protocol)
6. Paths cached in `ExchangePathsManager`, resource returned
7. Coordinator receives resource, calculates exchange amount, processes paths
8. Payment continues with normal execution flow

### Error Handling Specifications

#### Error Conditions
1. **No paths found after collection**: Transaction continues, returns `ExchangePathsResource`; coordinator detects empty cache and returns `resultNoPathsError()`
2. **Path collection timeout**: Coordinator transaction timeout handler triggers; returns appropriate timeout error to user
3. **Paths expired between check and usage**: Unlikely due to short time window; coordinator logs warning and returns `resultNoPathsError()`
4. **Exchange amount calculation fails**: Returns `resultProtocolError()` or `resultInsufficientFundsError()` based on specific failure
5. **Outgoing capacity validation fails**: Returns `resultInsufficientFundsError()` with descriptive message
6. **Resource signal connection missing**: Core initialization failure; node fails to start (critical error)

## Implementation Plan
### This Iteration Timeline
- **Duration**: 3-4 weeks implementation + 1-2 weeks testing
- **Sprint Breakdown**:
  - Sprint 1 (Week 1): Resource infrastructure (`RequestExchangePathsResourceSignal`, `ExchangePathsResource`, signal connections)
  - Sprint 2 (Week 2): `FindPathsByMaxFlowExchangeTransaction` implementation
  - Sprint 3 (Week 3): `CoordinatorExchangePaymentTransaction` modifications (path checking, resource handling, calculation relocation)
  - Sprint 4 (Week 4): Integration testing and edge case handling

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Resource Infrastructure Complete | Week 1 | Signal and resource classes implemented and connected | None | Low |
| FindPathsByMaxFlowExchangeTransaction Complete | Week 2 | Path collection transaction functional | Resource infrastructure | Medium |
| Coordinator Modifications Complete | Week 3 | Path checking and calculation relocation done | All previous | Medium |
| Integration Testing Complete | Week 4 | End-to-end flow validated | All previous | Low |

### Dependencies on Other Teams/Projects
- No external team dependencies identified

### Integration Points with Previous Work
- Builds directly upon `ExchangePathsManager` caching (PRD 05)
- Extends `CoordinatorExchangePaymentTransaction` (PRD 06)
- Reuses topology collection infrastructure (PRD 04)
- Follows `ResourcesManager` pattern from single-equivalent payments

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++ and payment systems experience
- **Developers**: 1 developer for implementation support
- **QA Engineers**: 1 engineer for integration testing

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Race condition between path expiry check and usage | Medium | Low | TTL window (150s) is large relative to transaction duration; log warning if occurs |
| Resource timeout causing payment failures | High | Low | Use appropriate timeout values from existing payment patterns; extensive timeout testing |
| Path collection transaction failures | Medium | Medium | Return resource even on failure; coordinator handles empty cache gracefully |
| Signal connection missing causes silent failures | High | Low | Validate signal connections during Core initialization; fail fast if missing |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| User confusion during automatic path collection | Low | Low | Clear logging of path collection activity; transparent behavior |

## Testing Strategy
### Testing Approach (Unit-Only)
- All testing is unit-only (no integration/E2E tests)
- Tests are built and executed exclusively in `build-tests`
- Use real objects following ExchangePathsManagerTest pattern
- Use TestEnvironment helper classes for consistent test setup
- Test files located in `tests/unit/` subdirectories

### Testing Best Practices
- **Real Objects Over Mocks**: Use real instances of managers and components
- **TestEnvironment Helpers**: Create helper classes for test initialization
- **Exception Testing**: Test that methods throw correct exceptions using EXPECT_THROW
- **Edge Case Coverage**: Test empty vectors, boundary values, null/invalid inputs
- **Parameterized Tests**: Use for testing different equivalent combinations

#### Unit Tests: New Components

**1. ExchangePathsResource Tests**:
- Constructor correctly initializes with transaction UUID
- transactionUUID() getter returns correct value
- Inherits from BaseResource correctly
- Resource type correctly identified

**2. RequestExchangePathsResourceSignal Tests**:
- Signal correctly defined with proper signature
- requestExchangePaths() method triggers signal with correct parameters
- Signal connection in Core::connectResourcesManagerSignals() works correctly
- Multiple signal connections can coexist

**3. FindPathsByMaxFlowExchangeTransaction Tests**:
- Constructor initializes with all required parameters
- sendRequestForCollectingTopology() initiates topology collection for specified equivalents
- processCollectingTopology() builds paths using OR-Tools integration
- Paths correctly cached in ExchangePathsManager for all PathCacheKey combinations
- Returns ExchangePathsResource via ResourcesManager
- Handles OR-Tools failures gracefully (returns resource with empty cache)
- Reuses InitiateMaxFlowExchangeCalculationTransaction logic correctly

**4. CoordinatorExchangePaymentTransaction Path Checking Tests**:
- **Test 1: All paths available and fresh**
  - Setup: Cache contains valid paths for all mExchangeEquivalents (age < 150s)
  - Call: `retrievePaths(key, 150)` returns valid paths
  - Expected: Proceeds directly to runPathsResourceProcessingStage() without resource request
- **Test 2: Some paths missing**
  - Setup: Cache missing paths for 2 of 3 exchange equivalents
  - Call: `retrievePaths(key, 150)` returns nullopt for missing
  - Expected: Requests collection for missing equivalents only, waits for resource
- **Test 3: Some paths expired**
  - Setup: Paths exist but age >= 150s (exactly 150s or older)
  - Call: `retrievePaths(key, 150)` returns nullopt for expired
  - Expected: Requests collection for expired equivalents, waits for resource
- **Test 4: Mixed missing and expired**
  - Setup: Some equivalents missing, others expired (age >= 150s), some valid (age < 150s)
  - Call: `retrievePaths(key, 150)` returns nullopt for missing/expired, paths for valid
  - Expected: Requests collection for missing and expired only
- **Test 5: All paths missing**
  - Setup: Empty cache
  - Call: `retrievePaths(key, 150)` returns nullopt for all
  - Expected: Requests collection for all mExchangeEquivalents
- **Test 6: Resource arrival triggers path processing**
  - Setup: Coordinator waiting for ExchangePathsResource
  - Expected: Resource arrival transitions to runPathsResourceProcessingStage()
- **Test 7: Custom TTL parameter used**
  - Setup: Paths exist with age = 160s
  - Call: `retrievePaths(key, 150)` returns nullopt (expired with 150s TTL)
  - Call: `retrievePaths(key, 600)` returns paths (valid with 600s TTL)
  - Expected: Custom TTL parameter correctly affects expiry decision

**5. Exchange Amount Calculation Relocation Tests**:
- **Test 1: Calculation in path processing stage**
  - Setup: Paths available in cache
  - Expected: mExchangeAmount calculated correctly in runPathsResourceProcessingStage()
- **Test 2: Calculation not in initialization stage**
  - Setup: Paths available
  - Expected: runPaymentInitializationStage() does NOT calculate mExchangeAmount
- **Test 3: Outgoing capacity validation in path processing**
  - Setup: Insufficient outgoing capacity
  - Expected: Validation fails in runPathsResourceProcessingStage(), returns resultInsufficientFundsError()
- **Test 4: Calculation with missing paths**
  - Setup: Paths disappeared between stages (edge case)
  - Expected: Returns resultNoPathsError() with warning logged

**6. Core Signal Connection Tests**:
- RequestExchangePathsResourceSignal connected in connectResourcesManagerSignals()
- Signal emission launches FindPathsByMaxFlowExchangeTransaction with correct parameters
- Transaction UUID correctly passed to launched transaction
- Contractor address, exchange equivalents, receiver equivalent all passed correctly

**7. Path Expiry Logic Tests**:
- Freshly cached paths (age < 150s) considered valid
- Paths exactly at TTL boundary (age == 150s) considered expired (age >= TTL semantics)
- Paths older than TTL (age > 150s) considered expired
- Missing cache entries treated as expired (retrievePaths returns nullopt)
- Custom TTL parameter correctly used instead of default 600s
- utc_now() time calculation correct

**8. ExchangePathsManager::retrievePaths() Extension Tests**:
- **Test 1: Default behavior preserved**
  - Call: `retrievePaths(key)` without customTTL
  - Expected: Uses default 600s TTL (backward compatible)
- **Test 2: Custom TTL used when provided**
  - Call: `retrievePaths(key, 150)`
  - Expected: Uses 150s TTL for expiry check
- **Test 3: TTL boundary semantics (age >= TTL)**
  - Setup: Path with age exactly 150s
  - Call: `retrievePaths(key, 150)`
  - Expected: Returns nullopt (expired, age >= 150)
- **Test 4: Just under TTL is valid**
  - Setup: Path with age 149s
  - Call: `retrievePaths(key, 150)`
  - Expected: Returns paths (valid, age < 150)
- **Test 5: Missing paths return nullopt**
  - Setup: Key not in cache
  - Call: `retrievePaths(key, 150)`
  - Expected: Returns nullopt
- **Test 6: Thread safety with customTTL**
  - Setup: Concurrent calls with different TTL values
  - Expected: Correct TTL applied for each call, no race conditions

**9. ResourcesManager Extension Tests**:
- requestExchangePaths() method exists and callable
- Signal emission with correct parameter types
- Signal can be connected and disconnected
- Multiple subscribers can connect to signal

#### Regression Testing (Unit)
- Scope: Ensure modifications don't break existing exchange payment execution
- Verify CoordinatorExchangePaymentTransaction behavior when paths already cached
- Validate single-equivalent payments remain unaffected (CoordinatorPaymentTransaction)
- Confirm ExchangePathsManager caching behavior unchanged

#### Execution in CI/Locally
- Build tests in `build-tests` and run the produced binaries
- All unit tests must pass before PRD completion

### Quality Gates
- All unit tests pass in `build-tests`
- Automatic path collection triggers correctly when paths missing/expired
- Exchange amount calculation always has access to valid paths
- Resource-based communication completes successfully end-to-end
- No regressions in existing exchange payment behavior (when paths cached)
- No regressions in single-equivalent payment behavior
- Path expiry logic correctly identifies stale cache entries
- Signal connections established correctly during Core initialization

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Feature addition (backward compatible enhancement)
- **Rollout Strategy**: Full deployment with automatic path collection; existing behavior preserved when paths pre-cached
- **Rollback Plan**: Revert to previous version if critical issues found; no data migration concerns

### Database Migrations
- No database migrations required
- No persistent data format changes

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: Node operator guidance on automatic path collection behavior
- **Documentation Updates**:
  - Exchange payment workflow documentation updated with automatic collection
  - Logging documentation for path collection events
  - Troubleshooting guide for path collection failures

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**:
  - Automatic path collection trigger rate (target: 100% when needed)
  - Exchange payment success rate (target: match single-equivalent success rate)
  - Manual topology collection command usage (target: 0 for exchange payments)
- **Leading Indicators**: Resource signal emission count, path cache hit rate
- **Baseline Values**: Current exchange payment success rate with manual topology collection
- **Target Values**:
  - 100% automatic path collection when paths missing/expired
  - Exchange payment success rate ≥ 95% (matching single-equivalent)
  - Path cache hit rate > 80% after initial collection
  - Path collection latency < 5 seconds (95th percentile)

### Monitoring Plan
- **New Dashboards/Alerts**: Path collection request count, resource timeout rate, cache hit/miss ratio
- **Enhanced Monitoring**: Extended logging for path availability checks, collection triggers, resource arrivals
- **A/B Testing**: Not applicable

### Review Schedule
- **Daily**: Development progress and blocker identification
- **Weekly**: Unit test results and integration validation
- **Post-Implementation Review**: Payment success rates, user feedback on automatic collection

## Appendices
### Glossary
- **Path Cache Key**: Unique identifier for cached exchange paths: `(contractorID, senderEquivalent, receiverEquivalent)`
- **Path Expiry**: TTL-based mechanism determining when cached paths are no longer considered valid (150 seconds)
- **Resource-Based Communication**: Pattern using `ResourcesManager` signals and `BaseResource` objects for asynchronous transaction communication
- **Exchange Amount**: Amount sender must pay (in sender equivalent) to deliver requested amount to receiver (in receiver equivalent)

### References
- [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
- [Exchange Flow Calculation PRD](04-exchange-flow-calculation.md)
- [Payment Estimation PRD](05-payment-estimation.md)
- [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
- [CoordinatorPaymentTransaction Implementation](../../../src/core/transactions/transactions/regular/payments/CoordinatorPaymentTransaction.h)
- [FindPathByMaxFlowTransaction Implementation](../../../src/core/transactions/transactions/find_path/FindPathByMaxFlowTransaction.h)

### Detailed Component Specifications

#### New Transaction Class

##### FindPathsByMaxFlowExchangeTransaction
- **Location**: `src/core/transactions/transactions/find_path/FindPathsByMaxFlowExchangeTransaction.h/.cpp`
- **Inheritance**: Extends `BaseCollectTopologyForExchangeTransaction`
- **Purpose**: Collect topology and build exchange paths on demand for payment coordinator
- **Constructor Parameters**:
  ```cpp
  FindPathsByMaxFlowExchangeTransaction(
      BaseAddress::Shared contractorAddress,           // Target contractor
      const TransactionUUID &requestedTransactionUUID, // Requesting coordinator UUID
      const SerializedEquivalent receiverEquivalent,   // Receiver's equivalent
      const vector<SerializedEquivalent> &exchangeEquivalents, // Sender's equivalents
      ContractorsManager *contractorsManager,
      ResourcesManager *resourcesManager,
      EquivalentsSubsystemsRouter *equivalentsSubsystemsRouter,
      TailManager *tailManager,
      ExchangePathsManager *exchangePathsManager,
      ExchangeRatesManager *exchangeRatesManager,
      CommissionsManager *commissionsManager,
      Logger &logger,
      HopsCount_t hopsCount);
  ```
- **Key Methods**:
  - `sendRequestForCollectingTopology()`: Initiates topology collection (inherits pattern from `BaseCollectTopologyForExchangeTransaction`)
  - `processCollectingTopology()`: Builds paths using OR-Tools, caches in `ExchangePathsManager`, returns `ExchangePathsResource`
- **Key Fields**:
  - `ContractorID mContractorID`: Target contractor for paths
  - `BaseAddress::Shared mContractorAddress`: Contractor address
  - `TransactionUUID mRequestedTransactionUUID`: UUID of requesting coordinator transaction
  - `SerializedEquivalent mReceiverEquivalent`: Receiver's equivalent (mEquivalent)
  - `vector<SerializedEquivalent> mExchangeEquivalents`: Sender's equivalents
  - `ExchangePathsManager *mExchangePathsManager`: For path caching
  - `HopsCount_t mHopsCount`: Topology collection depth

#### New Resource Class

##### ExchangePathsResource
- **Location**: `src/core/resources/resources/ExchangePathsResource.h`
- **Inheritance**: Extends `BaseResource`
- **Purpose**: Signal completion of exchange path collection
- **Definition**:
  ```cpp
  class ExchangePathsResource : public BaseResource {
  public:
      static const byte_t kResourceType = ExchangePaths;

      ExchangePathsResource(const TransactionUUID &transactionUUID);

      const TransactionUUID& transactionUUID() const;

      const byte_t resourceType() const override {
          return kResourceType;
      }

  private:
      TransactionUUID mTransactionUUID;
  };
  ```

#### Modified Component Specifications

##### ResourcesManager Extensions
- **Location**: `src/core/resources/manager/ResourcesManager.h/.cpp`
- **New Signal**:
  ```cpp
  typedef signals::signal<void(
      const TransactionUUID&,                  // Requesting transaction UUID
      BaseAddress::Shared,                     // Contractor address
      const vector<SerializedEquivalent>&,     // Exchange equivalents (sender)
      const SerializedEquivalent)>             // Receiver equivalent
  RequestExchangePathsResourceSignal;

  mutable RequestExchangePathsResourceSignal requestExchangePathsResourceSignal;
  ```
- **New Method**:
  ```cpp
  void requestExchangePaths(
      const TransactionUUID &transactionUUID,
      BaseAddress::Shared contractorAddress,
      const vector<SerializedEquivalent> &exchangeEquivalents,
      const SerializedEquivalent receiverEquivalent) const;
  ```

##### Core Signal Connection
- **Location**: `src/core/Core.cpp` in `Core::connectResourcesManagerSignals()`
- **Addition**:
  ```cpp
  void Core::connectResourcesManagerSignals() {
      // ... existing signal connections ...

      mResourcesManager->requestExchangePathsResourceSignal.connect(
          boost::bind(
              &Core::onExchangePathsResourceRequestedSlot,
              this,
              _1, _2, _3, _4));

      // ... rest of method ...
  }
  ```
- **New Slot Method**:
  ```cpp
  void Core::onExchangePathsResourceRequestedSlot(
      const TransactionUUID &transactionUUID,
      BaseAddress::Shared contractorAddress,
      const vector<SerializedEquivalent> &exchangeEquivalents,
      const SerializedEquivalent receiverEquivalent) {

      auto transaction = make_shared<FindPathsByMaxFlowExchangeTransaction>(
          contractorAddress,
          transactionUUID,
          receiverEquivalent,
          exchangeEquivalents,
          mContractorsManager.get(),
          mResourcesManager.get(),
          mEquivalentsSubsystemsRouter.get(),
          mTailManager.get(),
          mExchangePathsManager.get(),
          mExchangeRatesManager.get(),
          mCommissionsManager.get(),
          mLogger,
          /* hopsCount */ 7); // TODO: make configurable

      mTransactionsManager->scheduleTransaction(transaction);
  }
  ```

##### CoordinatorExchangePaymentTransaction Modifications

**New Constant**:
```cpp
class CoordinatorExchangePaymentTransaction : public BaseExchangePaymentTransaction {
    static const uint32_t kExchangePathsCacheTTLSeconds = 150;
    // ... rest of class ...
};
```

**Modified Stage Method** (`runPaymentInitializationStage()`):
- Removes: `mExchangeAmount` calculation
- Removes: `kTotalOutgoingPossibilities` validation
- Adds: Path availability checking for all `mExchangeEquivalents`
- Adds: Path expiry checking using TTL constant
- Adds: Resource request logic for missing/expired equivalents
- Adds: Wait state for `ExchangePathsResource`

**Modified Stage Method** (`runPathsResourceProcessingStage()`):
- Adds: `mExchangeAmount` calculation at method start
- Adds: `kTotalOutgoingPossibilities` validation after calculation
- Existing: Path retrieval and processing logic (unchanged)

**Resource Handling**:
- Existing `BaseExchangePaymentTransaction` resource handling infrastructure supports `ExchangePathsResource`
- Resource arrival triggers transition to `runPathsResourceProcessingStage()`

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-10-18 | Claude Code | Initial draft for exchange payment topology collection integration | Phase 1 |
| 1.1 | 2025-10-18 | Claude Code | Fixed TTL semantics to `age >= TTL`; Added `customTTL` parameter to `retrievePaths()`; Removed direct `mCachedPaths` access (encapsulation fix) | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [06-exchange-payment-with-commissions.md](06-exchange-payment-with-commissions.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)
- **Payment Protocol**: [payment-protocol.md](../../../architecture/vtcpd/protocols/payment-protocol.md)
