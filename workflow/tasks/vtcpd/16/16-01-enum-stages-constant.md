# 16-01 - Add PaymentObservingState::Conflicted and Transaction Stages

# Links
- [PRD](../../prd/vtcpd/16-payment-transaction-observing-states.md)

# Description

This task adds the foundational infrastructure for payment transaction observing states:

1. **New PaymentObservingState enum value**: Add `Conflicted = 4` to the `PaymentObservingState` enum. This state is used when `AcceptClaim` is rejected because `max_claim_block_number` is less than the current block number (claim window already closed).

2. **New Transaction Stages**: Add two new stages to the `Stages` enum in `BaseExchangePaymentTransaction`:
   - `Observing_AcceptClaim` - state for submitting claim to observer
   - `Observing_GetClaimStatus` - state for polling claim status

3. **New Constant**: Add `kObservingCheckPeriodMilliseconds = 60000` for delay between observing polls.

This task provides the base infrastructure that subsequent tasks will build upon.

# Requirements and DOD

## Requirements

1. **PaymentObservingState::Conflicted**
   - Add `Conflicted = 4` to `PaymentObservingState` enum in `src/core/io/storage/interfaces/PaymentTransactionsHandler.h`
   - Value must be 4 (follows existing sequence: Init=0, Committed=1, ParticipantsVotesPresent=2, RejectedByObserving=3)
   - Add documentation comment explaining when this state is used

2. **New Transaction Stages**
   - Add `Observing_AcceptClaim` to `Stages` enum in `BaseExchangePaymentTransaction.h`
   - Add `Observing_GetClaimStatus` to `Stages` enum in `BaseExchangePaymentTransaction.h`
   - Place these stages after existing stages in the enum

3. **Observing Check Period Constant**
   - Add `static const uint32_t kObservingCheckPeriodMilliseconds = 60000` to `BaseExchangePaymentTransaction.h`
   - Place in the protected constants section alongside existing constants like `kWaitMillisecondsToTryRecoverAgain`

## Definition of Done

- [ ] `PaymentObservingState::Conflicted = 4` added with documentation comment
- [ ] `Stages::Observing_AcceptClaim` added to enum
- [ ] `Stages::Observing_GetClaimStatus` added to enum
- [ ] `kObservingCheckPeriodMilliseconds = 60000` constant added
- [ ] Code compiles without warnings
- [ ] No existing functionality is broken

# Implementation Plan

## Step 1: Modify PaymentTransactionsHandler.h

File: `src/core/io/storage/interfaces/PaymentTransactionsHandler.h`

Add new enum value after `RejectedByObserving`:

```cpp
enum class PaymentObservingState : int
{
    Init = 0,
    Committed = 1,
    ParticipantsVotesPresent = 2,
    RejectedByObserving = 3,

    /**
     * Transaction reached Conflicted state when AcceptClaim was rejected
     * because max_claim_block_number is less than current block number
     * (claim window already closed). Trust lines are marked for manual resolution.
     */
    Conflicted = 4
};
```

## Step 2: Modify BaseExchangePaymentTransaction.h

File: `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.h`

### 2.1 Add new stages to Stages enum

Add after `Common_Uncertain` (or after the last existing stage):

```cpp
enum Stages
{
    // ... existing stages ...
    Common_Uncertain,

    Observing_AcceptClaim,
    Observing_GetClaimStatus
};
```

### 2.2 Add constant

Add in the protected constants section (around line 250-260):

```cpp
static const uint32_t kObservingCheckPeriodMilliseconds = 60000;
```

## Step 3: Verify Compilation

Run build to ensure no compilation errors:

```bash
make -j$(nproc)
```

# Test Plan

**Complexity**: Simple

This task only adds enum values and a constant. No runtime behavior changes.

## Validation Approach

1. **Compilation Test**: Code must compile without warnings
2. **Static Analysis**: Enum values must be unique and in correct sequence
3. **Unit Test**: See Task 16-06 for unit test implementation

## Manual Verification

After implementation, verify:
- `PaymentObservingState::Conflicted` has integer value 4
- `Stages::Observing_AcceptClaim` and `Stages::Observing_GetClaimStatus` are present
- Constant `kObservingCheckPeriodMilliseconds` equals 60000

# Verification and Validation

## Architecture integrity
- Follows existing enum extension patterns in the codebase
- Stages enum extension is consistent with existing stage definitions
- Constant naming follows project conventions (kCamelCase)

## Security
- N/A - No security implications for adding enum values and constants

## Performance
- N/A - No runtime performance impact

## Scalability
- N/A - No scalability implications

## Reliability
- Enum values are explicit (not auto-incremented) ensuring stability across builds

## Maintainability
- Documentation comments explain purpose of new enum value
- Constant is centralized in base class for easy modification

## Cost
- N/A - No cost implications

## Compliance
- Follows existing code style and conventions

# Restrictions
- Commit changes only after successful compilation
- Do not modify any existing enum values or constants
