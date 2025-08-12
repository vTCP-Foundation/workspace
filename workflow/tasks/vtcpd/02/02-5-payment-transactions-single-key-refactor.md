# 02-5 Payment transactions single-key refactor

## Links
- [PRD](../../prd/vtcpd/02_sphincs_plus_cryptography_implementation.md)

## Description
Refactor payment transaction signing to use a single reusable SPHINCS+ payment key across all transactions. Remove per-transaction key generation and retrieval by `transaction_uuid`.

## Requirements and DOD
- **Schema changes**:
  - `payment_keys`: remove `transaction_uuid`; add numeric auto-increment `id` (unique) with index.
  - `payment_transactions`: add `payment_key_id` FK to `payment_keys(id)`.
- **Interface changes**:
  - Change `getOwnPrivateKey` in `PaymentKeysHandler`: remove `transactionUUID` param; return key with the largest `id`.
  - Replace `deleteKeyByTransactionUUID` with `deleteKeyByID(id)` in interface and implementations.
  - Remove `allTransactionUUIDs` from `PaymentKeysHandler`.
  - Remove `removeOutdatedPaymentsKeysData` from `keychain.h` and all its invocations.
- **Payments code changes** (`src/core/transactions/transactions/regular/payments/`):
  - Remove payment key generation per transaction.
  - During signing, retrieve key by maximum `id` instead of by `transactionUUID`.
- **Startup behavior**: On node startup, ensure at least one payment key exists; if none, generate one.
  - Implement check inside `Keystore::init()`; add a `Keystore` method to check key presence; add a presence-check method to `PaymentKeysHandler` and its implementations.
- **DOD**:
  - SQLite and PostgreSQL schemas updated; handlers compile and pass unit tests.
  - All references to removed methods are eliminated.
  - Build succeeds; unit tests pass.

## Implementation Plan
1. Update DB schemas (SQLite and PostgreSQL):
   - Modify `payment_keys` and `payment_transactions` per requirements.
   - Apply indexes and FKs.
2. Update storage layer:
   - Adjust `PaymentKeysHandler` interface and all SQLite/PostgreSQL implementations.
   - Implement `getOwnPrivateKey()` to return the key with max `id`.
   - Replace `deleteKeyByTransactionUUID` with `deleteKeyByID`.
3. Update crypto/keychain:
   - Remove `removeOutdatedPaymentsKeysData` and its invocations.
4. Update payment transactions code:
   - Remove per-transaction key generation.
   - On signing, fetch key by max `id`.
5. Startup initialization:
   - Add check to generate initial payment key if none exists.
6. Clean-up and compile:
   - Remove all references to deleted methods.
7. Tests and validation:
   - Update/add unit tests for handlers and startup behavior.

## Test Plan
- Unit tests only; no interface/E2E tests. Provide mock data for both SQLite and PostgreSQL; do not use Docker. Build tests in the `build-tests` directory.
- Add tests for:
  - `getOwnPrivateKey()` returns the key with the largest `id` when multiple keys exist.
  - Behavior when no keys exist initially; startup path generates the first key.
  - Signing path in payments uses key fetched by max `id`.
  - `deleteKeyByID` removes the correct key and subsequent max-id selection updates accordingly.
  - No usages of removed methods remain; compilation guards via include/compile checks.
- Build and run tests in `build-tests` directory.

## Verification and Validation
- User review that refactor aligns with PRD and removes per-transaction key semantics.

### Architecture integrity
- Aligns with single-key SPHINCS+ architecture; reduces complexity.

### Security
- Maintains SPHINCS+ security; avoids key exhaustion; startup path prevents missing-key signing.

### Performance
- Expected improvement: avoids frequent key generation; simple max-id lookup.

### Scalability
- Simplified storage and access patterns; reduced DB growth.

### Reliability
- Deterministic signing; startup guard ensures availability.

### Maintainability
- Fewer code paths and simpler interfaces.

### Cost
- No additional infra; reduced storage footprint.

### Compliance
- Follows project testing and workflow policies.

## Restrictions
- Make commits only after a successful demo or passing tests, per project policy.

