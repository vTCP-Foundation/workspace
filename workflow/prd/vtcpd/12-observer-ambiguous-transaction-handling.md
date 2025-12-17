# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Observer Integration for Ambiguous Transaction Handling
- **PRD ID**: 12
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.0
- **Date**: 2025-12-09
- **Author(s)**: Claude Code, based on Architect's requirements
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-12-09
- **Previous PRD**: [11-path-rebuilding-on-inaccessible-nodes.md](11-path-rebuilding-on-inaccessible-nodes.md)
- **Related Documents**:
  - [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
  - [Observer README](../../../observer/README.md)
  - [ObservingHandler Implementation](../../../src/core/observing/ObservingHandler.h)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This PRD introduces observer-side infrastructure for handling ambiguous payment transactions where a node has signed but cannot obtain signatures from other participants. When payment transactions reach an uncertain state, the system will be able to engage observers to make final decisions; in this iteration we focus on the ObservingHandler capabilities only, while wiring to payment transactions is handled in later work.

### Current project state
- Exchange payment transactions (BaseExchangePaymentTransaction, CoordinatorExchangePaymentTransaction, IntermediateNodeExchangePaymentTransaction, ReceiverExchangePaymentTransaction) handle multi-equivalent payments with commissions
- ObservingHandler exists with basic infrastructure for observer communication via RPC
- Observers can accept claims, track claim status, provide participant votes, and issue rejection signatures
- Current implementation lacks automatic integration between payment transactions and observer system

### This iteration's focus
- Create ObservingPaymentClaim class for representing ambiguous payment transactions in ObservingHandler
- Implement periodic timer-based processing of claims with state machine
- Integrate RPC methods for claim submission, status checking, vote retrieval, and rejection handling
- Expose signals for future automatic transaction finalization or rejection based on observer decisions (connection to payment transactions is out of scope here)
- Implement proper cleanup of completed claims

### Connection to overall vision
This completes the observer integration for the vTCP payment network, ensuring that payment transactions can reach definitive outcomes even when network participants cannot reach consensus, improving network reliability and user experience.

## Iteration Context
### Previous Iterations Summary
- **PRD 06**: Exchange Payment with Commissions - multi-equivalent payment execution
- **PRD 11**: Path Rebuilding on Inaccessible Nodes - improved payment reliability
- **Completed Features**: Multi-equivalent payment transactions, observer RPC client, basic observer communication infrastructure

### Lessons Learned
- Separation of concerns between payment logic and observer communication is critical
- Timer-based processing with state machines provides reliable asynchronous operation
- RPC error handling requires robust exception management and retry logic

### Current State Analysis
- **What's working well**: Observer RPC infrastructure functions correctly for block number queries
- **Pain points identified**: Payment transactions cannot recover from ambiguous states without manual intervention
- **Performance metrics**: N/A (new feature)

## Problem Statement
### Background
In payment transactions, nodes sign their participation but must collect signatures from all other participants to finalize the transaction. Network issues, node failures, or malicious behavior can prevent signature collection, leaving transactions in an ambiguous state where they cannot be completed or safely rolled back.

The system includes an observer role designed to make final decisions on such transactions, but there is currently no automatic integration between payment transactions and the observer system. This requires manual intervention or leaves transactions permanently uncertain.

### Problem Description
**Who is affected**: All network participants executing payment transactions, particularly in scenarios with network instability or Byzantine behavior

**When and where**: During payment transaction voting stages when a node has signed but cannot obtain required signatures from other participants

**Current limitations**:
- No automatic mechanism to submit ambiguous transactions to observers
- No periodic checking of observer decisions on pending transactions
- No automatic application of observer decisions to finalize or reject transactions
- Ambiguous transactions remain in uncertain state indefinitely

### Impact of not solving this problem
- Users experience payment failures without resolution
- Network trust decreases due to unresolved transactions
- Manual intervention required for transaction recovery
- System resources locked by permanently uncertain transactions

### Success Metrics
**Primary KPIs**:
- 100% of claims provided via addPaymentClaim enter the processing state machine
- Observer decisions are detected and corresponding signals emitted within one processing cycle after observer finalizes
- No manual intervention required for claim progression after it is added (except re-adding after process restart, which is known and accepted)

**Target Values**:
- Claims are accepted immediately via addPaymentClaim; first processing occurs in the next timer cycle
- Status check every kPaymentClaimProcessingPeriodSeconds (default 60 seconds via class constant)
- Decision signal emission within one status check cycle after observer finalizes
- Cleanup of completed claims within the next processing cycle

## Goals
The primary goals of this iteration are to provide observer-side infrastructure for ambiguous transaction handling.

*   **Goal 1: Provide infrastructure to accept ambiguous payment claims in ObservingHandler.**
    *   **Description:** ObservingHandler must accept claims via addPaymentClaim and initialize them for processing without manual follow-up.
    *   **Success Metric:** 100% of claims supplied to ObservingHandler are enqueued and enter the processing state machine within the first timer cycle.

*   **Goal 2: Implement reliable periodic checking of observer decisions.**
    *   **Description:** Continuously monitor observer decisions on pending claims and respond appropriately to status changes.
    *   **Success Metric:** Observer decisions are detected and processed within one checking cycle (per kPaymentClaimProcessingPeriodSeconds).

*   **Goal 3: Expose observer decisions for downstream payment transaction handling.**
    *   **Description:** Emit participants votes or rejection signals based on observer responses so that payment transaction code can connect later.
    *   **Success Metric:** For approved or rejected claims, the corresponding signal is emitted within one processing cycle; consumption of the signal by payment transactions is out of scope for this iteration.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **ObservingPaymentClaim Class**: Representation of ambiguous payment transactions in ObservingHandler context
2. **Claim Storage Map**: Map for tracking pending claims in ObservingHandler (single-threaded access)
3. **Periodic Timer Processing**: Timer-based state machine for claim lifecycle management
4. **RPC Integration Methods**: Implementation of sendClaim, checkTransaction, getParticipantsSignatures, rejectTransaction
5. **Signal-Based Transaction Communication**: Expose signals for decision application; wiring to payment transactions is deferred to future work

#### Technical Infrastructure
- New ObservingPaymentClaim class with status enum
- Enhanced ObservingHandler with claim map and timer
- RPC method implementations for observer protocol
- Signal invocations for payment transaction callbacks

#### Integration Points
- Integration with ObserverRPCClient for network communication
- Usage of observer RPC protocol from observer/README.md
- Signal connections to payment transaction handlers (future integration)

### Explicitly Out of Scope
- Integration with payment transactions and automatic creation of claims from payment transaction flow (separate task)
- Observer blockchain implementation details
- Cryptographic validation of observer signatures
- Observer selection and failover logic
- UI/monitoring for claim tracking
- Persistent storage of claim history

### Dependencies from Previous Iterations
- **PRD 06**: BaseExchangePaymentTransaction and related payment classes
- **Existing observer infrastructure**: ObserverRPCClient, ObservingHandler base
- **Observer service**: Running observer with RPC endpoints from observer/README.md

### Future Roadmap Impact
This iteration establishes foundation for:
- **Payment transaction integration**: Automatic claim submission from payment transactions
- **Multi-observer support**: Observer selection and failover mechanisms
- **Claim analytics**: Tracking and analysis of observer decisions
- **Advanced recovery strategies**: Machine learning-based prediction of transaction outcomes

## User Stories & Requirements

### User Personas
#### Primary User: Payment Transaction
- **Role**: Software component executing payment operations
- **Goals**: Reach definitive outcome (complete or reject) for all transactions
- **Pain Points**: Cannot proceed when signatures are unavailable; uncertain state persists indefinitely
- **Technical Proficiency**: N/A (software component)

#### Secondary User: Network Operator
- **Role**: Node operator maintaining network infrastructure
- **Goals**: Ensure reliable payment processing without manual intervention
- **Pain Points**: Must manually investigate and resolve stuck transactions
- **Technical Proficiency**: Advanced

#### Tertiary User: Payment Participant
- **Role**: User sending or receiving payments
- **Goals**: Complete payments reliably and predictably
- **Pain Points**: Payments fail or hang without clear resolution
- **Technical Proficiency**: Intermediate

### Functional Requirements
#### New Features for This Iteration

1. **ObservingPaymentClaim Class**
   - **Description**: Class representing ambiguous payment transaction in observer context with status tracking
   - **User Story**: As ObservingHandler, I need to track payment claims with their current processing status
   - **Rationale**: Enables state machine-based processing of claims through observer protocol
   - **Builds Upon**: ObservingTransaction pattern
   - **Acceptance Criteria**:
     - Class contains fields: TransactionUUID, BlockNumber maxBlockNumberForClaiming, map<PaymentNodeID, PublicKey> participantsPublicKeys, PublicKey, Signature
     - Status enum with values: NoInfo = 0, Observing = 1, ParticipantsVotesPresent = 2, RejectedByObserving = 3, Done = 4
     - Constructor initializes all fields and sets status to NoInfo
     - Getters provided for all fields and status
     - Status setter for state transitions
     - Located in src/core/observing/
   - **Priority**: High
   - **Dependencies**: sphincs cryptography library

2. **Claim Storage and Management**
   - **Description**: Map-based storage for tracking pending claims with composite key
   - **User Story**: As ObservingHandler, I need to store and retrieve claims by transaction identity
   - **Rationale**: Enables efficient claim lookup and prevents duplicate submissions
   - **Builds Upon**: Existing ObservingHandler map pattern (mClaims)
   - **Acceptance Criteria**:
     - Map with key: pair<TransactionUUID, BlockNumber>
     - Map with value: ObservingPaymentClaim::Shared (shared_ptr)
     - Public method addPaymentClaim to create new entries
     - Composite key ensures uniqueness per (transactionUUID, maxBlockNumberForClaiming)
     - Access occurs in single-threaded context of vtcpd; no additional synchronization is required
   - **Priority**: High
   - **Dependencies**: ObservingPaymentClaim class

3. **Periodic Timer Processing**
   - **Description**: Timer-based automatic processing of all pending claims according to their status
   - **User Story**: As ObservingHandler, I need to periodically process claims without external triggers
   - **Rationale**: Ensures claims progress through observer protocol automatically
   - **Builds Upon**: Existing timer pattern in ObservingHandler (mClaimsTimer, mTransactionsTimer)
   - **Acceptance Criteria**:
     - New timer started in ObservingHandler constructor
     - Timer period defined as class constant (similar to kTransactionCheckingSignalRepeatTimeSeconds)
     - Timer callback method processes all claims in map
     - Processing dispatches to appropriate method based on claim status:
       - NoInfo → sendClaim
       - Observing → checkTransaction
       - ParticipantsVotesPresent → getParticipantsSignatures
       - RejectedByObserving → rejectTransaction
       - Done → skip (will be cleaned up)
     - Cleanup phase removes all claims with status Done
     - Timer reschedules after each processing cycle
   - **Priority**: High
   - **Dependencies**: ObservingPaymentClaim class, claim map

4. **RPC Method: sendClaim**
   - **Description**: Submit claim to observer via RPCService.AcceptClaim
   - **User Story**: As ObservingHandler, I need to send claim details to observer for tracking
   - **Rationale**: Initiates observer involvement in transaction resolution
   - **Builds Upon**: ObserverRPCClient pattern (getBlockNumber)
   - **Acceptance Criteria**:
     - Method accepts ObservingPaymentClaim parameter
     - Constructs JSON-RPC request with:
       - transaction_uuid (string)
       - max_claim_block_number (uint64)
       - participants (array of {index: PaymentNodeID, public_key: string})
       - public_key (string, base64)
       - signature (string, base64)
     - Public keys and signature converted to base64 via toString()
     - Uses first observer from mObservers (same as getActualBlockNumber)
     - On success: updates claim status to Observing
     - On error: throws exception (same pattern as getActualBlockNumber)
     - Located in ObservingHandler.cpp
   - **Priority**: High
   - **Dependencies**: ObserverRPCClient, JSON library, observer RPC service

5. **RPC Method: checkTransaction**
   - **Description**: Query observer for claim status via RPCService.GetClaimStatus
   - **User Story**: As ObservingHandler, I need to check if observer has made a decision on claim
   - **Rationale**: Enables detection of observer decisions for claim progression
   - **Builds Upon**: ObserverRPCClient RPC pattern
   - **Acceptance Criteria**:
     - Method accepts ObservingPaymentClaim parameter
     - Constructs JSON-RPC request with transaction_uuid and max_claim_block_number
     - Parses response state field
     - State "not found" → updates claim status to NoInfo
     - State "observing" → no status change (remains Observing)
     - State "approved" → updates claim status to ParticipantsVotesPresent
     - State "rejected" → updates claim status to RejectedByObserving
     - Uses first observer from mObservers
     - On error: throws exception
     - Located in ObservingHandler.cpp
   - **Priority**: High
   - **Dependencies**: ObserverRPCClient, observer RPC service

6. **RPC Method: getParticipantsSignatures**
   - **Description**: Retrieve participant votes from observer via RPCService.GetClaimVotes
   - **User Story**: As ObservingHandler, I need to obtain participant signatures to finalize transaction
   - **Rationale**: Provides signatures necessary for transaction completion
   - **Builds Upon**: ObserverRPCClient RPC pattern
   - **Acceptance Criteria**:
     - Method accepts ObservingPaymentClaim parameter
     - Constructs JSON-RPC request with transaction_uuid and max_claim_block_number
     - Parses response votes array: [{index: PaymentNodeID, signature: string}]
     - Converts each signature from base64 string to sphincs::Signature::Shared
     - Creates map<PaymentNodeID, sphincs::Signature::Shared>
     - Invokes mParticipantsVotesSignal(transactionUUID, maxBlockNumber, signatures_map)
     - Updates claim status to Done
     - Uses first observer from mObservers
     - On error: throws exception
     - Located in ObservingHandler.cpp
   - **Priority**: High
   - **Dependencies**: ObserverRPCClient, sphincs signature parsing

7. **RPC Method: rejectTransaction**
   - **Description**: Retrieve rejection signature from observer via RPCService.GetRejectionSignature
   - **User Story**: As ObservingHandler, I need to obtain rejection signature to rollback transaction
   - **Rationale**: Provides proof of observer rejection for safe transaction rollback
   - **Builds Upon**: ObserverRPCClient RPC pattern
   - **Acceptance Criteria**:
     - Method accepts ObservingPaymentClaim parameter
     - Constructs JSON-RPC request with transaction_uuid and max_claim_block_number
     - Parses response: state and signature fields
     - If signature field is non-empty:
       - Invokes mRejectTransactionSignal(transactionUUID, maxBlockNumber)
       - Updates claim status to Done
     - If signature field is empty:
       - No signal invocation
       - No status change (remains RejectedByObserving)
     - Uses first observer from mObservers
     - On error: throws exception
     - Located in ObservingHandler.cpp
   - **Priority**: High
   - **Dependencies**: ObserverRPCClient

#### Enhancements to Existing Features

1. **ObservingHandler Enhancement**
   - **Current State**: ObservingHandler manages ObservingTransaction objects for claim protocol
   - **Proposed Changes**:
     - Add map for ObservingPaymentClaim objects
     - Add timer for periodic claim processing
     - Add processing method dispatching based on claim status
     - Add cleanup logic for completed claims
   - **Impact Assessment**: No impact on existing ObservingTransaction functionality; purely additive
   - **Migration Strategy**: N/A (new functionality)

### Non-Functional Requirements
#### Performance
- Claim processing cycle completes within 5 seconds for up to 100 pending claims
- RPC request timeout: 10 seconds per request
- Timer period: 60 seconds (configurable via class constants; no external configuration in this iteration)
- Memory overhead: < 10MB per 1000 pending claims

#### Security
- No validation of observer signatures in this iteration (accepted as-is; verification will be added in future tasks)
- Observer responses are emitted via signals for downstream consumers; current iteration does not apply them inside payment transactions
- Use first observer only (no multi-observer validation; selection/quorum deferred)

#### Scalability
- Support up to 1000 concurrent pending claims
- Handle observer response delays up to timeout period
- Process all pending claims within single timer cycle

#### Reliability
- Automatic retry via status transition (NoInfo loop)
- Exception handling prevents timer failure
- Cleanup ensures map doesn't grow indefinitely

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: ObservingHandler with ObservingTransaction for claim protocol; separate payment transaction classes
- **Proposed Changes**: Add parallel ObservingPaymentClaim system with independent timer and map; integrate via signals
- **Execution Model**: vtcpd is single-threaded; ObservingHandler claim processing shares the same thread, so no explicit locking is required
- **Backwards Compatibility**: Existing ObservingTransaction functionality unchanged
- **Migration Requirements**: None

### Technology Stack Updates
#### New Technologies/Libraries
- No new external libraries required
- Uses existing: ObserverRPCClient, JSON library (nlohmann/json), sphincs crypto, boost signals

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- ObservingHandler integrates with observer RPC service (RPCService.AcceptClaim, GetClaimStatus, GetClaimVotes, GetRejectionSignature)
- ObservingHandler signals integrate with payment transaction callbacks (future work)

#### Modified Integrations
- None

### Data Requirements
#### Data Models

##### ObservingPaymentClaim Class
**Purpose**: Represent ambiguous payment transaction claim with status tracking

**Fields**:
```cpp
class ObservingPaymentClaim {
public:
    typedef shared_ptr<ObservingPaymentClaim> Shared;

    enum ClaimStatus {
        NoInfo = 0,
        Observing = 1,
        ParticipantsVotesPresent = 2,
        RejectedByObserving = 3,
        Done = 4
    };

    ObservingPaymentClaim(
        const TransactionUUID &transactionUUID,
        BlockNumber maxBlockNumberForClaiming,
        const map<PaymentNodeID, sphincs::PublicKey::Shared> &participantsPublicKeys,
        sphincs::PublicKey::Shared publicKey,
        sphincs::Signature::Shared signature);

    const TransactionUUID& transactionUUID() const;
    BlockNumber maxBlockNumberForClaiming() const;
    const map<PaymentNodeID, sphincs::PublicKey::Shared>& participantsPublicKeys() const;
    sphincs::PublicKey::Shared publicKey() const;
    sphincs::Signature::Shared signature() const;
    ClaimStatus status() const;
    void setStatus(ClaimStatus status);

private:
    TransactionUUID mTransactionUUID;
    BlockNumber mMaxBlockNumberForClaiming;
    map<PaymentNodeID, sphincs::PublicKey::Shared> mParticipantsPublicKeys;
    sphincs::PublicKey::Shared mPublicKey;
    sphincs::Signature::Shared mSignature;
    ClaimStatus mStatus;
};
```

##### ObservingHandler Enhancement
**Purpose**: Manage payment claims with periodic processing

**Added Fields**:
```cpp
// Claim storage
map<pair<TransactionUUID, BlockNumber>, ObservingPaymentClaim::Shared> mPaymentClaims;

// Timer for periodic processing
as::steady_timer mPaymentClaimsTimer;

// Timer period constant
static constexpr uint32_t kPaymentClaimProcessingPeriodSeconds = 60;
#ifdef TESTS
static constexpr uint32_t kPaymentClaimProcessingPeriodSecondsTests = 10;
#endif
```

**Added Methods**:
```cpp
// Public method to add claim
void addPaymentClaim(
    const TransactionUUID &transactionUUID,
    BlockNumber maxBlockNumberForClaiming,
    const map<PaymentNodeID, sphincs::PublicKey::Shared> &participantsPublicKeys,
    sphincs::PublicKey::Shared publicKey,
    sphincs::Signature::Shared signature);

// Protected processing methods
void processPaymentClaims();
void sendClaim(ObservingPaymentClaim::Shared claim);
void checkTransaction(ObservingPaymentClaim::Shared claim);
void getParticipantsSignatures(ObservingPaymentClaim::Shared claim);
void rejectTransaction(ObservingPaymentClaim::Shared claim);
```

#### Data Storage
- No persistent storage required in this iteration (intentional)
- In-memory map storage only
- Claims are lost on process restart; callers must re-add claims after restart

#### Data Migration
- No migration required (new feature)

### Algorithm Specifications

#### Periodic Claim Processing Algorithm
**Purpose**: Process all pending claims according to their status

**Algorithm**:
```cpp
void ObservingHandler::processPaymentClaims() {
    #ifdef DEBUG_LOG_OBSEVING_HANDLER
    debug() << "Processing payment claims, count: " << mPaymentClaims.size();
    #endif

    // Step 1: Process each claim based on status
    vector<pair<TransactionUUID, BlockNumber>> claimsToRemove;

    for (auto& [key, claim] : mPaymentClaims) {
        try {
            switch (claim->status()) {
                case ObservingPaymentClaim::NoInfo:
                    sendClaim(claim);
                    break;

                case ObservingPaymentClaim::Observing:
                    checkTransaction(claim);
                    break;

                case ObservingPaymentClaim::ParticipantsVotesPresent:
                    getParticipantsSignatures(claim);
                    break;

                case ObservingPaymentClaim::RejectedByObserving:
                    rejectTransaction(claim);
                    break;

                case ObservingPaymentClaim::Done:
                    // Mark for cleanup
                    claimsToRemove.push_back(key);
                    break;
            }
        } catch (const std::exception &e) {
            warning() << "Error processing claim " << claim->transactionUUID()
                      << ": " << e.what();
            // Continue processing other claims
        }
    }

    // Step 2: Cleanup completed claims
    for (const auto& key : claimsToRemove) {
        #ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Removing completed claim: " << key.first;
        #endif
        mPaymentClaims.erase(key);
    }

    // Step 3: Reschedule timer
    mPaymentClaimsTimer.expires_after(
        chrono::seconds(kPaymentClaimProcessingPeriodSeconds));
    #ifdef TESTS
    mPaymentClaimsTimer.expires_after(
        chrono::seconds(kPaymentClaimProcessingPeriodSecondsTests));
    #endif

    mPaymentClaimsTimer.async_wait([this](const boost::system::error_code &e) {
        if (e == boost::asio::error::operation_aborted) {
            return;
        }
        processPaymentClaims();
    });
}
```

#### sendClaim Implementation
**Purpose**: Submit claim to observer via RPC

**Algorithm**:
```cpp
void ObservingHandler::sendClaim(ObservingPaymentClaim::Shared claim) {
    #ifdef DEBUG_LOG_OBSEVING_HANDLER
    debug() << "Sending claim to observer: " << claim->transactionUUID();
    #endif

    if (mObservers.empty()) {
        warning() << "Cannot send claim: no observers configured";
        throw NotFoundError("No observers configured");
    }

    auto firstObserver = mObservers[0];

    try {
        // Step 1: Prepare participants array
        json::array_t participantsArray;
        for (const auto& [nodeId, publicKey] : claim->participantsPublicKeys()) {
            json participant = {
                {"index", nodeId},
                {"public_key", publicKey->toString()}
            };
            participantsArray.push_back(participant);
        }

        // Step 2: Prepare RPC request
        json request = {
            {"method", "RPCService.AcceptClaim"},
            {"params", json::array({
                {
                    {"transaction_uuid", boost::uuids::to_string(claim->transactionUUID())},
                    {"max_claim_block_number", claim->maxBlockNumberForClaiming()},
                    {"participants", participantsArray},
                    {"public_key", claim->publicKey()->toString()},
                    {"signature", claim->signature()->toString()}
                }
            })},
            {"id", 1}
        };

        // Step 3: Send request (using ObserverRPCClient pattern)
        tcp::resolver resolver(mIOCtx);
        auto endpoints = resolver.resolve(
            firstObserver->host(),
            to_string(firstObserver->port()));

        tcp::socket socket(mIOCtx);
        boost::asio::connect(socket, endpoints);

        string requestStr = request.dump() + "\n";
        boost::asio::write(socket, boost::asio::buffer(requestStr));

        // Step 4: Read response
        boost::asio::streambuf responseBuffer;
        boost::asio::read_until(socket, responseBuffer, '\n');

        std::istream responseStream(&responseBuffer);
        string responseLine;
        std::getline(responseStream, responseLine);

        json response = json::parse(responseLine);

        // Step 5: Check for errors
        if (response.contains("error") && !response["error"].is_null()) {
            string errorMsg = response["error"].is_string()
                ? response["error"].get<string>()
                : response["error"].dump();
            throw runtime_error("Observer RPC error: " + errorMsg);
        }

        // Step 6: Parse result
        if (!response.contains("result")) {
            throw runtime_error("Invalid RPC response: missing result");
        }

        auto result = response["result"];
        bool success = result.value("success", false);

        if (success) {
            // Update claim status
            claim->setStatus(ObservingPaymentClaim::Observing);
            info() << "Claim accepted by observer: " << claim->transactionUUID();
        } else {
            string message = result.value("message", "unknown error");
            warning() << "Observer rejected claim: " << message;
            throw runtime_error("Observer rejected claim: " + message);
        }

        socket.close();

    } catch (const std::exception &e) {
        error() << "Failed to send claim to observer "
                << firstObserver->fullAddress() << ": " << e.what();
        throw;
    }
}
```

#### checkTransaction Implementation
**Purpose**: Query observer for claim status

**Algorithm**:
```cpp
void ObservingHandler::checkTransaction(ObservingPaymentClaim::Shared claim) {
    #ifdef DEBUG_LOG_OBSEVING_HANDLER
    debug() << "Checking transaction status: " << claim->transactionUUID();
    #endif

    if (mObservers.empty()) {
        throw NotFoundError("No observers configured");
    }

    auto firstObserver = mObservers[0];

    try {
        // Step 1: Prepare RPC request
        json request = {
            {"method", "RPCService.GetClaimStatus"},
            {"params", json::array({
                {
                    {"transaction_uuid", boost::uuids::to_string(claim->transactionUUID())},
                    {"max_claim_block_number", claim->maxBlockNumberForClaiming()}
                }
            })},
            {"id", 1}
        };

        // Step 2: Send request and read response (same pattern as sendClaim)
        tcp::resolver resolver(mIOCtx);
        auto endpoints = resolver.resolve(
            firstObserver->host(),
            to_string(firstObserver->port()));

        tcp::socket socket(mIOCtx);
        boost::asio::connect(socket, endpoints);

        string requestStr = request.dump() + "\n";
        boost::asio::write(socket, boost::asio::buffer(requestStr));

        boost::asio::streambuf responseBuffer;
        boost::asio::read_until(socket, responseBuffer, '\n');

        std::istream responseStream(&responseBuffer);
        string responseLine;
        std::getline(responseStream, responseLine);

        json response = json::parse(responseLine);

        // Step 3: Check for errors
        if (response.contains("error") && !response["error"].is_null()) {
            string errorMsg = response["error"].is_string()
                ? response["error"].get<string>()
                : response["error"].dump();
            throw runtime_error("Observer RPC error: " + errorMsg);
        }

        // Step 4: Parse state
        if (!response.contains("result") || !response["result"].contains("state")) {
            throw runtime_error("Invalid RPC response: missing state");
        }

        string state = response["result"]["state"].get<string>();

        // Step 5: Update status based on state
        if (state == "not found") {
            claim->setStatus(ObservingPaymentClaim::NoInfo);
            info() << "Claim not found on observer, will retry: "
                   << claim->transactionUUID();
        } else if (state == "observing") {
            // No change, remains Observing
            #ifdef DEBUG_LOG_OBSEVING_HANDLER
            debug() << "Claim still observing: " << claim->transactionUUID();
            #endif
        } else if (state == "approved") {
            claim->setStatus(ObservingPaymentClaim::ParticipantsVotesPresent);
            info() << "Claim approved by observer: " << claim->transactionUUID();
        } else if (state == "rejected") {
            claim->setStatus(ObservingPaymentClaim::RejectedByObserving);
            info() << "Claim rejected by observer: " << claim->transactionUUID();
        } else {
            warning() << "Unknown claim state from observer: " << state;
        }

        socket.close();

    } catch (const std::exception &e) {
        error() << "Failed to check transaction status on observer "
                << firstObserver->fullAddress() << ": " << e.what();
        throw;
    }
}
```

#### getParticipantsSignatures Implementation
**Purpose**: Retrieve participant votes and notify payment transaction

**Algorithm**:
```cpp
void ObservingHandler::getParticipantsSignatures(ObservingPaymentClaim::Shared claim) {
    info() << "Getting participants signatures: " << claim->transactionUUID();

    if (mObservers.empty()) {
        throw NotFoundError("No observers configured");
    }

    auto firstObserver = mObservers[0];

    try {
        // Step 1: Prepare RPC request
        json request = {
            {"method", "RPCService.GetClaimVotes"},
            {"params", json::array({
                {
                    {"transaction_uuid", boost::uuids::to_string(claim->transactionUUID())},
                    {"max_claim_block_number", claim->maxBlockNumberForClaiming()}
                }
            })},
            {"id", 1}
        };

        // Step 2: Send request and read response
        tcp::resolver resolver(mIOCtx);
        auto endpoints = resolver.resolve(
            firstObserver->host(),
            to_string(firstObserver->port()));

        tcp::socket socket(mIOCtx);
        boost::asio::connect(socket, endpoints);

        string requestStr = request.dump() + "\n";
        boost::asio::write(socket, boost::asio::buffer(requestStr));

        boost::asio::streambuf responseBuffer;
        boost::asio::read_until(socket, responseBuffer, '\n');

        std::istream responseStream(&responseBuffer);
        string responseLine;
        std::getline(responseStream, responseLine);

        json response = json::parse(responseLine);

        // Step 3: Check for errors
        if (response.contains("error") && !response["error"].is_null()) {
            string errorMsg = response["error"].is_string()
                ? response["error"].get<string>()
                : response["error"].dump();
            throw runtime_error("Observer RPC error: " + errorMsg);
        }

        // Step 4: Parse votes array
        if (!response.contains("result") || !response["result"].contains("votes")) {
            throw runtime_error("Invalid RPC response: missing votes");
        }

        auto votesArray = response["result"]["votes"];

        // Step 5: Convert signatures from base64 to sphincs::Signature::Shared
        map<PaymentNodeID, sphincs::Signature::Shared> signaturesMap;

        for (const auto& vote : votesArray) {
            PaymentNodeID nodeId = vote["index"].get<PaymentNodeID>();
            string signatureBase64 = vote["signature"].get<string>();

            // Parse signature from base64
            auto signature = make_shared<sphincs::Signature>(signatureBase64);
            if (!signature->isValid()) {
                warning() << "Invalid signature from observer for node " << nodeId;
                continue;
            }

            signaturesMap[nodeId] = signature;
        }

        info() << "Retrieved " << signaturesMap.size()
               << " participant signatures from observer";

        // Step 6: Invoke signal to notify payment transaction
        mParticipantsVotesSignal(
            claim->transactionUUID(),
            claim->maxBlockNumberForClaiming(),
            signaturesMap);

        // Step 7: Update claim status to Done
        claim->setStatus(ObservingPaymentClaim::Done);

        socket.close();

    } catch (const std::exception &e) {
        error() << "Failed to get participants signatures from observer "
                << firstObserver->fullAddress() << ": " << e.what();
        throw;
    }
}
```

#### rejectTransaction Implementation
**Purpose**: Retrieve rejection signature and notify payment transaction

**Algorithm**:
```cpp
void ObservingHandler::rejectTransaction(ObservingPaymentClaim::Shared claim) {
    info() << "Getting rejection signature: " << claim->transactionUUID();

    if (mObservers.empty()) {
        throw NotFoundError("No observers configured");
    }

    auto firstObserver = mObservers[0];

    try {
        // Step 1: Prepare RPC request
        json request = {
            {"method", "RPCService.GetRejectionSignature"},
            {"params", json::array({
                {
                    {"transaction_uuid", boost::uuids::to_string(claim->transactionUUID())},
                    {"max_claim_block_number", claim->maxBlockNumberForClaiming()}
                }
            })},
            {"id", 1}
        };

        // Step 2: Send request and read response
        tcp::resolver resolver(mIOCtx);
        auto endpoints = resolver.resolve(
            firstObserver->host(),
            to_string(firstObserver->port()));

        tcp::socket socket(mIOCtx);
        boost::asio::connect(socket, endpoints);

        string requestStr = request.dump() + "\n";
        boost::asio::write(socket, boost::asio::buffer(requestStr));

        boost::asio::streambuf responseBuffer;
        boost::asio::read_until(socket, responseBuffer, '\n');

        std::istream responseStream(&responseBuffer);
        string responseLine;
        std::getline(responseStream, responseLine);

        json response = json::parse(responseLine);

        // Step 3: Check for errors
        if (response.contains("error") && !response["error"].is_null()) {
            string errorMsg = response["error"].is_string()
                ? response["error"].get<string>()
                : response["error"].dump();
            throw runtime_error("Observer RPC error: " + errorMsg);
        }

        // Step 4: Parse state and signature
        if (!response.contains("result")) {
            throw runtime_error("Invalid RPC response: missing result");
        }

        auto result = response["result"];
        string state = result.value("state", "");
        string signatureStr = result.value("signature", "");

        // Step 5: Process based on signature presence
        if (!signatureStr.empty()) {
            info() << "Received rejection signature from observer for "
                   << claim->transactionUUID();

            // Invoke signal to notify payment transaction
            mRejectTransactionSignal(
                claim->transactionUUID(),
                claim->maxBlockNumberForClaiming());

            // Update claim status to Done
            claim->setStatus(ObservingPaymentClaim::Done);
        } else {
            // No signature yet, keep status as RejectedByObserving
            // Will retry in next processing cycle
            #ifdef DEBUG_LOG_OBSEVING_HANDLER
            debug() << "No rejection signature available yet for "
                    << claim->transactionUUID();
            #endif
        }

        socket.close();

    } catch (const std::exception &e) {
        error() << "Failed to get rejection signature from observer "
                << firstObserver->fullAddress() << ": " << e.what();
        throw;
    }
}
```

**Key Points**:
- All RPC methods use first observer only (mObservers[0])
- Exceptions thrown on network errors or observer errors
- Signature parsing from base64 uses sphincs::Signature constructor
- Signal invocations use existing ObservingHandler signals
- Status transitions drive claim progression through state machine

### Error Handling Specifications

#### Error Conditions
1. **No observers configured**: Methods throw NotFoundError, claim processing skipped
2. **Network errors during RPC**: Methods throw exception, claim status unchanged, retry in next cycle
3. **Observer rejects claim**: sendClaim throws exception, claim status remains NoInfo, retry in next cycle
4. **Invalid signature from observer**: Log warning, skip signature, continue processing other signatures
5. **RPC timeout**: Network error, same as #2
6. **Invalid JSON response**: Parse exception, same as #2
7. **Claim not found on observer**: checkTransaction sets status to NoInfo for retry

## Implementation Plan
### This Iteration Timeline
- **Duration**: 3-4 weeks implementation + 1 week testing
- **Sprint Breakdown**:
  - Sprint 1 (Week 1): ObservingPaymentClaim class and claim storage
  - Sprint 2 (Week 2): Timer infrastructure and processing loop
  - Sprint 3 (Week 3): RPC method implementations (sendClaim, checkTransaction)
  - Sprint 4 (Week 4): RPC method implementations (getParticipantsSignatures, rejectTransaction)
  - Sprint 5 (Week 5): Unit testing and validation

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| ObservingPaymentClaim Complete | Week 1 | Class with status enum and fields | None | Low |
| Claim Storage Complete | Week 1 | Map and addPaymentClaim method | ObservingPaymentClaim | Low |
| Timer Infrastructure Complete | Week 2 | Timer with periodic processing | Claim storage | Medium |
| sendClaim and checkTransaction Complete | Week 3 | Basic RPC integration | Timer, ObserverRPCClient | High |
| getParticipantsSignatures and rejectTransaction Complete | Week 4 | Signal integration | Previous RPC methods | High |
| Unit Testing Complete | Week 5 | All components tested | All previous | Medium |

### Dependencies on Other Teams/Projects
- Observer service must be running and accessible via RPC

### Integration Points with Previous Work
- Uses existing ObserverRPCClient infrastructure
- Follows existing ObservingHandler patterns (timers, maps)
- Uses existing signal infrastructure

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++ and networking experience
- **Developers**: 1 developer for implementation support
- **QA Engineers**: 1 engineer for unit testing

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Observer RPC service unavailable | High | Medium | Exception handling with retry; status remains unchanged for next cycle |
| Signature parsing failures | Medium | Low | Validation after parsing; log warnings and skip invalid signatures |
| Timer performance with many claims | Medium | Low | Optimize processing loop; limit claim count or add batching |
| Map memory growth | Medium | Low | Automatic cleanup of Done claims; monitoring of claim count |
| Signal invocation failures | High | Low | Wrap signal calls in try-catch; log failures without crashing |
| Claims lost on restart (in-memory only) | Medium | Medium | Persistence and replay planned for later iteration; callers re-add claims after restart |
| Single observer without signature validation | High | Low | Accepted for this iteration; validation/quorum to be added in subsequent work |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Observer decisions incorrect | High | Low | Observer implementation responsibility; future multi-observer validation |
| Claims stuck in processing | Medium | Medium | Timer ensures periodic retry; monitoring of claim age |

## Testing Strategy
### Testing Approach
- All testing is unit-only (no integration/E2E tests)
- Tests built and executed in `build-tests`
- Use real objects following project patterns
- Test files in `tests/unit/observing/`

### Testing Best Practices
- **Real Objects Over Mocks**: Use real ObservingPaymentClaim instances
- **Exception Testing**: Verify correct exceptions thrown using EXPECT_THROW
- **Edge Case Coverage**: Test empty maps, null pointers, invalid states
- **State Machine Testing**: Verify all status transitions

#### Unit Tests: New Components

**1. ObservingPaymentClaim Tests** (`tests/unit/observing/ObservingPaymentClaimTest.cpp`):
- Constructor initializes all fields correctly
- Constructor sets status to NoInfo by default
- Getters return correct values for all fields
- Status setter updates status correctly
- Status enum values match specification (NoInfo=0, Observing=1, etc.)
- TransactionUUID and BlockNumber properly stored
- ParticipantsPublicKeys map correctly stored and retrieved
- PublicKey and Signature shared_ptr correctly stored

**Note on ObservingHandler Tests**: Tests for ObservingHandler components (claim map, timer infrastructure, RPC methods, signal integration) require integration testing infrastructure including IOCtx, StorageHandler, network communication, and running observer service. **These are NOT unit tests** and are explicitly out of scope for unit test tasks. Integration tests for RPC methods (sendClaim, checkTransaction, getParticipantsSignatures, rejectTransaction) will be covered in separate integration test tasks with appropriate infrastructure.

#### Regression Testing
- Scope: Ensure new code doesn't break existing ObservingHandler functionality
- Verify ObservingTransaction processing remains unchanged
- Validate existing timers (mClaimsTimer, mTransactionsTimer) still function

#### Execution in CI/Locally
- Build tests in `build-tests` directory
- Run unit test binary: `./build-tests/bin/unit_tests --gtest_filter=ObservingPayment*`
- All tests must pass before PRD completion

### Quality Gates
- All unit tests pass (100% pass rate)
- No memory leaks in claim processing (valgrind clean)
- Exception handling prevents timer failure in all scenarios
- Claim map cleanup verified to prevent unbounded growth
- Signal invocations work correctly with test handlers

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Minor feature addition (infrastructure enhancement)
- **Rollout Strategy**: Full deployment; no gradual rollout needed
- **Rollback Plan**: Revert to previous version if critical issues found

### Database Migrations
- No database migrations required (in-memory only)

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: Node operator documentation for observer configuration
- **Documentation Updates**:
  - ObservingHandler technical documentation
  - Observer integration guide
  - Payment transaction integration guide (future)

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**:
  - 100% of claims progress through state machine without manual intervention
  - Claims reach Done status within expected time (< 10 minutes under normal conditions)
  - Zero timer failures due to processing errors
- **Leading Indicators**: Successful unit test execution, claim map growth rate
- **Baseline Values**: N/A (new feature)
- **Target Values**:
  - 100% test pass rate
  - < 1000 claims in map at any time
  - < 5 second processing cycle for 100 claims

### Monitoring Plan
- **New Monitoring**: Claim count in map, claim age distribution, processing cycle duration
- **Enhanced Logging**: Debug logs for claim state transitions, RPC errors, signal invocations
- **Alerts**: Claim map size > 1000, processing cycle > 10 seconds, repeated RPC failures

### Review Schedule
- **Daily**: Development progress and unit test results
- **Weekly**: Code review and architecture validation
- **Post-Implementation Review**: Testing results and performance analysis

## Appendices
### Glossary
- **Ambiguous Transaction**: Payment transaction where node has signed but cannot obtain required signatures
- **Claim**: Request to observer for decision on ambiguous transaction
- **Observer**: Trusted third party making final decisions on transaction outcomes
- **Claim Status**: Current state in observer protocol (NoInfo, Observing, etc.)
- **Processing Cycle**: Single execution of claim processing loop

### References
- [Observer RPC Protocol](../../../observer/README.md)
- [ObservingHandler Implementation](../../../src/core/observing/ObservingHandler.h)
- [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
- [BaseExchangePaymentTransaction](../../../src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.h)

### Detailed Component Specifications

#### ObservingPaymentClaim Class
- **Location**: `src/core/observing/ObservingPaymentClaim.h` and `.cpp`
- **Header File**: ObservingPaymentClaim.h
- **Implementation File**: ObservingPaymentClaim.cpp
- **Namespace**: None (global)
- **Inheritance**: None

#### ObservingHandler Enhancement
- **Files Modified**:
  - `src/core/observing/ObservingHandler.h`
  - `src/core/observing/ObservingHandler.cpp`
- **New Methods**: addPaymentClaim, processPaymentClaims, sendClaim, checkTransaction, getParticipantsSignatures, rejectTransaction
- **New Fields**: mPaymentClaims, mPaymentClaimsTimer, kPaymentClaimProcessingPeriodSeconds

#### RPC Request Formats

**AcceptClaim Request**:
```json
{
  "method": "RPCService.AcceptClaim",
  "params": [{
    "transaction_uuid": "550e8400-e29b-41d4-a716-446655440000",
    "max_claim_block_number": 1000,
    "participants": [
      {"index": 0, "public_key": "base64_public_key_1"},
      {"index": 1, "public_key": "base64_public_key_2"}
    ],
    "public_key": "base64_node_public_key",
    "signature": "base64_signature_data"
  }],
  "id": 1
}
```

**GetClaimStatus Request**:
```json
{
  "method": "RPCService.GetClaimStatus",
  "params": [{
    "transaction_uuid": "550e8400-e29b-41d4-a716-446655440000",
    "max_claim_block_number": 1000
  }],
  "id": 1
}
```

**GetClaimVotes Request**:
```json
{
  "method": "RPCService.GetClaimVotes",
  "params": [{
    "transaction_uuid": "550e8400-e29b-41d4-a716-446655440000",
    "max_claim_block_number": 1000
  }],
  "id": 1
}
```

**GetRejectionSignature Request**:
```json
{
  "method": "RPCService.GetRejectionSignature",
  "params": [{
    "transaction_uuid": "550e8400-e29b-41d4-a716-446655440000",
    "max_claim_block_number": 1000
  }],
  "id": 1
}
```

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-12-09 | Claude Code | Initial draft for observer ambiguous transaction handling | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [11-path-rebuilding-on-inaccessible-nodes.md](11-path-rebuilding-on-inaccessible-nodes.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)
- **Observer Protocol**: [Observer README](../../../observer/README.md)
