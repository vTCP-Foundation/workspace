# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Debt Receipt Signature Structure Update
- **PRD ID**: 19
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.0
- **Date**: 2026-01-24
- **Author(s)**: Architect
- **Stakeholders**: Development Team, Security Team
- **PRD Status**: 1.2 - Initial brain dump complete
- **Last Status Update**: 2026-01-24
- **Previous PRD**: [PRD-06: Exchange Payment with Commissions](06-exchange-payment-with-commissions.md)
- **Related Documents**:
  - [Payment Protocol](../../architecture/vtcpd/protocols/payment-protocol.md)
  - [PRD-02: SPHINCS+ Cryptography Implementation](02_sphincs_plus_cryptography_implementation.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
- **Current project state**: Payment transactions are functional with debt receipt signing mechanism in place. The current signature structure includes ContractorID (source/target), which provides local identification but lacks cryptographic binding to specific payment keys.
- **This iteration's focus**: Update the debt receipt signature structure to use payment public keys instead of ContractorIDs, remove audit number from signed data, add equivalent to signed data, and implement payment public key exchange during channel creation.
- **Connection to overall vision**: This enhancement strengthens the cryptographic integrity of debt receipts by binding them directly to payment public keys rather than local identifiers, improving security and enabling better verification mechanisms.

## Iteration Context
### Previous Iterations Summary
- **Completed Features**:
  - SPHINCS+ cryptography implementation (PRD-02)
  - Exchange payment with commissions (PRD-06)
  - Payment transaction observing states (PRD-16)
  - Audit mechanism for finalized transactions (PRD-18)
- **Lessons Learned**: Payment public key exchange happens after debt receipt creation in current flow, requiring pre-emptive key exchange during channel setup
- **Technical Debt**: None identified
- **User Feedback**: N/A

### Current State Analysis
- **What's working well**: Debt receipt signing mechanism functions correctly with current structure
- **Pain points identified**:
  - ContractorIDs are local identifiers, not cryptographically verifiable across nodes
  - Audit number is unknown at receipt creation time
  - Payment public keys are exchanged only during payment flow, after receipt creation
- **Performance metrics**: N/A

## Problem Statement
### Background
The debt receipt is a cryptographic proof that one node owes a specific amount to another node. Currently, the receipt signature includes:
- `ContractorID source` - local ID of the sender
- `ContractorID target` - local ID of the recipient
- `BlockNumber mMaximalClaimingBlockNumber`
- `TransactionUUID mTransactionUUID`
- `TrustLineAmount amount`
- `AuditNumber currentAuditNumber`

This structure has several issues:
1. ContractorIDs are local to each node and not cryptographically verifiable
2. The audit number is unknown at receipt creation time (we don't know which audit the receipt will belong to)
3. The equivalent (currency type) is not included in the signed data
4. Payment public keys are only exchanged during payment flow, after receipts need to be created

### Problem Description
- **Who is affected**: All nodes participating in payment transactions
- **When and where**: During debt receipt creation and verification in payment transactions
- **Impact of not solving**:
  - Weaker cryptographic binding of debt receipts
  - Potential for receipt manipulation if ContractorID mappings change
  - Missing equivalent in receipt prevents proper currency verification

### Success Metrics
- **Primary KPIs**:
  - All debt receipts include recipient's payment public key
  - All debt receipts include equivalent
  - Payment public keys exchanged and stored during channel creation
  - Public key verification passes during payment flow
- **Secondary KPIs**:
  - No regression in existing payment functionality
  - All unit and integration tests pass
- **Target Values**: 100% of payment transactions use new receipt structure

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements

1. **Payment Public Key Storage in Contractor Entity**
   - Add `sphincs::PublicKey::Shared` field to `Contractor` class for payment public key storage
   - Add corresponding column `payment_public_key` (BLOB/BYTEA) to `contractors` table in SQLite and PostgreSQL
   - Update `ContractorsHandler` interface and implementations to handle the new field

2. **Payment Public Key Exchange During Channel Creation**
   - Extend `InitChannelMessage` to include sender's payment public key
   - Extend `ConfirmChannelMessage` to include responder's payment public key
   - Update `InitChannelTransaction` to:
     - Retrieve own payment public key from `PaymentKeysHandler`
     - Include it in `InitChannelMessage`
     - Store received payment public key when processing `ConfirmChannelMessage`
     - Return `responseThereAreNoKeys()` if own payment key cannot be retrieved
   - Update `ConfirmChannelTransaction` to:
     - Retrieve own payment public key from `PaymentKeysHandler`
     - Store received payment public key from `InitChannelMessage`
     - Include own payment public key in `ConfirmChannelMessage`
     - Log error and terminate without response if own payment key cannot be retrieved

3. **Debt Receipt Signature Structure Update**
   - Modify `BaseExchangePaymentTransaction::getSerializedReceipt()` to:
     - Remove `ContractorID source` from signed data
     - Remove `ContractorID target` from signed data
     - Remove `AuditNumber currentAuditNumber` from signed data
     - Add `sphincs::PublicKey` (recipient's payment public key) to signed data
     - Add `SerializedEquivalent equivalent` to signed data
   - Update all transactions that use `getSerializedReceipt()`:
     - `CycleCloserIntermediateNodeTransaction`
     - `CycleCloserInitiatorTransaction`
     - `ConflictResolverContractorTransaction`

4. **Payment Public Key Verification During Payment**
   - Add verification in payment transaction flow to check that public keys received from coordinator match the payment public keys stored in corresponding `Contractor` entities
   - Return same error as hash mismatch if keys don't match
   - Coordinator verifies `FinalAmountsConfigurationResponseMessage` public keys for neighbors with reservations

#### Bug Fixes & Technical Improvements
- None

#### Modifications to Existing Features
- `Contractor` class: Add payment public key field
- `ContractorsHandler` interface and implementations: Support payment public key persistence
- `InitChannelMessage`: Add payment public key field
- `ConfirmChannelMessage`: Add payment public key field
- `InitChannelTransaction`: Handle payment public key exchange
- `ConfirmChannelTransaction`: Handle payment public key exchange
- `BaseExchangePaymentTransaction::getSerializedReceipt()`: New signature structure

### Explicitly Out of Scope
- Migration of existing channels (not in production, clean deployment assumed)
- Backwards compatibility with old receipt format
- Re-synchronization mechanism for existing channels

### Dependencies from Previous Iterations
- PRD-02: SPHINCS+ Cryptography Implementation (provides `sphincs::PublicKey` types)
- PRD-06: Exchange Payment with Commissions (provides payment transaction base)
- Existing `PaymentKeysHandler` interface for retrieving own payment public key

### Future Roadmap Impact
- This change establishes a more secure foundation for debt receipt verification
- Enables future enhancements to receipt verification mechanisms

## User Stories & Requirements

### User Personas
#### Primary User: VTCP Node Operator
- **Role**: Operates a node in the VTCP network
- **Goals**: Secure and verifiable payment transactions
- **Pain Points**: Need assurance that debt receipts are cryptographically bound to correct parties
- **Technical Proficiency**: Advanced

### Functional Requirements
#### New Features for This Iteration

1. **[FR-1] Payment Public Key Storage in Contractor**
   - **Description**: Store contractor's payment public key alongside channel encryption key
   - **User Story**: As a node operator, I want the payment public keys of my contractors stored locally so that I can verify their identity during payments
   - **Rationale**: Required for receipt signing and verification during payment flow
   - **Builds Upon**: Existing `Contractor` class and `ContractorsHandler` interface
   - **Acceptance Criteria**:
     - `Contractor` class has `sphincs::PublicKey::Shared` field for payment public key
     - Payment public key can be set and retrieved from `Contractor` instance
     - Payment public key is persisted to and loaded from database
   - **Priority**: High
   - **Dependencies**: None

2. **[FR-2] Payment Public Key Exchange in InitChannelMessage**
   - **Description**: Include sender's payment public key in channel initialization message
   - **User Story**: As a node initiating a channel, I want to send my payment public key so the counterparty can verify my receipts
   - **Rationale**: Channel creation is the appropriate time to exchange payment keys before any transactions
   - **Builds Upon**: Existing `InitChannelMessage` structure
   - **Acceptance Criteria**:
     - `InitChannelMessage` includes `sphincs::PublicKey::Shared` field
     - Key is serialized/deserialized correctly in message
     - Recipient can extract and store the payment public key
   - **Priority**: High
   - **Dependencies**: FR-1

3. **[FR-3] Payment Public Key Exchange in ConfirmChannelMessage**
   - **Description**: Include responder's payment public key in channel confirmation message
   - **User Story**: As a node confirming a channel, I want to send my payment public key so the initiator can verify my receipts
   - **Rationale**: Completes the bidirectional key exchange during channel setup
   - **Builds Upon**: Existing `ConfirmChannelMessage` structure
   - **Acceptance Criteria**:
     - `ConfirmChannelMessage` includes `sphincs::PublicKey::Shared` field
     - Key is serialized/deserialized correctly in message
     - Initiator can extract and store the payment public key
   - **Priority**: High
   - **Dependencies**: FR-1

4. **[FR-4] InitChannelTransaction Payment Key Handling**
   - **Description**: Update transaction to exchange payment public keys
   - **User Story**: As a channel initiator, I want the channel creation to automatically exchange payment keys
   - **Rationale**: Automated key exchange ensures keys are available before any payment
   - **Builds Upon**: Existing `InitChannelTransaction` flow
   - **Acceptance Criteria**:
     - Transaction retrieves own payment public key from `PaymentKeysHandler`
     - Own key included in `InitChannelMessage`
     - Received key from `ConfirmChannelMessage` stored in `Contractor` and DB
     - Returns `responseThereAreNoKeys()` if own key unavailable
   - **Priority**: High
   - **Dependencies**: FR-2, FR-3

5. **[FR-5] ConfirmChannelTransaction Payment Key Handling**
   - **Description**: Update transaction to exchange payment public keys
   - **User Story**: As a channel responder, I want to receive and send payment public keys during confirmation
   - **Rationale**: Automated key exchange ensures keys are available before any payment
   - **Builds Upon**: Existing `ConfirmChannelTransaction` flow
   - **Acceptance Criteria**:
     - Transaction retrieves own payment public key from `PaymentKeysHandler`
     - Received key from `InitChannelMessage` stored in `Contractor` and DB
     - Own key included in `ConfirmChannelMessage`
     - Logs error and terminates without response if own key unavailable (silent termination)
   - **Priority**: High
   - **Dependencies**: FR-2, FR-3

6. **[FR-6] Updated Debt Receipt Signature Structure**
   - **Description**: Modify signed data structure for debt receipts
   - **User Story**: As a node, I want debt receipts to include cryptographically verifiable recipient identity
   - **Rationale**: Stronger cryptographic binding than local ContractorIDs
   - **Builds Upon**: Existing `getSerializedReceipt()` method
   - **Acceptance Criteria**:
     - `source` (ContractorID) removed from signed data
     - `target` (ContractorID) removed from signed data
     - `currentAuditNumber` removed from signed data
     - `sphincs::PublicKey` (recipient's payment key) added to signed data
     - `SerializedEquivalent equivalent` added to signed data
     - `recipientPaymentPublicKey` is retrieved from `ContractorsHandler` in all transaction types
   - **Priority**: High
   - **Dependencies**: FR-1, FR-4, FR-5

7. **[FR-7] Payment Public Key Verification During Payment**
   - **Description**: Verify coordinator-provided keys match stored channel keys
   - **User Story**: As a node, I want to verify that payment participants have the expected keys
   - **Rationale**: Prevents man-in-the-middle attacks using different keys
   - **Builds Upon**: Existing `checkPublicKeysAppropriate()` method
   - **Acceptance Criteria**:
     - For each neighbor in payment path, verify coordinator's provided public key matches stored `paymentPublicKey` in `Contractor`
     - Return same error code as hash mismatch if keys don't match
     - Verification applies to neighbors with reservations (incoming or outgoing)
     - If stored `paymentPublicKey` is `NULL` for an existing channel, verification fails and payment is rejected
     - Coordinator rejects when a neighbor's `FinalAmountsConfigurationResponseMessage` public key mismatches or is absent
   - **Priority**: High
   - **Dependencies**: FR-1, FR-6

8. **[FR-8] Update Cycle Closer Transactions**
   - **Description**: Update CycleCloserIntermediateNodeTransaction and CycleCloserInitiatorTransaction to use new receipt structure
   - **User Story**: As a node participating in cycle closing, I want receipts to use the updated signature structure
   - **Rationale**: All payment-related transactions must use consistent receipt format
   - **Builds Upon**: `getSerializedReceipt()` changes
   - **Acceptance Criteria**:
     - Both transactions pass correct parameters to `getSerializedReceipt()`
     - Receipts are created with new structure
   - **Priority**: High
   - **Dependencies**: FR-6

9. **[FR-9] Update ConflictResolverContractorTransaction**
   - **Description**: Update ConflictResolverContractorTransaction to use new receipt structure
   - **User Story**: As a node resolving conflicts, I want to verify receipts with the updated structure
   - **Rationale**: Conflict resolution must handle same receipt format as creation
   - **Builds Upon**: `getSerializedReceipt()` changes
   - **Acceptance Criteria**:
     - Transaction passes correct parameters to `getSerializedReceipt()`
     - Receipt verification works with new structure
   - **Priority**: High
   - **Dependencies**: FR-6

#### Testing Requirements

10. **[FR-10] Unit Tests for InitChannelMessage**
    - **Description**: Create/update unit tests for InitChannelMessage serialization
    - **User Story**: As a developer, I want comprehensive tests for message serialization
    - **Rationale**: Ensure message correctly handles payment public key
    - **Acceptance Criteria**:
      - Tests for serialization with valid payment public key
      - Tests for deserialization and key extraction
      - Tests for edge cases (null key handling if applicable)
    - **Priority**: High
    - **Dependencies**: FR-2

11. **[FR-11] Unit Tests for ConfirmChannelMessage**
    - **Description**: Create/update unit tests for ConfirmChannelMessage serialization
    - **User Story**: As a developer, I want comprehensive tests for message serialization
    - **Rationale**: Ensure message correctly handles payment public key
    - **Acceptance Criteria**:
      - Tests for serialization with valid payment public key
      - Tests for deserialization and key extraction
      - Tests for edge cases (null key handling if applicable)
    - **Priority**: High
    - **Dependencies**: FR-3

12. **[FR-12] Update ContractorsHandler Tests**
    - **Description**: Update unit and integration tests for ContractorsHandler implementations
    - **User Story**: As a developer, I want tests to cover payment public key persistence
    - **Rationale**: Ensure database operations handle new field correctly
    - **Acceptance Criteria**:
      - `ContractorsHandlerSQLiteTest` updated with payment public key tests
      - `ContractorsHandlerPostgreSQLIntegrationTest` updated with payment public key tests
      - Tests cover save, update, and retrieve operations with payment public key
    - **Priority**: High
    - **Dependencies**: FR-1

### Non-Functional Requirements
#### Performance
- No significant performance impact expected (key serialization is minimal overhead)

#### Security
- Payment public keys must be stored securely in database
- Key verification must be performed before accepting receipts
- No fallback to old receipt format allowed

#### Scalability
- N/A (no scalability concerns for this change)

#### Reliability
- Channel creation must fail cleanly if payment keys cannot be retrieved
- Payment transactions must reject mismatched keys

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Debt receipts use ContractorID for party identification
- **Proposed Changes**: Debt receipts use payment public keys for cryptographic identification
- **Backwards Compatibility**: Not required (not in production)
- **Protocol Versioning**: Not required; mixed versions are not supported
- **Migration Requirements**: None (clean deployment)

### Technology Stack Updates
#### New Technologies/Libraries
- None (uses existing SPHINCS+ implementation from PRD-02)

### Data Requirements
#### Data Models

**Contractor Class Updates:**
```cpp
class Contractor {
    // Existing fields...
    sphincs::PublicKey::Shared mPaymentPublicKey;

public:
    // New methods
    sphincs::PublicKey::Shared paymentPublicKey() const;
    void setPaymentPublicKey(sphincs::PublicKey::Shared key);
};
```

**Database Schema Updates:**

SQLite:
```sql
ALTER TABLE contractors ADD COLUMN payment_public_key BLOB; -- nullable for existing channels
```

PostgreSQL:
```sql
ALTER TABLE contractors ADD COLUMN payment_public_key BYTEA; -- nullable for existing channels
```

**Receipt Signature Data Structure (New):**
```
sphincs::PublicKey recipientPaymentPublicKey  // Recipient's payment public key
BlockNumber mMaximalClaimingBlockNumber       // Max claiming block
TransactionUUID mTransactionUUID              // Transaction ID
TrustLineAmount amount                        // Amount
SerializedEquivalent equivalent               // Currency equivalent
```
**Receipt Serialization Note:**
- `recipientPaymentPublicKey` is serialized using the same format and endianness as in payment flow messages `ParticipantVoteMessage` and `ParticipantsVotesMessage`.

### Files Requiring Modification

| File | Changes |
|------|---------|
| `src/core/contractors/Contractor.h` | Add `mPaymentPublicKey` field and accessor methods |
| `src/core/contractors/Contractor.cpp` | Implement payment public key handling |
| `src/core/io/storage/interfaces/ContractorsHandler.h` | Add methods for payment public key persistence |
| `src/core/io/storage/sqlite/ContractorsHandlerSQLite.h` | Implement new interface methods |
| `src/core/io/storage/sqlite/ContractorsHandlerSQLite.cpp` | Implement SQLite storage for payment key |
| `src/core/io/storage/postgresql/ContractorsHandlerPostgreSQL.h` | Implement new interface methods |
| `src/core/io/storage/postgresql/ContractorsHandlerPostgreSQL.cpp` | Implement PostgreSQL storage for payment key |
| `src/core/network/messages/trust_line_channels/InitChannelMessage.h` | Add payment public key field |
| `src/core/network/messages/trust_line_channels/InitChannelMessage.cpp` | Serialize/deserialize payment key |
| `src/core/network/messages/trust_line_channels/ConfirmChannelMessage.h` | Add payment public key field |
| `src/core/network/messages/trust_line_channels/ConfirmChannelMessage.cpp` | Serialize/deserialize payment key |
| `src/core/transactions/transactions/trust_line_channel/InitChannelTransaction.h` | Add PaymentKeysHandler dependency |
| `src/core/transactions/transactions/trust_line_channel/InitChannelTransaction.cpp` | Implement key exchange logic |
| `src/core/transactions/transactions/trust_line_channel/ConfirmChannelTransaction.h` | Add PaymentKeysHandler dependency |
| `src/core/transactions/transactions/trust_line_channel/ConfirmChannelTransaction.cpp` | Implement key exchange logic |
| `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.h` | Update `getSerializedReceipt()` signature |
| `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.cpp` | Implement new receipt structure |
| `src/core/transactions/transactions/regular/payments/CycleCloserIntermediateNodeTransaction.cpp` | Update receipt creation calls |
| `src/core/transactions/transactions/regular/payments/CycleCloserInitiatorTransaction.cpp` | Update receipt creation calls |
| `src/core/transactions/transactions/regular/payments/ConflictResolverContractorTransaction.cpp` | Update receipt verification |
| `tests/unit/sqlite/ContractorsHandlerSQLiteTest.cpp` | Add payment public key tests |
| `tests/storage/integration/postgresql/ContractorsHandlerPostgreSQLIntegrationTest.cpp` | Add payment public key tests |
| New: `tests/unit/messages/InitChannelMessageTest.cpp` | Create unit tests |
| New: `tests/unit/messages/ConfirmChannelMessageTest.cpp` | Create unit tests |

## Implementation Plan
### This Iteration Timeline
- **Duration**: TBD
- **Key Deliverables**:
  - Updated Contractor class with payment public key
  - Updated database schema and handlers
  - Updated channel creation messages and transactions
  - Updated debt receipt signature structure
  - Payment public key verification in payment flow
  - Comprehensive test coverage

### Iteration Milestones
| Milestone | Description | Dependencies | Risk Level |
|-----------|-------------|--------------|------------|
| M1 | Contractor class and DB schema updates | None | Low |
| M2 | Message updates (InitChannel, ConfirmChannel) | M1 | Low |
| M3 | Channel transaction updates | M2 | Medium |
| M4 | Receipt signature structure update | M3 | Medium |
| M5 | Payment verification update | M4 | Medium |
| M6 | Cycle closer and conflict resolver updates | M4 | Low |
| M7 | Test implementation | M1-M6 | Low |

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Message format incompatibility | High | Low | Thorough testing of serialization/deserialization |
| Key retrieval failure handling | Medium | Low | Clear error paths defined in requirements |
| Receipt verification regression | High | Medium | Comprehensive test coverage before changes |

## Testing Strategy
### Testing Approach for This Iteration
#### New Feature Testing
- Unit tests for `InitChannelMessage` serialization with payment key
- Unit tests for `ConfirmChannelMessage` serialization with payment key
- Unit tests for `Contractor` payment public key handling
- Unit tests for `ContractorsHandlerSQLite` payment key persistence
- Integration tests for `ContractorsHandlerPostgreSQL` payment key persistence
- Integration tests for channel creation with key exchange
- Integration tests for payment flow with key verification

#### Regression Testing
- **Scope**: All existing payment transaction tests
- **Critical User Journeys**:
  - Channel creation flow
  - Payment transaction flow
  - Cycle closing flow
  - Conflict resolution flow

### Quality Gates
- All existing tests pass
- New unit tests achieve 100% coverage of new code
- Integration tests verify end-to-end key exchange
- Manual verification of receipt structure in test transactions

## Appendices
### Glossary
| Term | Definition |
|------|------------|
| Debt Receipt | Cryptographic proof that one node owes an amount to another |
| Payment Public Key | SPHINCS+ public key used for signing payment-related data |
| Contractor | Entity representing a peer node with which a trust line channel exists |
| Trust Line Channel | Bidirectional credit relationship between two nodes |
| Equivalent | Numeric identifier for currency type in the VTCP network |

### References
- [SPHINCS+ Specification](https://sphincs.org/)
- [Payment Protocol Documentation](../../architecture/vtcpd/protocols/payment-protocol.md)

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2026-01-24 | Architect | Initial draft | Phase 1 |

**Related Documents**
- **Previous Iteration PRD**: [PRD-06: Exchange Payment with Commissions](06-exchange-payment-with-commissions.md)
- **Technical Architecture**: [Payment Protocol](../../architecture/vtcpd/protocols/payment-protocol.md)
- **Cryptography PRD**: [PRD-02: SPHINCS+ Cryptography Implementation](02_sphincs_plus_cryptography_implementation.md)
