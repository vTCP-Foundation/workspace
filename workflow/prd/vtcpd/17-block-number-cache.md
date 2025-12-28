# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Block Number Cache
- **PRD ID**: 17
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.0
- **Date**: 2025-12-28
- **Author(s)**: Claude (AI Agent)
- **Stakeholders**: Development Team
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-12-28
- **Previous PRD**: [16-payment-transaction-observing-states.md](16-payment-transaction-observing-states.md)
- **Related Documents**:
  - [14-async-observer-rpc-communication.md](14-async-observer-rpc-communication.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../policy.md).

## Executive Summary

This PRD introduces a caching mechanism for blockchain block numbers to reduce the frequency of RPC requests to the observer node. Currently, each payment transaction (Coordinator, Intermediate, Receiver) makes individual `GetBlockNumber` RPC requests, which creates unnecessary load on the observer when multiple transactions run concurrently.

- **Current project state**: Payment transactions successfully communicate with the observer via async RPC (PRD-14). Each transaction independently requests the current block number.
- **This iteration's focus**: Implement `BlockNumberCache` component that caches block numbers with predictive extrapolation based on known block generation period.
- **Connection to overall vision**: Reduces observer load, improves transaction performance, and prepares infrastructure for scaling.

## Iteration Context

### Previous Iterations Summary
- **Completed Features**:
  - Async observer RPC communication (PRD-14)
  - Payment transaction observing states (PRD-16)
  - Exchange payment with commissions (PRD-06)
- **Technical Debt**: None identified for this scope
- **User Feedback**: N/A

### Current State Analysis
- **What's working well**: RPC communication with observer is stable and async
- **Pain points identified**: Every transaction makes separate `GetBlockNumber` requests, creating redundant observer load
- **Performance metrics**: Each node makes N requests per N concurrent transactions instead of 1 shared request

## Problem Statement

### Background

Payment transactions (CoordinatorExchangePaymentTransaction, IntermediateNodeExchangePaymentTransaction, ReceiverExchangePaymentTransaction) require the current blockchain block number at specific stages:
- Coordinator: after amount collection, after capacity validation, during direct path processing
- Intermediate/Receiver: during final reservations confirmation

Each transaction independently calls:
```cpp
sendRpcRequest(make_shared<GetBlockNumberRpcRequest>(mTransactionUUID));
```

### Problem Description

- **Who is affected**: All nodes running payment transactions; the observer node handling requests
- **When and where**: During every payment transaction at block number retrieval stages
- **Impact of not solving**:
  - Unnecessary network traffic to observer
  - Increased latency due to redundant RPC calls
  - Observer overload during high transaction volume periods

### Success Metrics

| Metric | Current | Target |
|--------|---------|--------|
| RPC requests per transaction | 1+ per transaction | 0-1 (cache hit reduces to 0) |
| Cache hit rate | N/A | >90% during normal operation |
| Block number prediction accuracy | N/A | Within `kAllowableBlockNumberDifference` (10 blocks) |

## Project Scope

### This Iteration's Scope

#### New Features/Enhancements

1. **BlockNumberCache Component**
   - New standalone class for caching and predicting block numbers
   - Predictive extrapolation based on block generation period
   - Configurable TTL for cache validity
   - Integration with existing RPC flow

#### Bug Fixes & Technical Improvements
- None

#### Modifications to Existing Features
- Modify `Core` to intercept `GetBlockNumber` requests and check cache before forwarding to observer
- Update RPC response handling to refresh cache on successful responses

### Explicitly Out of Scope
- Background refresh mechanism (proactive cache updates)
- Metrics/monitoring for prediction accuracy
- Persistent cache storage across restarts
- Configuration via config file (constants only)

### Dependencies from Previous Iterations
- PRD-14: Async observer RPC communication infrastructure
- Existing RPC request/response signal flow in Core

### Future Roadmap Impact
- Foundation for additional RPC caching (e.g., claim statuses)
- Potential for shared cache across multiple observer interactions

## User Stories & Requirements

### User Personas

#### Primary User: VTCP Node Operator
- **Role**: Operates a node participating in payment transactions
- **Goals**: Minimize external dependencies, reduce network traffic
- **Pain Points**: Observer overload during peak transaction periods
- **Technical Proficiency**: Advanced

### Functional Requirements

#### New Features for This Iteration

1. **BlockNumberCache Class**
   - **Description**: Caches the current block number with timestamp, provides predicted block numbers based on elapsed time
   - **User Story**: As a node operator, I want my node to cache block numbers so that redundant observer requests are avoided
   - **Rationale**: Reduces observer load and improves transaction latency
   - **Builds Upon**: Existing RPC infrastructure
   - **Acceptance Criteria**:
     - Cache stores block number and retrieval timestamp
     - Returns cached/predicted value when within TTL
     - Returns `needsRefresh() = true` when cache is empty or TTL expired
     - Prediction adds blocks based on elapsed time and block generation period
     - Constants: `kBlockGenerationSeconds = 60`, `kCacheTTLSeconds = 600`
   - **Priority**: High
   - **Dependencies**: None

