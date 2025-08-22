# SPHINCS+ Compatibility Update (PRD)

## Document Information
- **Project Name**: VTCPD Test Suite SPHINCS+ Compatibility Update
- **Phase/Iteration**: Phase 2, Test Suite Modernization
- **Document Version**: 1.0
- **Date**: 2025-08-07
- **Author(s)**: Mykola Ilashchuk
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **Status**: Active
- **Previous PRD**: [PRD-01: Multi-DB Testing Support](01-multi-db-testing-support.md)
- **Related Documents**: [PRD-02: SPHINCS+ Cryptography Implementation](../vtcpd/02_sphincs_plus_cryptography_implementation.md), [SPHINCS+ Tasks](../../tasks/vtcpd/02/)

## Executive Summary
- **Current project state**: VTCPD has successfully migrated from Lamport to SPHINCS+ cryptography, but the test suite still uses Lamport-era assumptions
- **This iteration's focus**: Update Dockerfile and test methods to support SPHINCS+ single-key architecture and OpenSSL 3.5+ requirements
- **Connection to overall vision**: Ensures test suite compatibility with modern post-quantum cryptography implementation

## Iteration Context
### Previous Iterations Summary
- **Completed Features**: Multi-database testing support (SQLite/PostgreSQL), comprehensive test coverage for trust lines and payments
- **Lessons Learned**: Database-agnostic testing approach provides better coverage and flexibility
- **Technical Debt**: Test methods still assume Lamport multi-key architecture with `is_valid` fields
- **User Feedback**: Need for updated container environment to support SPHINCS+ cryptography

### Current State Analysis
- **What's working well**: Comprehensive test coverage, multi-database support, containerized testing environment
- **Pain points identified**: Tests fail with SPHINCS+ implementation due to outdated key validation assumptions, missing OpenSSL 3.5+ in container
- **Performance metrics**: Current test suite runs successfully with Lamport implementation in development environment

## Problem Statement
### Background
The VTCPD core project has been successfully migrated from Lamport one-time signature scheme to SPHINCS+ reusable signature scheme. This migration introduced fundamental changes to the cryptographic architecture: single reusable keys instead of key sets, removal of `is_valid` fields from database schema, and requirement for OpenSSL 3.5+ library.

### Problem Description
Clearly articulate the problem this project aims to solve. Include:
- **Who is affected**: All developers running tests, CI/CD pipelines, integration testing environments
- **When and where**: During test execution, particularly database key validation and container startup
- **Impact of not solving**: Test suite incompatibility with SPHINCS+ implementation, inability to validate cryptographic operations, broken CI/CD pipeline

### Success Metrics
Define how success will be measured:
- **Primary KPIs**: 
  - All existing tests pass with SPHINCS+ implementation
  - Container builds successfully with OpenSSL 3.5+
  - Key validation methods work with single-key architecture
- **Secondary KPIs**: 
  - No performance regression in test execution time
  - Maintained test coverage for all cryptographic operations
  - Successful validation of both SQLite and PostgreSQL scenarios
- **Target Values**: 
  - 100% test pass rate with SPHINCS+ vtcpd binary
  - Container build time increase <20%
  - All key validation assertions updated to single-key expectations

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- **OpenSSL 3.5+ Container Support**: Add OpenSSL 3.5+ to both Ubuntu and Manjaro container variants
- **Single-Key Validation Logic**: Update CheckValidKeys methods to expect single key per trust line
- **SPHINCS+ Test Compatibility**: Ensure all test expectations align with SPHINCS+ single-key architecture

#### Bug Fixes & Technical Improvements
- **Container Build Compatibility**: Fix container builds to include required OpenSSL 3.5+ libraries
- **Database Query Updates**: Remove `is_valid` field references from test database queries
- **Test Assertion Updates**: Update all test calls to use single-key expectations instead of multi-key counts

#### Modifications to Existing Features
- **CheckValidKeysSQLite Method**: Update to query total key count instead of valid key count
- **CheckValidKeysPostgreSQL Method**: Update to query total key count instead of valid key count  
- **Test Method Calls**: Update all `CheckValidKeys(t, expectedOwn, expectedContractor)` calls to use single key expectations
- **Dockerfile Variants**: Add OpenSSL 3.5+ packages to both Ubuntu and Manjaro runtime environments

### Explicitly Out of Scope
- Changes to VTCPD core cryptographic implementation (already completed)
- Database schema migrations (handled by VTCPD core)
- New test scenarios or coverage expansion
- Performance optimization of existing tests
- Changes to CLI or user interfaces

### Dependencies from Previous Iterations
- Multi-database testing infrastructure from PRD-01
- Containerized testing environment
- Existing test coverage and scenarios
- Database connection and query execution framework

### Future Roadmap Impact
This update ensures the test suite remains current with cryptographic modernization and provides a foundation for future post-quantum security testing.

## User Stories & Requirements

