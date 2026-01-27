# Project Requirements Document (PRD)

## Document Information
- **Project Name**: VTCPD Cleanup Historical Crypto Data
- **PRD ID**: 20
- **Phase/Iteration**: Phase 20, Database Optimization
- **Document Version**: 1.0
- **Date**: 2026-01-26
- **Author(s)**: AI Development Assistant
- **Stakeholders**: Core Development Team
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2026-01-26
- **Previous PRD**: [PRD-19: Debt Receipt Signature Structure Update](19-debt-receipt-signature-structure-update.md)
- **Related Documents**:
  - [PRD-18: Audit Mechanism Finalized Transactions](18-audit-mechanism-finalized-transactions.md)
  - [Payment Protocol](../../architecture/vtcpd/protocols/payment-protocol.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md).

## Executive Summary

- **Current project state**: Between VTCPD nodes, trust lines are established in specified equivalents. Periodically, nodes sign the current state of the trust line and exchange signatures. The signed state is called an audit. Based on the latest audit (with the highest number) and debt receipts (which also relate to the trust line) that appear during payment execution, the current trust line state is formed.
- **This iteration's focus**: Implement a transaction that removes obsolete historical data (signatures, audits, and debt receipts) from the database for a given equivalent, reducing database size and improving operation performance.
- **Connection to overall vision**: Over time, debt receipts and audits accumulate in the node's database, leading to increased database size and longer database operation times. Since we sign the current trust line state, storing previous signatures becomes unnecessary once they are no longer needed for finalization through observing.

## Iteration Context

### Previous Iterations Summary
- **Completed Features**:
  - Trust line audit mechanism (AuditSourceTransaction, AuditTargetTransaction)
  - Payment receipt storage (OutgoingPaymentReceiptHandler, IncomingPaymentReceiptHandler)
  - Payment transaction observing states (PaymentObservingState enum) used for finalization through observing
  - Audit mechanism based on finalized transactions (PRD-18)
- **Lessons Learned**: Database growth from accumulated historical data can impact node performance
- **Technical Debt**: No mechanism to clean up obsolete audit and receipt data
- **User Feedback**: Need for database size management and performance optimization

### Current State Analysis
- **What's working well**: Audit process, receipt storage, observer integration
- **Pain points identified**:
  - Database size grows indefinitely with accumulated audits and receipts
  - No automatic cleanup of data that is no longer needed
  - Potential performance degradation over time
- **Performance metrics**: N/A (development environment)

## Problem Statement

### Background
Trust lines in VTCPD maintain balance state between two nodes. This state is periodically synchronized through an audit process where both parties sign the current trust line state. Payment transactions create debt receipts that modify the trust line balance. These receipts are associated with specific audit numbers.

Over time, the following data accumulates in the database:
- **Audits**: Signed states of trust lines at specific points in time
- **Debt receipts**: Records of balance changes from payment transactions (incoming and outgoing)
- **Payment transactions**: Records in `payment_transactions` table with observing state information used for finalization through observing
- **Participant votes**: Records in `payment_participants_votes` table with signature data

### Problem Description
The current system has no mechanism to remove obsolete historical data:

1. **Unlimited database growth**: Every audit and payment creates permanent records that are never cleaned up.

2. **Redundant data storage**: Once a newer audit is signed and the finalization through observing window for related transactions has passed, older audits and their associated receipts serve no purpose.

3. **Performance impact**: Larger databases lead to slower query execution and increased storage costs.

**Who is affected**: All VTCPD node operators with active trust lines and payment activity.

**When and where**: On any node with long-running trust lines that have accumulated many audits and payment transactions.

**Impact of not solving**: Continuously growing database size, degraded performance, increased storage requirements.

### Success Metrics
- **Primary KPIs**:
  - Successful removal of obsolete audits and receipts where finalization through observing is no longer possible
  - Proper cleanup of global tables (payment_transactions, payment_participants_votes) when transactions are no longer referenced
  - No data loss for audits/receipts that are still needed
- **Secondary KPIs**:
  - No regression in existing trust line functionality
  - Atomic cleanup operations with proper rollback on failure
- **Target Values**:
  - Zero cleanup of data where finalization through observing is still possible
  - 100% cleanup of eligible obsolete data

## Project Scope

### This Iteration's Scope

#### New Features/Enhancements