2. **Core Integration**
   - **Description**: Integrate BlockNumberCache into Core's RPC request handling
   - **User Story**: As a node, when I request block number and cache is valid, I receive cached value without observer RPC
   - **Rationale**: Transparent caching without modifying transaction code
   - **Builds Upon**: Core RPC signal handling
   - **Acceptance Criteria**:
     - `Core::onRpcRequestSlot()` checks cache for `GetBlockNumber` requests
     - If cache valid: generate response from cache and send via `mTransactionsManager->onRpcResponseReceived()`
     - If cache invalid: forward request to observer as normal
     - `Core::onRpcResponseSlot()` updates cache on successful `GetBlockNumber` responses
     - When cache is expired and observer request fails: return error (not stale prediction)
   - **Priority**: High
   - **Dependencies**: BlockNumberCache class

### Non-Functional Requirements

#### Performance
- Cache lookup: O(1) time complexity
- No additional memory allocations during cache hit path
- Prediction calculation: simple arithmetic, negligible overhead

#### Security
- No security implications (block number is public information)

#### Scalability
- Single cache instance per node (sufficient for single-threaded model)

#### Reliability
- Cache miss gracefully falls back to observer request
- Error from observer when cache expired returns error to transaction

## Technical Specifications

### Architecture Evolution

#### Current Architecture
```
Transaction.sendRpcRequest()
    → TransactionsManager.rpcRequestSignal
        → Core.onRpcRequestSlot()
            → ObserverRpcCommunicator.sendRequest()
                → HTTP to Observer
```

#### Proposed Changes
```
Transaction.sendRpcRequest()
    → TransactionsManager.rpcRequestSignal
        → Core.onRpcRequestSlot()
            → [NEW] Check BlockNumberCache
                → If valid: Create response, send to TransactionsManager
                → If invalid: ObserverRpcCommunicator.sendRequest()
                    → HTTP to Observer
                        → [NEW] Update BlockNumberCache on success
```

#### Backwards Compatibility
- Transaction code unchanged
- RPC flow unchanged for non-cached methods
- Existing tests remain valid

### Technology Stack Updates
- No new technologies or libraries required
- Uses existing `boost::posix_time` for timestamps (consistent with `utc_now()`)

### Data Requirements

#### Data Models

**BlockNumberCache State:**
```cpp
class BlockNumberCache {
public:
    static constexpr uint32_t kBlockGenerationSeconds = 60;   // 1 minute
    static constexpr uint32_t kCacheTTLSeconds = 600;         // 10 minutes

    BlockNumberCache(Logger &logger);

    // Check if cache needs refresh (empty or TTL expired)
    bool needsRefresh() const;

    // Get cached/predicted block number (only valid when !needsRefresh())
    BlockNumber getCachedBlockNumber() const;

    // Update cache with fresh value from observer
    void update(BlockNumber blockNumber);

    // Clear cache (for testing)
    void clear();

private:
    BlockNumber mCachedBlockNumber;
    DateTime mCacheTimestamp;
    bool mHasValue;
    Logger &mLogger;
};
```

#### Data Storage
- In-memory only
- No persistence across restarts

## Implementation Plan

### This Iteration Timeline
- **Duration**: Single iteration
- **Key Deliverables**:
  1. BlockNumberCache class implementation
  2. Core integration
  3. Unit tests

### Iteration Milestones

| Milestone | Description | Dependencies | Risk Level |
|-----------|-------------|--------------|------------|
| BlockNumberCache class | Standalone cache with prediction | None | Low |
| Core integration | Request interception and response handling | BlockNumberCache | Medium |
| Unit tests | Test cache logic and prediction | BlockNumberCache | Low |

## Risk Management

### Technical Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| Prediction drift from actual block number | Low | Low | TTL limits maximum drift; `kAllowableBlockNumberDifference=10` in transactions provides tolerance |
| Cache stale during long node operation | Low | Low | 10-minute TTL ensures regular refresh |
| Race condition in cache access | Medium | Very Low | Single-threaded io_context model prevents races |

## Testing Strategy

### Testing Approach for This Iteration

#### New Feature Testing

**Unit Tests for BlockNumberCache:**

