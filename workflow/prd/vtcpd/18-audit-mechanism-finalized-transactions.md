# Project Requirements Document (PRD)

## Document Information
- **Project Name**: VTCPD Audit Mechanism Based on Finalized Transactions
- **PRD ID**: 18
- **Phase/Iteration**: Phase 18, Trust Line Audit Enhancement
- **Document Version**: 1.0
- **Date**: 2026-01-19
- **Author(s)**: AI Development Assistant
- **Stakeholders**: Core Development Team
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2026-01-19
- **Previous PRD**: [PRD-17: Block Number Cache](17-block-number-cache.md)
- **Related Documents**:
  - [Payment Protocol](../../architecture/vtcpd/protocols/payment-protocol.md)
  - [PRD-02: SPHINCS+ Cryptography](02_sphincs_plus_cryptography_implementation.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md).

## Executive Summary

- **Current project state**: The audit mechanism currently includes all payment receipts regardless of whether their corresponding payment transactions have been finalized. This can lead to trust line state inconsistencies when transactions are still pending observer confirmation or get rejected.
- **This iteration's focus**: Modify the audit mechanism to only include receipts from finalized (committed) transactions, with both parties agreeing on the exact set of transactions included in each audit.
- **Connection to overall vision**: This enhancement improves trust line reliability and prevents conflicts caused by including non-finalized transaction data in audits.

## Iteration Context

### Previous Iterations Summary
- **Completed Features**:
  - Trust line audit mechanism (AuditSourceTransaction, AuditTargetTransaction)
  - Payment receipt storage (OutgoingPaymentReceiptHandler, IncomingPaymentReceiptHandler)
  - Payment transaction observing states (PaymentObservingState enum)
  - Block number cache for efficient block number retrieval
- **Lessons Learned**: Including non-finalized transactions in audits can cause state desynchronization between nodes
- **Technical Debt**: Current audit includes all receipts without verifying transaction finalization status
- **User Feedback**: Need for more robust audit synchronization between trust line participants

### Current State Analysis
- **What's working well**: Basic audit flow, receipt storage, signature verification
- **Pain points identified**:
  - Receipts are included in audit regardless of transaction finalization status
  - No mechanism for nodes to agree on transaction set before audit
  - Potential for trust line conflicts due to desynchronized audit data
- **Performance metrics**: N/A (development environment)

## Problem Statement

### Background
Trust lines in VTCPD maintain balance state between two nodes. This state is periodically synchronized through an audit process where both parties sign the current trust line state. Payment transactions create receipts that modify the trust line balance, but these transactions go through an observer confirmation process that may take time or even result in rejection.

### Problem Description
The current audit mechanism has a critical flaw:

1. **Non-finalized transactions included**: When an audit is initiated, all payment receipts are included in the new balance calculation, regardless of whether their corresponding payment transactions have been confirmed by the observer (status `PaymentObservingState::Committed`).

2. **Desynchronization risk**: If a transaction is included in an audit but later gets rejected by the observer, the trust line state becomes incorrect.

3. **Asymmetric state**: One node may have a transaction finalized while the other doesn't, leading to different audit calculations and potential conflicts.

**Who is affected**: All nodes participating in trust lines with active payment activity.

**When and where**: During audit operations on trust lines with pending or non-finalized payment transactions.

**Impact of not solving**: Trust line conflicts, incorrect balance states, need for manual conflict resolution.

### Success Metrics
- **Primary KPIs**:
  - Audits only include receipts from transactions with `PaymentObservingState::Committed` status
  - Both audit parties agree on the exact set of transactions included
  - Proper handling of desynchronization scenarios
- **Secondary KPIs**:
  - No regression in existing audit functionality
  - Proper conflict detection and trust line state management
- **Target Values**:
  - Zero audits including non-finalized transaction data
  - 100% agreement on transaction sets between audit parties

## Project Scope

### This Iteration's Scope

#### New Features/Enhancements

1. **Receipt Audit Number Deferred Assignment**
   - Save new receipts with `auditNumber = 0` to indicate they haven't been included in any audit yet
   - Update receipt audit numbers only upon successful audit completion

2. **Transaction List in Audit Messages**
   - Add `vector<TransactionUUID>` to `AuditMessage` for initiator to propose transaction set
   - Add `vector<TransactionUUID>` to `AuditResponseMessage` for contractor to communicate discrepancies
   - Transaction UUID list must be ordered lexicographically by raw UUID bytes (same ordering as `compareTransactionUUID` in `CompletedPaymentsObserverMonitoringTransaction::runGetClaimStatusesStage`) and hashed; the hash is included in the audit signature payload after balance and before equivalentRegistryAddress

