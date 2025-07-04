# PRD-01 Database Provider Abstraction

## Document Information
- **Project Name**: vtcpd Database Provider Abstraction
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.0
- **Date**: 2024-12-28
- **Author(s)**: Development Team
- **Stakeholders**: Core Team, Operations Team
- **Status**: Draft
- **Previous PRD**: N/A (Initial PRD)
- **Related Documents**: [vtcpd Core Architecture]

## Executive Summary
Brief overview of this iteration, its purpose, and expected outcomes. Include:
- **Current project state**: The vtcpd project currently uses SQLite as the sole database provider for node data persistence
- **This iteration's focus**: Add PostgreSQL support as an alternative database provider with runtime selection capability
- **Connection to overall vision**: Enhances deployment flexibility and scalability options for production environments

## Iteration Context
### Previous Iterations Summary
- **Completed Features**: SQLite-based persistence layer fully implemented
- **Lessons Learned**: Single database dependency limits deployment flexibility 
- **Technical Debt**: Direct SQLite dependencies throughout storage layer
- **User Feedback**: Need for production-grade database support for large-scale deployments

### Current State Analysis
- **What's working well**: Solid storage abstraction within io/storage module
- **Pain points identified**: Hardcoded SQLite dependencies, lack of database choice for production
- **Performance metrics**: SQLite sufficient for development but PostgreSQL needed for production scalability

## Problem Statement
### Background
The vtcpd project currently uses SQLite exclusively for data persistence. While SQLite works well for development and testing, production deployments require more robust database solutions like PostgreSQL for better performance, concurrent access, and operational tooling.

### Problem Description
The current architecture tightly couples all storage handlers to SQLite, making it impossible to:
- Deploy with PostgreSQL for production environments
- Take advantage of PostgreSQL's advanced features (connection pooling, replication, etc.)
- Support different database providers based on deployment requirements
- Scale beyond SQLite's limitations for high-concurrency scenarios

### Success Metrics
- **Primary KPIs**: 
  - Ability to switch between SQLite and PostgreSQL via configuration
  - Zero data loss during provider transitions
  - No performance degradation for existing SQLite deployments
- **Secondary KPIs**: 
  - Code maintainability preserved
  - Test coverage maintained above 90%
- **Target Values**: 
  - Configuration-driven provider selection working in 100% of test cases
  - All existing functionality working with both providers

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- **Database Provider Abstraction**: Create interface layer for all storage handlers
- **PostgreSQL Implementation**: Full PostgreSQL implementation for all storage operations
- **Single-Line Configuration**: Database provider selection at node startup via RFC 3986 URI format
- **Connection Management**: Abstract connection handling for both providers

#### Bug Fixes & Technical Improvements
- Remove static connection dependencies
- Abstract SQL dialect differences
- Improve error handling consistency

#### Modifications to Existing Features
- Refactor all storage handlers to use interface-based approach
- Update configuration system to support database provider selection

### Explicitly Out of Scope
- Data migration tools between SQLite and PostgreSQL
- Database performance optimization beyond functional equivalence
- Support for additional database providers (MySQL, MongoDB, etc.)
- Automatic failover or multiple database support

### Dependencies from Previous Iterations
- Existing SQLite storage implementation
- Current configuration system (Settings class)
- Logging infrastructure

### Future Roadmap Impact
- Enables future support for additional database providers
- Provides foundation for database clustering and replication features
- Supports horizontal scaling initiatives

## User Stories & Requirements

### User Personas
#### Primary User: System Administrator
- **Role**: DevOps/Infrastructure engineer responsible for vtcpd deployment
- **Goals**: Deploy vtcpd with appropriate database for environment (SQLite for dev, PostgreSQL for prod)
- **Pain Points**: Forced to use SQLite even in production environments
- **Technical Proficiency**: Advanced

#### Secondary User: Developer
- **Role**: vtcpd core developer
- **Goals**: Develop and test features against both database providers
- **Pain Points**: Cannot test PostgreSQL integration locally
- **Technical Proficiency**: Advanced

