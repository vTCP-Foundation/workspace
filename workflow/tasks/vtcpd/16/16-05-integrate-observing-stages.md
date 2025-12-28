# 16-05 - Integrate Observing Stages into Payment Transactions

# Links
- [PRD](../../prd/vtcpd/16-payment-transaction-observing-states.md)
- [Previous task: 16-03-observing-accept-claim-stage](16-03-observing-accept-claim-stage.md)
- [Previous task: 16-04-observing-get-claim-status-stage](16-04-observing-get-claim-status-stage.md)

# Description

This task integrates the new observing stages into the payment transaction flow by:

1. **Modifying `processNextNodeToCheckVotes()`**: Instead of delegating to `ObservingHandler` and terminating, transition to the new `Observing_AcceptClaim` stage.

2. **Extending `IntermediateNodeExchangePaymentTransaction::run()`**: Add case handlers for the new observing stages.

3. **Extending `ReceiverExchangePaymentTransaction::run()`**: Add case handlers for the new observing stages.

This completes the integration, enabling payment transactions to handle their own observing lifecycle.

# Requirements and DOD

## Requirements

### 1. Modify processNextNodeToCheckVotes()

Current behavior (to be replaced):
```cpp
if (mCountRecoveryAttempts >= kMaxRecoveryAttempts) {
    mObservingHandler->addPaymentClaim(...);
    return resultDone();
}
```

New behavior:
```cpp
if (mCountRecoveryAttempts >= kMaxRecoveryAttempts) {
    mStep = Stages::Observing_AcceptClaim;
    return runObservingAcceptClaimStage();
}
```

- Remove call to `mObservingHandler->addPaymentClaim()`
- Set stage to `Observing_AcceptClaim`
- Return result of `runObservingAcceptClaimStage()`

### 2. Extend IntermediateNodeExchangePaymentTransaction::run()

Add case handlers in the switch statement:

```cpp
case Stages::Observing_AcceptClaim:
    return runObservingAcceptClaimStage();

case Stages::Observing_GetClaimStatus:
    return runObservingGetClaimStatusStage();
```

### 3. Extend ReceiverExchangePaymentTransaction::run()

Add case handlers in the switch statement:

```cpp
case Stages::Observing_AcceptClaim:
    return runObservingAcceptClaimStage();

case Stages::Observing_GetClaimStatus:
    return runObservingGetClaimStatusStage();
```

## Definition of Done

- [ ] `processNextNodeToCheckVotes()` modified to transition to `Observing_AcceptClaim`
- [ ] Call to `mObservingHandler->addPaymentClaim()` removed
- [ ] `IntermediateNodeExchangePaymentTransaction::run()` extended with new cases
- [ ] `ReceiverExchangePaymentTransaction::run()` extended with new cases
- [ ] Code compiles without warnings
- [ ] No existing functionality is broken for non-observing flows

# Implementation Plan

## Step 1: Modify processNextNodeToCheckVotes()

File: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.cpp`

Locate the section (around line 917-925):
```cpp
if (mCountRecoveryAttempts >= kMaxRecoveryAttempts) {
    info() << "Max recovery attempts reached";
    mObservingHandler->addPaymentClaim(
        make_shared<ObservingClaimAppendRequestMessage>(
            mEquivalent,
            mContractorsManager->ownAddresses(),
            currentTransactionUUID(),
            mParticipantsVotesMessage,
            // ... other parameters
        )
    );
    return resultDone();
}
```

Replace with:
```cpp
if (mCountRecoveryAttempts >= kMaxRecoveryAttempts) {
    info() << "Max recovery attempts reached, transitioning to observer monitoring";
    mStep = Stages::Observing_AcceptClaim;
    return runObservingAcceptClaimStage();
}
```

**Important**: Verify the exact location and surrounding context before modifying.

## Step 2: Extend IntermediateNodeExchangePaymentTransaction::run()

File: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.cpp`

Find the `run()` method's switch statement and add new cases. Look for existing patterns like:
```cpp
case Stages::Common_Recovery:
    return runVotesRecoveryParentStage();
```

Add after the last case (before default or at appropriate location):
```cpp
case Stages::Observing_AcceptClaim:
    return runObservingAcceptClaimStage();

case Stages::Observing_GetClaimStatus:
    return runObservingGetClaimStatusStage();
```

## Step 3: Extend ReceiverExchangePaymentTransaction::run()

File: `src/core/transactions/transactions/regular/payments/ReceiverExchangePaymentTransaction.cpp`

Same modification as Step 2. Find the `run()` method's switch statement and add:

```cpp
case Stages::Observing_AcceptClaim:
    return runObservingAcceptClaimStage();

case Stages::Observing_GetClaimStatus:
    return runObservingGetClaimStatusStage();
```

## Step 4: Verify No Orphaned ObservingHandler References

After removing the `mObservingHandler->addPaymentClaim()` call, verify there are no other references to this method in the payment transaction flow that should also be updated.

Search for:
```bash
grep -r "addPaymentClaim" src/core/transactions/
```

## Step 5: Verify Compilation

```bash
make -j$(nproc)
```

## Step 6: Verify Stage Handling Completeness

Ensure both new stages are handled in:
- `IntermediateNodeExchangePaymentTransaction::run()`
- `ReceiverExchangePaymentTransaction::run()`

Unhandled stages would result in default case behavior (likely an error).

# Test Plan

**Complexity**: Moderate

This task modifies the core transaction flow and integrates new stages.

## Validation Approach

1. **Compilation Test**: Code must compile without warnings
2. **Code Review**: Verify integration points are correct
3. **Integration Testing**:
   - Test IntermediateNode transaction entering observing flow
   - Test Receiver transaction entering observing flow
   - Test stage transitions work correctly
   - Test that removed ObservingHandler call doesn't break anything

## Key Scenarios to Validate

| Scenario | Expected Behavior |
|----------|------------------|
| IntermediateNode: recovery exhausted | Transitions to Observing_AcceptClaim |
| Receiver: recovery exhausted | Transitions to Observing_AcceptClaim |
| Transaction awakens in Observing_AcceptClaim | Calls runObservingAcceptClaimStage() |
| Transaction awakens in Observing_GetClaimStatus | Calls runObservingGetClaimStatusStage() |
| Normal payment flow (no recovery) | No changes, works as before |

## Regression Testing

Verify that:
- Normal payment completion still works
- Recovery flow still works (before max attempts)
- Only max recovery exhaustion triggers observing flow

# Verification and Validation

## Architecture integrity
- Follows existing stage transition patterns
- Integrates cleanly with existing run() switch structure
- Maintains consistency between IntermediateNode and Receiver

## Security
- N/A - No security changes, just flow routing

## Performance
- No performance impact on normal flows
- Observing flow activated only after recovery exhaustion

## Scalability
- N/A - No scalability changes

## Reliability
- Observing flow provides reliable fallback for recovery failure
- Both transaction types handle new stages consistently

## Maintainability
- Clear stage-to-handler mapping in run() methods
- Centralized observing logic in base class
- Easy to add new stages if needed

## Cost
- N/A - No cost implications

## Compliance
- Follows existing code style and conventions
- Consistent with existing stage handling patterns

# Restrictions
- Commit changes only after successful compilation
- Do not modify CoordinatorExchangePaymentTransaction (out of scope per PRD)
- Ensure both IntermediateNode and Receiver are updated consistently
- Test normal payment flows are not affected
