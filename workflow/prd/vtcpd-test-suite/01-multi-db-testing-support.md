# PRD-01: Multi-DB Testing Support

## Document Information
- **Project Name**: vtcpd-test-suite: Multi-DB Testing Support
- **Phase/Iteration**: Phase 1
- **Document Version**: 1.0
- **Date**: 2024-07-25
- **Author(s)**: Development Team
- **Stakeholders**: Core Team, QA Team
- **Status**: Draft
- **Previous PRD**: N/A
- **Related Documents**: [PRD-01: vtcpd Database Provider Abstraction](../../vtcpd/01-database-provider-abstraction.md)

## Executive Summary
This iteration extends the `vtcpd-test-suite` testing framework to enable testing of the `vtcpd` node with two different database providers: **SQLite** and **PostgreSQL**. This will allow verifying the node's correct operation in both configurations, increasing reliability and test coverage following the implementation of the DB provider abstraction in the main `vtcpd` project.

- **Current project state**: The testing framework can only test the `vtcpd` node using SQLite.
- **This iteration's focus**: Add support for PostgreSQL 16, update the infrastructure for dynamic selection of the DB provider via environment variables, and adapt the database state verification methods.
- **Connection to overall vision**: Ensuring full testing of all supported `vtcpd` configurations, which is critical for production-ready deployments.

## Problem Statement
### Background
The `vtcpd` project was extended to support two database providers (SQLite and PostgreSQL), selectable via the node's configuration. However, the current `vtcpd-test-suite` testing framework cannot test the node with PostgreSQL.

### Problem Description
The lack of PostgreSQL testing makes it impossible to:
- Guarantee the correct operation of `vtcpd` in a production-like environment.
- Detect potential errors specific to the SQL dialect or behavior of PostgreSQL.
- Automatically verify functionality that depends on the selected DB provider.

### Success Metrics
- **Primary KPIs**:
  - Ability to run the entire existing test suite against both SQLite and PostgreSQL.
  - The DB provider for testing is selected via a single environment variable `VTCPD_DATABASE_CONFIG`.
  - All existing tests pass successfully for both providers without modifying the tests themselves.
- **Secondary KPIs**:
  - Maintain the readability and maintainability of the testing framework's code.
- **Target Values**:
  - 100% of existing tests execute successfully in both modes (SQLite/PostgreSQL).

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- **PostgreSQL in Docker**: Integration of PostgreSQL 16 into the `Dockerfile` for the testing environment.
- **Dynamic DB Configuration**: Extending the `Dockerfile` and `pkg/testsuite/cluster.go` to pass the database configuration (`VTCPD_DATABASE_CONFIG`) into the container.
- **Adaptation of Verification Methods**: Refactoring methods in `pkg/testsuite/node.go` to work with both databases.

#### Modifications to Existing Features
- `Dockerfile`: will be updated to install and initialize PostgreSQL.
- `pkg/testsuite/cluster.go`: the `RunNode` method will be updated to pass the new environment variable.
- `pkg/testsuite/node.go`: methods that work with the DB will be split into SQLite and PostgreSQL versions and unified by a common wrapper.

### Explicitly Out of Scope
- Creating new functional tests.
- Changing the logic of existing tests.
- Support for other database providers (besides SQLite and PostgreSQL).
- Optimizing the performance of database queries.

## Functional Requirements

1. **Dockerfile Update**
   - **Description**: Modify the `Dockerfile` to install and configure PostgreSQL 16 in both environments: **Ubuntu** and **Manjaro**.
   - **Acceptance Criteria**:
     - In the `Dockerfile`, within the `runtime-ubuntu` stage, add commands to install `postgresql-16` and `postgresql-client-16`.
     - In the `Dockerfile`, within the `runtime-manjaro` stage, add commands to install `postgresql`.
     - An initialization script is created that, upon the container's first start:
       - Creates a `vtcpd_user` user with the password `vtcpd_pass`.
       - Creates a database named `storagedb`.
       - Assigns ownership of `storagedb` to `vtcpd_user`.
     - The `VTCPD_DATABASE_CONFIG` build argument and the corresponding `ENV VTCPD_DATABASE_CONFIG` environment variable are added for **both `final-*` stages (Ubuntu and Manjaro)**.
     - The `docker-entrypoint.sh` **for both `final-*` stages** is extended to use `VTCPD_DATABASE_CONFIG` when generating `conf.json` for `vtcpd`.

