# Project Requirements Document (PRD)

## Document Information
- **Project Name**: VTCPD SPHINCS+ Cryptography Implementation
- **Phase/Iteration**: Phase 2, Core Cryptography Implementation
- **Document Version**: 1.0
- **Date**: 2024-12-19
- **Author(s)**: AI Development Assistant
- **Stakeholders**: Core Development Team, Security Team
- **Status**: Draft
- **Previous PRD**: N/A
- **Related Documents**: [ADR-004: SPHINCS+ Signature Scheme Adoption](../../architecture/btc-federation/ADR-004-sphincs-signature-scheme.md)

## Executive Summary
Brief overview of this iteration, its purpose, and expected outcomes. Include:
- **Current project state**: VTCPD currently uses Lamport one-time signature scheme for cryptographic operations in trust lines and payment transactions
- **This iteration's focus**: Replace Lamport signature scheme with SPHINCS+ post-quantum signature scheme using OpenSSL 3.5+ deterministic variant
- **Connection to overall vision**: This implementation enhances security posture, improves performance, and simplifies key management while maintaining post-quantum cryptographic security

## Iteration Context
### Previous Iterations Summary
- **Completed Features**: Lamport signature implementation, trust line cryptography, payment transaction signing
- **Lessons Learned**: One-time key management complexity, large key storage requirements
- **Technical Debt**: Complex key rotation logic, database bloat from multiple keys per trust line
- **User Feedback**: Performance concerns with key generation and storage overhead

### Current State Analysis
- **What's working well**: Secure post-quantum cryptography framework, established trust line relationships
- **Pain points identified**: Key exhaustion issues, complex key management, large storage footprint
- **Performance metrics**: Current system operates in development environment without production constraints

## Problem Statement
### Background
The current VTCPD implementation uses Lamport one-time signature scheme, which requires generating and storing multiple key pairs per trust line. Each key can only be used once, leading to complex key management, potential key exhaustion scenarios, and significant storage overhead.

### Problem Description
Clearly articulate the problem this project aims to solve. Include:
- **Who is affected**: All VTCPD development and testing environments, future production deployments
- **When and where**: During trust line operations, payment processing, and cryptographic operations
- **Impact of not solving**: Continued key management complexity, poor scalability, inefficient resource usage

### Success Metrics
Define how success will be measured:
- **Primary KPIs**: 
  - Successfully implement SPHINCS+ deterministic signatures
  - Reduce key storage requirements by >90%
  - Maintain or improve signature generation/verification performance
- **Secondary KPIs**: 
  - Zero cryptographic security regressions
  - Successful database schema updates
  - All existing functionality preserved with new crypto backend
- **Target Values**: 
  - 100% functional test pass rate
  - Memory usage reduction of at least 80%
  - All cryptographic operations working with SPHINCS+

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- **OpenSSL 3.5+ Integration**: Add OpenSSL 3.5+ dependency with SPHINCS+ SLH-DSA-SHA2-256s support (OpenSSL NID = EVP_SIGNATURE-SLH-DSA-SHA2-256s)
- **SPHINCS+ Deterministic Primitives**: Implement PrivateKey, PublicKey, and Signature classes for deterministic SPHINCS+ scheme
- **Simplified Key Management**: Replace multi-key architecture with single key per trust line
- **Database Schema Updates**: Remove `number` and `is_valid` fields from key-related tables while preserving `keys_set_sequence_number` field

#### Bug Fixes & Technical Improvements
- **Key Exhaustion Prevention**: Eliminate one-time key limitation issues
- **Storage Optimization**: Reduce database size and memory footprint
- **Performance Improvements**: Optimize signature operations with reusable keys

#### Modifications to Existing Features
- **Keychain Methods**: Update all signing and verification methods in `keychain.h/cpp` to use SPHINCS+
- **Trust Line Transactions**: Remove KeyNumber parameters from all trust line operations
- **Payment Transactions**: Remove KeyNumber parameters from payment-related operations
- **Database Handlers**: Update SQLite and PostgreSQL handlers to simplified schema (remove `number` and `is_valid` fields, preserve `keys_set_sequence_number`)

### Explicitly Out of Scope
- Data migration from existing systems (project not in production)
- Backward compatibility with Lamport signatures
- Changes to trust line business logic beyond cryptographic operations
- Network protocol modifications
- User interface changes
- Comparative performance testing between schemes

