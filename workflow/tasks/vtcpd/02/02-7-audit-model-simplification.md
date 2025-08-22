# 02-7 - Audit Model Simplification (Remove key hash fields)

# Links
- [PRD-02: SPHINCS+ Cryptography Implementation](../../prd/vtcpd/02_sphincs_plus_cryptography_implementation.md)

# Description
Remove key-hash related fields from audit model and storage to align with the single-key SPHINCS+ architecture. This eliminates the need to store per-audit `ownKeyHash`, `contractorKeyHash`, `ownKeysSetHash`, and `contractorKeysSetHash`.

# Requirements and DOD
- Remove fields from class `AuditRecord` and its API: `ownKeyHash`, `contractorKeyHash`, `ownKeysSetHash`, `contractorKeysSetHash`.
- Remove related constructors, accessors, mutators, and serialization logic in `AuditRecord` (including `serializeTo*`, `recordSize*`).
- Update `AuditHandler` interface to drop these parameters from `saveFullAudit`, `saveOwnAuditPart`, and `saveContractorAuditPart`.
- Remove `AuditHandler::isContainsKeyHash(...)` from the interface and both implementations; delete all usages and simplify any cleanup logic.
- Update implementations: `AuditHandlerSQLite` and `AuditHandlerPostgreSQL` accordingly, including their method signatures, SQL, and internal logic.
- Update audit table schemas in SQLite and PostgreSQL storage layers to remove corresponding columns.
- Update `BaseTrustLineTransaction` serialization helpers `getOwnSerializedAuditData` and `getContractorSerializedAuditData` to exclude removed fields; change method signatures and update all call sites.
- Update keychain audit conflict helpers: change `checkOwnConflictedSignature` and `checkContractorConflictedSignature` to drop `KeyHash` parameters and work with a single key.
- Remove `checkKeysSetAppropriate(...)` from keychain and all usages.
- Remove methods `ownPublicKeysHash()` and `contractorPublicKeysHash()` from `Keystore/TrustLineKeychain` interfaces, and delete all call sites.
- Ensure all code compiles and tests build in `build-tests`.

# Implementation Plan
1. Update data model (`src/core/io/storage/record/audit/AuditRecord.h`):
   - Remove fields, constructors, getters, setters related to key hashes and keys set hashes.
   - Adjust `serializeTo*` methods and `recordSize*` accordingly.
2. Update storage interfaces (`src/core/io/storage/interfaces/AuditHandler.h`):
   - Remove key-hash parameters from method signatures.
3. Update storage implementations:
   - `src/core/io/storage/sqlite/AuditHandlerSQLite.h/.cpp` and `src/core/io/storage/postgresql/AuditHandlerPostgreSQL.h/.cpp` to match new signatures and logic.
   - Remove key-hash column usage from SQL and binding; update table schemas maintained in code.
4. Update transactions:
   - `src/core/transactions/transactions/trust_lines/base/BaseTrustLineTransaction.*`: modify `getOwnSerializedAuditData` and `getContractorSerializedAuditData` to exclude key-hash and keys-set-hash; update method signatures and all call sites in:
     - `AuditSourceTransaction.cpp`
     - `AuditTargetTransaction.cpp`
     - `SetOutgoingTrustLineTransaction.cpp`
     - `CloseIncomingTrustLineTransaction.cpp`
5. Update keystore interfaces:
   - `src/core/crypto/keychain.h/.cpp`: remove `ownPublicKeysHash()` and `contractorPublicKeysHash()` and delete all call sites across the codebase; update `checkOwnConflictedSignature`, `checkContractorConflictedSignature`, and remove `checkKeysSetAppropriate`.
6. Build and fix compilation errors as needed.
7. Run tests build under `build-tests` to validate successful compilation.

# Test Plan
- Build tests (unit + integration) in `build-tests` directory.
- Ensure all test targets compile successfully after interface and schema changes.
- Verify no remaining references to removed fields or methods via compilation.
  - Update and adjust affected tests and fixtures:
    - `tests/unit/fixtures/TestDataFactory.{h,cpp}`
    - `tests/unit/sqlite/AuditHandlerSQLiteTest.cpp`
    - `tests/storage/integration/postgresql/AuditHandlerPostgreSQLIntegrationTest.cpp`
- Optional: Add/adjust unit tests covering serialization size/format changes for audit records.

# Verification and Validation
- User validates that the audit model no longer contains hash fields and that storage/transaction/keystore code compiles and functions with single-key flow.

## Architecture integrity
- Changes align with PRD’s single-key SPHINCS+ architecture and remove obsolete Lamport-era artifacts.

## Security
- No reduction in security; hashes are unnecessary with single persistent keys.

## Performance
- Minor improvement due to reduced data stored and serialized.

## Scalability
- Reduced storage footprint per audit record.

## Reliability
- Simpler interfaces reduce room for inconsistency between model and storage.

## Maintainability
- Fewer fields and parameters improve readability and reduce coupling.

## Cost
- No additional infra cost. Slightly reduced storage usage.

## Compliance
- Complies with project policy and PRD scope.

# Restrictions
- Make commits only after successful demo/tests as per project policy.