1. **CleanupHistoricalCryptoDataTransaction**
   - New transaction that removes obsolete historical data from the database
   - Triggered by a signal after successful audit completion
   - Works for a specific trust line (identified by ContractorID + Equivalent)

2. **historyCryptoDataCleanupSignal**
   - New signal emitted from audit transactions after successful audit signing and saving
   - Contains ContractorID and Equivalent
   - Emitted immediately after `trustLineActionSignal` in relevant transactions

3. **Retention Policy with RETENTION_OFFSET Constant**
   - Constant value `RETENTION_OFFSET = 2` defined in the transaction
   - Determines how many recent audits to preserve
   - If current audit = 155, audits with number <= 153 are candidates for cleanup

4. **Sequential Audit Cleanup Logic**
   - Process audits from lowest to highest number
   - Stop processing if any audit cannot be cleaned up (finalization through observing still possible)
   - Ensures no gaps in audit history

5. **Finalization Through Observing Window Verification**
   - Retrieve current block number via `sendRpcRequest(make_shared<GetBlockNumberRpcRequest>(currentTransactionUUID()))`
   - Compare with `effective_claiming_block_number` from `payment_transactions`
   - Finalization through observing is still possible when `effective_claiming_block_number >= current_block_number`
   - Only delete data where finalization through observing is no longer possible (`effective_claiming_block_number < current_block_number`)

6. **Global Table Cleanup Logic**
   - Delete from `payment_transactions` and `payment_participants_votes` only when transaction UUID is no longer referenced in any receipt table
   - Check both `incoming_payment_receipts` and `outgoing_payment_receipts` across all trust lines

#### Modifications to Existing Features

1. **SetOutgoingTrustLineTransaction**
   - Add `historyCryptoDataCleanupSignal` emission immediately after `trustLineActionSignal` on successful audit completion

2. **CloseIncomingTrustLineTransaction**
   - Add `historyCryptoDataCleanupSignal` emission immediately after `trustLineActionSignal` on successful audit completion

3. **AuditSourceTransaction**
   - Add `historyCryptoDataCleanupSignal` emission immediately after `trustLineActionSignal` on successful audit completion

4. **AuditTargetTransaction**
   - Add `historyCryptoDataCleanupSignal` emission immediately after `trustLineActionSignal` on successful audit completion

5. **TransactionsManager**
   - Add slot for `historyCryptoDataCleanupSignal`
   - Add method to launch `CleanupHistoricalCryptoDataTransaction`
   - Subscribe to the new signal from relevant transactions

6. **BaseTransaction**
   - Add `HistoryCryptoDataCleanupSignal` typedef
   - Add `historyCryptoDataCleanupSignal` member

7. **IncomingPaymentReceiptHandler**
   - Add method: `deleteRecordsByAuditNumber(TrustLineID, AuditNumber)`

8. **OutgoingPaymentReceiptHandler**
   - Add method: `deleteRecordsByAuditNumber(TrustLineID, AuditNumber)`

### Explicitly Out of Scope
- Cleanup of other types of historical data (history records, etc.)
- Configurable retention policy (RETENTION_OFFSET is hardcoded)
- Manual trigger for cleanup (only triggered by audit completion)
- Cleanup across multiple equivalents in single transaction
- UI/CLI interface for cleanup operations

### Dependencies from Previous Iterations
- [PRD-18: Audit Mechanism Finalized Transactions](18-audit-mechanism-finalized-transactions.md) - for audit number assignment to receipts
- Existing audit transaction framework
- Existing receipt handler infrastructure
- Observer RPC communication for block number retrieval

### Future Roadmap Impact
This implementation provides foundation for:
- Configurable retention policies
- Manual cleanup triggers via CLI
- Database maintenance automation

## User Stories & Requirements

### User Personas

#### Primary User: VTCPD Node Operator
- **Role**: Operates a node in the VTCP network
- **Goals**: Maintain optimal database size and node performance
- **Pain Points**: Database growth over time, potential performance degradation
- **Technical Proficiency**: Intermediate

### Functional Requirements

#### New Features for This Iteration

