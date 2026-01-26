# 19-03 - Channel Transactions Payment Key Exchange

# Links
- [PRD-19: Debt Receipt Signature Structure Update](../../prd/vtcpd/19-debt-receipt-signature-structure-update.md)
- [Previous Task 19-01: Contractor Payment Public Key Storage](19-01-contractor-payment-key-storage.md)
- [Previous Task 19-02: Channel Messages Payment Public Key Extension](19-02-channel-messages-payment-key.md)

# Description
This task updates the channel creation transactions (`InitChannelTransaction` and `ConfirmChannelTransaction`) to exchange and persist payment public keys during the channel establishment protocol. This ensures that when a channel is created, both parties have each other's payment public keys stored locally before any payment transactions can occur.

**Context from PRD:**
- Payment public keys must be exchanged during channel creation, not during payment flow
- `InitChannelTransaction` is run by the channel initiator
- `ConfirmChannelTransaction` is run by the channel responder
- Payment public keys are retrieved from `PaymentKeysHandler` interface
- Keys must be stored in the `Contractor` entity and persisted to the database

**Error Handling from PRD:**
- `InitChannelTransaction`: Return `responseThereAreNoKeys()` if own payment key cannot be retrieved
- `ConfirmChannelTransaction`: Log error and terminate without response (silent termination) if own payment key cannot be retrieved

# Requirements and DOD

## Functional Requirements

### 1. InitChannelTransaction Updates
- Add dependency on `PaymentKeysHandler` to retrieve own payment public key
- In the transaction flow:
  1. Retrieve own payment public key from `PaymentKeysHandler::getOwnPublicKey()`
  2. If retrieval fails, return `responseThereAreNoKeys()` error to ResultInterface
  3. Include own payment public key in `InitChannelMessage` sent to counterparty
  4. When receiving `ConfirmChannelMessage`, extract counterparty's payment public key
  5. Store counterparty's payment public key in the `Contractor` object
  6. Persist payment public key to database via `ContractorsHandler`

### 2. ConfirmChannelTransaction Updates
- Add dependency on `PaymentKeysHandler` to retrieve own payment public key
- In the transaction flow:
  1. When receiving `InitChannelMessage`, extract initiator's payment public key
  2. Retrieve own payment public key from `PaymentKeysHandler::getOwnPublicKey()`
  3. If retrieval fails, log error and terminate transaction without sending response (silent termination)
  4. Store initiator's payment public key in the `Contractor` object being created
  5. Persist payment public key to database via `ContractorsHandler`
  6. Include own payment public key in `ConfirmChannelMessage` sent back to initiator

### 3. PaymentKeysHandler Integration
- Use existing `PaymentKeysHandler` interface method `getOwnPublicKey()`
- Access via existing `IOTransaction` mechanism

## Definition of Done
- [ ] `InitChannelTransaction` retrieves own payment public key from `PaymentKeysHandler`
- [ ] `InitChannelTransaction` returns `responseThereAreNoKeys()` if key retrieval fails
- [ ] `InitChannelTransaction` includes payment public key in `InitChannelMessage`
- [ ] `InitChannelTransaction` extracts and stores counterparty's key from `ConfirmChannelMessage`
- [ ] `InitChannelTransaction` persists counterparty's payment public key to database
- [ ] `ConfirmChannelTransaction` extracts initiator's key from `InitChannelMessage`
- [ ] `ConfirmChannelTransaction` retrieves own payment public key from `PaymentKeysHandler`
- [ ] `ConfirmChannelTransaction` terminates silently (no response) if key retrieval fails
- [ ] `ConfirmChannelTransaction` stores initiator's payment public key in Contractor
- [ ] `ConfirmChannelTransaction` persists initiator's payment public key to database
- [ ] `ConfirmChannelTransaction` includes payment public key in `ConfirmChannelMessage`
- [ ] Code compiles without errors
- [ ] Existing tests pass (no regression)

# Implementation Plan

## Step 1: Analyze Current Transaction Flow
**Files to Study:**
- `src/core/transactions/transactions/trust_line_channel/InitChannelTransaction.h/.cpp`
- `src/core/transactions/transactions/trust_line_channel/ConfirmChannelTransaction.h/.cpp`
- `src/core/io/storage/interfaces/PaymentKeysHandler.h`

**Actions:**
1. Understand current transaction state machine and stages
2. Identify where `InitChannelMessage` is created and sent
3. Identify where `ConfirmChannelMessage` is received/processed
4. Understand how `PaymentKeysHandler` is accessed (likely via `mStorageHandler` or similar)
5. Find `responseThereAreNoKeys()` usage pattern in codebase