### Functional Requirements
#### New Features for This Iteration

1. **Database Provider Interface Abstraction**
   - **Description**: Create interface classes for all current storage handler classes
   - **User Story**: As a developer, I want database operations to be abstracted so that I can implement different database providers
   - **Rationale**: Enables polymorphic database provider usage
   - **Builds Upon**: Current storage handler architecture
   - **Acceptance Criteria**: 
     - All storage handler classes have corresponding interface definitions
     - Interfaces contain all public methods from current implementations
     - No breaking changes to calling code outside storage module
   - **Priority**: High
   - **Dependencies**: N/A

2. **SQLite Implementation Classes**
   - **Description**: Refactor current storage classes to implement interfaces with SQLite suffix
   - **User Story**: As a developer, I want SQLite functionality preserved so that existing deployments continue working
   - **Rationale**: Maintains backward compatibility while enabling new architecture
   - **Builds Upon**: Current SQLite implementation
   - **Acceptance Criteria**: 
     - All classes moved to sqlite/ subdirectory
     - All classes renamed with SQLite suffix (e.g., AuditHandlerSQLite)
     - Identical functionality to current implementation
     - Table creation logic remains in class constructors
     - All existing tests pass
   - **Priority**: High
   - **Dependencies**: Database Provider Interface Abstraction

3. **PostgreSQL Implementation Classes**
   - **Description**: Create PostgreSQL implementations for all storage interfaces
   - **User Story**: As a system administrator, I want to deploy vtcpd with PostgreSQL so that I can use enterprise-grade database features
   - **Rationale**: Provides production-ready database option
   - **Builds Upon**: Database Provider Interface Abstraction
   - **Acceptance Criteria**: 
     - All interface methods implemented for PostgreSQL
     - Identical table schemas to SQLite versions (with appropriate data type mapping)
     - Table creation handled in class constructors
     - Connection management handles PostgreSQL specifics
     - All classes placed in postgresql/ subdirectory
   - **Priority**: High
   - **Dependencies**: Database Provider Interface Abstraction

4. **Single-Line Database Configuration** 
- **Description**: Extend Settings class to support RFC 3986 URI format for database provider configuration at node startup
   - **User Story**: As a system administrator, I want to specify database provider and credentials in one configuration line so that deployment is simplified
   - **Rationale**: Follows common practice for database connection strings and simplifies deployment
   - **Builds Upon**: Existing Settings infrastructure
   - **Acceptance Criteria**: 
     - Configuration format: `provider|credentials` where provider is "sqlite" or "postgresql"
     - SQLite format: `sqlite|directory_path` (e.g., `sqlite|io`)
     - PostgreSQL format: `postgresql|host|port|user|password` (e.g., `postgresql|localhost|5432|vtcpd_user|password`)
     - Database names remain fixed: "storageDB" and "communicatorStorageDB"
     - Clear error messages for invalid configuration strings
     - Default behavior maintains SQLite compatibility
   - **Priority**: High
   - **Dependencies**: N/A

5. **Provider Factory Implementation**
   - **Description**: Create factory pattern for instantiating appropriate database provider classes based on parsed configuration
   - **User Story**: As a developer, I want provider instantiation abstracted so that calling code doesn't need to know specific provider details
   - **Rationale**: Centralizes provider selection logic and simplifies calling code
   - **Builds Upon**: All provider implementations
   - **Acceptance Criteria**: 
     - Factory creates appropriate provider instances based on parsed configuration
     - Context stores selected provider type for consistent instantiation
     - Calling code in Core.cpp updated to use factory
     - Error handling for unsupported providers
     - Memory management properly handled
   - **Priority**: High
   - **Dependencies**: All implementation classes, Single-Line Database Configuration

#### Database Interface Classes List