3. **Finalized-Only Receipt Selection**
   - Modify audit logic to select only receipts where:
     - `auditNumber = 0` (not yet included in any audit)
     - Corresponding transaction has `PaymentObservingState::Committed` status

4. **Block Number Verification Step**
   - Add step in audit transactions to retrieve current block number via `GetBlockNumberRpcRequest`
   - Verify if observing is still possible for pending transactions
   - If block number retrieval fails, log the error and finish the transaction with `resultDone()`; add TODO comment in code to revisit this case

5. **Transaction List Reconciliation**
   - Implement logic for contractor to verify proposed transaction list
   - Handle cases where lists differ between initiator and contractor
   - Retry with updated list is allowed only once; second `Audit_UpdateTransactionsList` response sets TL to Conflict
   - If `Audit_UpdateTransactionsList` contains UUIDs not present in the original AuditMessage list, set TL to Conflict and finish the transaction

6. **Balance and Receipt Amount Correction**
   - Correct `mTotalIncomingReceiptsAmount` / `mTotalOutgoingReceiptsAmount` when excluding transactions
   - Preserve unrealized receipt amounts instead of resetting to zero
   - Add example in PRD to clarify balance and totals recomputation

#### Modifications to Existing Features

1. **TrustLineKeychain::saveOutgoingPaymentReceipt / saveIncomingPaymentReceipt**
   - Pass `auditNumber = 0` instead of current audit number

2. **AuditMessage (src/core/network/messages/trust_lines/AuditMessage.h/.cpp)**
   - Add `vector<TransactionUUID> mTransactionUUIDs` field
   - Update serialization/deserialization

3. **AuditResponseMessage (src/core/network/messages/trust_lines/AuditResponseMessage.h/.cpp)**
   - Add `vector<TransactionUUID> mTransactionUUIDs` field for `Audit_UpdateTransactionsList` status
   - Update serialization/deserialization

4. **AuditSourceTransaction (src/core/transactions/transactions/trust_lines/AuditSourceTransaction.cpp)**
   - Add block number retrieval step
   - Modify receipt selection to filter by finalized transactions
   - Add transaction list to AuditMessage
   - Handle `Audit_UpdateTransactionsList` response
   - Handle `Audit_Invalid` response (set TL to Conflict)
   - Implement retry logic with reduced transaction set

5. **AuditTargetTransaction (src/core/transactions/transactions/trust_lines/AuditTargetTransaction.cpp)**
   - Add block number retrieval step
   - Verify proposed transaction list against local data
   - Implement desynchronization handling logic
   - Correct receipt amounts when excluding transactions

6. **SetOutgoingTrustLineTransaction, CloseIncomingTrustLineTransaction**
   - Apply same changes as AuditSourceTransaction where applicable

7. **OutgoingPaymentReceiptHandler / IncomingPaymentReceiptHandler**
   - Add method to retrieve receipts with `auditNumber = 0` and finalized transaction status
   - Add method to update audit number for receipts by transaction UUIDs

8. **TrustLinesManager::resetTrustLineTotalReceiptsAmounts**
   - Modify to preserve unrealized receipt amounts instead of resetting to zero
   - Or add new method for partial reset

### Explicitly Out of Scope
- Conflict resolution mechanism (ConflictResolving) - separate PRD
- Changes to payment transaction flow
- Changes to observer communication protocol
- UI/CLI changes
- Transaction list size limits and batching (future work)

### Dependencies from Previous Iterations
- [PRD-17: Block Number Cache](17-block-number-cache.md) - for efficient block number retrieval
- [PRD-16: Payment Transaction Observing States](16-payment-transaction-observing-states.md) - for PaymentObservingState enum
- Existing audit transaction framework
- Existing receipt handler infrastructure

### Future Roadmap Impact
This implementation provides foundation for:
- Robust conflict resolution mechanism
- Enhanced trust line synchronization
- More reliable multi-node payment networks

## User Stories & Requirements

### User Personas

#### Primary User: VTCPD Node Operator
- **Role**: Operates a node in the VTCP network
- **Goals**: Maintain accurate trust line states, avoid conflicts
- **Pain Points**: Trust line conflicts due to audit desynchronization
- **Technical Proficiency**: Intermediate

### Functional Requirements

#### New Features for This Iteration

