# 02-4 - Database Schema Simplification

# Links
- [PRD-02: SPHINCS+ Cryptography Implementation](../../prd/vtcpd/02_sphincs_plus_cryptography_implementation.md)
- [Task 02-2: Keychain Classes Migration to SPHINCS+](./02-2-keychain-classes-migration-to-sphincs.md)

# Description
Simplify the database schema for key-related tables by removing `number` and `is_valid` fields while preserving the `keys_set_sequence_number` field. Update all database handler classes (both SQLite and PostgreSQL) to work with the simplified single-key architecture.

# Requirements and DOD

## Requirements
1. **Database Schema Updates**:
   - Remove `number` field from `own_keys` table (both SQLite and PostgreSQL)
   - Remove `is_valid` field from `own_keys` table (both SQLite and PostgreSQL)
   - Remove `number` field from `contractor_keys` table (both SQLite and PostgreSQL)
   - Remove `is_valid` field from `contractor_keys` table (both SQLite and PostgreSQL)
   - Preserve `keys_set_sequence_number` field in both tables
   - Update primary key constraints for simplified schema
   - Update indexes for optimized single-key access patterns

2. **SQLite Handler Updates**:
   - Update `OwnKeysHandlerSQLite` class in `src/core/io/storage/sqlite/`
   - Update `ContractorKeysHandlerSQLite` class
   - Remove methods: `keyByNumber()`, `getKeyNumberByHash()`, `getPublicKeyHash()` with KeyNumber parameter
   - Update SQL queries to work without `number` and `is_valid` fields
   - Preserve `keys_set_sequence_number` field and related operations
   - Simplify key storage and retrieval to single key per trust line
   - Update key invalidation logic to work without `is_valid` field

3. **PostgreSQL Handler Updates**:
   - Update `OwnKeysHandlerPostgreSQL` class in `src/core/io/storage/postgresql/`
   - Update `ContractorKeysHandlerPostgreSQL` class
   - Remove methods: `keyByNumber()`, `getKeyNumberByHash()`, `getPublicKeyHash()` with KeyNumber parameter
   - Update SQL queries for simplified schema
   - Preserve `keys_set_sequence_number` field and related operations
   - Update indexes and constraints
   - Simplify key storage and retrieval operations

4. **Interface Updates**:
   - Update `src/core/io/storage/interfaces/` for new handler signatures
   - Remove KeyNumber-related method declarations from interfaces
   - Update method signatures for single-key operations
   - Ensure interface compatibility with updated implementations

5. **Transaction Handler Updates**:
   - Update all transaction classes that use KeyNumber parameters
   - Remove KeyNumber usage from trust line transaction classes in `src/core/transactions/transactions/trust_lines/`
   - Remove KeyNumber usage from payment transaction classes in `src/core/transactions/transactions/regular/payments/`
   - Update serialization/deserialization to exclude KeyNumber fields
   - Modify audit, receipt, and signing logic to work with single-key architecture

## Definition of Done (DOD)
- [ ] `number` and `is_valid` fields removed from `own_keys` and `contractor_keys` tables
- [ ] `keys_set_sequence_number` field preserved in both tables
- [ ] Database schema updated in both SQLite and PostgreSQL implementations
- [ ] All database handler classes updated to work with simplified schema
- [ ] KeyNumber-related methods removed from handler classes
- [ ] Interface definitions updated to match new signatures
- [ ] Trust line transaction classes updated to remove KeyNumber usage
- [ ] Payment transaction classes updated to remove KeyNumber usage
- [ ] Primary key constraints and indexes updated appropriately
- [ ] All SQL queries work with simplified schema
- [ ] Code follows project coding standards and style guidelines
- [ ] All modified code properly documented

# Implementation Plan

## Phase 1: Schema Analysis and Planning (Days 1-2)
1. **Current Schema Analysis**:
   - Analyze current database schema for `own_keys` and `contractor_keys` tables
   - Identify all fields to be removed (`number`, `is_valid`)
   - Identify fields to be preserved (`keys_set_sequence_number`)
   - Map dependencies between tables and handler operations
   - Plan database handler method modifications

2. **Method Dependency Mapping**:
   - Identify all methods using KeyNumber parameters in database handlers
   - Map usage of KeyNumber in transaction classes
   - Plan method signature changes and call site updates
   - Document interface changes needed

## Phase 2: Database Handler Interface Updates (Day 3)
1. **Interface Definition Updates**:
   - Update interface files in `src/core/io/storage/interfaces/`
   - Remove KeyNumber-related method declarations
   - Update method signatures for single-key operations
   - Ensure consistency between SQLite and PostgreSQL interfaces

## Phase 3: SQLite Handler Implementation (Days 4-5)
1. **Schema Updates**:
   - Update table creation scripts to remove `number` and `is_valid` fields
   - Preserve `keys_set_sequence_number` field
   - Update primary key constraints and indexes
   - Create migration scripts if needed

