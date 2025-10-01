# 02-2 - Keychain Classes Migration to SPHINCS+

# Links
- [PRD-02: SPHINCS+ Cryptography Implementation](../../prd/vtcpd/02_sphincs_plus_cryptography_implementation.md)
- [Task 02-1: OpenSSL Integration and SPHINCS+ Primitives](./02-1-openssl-integration-sphincs-primitives.md)

# Description
Migrate the keychain classes (`TrustLineKeychain` and `Keystore`) from Lamport signature scheme to SPHINCS+ deterministic signatures. This task focuses specifically on updating keychain.h/cpp to use SPHINCS+ primitives, implementing single-key architecture, updating method signatures to remove KeyNumber parameters, and updating database handler interfaces to work with SPHINCS+ types.

# Requirements and DOD

## Requirements
1. **Keychain Class Updates**:
   - Replace all Lamport signature usage in `TrustLineKeychain` class with SPHINCS+ primitives
   - Update `Keystore` class to use SPHINCS+ for payment transactions
   - Replace `#include "lamportscheme.h"` with `#include "sphincsscheme.h"` in keychain.h
   - Update all method signatures to remove KeyNumber parameters
   - Implement single-key architecture (one key pair per trust line)
   - Update all signing methods: `sign()`, `checkSign()`, `saveOutgoingPaymentReceipt()`, `saveIncomingPaymentReceipt()`, `saveFullAudit()`, etc.

2. **Method Signature Updates**:
   - Remove KeyNumber parameters from all keychain method signatures
   - Update return types from `pair<Signature, KeyNumber>` to just `Signature`
   - Simplify key retrieval methods to work with single key per trust line
   - Update method implementations to use SPHINCS+ primitives instead of Lamport

3. **Key Management Logic Simplification**:
   - Remove key set generation logic (no more multiple keys per trust line)
   - Remove key availability counting and exhaustion checks
   - Simplify key rotation and regeneration logic
   - Update audit conditions to work without key counting
   - Remove methods like `generateKeyPairsSet()`, `allContractorKeysPresent()`, `ownKeysCriticalCount()`

4. **Interface Compatibility**:
   - Maintain existing public interfaces where possible
   - Update method calls throughout keychain classes
   - Ensure single-key operations work correctly
   - Preserve `keys_set_sequence_number` related functionality

5. **Database Handler Interface Updates**:
   - Update `OwnKeysHandler` interface to work with SPHINCS+ types
   - Update `ContractorKeysHandler` interface to work with SPHINCS+ types
   - Update `PaymentKeysHandler` interface to work with SPHINCS+ types
   - Modify method signatures in handler interfaces to accept `sphincs::PublicKey::Shared` and `sphincs::PrivateKey*`
   - Update return types to provide SPHINCS+ key types

## Definition of Done (DOD)
- [ ] All keychain methods updated to use SPHINCS+ instead of Lamport
- [ ] KeyNumber parameter removed from all keychain method signatures
- [ ] `#include "lamportscheme.h"` replaced with `#include "sphincsscheme.h"` in keychain.h
- [ ] Single-key architecture implemented in keychain classes
- [ ] All key management complexity removed (no key sets, availability counting, etc.)
- [ ] Method return types updated (no more KeyNumber returns)
- [ ] All signing and verification methods work with SPHINCS+ primitives
- [ ] Key generation simplified to single key per trust line
- [ ] Code follows project coding standards and style guidelines
- [ ] All modified code properly documented
- [ ] Database handler interfaces updated to work with SPHINCS+ types
- [ ] All interface method signatures updated to use SPHINCS+ primitives
- [ ] Keychain classes compile without errors
- [ ] Complete keychain + database handler integration works

# Implementation Plan

## Phase 1: Keychain Header Updates (Days 1-2)
1. **Header File Updates**:
   - Replace `#include "lamportscheme.h"` with `#include "sphincsscheme.h"` in `keychain.h`
   - Update all method signatures to remove KeyNumber parameters
   - Change return types from `pair<Signature, KeyNumber>` to `Signature` where applicable
   - Update class member variable types if needed

2. **Method Signature Planning**:
   - Identify all methods using KeyNumber parameters in keychain classes
   - Plan new method signatures for single-key operations
   - Document signature changes for implementation phase

## Phase 2: TrustLineKeychain Implementation (Days 3-4)
1. **Core Method Updates**:
   - Update `sign()` method to use SPHINCS+ signature generation and return single signature
   - Update `checkSign()` method to use SPHINCS+ verification without KeyNumber
   - Replace all `lamport::` namespace references with `sphincs::`
   - Simplify key retrieval logic to single key per trust line

2. **Receipt and Audit Methods**:
   - Update `saveOutgoingPaymentReceipt()` to work without KeyNumber
   - Update `saveIncomingPaymentReceipt()` to work without KeyNumber
   - Update `saveFullAudit()`, `saveOwnAuditPart()`, `saveContractorAuditPart()` methods
   - Remove key number references from audit record creation

3. **Key Management Simplification**:
   - Remove or simplify `generateKeyPairsSet()` method
   - Remove key availability counting methods: `allContractorKeysPresent()`, `ownKeysCriticalCount()`
   - Simplify `contractorKeysPresent()` and `ownKeysPresent()` methods
   - Update key hash generation methods

## Phase 3: Keystore Implementation (Day 5)
1. **Keystore Updates**:
   - Update `generateAndSaveKeyPairForPaymentTransaction()` to use SPHINCS+ primitives
   - Update `signPaymentTransaction()` to use SPHINCS+ signatures
   - Update return types and method signatures
   - Simplify key management for payment transactions

## Phase 4: Database Handler Interface Updates (Days 5-6)
1. **Handler Interface Updates**:
   - Update `OwnKeysHandler.h` interface to use `sphincs::PublicKey::Shared` and `sphincs::PrivateKey*`
   - Update `ContractorKeysHandler.h` interface to use `sphincs::PublicKey::Shared`
   - Update `PaymentKeysHandler.h` interface to use SPHINCS+ types
   - Update method signatures across all database handler interfaces

2. **Implementation Updates**:
   - Update SQLite implementations to work with SPHINCS+ key storage
   - Update PostgreSQL implementations to work with SPHINCS+ key storage
   - Ensure key serialization/deserialization works with SPHINCS+ types
   - Update database access patterns for single-key architecture

## Phase 5: Compilation and Basic Validation (Days 7-8)
1. **Compilation Fixes**:
   - Resolve any compilation errors from signature changes
   - Update method calls within keychain classes
   - Ensure all SPHINCS+ primitive usage is correct
   - Fix any missing includes or namespace issues

2. **Basic Functionality Verification**:
   - Verify keychain classes instantiate correctly
   - Check that basic signing operations work
   - Validate single-key architecture implementation
   - Document any issues for follow-up tasks

# Test Plan

**Note**: This task does not include test implementation. Tests will be implemented in a separate dedicated testing task.

## Test Objectives
Verify that keychain classes migration to SPHINCS+ works correctly with proper single-key architecture implementation.

## Test Scope
- TrustLineKeychain and Keystore class compilation
- Basic SPHINCS+ primitive usage in keychain methods
- Method signature correctness
- Single-key architecture implementation

## Manual Validation
- Verify classes compile without errors
- Check that SPHINCS+ primitives are properly used
- Validate method signatures match new single-key design
- Ensure basic instantiation works correctly

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