1. **Deferred Audit Number Assignment for Receipts**
   - **Description**: Save receipts with `auditNumber = 0` and update only upon successful audit
   - **User Story**: As a node operator, I want receipts to only be included in audits after their transactions are finalized, so that my trust line state remains accurate
   - **Rationale**: Prevents inclusion of non-finalized transaction data in audits
   - **Builds Upon**: Existing receipt storage mechanism
   - **Acceptance Criteria**:
     - New receipts saved with `auditNumber = 0`
     - Receipt audit numbers updated to actual audit number upon successful audit completion
     - Only receipts with `auditNumber = 0` considered for new audits
   - **Priority**: High
   - **Dependencies**: Receipt handler modifications

2. **Transaction List Exchange in Audit Protocol**
   - **Description**: Include list of transaction UUIDs in audit messages for both parties to agree on
   - **User Story**: As a node, I want to verify that my audit counterpart and I are using the same set of transactions, so that we don't create conflicting trust line states
   - **Rationale**: Enables detection and handling of desynchronization
   - **Builds Upon**: Existing AuditMessage/AuditResponseMessage
   - **Acceptance Criteria**:
     - AuditMessage contains `vector<TransactionUUID>`
     - AuditResponseMessage contains `vector<TransactionUUID>` when status is `Audit_UpdateTransactionsList`
     - Proper serialization/deserialization of transaction lists
     - Transaction UUID list is lexicographically sorted by raw UUID bytes (use the same ordering as `compareTransactionUUID` in `CompletedPaymentsObserverMonitoringTransaction::runGetClaimStatusesStage`) and contains no duplicates
     - Hash of the ordered transaction list is included in the audit signature payload after balance and before equivalentRegistryAddress
     - Hash algorithm: SHA-256 over `uint32_t count` (same byte order as `serializeGetClaimStatusesForSigning`) + concatenated `TransactionUUID::data` in sorted order
   - **Priority**: High
   - **Dependencies**: Message class modifications

3. **Finalized Transaction Filtering**
   - **Description**: Only include receipts from transactions with `PaymentObservingState::Committed` status
   - **User Story**: As a node, I want audits to only consider finalized transactions, so that pending or rejected transactions don't affect my trust line state
   - **Rationale**: Core requirement for preventing audit desynchronization
   - **Builds Upon**: PaymentTransactionsHandler, receipt handlers
   - **Acceptance Criteria**:
     - Query joins receipts with payment_transactions table
     - Only receipts where corresponding transaction has `observing_state = 1` (Committed) are selected
     - Non-existent or non-committed transactions are excluded
   - **Priority**: High
   - **Dependencies**: Database query modifications

4. **Block Number Verification**
   - **Description**: Retrieve current block number to determine if observing is still possible for transactions
   - **User Story**: As a node, I want to know if a transaction can still be observed, so that I can make informed decisions about including it in the audit
   - **Rationale**: Enables proper handling of transactions past their observing window
   - **Builds Upon**: BlockNumberCache, GetBlockNumberRpcRequest
   - **Acceptance Criteria**:
     - Audit transactions include step for block number retrieval via `sendRpcRequest(make_shared<GetBlockNumberRpcRequest>(...))`
     - Comparison of `currentBlockNumber` with `effectiveClaimingBlockNumber`
     - Observing is possible if `currentBlockNumber <= effectiveClaimingBlockNumber`
     - Observing is not possible if `currentBlockNumber > effectiveClaimingBlockNumber`
     - Correct determination of observing possibility
     - If block number retrieval fails, log the error and finish the transaction with `resultDone()`; add TODO comment in code to revisit this case
   - **Priority**: High
   - **Dependencies**: PRD-17 Block Number Cache