| # | Test Name | Description |
|---|-----------|-------------|
| 1 | `testInitialStateHasNoCache` | After construction, `needsRefresh()` returns true |
| 2 | `testUpdateStoresBlockNumberAndTimestamp` | `update()` stores value, `needsRefresh()` returns false |
| 3 | `testClearResetsCache` | `clear()` resets to initial state |
| 4 | `testNeedsRefreshReturnsTrueWhenCacheEmpty` | Empty cache requires refresh |
| 5 | `testNeedsRefreshReturnsFalseWithinTTL` | Cache valid within TTL period |
| 6 | `testGetCachedBlockNumberReturnsExactValueWhenJustUpdated` | Returns exact value immediately after update |
| 7 | `testGetCachedBlockNumberNeverDecreasesOnPrediction` | Predicted value >= initial cached value |
| 8 | `testConstantsHaveExpectedValues` | Verify `kBlockGenerationSeconds=60`, `kCacheTTLSeconds=600` |

> **Note**: Tests `testNeedsRefreshReturnsTrueAfterTTLExpired` and `testGetCachedBlockNumberPredictsCorrectly` were excluded - they require long wait times (600s and 60s respectively) without injectable clock, which is not suitable for unit tests.

**Test Implementation Notes:**
- Use injectable clock pattern (like `utc_now()` in ExchangeRatesManager)
- For time-dependent tests, use mock time or short sleep intervals
- Follow existing test patterns in `tests/unit/rates/ExchangeRatesManagerTest.cpp`

#### Regression Testing
- Existing payment transaction tests should pass unchanged
- Existing RPC tests should pass unchanged

### Quality Gates
- All unit tests pass
- Build succeeds without warnings
- Code follows project style (consistent with existing components)

## Deployment & Release Strategy

### Release Approach
- **Release Type**: Minor feature enhancement
- **Rollout Strategy**: Immediate (no feature flag needed)
- **Rollback Plan**: Remove cache integration from Core (transactions still work via direct observer calls)

### Database Migrations
- None required

## Success Metrics & Monitoring

### Iteration-Specific KPIs
- **Primary Metrics**: Reduced observer RPC calls during transaction execution
- **Leading Indicators**: Cache hit rate (can be logged for debugging)
- **Baseline Values**: 1 RPC call per transaction per block number request
- **Target Values**: ~0 RPC calls for transactions within 10-minute window of first request

## Appendices

### Glossary

| Term | Definition |
|------|------------|
| Block Number | Sequential identifier for blockchain blocks |
| TTL | Time-To-Live: duration for which cached value is considered valid |
| Observer | External node providing blockchain state information |
| RPC | Remote Procedure Call |

### References
- `src/core/network/rpc/ObserverRpcCommunicator.h` - RPC communication
- `src/core/Core.cpp` - RPC signal handling (`onRpcRequestSlot`, `onRpcResponseSlot`)
- `src/core/rates/manager/ExchangeRatesManager.h` - Similar caching pattern reference
- `tests/unit/rates/ExchangeRatesManagerTest.cpp` - Test pattern reference

### API Documentation

**BlockNumberCache Public Interface:**

```cpp
class BlockNumberCache {
public:
    // Constants
    static constexpr uint32_t kBlockGenerationSeconds = 60;
    static constexpr uint32_t kCacheTTLSeconds = 600;

    // Constructor
    explicit BlockNumberCache(Logger &logger);

    // Returns true if cache is empty or TTL has expired
    bool needsRefresh() const;

    // Returns cached block number with prediction based on elapsed time
    // Precondition: !needsRefresh() (behavior undefined if cache is stale)
    BlockNumber getCachedBlockNumber() const;

    // Updates cache with fresh block number from observer
    void update(BlockNumber blockNumber);

    // Clears cache (primarily for testing)
    void clear();
};
```

**Integration Points in Core:**

```cpp
// In Core.h - add member
unique_ptr<BlockNumberCache> mBlockNumberCache;

// In Core::onRpcRequestSlot() - add cache check
if (request->method() == RpcMethod::GetBlockNumber) {
    if (!mBlockNumberCache->needsRefresh()) {
        auto response = make_shared<GetBlockNumberRpcResponse>(
            request->transactionUUID(),
            RpcResponseStatus::Success,
            mBlockNumberCache->getCachedBlockNumber());
        mTransactionsManager->onRpcResponseReceived(response);
        return;
    }
}
// ... existing code to forward to observer

// In Core::onRpcResponseSlot() - add cache update
if (response->method() == RpcMethod::GetBlockNumber && response->isSuccess()) {
    auto blockResponse = dynamic_pointer_cast<GetBlockNumberRpcResponse>(response);
    if (blockResponse) {
        mBlockNumberCache->update(blockResponse->blockNumber());
    }
}
```

---

**Document History**

| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-12-28 | Claude (AI Agent) | Initial draft | Phase 1 |

**Related Documents**
- **Previous Iteration PRD**: [16-payment-transaction-observing-states.md](16-payment-transaction-observing-states.md)
- **Technical Reference**: [14-async-observer-rpc-communication.md](14-async-observer-rpc-communication.md)
