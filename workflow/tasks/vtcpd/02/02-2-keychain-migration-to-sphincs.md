# 02-2 - Keychain Migration to SPHINCS+

# Links
- [PRD-02: SPHINCS+ Cryptography Implementation](../../prd/vtcpd/02_sphincs_plus_cryptography_implementation.md)
- [Task 02-1: OpenSSL Integration and SPHINCS+ Primitives](./02-1-openssl-integration-sphincs-primitives.md)

# Description
Migrate the keychain implementation from Lamport signature scheme to SPHINCS+ deterministic signatures. This involves updating keychain.h/cpp to use SPHINCS+ primitives, removing KeyNumber and is_valid fields from database schema and methods, implementing single-key architecture, and updating all classes that interact with the keychain system.

# Requirements and DOD

## Requirements
1. **Keychain Class Updates**:
   - Replace all Lamport signature usage in `TrustLineKeychain` class with SPHINCS+ primitives
   - Update `Keystore` class to use SPHINCS+ for payment transactions
   - Remove all KeyNumber parameters from keychain methods
   - Implement single-key architecture (one key pair per trust line)
   - Update all signing methods: `sign()`, `checkSign()`, `saveOutgoingPaymentReceipt()`, `saveIncomingPaymentReceipt()`, `saveFullAudit()`, etc.

2. **Database Schema Simplification**:
   - Remove `number` field from all key-related tables (both SQLite and PostgreSQL)
   - Remove `is_valid` field from all key-related tables (both SQLite and PostgreSQL)
   - Update primary key constraints for simplified schema
   - Modify database handler classes to work with single-key lookup
   - Update indexes for optimized single-key access patterns

3. **Database Handler Updates**:
   - Update `OwnKeysHandler` (SQLite and PostgreSQL) to remove KeyNumber-related methods
   - Update `ContractorKeysHandler` (SQLite and PostgreSQL) to remove KeyNumber-related methods
   - Remove methods: `keyByNumber()`, `getKeyNumberByHash()`, `getPublicKeyHash()` with KeyNumber parameter
   - Simplify key storage and retrieval to single key per trust line
   - Update key invalidation logic to work without is_valid field

4. **Trust Line Transaction Updates**:
   - Remove KeyNumber parameters from all trust line transaction classes
   - Update transaction classes in `src/core/transactions/transactions/trust_lines/` directory
   - Modify audit, receipt, and signing logic to work with single-key architecture
   - Update serialization/deserialization to exclude KeyNumber fields

5. **Payment Transaction Updates**:
   - Remove KeyNumber parameters from payment transaction classes
   - Update transaction classes in `src/core/transactions/transactions/regular/payments/` directory
   - Modify payment signing and verification logic for single-key approach
   - Update all payment-related cryptographic operations

6. **Key Management Simplification**:
   - Remove key set generation logic (no more multiple keys per trust line)
   - Remove key availability counting and exhaustion checks
   - Simplify key rotation and regeneration logic
   - Update audit conditions to work without key counting

## Definition of Done (DOD)
- [ ] All keychain methods updated to use SPHINCS+ instead of Lamport
- [ ] KeyNumber parameter removed from all keychain methods and dependent classes
- [ ] Database schema updated in both SQLite and PostgreSQL implementations
- [ ] All database handler classes updated to work with simplified schema
- [ ] Trust line transaction classes updated to remove KeyNumber usage
- [ ] Payment transaction classes updated to remove KeyNumber usage
- [ ] Single-key architecture fully implemented and functional
- [ ] All key management complexity removed (no key sets, availability counting, etc.)
- [ ] Comprehensive unit tests for `TrustLineKeychain` and `Keystore` classes
- [ ] Integration tests for trust line and payment operations
- [ ] All existing functionality preserved with new cryptographic backend
- [ ] Database migration scripts created (even though no data needs migration)
- [ ] Code follows project coding standards and style guidelines
- [ ] All modified code properly documented

# Implementation Plan

## Phase 1: Database Schema Analysis and Planning (Week 1, Days 1-2)
1. **Schema Analysis**:
   - Analyze current database schema for key-related tables
   - Identify all tables with `number` and `is_valid` fields
   - Map dependencies between tables and keychain operations
   - Plan database handler method modifications

2. **Method Dependency Mapping**:
   - Identify all methods using KeyNumber parameters in keychain classes
   - Map usage of KeyNumber in trust line and payment transactions
   - Plan method signature changes and call site updates