5. **Transaction List Reconciliation Logic**
   - **Description**: Handle cases where initiator and contractor have different transaction sets
   - **User Story**: As a node, I want proper handling of transaction set mismatches, so that conflicts are detected early and handled appropriately
   - **Rationale**: Prevents silent state corruption
   - **Builds Upon**: Audit transaction logic
   - **Acceptance Criteria**:
     - **Contractor has less transactions**: If initiator proposes finalized transactions that contractor doesn't have (no receipts), set TL to Conflict and respond with `Audit_Invalid`
     - **Contractor has non-finalized versions**: If initiator proposes finalized transactions that are not finalized on contractor side but receipts exist, check if observing is still possible; if yes, respond with `Audit_UpdateTransactionsList` listing problematic transactions; if no, set TL to Conflict
     - **Contractor has more transactions**: If contractor has more finalized transactions than proposed, check if observing is still possible for extras; if not, set TL to Conflict; if yes, exclude them and continue
     - **Initiator receives Audit_Invalid**: Set TL to Conflict
     - **Initiator receives Audit_UpdateTransactionsList**: List contains only transactions that are not finalized on contractor side; exclude listed transactions, recalculate balance, retry audit
     - **Initiator observing check**: For each UUID in `Audit_UpdateTransactionsList`, verify `currentBlockNumber <= effectiveClaimingBlockNumber`; if any is not observable, set TL to Conflict and finish
     - **Initiator validation of Audit_UpdateTransactionsList**: Every UUID in response must exist in the original AuditMessage list (no injection)
     - **Invalid Audit_UpdateTransactionsList**: If response contains UUIDs outside the original list, set TL to Conflict and finish the transaction
     - **Retry limit**: If initiator receives `Audit_UpdateTransactionsList` a second time, set TL to Conflict and finish the transaction
   - **Priority**: High
   - **Dependencies**: All above features

6. **Receipt Amount Preservation**
   - **Description**: Preserve unrealized receipt amounts in `mTotalIncomingReceiptsAmount` / `mTotalOutgoingReceiptsAmount` instead of resetting
   - **User Story**: As a node, I want unrealized receipts to be carried forward, so that they can be included in future audits
   - **Rationale**: Ensures no receipt data is lost when transactions are excluded from audit
   - **Builds Upon**: TrustLinesManager, TrustLine class
   - **Acceptance Criteria**:
     - After audit, `mTotalIncomingReceiptsAmount` / `mTotalOutgoingReceiptsAmount` contain sum of excluded receipt amounts
     - Balance is correctly adjusted based on included receipts only
     - Excluded receipts retain `auditNumber = 0`
     - Example: receipts `uuid1: +1000`, `uuid2: -500`, `uuid3: -300`, `mTotalIncomingReceiptsAmount=1000`, `mTotalOutgoingReceiptsAmount=800`, `balance=2000`. Exclude `uuid1` and `uuid3`:
       - New balance = `1300` (apply `-1000 + 300`)
       - New `mTotalIncomingReceiptsAmount=0`, new `mTotalOutgoingReceiptsAmount=500` (included receipts only)
       - After audit, preserved unrealized amounts: `mTotalIncomingReceiptsAmount=1000`, `mTotalOutgoingReceiptsAmount=300`
   - **Priority**: High
   - **Dependencies**: Receipt amount tracking logic

### Non-Functional Requirements

#### Performance
- Block number retrieval should use cached value when available
- Transaction list processing should be efficient for typical transaction counts (< 100 per audit)
- No significant increase in audit message sizes for typical scenarios

#### Security
- Transaction UUID lists must be validated (no injection of fake UUIDs)
- For `Audit_UpdateTransactionsList`, initiator must ensure all UUIDs are present in the original AuditMessage list
- For `Audit_UpdateTransactionsList`, initiator must verify each listed transaction is still observable using `effectiveClaimingBlockNumber`
- If validation fails (unknown UUIDs), set TL to Conflict and finish the transaction
- Signature verification scheme remains unchanged (payload extended with tx list hash)
- No bypass of existing audit security measures

#### Reliability
- Proper error handling for all new code paths
- If block number retrieval fails, log the error, return `resultDone()`, and leave a TODO in code for future handling
- Clear conflict state transitions
- Audit success must atomically update receipt `auditNumber` values and trust line state (balance)

## Technical Specifications

### Architecture Evolution
- **Current Architecture**: Audit includes all receipts, resets receipt amounts after audit
- **Proposed Changes**:
  - Filter receipts by finalization status
  - Exchange and verify transaction lists
  - Preserve unrealized receipt amounts
- **Backwards Compatibility**: Not required (development phase); assume all nodes are upgraded
- **Migration Requirements**: None (non-production; no data migration planned)

### Data Requirements

#### Data Models

**Receipt Record Changes:**
- `auditNumber`: Use 0 for unprocessed receipts, actual audit number after inclusion

**AuditMessage Extension:**
- Add: `vector<TransactionUUID> mTransactionUUIDs`
- The list is lexicographically sorted by raw UUID bytes (same ordering as `compareTransactionUUID` in `CompletedPaymentsObserverMonitoringTransaction::runGetClaimStatusesStage`)
- Hash of the ordered list is inserted into the audit signature payload after balance and before equivalentRegistryAddress
- Hash algorithm: SHA-256 over `uint32_t count` (same byte order as `serializeGetClaimStatusesForSigning`) + concatenated `TransactionUUID::data` in sorted order

