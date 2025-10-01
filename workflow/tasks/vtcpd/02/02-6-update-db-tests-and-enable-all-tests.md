# 02-6 - Update DB tests and enable all tests in build

# Links
- [PRD](../../prd/vtcpd/02_sphincs_plus_cryptography_implementation.md)

# Description
Adapt SQLite and PostgreSQL test suites to the new SPHINCS+ single-key interfaces and schemas, and ensure all tests are included in the build. For this iteration the goal is successful compilation only (not runtime test passing).

# Requirements and DOD
- Update DB tests to remove legacy fields/methods (`number`, `is_valid`, `keyByNumber`, `keyHashByNumber`, `availableKeysCnt`, per-transaction payment keys).
- Use the new interfaces consistently (`getPublicKey`, `getPublicKeyHash`, `hasKey`, `invalidateKey`, `deleteKeysByTrustLineID`/`deleteKeyByID`, `getOwnPrivateKey()`, `hasAnyKeys`, `latestKeyID`).
- Ensure all unit and integration tests (SQLite and PostgreSQL) are included in CMake build (none disabled).
- DOD: All test targets compile successfully in `build-tests`. Runtime pass is out of scope for this task.

# Implementation Plan
1. Review handler interfaces and remove legacy API from tests.
2. Align test schema expectations to single-key model (no `number`/`is_valid`).
3. Update helpers (e.g., `Signature::sign`, key sizes via SPHINCS+ primitives).
4. Ensure CMake includes all test suites; re-enable if excluded.
5. Build in `build-tests` and fix compilation errors until all test targets compile.

# Test Plan
- Build-only verification: `cmake --build .` completes successfully with all test targets.
- Optional: `ctest` may be executed for visibility; passing is not required.

# Verification and Validation
- User confirms all test targets are present and compile successfully.

## Architecture integrity
- Tests reflect single-key SPHINCS+ design; no legacy key-number logic remains.

## Security
- Tests use deterministic SPHINCS+ APIs; no insecure ad-hoc crypto.

## Performance
- Not applicable (build-only scope).

## Scalability
- Not applicable.

## Reliability
- Not applicable.

## Maintainability
- Tests reference only current interfaces; deprecated code removed.

## Cost
- No additional infrastructure; local build only.

## Compliance
- Follows project policy and PRD scope for Iteration 2.

# Restrictions
- make commits only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