1. **CleanupHistoricalCryptoDataTransaction**
   - **Description**: Transaction that identifies and removes obsolete audit data, debt receipts, and related payment records
   - **User Story**: As a node operator, I want obsolete historical data to be automatically cleaned up after audits, so that my database doesn't grow indefinitely
   - **Rationale**: Prevents unlimited database growth while preserving data integrity
   - **Builds Upon**: Existing audit mechanism, receipt handlers, payment transaction handlers
   - **Acceptance Criteria**:
     - Transaction receives ContractorID and Equivalent as input
     - Calculates threshold: `current_audit_number - RETENTION_OFFSET`
     - If `threshold <= 0`, logs the fact and returns `resultDone()` without cleanup
     - Retrieves candidate audits where `audit_number <= threshold`
     - Processes audits sequentially from lowest to highest number
     - For each audit:
       - Retrieves debt receipts with matching audit_number (excluding audit_number = 0, because the payment transaction is still in progress and must not be considered for cleanup)
       - If no receipts exist, deletes the audit and continues
       - If receipts exist, collects unique transaction UUIDs
       - For each UUID, retrieves `effective_claiming_block_number` from `payment_transactions`
       - If record not found in `payment_transactions`, logs warning and continues (missing record does not block cleanup)
       - If `effective_claiming_block_number >= current_block_number` for any UUID, STOPS processing (finalization through observing still possible)
       - If `effective_claiming_block_number < current_block_number` for ALL UUIDs, proceeds with deletion (finalization through observing is no longer possible)
     - Deletion is atomic within single DB transaction:
       - Deletes incoming_payment_receipts by (TrustLineID, audit_number)
       - Deletes outgoing_payment_receipts by (TrustLineID, audit_number)
       - Deletes audit by (TrustLineID, audit_number)
       - For each deleted transaction UUID:
         - Checks if UUID exists in any incoming_payment_receipts (any TrustLine)
         - Checks if UUID exists in any outgoing_payment_receipts (any TrustLine)
         - If not found anywhere, deletes from payment_transactions and payment_participants_votes
     - On error, rolls back DB transaction and terminates
   - **Priority**: High
   - **Dependencies**: All handler modifications

2. **Signal-Based Transaction Triggering**
   - **Description**: New signal `historyCryptoDataCleanupSignal` to trigger cleanup after successful audit
   - **User Story**: As a system, I want cleanup to happen automatically after audits complete, so that no manual intervention is required
   - **Rationale**: Ensures timely cleanup without operator intervention
   - **Builds Upon**: Existing signal mechanism (similar to `trustLineActionSignal`)
   - **Acceptance Criteria**:
     - Signal defined as `signals::signal<void(ContractorID, const SerializedEquivalent)>`
     - Signal emitted in SetOutgoingTrustLineTransaction immediately after `trustLineActionSignal(...)`
     - Signal emitted in CloseIncomingTrustLineTransaction immediately after `trustLineActionSignal(...)`
     - Signal emitted in AuditSourceTransaction immediately after `trustLineActionSignal(...)`
     - Signal emitted in AuditTargetTransaction immediately after `trustLineActionSignal(...)`
     - TransactionsManager subscribes to signal and launches cleanup transaction
   - **Priority**: High
   - **Dependencies**: BaseTransaction signal infrastructure

3. **Receipt Deletion by Audit Number**
   - **Description**: New methods to delete receipts by TrustLineID and audit number
   - **User Story**: As the cleanup transaction, I need to efficiently delete all receipts for a specific audit number
   - **Rationale**: Enables batch deletion of receipts belonging to a specific audit
   - **Builds Upon**: Existing receipt handler infrastructure
   - **Acceptance Criteria**:
     - `IncomingPaymentReceiptHandler::deleteRecordsByAuditNumber(TrustLineID, AuditNumber)` implemented
     - `OutgoingPaymentReceiptHandler::deleteRecordsByAuditNumber(TrustLineID, AuditNumber)` implemented
     - Both SQLite and PostgreSQL implementations provided
     - Methods delete all records matching TrustLineID AND audit_number
   - **Priority**: High
   - **Dependencies**: None

4. **RETENTION_OFFSET Constant**
   - **Description**: Constant defining how many recent audits to preserve
   - **User Story**: As a system designer, I want a clear policy for how many audits to keep
   - **Rationale**: Provides safety margin for finalization through observing and potential issues
   - **Builds Upon**: N/A
   - **Acceptance Criteria**:
     - Constant `RETENTION_OFFSET = 2` defined in CleanupHistoricalCryptoDataTransaction
     - Used in threshold calculation: `current_audit - RETENTION_OFFSET`
     - Clear documentation of the constant's purpose
   - **Priority**: High
   - **Dependencies**: None

