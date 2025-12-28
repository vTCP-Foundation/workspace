# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Payment Transaction Observing States
- **PRD ID**: 16
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.0
- **Date**: 2025-12-24
- **Author(s)**: Claude Code
- **Stakeholders**: Development Team
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-12-24
- **Previous PRD**: [15-completed-payments-observer-monitoring.md](15-completed-payments-observer-monitoring.md)
- **Related Documents**:
  - [Observer README](../../../../observer/README.md)
  - [PRD-14: Async Observer RPC Communication](14-async-observer-rpc-communication.md)
  - [PRD-15: Completed Payments Observer Monitoring](15-completed-payments-observer-monitoring.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary

### Current project state
Payment transactions (IntermediateNodeExchangePaymentTransaction, ReceiverExchangePaymentTransaction) have a `Stages::Common_Recovery` state where `runVotesRecoveryParentStage()` attempts to recover final signatures from other participants. After exhausting recovery attempts, the transaction delegates observing to `ObservingHandler` via `mObservingHandler->addPaymentClaim()` and terminates. The `ObservingHandler` then separately monitors the claim and, upon finalization, must deserialize the transaction and inject context for proper completion.

### This iteration's focus
Move the observing logic directly into the payment transaction by adding new states:
1. **Observing_AcceptClaim**: Submit claim to observer and wait for response
2. **Observing_GetClaimStatus**: Poll observer for claim status until finalization

This eliminates the need for external transaction deserialization and provides a cleaner, more maintainable flow where the transaction handles its own observing lifecycle.

### Connection to overall vision
This iteration completes the observer integration for payment transactions by making them self-sufficient in handling the observing process, utilizing the async RPC infrastructure from PRD-14.

## Iteration Context

### Previous Iterations Summary
- **Completed Features**:
  - PRD-14: Async Observer RPC Communication infrastructure
  - PRD-15: Completed Payments Observer Monitoring (periodic monitoring transaction)
  - `AcceptClaimRpcRequest` and `GetClaimStatusRpcRequest` already implemented
- **Lessons Learned**: Async RPC pattern enables non-blocking observer communication
- **Technical Debt**: Current observing delegation to ObservingHandler requires complex transaction deserialization for completion

### Current State Analysis
- **What's working well**: Async RPC infrastructure, recovery mechanism for collecting votes from participants
- **Pain points identified**: ObservingHandler must deserialize transactions and inject context, which is error-prone and hard to maintain
- **Performance metrics**: N/A

## Problem Statement

### Background
When a payment transaction enters the recovery stage (`Stages::Common_Recovery`), it attempts to collect final signatures from other participants. If this fails after `kMaxRecoveryAttempts` (30 attempts), the transaction currently:
1. Calls `mObservingHandler->addPaymentClaim()` to register the claim
2. Returns `resultDone()` to terminate

The `ObservingHandler` then independently monitors the claim status. When the claim is finalized (approved or rejected), `ObservingHandler` must:
1. Deserialize the original transaction
2. Inject the observer's response data
3. Resume the transaction for proper completion

### Problem Description
**Who is affected**: IntermediateNodeExchangePaymentTransaction and ReceiverExchangePaymentTransaction

**When and where the problem occurs**: After exhausting recovery attempts in `processNextNodeToCheckVotes()` method

**Impact of not solving this problem**:
- Complex deserialization logic in ObservingHandler
- Transaction context must be reconstructed externally
- Difficult to maintain and extend
- Potential for inconsistent state handling

### Success Metrics
- **Primary KPIs**:
  - Payment transactions handle their own observing lifecycle without external intervention
  - All observing responses (approved, rejected, not found) are properly handled within the transaction
- **Secondary KPIs**:
  - Clean state machine implementation
  - Consistent with existing async RPC patterns

## Project Scope

### This Iteration's Scope

#### New Features/Enhancements

1. **New PaymentObservingState: Conflicted**
   - Add `Conflicted = 4` to `PaymentObservingState` enum in `PaymentTransactionsHandler.h`
   - Used when `AcceptClaim` is rejected because `max_claim_block_number` is less than the current block number (claim window already closed)

2. **New Transaction Stages in BaseExchangePaymentTransaction**
   - `Observing_AcceptClaim` - state for submitting claim to observer
   - `Observing_GetClaimStatus` - state for polling claim status

3. **New Methods in BaseExchangePaymentTransaction**
   - `runObservingAcceptClaimStage()` - handles AcceptClaim RPC request/response
   - `runObservingGetClaimStatusStage()` - handles GetClaimStatus RPC request/response
   - `setTrustLinesToConflictState()` - sets all trust lines with reservations to Conflict state

4. **New Constant in BaseExchangePaymentTransaction**
   - `kObservingCheckPeriodMilliseconds = 60000` - delay between GetClaimStatus polling attempts

5. **Modifications to processNextNodeToCheckVotes()**
   - Instead of calling `mObservingHandler->addPaymentClaim()` and `resultDone()`
   - Transition to `Observing_AcceptClaim` stage

6. **Modifications to IntermediateNodeExchangePaymentTransaction::run()**
   - Add case handlers for new observing stages

7. **Modifications to ReceiverExchangePaymentTransaction::run()**
   - Add case handlers for new observing stages

#### State Machine Details

##### Observer RPC Response Fields (from observer/README.md)

**AcceptClaim response**
- `result.success` (bool)
- `result.message` (string)
  - `"claim accepted successfully"` on success
  - `"claim already exists ..."` when duplicate
  - `"max_claim_block_number (...) must be less than current block number (...)"` when claim window is closed

**GetClaimStatus response**
- `result.state` (string): `"observing"`, `"approved"`, `"rejected"`, `"not found"`
- `result.votes` (array): populated only when `state == "approved"`
- `result.signature` (string): `"REJECTED_BY_TIMEOUT"` when `state == "rejected"`, empty otherwise

**Observer guarantees**: for existing claims, the state eventually becomes `"approved"` or `"rejected"`. `"not found"` means the claim is missing and must be re-submitted.

##### State: Observing_AcceptClaim

**Entry**: Transition from `processNextNodeToCheckVotes()` after exhausting recovery attempts

**Actions**:
1. Create `AcceptClaimRpcRequest` with:
   - `mTransactionUUID`
   - `mMaximalClaimingBlockNumber`
   - `mParticipantsPublicKeys`
   - `mPublicKey` (own public key)
   - `mSignedTransaction` (own signature)
2. Send request via `sendRpcRequest()`
3. Return `resultWaitForRpcResponse(RpcMethod::AcceptClaim, kObservingCheckPeriodMilliseconds)`

**On Response**:
- **Success (`result.success=true`)**: Transition to `Observing_GetClaimStatus`
- **Duplicate (`result.success=false` and `result.message` contains "claim already exists")**: Transition to `Observing_GetClaimStatus`
- **Expired window (`result.success=false` and `result.message` contains "must be less than current block number")**:
  1. Call `setTrustLinesToConflictState()` to set all trust lines with reservations to `TrustLineState::Conflict`
  2. Update transaction state via `paymentTransactionsHandler()->updateTransactionState(mTransactionUUID, PaymentObservingState::Conflicted)`
  3. Do not call `rollBack()`; reservations remain for manual resolution
  4. Return `resultDone()`
- **Network error / timeout / other error**:
  1. Log the error
  2. Return `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)` to retry

##### State: Observing_GetClaimStatus

**Entry**: Transition from `Observing_AcceptClaim` on success or "claim already exists"

**Actions**:
1. Create `GetClaimStatusRpcRequest` with:
   - `mTransactionUUID`
   - `mMaximalClaimingBlockNumber`
2. Send request via `sendRpcRequest()`
3. Return `resultWaitForRpcResponse(RpcMethod::GetClaimStatus, kObservingCheckPeriodMilliseconds)`

**On Response**:
- **Status `"not found"`**: Transition back to `Observing_AcceptClaim`
- **Status `"observing"`**: Return `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)` to retry polling
- **Status `"approved"`**:
  1. Extract votes from response (`mParticipantsSignatures = response->votes()`)
  2. Call `processParticipantsVotesMessage()` to validate and finalize
  3. If validation fails: log error and return `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)` to retry
  4. If validation succeeds: transaction finalizes via `approve()`
- **Status `"rejected"`**:
  1. Update transaction state via `paymentTransactionsHandler()->updateTransactionState(mTransactionUUID, PaymentObservingState::RejectedByObserving)`
  2. Call `rollBack()` to release reservations
  3. Return `resultDone()`
- **Network error / timeout / parse error**:
  1. Log the error
  2. Return `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)` to retry

##### Method: setTrustLinesToConflictState()

**Purpose**: Set all trust lines that have reservations in this transaction to Conflict state

**Implementation**:
```cpp
void BaseExchangePaymentTransaction::setTrustLinesToConflictState()
{
    for (const auto &nodeAndReservations : mReservations) {
        for (const auto &pathIDAndReservation : nodeAndReservations.second) {
            auto trustLineManager = trustLinesManager(pathIDAndReservation.second->equivalent());
            trustLineManager->setTrustLineState(
                nodeAndReservations.first,
                TrustLineState::Conflict);
        }
    }
}
```

#### Flow Diagram

```
processNextNodeToCheckVotes()
    |
    +-- mCountRecoveryAttempts >= kMaxRecoveryAttempts
    |
    v
Observing_AcceptClaim
    |
    +-- Send AcceptClaimRpcRequest
    |
    +-- Response: success=true OR "claim already exists"
    |       |
    |       v
    |   Observing_GetClaimStatus
    |       |
    |       +-- Send GetClaimStatusRpcRequest
    |       |
    |       +-- Response: "not found" --> Observing_AcceptClaim
    |       |
    |       +-- Response: "observing" --> sleep, retry GetClaimStatus
    |       |
    |       +-- Response: "approved" --> validate votes --> approve()
    |       |
    |       +-- Response: "rejected" --> updateState(RejectedByObserving) --> rollBack() --> done
    |       |
    |       +-- Error --> log, sleep, retry GetClaimStatus
    |
    +-- Response: "must be less than current block number"
    |       |
    |       v
    |   setTrustLinesToConflictState()
    |   updateState(Conflicted)
    |   resultDone()
    |
    +-- Error --> log, sleep, retry AcceptClaim
```

### Explicitly Out of Scope
- Modifications to CoordinatorExchangePaymentTransaction (coordinator always has all signatures when signing)
- Changes to ObservingHandler (will be deprecated in future iterations)
- Maximum retry limits for observing (regulated by observer's "must be less than current block number" response)
- Schema/serialization updates for new `Stages` on transaction restart/resume (deferred to a separate PRD)

### Dependencies from Previous Iterations
- PRD-14: Async RPC infrastructure (`sendRpcRequest()`, `resultWaitForRpcResponse()`, RPC request/response classes)
- Existing `AcceptClaimRpcRequest` and `AcceptClaimRpcResponse`
- Existing `GetClaimStatusRpcRequest` and `GetClaimStatusRpcResponse`
- Existing `processParticipantsVotesMessage()` for vote validation

### Future Roadmap Impact
This PRD enables deprecation of the transaction deserialization logic in ObservingHandler in future iterations.

## User Stories & Requirements

### User Personas

#### Primary User: vtcpd Node
- **Role**: Automated payment processing node
- **Goals**: Complete payment transactions even when direct participant communication fails
- **Pain Points**: Complex external observing logic
- **Technical Proficiency**: N/A (automated system)

### Functional Requirements

#### 1. PaymentObservingState Extension

- **Description**: Add Conflicted state to PaymentObservingState enum
- **Acceptance Criteria**:
  - New enum value `Conflicted = 4`
  - Located in `src/core/io/storage/interfaces/PaymentTransactionsHandler.h`
  - Used when `AcceptClaim` returns `result.success=false` with `result.message` containing "must be less than current block number"

#### 2. New Transaction Stages

- **Description**: Add observing stages to Stages enum in BaseExchangePaymentTransaction
- **Acceptance Criteria**:
  - New stage `Observing_AcceptClaim`
  - New stage `Observing_GetClaimStatus`
  - Added after existing stages in the enum

#### 3. Observing Check Period Constant

- **Description**: Constant for delay between observing polls
- **Acceptance Criteria**:
  - `static const uint32_t kObservingCheckPeriodMilliseconds = 60000`
  - Located in BaseExchangePaymentTransaction

#### 4. runObservingAcceptClaimStage() Method

- **Description**: Handle AcceptClaim RPC communication
- **Acceptance Criteria**:
  - Creates AcceptClaimRpcRequest with transaction data
  - Sends request via `sendRpcRequest()`
  - On first call (no response yet): returns `resultWaitForRpcResponse()`
  - Uses `result.success` and `result.message` from `AcceptClaim` response (per observer/README.md)
  - On `result.success=true`: transitions to `Observing_GetClaimStatus`
  - On `result.success=false` with "claim already exists": transitions to `Observing_GetClaimStatus`
  - On `result.success=false` with "must be less than current block number": calls `setTrustLinesToConflictState()`, updates state to Conflicted, returns `resultDone()` without calling `rollBack()`
  - On error: logs and returns `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)`

#### 5. runObservingGetClaimStatusStage() Method

- **Description**: Handle GetClaimStatus RPC communication and claim finalization
- **Acceptance Criteria**:
  - Creates GetClaimStatusRpcRequest with transaction UUID and block number
  - Sends request via `sendRpcRequest()`
  - Uses `result.state`, `result.votes`, and `result.signature` from `GetClaimStatus` response (per observer/README.md)
  - On `"not found"`: transitions to `Observing_AcceptClaim`
  - On `"observing"`: returns `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)`
  - On `"approved"`: extracts votes, calls `processParticipantsVotesMessage()`, finalizes via `approve()` if valid
  - On `"rejected"`: updates state to `RejectedByObserving`, calls `rollBack()`, returns `resultDone()`
  - On validation failure or error: logs and returns `resultAwakeAfterMilliseconds(kObservingCheckPeriodMilliseconds)`

#### 6. setTrustLinesToConflictState() Method

- **Description**: Set trust lines with reservations to Conflict state
- **Acceptance Criteria**:
  - Iterates over `mReservations`
  - For each reservation, gets the trust line manager for the reservation's equivalent
  - Sets trust line state to `TrustLineState::Conflict` for the contractor

#### 7. processNextNodeToCheckVotes() Modification

- **Description**: Transition to observing stages instead of delegating to ObservingHandler
- **Acceptance Criteria**:
  - Remove call to `mObservingHandler->addPaymentClaim()`
  - Instead, set `mStep = Stages::Observing_AcceptClaim`
  - Return `runObservingAcceptClaimStage()`

#### 8. IntermediateNodeExchangePaymentTransaction::run() Extension

- **Description**: Add case handlers for new observing stages
- **Acceptance Criteria**:
  - Case `Stages::Observing_AcceptClaim`: return `runObservingAcceptClaimStage()`
  - Case `Stages::Observing_GetClaimStatus`: return `runObservingGetClaimStatusStage()`

#### 9. ReceiverExchangePaymentTransaction::run() Extension

- **Description**: Add case handlers for new observing stages
- **Acceptance Criteria**:
  - Case `Stages::Observing_AcceptClaim`: return `runObservingAcceptClaimStage()`
  - Case `Stages::Observing_GetClaimStatus`: return `runObservingGetClaimStatusStage()`

### Non-Functional Requirements

#### Performance
- Observing operations must not block the event loop (async RPC)
- Polling interval of 60 seconds balances responsiveness with observer load

#### Reliability
- All error conditions result in retry after delay
- Only the `AcceptClaim` validation message "must be less than current block number" causes permanent failure with Conflict state
- Transaction state is always updated before completion

#### Maintainability
- Observing logic is contained within the transaction
- No external deserialization required
- Consistent with existing async RPC patterns

## Technical Specifications

### Architecture Evolution

#### Current Architecture
```
Payment Transaction (recovery failed)
    |
    v
mObservingHandler->addPaymentClaim()
    |
    v
resultDone() [transaction terminated]

    ... later ...

ObservingHandler monitors claim
    |
    v
On finalization: deserialize transaction, inject context, resume
```

#### Proposed Architecture
```
Payment Transaction (recovery failed)
    |
    v
Observing_AcceptClaim [send claim to observer]
    |
    +-- success --> Observing_GetClaimStatus [poll for result]
    |                   |
    |                   +-- approved --> processParticipantsVotesMessage() --> approve()
    |                   |
    |                   +-- rejected --> rollBack() --> done
    |                   |
    |                   +-- observing --> sleep, retry
    |
    +-- AcceptClaim rejected: "must be less than current block number" --> setTrustLinesToConflictState() --> done
```

### File Structure

```
src/core/io/storage/interfaces/
├── PaymentTransactionsHandler.h  [modify: add Conflicted enum value]

src/core/transactions/transactions/regular/payments/base/
├── BaseExchangePaymentTransaction.h  [modify: add stages, constants, method declarations]
├── BaseExchangePaymentTransaction.cpp  [modify: add method implementations]

src/core/transactions/transactions/regular/payments/
├── IntermediateNodeExchangePaymentTransaction.cpp  [modify: add case handlers in run()]
├── ReceiverExchangePaymentTransaction.cpp  [modify: add case handlers in run()]
```

### Technology Stack
- **Async RPC**: Existing infrastructure from PRD-14
- **State Machine**: Existing transaction stage mechanism

## Testing Strategy

### Unit Tests

Location: `tests/unit/`

#### 1. PaymentObservingStateTest.cpp

- **Test: Conflicted enum value equals 4**
  - Verify `PaymentObservingState::Conflicted` has value 4
  - Verify all enum values are distinct (Init=0, Committed=1, ParticipantsVotesPresent=2, RejectedByObserving=3, Conflicted=4)

### Test Files to Add

```cmake
# tests/unit/CMakeLists.txt additions:
sqlite/PaymentObservingStateTest.cpp
```

### Quality Gates
- All unit tests pass
- Code compiles without warnings
- Follows existing code style
- State machine transitions are correct

## Risk Management

### Technical Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| RPC timeout during observing | Low | Medium | Automatic retry after delay |
| Observer unavailable | Low | Low | Retry mechanism with configurable delay |
| Votes validation failure from observer | Low | Low | Retry GetClaimStatus to get correct data |
| Trust line conflict state handling | Medium | Low | Use existing TrustLinesManager API |

## Appendices

### Glossary
- **Claim**: Request to observer for transaction verification
- **Vote**: Participant's signature confirming transaction validity
- **Observing**: Process of monitoring claim status on the observer
- **Conflicted**: State when `AcceptClaim` is rejected because the claim window is already closed; trust lines are marked for manual resolution

### References
- [Observer RPC Protocol](../../../../observer/README.md)
- [PRD-14: Async Observer RPC Communication](14-async-observer-rpc-communication.md)
- [PRD-15: Completed Payments Observer Monitoring](15-completed-payments-observer-monitoring.md)

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-12-24 | Claude Code | Initial draft | Phase 1 |

**Related Documents**
- **Previous PRD**: [15-completed-payments-observer-monitoring.md](15-completed-payments-observer-monitoring.md)
- **Observer README**: [observer/README.md](../../../../observer/README.md)