**Main Storage Handlers:**
- `StorageHandler` (interface) → `StorageHandlerSQLite`, `StorageHandlerPostgreSQL`
- `CommunicatorStorageHandler` (interface) → `CommunicatorStorageHandlerSQLite`, `CommunicatorStorageHandlerPostgreSQL`

**Transaction Handlers:**
- `IOTransaction` (interface) → `IOTransactionSQLite`, `IOTransactionPostgreSQL`
- `CommunicatorIOTransaction` (interface) → `CommunicatorIOTransactionSQLite`, `CommunicatorIOTransactionPostgreSQL`

**Data Handlers:**
- `TrustLineHandler` (interface) → `TrustLineHandlerSQLite`, `TrustLineHandlerPostgreSQL`
- `TransactionsHandler` (interface) → `TransactionsHandlerSQLite`, `TransactionsHandlerPostgreSQL`
- `HistoryStorage` (interface) → `HistoryStorageSQLite`, `HistoryStoragePostgreSQL`
- `OwnKeysHandler` (interface) → `OwnKeysHandlerSQLite`, `OwnKeysHandlerPostgreSQL`
- `ContractorKeysHandler` (interface) → `ContractorKeysHandlerSQLite`, `ContractorKeysHandlerPostgreSQL`
- `AuditHandler` (interface) → `AuditHandlerSQLite`, `AuditHandlerPostgreSQL`
- `IncomingPaymentReceiptHandler` (interface) → `IncomingPaymentReceiptHandlerSQLite`, `IncomingPaymentReceiptHandlerPostgreSQL`
- `OutgoingPaymentReceiptHandler` (interface) → `OutgoingPaymentReceiptHandlerSQLite`, `OutgoingPaymentReceiptHandlerPostgreSQL`
- `PaymentKeysHandler` (interface) → `PaymentKeysHandlerSQLite`, `PaymentKeysHandlerPostgreSQL`
- `PaymentParticipantsVotesHandler` (interface) → `PaymentParticipantsVotesHandlerSQLite`, `PaymentParticipantsVotesHandlerPostgreSQL`
- `PaymentTransactionsHandler` (interface) → `PaymentTransactionsHandlerSQLite`, `PaymentTransactionsHandlerPostgreSQL`
- `ContractorsHandler` (interface) → `ContractorsHandlerSQLite`, `ContractorsHandlerPostgreSQL`
- `AddressHandler` (interface) → `AddressHandlerSQLite`, `AddressHandlerPostgreSQL`
- `FeaturesHandler` (interface) → `FeaturesHandlerSQLite`, `FeaturesHandlerPostgreSQL`
- `AuditRulesHandler` (interface) → `AuditRulesHandlerSQLite`, `AuditRulesHandlerPostgreSQL`
- `CommunicatorMessagesQueueHandler` (interface) → `CommunicatorMessagesQueueHandlerSQLite`, `CommunicatorMessagesQueueHandlerPostgreSQL`

**Utility Classes:**
- `DatabaseStatementRAII` (interface) → `SQLiteStatementRAII`, `PostgreSQLStatementRAII`