### Dependencies from Previous Iterations
- Existing trust line infrastructure framework
- Current database handler interfaces
- Payment transaction framework structure
- Audit and receipt mechanism patterns

### Future Roadmap Impact
This implementation establishes a modern cryptographic foundation that simplifies system architecture and enables easier maintenance for future development.

## User Stories & Requirements

### User Personas
#### Primary User: VTCPD Developer
- **Role**: Core system developer
- **Goals**: Implement secure, efficient cryptographic operations
- **Pain Points**: Complex multi-key management, key exhaustion scenarios
- **Technical Proficiency**: Advanced

#### Secondary User: System Integrator
- **Role**: Developer integrating with VTCPD APIs
- **Goals**: Simple, reliable cryptographic interfaces
- **Pain Points**: Complex key lifecycle management
- **Technical Proficiency**: Intermediate

### Functional Requirements
#### New Features for This Iteration
1. **OpenSSL 3.5+ Integration with Deterministic SPHINCS+**
   - **Description**: Integrate OpenSSL 3.5+ with deterministic SPHINCS+ SLH-DSA-SHA2-256s algorithm
   - **User Story**: As a developer, I want to use industry-standard OpenSSL 3.5+ library with deterministic SPHINCS+ for consistent, reproducible signatures
   - **Rationale**: Deterministic variant ensures reproducible signatures for the same input and key, which is important for system consistency
   - **Builds Upon**: Current crypto namespace structure
   - **Acceptance Criteria**: 
     - OpenSSL 3.5+ successfully integrated into build system
     - EVP_SIGNATURE-SLH-DSA-SHA2-256s algorithm available and functional
     - Deterministic variant properly configured
     - No conflicts with existing dependencies
   - **Priority**: High
   - **Dependencies**: Build system updates, CMakeLists.txt modifications

2. **SPHINCS+ Cryptographic Primitives**
   - **Description**: Implement PrivateKey, PublicKey, and Signature classes for deterministic SPHINCS+ scheme
   - **User Story**: As a developer, I want SPHINCS+ primitives that provide consistent interfaces for cryptographic operations
   - **Rationale**: Maintains code consistency while providing new cryptographic backend with deterministic behavior
   - **Builds Upon**: Existing crypto namespace and interfaces
   - **Acceptance Criteria**: 
     - PrivateKey class with key generation and secure storage
     - PublicKey class with key derivation and storage
     - Signature class with deterministic signing and verification
     - Memory-safe implementations
     - Thread-safe operations
   - **Priority**: High
   - **Dependencies**: OpenSSL integration

3. **Single Key Architecture**
   - **Description**: Replace multi-key sets with single reusable key pair per trust line
   - **User Story**: As a developer, I want simplified key management with single persistent key per trust line
   - **Rationale**: SPHINCS+ allows key reuse, eliminating need for complex key set management
   - **Builds Upon**: Existing trust line structure
   - **Acceptance Criteria**: 
     - One key pair per trust line maximum
     - No key rotation/regeneration logic required
     - Simplified key access patterns
     - Key persistence across multiple operations
   - **Priority**: High
   - **Dependencies**: SPHINCS+ primitives, database schema changes

#### Enhancements to Existing Features
1. **Keychain Method Updates**
   - **Current State**: Methods use Lamport signatures with KeyNumber parameters
   - **Proposed Changes**: 
     - Replace Lamport calls with SPHINCS+ deterministic operations
     - Remove KeyNumber parameters from all methods
     - Simplify key retrieval to single key per trust line
     - Update all signing methods: `sign()`, `saveOutgoingPaymentReceipt()`, `saveIncomingPaymentReceipt()`, `saveFullAudit()`, etc.
   - **Impact Assessment**: Complete replacement of cryptographic backend
   - **Migration Strategy**: Direct replacement (no backward compatibility needed)

2. **Database Schema Simplification**
   - **Current State**: Tables store multiple keys with `number` and `is_valid` fields
   - **Proposed Changes**: 
     - Remove `number` field from all key-related tables
     - Remove `is_valid` field from all key-related tables
     - Preserve `keys_set_sequence_number` field for key set versioning
     - Simplify primary key constraints
     - Update indexes for single-key lookup
   - **Impact Assessment**: Simplified database schema
   - **Migration Strategy**: Schema update without data preservation