## Phase 2: Database Layer Updates (Week 1, Days 3-5)
1. **Database Handler Interface Updates**:
   - Update `src/core/io/storage/interfaces/` for new handler signatures
   - Remove KeyNumber-related method declarations
   - Update method signatures for single-key operations

2. **SQLite Handler Updates**:
   - Update `OwnKeysHandlerSQLite` class in `src/core/io/storage/sqlite/`
   - Update `ContractorKeysHandlerSQLite` class
   - Remove KeyNumber-related methods
   - Update SQL queries to work without `number` and `is_valid` fields
   - Simplify primary key constraints and indexes

3. **PostgreSQL Handler Updates**:
   - Update `OwnKeysHandlerPostgreSQL` class in `src/core/io/storage/postgresql/`
   - Update `ContractorKeysHandlerPostgreSQL` class
   - Remove KeyNumber-related methods
   - Update SQL queries for simplified schema
   - Update indexes and constraints

## Phase 3: Keychain Core Implementation (Week 2, Days 1-3)
1. **TrustLineKeychain Updates**:
   - Replace Lamport imports with SPHINCS+ imports in `keychain.h`
   - Update `sign()` method to use SPHINCS+ signature generation
   - Update `checkSign()` method to use SPHINCS+ verification
   - Remove KeyNumber parameters from all method signatures
   - Simplify key retrieval logic to single key per trust line

2. **Payment Receipt Methods**:
   - Update `saveOutgoingPaymentReceipt()` to work without KeyNumber
   - Update `saveIncomingPaymentReceipt()` to work without KeyNumber
   - Simplify key usage logic (no key invalidation needed)
   - Update audit record creation to reference single keys

3. **Audit Methods**:
   - Update `saveFullAudit()` to work with single-key approach
   - Update `saveOwnAuditPart()` and `saveContractorAuditPart()`
   - Remove key number references from audit records
   - Simplify audit signature and verification logic

4. **Keystore Updates**:
   - Update `generateAndSaveKeyPairForPaymentTransaction()` in `Keystore` class
   - Update `signPaymentTransaction()` to use SPHINCS+ signatures
   - Simplify key management for payment transactions

## Phase 4: Transaction Layer Updates (Week 2, Days 4-5)
1. **Trust Line Transaction Updates**:
   - Update all transaction classes in `src/core/transactions/transactions/trust_lines/`
   - Remove KeyNumber parameters from constructors and methods
   - Update audit transactions: `AuditSourceTransaction`, `AuditTargetTransaction`
   - Update key sharing transactions: `PublicKeysSharingSourceTransaction`, etc.

2. **Payment Transaction Updates**:
   - Update payment transaction classes in `src/core/transactions/transactions/regular/payments/`
   - Remove KeyNumber usage from `CoordinatorPaymentTransaction`
   - Update `ReceiverPaymentTransaction`, `IntermediateNodePaymentTransaction`
   - Simplify signature and verification logic in all payment classes

## Phase 5: Key Management Simplification (Week 3, Days 1-2)
1. **Remove Key Set Logic**:
   - Remove `generateKeyPairsSet()` method or simplify to single key generation
   - Remove key availability counting methods
   - Remove key exhaustion checking logic
   - Update initial audit conditions to work without key counting

2. **Simplify Key Access**:
   - Remove `allContractorKeysPresent()`, `ownKeysCriticalCount()` methods
   - Simplify `contractorKeysPresent()` and `ownKeysPresent()` methods
   - Update key hash generation methods to work with single keys

## Phase 6: Testing and Validation (Week 3, Days 3-5)
1. **Unit Test Development**:
   - Create comprehensive tests for updated `TrustLineKeychain` class
   - Create tests for updated `Keystore` class
   - Test all keychain methods with SPHINCS+ backend
   - Test database operations with simplified schema

2. **Integration Testing**:
   - Test complete trust line creation and operation workflows
   - Test payment transaction signing and verification
   - Test audit operations with single-key architecture
   - Test database handler integration with new schema

3. **Regression Testing**:
   - Verify all existing functionality works with new implementation
   - Test error handling and edge cases
   - Verify performance meets requirements

# Test Plan

## Test Objectives
Verify that keychain migration to SPHINCS+ maintains all existing functionality while properly implementing single-key architecture and removing unnecessary complexity.

