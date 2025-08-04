# ADR-004: SPHINCS+ Signature Scheme Adoption

## Status
Accepted

## Date
2025-08-04

## Context
The BTC Federation requires a quantum-resistant signature scheme to ensure long-term security against quantum computing threats. The project previously used Lamport One-Time Signatures due to their proven security based on hash function safety against quantum attacks.

### Current Requirements
- Quantum-resistant cryptographic signatures (proven security model)
- Standardized implementation for interoperability

### Previous Approach: Lamport One-Time Signatures
Lamport signatures were chosen because:
- Proven security based on hash function properties
- Theoretical guarantee of quantum resistance
- Well-understood security model

### Alternative Considerations
Other post-quantum signature schemes include:
- **Lattice-based cryptography**: While standardized, lacks theoretical proofs of security
- **Other hash-based schemes**: Various implementations with different trade-offs

## Decision
We have decided to adopt **SPHINCS+ (SLH-DSA-SHA2-256s)** as our quantum-resistant signature scheme.

### Primary Rationale
1. **Proven Security Foundation**: SPHINCS+ is based on hash functions, providing the same theoretical security guarantees as Lamport signatures
2. **Standardization**: SPHINCS+ is a NIST-standardized post-quantum signature scheme
3. **Non-random Implementation**: Using the deterministic (non-random) version for consistent behavior
4. **OpenSSL Integration**: Available as EVP_SIGNATURE-SLH-DSA-SHA2-256s in OpenSSL 3.7+
5. **256s variant**: Reduced signature size compared to 256f variant


### Technical Specifications
- **Algorithm**: SPHINCS+ with SHA2-256 hash function
- **Variant**: Short signature version (256s) for reduced signature size
- **Mode**: Non-random version for deterministic signatures
- **Implementation**: OpenSSL EVP_SIGNATURE-SLH-DSA-SHA2-256s (requires OpenSSL 3.7+)
- **Hash Function Choice**: SHA2-256 chosen over SHAKE256 for hardware acceleration availability on modern CPUs
- **Signature Size**: Approximately 30KB signature size is acceptable for channel-based operations, as transaction statements are expected to be aggregated into consolidated state representations

## Consequences

### Positive
- **Quantum Resistance**: Maintains security against quantum computing threats
- **Proven Security**: Hash-based security model with theoretical guarantees
- **Standardization**: NIST-approved algorithm ensures interoperability
- **OpenSSL Support**: Leverages well-maintained, audited implementation
- **Deterministic Behavior**: Non-random version provides consistent signatures

### Negative
- **Signature Size**: Larger signatures compared to Lamport signatures
- **OpenSSL Dependency**: Requires OpenSSL 3.7 or later (replaces libsodium)
- **Performance**: Slower signing/verification compared to Lamport signatures
- **Migration Effort**: Transition from Lamport signatures requires implementation changes

### Security Considerations
- **Hash Function Dependency**: Security relies on SHA2-256 resistance to quantum attacks. This is the point that must be tracked
- **Implementation Security**: Depends on OpenSSL's implementation quality
- **Key Management**: Requires secure key generation and storage practices

## Implementation Requirements

### Dependencies
```bash
# Minimum OpenSSL version required
OpenSSL >= 3.7.0
```