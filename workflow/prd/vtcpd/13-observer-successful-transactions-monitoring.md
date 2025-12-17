# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Observer Successful Transactions Monitoring
- **PRD ID**: 13
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.0
- **Date**: 2025-12-11
- **Author(s)**: Claude Code, based on Architect's requirements
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-12-11
- **Previous PRD**: [12-observer-ambiguous-transaction-handling.md](12-observer-ambiguous-transaction-handling.md)
- **Related Documents**:
  - [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
  - [Observer README](../../../observer/README.md)
  - [ObservingHandler Implementation](../../../src/core/observing/ObservingHandler.h)
  - [PRD 12: Observer Ambiguous Transaction Handling](12-observer-ambiguous-transaction-handling.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This PRD introduces proactive monitoring of successfully executed payment transactions to detect and respond to observer claims. When observers receive claims about ambiguous transactions, nodes with finalized (fully signed) versions of those transactions can provide their signatures to observers, helping to resolve transaction states for all network participants.

### Current project state
- Payment transactions (BaseExchangePaymentTransaction, CoordinatorExchangePaymentTransaction, IntermediateNodeExchangePaymentTransaction, ReceiverExchangePaymentTransaction) execute multi-equivalent payments with commissions
- ObservingHandler manages ambiguous transaction claims from the node's perspective (PRD 12)
- Observers maintain blockchain of transaction claims and can finalize or reject them
- Nodes can submit claims when they encounter ambiguous transactions
- **Missing capability**: Nodes cannot proactively detect and respond to claims submitted by other participants

### This iteration's focus
- Implement periodic monitoring of successfully executed transactions
- Detect observer claims related to node's successful transactions
- Automatically submit finalized transaction signatures to observers
- Add database query method to retrieve transactions by block number criteria
- Ensure efficient processing with configurable limits and proper ordering

### Connection to overall vision
This completes the bidirectional observer integration: nodes can both submit their own ambiguous transactions to observers (PRD 12) AND help resolve other participants' ambiguous transactions by providing finalized signatures when available. This improves overall network reliability and reduces transaction uncertainty for all participants.

## Iteration Context
### Previous Iterations Summary
- **PRD 06**: Exchange Payment with Commissions - multi-equivalent payment execution
- **PRD 12**: Observer Ambiguous Transaction Handling - nodes submitting their own ambiguous transactions to observers
- **Completed Features**: Payment transactions execution, observer claim submission from node perspective, observer claim status monitoring

### Lessons Learned
- Timer-based periodic processing provides reliable asynchronous operation
- RPC error handling requires robust exception management
- Observer communication should continue despite individual request failures
- Efficient database queries are critical for periodic processing

### Current State Analysis
- **What's working well**: Observer RPC infrastructure for claim submission and monitoring, timer-based claim processing
- **Pain points identified**: Nodes with finalized transactions cannot help resolve claims submitted by other participants
- **Performance metrics**: N/A (new feature)

## Problem Statement
### Background
When a participant submits a claim to observers about an ambiguous transaction, other participants who successfully completed that transaction hold valuable information: the complete set of participant signatures that finalized the transaction. Currently, there is no mechanism for nodes to proactively detect such situations and provide their finalized transaction data to observers.

This creates unnecessary transaction rejections: if even one participant has the finalized version but cannot communicate it to observers, the transaction may be incorrectly rejected by timeout, harming all participants.

### Problem Description
**Who is affected**: All network participants, particularly those who successfully completed transactions but don't know that other participants encountered issues

**When and where**: When one participant has connectivity or timing issues preventing them from collecting all signatures, while other participants successfully completed the transaction

**Current limitations**:
- No periodic monitoring of successful transactions against observer claims
- No automatic detection of relevant claims for node's successful transactions
- No automatic submission of finalized signatures to help resolve claims
- Nodes only interact with observers reactively (when they have problems), not proactively (when they can help)

### Impact of not solving this problem
- Increased transaction rejection rate due to lack of signature availability
- Degraded user experience when transactions fail unnecessarily
- Reduced network reliability and trust
- Manual intervention required to resolve solvable transaction uncertainties

### Success Metrics
**Primary KPIs**:
- 100% of relevant claims (for node's successful transactions) are detected within one monitoring cycle after claim appears
- Finalized signatures are submitted to observers within one monitoring cycle after detection
- No performance degradation in ObservingHandler timer processing

**Target Values**:
- Monitoring cycle runs every 60 seconds (configurable constant)
- Processing up to 100 transactions per cycle (configurable constant)
- Detection and submission within the same or next monitoring cycle after claim appearance
- Database query executes efficiently for typical transaction volumes (performance validated outside unit tests)

## Goals
The primary goals of this iteration are to enable nodes to proactively help resolve observer claims by providing finalized transaction signatures.

*   **Goal 1: Implement periodic monitoring of successful transactions.**
    *   **Description:** Add timer-based processing that periodically checks successful transactions against observer claims.
    *   **Success Metric:** Timer executes every 60 seconds, processing transactions efficiently without blocking other operations.

*   **Goal 2: Detect relevant observer claims for node's transactions.**
    *   **Description:** Query observers to identify claims related to node's successfully executed transactions.
    *   **Success Metric:** 100% of relevant claims detected within one monitoring cycle after claim creation.

*   **Goal 3: Submit finalized signatures to help resolve claims.**
    *   **Description:** Automatically provide complete participant signatures to observers when claims are detected.
    *   **Success Metric:** Signatures submitted successfully within one monitoring cycle of detection, with proper error handling for RPC failures.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **Successful Transaction Monitoring Timer**: New periodic timer in ObservingHandler for monitoring node's successful transactions
2. **Observer Claim Bulk Query**: Query multiple transactions against observer claims in single RPC call
3. **Finalized Signature Submission**: Automatic submission of participant signatures for relevant claims
4. **Database Query Enhancement**: New method to retrieve transactions by block number and state criteria

#### Technical Infrastructure
- New timer and processing method in ObservingHandler
- RPC integration with RPCService.GetClaimStatuses (bulk query)
- RPC integration with RPCService.SubmitClaimVotes
- New database query method in PaymentTransactionsHandler interface
- SQLite and PostgreSQL implementations of new query method

#### Integration Points
- Integration with existing ObservingHandler timer infrastructure
- Usage of existing PaymentParticipantsVotesHandler for signature retrieval
- Usage of observer RPC protocol from observer/README.md
- Integration with existing payment transactions storage

### Explicitly Out of Scope
- Optimization of observer connection reliability (deferred to future task)
- Signature validation before submission (accepted as-is from database)
- Multi-observer support and failover logic
- Monitoring/metrics for submission success rates
- UI/dashboard for monitoring transaction assistance
- Historical analysis of claim resolution contributions
- Updating `observing_state` semantics or adding new DB states for monitoring lifecycle
- Reactive handling for finalized/rejected/unknown observer statuses beyond logging and continued polling

### Dependencies from Previous Iterations
- **PRD 06**: Payment transaction implementations with signature storage
- **PRD 12**: ObservingHandler infrastructure, observer RPC client, observer protocol
- **Existing database infrastructure**: PaymentTransactionsHandler, PaymentParticipantsVotesHandler

### Future Roadmap Impact
This iteration enables:
- **Observer connection optimization**: Retry logic, connection pooling, multi-observer failover
- **Metrics and monitoring**: Track how often node helps resolve claims
- **Advanced claim detection**: Predictive detection based on network conditions
- **Signature validation**: Verify signatures before submission to observers

## User Stories & Requirements

### User Personas
#### Primary User: Payment Transaction
- **Role**: Software component executing payment operations
- **Goals**: Maximize transaction success rate across network by providing assistance when possible
- **Pain Points**: Cannot help other participants resolve transaction uncertainties
- **Technical Proficiency**: N/A (software component)

#### Secondary User: Network Participant
- **Role**: Node operator or end user executing payments
- **Goals**: Reliable payment completion with minimal failures
- **Pain Points**: Transactions fail even when some participants completed successfully
- **Technical Proficiency**: Intermediate to Advanced

### Functional Requirements
#### New Features for This Iteration

1. **Successful Transaction Monitoring Timer**
   - **Description**: Periodic timer that monitors node's successful transactions and checks for relevant observer claims
   - **User Story**: As ObservingHandler, I need to periodically check if any of my successful transactions have claims submitted by other participants
   - **Rationale**: Enables proactive detection of opportunities to help resolve transaction uncertainties
   - **Builds Upon**: Existing timer pattern in ObservingHandler (mPaymentClaimsTimer, mTransactionsTimer)
   - **Acceptance Criteria**:
     - New timer `mSuccessfulTransactionsMonitorTimer` added to ObservingHandler
     - Timer period defined as class constant `kSuccessfulTransactionsMonitoringPeriodSeconds = 60`
     - Test mode constant `kSuccessfulTransactionsMonitoringPeriodSecondsTests = 10`
     - Timer started in ObservingHandler constructor
     - Timer callback method `monitorSuccessfulTransactions()` processes transactions
     - Timer reschedules after each processing cycle
     - Processing limit constant `kMaxTransactionsPerMonitoringCycle = 100`
     - Status polling continues each cycle until transaction ages out by `maximal_claiming_block_number`, even if observer reports a non-`observing` status, to ensure signatures are persisted on observer side
   - **Priority**: High
   - **Dependencies**: ObservingHandler infrastructure, timer support

2. **Block Number-Based Transaction Retrieval**
   - **Description**: Database query method to retrieve transactions based on block number criteria and state
   - **User Story**: As ObservingHandler, I need to efficiently retrieve transactions that are still within their claiming period
   - **Rationale**: Enables efficient filtering of transactions relevant for monitoring
   - **Builds Upon**: Existing PaymentTransactionsHandler interface and implementations
   - **Acceptance Criteria**:
     - New interface method in PaymentTransactionsHandler: `transactionsForObserverMonitoring(BlockNumber minBlockNumber, uint32_t limit)`
     - Method returns `vector<pair<TransactionUUID, BlockNumber>>` sorted by block number ascending
     - Filters: `maximal_claiming_block_number > minBlockNumber` AND `observing_state = 0`
     - Presence in the payment transactions table with `observing_state = 0` is treated as "successful transaction"; no additional success flags or signature completeness checks are required in this iteration
     - Implementation in PaymentTransactionsHandlerSQLite with proper SQL query
     - Implementation in PaymentTransactionsHandlerPostgreSQL with proper SQL query
     - Limit parameter restricts result count (for pagination/performance)
     - Results sorted by `maximal_claiming_block_number ASC` (process oldest first)
   - **Priority**: High
   - **Dependencies**: Database infrastructure, PaymentTransactionsHandler interface

3. **Observer Claim Bulk Detection**
   - **Description**: Query observer for claim statuses of multiple transactions in single RPC call
   - **User Story**: As ObservingHandler, I need to efficiently check which of my successful transactions have claims
   - **Rationale**: Bulk query reduces network overhead and improves performance
   - **Builds Upon**: Observer RPC client, RPCService.GetClaimStatuses method
   - **Acceptance Criteria**:
     - Construct RPC request with array of transaction queries
     - Each query contains: transaction_uuid, max_claim_block_number
     - Use first observer from mObservers (same as other RPC methods)
     - Parse response to extract claims with status "observing"
     - If a queried transaction UUID is absent in the response, treat it as "no claims" for this cycle; no submission attempted
     - Non-"observing" statuses are logged and skipped for submission but the transaction stays in periodic polling until claiming window expiry (no DB state changes in this iteration)
     - Handle empty response (no claims found) gracefully
     - On RPC error: log error and continue (don't block monitoring cycle)
     - Located in new method `checkForRelevantClaims()` in ObservingHandler.cpp
   - **Priority**: High
   - **Dependencies**: Observer RPC client, observer service availability

4. **Finalized Signature Submission**
   - **Description**: Submit complete participant signatures to observer for claims
   - **User Story**: As ObservingHandler, I need to provide finalized transaction signatures to help resolve claims
   - **Rationale**: Enables transaction resolution when node has complete signature set
   - **Builds Upon**: PaymentParticipantsVotesHandler, RPCService.SubmitClaimVotes
   - **Acceptance Criteria**:
     - For each detected claim with status "observing":
       - Retrieve participant signatures via PaymentParticipantsVotesHandler::participantsSignatures(); the "full" set equals all signatures currently stored for the transaction UUID (no completeness validation vs participant list in this iteration)
       - Construct RPC request with votes array: [{index: PaymentNodeID, signature: string}]
       - Set public_key field to empty string (TODO: populate in future)
       - Set signature field to empty string (TODO: populate in future)
       - Submit to observer via RPCService.SubmitClaimVotes
     - Use first observer from mObservers
     - On RPC error: log error and continue to next transaction
     - On empty signatures: log warning and skip submission
     - Located in new method `submitFinalizedSignatures()` in ObservingHandler.cpp
   - **Priority**: High
   - **Dependencies**: PaymentParticipantsVotesHandler, Observer RPC client

5. **Monitoring Cycle Processing**
   - **Description**: Main processing loop that coordinates monitoring activities
   - **User Story**: As ObservingHandler, I need to orchestrate the monitoring, detection, and submission process
   - **Rationale**: Provides coordinated execution of all monitoring activities
   - **Builds Upon**: Existing processing pattern in ObservingHandler
   - **Acceptance Criteria**:
     - Method `monitorSuccessfulTransactions()` in ObservingHandler:
       1. Call `getActualBlockNumber()` to get current observer block number
       2. Call `transactionsForObserverMonitoring(currentBlockNumber, kMaxTransactionsPerMonitoringCycle)`
       3. If no transactions, skip to timer rescheduling
       4. Call `checkForRelevantClaims()` with transaction list
       5. For each relevant claim, call `submitFinalizedSignatures()`
       6. Reschedule timer for next cycle
     - Observer status is re-queried every cycle until transaction ages out by block number; even when a prior response was non-"observing" (e.g., "finalized"/"rejected"/"unknown"), no DB state updates occur in this iteration and no re-submission is attempted unless status returns to "observing"
     - No explicit retry/backoff per transaction; periodic scheduling provides repeated attempts
     - Proper exception handling: catch exceptions, log errors, continue processing
     - Debug logging under `#ifdef DEBUG_LOG_OBSEVING_HANDLER`
     - No exceptions propagate outside method (timer continues functioning)
   - **Priority**: High
   - **Dependencies**: All above features

#### Non-Functional Requirements

### Non-Functional Requirements
#### Performance
- Database query executes efficiently for up to 1000 transactions (performance validation handled separately from unit tests)
- Processing cycle completes within 10 seconds for 100 transactions
- RPC request timeout: 10 seconds per request
- Timer period: 60 seconds (configurable via class constants)
- Memory overhead: < 5MB per monitoring cycle

#### Security
- No validation of retrieved signatures in this iteration (accepted from database as-is)
- Observer responses are submitted without verification (trust database content)
- Use first observer only (no multi-observer verification)
- TODO markers for future security enhancements (public_key, signature authentication)

#### Scalability
- Support up to 100 transactions per monitoring cycle (configurable limit)
- Handle observer response delays up to timeout period
- Process transactions efficiently with proper ordering (oldest first)
- Database query optimized with proper indexing and filtering

#### Reliability
- Automatic continuation on RPC failures (log and continue)
- Exception handling prevents timer failure
- Empty result sets handled gracefully
- Network timeouts don't block processing cycle

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: ObservingHandler with separate timers for different monitoring activities
- **Proposed Changes**: Add new timer for successful transaction monitoring with independent processing cycle
- **Backwards Compatibility**: No changes to existing functionality; purely additive
- **Migration Requirements**: None

### Technology Stack Updates
#### New Technologies/Libraries
- No new external libraries required
- Uses existing: ObserverRPCClient, JSON library (nlohmann/json), database handlers

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- ObservingHandler integrates with RPCService.GetClaimStatuses (bulk query)
- ObservingHandler integrates with RPCService.SubmitClaimVotes
- New database query method in PaymentTransactionsHandler implementations

#### Modified Integrations
- None

### Data Requirements
#### Data Models

##### PaymentTransactionsHandler Interface Enhancement
**Purpose**: Add method to retrieve transactions by block number criteria

**New Method Signature**:
```cpp
virtual vector<pair<TransactionUUID, BlockNumber>> transactionsForObserverMonitoring(
    BlockNumber minBlockNumber,
    uint32_t limit) = 0;
```

**Method Behavior**:
- Query: `WHERE maximal_claiming_block_number > minBlockNumber AND observing_state = 0`
- Order: `ORDER BY maximal_claiming_block_number ASC`
- Limit: `LIMIT limit`
- Returns: Vector of (TransactionUUID, maximal_claiming_block_number) pairs

##### ObservingHandler Enhancement
**Purpose**: Add monitoring infrastructure for successful transactions

**Added Fields**:
```cpp
// Timer for successful transaction monitoring
as::steady_timer mSuccessfulTransactionsMonitorTimer;

// Timer period constant
static constexpr uint32_t kSuccessfulTransactionsMonitoringPeriodSeconds = 60;
#ifdef TESTS
static constexpr uint32_t kSuccessfulTransactionsMonitoringPeriodSecondsTests = 10;
#endif

// Processing limit constant
static constexpr uint32_t kMaxTransactionsPerMonitoringCycle = 100;
```

**Added Methods**:
```cpp
// Main monitoring cycle
void monitorSuccessfulTransactions();

// Helper methods
void checkForRelevantClaims(
    const vector<pair<TransactionUUID, BlockNumber>>& transactions);

void submitFinalizedSignatures(
    const TransactionUUID& transactionUUID,
    BlockNumber maxBlockNumber);
```

#### Data Storage
- No new persistent storage required
- Uses existing payment_transactions table
- Uses existing payment_participants_votes table

#### Data Migration
- No migration required (uses existing tables)

### Algorithm Specifications

#### Successful Transaction Monitoring Algorithm
**Purpose**: Monitor successful transactions and submit signatures for relevant claims

**Algorithm**:
```cpp
void ObservingHandler::monitorSuccessfulTransactions() {
    #ifdef DEBUG_LOG_OBSEVING_HANDLER
    debug() << "Starting successful transactions monitoring cycle";
    #endif

    try {
        // Step 1: Get current observer block number
        BlockNumber currentBlockNumber;
        try {
            currentBlockNumber = getActualBlockNumber();
        } catch (const std::exception &e) {
            error() << "Failed to get actual block number: " << e.what();
            // Reschedule and return
            rescheduleSuccessfulTransactionsMonitor();
            return;
        }

        #ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Current observer block number: " << currentBlockNumber;
        #endif

        // Step 2: Retrieve transactions for monitoring
        auto ioTransaction = mStorageHandler->beginTransaction();
        auto transactions = ioTransaction->paymentTransactionsHandler()
            ->transactionsForObserverMonitoring(
                currentBlockNumber,
                kMaxTransactionsPerMonitoringCycle);

        #ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Retrieved " << transactions.size()
                << " transactions for monitoring";
        #endif

        if (transactions.empty()) {
            // No transactions to monitor
            rescheduleSuccessfulTransactionsMonitor();
            return;
        }

        // Step 3: Check for relevant claims
        checkForRelevantClaims(transactions);

    } catch (const std::exception &e) {
        error() << "Error in successful transactions monitoring: " << e.what();
        // Continue - don't let exceptions break the timer
    }

    // Step 4: Reschedule timer
    rescheduleSuccessfulTransactionsMonitor();
}

void ObservingHandler::rescheduleSuccessfulTransactionsMonitor() {
    #ifdef TESTS
    mSuccessfulTransactionsMonitorTimer.expires_after(
        chrono::seconds(kSuccessfulTransactionsMonitoringPeriodSecondsTests));
    #else
    mSuccessfulTransactionsMonitorTimer.expires_after(
        chrono::seconds(kSuccessfulTransactionsMonitoringPeriodSeconds));
    #endif

    mSuccessfulTransactionsMonitorTimer.async_wait(
        [this](const boost::system::error_code &e) {
            if (e == boost::asio::error::operation_aborted) {
                return;
            }
            monitorSuccessfulTransactions();
        });
}
```

#### Relevant Claims Detection Algorithm
**Purpose**: Query observer for claims related to node's transactions

**Algorithm**:
```cpp
void ObservingHandler::checkForRelevantClaims(
    const vector<pair<TransactionUUID, BlockNumber>>& transactions) {

    #ifdef DEBUG_LOG_OBSEVING_HANDLER
    debug() << "Checking for relevant claims: " << transactions.size()
            << " transactions";
    #endif

    if (mObservers.empty()) {
        warning() << "Cannot check claims: no observers configured";
        return;
    }

    auto firstObserver = mObservers[0];

    try {
        // Step 1: Prepare bulk claims query
        json::array_t claimsArray;
        for (const auto& [uuid, blockNumber] : transactions) {
            json claimQuery = {
                {"transaction_uuid", boost::uuids::to_string(uuid)},
                {"max_claim_block_number", blockNumber}
            };
            claimsArray.push_back(claimQuery);
        }

        json request = {
            {"method", "RPCService.GetClaimStatuses"},
            {"params", json::array({
                {{"claims", claimsArray}}
            })},
            {"id", 1}
        };

        // Step 2: Send RPC request
        string requestStr = request.dump() + "\n";

        auto &ioCtx = static_cast<IOCtx &>(
            mSuccessfulTransactionsMonitorTimer.get_executor().context());
        tcp::resolver resolver(ioCtx);
        boost::system::error_code errorCode;

        auto endpoints = resolver.resolve(
            firstObserver->host(),
            to_string(firstObserver->port()),
            errorCode);

        if (errorCode) {
            throw errorCode;
        }

        tcp::socket socket(ioCtx);
        boost::asio::connect(socket, endpoints, errorCode);

        if (errorCode) {
            throw errorCode;
        }

        #ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Sending bulk claims query to observer";
        #endif

        boost::asio::write(socket, boost::asio::buffer(requestStr));

        boost::asio::streambuf responseBuffer;
        boost::asio::read_until(socket, responseBuffer, '\n', errorCode);

        if (errorCode && errorCode != boost::asio::error::eof) {
            throw errorCode;
        }

        std::istream responseStream(&responseBuffer);
        string responseLine;
        std::getline(responseStream, responseLine);

        #ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Received bulk claims response";
        #endif

        json response = json::parse(responseLine);

        // Step 3: Check for RPC errors
        if (response.contains("error") && !response["error"].is_null()) {
            string errorMsg = response["error"].is_string()
                ? response["error"].get<string>()
                : response["error"].dump();
            throw runtime_error("Observer RPC error: " + errorMsg);
        }

        // Step 4: Parse response
        if (!response.contains("result") || !response["result"].contains("statuses")) {
            warning() << "Invalid RPC response: missing statuses";
            socket.close();
            return;
        }

        auto statusesArray = response["result"]["statuses"];

        if (!statusesArray.is_array()) {
            warning() << "Invalid RPC response: statuses is not array";
            socket.close();
            return;
        }

        info() << "Found " << statusesArray.size() << " relevant claims";

        // Step 5: Process each relevant claim
        for (const auto& claimStatus : statusesArray) {
            try {
                string uuidStr = claimStatus.at("transaction_uuid").get<string>();
                BlockNumber blockNumber = claimStatus.at("max_claim_block_number")
                    .get<BlockNumber>();
                string state = claimStatus.at("state").get<string>();

                // Only process claims in "observing" state
                if (state == "observing") {
                    TransactionUUID uuid(uuidStr);
                    submitFinalizedSignatures(uuid, blockNumber);
                }
            } catch (const std::exception &e) {
                warning() << "Error processing claim status: " << e.what();
                continue;
            }
        }

        socket.close();

    } catch (const std::exception &e) {
        error() << "Failed to check for relevant claims from observer "
                << firstObserver->fullAddress() << ": " << e.what();
        // Don't throw - continue monitoring
    }
}
```

#### Finalized Signatures Submission Algorithm
**Purpose**: Submit participant signatures to observer for claim resolution

**Algorithm**:
```cpp
void ObservingHandler::submitFinalizedSignatures(
    const TransactionUUID& transactionUUID,
    BlockNumber maxBlockNumber) {

    info() << "Submitting finalized signatures for transaction: "
           << transactionUUID;

    try {
        // Step 1: Retrieve participant signatures from database
        auto ioTransaction = mStorageHandler->beginTransaction();
        auto participantsSignatures = ioTransaction->paymentParticipantsVotesHandler()
            ->participantsSignatures(transactionUUID);

        if (participantsSignatures.empty()) {
            warning() << "No participant signatures found for transaction: "
                     << transactionUUID;
            return;
        }

        #ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Retrieved " << participantsSignatures.size()
                << " participant signatures";
        #endif

        // Step 2: Prepare votes array
        json::array_t votesArray;
        for (const auto& [nodeId, signature] : participantsSignatures) {
            json vote = {
                {"index", nodeId},
                {"signature", signature->toString()}
            };
            votesArray.push_back(vote);
        }

        // Step 3: Prepare RPC request
        json request = {
            {"method", "RPCService.SubmitClaimVotes"},
            {"params", json::array({
                {
                    {"transaction_uuid", boost::uuids::to_string(transactionUUID)},
                    {"max_claim_block_number", maxBlockNumber},
                    {"votes", votesArray},
                    {"public_key", ""},  // TODO: populate with node's public key
                    {"signature", ""}    // TODO: populate with submission signature
                }
            })},
            {"id", 1}
        };

        if (mObservers.empty()) {
            warning() << "Cannot submit votes: no observers configured";
            return;
        }

        auto firstObserver = mObservers[0];

        // Step 4: Send RPC request
        string requestStr = request.dump() + "\n";

        auto &ioCtx = static_cast<IOCtx &>(
            mSuccessfulTransactionsMonitorTimer.get_executor().context());
        tcp::resolver resolver(ioCtx);
        boost::system::error_code errorCode;

        auto endpoints = resolver.resolve(
            firstObserver->host(),
            to_string(firstObserver->port()),
            errorCode);

        if (errorCode) {
            throw errorCode;
        }

        tcp::socket socket(ioCtx);
        boost::asio::connect(socket, endpoints, errorCode);

        if (errorCode) {
            throw errorCode;
        }

        #ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Sending votes submission request";
        #endif

        boost::asio::write(socket, boost::asio::buffer(requestStr));

        boost::asio::streambuf responseBuffer;
        boost::asio::read_until(socket, responseBuffer, '\n', errorCode);

        if (errorCode && errorCode != boost::asio::error::eof) {
            throw errorCode;
        }

        std::istream responseStream(&responseBuffer);
        string responseLine;
        std::getline(responseStream, responseLine);

        #ifdef DEBUG_LOG_OBSEVING_HANDLER
        debug() << "Received votes submission response";
        #endif

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
            info() << "Successfully submitted finalized signatures for: "
                   << transactionUUID;
        } else {
            string message = result.value("message", "unknown error");
            warning() << "Observer rejected signature submission: " << message;
        }

        socket.close();

    } catch (const std::exception &e) {
        error() << "Failed to submit finalized signatures: " << e.what();
        // Don't throw - continue monitoring other transactions
    }
}
```

#### Database Query Implementation (SQLite)
**Purpose**: Retrieve transactions for observer monitoring

**SQL Query**:
```cpp
vector<pair<TransactionUUID, BlockNumber>>
PaymentTransactionsHandlerSQLite::transactionsForObserverMonitoring(
    BlockNumber minBlockNumber,
    uint32_t limit) {

    vector<pair<TransactionUUID, BlockNumber>> result;

    string query = "SELECT uuid, maximal_claiming_block_number FROM " + mTableName +
                   " WHERE maximal_claiming_block_number > ? AND observing_state = 0 " +
                   " ORDER BY maximal_claiming_block_number ASC LIMIT ?";

    SQLiteStatementRAII stmt(mDataBase, query.c_str());

    // Bind minBlockNumber
    int rc = sqlite3_bind_blob(stmt.get(), 1, &minBlockNumber,
                               sizeof(BlockNumber), SQLITE_STATIC);
    if (rc != SQLITE_OK) {
        throw IOError("Failed to bind minBlockNumber: " +
                     string(sqlite3_errmsg(mDataBase)));
    }

    // Bind limit
    rc = sqlite3_bind_int(stmt.get(), 2, limit);
    if (rc != SQLITE_OK) {
        throw IOError("Failed to bind limit: " +
                     string(sqlite3_errmsg(mDataBase)));
    }

    // Execute and collect results
    while (sqlite3_step(stmt.get()) == SQLITE_ROW) {
        TransactionUUID uuid((uint8_t*)sqlite3_column_blob(stmt.get(), 0));

        auto blockNumberBytes = sqlite3_column_blob(stmt.get(), 1);
        BlockNumber blockNumber;
        memcpy(&blockNumber, blockNumberBytes, sizeof(BlockNumber));

        result.emplace_back(uuid, blockNumber);
    }

    #ifdef STORAGE_HANDLER_DEBUG_LOG
    info() << "Retrieved " << result.size()
           << " transactions for observer monitoring";
    #endif

    return result;
}
```

#### Database Query Implementation (PostgreSQL)
**Purpose**: Retrieve transactions for observer monitoring

**SQL Query**:
```cpp
vector<pair<TransactionUUID, BlockNumber>>
PaymentTransactionsHandlerPostgreSQL::transactionsForObserverMonitoring(
    BlockNumber minBlockNumber,
    uint32_t limit) {

    vector<pair<TransactionUUID, BlockNumber>> result;

    const string query =
        "SELECT uuid, maximal_claiming_block_number FROM " + mTableName +
        " WHERE maximal_claiming_block_number > $1 AND observing_state = 0 " +
        " ORDER BY maximal_claiming_block_number ASC LIMIT $2";

    const int kParams = 2;
    const char *params[kParams];
    int lengths[kParams];
    int formats[kParams] = {1, 0};  // binary for BlockNumber, text for limit

    // Bind minBlockNumber (binary)
    params[0] = reinterpret_cast<const char*>(&minBlockNumber);
    lengths[0] = sizeof(BlockNumber);

    // Bind limit (text)
    string limitStr = to_string(limit);
    params[1] = limitStr.c_str();
    lengths[1] = 0;

    PGresult *res = PQexecParams(mDataBase, query.c_str(), kParams,
                                 nullptr, params, lengths, formats, 0);
    checkTuples(mDataBase, res, "transactionsForObserverMonitoring");

    int rows = PQntuples(res);
    for (int i = 0; i < rows; ++i) {
        const unsigned char *uuidBytes =
            reinterpret_cast<const unsigned char*>(PQgetvalue(res, i, 0));
        const unsigned char *blockBytes =
            reinterpret_cast<const unsigned char*>(PQgetvalue(res, i, 1));

        TransactionUUID uuid(uuidBytes);
        BlockNumber blockNumber;
        memcpy(&blockNumber, blockBytes, sizeof(BlockNumber));

        result.emplace_back(uuid, blockNumber);
    }

    PQclear(res);

    #ifdef STORAGE_HANDLER_DEBUG_LOG
    info() << "Retrieved " << result.size()
           << " transactions for observer monitoring";
    #endif

    return result;
}
```

### Error Handling Specifications

#### Error Conditions
1. **No observers configured**: Log warning, skip monitoring cycle gracefully
2. **getActualBlockNumber() fails**: Log error, reschedule timer, skip current cycle
3. **Database query fails**: Exception propagates, logged, timer continues
4. **RPC request to observer fails**: Log error, continue to next transaction
5. **Observer rejects submission**: Log warning, continue to next transaction
6. **Empty participant signatures**: Log warning, skip submission
7. **Invalid JSON response**: Log error, continue monitoring
8. **Network timeout**: Log error, continue to next transaction

## Implementation Plan
### This Iteration Timeline
- **Duration**: 2-3 weeks implementation + 1 week testing
- **Sprint Breakdown**:
  - Sprint 1 (Week 1): Database query method implementation and testing
  - Sprint 2 (Week 2): Timer infrastructure and monitoring cycle
  - Sprint 3 (Week 3): RPC integration and signature submission
  - Sprint 4 (Week 4): Unit testing and validation

### Iteration Milestones
| Milestone | Description | Dependencies | Risk Level |
|-----------|-------------|--------------|------------|
| Database Query Complete | New method in PaymentTransactionsHandler with SQLite and PostgreSQL implementations | None | Low |
| Timer Infrastructure Complete | New timer and basic monitoring cycle | Database query | Low |
| Claim Detection Complete | Bulk claim query integration | Timer infrastructure | Medium |
| Signature Submission Complete | Full signature submission flow | Claim detection | Medium |
| Unit Testing Complete | All database query tests passing | All implementations | Low |

### Dependencies on Other Teams/Projects
- Observer service must be running and accessible via RPC
- Observer must support RPCService.GetClaimStatuses and RPCService.SubmitClaimVotes

### Integration Points with Previous Work
- Uses existing ObservingHandler infrastructure (PRD 12)
- Uses existing PaymentTransactionsHandler interface
- Uses existing PaymentParticipantsVotesHandler for signature retrieval
- Uses existing observer RPC client

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++ and database experience
- **Developers**: 1 developer for implementation support
- **QA Engineers**: 1 engineer for unit testing

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Observer RPC service unavailable | Medium | Medium | Exception handling with graceful continuation; monitoring cycle continues |
| Large number of transactions | Medium | Low | Configurable limit (kMaxTransactionsPerMonitoringCycle = 100); process oldest first |
| Database query performance | Medium | Low | Proper indexing on maximal_claiming_block_number; tested with large datasets |
| Timer performance degradation | Low | Low | Separate timer from existing timers; independent processing |
| Network timeout during submission | Low | Medium | Individual transaction timeouts; continue to next transaction on failure |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Delayed signature submission | Medium | Low | 60-second monitoring cycle ensures timely detection |
| Observer decisions incorrect | High | Low | Observer implementation responsibility; future multi-observer validation |

## Testing Strategy
### Testing Approach
- All testing is unit-only for database layer
- Integration testing with observers is out of scope
- Tests built and executed in `build-tests`
- Use real objects following project patterns

### Testing Best Practices
- **Real Objects Over Mocks**: Use real database instances for testing
- **Exception Testing**: Verify correct exceptions thrown using EXPECT_THROW
- **Edge Case Coverage**: Test boundary conditions, empty results, large datasets
- **Performance Testing**: Verify query performance with various dataset sizes

#### Unit Tests: Database Query Method

**SQLite Tests** (`tests/unit/sqlite/PaymentTransactionsHandlerSQLiteTest.cpp`):

1. **BasicRetrieval_ReturnsMatchingTransactions**
   - Setup: Insert 5 transactions with blockNumbers: 50, 100, 150, 200, 250 (all state=0)
   - Execute: transactionsForObserverMonitoring(100, 10)
   - Verify: Returns 3 transactions (150, 200, 250) in ascending order

2. **StateFiltering_OnlyReturnsUncertainState**
   - Setup: Insert transactions with same blockNumber but different states (0, 1, 2)
   - Execute: transactionsForObserverMonitoring(0, 10)
   - Verify: Returns only transactions with state=0

3. **LimitParameter_RespectsLimit**
   - Setup: Insert 10 transactions with blockNumbers 10-100 step 10 (all state=0)
   - Execute: transactionsForObserverMonitoring(0, 3)
   - Verify: Returns exactly 3 transactions with smallest blockNumbers (10, 20, 30)

4. **AscendingOrder_OldestFirst**
   - Setup: Insert transactions with blockNumbers: 30, 10, 20
   - Execute: transactionsForObserverMonitoring(0, 10)
   - Verify: Returns in order: 10, 20, 30

5. **EmptyResult_NoMatchingTransactions**
   - Setup: Insert transactions with blockNumbers all <= 100
   - Execute: transactionsForObserverMonitoring(100, 10)
   - Verify: Returns empty vector

6. **BoundaryCondition_ExactBlockNumber**
   - Setup: Insert transaction with blockNumber = 100
   - Execute: transactionsForObserverMonitoring(100, 10)
   - Verify: Returns empty (> not >=)

7. **ZeroLimit_ReturnsEmpty**
   - Setup: Insert transactions
   - Execute: transactionsForObserverMonitoring(0, 0)
   - Verify: Returns empty vector

8. **LargeDataset_ReturnsExpectedRange**
   - Setup: Insert 200 transactions
   - Execute: transactionsForObserverMonitoring(50, 100)
   - Verify: Returns correct 100 transactions (block numbers 51-150)

**PostgreSQL Integration Tests** (`tests/storage/integration/postgresql/PaymentTransactionsHandlerPostgreSQLIntegrationTest.cpp`):

Same test cases as SQLite, adapted for PostgreSQL integration test environment:

1. **BasicRetrieval_ReturnsMatchingTransactions**
2. **StateFiltering_OnlyReturnsUncertainState**
3. **LimitParameter_RespectsLimit**
4. **AscendingOrder_OldestFirst**
5. **EmptyResult_NoMatchingTransactions**
6. **BoundaryCondition_ExactBlockNumber**
7. **ZeroLimit_ReturnsEmpty**
8. **LargeDataset_ReturnsExpectedRange**

#### Regression Testing
- Scope: Ensure new code doesn't break existing functionality
- Verify existing PaymentTransactionsHandler methods still work correctly
- Validate existing ObservingHandler timers (mPaymentClaimsTimer, etc.) still function
- Confirm no performance degradation in existing payment transaction operations

#### Execution in CI/Locally
- Build tests in `build-tests` directory
- Run SQLite unit tests: `./build-tests/bin/unit_tests --gtest_filter=*PaymentTransactionsHandlerSQLite*transactionsForObserverMonitoring*`
- Run PostgreSQL integration tests: `./build-tests/bin/postgresql_integration_tests --gtest_filter=*PaymentTransactionsHandlerPostgreSQL*transactionsForObserverMonitoring*`
- All tests must pass before PRD completion

### Quality Gates
- All unit tests pass (100% pass rate)
- Database query correctness verified on large dataset; performance validated separately
- No memory leaks in query execution (valgrind clean)
- Proper exception handling verified in all error scenarios
- SQL queries optimized with proper indexing

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Minor feature addition (infrastructure enhancement)
- **Rollout Strategy**: Full deployment; no gradual rollout needed
- **Rollback Plan**: Revert to previous version if critical issues found

### Database Migrations
- No database migrations required (uses existing tables and indexes)

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: Node operator documentation for observer integration
- **Documentation Updates**:
  - ObservingHandler technical documentation
  - Observer integration guide
  - Database query optimization guide

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**:
  - 100% of relevant claims detected within one monitoring cycle
  - Finalized signatures submitted within 120 seconds of claim creation
  - Zero timer failures due to processing errors
- **Leading Indicators**: Successful test execution, query performance benchmarks
- **Baseline Values**: N/A (new feature)
- **Target Values**:
  - 100% test pass rate
  - Database query executes efficiently (benchmarks tracked outside unit tests)
  - Processing cycle < 10 seconds for 100 transactions
  - No memory leaks

### Monitoring Plan
- **New Monitoring**: Monitoring cycle duration, transaction processing count, RPC success/failure rates
- **Enhanced Logging**: Debug logs for claim detection, signature submission, processing cycle statistics
- **Alerts**: Processing cycle > 30 seconds, repeated RPC failures, database query > 1 second

### Review Schedule
- **Daily**: Development progress and unit test results
- **Weekly**: Code review and performance validation
- **Post-Implementation Review**: Testing results and performance analysis

## Appendices
### Glossary
- **Successful Transaction**: Payment transaction that completed with all participant signatures collected
- **Observer Claim**: Request to observer for decision on ambiguous transaction
- **Finalized Signatures**: Complete set of participant signatures that finalize a transaction
- **Monitoring Cycle**: Single execution of successful transaction monitoring loop
- **Claiming Period**: Time window during which transaction can be claimed (up to maxBlockNumberForClaiming)

### References
- [Observer RPC Protocol](../../../observer/README.md)
- [ObservingHandler Implementation](../../../src/core/observing/ObservingHandler.h)
- [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
- [PRD 12: Observer Ambiguous Transaction Handling](12-observer-ambiguous-transaction-handling.md)
- [PaymentTransactionsHandler Interface](../../../src/core/io/storage/interfaces/PaymentTransactionsHandler.h)

### Detailed Component Specifications

#### Timer Initialization in ObservingHandler Constructor
```cpp
// Add to constructor after existing timer initialization
#ifdef TESTS
mSuccessfulTransactionsMonitorTimer.expires_after(
    chrono::seconds(kSuccessfulTransactionsMonitoringPeriodSecondsTests));
#else
mSuccessfulTransactionsMonitorTimer.expires_after(
    chrono::seconds(kSuccessfulTransactionsMonitoringPeriodSeconds));
#endif

mSuccessfulTransactionsMonitorTimer.async_wait(
    [this](const boost::system::error_code &e) {
        if (e == boost::asio::error::operation_aborted) {
            return;
        }
        monitorSuccessfulTransactions();
    });
```

#### RPC Request Format Examples

**GetClaimStatuses Request** (Bulk Query):
```json
{
  "method": "RPCService.GetClaimStatuses",
  "params": [{
    "claims": [
      {
        "transaction_uuid": "550e8400-e29b-41d4-a716-446655440000",
        "max_claim_block_number": 1000
      },
      {
        "transaction_uuid": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
        "max_claim_block_number": 1100
      }
    ]
  }],
  "id": 1
}
```

**GetClaimStatuses Response**:
```json
{
  "result": {
    "statuses": [
      {
        "transaction_uuid": "550e8400-e29b-41d4-a716-446655440000",
        "max_claim_block_number": 1000,
        "state": "observing"
      }
    ]
  },
  "error": null,
  "id": 1
}
```

**SubmitClaimVotes Request**:
```json
{
  "method": "RPCService.SubmitClaimVotes",
  "params": [{
    "transaction_uuid": "550e8400-e29b-41d4-a716-446655440000",
    "max_claim_block_number": 1000,
    "votes": [
      {"index": 0, "signature": "base64_signature_1"},
      {"index": 1, "signature": "base64_signature_2"},
      {"index": 2, "signature": "base64_signature_3"}
    ],
    "public_key": "",
    "signature": ""
  }],
  "id": 1
}
```

**SubmitClaimVotes Response**:
```json
{
  "result": {
    "success": true,
    "message": "votes submitted successfully"
  },
  "error": null,
  "id": 1
}
```

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-12-11 | Claude Code | Initial draft for observer successful transactions monitoring | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [12-observer-ambiguous-transaction-handling.md](12-observer-ambiguous-transaction-handling.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)
- **Observer Protocol**: [Observer README](../../../observer/README.md)