### User Personas
#### Primary User: Test Engineer
- **Role**: QA engineer responsible for running and maintaining test suite
- **Goals**: Execute comprehensive tests to validate VTCPD functionality with SPHINCS+ cryptography
- **Pain Points**: Current tests fail due to outdated cryptographic assumptions
- **Technical Proficiency**: Intermediate

#### Secondary User: VTCPD Developer
- **Role**: Core system developer working on VTCPD features
- **Goals**: Run tests locally to validate changes before commits
- **Pain Points**: Cannot validate cryptographic operations with current test suite
- **Technical Proficiency**: Advanced

### Functional Requirements
#### New Features for This Iteration
1. **OpenSSL 3.5+ Docker Integration**
   - **Description**: Add OpenSSL 3.5+ libraries to container runtime environments
   - **User Story**: As a test engineer, I want containers to include OpenSSL 3.5+ so that SPHINCS+ cryptography works correctly
   - **Rationale**: SPHINCS+ implementation requires OpenSSL 3.5+ for EVP_SIGNATURE-SLH-DSA-SHA2-256s algorithm support
   - **Builds Upon**: Existing multi-stage Dockerfile architecture
   - **Acceptance Criteria**: 
     - Ubuntu container includes OpenSSL 3.5+ development and runtime libraries
     - Manjaro container includes OpenSSL 3.5+ packages
     - Container builds successfully without conflicts
     - SPHINCS+ algorithms are available in container environment
   - **Priority**: High
   - **Dependencies**: OpenSSL 3.5+ package availability

2. **Single-Key Validation Methods**
   - **Description**: Update CheckValidKeys methods to work with SPHINCS+ single-key architecture
   - **User Story**: As a test engineer, I want key validation methods to work with single keys per trust line
   - **Rationale**: SPHINCS+ uses one reusable key per trust line instead of multiple one-time keys
   - **Builds Upon**: Existing database query and validation framework
   - **Acceptance Criteria**: 
     - CheckValidKeysSQLite queries total keys without `is_valid` filter
     - CheckValidKeysPostgreSQL queries total keys without `is_valid` filter
     - Methods validate presence of at least one key per trust line
     - Error messages updated for single-key context
   - **Priority**: High
   - **Dependencies**: Understanding of SPHINCS+ database schema changes

#### Enhancements to Existing Features
1. **Test Expectation Updates**
   - **Current State**: Tests expect multiple keys with complex counting logic (DefaultKeysCount-1, etc.)
   - **Proposed Changes**: 
     - Update all CheckValidKeys calls to expect 1 key for basic scenarios
     - Remove complex key counting arithmetic
     - Simplify test assertions to focus on key presence rather than counts
     - Update test documentation to reflect single-key expectations
   - **Impact Assessment**: Requires updating numerous test files with key validation calls
   - **Migration Strategy**: Systematic update of all test method calls

2. **Container Environment Modernization**
   - **Current State**: Containers include basic cryptographic libraries but not OpenSSL 3.5+
   - **Proposed Changes**: 
     - Add OpenSSL 3.5+ packages to both Ubuntu and Manjaro variants
     - Update package installation commands
     - Verify library linking and availability
     - Maintain backward compatibility with existing functionality
   - **Impact Assessment**: Increased container size, potential build time increase
   - **Migration Strategy**: Incremental package addition with validation

#### Bug Fixes & Technical Improvements
- **Database Query Modernization**: Remove obsolete `is_valid` field references
- **Error Message Clarity**: Update validation error messages for single-key context
- **Test Reliability**: Ensure consistent behavior across SQLite and PostgreSQL variants

### Non-Functional Requirements
#### Performance
- Container build time increase should not exceed 20%
- Test execution time should remain comparable to current performance
- Database query performance maintained or improved with simplified queries

#### Security
- OpenSSL 3.5+ provides enhanced cryptographic security
- Container security maintained with updated libraries
- No exposure of cryptographic keys during testing

#### Scalability
- Support current test suite scale with updated cryptographic backend
- Maintain ability to test multiple database configurations
- Container resource usage should remain reasonable

#### Reliability
- Robust error handling for cryptographic operations
- Consistent behavior across different container environments
- Reliable database connection and query execution

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Test suite with Lamport multi-key validation assumptions
- **Proposed Changes**: Single-key SPHINCS+ validation with OpenSSL 3.5+ container support
- **Backwards Compatibility**: Not required (direct migration to SPHINCS+)
- **Migration Requirements**: Update test expectations and container environment

### Technology Stack Updates
#### New Technologies/Libraries
- **OpenSSL 3.5+**: Core cryptographic library providing SPHINCS+ support
- **Container Packages**: libssl3, openssl development packages
- **Integration approach**: Add to existing multi-stage Dockerfile architecture

#### Version Updates
- **OpenSSL**: Not present → OpenSSL 3.5+
- **Impact**: New dependency, container build updates required

### Integration Requirements
#### New Integrations
- **OpenSSL 3.5+**: Direct integration into container runtime environments

