# 01-1 - Docker & PostgreSQL Integration

# Links
- [PRD](../../../prd/vtcpd-test-suite/01-multi-db-testing-support.md)

# Description
Update the project's `Dockerfile` to install PostgreSQL 16, create a database initialization script, and add support for the `VTCPD_DATABASE_CONFIG` environment variable to enable testing with PostgreSQL. This will allow the test suite to run against a node using either SQLite or PostgreSQL.

# Requirements and DOD
- In the `Dockerfile`, within the `runtime-ubuntu` stage, commands are added to install `postgresql-16` and `postgresql-client-16`.
- In the `Dockerfile`, within the `runtime-manjaro` stage, commands are added to install `postgresql`.
- An initialization script is created that, upon the container's first start:
    - Creates a `vtcpd_user` user with the password `vtcpd_pass`.
    - Creates a database named `storagedb`.
    - Assigns ownership of `storagedb` to `vtcpd_user`.
- The `VTCPD_DATABASE_CONFIG` build argument and the corresponding `ENV VTCPD_DATABASE_CONFIG` environment variable are added for both `final-*` stages (Ubuntu and Manjaro).
- The `docker-entrypoint.sh` for both `final-*` stages is extended to use `VTCPD_DATABASE_CONFIG` when generating `conf.json` for `vtcpd`.
- The changes must be applied to both build stages defined in the Dockerfile (Ubuntu and Manjaro).

# Implementation Plan
1.  **Modify Dockerfile**: Edit both `runtime-ubuntu` and `runtime-manjaro` stages to install the respective PostgreSQL server and client packages.
2.  **Create Pre-initialization Stages**: Add `postgres-init-ubuntu` and `postgres-init-manjaro` stages to the Dockerfile that:
    - Perform full PostgreSQL initialization during image build (not at container startup)
    - Run `initdb` to create the database cluster
    - Start PostgreSQL temporarily to create the `vtcpd_user` role and `storagedb` database
    - Create a flag file to indicate initialization is complete
3.  **Update Final Stages**: In both `final-ubuntu` and `final-manjaro` stages:
    - Copy the pre-initialized PostgreSQL data directory from the corresponding init stage
    - Add the `VTCPD_DATABASE_CONFIG` `ARG` and `ENV`
    - Update the entrypoint to start PostgreSQL without initialization
4.  **Optimize Entrypoint**: Modify the main `docker-entrypoint.sh` to:
    - Skip PostgreSQL initialization (already done during build)
    - Use the value of `VTCPD_DATABASE_CONFIG` when generating the `vtcpd` configuration file (`conf.json`)
    - Start the pre-initialized PostgreSQL server directly

# Test Plan
- Build the Docker image successfully.
- Run a container with `VTCPD_DATABASE_CONFIG` set for SQLite and verify that the `vtcpd` node starts correctly.
- Stop the container and run a new one with `VTCPD_DATABASE_CONFIG` set for PostgreSQL.
- Exec into the PostgreSQL container and use `psql` to verify:
    - The `storagedb` database exists.
    - The `vtcpd_user` role exists.
    - The `vtcpd_user` is the owner of `storagedb`.
- Inspect the `conf.json` file inside the container to confirm that it correctly reflects the database configuration passed via the environment variable.

# Verification and Validation
This task is of **Moderate** complexity.

## Architecture integrity
The changes modify the test environment's container setup but do not alter the test suite's architecture. The approach maintains separation of concerns by using a dedicated init script.

## Security
Database credentials are hardcoded (`vtcpd_pass`) as this is for a closed testing environment. This is acceptable.

## Performance
The pre-initialization approach significantly improves container startup performance:
- **Before**: PostgreSQL initialization during container startup took 18-19 seconds
- **After**: Pre-initialized PostgreSQL starts in ~2 seconds (90% improvement)
- Setup cost is moved to image build time, which is done once and cached
- This dramatically reduces test execution time for PostgreSQL-based tests

## Scalability
Not applicable for this task.

## Reliability
The pre-initialization approach ensures the database setup is reliable and repeatable:
- Database initialization is performed once during image build in a controlled environment
- Eliminates timing-related issues that could occur during container startup
- Reduces the chance of initialization failures during test execution
- Provides consistent database state across all container instances

## Maintainability
The `Dockerfile` becomes slightly more complex, but the logic is contained and follows standard practices for initializing a service within a container.

## Cost
No cost impact.

## Compliance
Compliant with project policies.

# Restrictions
- make commits only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task) 