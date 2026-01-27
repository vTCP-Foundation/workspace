# 20-04 - Signal Emission in Audit Transactions

# Links
- [PRD-20: Cleanup Historical Crypto Data](../../prd/vtcpd/20-cleanup-historical-crypto-data.md)
- [Task 20-03: Signal Infrastructure](20-03-signal-infrastructure.md)

# Description
Add emission of `historyCryptoDataCleanupSignal` immediately after `trustLineActionSignal` in four audit-related transactions: `SetOutgoingTrustLineTransaction`, `CloseIncomingTrustLineTransaction`, `AuditSourceTransaction`, and `AuditTargetTransaction`.

This task is part of PRD-20 which implements automatic cleanup of obsolete historical crypto data. The signal emission triggers the cleanup transaction after each successful audit completion.

# Requirements and DOD

## Requirements
1. In `SetOutgoingTrustLineTransaction::runResponseProcessingStage()`, emit `historyCryptoDataCleanupSignal` immediately after `trustLineActionSignal`
2. In `CloseIncomingTrustLineTransaction::runResponseProcessingStage()`, emit `historyCryptoDataCleanupSignal` immediately after `trustLineActionSignal`
3. In `AuditSourceTransaction::runResponseProcessingStage()`, emit `historyCryptoDataCleanupSignal` immediately after `trustLineActionSignal`
4. In `AuditTargetTransaction::runAuditProcessingStage()`, emit `historyCryptoDataCleanupSignal` immediately after `trustLineActionSignal`
5. Signal must pass `mContractorID` and `mEquivalent` as parameters

## Definition of Done
- [ ] Signal emission added to `SetOutgoingTrustLineTransaction.cpp`
- [ ] Signal emission added to `CloseIncomingTrustLineTransaction.cpp`
- [ ] Signal emission added to `AuditSourceTransaction.cpp`
- [ ] Signal emission added to `AuditTargetTransaction.cpp`
- [ ] Code compiles without errors
- [ ] Signal is emitted only after successful audit completion (after `trustLineActionSignal`)

# Implementation Plan

## Step 1: Update SetOutgoingTrustLineTransaction
**File**: `src/core/transactions/transactions/trust_lines/SetOutgoingTrustLineTransaction.cpp`

In `runResponseProcessingStage()`, after line with `trustLineActionSignal(...)` (around line 398-402):
```cpp
    mTrustLines->resetAuditRule(mContractorID);
    trustLineActionSignal(
        mContractorID,
        mEquivalent,
        false);

    // Trigger cleanup of historical crypto data
    historyCryptoDataCleanupSignal(
        mContractorID,
        mEquivalent);

    return resultDone();
```

## Step 2: Update CloseIncomingTrustLineTransaction
**File**: `src/core/transactions/transactions/trust_lines/CloseIncomingTrustLineTransaction.cpp`

In `runResponseProcessingStage()`, after line with `trustLineActionSignal(...)` (around line 379-383):
```cpp
    mTrustLines->resetAuditRule(mContractorID);
    trustLineActionSignal(
        mContractorID,
        mEquivalent,
        false);

    // Trigger cleanup of historical crypto data
    historyCryptoDataCleanupSignal(
        mContractorID,
        mEquivalent);

    return resultDone();
```

## Step 3: Update AuditSourceTransaction
**File**: `src/core/transactions/transactions/trust_lines/AuditSourceTransaction.cpp`

In `runResponseProcessingStage()`, after line with `trustLineActionSignal(...)` (around line 591-595):
```cpp
    mTrustLines->resetAuditRule(mContractorID);
    trustLineActionSignal(
        mContractorID,
        mEquivalent,
        false);

    // Trigger cleanup of historical crypto data
    historyCryptoDataCleanupSignal(
        mContractorID,
        mEquivalent);

    return resultDone();
```

## Step 4: Update AuditTargetTransaction
**File**: `src/core/transactions/transactions/trust_lines/AuditTargetTransaction.cpp`

In `runAuditProcessingStage()`, after line with `trustLineActionSignal(...)` (around line 514-518):
```cpp
    mTrustLines->resetAuditRule(mContractorID);
    trustLineActionSignal(
        mContractorID,
        mEquivalent,
        false);

    // Trigger cleanup of historical crypto data
    historyCryptoDataCleanupSignal(
        mContractorID,
        mEquivalent);

    return resultDone();
```

## Verification
Verify the signal emission locations by checking that:
1. It is placed immediately after `trustLineActionSignal`
2. It is before `return resultDone()`
3. It is within the success path (not in error handling branches)

# Test Plan

**Complexity**: Simple

**Validation for this task**:
- Code compiles successfully
- Signal emission is in correct location (after `trustLineActionSignal`, before `return`)
- Verify via code review that signal is only emitted on successful audit completion

No unit tests required; validation through code review and integration testing when combined with Task 20-05.

# Verification and Validation

## Architecture integrity
- Signal emission follows same pattern as existing `trustLineActionSignal`
- No changes to transaction flow logic
- Cleanup is triggered only after successful audit completion

## Security
- No security implications; cleanup is triggered internally only

## Performance
- Signal emission is lightweight
- Cleanup transaction runs asynchronously

## Scalability
- N/A for this simple task

## Reliability
- Signal emitted only after successful audit state changes
- If signal handler fails, it does not affect the audit transaction (already completed)

## Maintainability
- Consistent pattern across all four transactions
- Clear comment indicating purpose of signal emission

## Cost
- N/A

## Compliance
- N/A

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- This task depends on Task 20-03 being completed (signal must be defined in BaseTransaction)
- Do not modify any other logic in the transactions