### Non-Functional Requirements
#### Performance
- PostgreSQL operations must perform comparably to SQLite for similar workloads
- No performance degradation for existing SQLite deployments
- Efficient single static connection pattern for PostgreSQL (підготовка до майбутнього пулу з'єднань)
- Performance considerations must be taken into account during implementation
- No performance measurements or benchmarks required after implementation completion

#### Security
- Database credentials secure parsing from configuration string
- SQL injection prevention for both providers

#### Scalability
- PostgreSQL implementation must support concurrent connections
- Підготовка до можливого майбутнього пулу з'єднань PostgreSQL (не входить до поточної ітерації)
- Prepared statement caching

#### Reliability
- Transaction consistency across both providers
- Proper error handling and recovery
- Connection failure resilience

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Direct SQLite dependencies in all storage classes
- **Proposed Changes**: Interface-based abstraction with provider-specific implementations
- **Backwards Compatibility**: Full compatibility maintained for existing SQLite deployments
- **Migration Requirements**: No data migration required; provider selection via configuration only

### Connection Management Strategy
#### Provider-Specific Connection Approaches

**SQLite Implementation (Static Connections):**
- Retains current `static sqlite3* mDBConnection` approach
- Optimized for single-threaded file-based access
- Preserves existing SQLITE_CONFIG_SINGLETHREAD configuration
- Singleton pattern prevents file locking issues
- Minimal changes to current codebase

**PostgreSQL Implementation (Static Connections):**
- Uses `static PGconn* mDBConnection` approach (same pattern as SQLite)
- Single connection per database provider for single-threaded process
- Consistent architecture across both providers
- Simplified connection management and error handling
- Future extensibility preserved for connection pooling when needed

#### Rationale for Unified Static Approach
- **Process Characteristics**: Single-threaded application design - one connection per provider is optimal
- **Consistency**: Both SQLite and PostgreSQL use same static connection pattern
- **Code Simplicity**: Uniform connection management across providers
- **Resource Efficiency**: Minimal connection overhead for single-threaded usage
- **Future Extensibility**: Can be extended to connection pooling when multi-threading is introduced

### Technology Stack Updates
#### New Technologies/Libraries
- `libpq`: PostgreSQL C client library for PostgreSQL implementation
- `Google Test (GTest)`: Unit testing framework
- `Google Mock (GMock)`: Mocking framework for database API unit testing
- Connection pool library (future consideration when multi-threading is introduced)

#### Testing Infrastructure
- `Google Test (GTest)`: Primary unit testing framework
- `Google Mock (GMock)`: For creating mock objects of database APIs
- Enhanced CMake test configuration for storage tests
- All legacy Catch2-based tests have been removed to eliminate dependency on Catch2

#### Version Updates
- No version updates required for existing dependencies
- GMock integration into existing test build system

### Integration Requirements
#### New Integrations
- PostgreSQL server connection and management
- Configuration parsing for RFC 3986 URI format database configuration

#### Modified Integrations
- Settings class extension for database configuration parsing
- Core initialization process updated for provider factory

### Data Requirements
#### Data Models
- Identical table schemas across both providers
- Data type mapping between SQLite and PostgreSQL:
  - BLOB → BYTEA
  - INTEGER → INTEGER
  - STRING → VARCHAR/TEXT
  - Different index creation syntax

#### Data Storage
- SQLite: File-based storage in specified directory
- PostgreSQL: Server-based storage with connection parameters
- Fixed database names: "storageDB" and "communicatorStorageDB"

#### Data Migration
- Out of scope for this iteration
- Future consideration for migration utilities

## Implementation Plan
### This Iteration Timeline
- **Duration**: 6-8 weeks
- **Sprint Breakdown**: 4 sprints of 2 weeks each
- **Key Deliverables**: 
  - Week 2: Interface abstraction complete
  - Week 4: SQLite refactoring complete
  - Week 6: PostgreSQL implementation complete
  - Week 8: Configuration and factory integration complete

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Interface Design Complete | Week 2 | All interface classes defined and validated | N/A | Low |
| SQLite Refactoring Complete | Week 4 | All SQLite classes refactored and working | Interface Design | Medium |
| PostgreSQL Implementation Complete | Week 6 | All PostgreSQL classes implemented | SQLite Refactoring | High |
| Unit Tests Complete | Week 7 | Comprehensive unit test suite with 95% coverage | PostgreSQL Implementation | Medium |
| Integration Complete | Week 8 | Configuration and factory integration working | Unit Tests | Medium |

### Dependencies on Other Teams/Projects
- PostgreSQL server setup for testing environments
- Documentation team for configuration documentation updates

### Integration Points with Previous Work
- Builds directly on existing storage implementation
- Preserves all current functionality and interfaces
- Maintains existing test coverage

### Detailed Task Breakdown

#### Phase 0: Legacy Tests Cleanup (Blocking)
**Rationale:** The project currently contains a Catch2-based test suite located under `src/tests`. Since the new testing stack will use Google Test (GTest) + Google Mock (GMock), all existing Catch2 tests must be removed before the new test infrastructure is introduced to avoid conflicting dependencies.

**Tasks:**
1. Delete the entire `src/tests` directory and all sub-directories.
2. Remove the bundled `catch.hpp` single-header library.
3. Delete or comment out any references to Catch2 or the removed test sources in all `CMakeLists.txt` files (root and submodules).
4. Push a dedicated "Remove legacy Catch2 tests" commit to the branch before starting any GTest development work.

**Exit Criteria:**
– Clean build with `BUILD_TESTING` turned **OFF** succeeds without any Catch2 references.
– Continuous Integration pipeline passes with the legacy tests absent.

#### Phase 1: Interface Abstraction
**Tasks:**
- Create abstract base classes for all 16 handlers
- Define pure virtual methods matching existing signatures
- Establish inheritance hierarchy
- Create interface header files

**Connection Management Notes:**
- Interface classes define virtual methods only, no connection management
- Implementation classes handle provider-specific connections

#### Phase 2: SQLite Implementation Refactoring
**Tasks:**
- Refactor existing classes to inherit from interfaces
- Preserve static connection management for SQLite
- Maintain SQLITE_CONFIG_SINGLETHREAD configuration  
- Update constructor signatures for interface compliance
- Move files to `sqlite/` subdirectory

**Example Implementation:**
```cpp
// AuditHandlerSQLite.h
class AuditHandlerSQLite : public AuditHandler {
private:
    static sqlite3* mDBConnection;  // ← Static preserved for SQLite
    string mTableName;
    Logger* mLogger;
    
public:
    AuditHandlerSQLite(sqlite3* dbConnection, const string& tableName, Logger& logger);
    void saveFullAudit(...) override;
    const AuditRecord::Shared getActualAudit(...) override;
    // ... other virtual methods implementation
    
private:
    static sqlite3* connection(const string& dbName, const string& dir);
};
```

#### Phase 3: PostgreSQL Implementation
**Tasks:**
- Implement all 16 PostgreSQL handler classes
- Use static connection management (same pattern as SQLite)
- Create PostgreSQL-specific table schemas with BYTEA types
- Implement data type mappings (BLOB→BYTEA, etc.)
- Add connection persistence and error handling

**Example Implementation:**
```cpp
// AuditHandlerPostgreSQL.h  
class AuditHandlerPostgreSQL : public AuditHandler {
private:
    static PGconn* mDBConnection;  // ← Static connection for PostgreSQL (same as SQLite)
    string mTableName;
    Logger* mLogger;
    
public:
    AuditHandlerPostgreSQL(PGconn* dbConnection, const string& tableName, Logger& logger);
    void saveFullAudit(...) override;
    const AuditRecord::Shared getActualAudit(...) override;
    // ... other virtual methods implementation
    
private:
    static PGconn* connection(const string& host, int port, const string& user, const string& password, const string& dbName);
    PGresult* executeQuery(const string& query);
};
```

#### Phase 4: Unit Tests Implementation
**Tasks:**
- Set up testing infrastructure with Google Test and GMock integration
- Create mock objects for SQLite and PostgreSQL APIs
- Implement helper classes for common mock expectations
- Create test fixtures and data factories
- Write comprehensive unit tests for all 32 handler classes (16 SQLite + 16 PostgreSQL)

**Test Implementation Coverage:**
- Parameter validation tests for all methods
- Error handling and exception tests
- Business logic tests with mocked database responses
- Edge case testing (null parameters, empty data, boundary conditions)

#### Phase 5: Configuration & Factory Integration
**Tasks:**
- Extend Settings class for RFC 3986 URI format configuration parsing
- Implement provider factory pattern with static connection initialization
- Update initialization points in Core classes
- Add configuration validation and error handling
- Ensure both providers use singleton connection pattern

**Connection Initialization Examples:**
```cpp
// For SQLite (existing pattern)
StorageHandlerSQLite::connection("storageDB", "io");

// For PostgreSQL (new static pattern)
StorageHandlerPostgreSQL::connection("localhost", 5432, "user", "pass", "storageDB");
```

### Resource Requirements
#### Team Structure
- **Technical Lead**: Senior developer with database expertise
- **Developers**: 2 developers with C++ and database experience
- **QA Engineers**: 1 QA engineer for comprehensive testing

#### Budget Estimates
- Development effort: 320-480 hours
- Testing effort: 160-240 hours
- Infrastructure: PostgreSQL testing environments

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| PostgreSQL SQL compatibility issues | High | Medium | Early prototyping with mocked unit tests, real SQL testing in later phases |
| Configuration parsing complexity | Medium | Low | Thorough unit testing of edge cases with mocked data |
| Unified static connection management | Low | Low | Both providers use same proven static pattern, consistent architecture |
| Data type mapping issues | High | Medium | Detailed mapping documentation, comprehensive unit test coverage |
| PostgreSQL connection persistence | Medium | Low | Static connection with reconnection logic, proper error handling |
| Mock complexity for database APIs | Medium | Low | Use proven GMock framework, helper classes for common expectations |
| Limited real database testing | Medium | Medium | Focus unit tests on business logic, defer integration testing to later phases |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Extended development timeline | Medium | Medium | Regular milestone reviews, scope adjustment |
| Breaking changes to existing deployments | High | Low | Comprehensive backwards compatibility testing |

## Testing Strategy
### Testing Approach for This Iteration
#### Unit Testing Only
- **Focus**: Pure unit tests for business logic and parameter validation
- **Scope**: SQLite and PostgreSQL handler implementations only (no interface tests)
- **Approach**: Mock database APIs for both providers to ensure consistency and speed
- **Framework**: Google Test (GTest)

#### Mocking Strategy
- **SQLite**: Mock sqlite3 API calls (sqlite3_prepare_v2, sqlite3_bind_*, sqlite3_step, etc.)
- **PostgreSQL**: Mock libpq API calls (PQexecParams, PQgetvalue, PQclear, etc.)
- **Benefits**: Microsecond execution, full isolation, deterministic results
- **Tools**: Google Mock (GMock) for creating mock objects

#### Test Structure
```
src/tests/storage/
├── fixtures/
│   ├── TestDataFactory.h/.cpp         # Factory for creating test data
│   ├── AuditTestFixtures.h/.cpp        # Audit-specific test data
│   └── TrustLineTestFixtures.h/.cpp    # TrustLine-specific test data
├── helpers/
│   ├── MockExpectationsHelper.h/.cpp   # Helper for setting up mock expectations
│   └── MockDatabaseAPIs.h              # Common mock interfaces
├── mocks/
│   ├── MockSQLiteAPI.h                 # Mock for SQLite3 API
│   └── MockPostgreSQLAPI.h             # Mock for PostgreSQL libpq API
├── sqlite/
│   ├── AuditHandlerSQLiteTest.cpp      # Unit tests for SQLite implementation
│   ├── TrustLineHandlerSQLiteTest.cpp
│   └── [+14 other SQLite handler tests]
├── postgresql/
│   ├── AuditHandlerPostgreSQLTest.cpp  # Unit tests for PostgreSQL implementation
│   ├── TrustLineHandlerPostgreSQLTest.cpp
│   └── [+14 other PostgreSQL handler tests]
└── CMakeLists.txt
```

#### What We Test (Unit Tests Only)
- **Parameter validation**: null checks, range validation, type validation
- **Business logic**: data transformation, calculation logic, state management
- **Error handling**: exception throwing, error recovery, boundary conditions
- **API interaction**: correct sequence of database API calls through mocks
- **Edge cases**: empty data, large datasets, boundary values

#### What We Don't Test (Out of Scope)
- **SQL syntax correctness**: Database responsibility
- **Network connectivity**: Infrastructure concern
- **File system operations**: Integration testing concern
- **Transaction consistency**: Integration testing concern
- **Performance**: Separate performance testing phase

#### Example Unit Test Implementation
```cpp
// src/tests/storage/sqlite/AuditHandlerSQLiteTest.cpp
#include "catch.hpp"
#include "../../../core/io/storage/sqlite/AuditHandlerSQLite.h"
#include "../mocks/MockSQLiteAPI.h"
#include "../helpers/MockExpectationsHelper.h"
#include "../fixtures/AuditTestFixtures.h"

TEST_CASE("AuditHandlerSQLite unit tests", "[audit][sqlite][unit]") {
    // Arrange
    MockSQLiteAPI mockAPI;
    Logger logger;
    
    SECTION("saveFullAudit - successful save") {
        // Arrange
        MockExpectationsHelper::expectSQLiteTableCreation(mockAPI);
        AuditHandlerSQLite handler(&mockAPI, "test_audit", logger);
        
        MockExpectationsHelper::expectSQLiteSuccessfulInsert(mockAPI);
        auto testData = AuditTestFixtures::createValidFullAuditData();
        
        // Act & Assert
        REQUIRE_NOTHROW(handler.saveFullAudit(
            testData.auditNumber,
            testData.trustLineID,
            testData.ownKeyHash,
            testData.ownSignature,
            testData.contractorKeyHash,
            testData.contractorSignature,
            testData.ownKeysSetHash,
            testData.contractorKeysSetHash,
            testData.incomingAmount,
            testData.outgoingAmount,
            testData.balance
        ));
    }
    
    SECTION("saveFullAudit - null parameter validation") {
        // Arrange
        MockExpectationsHelper::expectSQLiteTableCreation(mockAPI);
        AuditHandlerSQLite handler(&mockAPI, "test_audit", logger);
        auto testData = AuditTestFixtures::createValidFullAuditData();
        
        // Act & Assert (no mock expectations - validation should fail before DB call)
        REQUIRE_THROWS_AS(handler.saveFullAudit(
            testData.auditNumber,
            testData.trustLineID,
            nullptr,  // ← Invalid parameter
            testData.ownSignature,
            testData.contractorKeyHash,
            testData.contractorSignature,
            testData.ownKeysSetHash,
            testData.contractorKeysSetHash,
            testData.incomingAmount,
            testData.outgoingAmount,
            testData.balance
        ), std::invalid_argument);
    }
}
```

#### Test Coverage Requirements
- **Unit Tests**: 95% code coverage for all handler implementations
- **Parameter Validation**: 100% coverage of input validation logic
- **Error Handling**: 90% coverage of exception scenarios
- **Business Logic**: 100% coverage of core functionality methods

#### Technology Stack for Testing
- **Framework**: Google Test (GTest)
- **Mocking**: Google Mock (GMock) for database API mocking
- **Build System**: CMake integration with Google Test/GMock infrastructure
- **Execution**: Local development environment, CI/CD pipeline integration

### Quality Gates
- All existing SQLite functionality tests pass
- New unit tests achieve 95% code coverage
- No memory leaks or resource issues in mocked tests
- All parameter validation edge cases covered
- Mock expectations properly validate API call sequences

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Minor release with new functionality
- **Rollout Strategy**: Configuration-based provider selection, default to SQLite
- **Rollback Plan**: Configuration change to revert to SQLite provider

### Feature Flags & Gradual Rollout
- Database provider selection via configuration at node startup only
- No runtime switching between providers during node operation
- Provider selection is permanent for the entire node lifecycle
- Clear error messages for configuration issues

### Database Migrations
- No database migrations required for this iteration
- SQLite databases continue working unchanged
- PostgreSQL databases created fresh on first use

### Communication Plan
- **Internal**: Development team coordination for interface changes
- **External**: User documentation for new configuration options
- **Documentation Updates**: Configuration reference, deployment guides

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**: 
  - Configuration-driven provider selection working in 100% of test scenarios
  - Zero regression in SQLite functionality
  - PostgreSQL provider functionality equivalent to SQLite
  - Unit test coverage: 95% for all handler implementations
- **Leading Indicators**: Unit test pass rates, code coverage metrics, mock test execution speed
- **Baseline Values**: Current SQLite functionality and existing test coverage
- **Target Values**: 
  - 100% functional compatibility between providers
  - 95% unit test code coverage
  - Тривалість виконання всіх юніт-тестів ≤ 30 секунд у CI-середовищі
  - JSON Schema та Doxygen XML артефакти успішно проходять автоматичну перевірку

### Monitoring Plan
- **New Dashboards/Alerts**: Database connection health monitoring
- **Enhanced Monitoring**: Performance metrics for both providers
- **A/B Testing**: Not applicable for this iteration

### Review Schedule
- **Daily**: Development progress and test results
- **Weekly**: Performance benchmark reviews and milestone progress
- **Post-Release Review**: після завершення релізу

## Appendices
### Glossary
- **Provider**: Database implementation (SQLite or PostgreSQL)
- **Interface**: Abstract class defining storage handler contract
- **Handler**: Class responsible for specific data type storage operations
- **Factory**: Pattern for creating appropriate provider instances
- **Single-Line Configuration**: RFC 3986 URI format for database provider configuration with embedded credentials and connection details

### References
- SQLite Documentation: https://sqlite.org/docs.html
- PostgreSQL Documentation: https://www.postgresql.org/docs/
- libpq Documentation: https://www.postgresql.org/docs/current/libpq.html

### Wireframes/Mockups
Not applicable for backend database implementation.

### API Documentation
Interface definitions will be documented in header files with comprehensive documentation comments.

### Configuration Examples
Following RFC 3986 URI standard with adaptations for database providers:

```json
# SQLite configuration (scheme + directory)
"database_config": "sqlite3:///path/to/database/directory"

# PostgreSQL configuration (full URI without database names)
"database_config": "postgresql://vtcpd_user:secure_password@localhost:5432/"
```

**Configuration Format Details:**
- **SQLite**: `sqlite3:///directory_path` (version specified as sqlite3, directory only)
- **PostgreSQL**: `postgresql://user:password@host:port/` (standard URI without database names)
- **Database Names**: Fixed as "storageDB" and "communicatorStorageDB" for both providers
- **Path Handling**: SQLite uses absolute paths, PostgreSQL follows URI specification

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2024-12-28 | Development Team | Initial draft for database provider abstraction | Phase 1 |

**Related Documents**
- **Master Project Vision**: vtcpd Core Architecture
- **Previous Iteration PRD**: N/A (Initial PRD)
- **Technical Architecture**: src/core/io/storage module documentation
- **User Research**: Production deployment requirements analysis 

6. **AI-Agent Oriented Machine-Readable Artefacts**
   - **Description**: Generate and publish machine-readable artefacts (JSON Schema for configuration, Doxygen XML interface descriptors) to facilitate autonomous tooling and AI-agent integration.
   - **User Story**: Як AI-агент, я хочу отримувати структуровані дані про конфігурацію та інтерфейси, щоб автоматично генерувати інтеграційні тести та документацію.
   - **Rationale**: Виконання корпоративної вимоги «Орієнтація на AI-агентів».
   - **Acceptance Criteria**:
     - JSON Schema для параметра `database_config` розміщено у каталозі `docs/` і проходить валідацію CI.
     - При побудові CI генерується Doxygen XML для всіх нових інтерфейсів та реалізацій.
     - Приклади конфігураційних JSON-файлів успішно перевіряються скриптом валідації у CI.
   - **Priority**: Medium
   - **Dependencies**: Single-Line Database Configuration, Provider Factory Implementation 