3. **Payment Transactions Single-Key Refactor**
   - **Description**: Unify payment transaction signing to use a single reusable payment key (SPHINCS+) across all transactions; eliminate per-transaction key coupling.
   - **Requirements**:
     1) Update `payment_keys` table: remove `transaction_uuid`; add numeric `id` (auto-increment, unique) and create an index on it.
     2) Update `payment_transactions` table: add `payment_key_id` referencing `payment_keys(id)` as a foreign key.
     3) Change `getOwnPrivateKey` in `src/core/io/storage/interfaces/PaymentKeysHandler.h` and implementations: remove `transactionUUID` parameter; return the key with the largest `id`.
     4) In payment transactions under `src/core/transactions/transactions/regular/payments/`, remove code that generates a payment key per transaction.
     5) In payment transactions signing flow, obtain the key by the maximum `id` instead of by `transactionUUID`.
     6) Startup behavior: implement the key existence check in `Keystore::init()` (`src/core/crypto/keychain.h` / `.cpp`). Add a `Keystore` method to check key presence and call it from `init()`. Add a similar presence-check method to `PaymentKeysHandler` interface and all implementations.
     7) Replace `deleteKeyByTransactionUUID` with `deleteKeyByID(id)` across `PaymentKeysHandler` interface and implementations.
     8) Remove `allTransactionUUIDs` from `PaymentKeysHandler` and remove `removeOutdatedPaymentsKeysData` from `src/core/crypto/keychain.h` together with all its invocations.
   - **Acceptance Criteria**:
     - SQLite and PostgreSQL schema definitions updated (no migrations required)
     - Field naming is `payment_key_id` in `payment_transactions`
     - Startup path ensures at least one payment key via `Keystore::init()`
     - All storage handlers updated and compiled
     - Payment transaction code updated to single-key flow and compiled
     - Unit tests updated/added to verify max-id key retrieval and startup generation path; all tests pass
 
4. **Audit Model Simplification (Single-Key Architecture Alignment)**
   - **Current State**: Audit model and storage track per-record `ownKeyHash`, `contractorKeyHash`, `ownKeysSetHash`, and `contractorKeysSetHash` to support Lamport multi-key sets.
   - **Proposed Changes**:
     - Remove fields from audit model: `ownKeyHash`, `contractorKeyHash`, `ownKeysSetHash`, `contractorKeysSetHash`.
     - Update class `AuditRecord` to drop these fields, related constructors, accessors, and serialization footprint.
     - Update interface `AuditHandler` and implementations (SQLite and PostgreSQL) to remove these parameters from methods such as `saveFullAudit`, `saveOwnAuditPart`, `saveContractorAuditPart`, and any queries relying on key hashes.
     - Remove `AuditHandler::isContainsKeyHash(...)` from the interface and both implementations; delete all usages (e.g., outdated key cleanup paths) and simplify logic accordingly.
     - Update DB schemas used by the storage implementations to remove corresponding columns from audit tables (SQLite and PostgreSQL) while preserving audit numbers and amounts/balance fields.
     - Adjust `BaseTrustLineTransaction` methods `getOwnSerializedAuditData` and `getContractorSerializedAuditData` to exclude key-hash and keys-set-hash data from serialization; update method signatures and all call sites.
     - Update keychain audit conflict helpers `checkOwnConflictedSignature` and `checkContractorConflictedSignature` to drop key-hash parameters and work with a single key.
     - Remove `checkKeysSetAppropriate(...)` and all usages since keys-set hashes are no longer used.
     - Remove methods `ownPublicKeysHash()` and `contractorPublicKeysHash()` from `Keystore/TrustLineKeychain` interfaces and all call sites.
     - Update unit/integration tests and fixtures to reflect removed fields, updated signatures, and simplified schemas so all tests compile.
   - **Rationale**: With SPHINCS+ single reusable key per trust line, storing key hash and keys-set-hash is unnecessary.
   - **Acceptance Criteria**:
     - Code compiles after removing fields and method parameters/usages across `AuditRecord`, `AuditHandler` and its implementations, and transaction serialization helpers.
     - Storage handlers (SQLite and PostgreSQL) compile with simplified audit schemas.
     - All references to removed hash fields/methods are eliminated (including `isContainsKeyHash`, `ownPublicKeysHash`, `contractorPublicKeysHash`, and `checkKeysSetAppropriate`).
     - Unit and integration tests compile successfully under the updated interfaces and schemas.

#### Bug Fixes & Technical Improvements
- **Key Exhaustion Elimination**: Remove all logic related to key depletion and availability counting
- **Storage Optimization**: Reduce key storage requirements by >90%
- **Memory Management**: Optimize key loading with single key per trust line

