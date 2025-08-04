# 01-1 Add Interfaces and SQLite Refactor

# Links
- [PRD](../../../prd/vtcpd/01-database-provider-abstraction.md)

# Description
Introduce provider interfaces and migrate existing SQLite storage implementation to these interfaces while maintaining current node behaviour. Remove legacy Catch2 tests, integrate Google Test / Google Mock, and ensure full unit-test coverage for SQLite classes.

# Requirements and DOD
- Create abstract interface classes for all 16 storage handlers listed in the PRD.
- Move existing SQLite classes into a `sqlite/` sub-namespace / sub-directory and rename with the `SQLite` suffix.
- Replace all direct usages of the old concrete classes with the new interface types; instantiate `*SQLite` implementations everywhere.
- Keep static connection constants (database directory and names) unchanged in code.
- Delete the legacy Catch2 test suite (`src/tests`, `catch.hpp`, CMake references).
- Add Google Test (GTest) and Google Mock (GMock) via CMake `FetchContent`.
- Implement unit tests for every SQLite handler achieving ≥ 95 % line coverage.
- CI build with `BUILD_TESTING=ON` passes all tests; build with `BUILD_TESTING=OFF` compiles without test dependencies.
- No functional regression: node starts and synchronises as before (manual smoke test).

# Implementation Plan
1. **Interfaces**: Create header files in `src/core/io/storage/interfaces/` defining pure-virtual methods for all handlers.
2. **SQLite Migration**: Move current files into `src/core/io/storage/sqlite/`, adjust namespaces, add `SQLite` suffix.
3. **Compile Fixes**: Update include paths and `CMakeLists.txt` across modules.
4. **Factory Update**: Temporarily instantiate `*SQLite` classes directly (provider factory will come in Task 3).
5. **Remove Catch2**: Delete `src/tests`, `catch.hpp`, and old test CMake entries; verify clean build.
6. **Add GTest/GMock**: Use `FetchContent` to download GoogleTest; create `test/sqlite` target in new `test/CMakeLists.txt`.
7. **Mocks**: Implement sqlite3 API mocks using GMock in `test/mocks/MockSQLiteAPI.h`.
8. **Unit Tests**: Write tests for parameter validation, error handling, and core logic of each handler.
9. **CI**: Ensure `ctest` runs in < 30 s on CI hardware.
10. **Documentation**: Update build instructions and PRD section if needed.

# Test Plan
- Run `cmake -DBUILD_TESTING=ON .. && make && ctest --output-on-failure`.
- Validate coverage report ≥ 95 % for handler source files.
- Turn `BUILD_TESTING=OFF` to ensure production build unaffected.

# Verification and Validation
User acceptance after:
- All tests pass locally and in CI.
- Node binary starts successfully with default SQLite configuration.

## Architecture integrity
Interfaces introduce no breaking changes; overall layered architecture preserved.

## Security
No new security surface; mocks run in-process.

## Performance
No runtime performance impact; only build-time test dependencies added.

## Scalability
Unchanged.

## Reliability
Unit tests increase reliability.

## Maintainability
Clear separation between interface and implementation; easier future provider additions.

## Cost
Open-source dependencies only.

## Compliance
Follows project policy and licensing of GoogleTest (BSD-3). 