2. **Handler Method Updates**:
   - Update `OwnKeysHandlerSQLite` class implementation
   - Update `ContractorKeysHandlerSQLite` class implementation
   - Remove KeyNumber-related methods
   - Update SQL queries to work with simplified schema
   - Simplify key storage and retrieval logic
   - Update key invalidation logic

## Phase 4: PostgreSQL Handler Implementation (Days 6-7)
1. **Schema Updates**:
   - Update table creation scripts to remove `number` and `is_valid` fields
   - Preserve `keys_set_sequence_number` field
   - Update primary key constraints and indexes
   - Create migration scripts if needed

2. **Handler Method Updates**:
   - Update `OwnKeysHandlerPostgreSQL` class implementation
   - Update `ContractorKeysHandlerPostgreSQL` class implementation
   - Remove KeyNumber-related methods
   - Update SQL queries for simplified schema
   - Update indexes and constraints
   - Simplify key storage and retrieval operations

## Phase 5: Transaction Layer Updates (Days 8-9)
1. **Trust Line Transaction Updates**:
   - Update all transaction classes in `src/core/transactions/transactions/trust_lines/`
   - Remove KeyNumber parameters from constructors and methods
   - Update audit transactions: `AuditSourceTransaction`, `AuditTargetTransaction`
   - Update key sharing transactions: `PublicKeysSharingSourceTransaction`, etc.
   - Update serialization/deserialization logic

2. **Payment Transaction Updates**:
   - Update payment transaction classes in `src/core/transactions/transactions/regular/payments/`
   - Remove KeyNumber usage from `CoordinatorPaymentTransaction`
   - Update `ReceiverPaymentTransaction`, `IntermediateNodePaymentTransaction`
   - Simplify signature and verification logic in all payment classes
   - Update transaction processing logic

## Phase 6: Integration and Validation (Day 10)
1. **Integration Testing**:
   - Verify database operations work with simplified schema
   - Test transaction processing with updated handlers
   - Validate key storage and retrieval operations
   - Check that `keys_set_sequence_number` functionality is preserved

2. **Compilation and Basic Validation**:
   - Resolve any compilation errors from signature changes
   - Ensure all database operations work correctly
   - Validate single-key architecture implementation
   - Document any issues for follow-up

# Test Plan

**Note**: This task does not include test implementation. Tests will be implemented in a separate dedicated testing task.

## Test Objectives
Verify that database schema simplification works correctly and all database operations function properly with the new single-key architecture.

## Test Scope
- Database schema changes in both SQLite and PostgreSQL
- Database handler class functionality
- Transaction class integration with simplified schema
- Key storage and retrieval operations
- `keys_set_sequence_number` field preservation

## Manual Validation
- Verify database operations work with simplified schema
- Check that key storage and retrieval function correctly
- Validate transaction processing with updated handlers
- Ensure `keys_set_sequence_number` operations work properly
- Confirm compilation succeeds with all changes

# Verification and Validation

## Architecture integrity
- Verify database layer changes maintain separation of concerns
- Confirm simplified schema supports single-key architecture
- Validate transaction layer integration with database changes
- Ensure data access patterns remain efficient

## Security
- Verify no security regressions from schema changes
- Confirm secure key storage practices are maintained
- Validate that simplified schema doesn't expose sensitive data
- Test for potential SQL injection vulnerabilities

## Performance
- Measure database operation performance with simplified schema
- Compare query performance before and after changes
- Verify index optimization for single-key lookups
- Assess impact on database storage requirements

## Scalability
- Test database operations under load with simplified schema
- Verify scalability improvements from reduced complexity
- Check that single-key architecture scales appropriately
- Validate performance with large numbers of trust lines

## Reliability
- Verify robust error handling in database operations
- Test graceful failure modes for database errors
- Validate data consistency across simplified schema
- Ensure reliable recovery from error conditions

## Maintainability
- Confirm simplified schema improves code maintainability
- Verify clear documentation for database changes
- Validate consistent patterns across database handlers
- Ensure easier debugging and troubleshooting

## Cost
- Assess storage requirement reductions from simplified schema
- Evaluate performance improvements from reduced complexity
- Monitor database operation efficiency gains
- Calculate maintenance cost benefits

## Compliance
- Verify database changes meet project standards
- Confirm transaction processing maintains audit requirements
- Validate data retention policies are preserved
- Ensure security practices meet project policies

# Restrictions
- Make commits only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify database schema until Task 02-2 (Keychain Migration) is fully completed
- Preserve `keys_set_sequence_number` field and all related functionality
- Maintain data consistency during schema changes
- Follow secure coding practices for all database operations
- Do not remove any functionality that depends on `keys_set_sequence_number`