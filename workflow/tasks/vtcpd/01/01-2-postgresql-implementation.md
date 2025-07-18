# 01-2 PostgreSQL Implementation

# Links
- [PRD](../../../prd/vtcpd/01-database-provider-abstraction.md)

# Description
Implement PostgreSQL versions of all storage handlers, add accompanying unit tests, and ensure both SQLite and PostgreSQL test suites pass.

# Requirements and DOD
- Implement `*PostgreSQL` classes for all 16 handlers under `src/core/io/storage/postgresql/`.
- Maintain identical behaviour and schema to SQLite counterparts (type mapping BLOB→BYTEA etc.).
- Use static `PGconn*` connection pattern mirroring SQLite.
- Reuse existing interfaces created in Task 1.
- Provide libpq connection helper with fixed DB names and default credentials (updated later by Task 3).
- Extend Google Test suite with PostgreSQL-specific tests using GMock-based libpq mocks.
- Ensure combined test run (SQLite + PostgreSQL) passes in CI within 30 s.
- Build with `BUILD_TESTING=OFF` remains functional.

# Implementation Plan
1. **Directory & CMake**: Add `postgresql/` subdir and update build scripts; link against `libpq` when enabled.
2. **Class Implementation**: Port logic from SQLite, replacing SQL syntax where necessary.
3. **Static Connection**: Create `PGconn* connection()` helper analogous to SQLite implementation.
4. **Mocks**: Implement `MockPostgreSQLAPI.h` with GMock wrappers of libpq functions.
5. **Tests**: Add tests mirroring SQLite scenarios but against PostgreSQL mocks.
6. **CI Matrix**: Ensure tests build on systems with and without libpq by conditionally compiling mocks only.
7. **Documentation**: Update README and PRD sections if needed.

# Test Plan
- Execute full test suite: `ctest --output-on-failure` should run SQLite and PostgreSQL cases.
- Validate handler behaviour equivalence via shared test fixtures.

# Verification and Validation
Success criteria:
- All unit tests pass for both providers.
- Node binary still starts with default SQLite configuration.

## Architecture integrity
Provider abstraction demonstrates polymorphic usage.

## Security
No credential handling yet; fixed defaults.

## Performance
Comparable execution time to SQLite in unit tests.

## Scalability
Static connection approach maintained.

## Reliability
Increased coverage ensures reliability across providers.

## Maintainability
Code organised per-provider; interfaces unchanged.

## Cost
Uses existing open-source dependencies (libpq).

## Compliance
License compliance for libpq maintained. 