### Non-Functional Requirements

#### Performance
- Block number retrieval should be efficient (single RPC call)
- Receipt deletion should use batch operations where possible
- Cleanup should not block other transactions significantly

#### Security
- No cleanup of data where finalization through observing is still possible
- Atomic operations to prevent partial cleanup states
- Proper validation of all parameters

#### Reliability
- Proper error handling for all operations
- DB transaction rollback on any failure
- Warning logs for missing payment_transactions records, cleanup continues
- No data loss for valid, needed records

#### Scalability
- Handles trust lines with many historical audits
- Efficient queries for receipt and audit retrieval

## Technical Specifications

### Architecture Evolution
- **Current Architecture**: No cleanup mechanism; all historical data retained indefinitely
- **Proposed Changes**:
  - Add cleanup transaction triggered by signal
  - Add new handler methods for batch deletion
  - Integrate with existing audit flow
- **Backwards Compatibility**: Not required (development phase)
- **Migration Requirements**: None

### Data Flow

#### Cleanup Algorithm

```
1. INPUT: ContractorID, Equivalent

2. GET TrustLineID from TrustLinesManager

3. GET current_audit_number from AuditHandler.getActualAuditNumber(TrustLineID)

4. CALCULATE threshold = current_audit_number - RETENTION_OFFSET (2)
   - IF threshold <= 0: log and return resultDone()

5. GET current_block_number via GetBlockNumberRpcRequest
   - If RPC fails: log error, return resultDone()

6. GET candidate_audits = AuditHandler.auditsLessEqualThanAuditNumber(TrustLineID, threshold)
   - Sort by audit_number ASC

7. FOR EACH audit IN candidate_audits (from lowest to highest):

   7.1. GET incoming_receipts = IncomingPaymentReceiptHandler.receiptsByAuditNumber(TrustLineID, audit.number)
        - Exclude receipts with audit_number = 0 (payment transaction still in progress; do not consider for cleanup)

   7.2. GET outgoing_receipts = OutgoingPaymentReceiptHandler.receiptsByAuditNumber(TrustLineID, audit.number)
        - Exclude receipts with audit_number = 0 (payment transaction still in progress; do not consider for cleanup)

   7.3. COLLECT unique_uuids from all receipts

   7.4. IF unique_uuids is empty:
        - DELETE audit
        - CONTINUE to next audit

   7.5. FOR EACH uuid IN unique_uuids:
        - TRY: effective_block = PaymentTransactionsHandler.effectiveClaimingBlockNumber(uuid)
        - CATCH NotFoundError: log warning, continue (missing record does not block cleanup)
        - IF effective_block >= current_block_number:
            - STOP entire cleanup (finalization through observing still possible)
            - return resultDone()

   7.6. BEGIN DB TRANSACTION:
        - IncomingPaymentReceiptHandler.deleteRecordsByAuditNumber(TrustLineID, audit.number)
        - OutgoingPaymentReceiptHandler.deleteRecordsByAuditNumber(TrustLineID, audit.number)
        - AuditHandler.deleteAuditByNumber(TrustLineID, audit.number)

        - FOR EACH uuid IN unique_uuids:
            - IF NOT IncomingPaymentReceiptHandler.isContainsTransaction(uuid)
              AND NOT OutgoingPaymentReceiptHandler.isContainsTransaction(uuid):
                - PaymentTransactionsHandler.deleteRecord(uuid)
                - PaymentParticipantsVotesHandler.deleteRecords(uuid)

        - COMMIT

   7.7. ON ERROR: ROLLBACK, log error, return resultDone()

8. return resultDone()
```

#### Visual Example

```
Current audit: 155
RETENTION_OFFSET: 2
Threshold: 153

Database state:
  Audits: [149, 150, 151, 152, 153, 154, 155]
                                    └─────┴─────┘ preserved (> threshold)
          └────┴────┴────┴────┴────┘ candidates for cleanup (<= threshold)

Processing order: 149 → 150 → 151 → 152 → 153

Example execution:
  149: no receipts → DELETE audit → continue
  150: receipts [uuid-A, uuid-B], finalization through observing no longer possible → DELETE all → continue
  151: receipts [uuid-C], finalization through observing still possible → STOP

Result: 149, 150 deleted; 151, 152, 153 preserved
```

#### Global Table Cleanup Example