## Test Scope
- TrustLineKeychain and Keystore class functionality
- Database operations with simplified schema
- Trust line and payment transaction operations
- Integration between keychain and transaction layers
- Error handling and edge cases

## Environment & Setup
- Development environment with SPHINCS+ primitives available (from Task 02-1)
- Test databases (SQLite and PostgreSQL) with updated schema
- C++ testing framework for unit and integration tests
- Mock data for trust line and payment scenarios

## Mocking Strategy
- Mock database connections for isolated unit tests
- Use real database instances for integration tests
- Mock network components not related to cryptographic operations
- Use real SPHINCS+ primitives for cryptographic operations

## Key Test Scenarios

### TrustLineKeychain Unit Tests
- **Single Key Generation**: Test key pair generation for trust lines
- **Signing Operations**: Test `sign()` method with SPHINCS+ signatures
- **Signature Verification**: Test `checkSign()` method with SPHINCS+ verification
- **Receipt Operations**: Test payment receipt saving and retrieval
- **Audit Operations**: Test audit record creation and verification
- **Key Retrieval**: Test single-key retrieval methods

### Keystore Unit Tests
- **Payment Key Generation**: Test key pair generation for payment transactions
- **Payment Signing**: Test payment transaction signing with SPHINCS+
- **Key Management**: Test keychain creation and management

### Database Handler Tests
- **Key Storage**: Test key saving and loading with simplified schema
- **Single Key Lookup**: Test key retrieval without KeyNumber
- **Schema Compliance**: Verify database operations work with updated tables
- **Error Handling**: Test behavior with database errors

### Transaction Integration Tests
- **Trust Line Operations**: Test complete trust line creation and management
- **Payment Processing**: Test payment transaction creation and verification
- **Audit Workflows**: Test audit creation and resolution processes
- **Cross-Transaction Integrity**: Test data consistency across operations

### Migration and Compatibility Tests
- **Functionality Preservation**: Verify all existing features work with new implementation
- **Performance Comparison**: Compare performance with previous Lamport implementation
- **Error Scenarios**: Test error handling in various failure conditions
- **Concurrent Operations**: Test thread safety and concurrent access

## Success Criteria
- All keychain unit tests pass with 100% success rate
- Integration tests demonstrate proper functionality across all components
- Database operations work correctly with simplified schema
- All existing functionality preserved with SPHINCS+ backend
- Performance meets or exceeds previous implementation
- No memory leaks or security vulnerabilities introduced

# Verification and Validation

## Architecture integrity
- Verify single-key architecture maintains separation of concerns
- Confirm database layer updates preserve data access patterns
- Validate transaction layer changes maintain existing interfaces
- Ensure cryptographic operations follow established patterns

## Security
- Verify SPHINCS+ integration maintains post-quantum security level
- Confirm secure key storage and handling practices
- Validate deterministic signature behavior in production scenarios
- Test for potential security regressions in key management

## Performance
- Measure key generation and storage performance improvements
- Compare signature generation and verification times
- Assess database operation performance with simplified schema
- Verify memory usage reduction from single-key architecture

## Scalability
- Test keychain operations under concurrent trust line creation
- Verify database performance with single-key lookup patterns
- Test system behavior with large numbers of trust lines
- Validate scaling characteristics of simplified architecture

## Reliability
- Verify robust error handling in all keychain operations
- Test graceful failure modes for cryptographic operations
- Validate data consistency across database operations
- Ensure reliable recovery from error conditions

## Maintainability
- Confirm code simplification improves maintainability
- Verify clear documentation for modified interfaces
- Validate consistent coding patterns across updated classes
- Ensure easier debugging and troubleshooting

## Cost
- Assess reduction in storage requirements from simplified schema
- Evaluate performance improvements from single-key operations
- Monitor memory usage improvements
- Calculate development and maintenance cost benefits

## Compliance
- Verify continued compliance with NIST standards using SPHINCS+
- Confirm database schema changes follow project standards
- Validate transaction processing maintains audit requirements
- Ensure security practices meet project policies

# Restrictions
- Make commits only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not remove Lamport implementation until all SPHINCS+ functionality is verified
- Maintain database backward compatibility during development (use feature flags if needed)
- Preserve all existing public interfaces during migration where possible
- Follow secure coding practices for all cryptographic operations
- Do not modify unrelated transaction or database code outside the scope