# 02-1 - OpenSSL Integration and SPHINCS+ Primitives

# Links
- [PRD-02: SPHINCS+ Cryptography Implementation](../../prd/vtcpd/02_sphincs_plus_cryptography_implementation.md)

# Description
Integrate OpenSSL 3.0.8+ into the VTCPD project and implement SPHINCS+ cryptographic primitives (PrivateKey, PublicKey, and Signature classes) in the crypto namespace. This task establishes the foundation for migrating from Lamport to SPHINCS+ signature scheme using the deterministic variant (EVP_SIGNATURE-SLH-DSA-SHA2-256s).

# Requirements and DOD

## Requirements
1. **OpenSSL 3.0.8+ Integration**:
   - Add OpenSSL 3.0.8+ as a project dependency in CMakeLists.txt
   - Configure build system to link against OpenSSL libraries
   - Verify EVP_SIGNATURE-SLH-DSA-SHA2-256s algorithm availability
   - Ensure no conflicts with existing dependencies

2. **SPHINCS+ PrivateKey Class**:
   - Implement `sphincs::PrivateKey` class in `src/core/crypto/` directory
   - Support key generation using OpenSSL EVP interface
   - Provide secure memory management and key storage
   - Implement key serialization/deserialization methods
   - Include proper destructor for secure memory cleanup

3. **SPHINCS+ PublicKey Class**:
   - Implement `sphincs::PublicKey` class in `src/core/crypto/` directory
   - Support public key derivation from private key
   - Implement key serialization/deserialization methods
   - Provide key comparison and hash generation methods
   - Include keySize() static method for consistent key size reporting

4. **SPHINCS+ Signature Class**:
   - Implement `sphincs::Signature` class in `src/core/crypto/` directory
   - Support deterministic signature generation (same input + key = same signature)
   - Implement signature verification against public keys
   - Provide signature serialization/deserialization methods
   - Include toString() method for logging purposes

5. **Memory Safety and Security**:
   - Use secure memory allocation for private keys
   - Implement proper memory wiping on destruction
   - Thread-safe operations where applicable
   - Follow cryptographic best practices for key handling

## Definition of Done (DOD)
- [ ] OpenSSL 3.0.8+ successfully integrated and building without errors
- [ ] EVP_SIGNATURE-SLH-DSA-SHA2-256s algorithm functional and accessible
- [ ] All three SPHINCS+ primitive classes implemented and documented
- [ ] Comprehensive unit tests written and passing for all classes
- [ ] Deterministic signature test passes (identical input yields identical signature)
- [ ] Memory leak tests pass (no memory leaks in key operations)
- [ ] Code follows project coding standards and style guidelines
- [ ] All new code properly documented with inline comments
- [ ] Build system updated and CMakeLists.txt modifications complete

# Implementation Plan

## Phase 1: OpenSSL Integration (Week 1, Days 1-3)
1. **Build System Updates**:
   - Modify `src/core/crypto/CMakeLists.txt` to include OpenSSL 3.0.8+ dependency
   - Add FindOpenSSL module configuration if needed
   - Update project-level CMakeLists.txt for OpenSSL linking
   - Verify build system can locate and link OpenSSL libraries

2. **Algorithm Verification**:
   - Create simple test program to verify EVP_SIGNATURE-SLH-DSA-SHA2-256s availability
   - Test basic OpenSSL EVP interface functionality
   - Confirm deterministic variant configuration

## Phase 2: SPHINCS+ Primitives Implementation (Week 1 Day 4 - Week 2 Day 3)
1. **File Structure Setup**:
   - Create `src/core/crypto/sphincskeys.h` for key class declarations
   - Create `src/core/crypto/sphincskeys.cpp` for key implementations
   - Create `src/core/crypto/sphincsscheme.h` for signature class declaration
   - Create `src/core/crypto/sphincsscheme.cpp` for signature implementation

2. **PrivateKey Implementation**:
   - Implement key generation using EVP_PKEY_keygen
   - Add secure memory management with proper cleanup
   - Implement serialization methods for key storage
   - Add key derivation methods for public key generation

3. **PublicKey Implementation**:
   - Implement public key extraction from private key
   - Add key comparison and equality operators
   - Implement hash generation for key identification
   - Add serialization methods for storage and transmission

4. **Signature Implementation**:
   - Implement deterministic signing using EVP_DigestSign
   - Add signature verification using EVP_DigestVerify
   - Implement proper error handling for cryptographic operations
   - Add methods for signature serialization and string representation

## Phase 3: Testing and Validation (Week 2, Days 4-5)
1. **Unit Test Development**:
   - Create comprehensive test suite for each primitive class
   - Test key generation, serialization, and deserialization
   - Test signature generation and verification workflows
   - Test error conditions and edge cases

