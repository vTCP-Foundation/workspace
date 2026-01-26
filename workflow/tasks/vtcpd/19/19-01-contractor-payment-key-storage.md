# 19-01 - Contractor Payment Public Key Storage

# Links
- [PRD-19: Debt Receipt Signature Structure Update](../../prd/vtcpd/19-debt-receipt-signature-structure-update.md)

# Description
This task implements the foundational layer for payment public key storage. It adds a new field to the `Contractor` class for storing the contractor's payment public key (`sphincs::PublicKey::Shared`) and updates the database schema and handlers (both SQLite and PostgreSQL) to persist this new field.

This is the first task in PRD-19 and establishes the data model foundation that all subsequent tasks depend on. The payment public key will be used during debt receipt signing (instead of ContractorID) and for verification during payment flows.

**Context from PRD:**
- ContractorIDs are local identifiers that are not cryptographically verifiable across nodes
- Payment public keys need to be stored alongside channel encryption keys in the `Contractor` entity
- The field should be nullable to handle existing channels (though migration is out of scope)

# Requirements and DOD

## Functional Requirements
1. **Contractor Class Extension**
   - Add `sphincs::PublicKey::Shared mPaymentPublicKey` private field to `Contractor` class
   - Add `sphincs::PublicKey::Shared paymentPublicKey() const` accessor method
   - Add `void setPaymentPublicKey(sphincs::PublicKey::Shared key)` mutator method
   - Update relevant constructors if needed to optionally accept payment public key

2. **ContractorsHandler Interface Update**
   - Add method declaration for updating payment public key: `virtual void updatePaymentPublicKey(Contractor::Shared contractor) = 0`
   - Update `saveContractor()` to handle payment public key (nullable)
   - Update `saveContractorFull()` to handle payment public key
   - Update `allContractors()` to load payment public key from database
   - Ensure any contractor retrieval method (for example `getContractor()` or similar) loads `payment_public_key` when reading from database

3. **SQLite Implementation (ContractorsHandlerSQLite)**
   - Add column `payment_public_key BLOB` to table schema (nullable)
   - Implement `updatePaymentPublicKey()` method
   - Update `saveContractor()` to store payment public key
   - Update `saveContractorFull()` to store payment public key
   - Update `allContractors()` to retrieve payment public key
   - Update contractor lookup by ID (if present) to retrieve payment public key

4. **PostgreSQL Implementation (ContractorsHandlerPostgreSQL)**
   - Add column `payment_public_key BYTEA` to table schema (nullable)
   - Implement `updatePaymentPublicKey()` method
   - Update `saveContractor()` to store payment public key
   - Update `saveContractorFull()` to store payment public key
   - Update `allContractors()` to retrieve payment public key
   - Update contractor lookup by ID (if present) to retrieve payment public key

## Definition of Done
- [ ] `Contractor` class has `mPaymentPublicKey` field with getter and setter
- [ ] `ContractorsHandler.h` interface includes `updatePaymentPublicKey()` method
- [ ] `ContractorsHandlerSQLite` creates table with `payment_public_key` column
- [ ] `ContractorsHandlerSQLite` implements all CRUD operations for payment public key
- [ ] `ContractorsHandlerPostgreSQL` creates table with `payment_public_key` column
- [ ] `ContractorsHandlerPostgreSQL` implements all CRUD operations for payment public key
- [ ] Contractor retrieval by ID (if available) returns `paymentPublicKey`
- [ ] Code compiles without errors
- [ ] Existing tests pass (no regression)

# Implementation Plan

## Step 1: Update Contractor Class
**Files:**
- `src/core/contractors/Contractor.h`
- `src/core/contractors/Contractor.cpp`

**Changes:**
1. Add include for `sphincs::PublicKey` if not present
2. Add private field: `sphincs::PublicKey::Shared mPaymentPublicKey;`
3. Add public methods:
   ```cpp
   sphincs::PublicKey::Shared paymentPublicKey() const;
   void setPaymentPublicKey(sphincs::PublicKey::Shared key);
   ```
4. Implement methods in .cpp file

## Step 2: Update ContractorsHandler Interface
**Files:**
- `src/core/io/storage/interfaces/ContractorsHandler.h`

**Changes:**
1. Add method declaration:
   ```cpp
   virtual void updatePaymentPublicKey(Contractor::Shared contractor) = 0;
   ```

## Step 3: Update SQLite Implementation
**Files:**
- `src/core/io/storage/sqlite/ContractorsHandlerSQLite.h`
- `src/core/io/storage/sqlite/ContractorsHandlerSQLite.cpp`

**Changes:**
1. Update CREATE TABLE statement to add `payment_public_key BLOB` column
2. Update `saveContractor()` to bind payment public key (can be NULL)
3. Update `saveContractorFull()` to bind payment public key
4. Update `allContractors()` query to SELECT payment_public_key
5. Update `allContractors()` to deserialize payment public key and set on Contractor
6. Update contractor lookup by ID (if present) to include payment public key
7. Implement `updatePaymentPublicKey()`:
   ```cpp
   void updatePaymentPublicKey(Contractor::Shared contractor) override;
   ```

**Serialization approach:**
- Use same serialization format as existing SPHINCS+ keys in the codebase
- Handle NULL case (empty blob or NULL column)

## Step 4: Update PostgreSQL Implementation
**Files:**
- `src/core/io/storage/postgresql/ContractorsHandlerPostgreSQL.h`
- `src/core/io/storage/postgresql/ContractorsHandlerPostgreSQL.cpp`

**Changes:**
1. Update CREATE TABLE statement to add `payment_public_key BYTEA` column
2. Update all methods analogously to SQLite implementation
3. Handle binary format for BYTEA column
4. Update contractor lookup by ID (if present) to include payment public key

## Step 5: Verification
1. Compile the project to ensure no errors
2. Run existing unit tests to verify no regression
3. Manually verify table schema includes new column

# Test Plan

**Complexity Level:** Moderate

This task's testing will be covered in Task 19-06 (Testing). For this task, verification focuses on:

1. **Compilation Verification**
   - Project compiles without errors after all changes
   - No new warnings introduced

2. **Regression Testing**
   - All existing `ContractorsHandlerSQLiteTest` tests pass
   - All existing `ContractorsHandlerPostgreSQLIntegrationTest` tests pass

3. **Manual Verification (Demo)**
   - Create a simple test program or debug session that:
     - Creates a Contractor with payment public key set
     - Saves via handler
     - Retrieves via `allContractors()`
     - Verifies payment public key is correctly round-tripped

# Verification and Validation
_To be completed by Architect after implementation_

## Architecture integrity
- [ ] New field follows existing patterns in Contractor class
- [ ] Database schema changes are minimal and additive
- [ ] Handler interface changes are backward compatible (new method only)

## Security
- [ ] Payment public key stored as binary blob (no encoding issues)
- [ ] Key bytes are not logged or printed in debug output
- [ ] No sensitive data exposed in logs

## Performance
- [ ] No performance impact on existing operations
- [ ] Additional column adds minimal overhead

## Scalability
- N/A for this task

## Reliability
- [ ] NULL handling is consistent across SQLite and PostgreSQL
- [ ] Deserialization handles missing/NULL values gracefully

## Maintainability
- [ ] Code follows existing project patterns
- [ ] Methods are clearly documented

## Cost
- N/A for this task

## Compliance
- N/A for this task

# Restrictions
- Commit changes only after successfully executing the demo or after successfully passing the tests
- Do not modify any files outside the scope defined in Implementation Plan
- Do not implement tests in this task (tests are in Task 19-06)