**AuditResponseMessage Extension:**
- Add: `vector<TransactionUUID> mTransactionUUIDs` (used with `Audit_UpdateTransactionsList` status)

#### Database Queries

**New Query - Get Finalized Receipts:**
```sql
SELECT r.*
FROM outgoing_payment_receipts r
JOIN payment_transactions pt ON r.transaction_uuid = pt.transaction_uuid
WHERE r.trust_line_id = ?
  AND r.audit_number = 0
  AND pt.observing_state = 1
```

**New Query - Update Receipt Audit Numbers:**
```sql
UPDATE outgoing_payment_receipts
SET audit_number = ?
WHERE trust_line_id = ?
  AND transaction_uuid IN (?, ?, ...)
```

### Integration Requirements

#### Modified Integrations
- **PaymentTransactionsHandler**: Query for transaction observing state
- **OutgoingPaymentReceiptHandler/IncomingPaymentReceiptHandler**: New methods for finalized receipt selection and audit number updates
- **BlockNumberCache/GetBlockNumberRpcRequest**: For current block number retrieval

### Key Files to Modify

| File | Changes |
|------|---------|
| `src/core/network/messages/trust_lines/AuditMessage.h/.cpp` | Add transaction UUID vector |
| `src/core/network/messages/trust_lines/AuditResponseMessage.h/.cpp` | Add transaction UUID vector |
| `src/core/transactions/transactions/trust_lines/AuditSourceTransaction.h/.cpp` | New steps, transaction filtering, response handling |
| `src/core/transactions/transactions/trust_lines/AuditTargetTransaction.h/.cpp` | Transaction verification, desync handling |
| `src/core/transactions/transactions/trust_lines/SetOutgoingTrustLineTransaction.cpp` | Same audit logic changes |
| `src/core/transactions/transactions/trust_lines/CloseIncomingTrustLineTransaction.cpp` | Same audit logic changes |
| `src/core/io/storage/interfaces/OutgoingPaymentReceiptHandler.h` | New methods |
| `src/core/io/storage/interfaces/IncomingPaymentReceiptHandler.h` | New methods |
| `src/core/io/storage/sqlite/OutgoingPaymentReceiptHandler.h/.cpp` | Implement new methods |
| `src/core/io/storage/sqlite/IncomingPaymentReceiptHandler.h/.cpp` | Implement new methods |
| `src/core/io/storage/postgresql/OutgoingPaymentReceiptHandler.h/.cpp` | Implement new methods |
| `src/core/io/storage/postgresql/IncomingPaymentReceiptHandler.h/.cpp` | Implement new methods |
| `src/core/crypto/keychain.h/.cpp` | Update saveOutgoingPaymentReceipt/saveIncomingPaymentReceipt calls |
| `src/core/trust_lines/manager/TrustLinesManager.h/.cpp` | Modify resetTrustLineTotalReceiptsAmounts or add new method |
| Payment transaction files in `src/core/transactions/transactions/regular/payments/` | Pass auditNumber=0 when saving receipts |

## Implementation Plan

### This Iteration Timeline
- **Duration**: 3-4 weeks
- **Key Deliverables**:
  - Week 1: Message modifications, receipt handler updates
  - Week 2: AuditSourceTransaction changes
  - Week 3: AuditTargetTransaction changes, other audit transactions
  - Week 4: Testing and validation

### Iteration Milestones

| Milestone | Description | Dependencies | Risk Level |
|-----------|-------------|--------------|------------|
| Message Protocol Update | AuditMessage/AuditResponseMessage extended | None | Low |
| Receipt Handler Updates | New methods for finalized receipt queries | Message updates | Medium |
| AuditSourceTransaction | Full implementation of initiator logic | Receipt handlers | High |
| AuditTargetTransaction | Full implementation of contractor logic | Receipt handlers | High |
| Other Audit Transactions | SetOutgoing, CloseIncoming updates | Source/Target complete | Medium |

## Risk Management

### Technical Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| Complex state machine changes | High | Medium | Thorough testing, clear documentation |
| Message serialization errors | Medium | Low | Unit tests for serialization |
| Database query performance | Low | Low | Index optimization, query analysis |
| Race conditions in receipt updates | Medium | Medium | Proper transaction handling |

## Testing Strategy

### Testing Approach

#### Unit Tests
- AuditMessage serialization/deserialization with transaction lists
- AuditResponseMessage serialization/deserialization with transaction lists
- Audit signature payload includes hash of ordered transaction list
- Receipt handler new methods
- Transaction filtering logic
- Block number retrieval failure results in `resultDone()` with log entry