2. **Deterministic Testing**:
   - Create specific test for deterministic signature behavior
   - Verify identical input and key produce identical signatures
   - Test across multiple iterations to ensure consistency

3. **Memory and Security Testing**:
   - Test for memory leaks in key operations
   - Verify secure memory wiping on object destruction
   - Test thread safety of operations

# Test Plan

## Test Objectives
Verify that OpenSSL integration and SPHINCS+ primitives work correctly, securely, and deterministically as required for the cryptographic migration.

## Test Scope
- OpenSSL 3.0.8+ integration and build system
- SPHINCS+ PrivateKey, PublicKey, and Signature classes
- Deterministic signature behavior
- Memory management and security
- Error handling and edge cases

## Environment & Setup
- Development environment with OpenSSL 3.0.8+ available
- C++ testing framework (existing project test infrastructure)
- Memory testing tools (valgrind or equivalent)
- Test build targets in CMake

## Mocking Strategy
- Mock external dependencies outside of OpenSSL
- Use real OpenSSL library for cryptographic operations
- Mock file system operations for key storage tests if needed

## Key Test Scenarios

### Unit Tests for PrivateKey Class
- **Key Generation**: Verify new private keys can be generated successfully
- **Serialization**: Test key can be serialized and deserialized correctly
- **Security**: Verify memory is properly wiped on destruction
- **Error Handling**: Test behavior with invalid inputs or OpenSSL errors

### Unit Tests for PublicKey Class
- **Derivation**: Verify public key can be derived from private key
- **Serialization**: Test public key serialization and deserialization
- **Comparison**: Test key equality and inequality operations
- **Hashing**: Verify consistent hash generation for key identification

### Unit Tests for Signature Class
- **Signature Generation**: Test signature creation from data and private key
- **Verification**: Test signature verification with correct and incorrect public keys
- **Serialization**: Test signature serialization and deserialization
- **Error Handling**: Test invalid signature and key combinations

### Integration Tests
- **End-to-End Workflow**: Test complete sign-verify cycle
- **Cross-Class Integration**: Test interaction between all three primitive classes
- **OpenSSL Integration**: Verify proper use of OpenSSL EVP interface

### Deterministic Behavior Tests
- **Consistency Test**: Sign identical data with same key multiple times, verify identical signatures
- **Reproducibility**: Test signature generation across different program runs
- **Input Variation**: Test that different inputs produce different signatures with same key

### Performance and Memory Tests
- **Memory Leaks**: Verify no memory leaks in key generation and signature operations
- **Performance Baseline**: Measure signature generation and verification times
- **Resource Usage**: Monitor memory usage during cryptographic operations

## Success Criteria
- All unit tests pass with 100% success rate
- Deterministic signature test demonstrates identical outputs for identical inputs
- Memory tests show no leaks or security vulnerabilities
- Integration tests demonstrate proper inter-class functionality
- Performance tests establish baseline metrics for comparison

# Verification and Validation

## Architecture integrity
- Verify SPHINCS+ implementation follows existing crypto namespace patterns
- Confirm integration maintains separation of concerns between crypto primitives
- Validate that OpenSSL integration doesn't compromise existing architecture

## Security
- Verify secure memory handling for private keys
- Confirm proper key generation using cryptographically secure randomness
- Validate deterministic signature implementation follows security best practices
- Test for timing attack vulnerabilities in cryptographic operations

## Performance
- Establish performance baselines for key generation (< 100ms per key pair)
- Measure signature generation and verification times
- Compare memory usage against current Lamport implementation
- Verify no significant performance regressions in cryptographic operations

## Scalability
- Test cryptographic operations under concurrent access
- Verify thread safety of primitive operations
- Test memory usage scaling with multiple key operations

## Reliability
- Verify robust error handling for OpenSSL failures
- Test graceful degradation when cryptographic operations fail
- Validate consistent behavior across different system configurations

## Maintainability
- Confirm code follows project coding standards and conventions
- Verify comprehensive documentation and inline comments
- Validate clear separation between OpenSSL interface and application logic

## Cost
- Assess memory usage impact of new cryptographic primitives
- Evaluate build time impact of OpenSSL integration
- Monitor any additional runtime dependencies introduced

## Compliance
- Verify implementation follows NIST Post-Quantum Cryptography standards
- Confirm adherence to OpenSSL 3.0+ API best practices
- Validate compliance with project security and coding policies

# Restrictions
- Make commits only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify existing Lamport implementation during this task
- Maintain compatibility with existing crypto namespace interfaces where possible
- Follow secure coding practices for all cryptographic operations