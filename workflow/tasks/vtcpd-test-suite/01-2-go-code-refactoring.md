# 01-2 - Go Test Suite Refactoring for Multi-DB Support

# Links
- [PRD](../../../prd/vtcpd-test-suite/01-multi-db-testing-support.md)

# Description
Refactor the Go test suite (`pkg/testsuite/cluster.go` and `pkg/testsuite/node.go`) to support testing against both SQLite and PostgreSQL. This involves propagating the database configuration to the test containers and creating database-specific query methods that are dispatched based on the configuration.

# Requirements and DOD
- The `RunNode` method in `pkg/testsuite/cluster.go` reads the `VTCPD_DATABASE_CONFIG` environment variable from the host and passes its value into the list of environment variables (`node.Env`) for the created container.
- In `pkg/testsuite/node.go`, the methods that perform direct database queries are refactored:
    - Existing methods are renamed with an `SQLite` suffix (e.g., `CheckCurrentAudit` becomes `CheckCurrentAuditSQLite`).
    - New methods are created with a `PostgreSQL` suffix (e.g., `CheckCurrentAuditPostgreSQL`) that perform the equivalent checks using `psql`.
        - These methods must use the correct credentials (`vtcpd_user:vtcpd_pass`) and database (`storagedb`).
        - Queries must be adapted for the PostgreSQL dialect (e.g., `x'UUID'` becomes `E'\\xUUID'`).
    - Wrapper methods with the original names (e.g., `CheckCurrentAudit`) are created.
    - The wrapper methods read the `VTCPD_DATABASE_CONFIG` environment variable and call the appropriate `*SQLite` or `*PostgreSQL` implementation based on whether the variable contains "sqlite" or "postgresql". If the configuration is unrecognized, the test must fail.
- The list of methods to be refactored includes:
    - `CheckCurrentAudit`
    - `CheckPaymentRecordWithCommandUUID`
    - `CheckSettlementLineState`
    - `CheckValidKeys`
    - `CheckSerializedTransaction`
    - `CheckPaymentTransaction`

# Implementation Plan
1.  **Update Cluster Manager**: In `pkg/testsuite/cluster.go`, modify the `RunNode` function to get the `VTCPD_DATABASE_CONFIG` variable from the environment (`os.Getenv`) and append it to the `node.Env` slice passed to the container configuration.
2.  **Refactor SQLite Methods**: In `pkg/testsuite/node.go`, rename the six specified database-checking methods by adding the `SQLite` suffix.
3.  **Implement PostgreSQL Methods**: Create new `*PostgreSQL` versions for each of the six methods. These methods will construct and execute a `docker exec ... psql ...` command with the appropriate query syntax for PostgreSQL.
4.  **Create Dispatcher Wrappers**: Implement the wrapper functions with the original names. Each wrapper will contain a simple `if/else if/else` or `switch` block based on the contents of `os.Getenv("VTCPD_DATABASE_CONFIG")` to delegate the call to the correct underlying implementation.

# Test Plan
- With the `VTCPD_DATABASE_CONFIG` environment variable **unset** or set to an invalid value, run a test that calls one of the wrapper methods and verify that it fails with a clear error message.
- Set `VTCPD_DATABASE_CONFIG` to a valid SQLite configuration (e.g., `sqlite3:///var/db`). Run the entire test suite and verify that all tests pass, confirming no regressions.
- Set `VTCPD_DATABASE_CONFIG` to a valid PostgreSQL configuration (e.g., `postgresql://vtcpd_user:vtcpd_pass@127.0.0.1:5432/`). Run the entire test suite and verify that all tests pass, confirming the new PostgreSQL logic works correctly.

# Verification and Validation
This task is of **Moderate** complexity.

## Architecture integrity
The changes follow the Strategy design pattern, separating the DB checking algorithms into different implementations while maintaining a common interface (the wrapper function). This improves architectural clarity.

## Security
No new security risks are introduced. The database credentials used are for a local, ephemeral test environment.

## Performance
There is a negligible performance impact from reading an environment variable and the conditional logic in the wrapper.

## Scalability
Not applicable for this task.

## Reliability
The change increases reliability by enabling testing against both supported database backends, which helps catch provider-specific bugs.

## Maintainability
Code is more maintainable as logic for each database is now separated. Adding a third provider in the future would be straightforward.

## Cost
No cost impact.

## Compliance
Compliant with project policies.

# Restrictions
- make commits only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task) 