### Non-Functional Requirements
#### Performance
- Signature generation time: Efficient with reusable keys
- Signature verification time: Fast verification with public keys
- Key generation time: < 100ms per key pair
- Memory usage: Significantly reduced compared to multi-key approach

#### Security
- Post-quantum security level maintained
- Deterministic signature generation for consistency
- Secure key storage and handling
- Memory protection for private keys
- Cryptographic randomness for key generation

#### Scalability
- Support current trust line capacity with improved efficiency
- Reduced storage growth rate
- Improved memory utilization
- Simplified key management scaling

#### Reliability
- Robust error handling for cryptographic operations
- Consistent deterministic behavior
- Thread-safe operations
- Proper resource cleanup

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Multi-key Lamport scheme with complex key lifecycle management
- **Proposed Changes**: Single-key deterministic SPHINCS+ scheme with simplified management
- **Backwards Compatibility**: Not required (pre-production)
- **Migration Requirements**: Direct replacement of cryptographic implementation

### Technology Stack Updates
#### New Technologies/Libraries
- **OpenSSL 3.5+**: Core cryptographic library providing deterministic SPHINCS+ implementation
- **Specific Algorithm (OpenSSL NID)**: EVP_SIGNATURE-SLH-DSA-SHA2-256s (deterministic variant)
- **Integration approach**: Use OpenSSL EVP interface for SPHINCS+ operations

#### Version Updates
- **OpenSSL**: Not currently used → OpenSSL 3.5+
- **Impact**: New dependency, build system updates required, minimum version 3.5+ for SPHINCS+ support

### Integration Requirements
#### New Integrations
- **OpenSSL 3.5+**: Direct integration for deterministic SPHINCS+ cryptographic operations

#### Modified Integrations
- **Database layer**: Updated to handle simplified single-key schema
- **Crypto namespace**: Complete replacement with SPHINCS+ implementation

### Data Requirements
#### Data Models
- **Key storage**: Single key pair per trust line instead of key sets
- **Signature storage**: SPHINCS+ signature format instead of Lamport
- **Audit records**: Updated to reference single keys without key numbers

#### Data Storage
- Significantly reduced storage requirements for keys
- Simplified database schema
- Optimized database indexes for single-key lookups

#### Data Migration
- No data migration required (pre-production)
- Clean implementation without legacy data concerns

## Implementation Plan
### This Iteration Timeline
- **Duration**: 4-6 weeks
- **Sprint Breakdown**: 3 sprints of 2 weeks each
- **Key Deliverables**: 
  - Week 2: OpenSSL integration and SPHINCS+ primitives
  - Week 4: Database schema updates and keychain method implementations
  - Week 6: Testing, validation, and cleanup

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| OpenSSL Integration | Week 1 | OpenSSL 3.5+ integrated with deterministic SPHINCS+ | Build system | Low |
| SPHINCS+ Primitives | Week 2 | Core cryptographic classes implemented | OpenSSL integration | Medium |
| Database Schema Update | Week 3 | Schema simplified, handlers updated | SPHINCS+ primitives | Medium |
| Keychain Implementation | Week 4 | All keychain methods updated | Database changes | Medium |
| System Integration | Week 5 | All components working together | All previous milestones | High |
| Testing & Validation | Week 6 | Comprehensive testing complete | System integration | Medium |

### Dependencies on Other Teams/Projects
- **Infrastructure Team**: OpenSSL package availability
- **Build System**: CMakeLists.txt updates for OpenSSL linking

### Integration Points with Previous Work
- Maintains existing trust line business logic interfaces
- Preserves audit and receipt functionality patterns
- Compatible with existing network communication patterns

### Resource Requirements
#### Team Structure
- **Technical Lead**: Senior developer with cryptographic experience
- **Developers**: 2 developers familiar with the codebase
- **QA Engineers**: 1 engineer for testing

#### Budget Estimates
- Development effort: 8-12 person-weeks
- Infrastructure costs: Minimal (OpenSSL is open source)

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| OpenSSL Integration Issues | High | Medium | Proof of concept first, documentation review |
| SPHINCS+ Performance Issues | Medium | Low | Early performance validation, parameter tuning |
| Database Schema Update Issues | Medium | Low | Careful testing, staged deployment |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Implementation Delays | Medium | Medium | Adequate buffer time, parallel development |
| Security Issues | High | Low | Security review, thorough testing |