2. **Cluster Manager Update**
   - **Description**: Modify the `RunNode` method in the `pkg/testsuite/cluster.go` file to pass the DB configuration into the container.
   - **Acceptance Criteria**:
     - The `RunNode` method reads the `VTCPD_DATABASE_CONFIG` environment variable from the environment where the tests are running.
     - The value of this variable is added to the list of environment variables (`node.Env`) passed to the container upon creation.

3. **Refactoring of DB State Verification Methods**
   - **Description**: In the `pkg/testsuite/node.go` file, refactor the methods that execute DB queries to support both providers.
   - **Acceptance Criteria**:
     - The existing methods are renamed with the `SQLite` suffix:
       - `CheckCurrentAudit` → `CheckCurrentAuditSQLite`
       - `CheckPaymentRecordWithCommandUUID` → `CheckPaymentRecordWithCommandUUIDSQLite`
       - `CheckSettlementLineState` → `CheckSettlementLineStateSQLite`
       - `CheckValidKeys` → `CheckValidKeysSQLite`
       - `CheckSerializedTransaction` → `CheckSerializedTransactionSQLite`
       - `CheckPaymentTransaction` → `CheckPaymentTransactionSQLite`
     - Similar methods with the `PostgreSQL` suffix are created, which perform the same checks but use `psql` for queries to PostgreSQL.
       - Access credentials: host: `127.0.0.1`, port: `5432`, user: `vtcpd_user`, password: `vtcpd_pass`, dbname: `storagedb`.
       - Queries are adapted to the PostgreSQL dialect (e.g., `x'UUID'` is replaced with `E'\\xUUID'`).
     - Wrapper methods with the original names (`CheckCurrentAudit`, etc.) are created.
     - Each wrapper analyzes the `VTCPD_DATABASE_CONFIG` environment variable:
       - If the variable contains the substring `sqlite`, the `*SQLite` version of the method is called.
       - If the variable contains the substring `postgresql`, the `*PostgreSQL` version of the method is called.
       - Otherwise, the test fails with an error.

## Technical Specifications
### Database Configuration
- **Configuration Format**: `VTCPD_DATABASE_CONFIG` will accept values compatible with `vtcpd`, for example:
  - `sqlite3:///path/to/db`
  - `postgresql://vtcpd_user:vtcpd_pass@127.0.0.1:5432/`
- **psql Commands**: To execute queries in the `*PostgreSQL` methods, the command `docker exec <id> sh -c "PGPASSWORD=vtcpd_pass psql -h 127.0.0.1 -U vtcpd_user -d storagedb -t -A -c '<query>'"` will be used.
  - `-t`: tuples only, no headers.
  - `-A`: unaligned output.

## Implementation Plan
### Task Breakdown
1.  **Task 1: Docker & PostgreSQL Integration**
    - Update `Dockerfile` to install PostgreSQL 16.
    - Create a DB initialization script.
    - Add support for the `VTCPD_DATABASE_CONFIG` variable in `Dockerfile` and `docker-entrypoint.sh`.
2.  **Task 2: Environment Variable Propagation**
    - Modify `RunNode` in `pkg/testsuite/cluster.go` to pass `VTCPD_DATABASE_CONFIG` into containers.
3.  **Task 3: Refactor DB Check Methods**
    - Rename existing methods in `pkg/testsuite/node.go` to `*SQLite`.
    - Implement `*PostgreSQL` versions of the methods.
    - Create wrapper methods that perform dispatch based on `VTCPD_DATABASE_CONFIG`.

## Testing Strategy
- **Goal**: To ensure that all existing tests work correctly with both DB providers.
- **Approach**:
  - **Regression Testing (SQLite)**: Run the full test suite with the configuration `VTCPD_DATABASE_CONFIG="sqlite3:///var/db"` to ensure there are no regressions.
  - **Regression Testing (PostgreSQL)**: Run the full test suite with the configuration `VTCPD_DATABASE_CONFIG="postgresql://vtcpd_user:vtcpd_pass@127.0.0.1:5432/"` to test the new functionality.
- **CI/CD**: Ideally, the CI pipeline should be configured to run tests in both modes.

## Risk Management
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| SQL Query Incompatibility | Medium | Medium | Thorough testing of `*PostgreSQL` methods. Use of parameterized queries where possible. |
| Errors during DB initialization in Docker | Medium | Low | Using a reliable initialization script that is idempotent (safe for re-running). |
| Incorrect propagation of environment variables | Low | Low | Add logging of environment variables at node startup for easy diagnosis. | 