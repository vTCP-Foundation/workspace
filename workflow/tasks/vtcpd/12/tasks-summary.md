# PRD 12: Observer Ambiguous Transaction Handling - Tasks Summary

**PRD**: [12-observer-ambiguous-transaction-handling.md](../../../prd/vtcpd/12-observer-ambiguous-transaction-handling.md)

**Total Tasks**: 9 (7 implementation + 2 testing)

---

## Implementation Tasks (12-01 to 12-07)

### 12-01 - ObservingPaymentClaim Class Implementation
**Type**: Simple
**Description**: Create ObservingPaymentClaim class with all required fields (TransactionUUID, BlockNumber, participants map, public key, signature) and ClaimStatus enum (NoInfo=0, Observing=1, ParticipantsVotesPresent=2, RejectedByObserving=3, Done=4).

**Files**:
- `src/core/observing/ObservingPaymentClaim.h`
- `src/core/observing/ObservingPaymentClaim.cpp`

**Dependencies**: None

---

### 12-02 - Claim Storage Map in ObservingHandler
**Type**: Simple
**Description**: Add mPaymentClaims map to ObservingHandler with composite key (TransactionUUID, BlockNumber) and implement addPaymentClaim public method for adding new claims.

**Files**:
- `src/core/observing/ObservingHandler.h`
- `src/core/observing/ObservingHandler.cpp`

**Dependencies**: 12-01

---

### 12-03 - Periodic Timer Processing for Payment Claims
**Type**: Moderate
**Description**: Implement timer-based claim processing with state machine dispatch (NoInfo→sendClaim, Observing→checkTransaction, ParticipantsVotesPresent→getParticipantsSignatures, RejectedByObserving→rejectTransaction, Done→cleanup). Includes timer initialization, processPaymentClaims method, and method stubs.

**Files**:
- `src/core/observing/ObservingHandler.h`
- `src/core/observing/ObservingHandler.cpp`

**Dependencies**: 12-02

---

### 12-04 - RPC Method: sendClaim Implementation
**Type**: Moderate
**Description**: Implement sendClaim to submit claims to observer via RPCService.AcceptClaim. Constructs JSON-RPC request with participants array, public keys and signature in base64, sends via TCP, updates status to Observing on success.

**Files**:
- `src/core/observing/ObservingHandler.cpp`

**Dependencies**: 12-03

---

### 12-05 - RPC Method: checkTransaction Implementation
**Type**: Moderate
**Description**: Implement checkTransaction to query claim status via RPCService.GetClaimStatus. Updates status based on response: "not found"→NoInfo, "observing"→unchanged, "approved"→ParticipantsVotesPresent, "rejected"→RejectedByObserving.

**Files**:
- `src/core/observing/ObservingHandler.cpp`

**Dependencies**: 12-04

---

### 12-06 - RPC Method: getParticipantsSignatures Implementation
**Type**: Moderate
**Description**: Implement getParticipantsSignatures to retrieve participant votes via RPCService.GetClaimVotes. Converts base64 signatures to sphincs::Signature::Shared, builds map by PaymentNodeID, invokes mParticipantsVotesSignal, sets status to Done.

**Files**:
- `src/core/observing/ObservingHandler.cpp`

**Dependencies**: 12-05

---

### 12-07 - RPC Method: rejectTransaction Implementation
**Type**: Moderate
**Description**: Implement rejectTransaction to retrieve rejection signature via RPCService.GetRejectionSignature. If signature present: invokes mRejectTransactionSignal and sets status Done. If empty: logs debug and keeps status RejectedByObserving for retry.

**Files**:
- `src/core/observing/ObservingHandler.cpp`

**Dependencies**: 12-06

---

## Testing Tasks (12-08 to 12-09)

### 12-08 - Unit Tests: Core Components
**Type**: Simple
**Description**: Unit tests for ObservingPaymentClaim class (already implemented in task 12-01). Tests for ObservingHandler components (claim map, timer, RPC methods) are integration tests and out of scope for unit testing per PRD.

**Test Files**:
- `tests/unit/observing/ObservingPaymentClaimTest.cpp` (5 tests) - ✅ Already implemented

**Total**: 5 test cases

**Dependencies**: 12-01

**Note**: Originally included tests for ObservingHandler claim map, timer, and processing, but these require integration infrastructure (IOContext, network, observer service) and are not true unit tests.

---

### 12-09 - Unit Tests: RPC Methods
**Type**: Simple
**Description**: Verification that ObservingPaymentClaim unit tests exist and pass. Originally scoped for RPC method tests, but these are integration tests requiring IOContext, network infrastructure, and running observer service - not suitable for unit testing.

**Test Files**:
- `tests/unit/observing/ObservingPaymentClaimTest.cpp` (5 tests) - ✅ Already implemented

**Total**: 5 test cases (same as 12-08, verification task)

**Dependencies**: 12-08

**Note**: RPC method tests (sendClaim, checkTransaction, getParticipantsSignatures, rejectTransaction) removed from scope as they are integration tests, not unit tests. Will be covered in separate integration test tasks.

---

## Overall Test Coverage

**Total Test Files**: 1
**Total Test Cases**: 5 (all in ObservingPaymentClaimTest.cpp)

**Coverage Areas (Unit Tests Only)**:
- ✅ ObservingPaymentClaim class (constructor, getters, setters, enum)

**Integration Test Coverage (Out of Scope for Unit Tests)**:
- ⏸️ Claim storage and map operations (requires IOContext)
- ⏸️ Timer infrastructure and periodic processing (requires IOContext, timers)
- ⏸️ State machine dispatch logic (requires running ObservingHandler)
- ⏸️ All 4 RPC methods (requires network, observer service)
- ⏸️ JSON-RPC request construction (requires network infrastructure)
- ⏸️ Response parsing and status updates (requires observer service)
- ⏸️ Signal emission (requires full ObservingHandler context)
- ⏸️ Exception handling and error recovery (requires network errors)
- ⏸️ Cleanup logic (requires timer processing)
- ⏸️ Edge cases (requires full system context)
- ⏸️ Performance testing (requires full system)

**Note**: Integration test coverage will be addressed in separate integration test tasks with appropriate infrastructure.

---

## Execution Order

### Sequential Implementation (cannot be parallelized):
1. 12-01 (class foundation)
2. 12-02 (storage infrastructure)
3. 12-03 (timer and dispatch)
4. 12-04 (sendClaim)
5. 12-05 (checkTransaction)
6. 12-06 (getParticipantsSignatures)
7. 12-07 (rejectTransaction)

### Testing (after implementation):
- 12-08 (verification that ObservingPaymentClaimTest.cpp exists, depends on 12-01)
- 12-09 (verification task, same as 12-08)

---

## Key Validation Criteria

### All Implementation Tasks:
- Code compiles without errors or warnings
- Follows existing project patterns
- Includes appropriate debug logging
- Integrates with processPaymentClaims timer loop

### All Testing Tasks:
- 100% test pass rate
- Tests compile without warnings
- Tests run in `build-tests`
- Can execute via `./bin/unit_tests --gtest_filter=...`

---

## Notes

- **Single-threaded execution**: No synchronization needed for map access
- **No persistent storage**: Claims lost on restart (intentional per PRD)
- **First observer only**: No multi-observer support in this iteration
- **No signature validation**: Observer responses accepted as-is (per PRD)
- **Timer period**: 60s production, 10s for tests (via class constants)

---

**Status**: Ready for implementation
**Next Step**: Begin with task 12-01
