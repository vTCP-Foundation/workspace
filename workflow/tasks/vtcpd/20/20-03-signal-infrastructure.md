# 20-03 - Signal Infrastructure

# Links
- [PRD-20: Cleanup Historical Crypto Data](../../prd/vtcpd/20-cleanup-historical-crypto-data.md)

# Description
Add the signal infrastructure required for triggering the cleanup transaction. This includes adding a new signal type `HistoryCryptoDataCleanupSignal` to `BaseTransaction` and setting up the subscription and launch mechanism in `TransactionsManager`.

This task is part of PRD-20 which implements automatic cleanup of obsolete historical crypto data. The signal mechanism allows audit transactions to trigger cleanup after successful completion, following the existing pattern established by `TrustLineActionSignal`.

# Requirements and DOD

## Requirements
1. Add `HistoryCryptoDataCleanupSignal` typedef to `BaseTransaction`
   - Signature: `signals::signal<void(ContractorID, const SerializedEquivalent)>`
2. Add `historyCryptoDataCleanupSignal` member to `BaseTransaction`
3. Add subscription method `subscribeForHistoryCryptoDataCleanupSignal` to `TransactionsManager`
4. Add slot method `onHistoryCryptoDataCleanupSlot` to `TransactionsManager`
5. Add launch method `launchCleanupHistoricalCryptoDataTransaction` to `TransactionsManager` (can be stub initially, full implementation in Task 20-05)

## Definition of Done
- [ ] `HistoryCryptoDataCleanupSignal` typedef added to `BaseTransaction.h`
- [ ] `historyCryptoDataCleanupSignal` mutable member added to `BaseTransaction.h`
- [ ] Subscription method added to `TransactionsManager.h/.cpp`
- [ ] Slot method added to `TransactionsManager.h/.cpp`
- [ ] Launch method stub added to `TransactionsManager.h/.cpp`
- [ ] Code compiles without errors
- [ ] Signal infrastructure follows existing patterns (similar to `TrustLineActionSignal`)

# Implementation Plan

## Step 1: Update BaseTransaction Header
**File**: `src/core/transactions/transactions/base/BaseTransaction.h`

Add after existing signal typedefs (around line 48-52):
```cpp
typedef signals::signal<void(ContractorID, const SerializedEquivalent)> HistoryCryptoDataCleanupSignal;
```

Add after existing signal members (around line 390-393):
```cpp
mutable HistoryCryptoDataCleanupSignal historyCryptoDataCleanupSignal;
```

## Step 2: Add Subscription Method to TransactionsManager
**File**: `src/core/transactions/manager/TransactionsManager.h`

Add in protected section with other subscription methods (around line 513-523):
```cpp
void subscribeForHistoryCryptoDataCleanupSignal(
    BaseTransaction::HistoryCryptoDataCleanupSignal &signal);
```

**File**: `src/core/transactions/manager/TransactionsManager.cpp`

Implement subscription:
```cpp
void TransactionsManager::subscribeForHistoryCryptoDataCleanupSignal(
    BaseTransaction::HistoryCryptoDataCleanupSignal &signal)
{
    signal.connect(
        boost::bind(
            &TransactionsManager::onHistoryCryptoDataCleanupSlot,
            this,
            _1,
            _2));
}
```

## Step 3: Add Slot Method to TransactionsManager
**File**: `src/core/transactions/manager/TransactionsManager.h`

Add in protected section with other slot methods (around line 588-604):
```cpp
void onHistoryCryptoDataCleanupSlot(
    ContractorID contractorID,
    const SerializedEquivalent equivalent);
```

**File**: `src/core/transactions/manager/TransactionsManager.cpp`

Implement slot:
```cpp
void TransactionsManager::onHistoryCryptoDataCleanupSlot(
    ContractorID contractorID,
    const SerializedEquivalent equivalent)
{
    launchCleanupHistoricalCryptoDataTransaction(
        contractorID,
        equivalent);
}
```

## Step 4: Add Launch Method Stub to TransactionsManager
**File**: `src/core/transactions/manager/TransactionsManager.h`

Add in protected section:
```cpp
void launchCleanupHistoricalCryptoDataTransaction(
    ContractorID contractorID,
    const SerializedEquivalent equivalent);
```

**File**: `src/core/transactions/manager/TransactionsManager.cpp`

Implement as stub (full implementation in Task 20-05):
```cpp
void TransactionsManager::launchCleanupHistoricalCryptoDataTransaction(
    ContractorID contractorID,
    const SerializedEquivalent equivalent)
{
    // TODO: Implement in Task 20-05
    info() << "launchCleanupHistoricalCryptoDataTransaction for contractor "
           << contractorID << " equivalent " << equivalent;
}
```

## Reference: Existing Signal Patterns
Use `TrustLineActionSignal` as reference:
- `BaseTransaction.h` lines 48 and 390
- `TransactionsManager::subscribeForTrustLineActionSignal`
- `TransactionsManager::onTrustLineActionSlot`

# Test Plan

**Complexity**: Simple

**Validation for this task**:
- Code compiles successfully
- Signal can be emitted without runtime errors
- Slot is called when signal is emitted (verify via log output)

No unit tests required for signal infrastructure; validation is through compilation and integration with Task 20-04.

# Verification and Validation

## Architecture integrity
- Signal pattern matches existing `TrustLineActionSignal` pattern exactly
- No changes to existing signal mechanisms
- Clean separation between signal definition (BaseTransaction) and handling (TransactionsManager)

## Security
- No security implications; internal signaling mechanism only

## Performance
- Boost.Signals2 is already used throughout the codebase
- No performance concerns

## Scalability
- N/A for this simple task

## Reliability
- Follows proven signal pattern used throughout codebase
- Signal connections are established during transaction preparation

## Maintainability
- Consistent with existing signal patterns
- Clear naming convention

## Cost
- N/A

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Launch method can be a stub; full implementation is in Task 20-05
