# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Completed Payments Observer Monitoring
- **PRD ID**: 15
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.0
- **Date**: 2025-12-22
- **Author(s)**: Claude Code
- **Stakeholders**: Development Team
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-12-22
- **Previous PRD**: [14-async-observer-rpc-communication.md](14-async-observer-rpc-communication.md)
- **Related Documents**:
  - [Observer README](../../../../observer/README.md)
  - [PRD-14: Async Observer RPC Communication](14-async-observer-rpc-communication.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary

### Current project state
The vtcpd node has async RPC infrastructure for communicating with the observer (PRD-14). Payment transactions save their state to the database after successful completion (state = 1, Committed). However, when another node requests dispute resolution for a completed transaction through the observer, there is no mechanism to automatically respond with the required vote signatures.

### This iteration's focus
Implement a periodic monitoring transaction that:
1. Checks with the observer if any completed transactions have pending claim requests
2. Automatically submits vote signatures for transactions that are being observed
3. Logs observer responses and tracks submission status

### Connection to overall vision
This iteration completes the observer integration by ensuring that completed payments can be properly resolved even when other nodes request dispute resolution. This maintains network integrity and allows for proper claim verification.

## Iteration Context

### Previous Iterations Summary
- **Completed Features**:
  - PRD-12: Observer ambiguous transaction handling
  - PRD-13: Observer successful transactions monitoring
  - PRD-14: Async Observer RPC Communication infrastructure
- **Lessons Learned**: Async RPC pattern enables non-blocking observer communication
- **Technical Debt**: None identified

### Current State Analysis
- **What's working well**: Async RPC infrastructure from PRD-14, payment transaction state persistence
- **Pain points identified**: No mechanism to respond to observer claim requests for completed transactions
- **Performance metrics**: N/A (new feature)

## Problem Statement

### Background
When a payment transaction completes successfully, participants save their vote signatures to the database and update the transaction state to "Committed" (state = 1). If another participant later requests dispute resolution through the observer (by calling `AcceptClaim`), the observer needs vote signatures from all participants to verify the transaction.

Currently, there is no automated mechanism for nodes to:
1. Detect that their completed transactions have pending claim requests on the observer
2. Submit their vote signatures to help resolve the claims

### Problem Description
**Who is affected**: All vtcpd nodes participating in payment transactions

**When and where the problem occurs**: After a payment transaction completes, if any participant requests claim verification through the observer

**Impact of not solving this problem**:
- Claim requests cannot be properly verified
- Dispute resolution fails due to missing signatures
- Network integrity is compromised

### Success Metrics
- **Primary KPIs** (definition deferred): метрики не є предметом цієї ітерації; реалізація функціональності має дозволити подальше вимірювання
- **Operational constraints**:
  - Initial monitoring delay: 60 seconds after node start
  - Subsequent monitoring interval: 300 seconds
  - Maximum transactions per monitoring cycle: 100

## Project Scope

### This Iteration's Scope

#### New Features/Enhancements

1. **CompletedPaymentsObserverMonitoringTransaction** (`src/core/transactions/transactions/regular/payments/`)
   - Periodic transaction that monitors observer for claim requests
   - Uses async RPC infrastructure from PRD-14
   - State-based execution flow

2. **CompletedPaymentsMonitoringDelayedTask** (`src/core/delayed_tasks/`)
   - Periodic task launcher using boost::asio::steady_timer
   - Initial delay: 60 seconds (constant `kInitialDelaySeconds`)
   - Subsequent interval: 300 seconds (constant `kMonitoringIntervalSeconds`)
   - Emits signal to trigger transaction creation
   - TransactionsManager запускає транзакцію тільки якщо у TransactionsScheduler відсутній активний/запланований `CompletedPaymentsObserverMonitoringTransaction` (запобігання паралельним циклам)

3. **GetClaimStatusesRpcRequest Update**
   - Add `public_key` and `signature` fields to `GetClaimStatusesRpcRequest`
   - Add getters/setters for new fields
   - Serialization for signing is done in `CompletedPaymentsObserverMonitoringTransaction`

4. **Signing Serialization Methods in Transaction**
   - All signing serialization logic resides in `CompletedPaymentsObserverMonitoringTransaction`
   - New private method: `serializeGetClaimStatusesForSigning()` - serializes claims data for signing
   - New private method: `serializeSubmitClaimVotesForSigning()` - serializes votes data for signing
   - Use `Keystore::signPaymentTransaction()` for signing

#### Transaction State Machine

```
State 1: GetCurrentBlock
    |
    v
State 2: GetTransactionsForMonitoring
    |
    v
State 3: GetClaimStatuses (async RPC)
    |
    +-- Empty statuses array --> Done
    |
    v
State 4: ProcessClaimStatuses
    |
    +-- For each "observing" status:
    |       - Fetch signatures from PaymentParticipantsVotesHandler
    |       - Send SubmitClaimVotes (async RPC)
    |       - Store pending response in context
    |
    v
State 5: WaitForVotesSubmissionResponses
    |
    +-- All responses received --> Done
    |
    v
Done: Log results, transaction completes
```

#### Transaction Flow Details

**State 1: GetCurrentBlock**
- Send `GetBlockNumberRpcRequest` to observer
- Wait for response
- Store current block number in transaction context
- On error: log and finish transaction

**State 2: GetTransactionsForMonitoring**
- Call `storageHandler->paymentTransactionsHandler()->transactionsForObserverMonitoring(currentBlockNumber, kMaxTransactionsPerCycle)`
- If empty: finish transaction (no work to do)
- Store transactions list in context
- Retrieve own public key once via `ioTransaction->paymentKeysHandler()->getOwnPublicKey()`; on failure log and finish transaction
- Store public key in context for reuse across RPC calls

**State 3: GetClaimStatuses**
- Create `GetClaimStatusesRpcRequest` with:
  - List of (transactionUUID, maxClaimBlockNumber) pairs, відсортований за `transactionUUID` у лексикографічному порядку зростання (канонічний порядок для підпису)
  - Own public key from `ioTransaction->paymentKeysHandler()->getOwnPublicKey()`
  - Signature from `keystore->signPaymentTransaction(serializedData)`
- Serialization order for signing: claims size (uint32), claims array, public key
- Числові поля серіалізуються з тим самим порядком байтів, що в `BaseExchangePaymentTransaction::getSerializedReceipt`
- On signing error: log and finish transaction
- Send request, wait for response
- On empty statuses array: finish transaction (no claims pending)

**State 4: ProcessClaimStatuses**
- For each status with state == "observing":
  - Get votes from `paymentParticipantsVotesHandler->participantsSignatures(transactionUUID)`
  - Create `SubmitClaimVotesRpcRequest` with:
    - transactionUUID, maxClaimBlockNumber
    - votes map (PaymentNodeID -> Signature) відсортований за `PaymentNodeID` зростанням перед серіалізацією
    - Own public key
    - Signature (serialization: transactionUUID, maxClaimBlockNumber, votes size, votes, public key)
  - Send request
  - Store request ID in pending responses context
- For statuses in states "accepted", "rejected", "expired": log and skip submission
- Числові поля серіалізуються з тим самим порядком байтів, що в `BaseExchangePaymentTransaction::getSerializedReceipt`
- On signing error: log, skip this claim, continue with next

**State 5: WaitForVotesSubmissionResponses**
- Wait for all pending RPC responses
- For each response:
  - Log success/failure
  - Mark as received in context
- When all responses received: finish transaction
- RPC timeout/invalid response: log and mark failed; transaction завершується після обробки всіх відповідей

### Explicitly Out of Scope
- Retry logic for failed submissions (next monitoring cycle will retry)
- Observer failover/load balancing
- Real-time claim notification (uses polling approach)

### Dependencies from Previous Iterations
- PRD-14: Async RPC infrastructure (`ObserverRpcCommunicator`, request/response classes)
- Existing `PaymentTransactionsHandler::transactionsForObserverMonitoring()` method
- Existing `PaymentParticipantsVotesHandler::participantsSignatures()` method
- Existing `PaymentKeysHandler::getOwnPublicKey()` method
- Existing `Keystore::signPaymentTransaction()` method

### Future Roadmap Impact
This PRD completes the observer integration for payment verification.

## User Stories & Requirements

### User Personas

#### Primary User: vtcpd Node
- **Role**: Automated payment processing node
- **Goals**: Automatically respond to observer claim requests for completed payments
- **Pain Points**: Manual intervention required for claim verification
- **Technical Proficiency**: N/A (automated system)

### Functional Requirements

#### 1. CompletedPaymentsMonitoringDelayedTask Class

- **Description**: Periodic task that triggers monitoring transaction
- **Acceptance Criteria**:
  - Constructor takes: `as::io_context&`, `Logger&`
  - Uses `boost::asio::steady_timer` for scheduling
  - Class constants: `kInitialDelaySeconds = 60`, `kMonitoringIntervalSeconds = 300`
  - First run scheduled after `kInitialDelaySeconds`
  - Subsequent runs scheduled every `kMonitoringIntervalSeconds`
  - Emits `monitoringSignal()` on each timer expiration
  - Signal type: `signals::signal<void()>`

#### 2. CompletedPaymentsObserverMonitoringTransaction Class

- **Description**: Transaction that monitors and responds to observer claims
- **Acceptance Criteria**:
  - Inherits from `BaseTransaction`
  - Constructor takes: `StorageHandler*`, `Keystore*`, `Logger&`
  - Class constant: `kMaxTransactionsPerCycle = 100`
  - Uses async RPC via `outgoingRpcRequestSignal` and `resultWaitForRpcResponse()`
  - Implements state machine as described above
  - Logs all significant events (block number, transactions found, claims statuses, submission results, пропущені/завершені через помилки кроки)
  - Витягує власний публічний ключ один раз і зберігає в контексті; у разі помилки завершує транзакцію
  - При помилці підписання GetClaimStatuses завершує транзакцію; при помилці підписання SubmitClaimVotes пропускає конкретний claim і продовжує
  - **Private signing serialization methods**:
    - `pair<BytesShared, size_t> serializeGetClaimStatusesForSigning(const vector<pair<TransactionUUID, BlockNumber>>& claims, const PublicKey::Shared& publicKey)` - serializes claims data for signing
    - `pair<BytesShared, size_t> serializeSubmitClaimVotesForSigning(const TransactionUUID& transactionUUID, BlockNumber maxClaimBlockNumber, const map<PaymentNodeID, Signature::Shared>& votes, const PublicKey::Shared& publicKey)` - serializes votes data for signing

#### 3. GetClaimStatusesRpcRequest Update

- **Description**: Add public key and signature fields to GetClaimStatusesRpcRequest
- **Acceptance Criteria**:
  - `GetClaimStatusesRpcRequest` updated with `publicKey` and `signature` fields
  - Add methods: `publicKey()`, `signature()`, `setPublicKey()`, `setSignature()`
  - Update JSON serialization to include new fields
  - **Note**: Serialization for signing is done in transaction via `serializeGetClaimStatusesForSigning()`

#### 4. Signing Serialization in Transaction

- **Description**: Transaction methods for serializing data before signing
- **Acceptance Criteria for `serializeGetClaimStatusesForSigning()`**:
  - Serialization order:
    1. `uint32_t` claims count
    2. For each claim (sorted за `transactionUUID` у лексикографічному порядку зростання): `TransactionUUID` (16 bytes), `BlockNumber` (8 bytes)
    3. Public key bytes
  - Використовує порядок байтів як у `BaseExchangePaymentTransaction::getSerializedReceipt`
  - Returns `pair<BytesShared, size_t>` with serialized data and size
- **Acceptance Criteria for `serializeSubmitClaimVotesForSigning()`**:
  - Serialization order:
    1. `TransactionUUID` (16 bytes)
    2. `BlockNumber` (8 bytes)
    3. `uint32_t` votes count
    4. For each vote (sorted за `PaymentNodeID` зростанням): `PaymentNodeID` (2 bytes), signature bytes
    5. Public key bytes
  - Використовує порядок байтів як у `BaseExchangePaymentTransaction::getSerializedReceipt`
  - Returns `pair<BytesShared, size_t>` with serialized data and size
- **Signing workflow**:
  - Transaction obtains public key via `ioTransaction->paymentKeysHandler()->getOwnPublicKey()`
  - Transaction calls serialization method
  - Transaction signs via `keystore->signPaymentTransaction(ioTransaction, serializedData, dataSize)`
  - On GetClaimStatuses signing error: log і завершити транзакцію
  - On SubmitClaimVotes signing error: log, skip claim, continue

#### 5. Core Integration

- **Description**: Wire up delayed task and transaction creation
- **Acceptance Criteria**:
  - Core creates `CompletedPaymentsMonitoringDelayedTask` instance
  - New method `Core::initCompletedPaymentsMonitoringDelayedTask()`
  - Delayed task signal connected to TransactionsManager
  - TransactionsManager creates transaction on signal

#### 6. TransactionsManager Integration

- **Description**: Handle monitoring signal and create transaction
- **Acceptance Criteria**:
  - New method `subscribeForCompletedPaymentsMonitoringSignal()`
  - New slot `onCompletedPaymentsMonitoringSlot()`
  - New method `launchCompletedPaymentsObserverMonitoringTransaction()`
  - Proper signal/slot subscription during initialization
  - Перед створенням транзакції перевіряє через `TransactionsScheduler`, що немає активної/запланованої транзакції типу `CompletedPaymentsObserverMonitoringTransaction`; якщо є — лог і вихід без створення

### Non-Functional Requirements

#### Performance
- Monitoring must not block the event loop (async RPC)
- Maximum 100 transactions processed per cycle
- Timer operations use steady_timer (not affected by system clock changes)

#### Reliability
- Observer unavailability gracefully handled (log and finish)
- RPC errors logged with details
- Next monitoring cycle retries failed operations

#### Maintainability
- Constants defined at class level
- Clear state machine implementation
- Consistent with existing transaction patterns

## Technical Specifications

### Architecture Evolution

#### Current Architecture
```
Delayed Task -> Signal -> TransactionsManager -> Transaction
                                                      |
                                                      v
                                               [Sync operations]
```

#### Proposed Architecture
```
CompletedPaymentsMonitoringDelayedTask
    |
    v
monitoringSignal
    |
    v
TransactionsManager::onCompletedPaymentsMonitoringSlot
    |
    v
CompletedPaymentsObserverMonitoringTransaction
    |
    +-- GetBlockNumber RPC (async)
    +-- GetClaimStatuses RPC (async, signed)
    +-- SubmitClaimVotes RPC (async, signed, multiple)
    |
    v
Observer responses via RPC infrastructure
```

### File Structure

```
src/core/delayed_tasks/
├── CompletedPaymentsMonitoringDelayedTask.h
├── CompletedPaymentsMonitoringDelayedTask.cpp
└── CMakeLists.txt (update)

src/core/transactions/transactions/regular/payments/
├── CompletedPaymentsObserverMonitoringTransaction.h
└── CompletedPaymentsObserverMonitoringTransaction.cpp

src/core/network/rpc/requests/
├── GetClaimStatusesRpcRequest.h (update - add signature fields)
└── GetClaimStatusesRpcRequest.cpp (update)
```

### CMake Updates

1. Update `src/core/delayed_tasks/CMakeLists.txt`:
   - Add `CompletedPaymentsMonitoringDelayedTask.cpp`

2. Update payments transaction CMakeLists.txt:
   - Add `CompletedPaymentsObserverMonitoringTransaction.cpp`

### Technology Stack
- **Async Timer**: boost::asio::steady_timer
- **Signals**: boost::signals2
- **Async RPC**: Existing infrastructure from PRD-14
- **Cryptography**: sphincs (existing Keystore integration)

## Testing Strategy

### Unit Tests

Location: `tests/unit/`

#### 1. CompletedPaymentsMonitoringDelayedTaskTest.cpp

- **Test: Constructor initializes timer correctly**
  - Verify timer is created
  - Verify initial delay is `kInitialDelaySeconds`

- **Test: Signal emitted on timer expiration**
  - Mock io_context with controlled time advancement
  - Verify signal is emitted after initial delay
  - Verify subsequent emissions at `kMonitoringIntervalSeconds` intervals

- **Test: Constants have expected values**
  - `kInitialDelaySeconds == 60`
  - `kMonitoringIntervalSeconds == 300`

#### 2. CompletedPaymentsObserverMonitoringTransactionTest.cpp

- **Test: Constructor stores dependencies correctly**
  - Verify StorageHandler, Keystore, Logger stored

- **Test: Transaction type is correct**
  - Verify transaction type is `TransactionType::Payments_CompletedPaymentsObserverMonitoring`

- **Test: kMaxTransactionsPerCycle constant**
  - Verify `kMaxTransactionsPerCycle == 100`

- **Test: serializeGetClaimStatusesForSigning serialization order**
  - Create known claims data
  - Call serialization method
  - Verify output matches expected byte sequence
  - Verify claims are sorted by transactionUUID
  - Order: claims count (uint32), claims array, public key

- **Test: serializeSubmitClaimVotesForSigning serialization order**
  - Create known votes data
  - Call serialization method
  - Verify output matches expected byte sequence
  - Verify votes are sorted by PaymentNodeID
  - Order: transactionUUID, blockNumber, votes count, votes, public key

- **Test: Empty transactions list finishes immediately**
  - Mock `transactionsForObserverMonitoring` to return empty
  - Verify transaction completes without RPC calls

- **Test: Empty claim statuses finishes transaction**
  - Mock `GetClaimStatuses` response with empty array
  - Verify transaction completes without SubmitClaimVotes calls

#### 3. GetClaimStatusesRpcRequestTest.cpp

- **Test: Public key getter/setter**
  - Set public key via setter
  - Verify `publicKey()` returns stored value

- **Test: Signature getter/setter**
  - Set signature via setter
  - Verify `signature()` returns stored value

- **Test: JSON serialization includes public_key and signature**
  - Create request with public key and signature
  - Verify JSON output contains both fields

### Test Files to Add

```cmake
# tests/unit/CMakeLists.txt additions:
delayed_tasks/CompletedPaymentsMonitoringDelayedTaskTest.cpp
transactions/CompletedPaymentsObserverMonitoringTransactionTest.cpp
network/rpc/GetClaimStatusesRpcRequestTest.cpp
```

### Quality Gates
- All unit tests pass
- Code compiles without warnings
- Follows existing code style
- Constants properly defined

## Risk Management

### Technical Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| Observer unavailable | Low | Medium | Log error and finish; next cycle retries |
| Signing failure | Medium | Low | Use existing proven Keystore methods |
| RPC timeout | Low | Medium | Standard timeout handling from PRD-14 |
| Multiple simultaneous monitoring | Low | Low | Single delayed task instance ensures sequential execution |

## Appendices

### Glossary
- **Claim**: Request to observer for transaction verification
- **Vote**: Participant's signature confirming transaction validity
- **Committed**: Transaction state (1) indicating successful local completion
- **Observing**: Claim state indicating active monitoring by observer

### References
- [Observer RPC Protocol](../../../../observer/README.md)
- [PRD-14: Async Observer RPC Communication](14-async-observer-rpc-communication.md)
- [Keystore API](../../../../src/core/crypto/keychain.h)

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-12-22 | Claude Code | Initial draft | Phase 1 |

**Related Documents**
- **Previous PRD**: [14-async-observer-rpc-communication.md](14-async-observer-rpc-communication.md)
- **Observer README**: [observer/README.md](../../../../observer/README.md)