```
Deleting audit 150 on TrustLine TL1:
├── Receipts: [uuid-A, uuid-B, uuid-C]
│
├── 1) DELETE incoming_receipts WHERE trust_line_id=TL1 AND audit_number=150
├── 2) DELETE outgoing_receipts WHERE trust_line_id=TL1 AND audit_number=150
├── 3) DELETE audit WHERE trust_line_id=TL1 AND audit_number=150
│
├── 4) Check uuid-A:
│      ├── incoming_receipts.isContainsTransaction(uuid-A)? → NO
│      ├── outgoing_receipts.isContainsTransaction(uuid-A)? → NO
│      └── DELETE from payment_transactions AND payment_participants_votes
│
├── 5) Check uuid-B:
│      ├── incoming_receipts.isContainsTransaction(uuid-B)? → YES (exists on TL2)
│      └── DO NOT delete from global tables
│
└── 6) Check uuid-C:
       ├── incoming_receipts.isContainsTransaction(uuid-C)? → NO
       ├── outgoing_receipts.isContainsTransaction(uuid-C)? → NO
       └── DELETE from payment_transactions AND payment_participants_votes
```

### Key Files to Modify

| File | Changes |
|------|---------|
| `src/core/transactions/transactions/trust_lines/CleanupHistoricalCryptoDataTransaction.h/.cpp` | **NEW** - Main cleanup transaction |
| `src/core/transactions/transactions/base/BaseTransaction.h` | Add HistoryCryptoDataCleanupSignal typedef and signal member |
| `src/core/transactions/manager/TransactionsManager.h/.cpp` | Add subscription and launch method for cleanup transaction |
| `src/core/transactions/transactions/trust_lines/SetOutgoingTrustLineTransaction.cpp` | Emit historyCryptoDataCleanupSignal |
| `src/core/transactions/transactions/trust_lines/CloseIncomingTrustLineTransaction.cpp` | Emit historyCryptoDataCleanupSignal |
| `src/core/transactions/transactions/trust_lines/AuditSourceTransaction.cpp` | Emit historyCryptoDataCleanupSignal |
| `src/core/transactions/transactions/trust_lines/AuditTargetTransaction.cpp` | Emit historyCryptoDataCleanupSignal |
| `src/core/io/storage/interfaces/IncomingPaymentReceiptHandler.h` | Add deleteRecordsByAuditNumber method |
| `src/core/io/storage/interfaces/OutgoingPaymentReceiptHandler.h` | Add deleteRecordsByAuditNumber method |
| `src/core/io/storage/sqlite/IncomingPaymentReceiptHandler.h/.cpp` | Implement deleteRecordsByAuditNumber |
| `src/core/io/storage/sqlite/OutgoingPaymentReceiptHandler.h/.cpp` | Implement deleteRecordsByAuditNumber |
| `src/core/io/storage/postgresql/IncomingPaymentReceiptHandler.h/.cpp` | Implement deleteRecordsByAuditNumber |
| `src/core/io/storage/postgresql/OutgoingPaymentReceiptHandler.h/.cpp` | Implement deleteRecordsByAuditNumber |

### New Transaction Type

Add to `BaseTransaction::TransactionType` enum:
```cpp
// General
CleanupHistoricalCryptoDataType = 1203
```

## Implementation Plan

### This Iteration Plan
- **Key Deliverables**:
  - Handler methods and storage updates
  - Signal infrastructure and emission
  - CleanupHistoricalCryptoDataTransaction implementation
  - Testing and validation

### Iteration Milestones

| Milestone | Description | Dependencies | Risk Level |
|-----------|-------------|--------------|------------|
| Handler Methods | New deleteRecordsByAuditNumber methods | None | Low |
| Signal Infrastructure | New signal in BaseTransaction, subscription in TransactionsManager | None | Low |
| Signal Emission | Add signal emission to 4 audit transactions | Signal infrastructure | Low |
| Cleanup Transaction | Full CleanupHistoricalCryptoDataTransaction | All above | High |
| Testing | Unit and integration tests | All above | Medium |

## Risk Management

### Technical Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| Accidental deletion of needed data | High | Low | Strict finalization through observing window check, RETENTION_OFFSET buffer |
| Partial cleanup on failure | Medium | Medium | Atomic DB transactions with rollback |
| Performance impact during cleanup | Low | Low | Efficient queries, batch operations |
| Race condition with audit transactions | Medium | Low | Cleanup triggered after audit completion |

