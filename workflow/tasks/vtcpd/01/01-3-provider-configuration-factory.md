# 01-3 Provider Configuration & Factory Integration

# Links
- [PRD](../../../prd/vtcpd/01-database-provider-abstraction.md)

# Description
Extend node configuration to specify the database provider (SQLite or PostgreSQL) and connection credentials using RFC 3986 URI format. Implement provider factory that instantiates the correct handler implementations based on configuration and passes connection parameters into constructors.

# Requirements and DOD
- Extend `Settings` class to parse `database_config` URI string as defined in PRD.
- Accept `sqlite3:///path/to/dir` and `postgresql://user:pass@host:port/` schemes.
- Parse credentials and store provider type & parameters globally.
- Implement `StorageProviderFactory` that returns properly initialised handler objects (singleton pattern per provider).
- Update Core initialisation to use factory instead of direct `*SQLite` instantiation.
- Propagate parsed credentials to `*PostgreSQL` constructors.
- Provide clear error messages for malformed configuration.
- Update CMake to link libpq only when PostgreSQL provider compiled.
- Ensure all unit tests (SQLite + PostgreSQL) still pass with factory in place.

# Implementation Plan
1. **Settings Update**: Add URI parsing logic; introduce enum `DatabaseProviderType`.
2. **Factory**: Create `StorageProviderFactory` with static method `create*Handler(...)` per interface.
3. **Core Wiring**: Replace previous direct constructions with factory calls.
4. **Tests Adaptation**: Add test fixture that initialises Settings with each provider and verifies correct object types are produced.
5. **CI**: Parameterise tests to run in both provider modes.
6. **Docs**: Update configuration documentation and sample JSON in PRD.

# Test Plan
- Unit: Factory returns correct concrete types given mock Settings.
- Integration (in-process): Start node with each provider URI and verify no exceptions until first DB operation (mocks).
- Regression: Full SQLite & PostgreSQL handler test suites must still pass.

# Verification and Validation
Success = all tests pass; User can start node with either provider by adjusting `database_config` in config file.

## Architecture integrity
Factory introduces centralised provider selection aligning with PRD.

## Security
Credentials handled in-memory only; future encryption TBD.

## Performance
Negligible overhead per instantiation.

## Scalability
Lays groundwork for future connection pooling.

## Reliability
Prevents mismatched provider usage via centralised point.

## Maintainability
Cleaner separation of concerns.

## Cost
No additional third-party dependencies beyond prior tasks.

## Compliance
Follows project policy and URI standards. 