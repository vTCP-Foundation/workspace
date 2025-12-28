# 17-02 - Core Integration

# Links
- [PRD-17: Block Number Cache](../../prd/vtcpd/17-block-number-cache.md)
- [Task 17-01: BlockNumberCache Class](17-01-block-number-cache-class.md)

# Description

Integrate the `BlockNumberCache` class (created in Task 17-01) into the `Core` component to enable transparent caching of `GetBlockNumber` RPC requests. This integration intercepts outgoing RPC requests, checks the cache, and either returns a cached response or forwards the request to the observer.

**Key behavior:**
- When a `GetBlockNumber` request arrives and cache is valid: create response from cache and send directly to TransactionsManager (no observer RPC)
- When cache is invalid (empty or TTL expired): forward request to observer as normal
- When a successful `GetBlockNumber` response is received from observer: update the cache
- If observer request fails and cache is expired: return error to transaction (no stale predictions)

This provides transparent caching without modifying transaction code.

# Requirements and DOD

## Functional Requirements

1. **FR-01**: Add `#include "network/rpc/BlockNumberCache.h"` to `Core.h`
2. **FR-02**: Add member `unique_ptr<BlockNumberCache> mBlockNumberCache` to `Core` class
3. **FR-03**: Initialize `mBlockNumberCache` in `Core::init()` or appropriate initialization location (after Logger is available)
4. **FR-04**: Modify `Core::onRpcRequestSlot()` to check cache for `GetBlockNumber` requests:
   - If `request->method() == RpcMethod::GetBlockNumber`:
     - If `!mBlockNumberCache->needsRefresh()`:
       - Create `GetBlockNumberRpcResponse` with cached block number
       - Call `mTransactionsManager->onRpcResponseReceived(response)`
       - Return early (do not forward to observer)
     - Else: continue with existing logic (forward to observer)
5. **FR-05**: Modify `Core::onRpcResponseSlot()` to update cache on successful responses:
   - If `response->method() == RpcMethod::GetBlockNumber && response->isSuccess()`:
     - Cast to `GetBlockNumberRpcResponse`
     - Call `mBlockNumberCache->update(blockResponse->blockNumber())`
6. **FR-06**: Add appropriate logging for cache hits and misses

## Non-Functional Requirements

1. **NFR-01**: Maintain backwards compatibility - transactions work unchanged
2. **NFR-02**: No changes to RPC flow for non-GetBlockNumber methods
3. **NFR-03**: Existing tests must continue to pass

## Definition of Done

- [ ] BlockNumberCache member added to Core
- [ ] Cache initialized during Core startup
- [ ] onRpcRequestSlot() checks cache and returns cached response when valid
- [ ] onRpcResponseSlot() updates cache on successful GetBlockNumber responses
- [ ] Code compiles without errors or warnings
- [ ] Logging added for cache hits/misses
- [ ] Existing functionality preserved (no regressions)

# Implementation Plan

## Step 1: Modify Core.h

Add include and member:

```cpp
// In includes section
#include "network/rpc/BlockNumberCache.h"

// In private members section (near mObserverRpcCommunicator)
unique_ptr<BlockNumberCache> mBlockNumberCache;
```

## Step 2: Initialize BlockNumberCache in Core

Find appropriate initialization location (likely in `Core::init()` after Logger initialization or near `initObserverRpcCommunicator()`):

```cpp
mBlockNumberCache = make_unique<BlockNumberCache>(*mLog);
```

## Step 3: Modify Core::onRpcRequestSlot()

Location: `src/core/Core.cpp`, method `onRpcRequestSlot()`

Add cache check before forwarding to observer:

```cpp
void Core::onRpcRequestSlot(RpcRequest::Shared request)
{
    // ... existing validation code ...

    // NEW: Check cache for GetBlockNumber requests
    if (request->method() == RpcMethod::GetBlockNumber) {
        if (mBlockNumberCache && !mBlockNumberCache->needsRefresh()) {
            info() << "GetBlockNumber cache hit for transaction "
                   << request->transactionUUID();
            auto response = make_shared<GetBlockNumberRpcResponse>(
                request->transactionUUID(),
                RpcResponseStatus::Success,
                mBlockNumberCache->getCachedBlockNumber());
            mTransactionsManager->onRpcResponseReceived(response);
            return;
        }
        debug() << "GetBlockNumber cache miss, forwarding to observer";
    }

    // ... existing code to forward to observer ...
}
```

## Step 4: Modify Core::onRpcResponseSlot()

Location: `src/core/Core.cpp`, method `onRpcResponseSlot()`

Add cache update after receiving successful response:

```cpp
void Core::onRpcResponseSlot(RpcResponse::Shared response)
{
    // ... existing code ...

    // NEW: Update cache on successful GetBlockNumber response
    if (response->method() == RpcMethod::GetBlockNumber && response->isSuccess()) {
        auto blockResponse = dynamic_pointer_cast<GetBlockNumberRpcResponse>(response);
        if (blockResponse && mBlockNumberCache) {
            mBlockNumberCache->update(blockResponse->blockNumber());
            info() << "BlockNumberCache updated with block "
                   << blockResponse->blockNumber();
        }
    }

    // ... existing code to forward response to TransactionsManager ...
}
```

## Step 5: Add Required Include

Ensure `GetBlockNumberRpcResponse.h` is included in Core.cpp if not already:

```cpp
#include "network/rpc/responses/GetBlockNumberRpcResponse.h"
```

## Step 6: Verify Build and Test

1. Build the project
2. Run existing tests to ensure no regressions
3. Manual verification: observe logs during payment transaction

# Test Plan

**Complexity**: Moderate

This task modifies Core's RPC handling flow. Full testing requires integration with actual transactions.

**Validation for this task:**

1. **Build Validation**: Code compiles without errors or warnings
2. **Regression Testing**: Existing unit tests pass unchanged
3. **Manual Integration Test**:
   - Start node with observer configured
   - Execute a payment transaction that triggers GetBlockNumber
   - Verify in logs:
     - First request: "cache miss, forwarding to observer"
     - Cache update: "BlockNumberCache updated with block X"
   - Execute another payment transaction within 10 minutes
   - Verify in logs:
     - "GetBlockNumber cache hit for transaction Y"
     - No observer RPC for this request

**Demo:**
- Show logs demonstrating cache miss on first request
- Show logs demonstrating cache hit on subsequent request
- Show that payment transactions complete successfully

# Verification and Validation

## Architecture integrity
- Integration follows existing Core patterns
- Uses existing signal flow (no new signals)
- Cache is a private member, not exposed externally
- Maintains separation of concerns (Core handles caching, transactions unchanged)

## Security
- N/A - Block number is public information
- No new attack vectors introduced

## Performance
- Cache hit path: O(1) lookup, no network I/O
- Reduces observer load during high transaction volume
- Minimal overhead for cache miss (one conditional check)

## Scalability
- Single cache instance per node (appropriate for design)
- Cache benefits multiply with concurrent transactions

## Reliability
- Cache miss gracefully falls back to observer
- Error responses from observer are not cached
- Cache clear on restart (intentional - no stale data)

## Maintainability
- Changes isolated to Core.h and Core.cpp
- Clear logging for debugging
- Follows existing code patterns

## Cost
- Reduces network traffic to observer
- Minimal memory overhead

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Only modify Core.h and Core.cpp
- Do not modify transaction classes
- Do not modify ObserverRpcCommunicator
- Preserve all existing functionality