## Testing Strategy

### Testing Approach

#### Unit Tests (SQLite - tests/unit/sqlite/)

**IncomingPaymentReceiptHandlerSQLiteTest.cpp:**
- `deleteRecordsByAuditNumber_deletesMatchingRecords`
- `deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers`
- `deleteRecordsByAuditNumber_doesNotDeleteOtherTrustLines`
- `deleteRecordsByAuditNumber_handlesEmptyTable`

**OutgoingPaymentReceiptHandlerSQLiteTest.cpp:**
- `deleteRecordsByAuditNumber_deletesMatchingRecords`
- `deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers`
- `deleteRecordsByAuditNumber_doesNotDeleteOtherTrustLines`
- `deleteRecordsByAuditNumber_handlesEmptyTable`

#### Integration Tests (PostgreSQL - tests/storage/integration/postgresql/)

**IncomingPaymentReceiptHandlerPostgreSQLIntegrationTest.cpp:**
- `deleteRecordsByAuditNumber_deletesMatchingRecords`
- `deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers`
- `deleteRecordsByAuditNumber_doesNotDeleteOtherTrustLines`

**OutgoingPaymentReceiptHandlerPostgreSQLIntegrationTest.cpp:**
- `deleteRecordsByAuditNumber_deletesMatchingRecords`
- `deleteRecordsByAuditNumber_doesNotDeleteOtherAuditNumbers`
- `deleteRecordsByAuditNumber_doesNotDeleteOtherTrustLines`

#### Scenario Tests

1. **Happy path**: All candidate audits have expired finalization through observing windows
2. **Partial cleanup**: Some audits cleanable, one blocks further cleanup
3. **No cleanup needed**: All audits within retention window
4. **Empty receipts**: Audit exists but has no associated receipts
5. **Missing payment_transactions**: Receipt exists but no payment_transactions record; cleanup proceeds
6. **Global table preservation**: Transaction UUID still referenced on another trust line
7. **Block number retrieval failure**: RPC fails, cleanup terminates gracefully
8. **Database error**: Rollback on deletion failure

### Quality Gates
- All new methods have unit test coverage
- No regression in existing audit functionality
- Code review completed
- Atomic operations verified

## Appendices

### Glossary
- **Audit**: Signed state of a trust line at a specific point in time, identified by audit number
- **Debt Receipt**: Record of balance change on a trust line from a payment transaction
- **Trust Line (TL)**: Bidirectional credit relationship between two nodes in a specific equivalent
- **Finalization through observing**: Process of finalizing payment transactions through the observer network
- **Effective Claiming Block Number**: Block number until which a transaction can be finalized through observing
- **RETENTION_OFFSET**: Number of recent audits to always preserve (value: 2)

### References
- [Payment Protocol](../../architecture/vtcpd/protocols/payment-protocol.md)
- [PRD-18: Audit Mechanism Finalized Transactions](18-audit-mechanism-finalized-transactions.md)
- [AuditHandler interface](../../../src/core/io/storage/interfaces/AuditHandler.h)
- [PaymentTransactionsHandler interface](../../../src/core/io/storage/interfaces/PaymentTransactionsHandler.h)

### Interface Method Signatures

#### IncomingPaymentReceiptHandler (new method)
```cpp
virtual void deleteRecordsByAuditNumber(
    const TrustLineID trustLineID,
    const AuditNumber auditNumber) = 0;
```

#### OutgoingPaymentReceiptHandler (new method)
```cpp
virtual void deleteRecordsByAuditNumber(
    const TrustLineID trustLineID,
    const AuditNumber auditNumber) = 0;
```

#### BaseTransaction (new signal)
```cpp
typedef signals::signal<void(ContractorID, const SerializedEquivalent)> HistoryCryptoDataCleanupSignal;

mutable HistoryCryptoDataCleanupSignal historyCryptoDataCleanupSignal;
```

---

**Document History**

| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2026-01-26 | AI Assistant | Initial draft | Phase 20 |

**Related Documents**
- **Previous PRD**: [PRD-19: Debt Receipt Signature Structure Update](19-debt-receipt-signature-structure-update.md)
- **Audit Mechanism**: [PRD-18: Audit Mechanism Finalized Transactions](18-audit-mechanism-finalized-transactions.md)
- **Payment Protocol**: [payment-protocol.md](../../architecture/vtcpd/protocols/payment-protocol.md)