#### Integration Tests
- PostgreSQL receipt handler methods for finalized receipts and audit number updates

#### Scenario Tests
1. **Happy path**: Both nodes have same finalized transactions
2. **Contractor missing transactions**: Initiator has finalized tx, contractor has no receipt
3. **Contractor has non-finalized**: Initiator has finalized tx, contractor has receipt but tx not finalized
4. **Contractor has extra transactions**: Contractor has more finalized tx than initiator proposed
5. **Observing still possible**: Transaction not finalized but observing window open
6. **Observing expired**: Transaction not finalized and observing window closed
7. **Retry limit**: Initiator receives `Audit_UpdateTransactionsList` twice -> sets TL to Conflict
8. **Invalid update list**: Initiator receives `Audit_UpdateTransactionsList` with unknown UUIDs -> sets TL to Conflict
9. **Block number failure**: GetBlockNumber RPC fails -> log and `resultDone()`

### Quality Gates
- All new code has unit test coverage
- No regression in existing audit functionality
- Code review completed

## Appendices

### Glossary
- **Audit**: Process of synchronizing trust line state between two nodes by signing the current balance
- **Receipt**: Signed record of debt movement on a trust line from a payment transaction
- **Finalized Transaction**: Payment transaction with `PaymentObservingState::Committed` status
- **Observing**: Process of confirming payment transactions through the observer network
- **Effective Claiming Block Number**: Block number until which a transaction can be disputed/observed

### References
- [Payment Protocol](../../architecture/vtcpd/protocols/payment-protocol.md)
- [TrustLine class](../../../src/core/trust_lines/TrustLine.h)
- [PaymentObservingState enum](../../../src/core/io/storage/interfaces/PaymentTransactionsHandler.h)

### State Machine Diagrams

#### AuditSourceTransaction Updated Flow
```
[Initialization]
    |
    v
[Get Block Number] <-- NEW STEP
    |
    +-- [Failed] --> [Log + Done (TODO)] <-- NEW
    |
    +-- [OK] --> [Select Finalized Receipts] <-- MODIFIED
    |
    v
[Calculate Balance & Build Transaction List] <-- MODIFIED
    |
    v
[Send AuditMessage with Transaction List] <-- MODIFIED
    |
    v
[Wait for Response]
    |
    +-- [OK] --> [Save Audit, Update Receipt Numbers] --> [Done]
    |
    +-- [Audit_UpdateTransactionsList] --> [Exclude Listed Txs] --> [Retry Audit] <-- NEW
    |
    +-- [Audit_UpdateTransactionsList] (second time) --> [Set TL to Conflict] --> [Done] <-- NEW
    |
    +-- [Audit_Invalid] --> [Set TL to Conflict] --> [Done] <-- MODIFIED
```

#### AuditTargetTransaction Updated Flow
```
[Receive AuditMessage]
    |
    v
[Get Block Number] <-- NEW STEP
    |
    +-- [Failed] --> [Log + Done (TODO)] <-- NEW
    |
    +-- [OK] --> [Select Own Finalized Receipts] <-- MODIFIED
    |
    v
[Compare Transaction Lists] <-- NEW STEP
    |
    +-- [Lists Match] --> [Verify Audit Data] --> [Sign & Respond OK]
    |
    +-- [Initiator Has Unknown Finalized Txs] --> [Set Conflict, Respond Invalid] <-- NEW
    |
    +-- [Own Non-Finalized Txs in List]
    |       |
    |       +-- [Observing Possible] --> [Respond Audit_UpdateTransactionsList] <-- NEW
    |       |
    |       +-- [Observing Not Possible] --> [Set Conflict, Respond Invalid] <-- NEW
    |
    +-- [Own Extra Finalized Txs]
            |
            +-- [Observing Possible] --> [Exclude & Continue] <-- NEW
            |
            +-- [Observing Not Possible] --> [Set Conflict, Respond Invalid] <-- NEW
```

---

**Document History**

| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2026-01-19 | AI Assistant | Initial draft | Phase 18 |

**Related Documents**
- **Previous PRD**: [PRD-17: Block Number Cache](17-block-number-cache.md)
- **Payment Protocol**: [payment-protocol.md](../../architecture/vtcpd/protocols/payment-protocol.md)
- **SPHINCS+ Cryptography**: [PRD-02](02_sphincs_plus_cryptography_implementation.md)