## Testing Strategy
### Testing Approach for This Iteration
#### New Feature Testing
- Unit tests for SPHINCS+ primitives
- Deterministic output test: repeat signing on identical input yields identical signature bytes
- Integration tests for keychain operations with new crypto backend
- End-to-end tests for trust line operations
- Database handler tests with simplified schema

#### Regression Testing
- **Scope**: All existing functionality must work with new cryptographic backend
- **Automated vs Manual**: Automated testing for core functionality
- **Critical User Journeys**: Trust line creation, payment processing, audit operations

#### Performance Testing
- **Baseline Metrics**: Current development environment performance
- **Target Metrics**: Improved or maintained performance with SPHINCS+
- **Load Testing**: Multi-threading and concurrent operations

### Quality Gates
- All test targets (unit + integration) are included in the CMake build; no tests are disabled
- All test suites (SQLite and PostgreSQL) compile successfully under the updated SPHINCS+ APIs (passing execution is not required in this iteration)
- New SPHINCS+ tests compile successfully
- Performance benchmarks meet requirements
- Security review completed
- Code quality standards met

### Build and Test Suite Requirements (Iteration 2)
To support the migration, the following explicit requirements apply to the test infrastructure in this iteration:
- Update database-related tests (SQLite and PostgreSQL) to the simplified single-key schema and new interfaces (e.g., remove `number` and `is_valid` usage; use `getPublicKey`, `getPublicKeyHash`, `hasKey`, `invalidateKey`, `deleteKeyByTrustLineID`/`deleteKeyByID`, etc.).
- Ensure all tests are part of the build (unit and integration), with no tests excluded by CMake.
- Success criteria for this iteration with regard to tests is a successful compilation of all test targets; passing runtime execution of tests is not required.

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Major implementation update
- **Rollout Strategy**: Direct replacement in development environment
- **Rollback Plan**: Git revert capability

### Feature Flags & Gradual Rollout
Not applicable - direct replacement approach

### Database Migrations
- **Schema Update**: Remove obsolete fields, update constraints
- **Impact**: Clean schema without legacy concerns

### Communication Plan
- **Internal**: Regular updates to development team
- **Documentation Updates**: Updated technical documentation and code comments

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**: 
  - All tests passing with new implementation
  - Storage reduction achieved (>90%)
  - Performance maintained or improved
- **Leading Indicators**: Unit test pass rates, integration test results
- **Baseline Values**: Current development environment metrics
- **Target Values**: Full functionality with improved efficiency

### Monitoring Plan
- **New Monitoring**: SPHINCS+ operation monitoring
- **Enhanced Monitoring**: Key storage metrics, signature performance
- **Validation**: Deterministic signature consistency checks

### Review Schedule
- **Daily**: Development progress and test results
- **Weekly**: Integration progress and performance metrics
- **Post-Implementation**: Full system validation and performance assessment

## Appendices
### Glossary
- **SPHINCS+**: Post-quantum digital signature scheme
- **SLH-DSA-SHA2-256s**: Specific SPHINCS+ variant using SHA2-256 with 256-bit security and small signatures
- **Deterministic Variant**: SPHINCS+ mode that produces consistent signatures for same input and key
- **Trust Line**: Cryptographic relationship between two parties
- **EVP Interface**: OpenSSL's high-level cryptographic interface

### References
- OpenSSL 3.5+ SPHINCS+ documentation
- EVP_SIGNATURE-SLH-DSA-SHA2-256s specification
- SPHINCS+ deterministic variant documentation
- NIST Post-Quantum Cryptography Standardization (FIPS 203 Draft – SPHINCS+)
- Current VTCPD cryptographic architecture

### Wireframes/Mockups
N/A - Internal cryptographic changes without UI impact

### API Documentation
Updated documentation for:
- Keychain interface changes
- Database handler modifications
- SPHINCS+ cryptographic primitive APIs

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2024-12-19 | AI Assistant | Initial draft for SPHINCS+ implementation | Phase 2 |

**Related Documents**
- **Master Project Vision**: [Link to overall VTCPD vision]
- **Technical Architecture**: [Link to current architecture docs]
- **Architecture Decision Record**: [ADR-004: SPHINCS+ Signature Scheme Adoption](../../architecture/btc-federation/ADR-004-sphincs-signature-scheme.md)
- **OpenSSL Integration Guide**: [Link to integration documentation]