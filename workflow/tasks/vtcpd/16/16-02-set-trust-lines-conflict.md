# 16-02 - Implement setTrustLinesToConflictState() Method

# Links
- [PRD](../../prd/vtcpd/16-payment-transaction-observing-states.md)
- [Previous task: 16-01-enum-stages-constant](16-01-enum-stages-constant.md)

# Description

This task implements the `setTrustLinesToConflictState()` method in `BaseExchangePaymentTransaction`. This method is called when the observer rejects an `AcceptClaim` request because the claim window has already closed (message contains "must be less than current block number").

When this happens, all trust lines that have reservations in the current transaction must be set to `TrustLineState::Conflict` status. This marks them for manual resolution since the transaction cannot be properly finalized through the normal observing process.

The method iterates over `mReservations` (which maps `ContractorID` to vectors of `PathID` and `AmountReservation` pairs), extracts the equivalent from each reservation, and uses the appropriate `TrustLinesManager` to set the trust line state.

# Requirements and DOD

## Requirements

1. **Method Declaration**
   - Add `void setTrustLinesToConflictState()` declaration to `BaseExchangePaymentTransaction.h`
   - Method should be protected (accessible by derived classes)

2. **Method Implementation**
   - Iterate over all entries in `mReservations`
   - For each `ContractorID` and its reservations:
     - Get the equivalent from the reservation (`pathIDAndReservation.second->equivalent()`)
     - Get the TrustLinesManager for that equivalent via `trustLinesManager(equivalent)`
     - Call `setTrustLineState(contractorID, TrustLineState::Conflict)`
   - Handle potential duplicates gracefully (same trust line may be set multiple times, which is harmless)

3. **Logging**
   - Add info-level log at method entry indicating the action
   - Consider debug-level logging for each trust line affected

## Definition of Done

- [ ] Method declaration added to `BaseExchangePaymentTransaction.h`
- [ ] Method implementation added to `BaseExchangePaymentTransaction.cpp`
- [ ] Method iterates over all reservations and sets trust lines to Conflict state
- [ ] Appropriate logging added
- [ ] Code compiles without warnings
- [ ] No existing functionality is broken

# Implementation Plan

## Step 1: Add Method Declaration

File: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.h`

Add in the protected section (near other helper methods like `rollBack()`):

```cpp
protected:
    // ... existing methods ...
    void setTrustLinesToConflictState();
```

## Step 2: Implement Method

File: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.cpp`

Add implementation:

```cpp
void BaseExchangePaymentTransaction::setTrustLinesToConflictState()
{
    info() << "Setting trust lines with reservations to Conflict state";

    for (const auto &nodeAndReservations : mReservations) {
        auto contractorID = nodeAndReservations.first;
        for (const auto &pathIDAndReservation : nodeAndReservations.second) {
            auto equivalent = pathIDAndReservation.second->equivalent();
            auto tlManager = trustLinesManager(equivalent);

            debug() << "Setting trust line to Conflict: contractor " << contractorID
                    << ", equivalent " << equivalent;

            tlManager->setTrustLineState(
                contractorID,
                TrustLineState::Conflict);
        }
    }
}
```

## Step 3: Verify TrustLineState::Conflict Exists

Before implementation, verify that `TrustLineState::Conflict` exists in the codebase. Check:
- `src/core/trust_lines/TrustLine.h` or similar for enum definition

## Step 4: Verify Compilation

```bash
make -j$(nproc)
```

# Test Plan

**Complexity**: Simple

This method is a straightforward iteration and delegation to existing `TrustLinesManager` API.

## Validation Approach

1. **Compilation Test**: Code must compile without warnings
2. **Code Review**: Verify logic matches PRD specification
3. **Integration Testing**: Will be tested as part of Task 16-03 when `runObservingAcceptClaimStage()` calls this method

## Edge Cases to Consider

- Empty `mReservations` - method should handle gracefully (no-op)
- Same trust line appears multiple times in reservations - should not cause errors
- TrustLinesManager returns nullptr - should not happen in normal flow, but consider defensive check

# Verification and Validation

## Architecture integrity
- Method follows existing patterns for trust line state manipulation
- Uses established `trustLinesManager()` accessor method
- Consistent with other trust line modification patterns in the codebase

## Security
- N/A - No external input processing
- Trust line state changes are authorized within the transaction context

## Performance
- Method iterates once over all reservations - O(n) complexity
- No blocking operations
- Acceptable for the use case (called once per transaction conflict)

## Scalability
- Scales linearly with number of reservations
- Typical reservation count is small (path length limited by `kMaxPathLength = 7`)

## Reliability
- Uses existing, tested TrustLinesManager API
- Idempotent - calling multiple times with same data produces same result

## Maintainability
- Clear, readable implementation
- Logging provides visibility into operation
- Follows existing code patterns

## Cost
- N/A - No cost implications

## Compliance
- Follows existing code style and conventions

# Restrictions
- Commit changes only after successful compilation
- Do not modify TrustLinesManager or TrustLineState - use existing API only