#### Modified Integrations
- **Database validation**: Updated to work with single-key schema
- **Test framework**: Updated expectations for SPHINCS+ architecture

### Data Requirements
#### Data Models
- **Key validation**: Single key per trust line instead of key sets
- **Database queries**: Simplified queries without `is_valid` filtering
- **Test expectations**: Updated to reflect single-key architecture

#### Data Storage
- No changes to data storage (handled by VTCPD core)
- Test validation updated to match new schema

#### Data Migration
- No data migration required (test suite only)
- Test expectations updated to match new architecture

### Dependencies on Other Teams/Projects
- **VTCPD Core Team**: Confirmation of database schema changes
- **Infrastructure Team**: OpenSSL 3.5+ package availability in container registries

### Integration Points with Previous Work
- Maintains existing multi-database testing approach
- Preserves container-based testing environment
- Compatible with existing test scenarios and coverage

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| OpenSSL 3.5+ Package Availability | High | Low | Verify package availability before implementation |
| Container Build Issues | Medium | Medium | Test container builds in isolation first |
| Test Expectation Complexity | Medium | Medium | Systematic review of all test method calls |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Implementation Delays | Medium | Low | Adequate buffer time, parallel development |
| Test Coverage Gaps | High | Low | Comprehensive validation of updated tests |

## Testing Strategy
### Testing Approach for This Iteration
#### New Feature Testing
- Container build validation with OpenSSL 3.5+
- SPHINCS+ algorithm availability verification in containers
- Updated test method functionality validation
- Database query correctness with single-key expectations

#### Regression Testing
- **Scope**: All existing test scenarios must pass with updated implementation
- **Automated vs Manual**: Automated testing for core functionality
- **Critical User Journeys**: Trust line creation, payment processing, key validation

#### Performance Testing
- **Baseline Metrics**: Current container build and test execution times
- **Target Metrics**: Maintained or improved performance
- **Load Testing**: Multi-container test scenarios

### Quality Gates
- All existing functional tests pass with updated implementation
- Container builds successfully with OpenSSL 3.5+
- Performance benchmarks meet requirements
- Code quality standards met

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Major compatibility update
- **Rollout Strategy**: Direct replacement in development environment
- **Rollback Plan**: Git revert capability

### Feature Flags & Gradual Rollout
Not applicable - direct replacement approach

### Database Migrations
- **Schema Impact**: No changes (handled by VTCPD core)
- **Test Updates**: Updated expectations only

### Communication Plan
- **Internal**: Regular updates to development and QA teams
- **Documentation Updates**: Updated test documentation and README files

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**: 
  - All tests passing with SPHINCS+ implementation
  - Successful container builds with OpenSSL 3.5+
  - Single-key validation working correctly
- **Leading Indicators**: Container build success, test method unit tests
- **Baseline Values**: Current test suite performance metrics
- **Target Values**: Full functionality with SPHINCS+ compatibility

### Monitoring Plan
- **New Monitoring**: OpenSSL library availability verification
- **Enhanced Monitoring**: Test execution success rates, container build metrics
- **Validation**: SPHINCS+ operation testing in container environment

### Review Schedule
- **Daily**: Development progress and container build results
- **Weekly**: Test suite execution results and integration progress
- **Post-Implementation**: Full system validation and performance assessment

## Appendices
### Glossary
- **SPHINCS+**: Post-quantum digital signature scheme using hash-based cryptography
- **Single-Key Architecture**: SPHINCS+ approach using one reusable key per trust line
- **OpenSSL 3.5+**: Minimum OpenSSL version required for SPHINCS+ support
- **Trust Line**: Cryptographic relationship between two parties in VTCPD network
- **CheckValidKeys**: Test method for validating cryptographic key presence and validity

### References
- [VTCPD SPHINCS+ Implementation PRD](../vtcpd/02_sphincs_plus_cryptography_implementation.md)
- [SPHINCS+ Implementation Tasks](../../tasks/vtcpd/02/)
- [OpenSSL 3.5+ SPHINCS+ Documentation](https://www.openssl.org/docs/)
- [NIST Post-Quantum Cryptography Standards](https://csrc.nist.gov/projects/post-quantum-cryptography)
- Current VTCPD test suite architecture

### Wireframes/Mockups
N/A - Backend testing infrastructure changes without UI impact

### API Documentation
Updated documentation for:
- CheckValidKeys method signatures
- Container environment variables
- Test execution procedures

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-08-07 | Mykola Ilashchuk | Initial draft for SPHINCS+ compatibility | Phase 2 |

**Related Documents**
- **Master Project Vision**: [VTCPD Test Suite Overview](../../readme.md)
- **Previous Iteration PRD**: [PRD-01: Multi-DB Testing Support](01-multi-db-testing-support.md)
- **Technical Architecture**: [VTCPD Core SPHINCS+ Implementation](../vtcpd/02_sphincs_plus_cryptography_implementation.md)
- **Implementation Tasks**: [SPHINCS+ Implementation Tasks](../../tasks/vtcpd/02/)