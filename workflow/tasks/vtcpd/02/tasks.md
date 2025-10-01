# PRD-02 Task List: SPHINCS+ Cryptography Implementation

## Task Overview
This PRD involves replacing the Lamport one-time signature scheme with SPHINCS+ post-quantum signature scheme using OpenSSL 3.0.8+.

## Task List

### Task 02-1: OpenSSL Integration and SPHINCS+ Primitives
- **File**: `02-1-openssl-integration-sphincs-primitives.md`
- **Complexity**: Moderate
- **Description**: Integrate OpenSSL 3.0.8+ into the project and implement SPHINCS+ cryptographic primitives (PrivateKey, PublicKey, Signature classes) with comprehensive testing
- **Dependencies**: Build system, CMakeLists.txt
- **Estimated Duration**: 2 weeks

### Task 02-2: Keychain Classes Migration to SPHINCS+
- **File**: `02-2-keychain-classes-migration-to-sphincs.md`
- **Complexity**: Moderate
- **Description**: Migrate keychain.h/cpp classes to use SPHINCS+ scheme instead of Lamport, update method signatures and implementations
- **Dependencies**: Task 02-1 (SPHINCS+ primitives must be completed first)
- **Estimated Duration**: 1 week

### Task 02-3: Remove Lamport Cryptography Code
- **File**: `02-3-remove-lamport-cryptography-code.md`
- **Complexity**: Simple
- **Description**: Remove all Lamport-related code, classes, and dependencies from the project
- **Dependencies**: Task 02-2 (Keychain migration must be completed first)
- **Estimated Duration**: 2-3 days

### Task 02-4: Database Schema Simplification
- **File**: `02-4-database-schema-simplification.md`
- **Complexity**: Moderate
- **Description**: Remove `number` and `is_valid` fields from key-related tables while preserving `keys_set_sequence_number` field
- **Dependencies**: Task 02-2 (Keychain migration must be completed first)
- **Estimated Duration**: 1 week

### Task 02-6: Update DB Tests and Enable All Tests in Build
- **File**: `02-6-update-db-tests-and-enable-all-tests.md`
- **Complexity**: Simple
- **Description**: Adapt SQLite and PostgreSQL tests to SPHINCS+ single-key interfaces and ensure all unit/integration tests are included in the build. Only successful compilation is required in this iteration (test passing not required).
- **Dependencies**: Task 02-4 (schema simplified) and Task 02-5 (payment keys refactor)
- **Estimated Duration**: 1-2 days

### Task 02-7: Audit Model Simplification (Remove key hash fields)
- **File**: `02-7-audit-model-simplification.md`
- **Complexity**: Moderate
- **Description**: Remove `ownKeyHash`, `contractorKeyHash`, `ownKeysSetHash`, and `contractorKeysSetHash` from audit model, interfaces, implementations, and DB schemas; update serialization and remove public key hash helpers in keychain.
- **Dependencies**: Task 02-2 (Keychain migration), Task 02-4 (Schema simplification groundwork)
- **Estimated Duration**: 3-4 days

## Task Dependencies
```
02-1 (OpenSSL + SPHINCS+ Primitives)
  ↓
02-2 (Keychain Classes Migration)
  ↓
02-3 (Remove Lamport Code) & 02-4 (Database Schema Simplification)
```

## Total Estimated Timeline
5-6 weeks for all tasks with dependencies