## Step 2: Update InitChannelTransaction
**Files:**
- `src/core/transactions/transactions/trust_line_channel/InitChannelTransaction.h`
- `src/core/transactions/transactions/trust_line_channel/InitChannelTransaction.cpp`

**Changes:**

1. **Add PaymentKeysHandler access** (if not already available):
   - Check if transaction has access to storage handlers
   - May need to add member or parameter for `PaymentKeysHandler`

2. **Update message sending stage:**
   ```cpp
   // Before creating InitChannelMessage:
   auto ownPaymentKey = mPaymentKeysHandler->getOwnPublicKey();
   if (ownPaymentKey == nullptr) {
       return responseThereAreNoKeys();
   }

   // Create message with payment key
   auto message = make_shared<InitChannelMessage>(
       ..., // existing params
       ownPaymentKey
   );
   ```

3. **Update ConfirmChannelMessage processing stage:**
   ```cpp
   // After receiving ConfirmChannelMessage:
   auto counterpartyPaymentKey = confirmMessage->paymentPublicKey();
   mContractor->setPaymentPublicKey(counterpartyPaymentKey);

   // Persist to database
   mContractorsHandler->updatePaymentPublicKey(mContractor);
   // Or include in existing save operation
   ```

## Step 3: Update ConfirmChannelTransaction
**Files:**
- `src/core/transactions/transactions/trust_line_channel/ConfirmChannelTransaction.h`
- `src/core/transactions/transactions/trust_line_channel/ConfirmChannelTransaction.cpp`

**Changes:**

1. **Update InitChannelMessage processing:**
   ```cpp
   // Extract initiator's payment key from received message
   auto initiatorPaymentKey = initMessage->paymentPublicKey();
   ```

2. **Retrieve own payment key:**
   ```cpp
   auto ownPaymentKey = mPaymentKeysHandler->getOwnPublicKey();
   if (ownPaymentKey == nullptr) {
       warning() << "Cannot retrieve own payment public key, terminating";
       // Silent termination - do not send ConfirmChannelMessage
       return resultDone();  // or appropriate termination result
   }
   ```

3. **Store initiator's key in Contractor:**
   ```cpp
   // When creating/saving Contractor:
   contractor->setPaymentPublicKey(initiatorPaymentKey);
   // Ensure it's persisted in saveContractor/saveContractorFull call
   ```

4. **Include own key in response:**
   ```cpp
   auto confirmMessage = make_shared<ConfirmChannelMessage>(
       ..., // existing params
       ownPaymentKey
   );
   ```

## Step 4: Verify Handler Access
Ensure both transactions have access to:
- `PaymentKeysHandler` for retrieving own public key
- `ContractorsHandler` for persisting counterparty's public key

If access patterns differ, adapt accordingly to match existing codebase patterns.

## Step 5: Verification
1. Compile the project
2. Run existing tests
3. Manual test: Run two nodes and establish a channel, verify:
   - Channel is created successfully
   - Both nodes have each other's payment public key stored

# Test Plan

**Complexity Level:** Moderate

This task's E2E tests will be implemented in Task 19-06. For this task, verification focuses on:

1. **Compilation Verification**
   - Project compiles without errors
   - No new warnings

2. **Regression Testing**
   - All existing channel creation tests pass

3. **Manual Integration Verification (Demo)**
   - Set up two test nodes
   - Initiate channel creation from Node A to Node B
   - Verify:
     - Channel is created successfully
     - Node A's database has Node B's payment public key in contractors table
     - Node B's database has Node A's payment public key in contractors table
   - Test error case: Simulate missing payment key (if possible), verify appropriate error response

# Verification and Validation
_To be completed by Architect after implementation_

## Architecture integrity
- [ ] Transaction flow changes follow existing patterns
- [ ] Handler access follows established dependency injection patterns
- [ ] Error handling is consistent with other transaction error cases

## Security
- [ ] Payment public keys are validated before storage
- [ ] No sensitive data leaked in error messages or logs
- [ ] Silent termination in ConfirmChannelTransaction doesn't expose information

## Performance
- [ ] Key retrieval is efficient (single call per transaction)
- [ ] Database write is batched with existing Contractor save operation where possible

## Scalability
- N/A for this task

## Reliability
- [ ] Error handling paths are complete
- [ ] Transaction state is consistent after key retrieval failure
- [ ] Database persistence is atomic with channel creation

## Maintainability
- [ ] Code follows existing transaction patterns
- [ ] Error messages are clear and actionable
- [ ] Logic flow is easy to follow

## Cost
- N/A for this task

## Compliance
- N/A for this task

# Restrictions
- Commit changes only after successfully executing the demo or after successfully passing the tests
- Do not modify message classes in this task (completed in Task 19-02)
- Do not modify receipt signature structure in this task (that's Task 19-04)
- Do not implement unit tests in this task (that's Task 19-06)
