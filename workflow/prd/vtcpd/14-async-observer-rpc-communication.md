# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Async Observer RPC Communication
- **PRD ID**: 14
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.1
- **Date**: 2025-12-17
- **Author(s)**: Claude Code
- **Stakeholders**: Development Team
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-12-17
- **Previous PRD**: [13-observer-successful-transactions-monitoring.md](13-observer-successful-transactions-monitoring.md)
- **Related Documents**:
  - [Architecture Plan](../../../../docs/async-rpc-implementation-plan.md)
  - [Observer README](../../../../observer/README.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary

### Current project state
The vtcpd node currently communicates with the observer via synchronous RPC calls implemented in `ObserverRPCClient`. These blocking calls halt the entire event loop while waiting for observer responses, which can significantly impact node performance and responsiveness.

### This iteration's focus
Implement an asynchronous RPC communication layer for observer interaction, following the same architectural patterns used by the `Communicator` class for UDP messaging. This will allow transactions to initiate RPC requests and "sleep" while waiting for responses, with the event loop remaining free to process other operations.

### Connection to overall vision
This iteration establishes the async RPC infrastructure that will be used by payment transactions to communicate with the observer without blocking. Future PRDs will migrate specific transaction types to use this new async mechanism.

## Iteration Context

### Previous Iterations Summary
- **Completed Features**:
  - PRD-12: Observer ambiguous transaction handling
  - PRD-13: Observer successful transactions monitoring
  - Synchronous `ObserverRPCClient` implementation
- **Lessons Learned**: Synchronous RPC calls block the event loop and affect overall node performance
- **Technical Debt**: Current `ObserverRPCClient` uses blocking `boost::asio::connect()`, `write()`, and `read_until()` operations

### Current State Analysis
- **What's working well**: The signal-based async pattern for UDP messaging via Communicator
- **Pain points identified**: RPC calls to observer block the event loop for the duration of the request
- **Performance metrics**: Each RPC call can block for up to 10 seconds on timeout

## Problem Statement

### Background
The vtcpd node uses a single-threaded event loop architecture based on `boost::asio`. Network communication with other nodes via UDP is handled asynchronously through the `Communicator` class, which uses signals to decouple message sending/receiving from transaction processing. However, communication with the observer via JSON-RPC over TCP is currently synchronous.

### Problem Description
**Who is affected**: All transactions that need to communicate with the observer (payment transactions, claim verification)

**When and where the problem occurs**: Every time a transaction calls methods on `ObserverRPCClient`:
- `getBlockNumber()` - blocks during TCP connect, write, read
- `acceptClaim()` - blocks during entire RPC round-trip
- `getClaimStatus()` - blocks during entire RPC round-trip
- `submitClaimVotes()` - blocks during entire RPC round-trip
- `getClaimStatuses()` - blocks during entire RPC round-trip

**Impact of not solving this problem**:
- Node becomes unresponsive during observer communication
- Other transactions cannot progress while waiting for RPC responses
- Network message processing is delayed
- Overall throughput is reduced

### Success Metrics
- **Primary KPIs**:
  - Event loop never blocks during observer RPC communication
  - Transactions can sleep while waiting for RPC responses
  - All existing RPC functionality preserved
- **Secondary KPIs**:
  - Clean signal-based integration with existing architecture
  - Type-safe request/response classes
- **Target Values**:
  - RPC timeout: 10 seconds maximum, enforced via класової константи в `AsyncRpcSession`
  - Zero blocking calls in the main event loop

## Project Scope

### This Iteration's Scope

#### New Features/Enhancements

1. **Async RPC Infrastructure** (`src/core/network/rpc/`)
   - Base `RpcRequest` and `RpcResponse` classes
   - `RpcMethod` enum for all supported methods
   - `RpcResponseStatus` enum (Success, Timeout, NetworkError, ParseError, RpcError)
   - Typed request/response classes for each RPC method

2. **AsyncRpcSession Class**
   - Manages single async TCP connection to observer
   - Implements async_resolve -> async_connect -> async_write -> async_read chain
   - 10-second timeout timer
   - Completion callback with typed RpcResponse

3. **ObserverRpcCommunicator Class**
   - Main entry point for sending RPC requests
   - Manages observer address configuration
   - Handles JSON serialization/deserialization internally
   - Emits `rpcResponseSignal` when response received

4. **TransactionState Extensions**
   - New `waitForRpcResponse(RpcMethod, timeout)` method
   - New `mRequiredRpcMethods` vector for RPC wait states
   - Query methods: `isWaitingForRpcResponse()`, `requiredRpcMethod()`

5. **BaseTransaction Extensions**
   - New `outgoingRpcRequestSignal` for sending RPC requests
   - New `mRpcContext` queue for received responses
   - Methods: `sendRpcRequest()`, `pushRpcResponse()`, `popRpcResponse()`, `hasRpcResponse()`
   - Helper method: `resultWaitForRpcResponse()`

6. **TransactionsScheduler Extensions**
   - New `tryAttachRpcResponseToTransaction()` method
   - Wakes transaction when matching RPC response arrives

7. **TransactionsManager Extensions**
   - New `rpcRequestSignal` for forwarding requests to Core
   - New `onRpcResponseReceived()` handler
   - Signal subscription for transaction RPC requests

8. **Core Integration**
   - Create `ObserverRpcCommunicator` instance
   - Connect RPC signals between components
   - New `connectObserverRpcSignals()` method

#### RPC Methods to Implement

| Method | Request Class | Response Class |
|--------|--------------|----------------|
| `RPCService.GetCurrentBlock` | `GetBlockNumberRpcRequest` | `GetBlockNumberRpcResponse` |
| `RPCService.AcceptClaim` | `AcceptClaimRpcRequest` | `AcceptClaimRpcResponse` |
| `RPCService.GetClaimStatus` | `GetClaimStatusRpcRequest` | `GetClaimStatusRpcResponse` |
| `RPCService.SubmitClaimVotes` | `SubmitClaimVotesRpcRequest` | `SubmitClaimVotesRpcResponse` |
| `RPCService.GetClaimStatuses` | `GetClaimStatusesRpcRequest` | `GetClaimStatusesRpcResponse` |

**Note**: `GetClaimVotes` and `GetRejectionSignature` are NOT implemented as separate methods - their data is included in the extended `GetClaimStatus` response.

### Explicitly Out of Scope
- Migration of existing transactions to use async RPC (BaseExchangePaymentTransaction, CoordinatorExchangePaymentTransaction, IntermediateNodeExchangePaymentTransaction, ReceiverExchangePaymentTransaction)
- Removal of existing synchronous `ObserverRPCClient` (залишається тимчасово, вилучення буде окремою ітерацією; жодного фічефлагу/перемикача між sync/async не додається)
- Integration tests with real observer
- Connection pooling, request batching, circuit breaker patterns
- Multiple observer support (load balancing, failover)

### Dependencies from Previous Iterations
- Existing `Communicator` pattern for signal-based async communication
- `TransactionState` and `TransactionResult` classes
- `TransactionsScheduler` message attachment mechanism
- Boost.Asio for async I/O
- Boost.Signals2 for signal/slot mechanism

### Future Roadmap Impact
This PRD establishes infrastructure for:
- PRD-15+: Migration of payment transactions to async RPC
- Future: Connection pooling for better performance
- Future: Multiple observer support

## User Stories & Requirements

### User Personas

#### Primary User: vtcpd Node
- **Role**: Automated payment processing node
- **Goals**: Process payments without blocking on observer communication
- **Pain Points**: Event loop blocks during RPC calls
- **Technical Proficiency**: N/A (automated system)

### Functional Requirements

> Усі схеми запитів/відповідей (назви полів, структури, типи) беруться з [observer/README.md](../../../../observer/README.md). Будь-яка серіалізація/десеріалізація має відповідати цим контрактам, включно з форматами помилок Observer.

#### 1. RpcMethod Enum
- **Description**: Enumeration of all supported RPC methods
- **Acceptance Criteria**:
  - Contains: Unknown, GetBlockNumber, AcceptClaim, GetClaimStatus, SubmitClaimVotes, GetClaimStatuses
  - Values are distinct integers

#### 2. RpcResponseStatus Enum
- **Description**: Status codes for RPC responses
- **Acceptance Criteria**:
  - Contains: Success, Timeout, NetworkError, ParseError, RpcError
  - `Success` indicates observer returned valid result
  - `Timeout` indicates 10-second deadline exceeded
  - `NetworkError` indicates TCP connection/communication failure
  - `ParseError` indicates JSON parsing failure
  - `RpcError` indicates observer returned error in JSON response
  - Мапінг помилок: resolve/connect/write/read -> NetworkError; JSON parse -> ParseError; observer повернув error-поле -> RpcError; перевищення дедлайну -> Timeout; успіх -> Success; `errorMessage` містить деталі оригінальної помилки/коду Observer

#### 3. Base RpcRequest Class
- **Description**: Abstract base class for all RPC requests
- **Acceptance Criteria**:
  - Stores `TransactionUUID` of owning transaction
  - Pure virtual `method()` returns `RpcMethod`
  - `Shared` typedef for `shared_ptr<RpcRequest>`

#### 4. Base RpcResponse Class
- **Description**: Abstract base class for all RPC responses
- **Acceptance Criteria**:
  - Stores `TransactionUUID`, `RpcResponseStatus`, `errorMessage`
  - Pure virtual `method()` returns `RpcMethod`
  - `isSuccess()` returns true only when status is Success
  - `Shared` typedef for `shared_ptr<RpcResponse>`

#### 5. GetBlockNumberRpcRequest / GetBlockNumberRpcResponse
- **Description**: Request/response for getting current block number
- **Request fields**: transactionUUID only
- **Response fields**: blockNumber (BlockNumber type)
- **Acceptance Criteria**:
  - Request serializes to `{"method":"RPCService.GetCurrentBlock","params":[{}],"id":1}`
  - Response parses `block_number` from JSON result

#### 6. AcceptClaimRpcRequest / AcceptClaimRpcResponse
- **Description**: Request/response for submitting a claim to observer
- **Request fields**:
  - transactionUUID
  - claimTransactionUUID
  - maxClaimBlockNumber
  - participantsPublicKeys (map<PaymentNodeID, sphincs::PublicKey::Shared>)
  - publicKey (sphincs::PublicKey::Shared)
  - signature (sphincs::Signature::Shared)
- **Response fields**: success (bool), message (string)
- **Acceptance Criteria**:
  - Request serializes participants as array of {index, public_key}
  - Response parses success and message from JSON result

#### 7. GetClaimStatusRpcRequest / GetClaimStatusRpcResponse
- **Description**: Request/response for checking claim status (extended response)
- **Request fields**: transactionUUID, claimTransactionUUID, maxClaimBlockNumber
- **Response fields**:
  - state (enum: NotFound, Observing, Approved, Rejected)
  - votes (map<PaymentNodeID, sphincs::Signature::Shared>) - populated only when Approved
  - rejectionSignature (string) - populated only when Rejected
- **Acceptance Criteria**:
  - Maps observer states: "not found" -> NotFound, "observing" -> Observing, "approved" -> Approved, "rejected" -> Rejected
  - Parses votes array when state is "approved"
  - Parses signature field when state is "rejected"

#### 8. SubmitClaimVotesRpcRequest / SubmitClaimVotesRpcResponse
- **Description**: Request/response for submitting votes to observer
- **Request fields**:
  - transactionUUID
  - claimTransactionUUID
  - maxClaimBlockNumber
  - votes (map<PaymentNodeID, sphincs::Signature::Shared>)
  - publicKey, signature (for verification)
- **Response fields**: success (bool), message (string)
- **Acceptance Criteria**:
  - Request serializes votes as array of {index, signature}
  - Response parses success and message

#### 9. GetClaimStatusesRpcRequest / GetClaimStatusesRpcResponse
- **Description**: Request/response for batch claim status check
- **Request fields**: transactionUUID, claims (vector of {TransactionUUID, BlockNumber})
- **Response fields**: statuses (vector of {transactionUUID, maxClaimBlockNumber, state})
- **Acceptance Criteria**:
  - Request serializes claims array
  - Response parses statuses array (only existing claims included)

#### 10. AsyncRpcSession Class
- **Description**: Manages single async RPC request/response cycle
- **Acceptance Criteria**:
  - Constructor takes: IOCtx, observerAddress, RpcRequest, completionCallback, Logger
  - `start()` begins async_resolve
  - Chains: async_resolve -> async_connect -> async_write -> async_read_until
  - 10-second timeout timer runs in parallel (значення з константи класу, наприклад `kObserverRpcTimeoutMs`)
  - Таймер скасовується після успішного завершення, callback гарантується рівно один раз навіть при гонці таймаут/відповідь
  - На завершенні (успіх/помилка/таймаут) викликає callback з RpcResponse
  - Preserves TransactionUUID from original request in response
  - Код містить TODO про потенційний ліміт одночасних сесій (зараз необмежено)

#### 11. ObserverRpcCommunicator Class
- **Description**: Main class for async RPC communication
- **Acceptance Criteria**:
  - Constructor takes: IOCtx, observerAddress, Logger
  - `sendRequest(RpcRequest::Shared)` creates AsyncRpcSession and starts it
  - Stores observer address, provides getter/setter
  - Emits `rpcResponseSignal(RpcResponse::Shared)` when response received
  - Handles JSON serialization of requests internally (схеми та назви полів строго за [observer/README.md](../../../../observer/README.md))
  - Handles JSON deserialization of responses internally (схеми та назви полів строго за [observer/README.md](../../../../observer/README.md))
  - Мапить помилки transport/timeout/parse/RPC error на `RpcResponseStatus` та `errorMessage` (див. таблицю статусів)

#### 12. TransactionState Extensions
- **Description**: New wait state for RPC responses
- **Acceptance Criteria**:
  - `waitForRpcResponse(RpcMethod, timeout)` creates state waiting for specific RPC method
  - `isWaitingForRpcResponse()` returns true if waiting for RPC
  - `requiredRpcMethod()` returns the expected RpcMethod
  - `mustBeAwakenedOnRpcResponse()` returns true
  - Підтримує кілька паралельних RPC для транзакції; пробудження відбувається за фактичним порядком надходження відповідей (однопотоковий цикл)

#### 13. BaseTransaction Extensions
- **Description**: RPC request/response handling in base transaction
- **Acceptance Criteria**:
  - `outgoingRpcRequestSignal` emits RpcRequest::Shared
  - `sendRpcRequest(RpcRequest::Shared)` emits signal
  - `pushRpcResponse(RpcResponse::Shared)` adds to mRpcContext queue (FIFO). Черга обмежена 50 елементами через константу класу (наприклад, `kMaxRpcResponsesPerTransaction = 50`)
  - `pushRpcResponse()` повертає false, якщо черга переповнена; у цьому випадку формується синтетична RpcResponse зі статусом `RpcResponseStatus::RpcError` і повідомленням про переповнення черги та одразу повертається транзакції (без додавання в чергу) для обробки
  - `popRpcResponse<ResponseType>()` returns typed response from queue
  - `hasRpcResponse()` returns true if queue not empty
  - `resultWaitForRpcResponse(RpcMethod, timeout)` returns appropriate TransactionResult

#### 14. TransactionsScheduler Extensions
- **Description**: RPC response routing to transactions
- **Acceptance Criteria**:
  - `tryAttachRpcResponseToTransaction(RpcResponse::Shared)` finds transaction by UUID
  - Checks if transaction is waiting for this RpcMethod
  - Calls `pushRpcResponse()` on transaction; якщо повертає false (черга переповнена), формує RpcResponse зі статусом `RpcResponseStatus::RpcError` з повідомленням про переповнення та передає її транзакції для негайної обробки
  - If `mustBeAwakenedOnRpcResponse()`, launches transaction (порядок пробудження визначається порядком фактичного прибуття відповідей в однопотоковому циклі)

#### 15. TransactionsManager Extensions
- **Description**: Signal routing for RPC
- **Acceptance Criteria**:
  - `rpcRequestSignal` forwards requests to Core
  - `onRpcResponseReceived()` calls scheduler's tryAttachRpcResponseToTransaction
  - Subscribes to transaction's outgoingRpcRequestSignal when transaction created

#### 16. Core Integration
- **Description**: Wire up ObserverRpcCommunicator
- **Acceptance Criteria**:
  - Creates ObserverRpcCommunicator with observer address from config
  - `connectObserverRpcSignals()` connects:
    - TransactionsManager::rpcRequestSignal -> Core::onRpcRequestSlot
    - ObserverRpcCommunicator::rpcResponseSignal -> Core::onRpcResponseSlot
  - onRpcRequestSlot forwards to ObserverRpcCommunicator::sendRequest
  - onRpcResponseSlot forwards to TransactionsManager::onRpcResponseReceived

### Non-Functional Requirements

#### Performance
- RPC operations must not block the event loop
- Timeout must not exceed 10 seconds (класова константа в AsyncRpcSession)
- Memory usage per active RPC session should be minimal

#### Reliability
- All error conditions (timeout, network error, parse error, RPC error) must be properly reported via RpcResponseStatus
- Transaction must receive response (success or error) for every request sent
- No memory leaks from incomplete sessions
- Черга RPC-відповідей на транзакцію обмежена 50 елементами; при переповненні формується RpcError-відповідь про overflow і повертається транзакції

#### Maintainability
- Type-safe request/response classes
- Clear separation of concerns (session, communicator, serialization)
- Consistent with existing Communicator patterns

## Technical Specifications

### Architecture Evolution

#### Current Architecture
```
Transaction -> ObserverRPCClient -> [BLOCKS] -> Observer
                                 <- [BLOCKS] <-
```

#### Proposed Architecture
```
Transaction
    |
    v
outgoingRpcRequestSignal
    |
    v
TransactionsManager
    |
    v
rpcRequestSignal
    |
    v
Core::onRpcRequestSlot
    |
    v
ObserverRpcCommunicator::sendRequest
    |
    v
AsyncRpcSession [async TCP] -> Observer
                            <-
    |
    v
rpcResponseSignal
    |
    v
Core::onRpcResponseSlot
    |
    v
TransactionsManager::onRpcResponseReceived
    |
    v
TransactionsScheduler::tryAttachRpcResponseToTransaction
    |
    v
Transaction wakes up with RpcResponse in context
```

### File Structure

```
src/core/network/rpc/
├── RpcMethod.h
├── RpcRequest.h
├── RpcRequest.cpp
├── RpcResponse.h
├── RpcResponse.cpp
├── requests/
│   ├── GetBlockNumberRpcRequest.h
│   ├── GetBlockNumberRpcRequest.cpp
│   ├── AcceptClaimRpcRequest.h
│   ├── AcceptClaimRpcRequest.cpp
│   ├── GetClaimStatusRpcRequest.h
│   ├── GetClaimStatusRpcRequest.cpp
│   ├── SubmitClaimVotesRpcRequest.h
│   ├── SubmitClaimVotesRpcRequest.cpp
│   ├── GetClaimStatusesRpcRequest.h
│   └── GetClaimStatusesRpcRequest.cpp
├── responses/
│   ├── GetBlockNumberRpcResponse.h
│   ├── GetBlockNumberRpcResponse.cpp
│   ├── AcceptClaimRpcResponse.h
│   ├── AcceptClaimRpcResponse.cpp
│   ├── GetClaimStatusRpcResponse.h
│   ├── GetClaimStatusRpcResponse.cpp
│   ├── SubmitClaimVotesRpcResponse.h
│   ├── SubmitClaimVotesRpcResponse.cpp
│   ├── GetClaimStatusesRpcResponse.h
│   └── GetClaimStatusesRpcResponse.cpp
├── AsyncRpcSession.h
├── AsyncRpcSession.cpp
├── ObserverRpcCommunicator.h
└── ObserverRpcCommunicator.cpp
```

### CMake Structure

Create new library `network__rpc` in `src/core/network/rpc/CMakeLists.txt`:
- Links to: common, logger, contractors, observing (for types), Boost::asio, nlohmann_json
- Add to main CMakeLists.txt
- Link to unit_tests

### Technology Stack
- **Async I/O**: Boost.Asio (async_connect, async_write, async_read_until, steady_timer)
- **Signals**: Boost.Signals2
- **JSON**: nlohmann/json (already used in project)
- **TCP**: boost::asio::ip::tcp

## Testing Strategy

### Unit Tests

Location: `tests/unit/network/rpc/`

#### 1. RpcMethodTest.cpp
- Test enum values are distinct
- Test all expected values exist

#### 2. RpcResponseStatusTest.cpp
- Test enum values are distinct
- Test all expected values exist

#### 3. RpcRequestTest.cpp
- Test base class constructor stores transactionUUID correctly
- Test transactionUUID() getter

#### 4. RpcResponseTest.cpp
- Test constructor stores all fields correctly
- Test isSuccess() returns true only for Success status
- Test getters return correct values
- Test all status values

#### 5. GetBlockNumberRpcRequestTest.cpp
- Test constructor
- Test method() returns RpcMethod::GetBlockNumber
- Test transactionUUID() getter

#### 6. GetBlockNumberRpcResponseTest.cpp
- Test constructor with success status
- Test constructor with error status
- Test blockNumber() getter
- Test method() returns RpcMethod::GetBlockNumber

#### 7. AcceptClaimRpcRequestTest.cpp
- Test constructor stores all fields
- Test all getters (claimTransactionUUID, maxClaimBlockNumber, participantsPublicKeys, publicKey, signature)
- Test method() returns RpcMethod::AcceptClaim

#### 8. AcceptClaimRpcResponseTest.cpp
- Test constructor with success=true
- Test constructor with success=false
- Test success() and message() getters
- Test method() returns RpcMethod::AcceptClaim

#### 9. GetClaimStatusRpcRequestTest.cpp
- Test constructor
- Test getters (claimTransactionUUID, maxClaimBlockNumber)
- Test method() returns RpcMethod::GetClaimStatus

#### 10. GetClaimStatusRpcResponseTest.cpp
- Test constructor with NotFound state
- Test constructor with Observing state
- Test constructor with Approved state and votes
- Test constructor with Rejected state and rejectionSignature
- Test state() getter
- Test votes() getter (empty for non-Approved)
- Test rejectionSignature() getter (empty for non-Rejected)
- Test method() returns RpcMethod::GetClaimStatus

#### 11. SubmitClaimVotesRpcRequestTest.cpp
- Test constructor stores all fields
- Test all getters
- Test method() returns RpcMethod::SubmitClaimVotes

#### 12. SubmitClaimVotesRpcResponseTest.cpp
- Test constructor with success/failure
- Test getters
- Test method() returns RpcMethod::SubmitClaimVotes

#### 13. GetClaimStatusesRpcRequestTest.cpp
- Test constructor with claims vector
- Test claims() getter
- Test method() returns RpcMethod::GetClaimStatuses

#### 14. GetClaimStatusesRpcResponseTest.cpp
- Test constructor with statuses vector
- Test statuses() getter
- Test method() returns RpcMethod::GetClaimStatuses

#### 15. ObserverRpcCommunicatorSerializationTest.cpp
- Test JSON serialization of GetBlockNumberRpcRequest
- Test JSON serialization of AcceptClaimRpcRequest
- Test JSON serialization of GetClaimStatusRpcRequest
- Test JSON serialization of SubmitClaimVotesRpcRequest
- Test JSON serialization of GetClaimStatusesRpcRequest
- Test JSON deserialization of GetBlockNumberRpcResponse
- Test JSON deserialization of AcceptClaimRpcResponse
- Test JSON deserialization of GetClaimStatusRpcResponse (all states)
- Test JSON deserialization of SubmitClaimVotesRpcResponse
- Test JSON deserialization of GetClaimStatusesRpcResponse
- Усі серіалізаційні/десеріалізаційні тести мають перевіряти відповідність контрактам з [observer/README.md](../../../../observer/README.md)

#### 16. AsyncRpcSessionTest.cpp
- Success path: resolve/connect/write/read, таймер скасовано після успіху, callback рівно раз
- Timeout path: спрацьовує 10-секундний таймер (класова константа)
- Network error path: resolve/connect/write/read дають NetworkError
- Parse error path: некоректний JSON -> ParseError
- Rpc error path: помилка від Observer -> RpcError з переданим повідомленням
- Пізня відповідь після таймауту не викликає другий callback

#### 17. TransactionsSchedulerRpcRoutingTest.cpp
- tryAttachRpcResponseToTransaction доставляє відповідь транзакції, яка чекає відповідний RpcMethod
- Порядок пробудження відповідає порядку фактичного надходження відповідей
- Якщо `pushRpcResponse()` повертає false (черга заповнена), формується RpcError-відповідь про переповнення і передається транзакції

#### 18. BaseTransactionRpcQueueTest.cpp
- Ліміт черги mRpcContext = 50 (класова константа); 51-й елемент не додається, push повертає false
- FIFO порядок popRpcResponse
- hasRpcResponse відображає стан черги

### Test Files to Add to CMakeLists.txt

```cmake
# In tests/unit/CMakeLists.txt, add:
network/rpc/RpcMethodTest.cpp
network/rpc/RpcResponseStatusTest.cpp
network/rpc/RpcRequestTest.cpp
network/rpc/RpcResponseTest.cpp
network/rpc/GetBlockNumberRpcRequestTest.cpp
network/rpc/GetBlockNumberRpcResponseTest.cpp
network/rpc/AcceptClaimRpcRequestTest.cpp
network/rpc/AcceptClaimRpcResponseTest.cpp
network/rpc/GetClaimStatusRpcRequestTest.cpp
network/rpc/GetClaimStatusRpcResponseTest.cpp
network/rpc/SubmitClaimVotesRpcRequestTest.cpp
network/rpc/SubmitClaimVotesRpcResponseTest.cpp
network/rpc/GetClaimStatusesRpcRequestTest.cpp
network/rpc/GetClaimStatusesRpcResponseTest.cpp
network/rpc/ObserverRpcCommunicatorSerializationTest.cpp
network/rpc/AsyncRpcSessionTest.cpp
network/rpc/TransactionsSchedulerRpcRoutingTest.cpp
network/rpc/BaseTransactionRpcQueueTest.cpp
```

### Quality Gates
- All unit tests pass
- No memory leaks (valgrind clean)
- Code compiles without warnings
- Follows existing code style

## Risk Management

### Technical Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| Async TCP complexity | Medium | Medium | Follow existing Communicator patterns, thorough testing |
| Signal connection errors | High | Low | Copy patterns from existing signal connections in Core |
| JSON serialization bugs | Medium | Medium | Comprehensive serialization tests |
| Timeout handling issues | High | Low | Use steady_timer with proper cancellation |

## Appendices

### Glossary
- **RPC**: Remote Procedure Call - synchronous-style API over network
- **Async**: Asynchronous - non-blocking operations using callbacks
- **Observer**: External blockchain-like service for claim verification
- **Claim**: Transaction commitment request to observer
- **Signal**: Boost.Signals2 signal for decoupled event handling

### References
- [Boost.Asio Documentation](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html)
- [Boost.Signals2 Documentation](https://www.boost.org/doc/libs/release/doc/html/signals2.html)
- [Observer RPC Protocol](../../../../observer/README.md)
- [Architecture Plan](../../../../docs/async-rpc-implementation-plan.md)

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-12-17 | Claude Code | Initial draft | Phase 1 |
| 1.1 | 2025-12-17 | Claude Code | Clarified async RPC timeouts, queue limits, error mapping, and testing | Phase 1 |

**Related Documents**
- **Previous PRD**: [13-observer-successful-transactions-monitoring.md](13-observer-successful-transactions-monitoring.md)
- **Architecture Plan**: [async-rpc-implementation-plan.md](../../../../docs/async-rpc-implementation-plan.md)
