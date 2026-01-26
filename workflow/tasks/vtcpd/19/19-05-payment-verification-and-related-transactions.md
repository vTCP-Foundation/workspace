# 19-05 - Payment Verification and Related Transactions Update

# Links
- [PRD-19: Debt Receipt Signature Structure Update](../../prd/vtcpd/19-debt-receipt-signature-structure-update.md)
- [Previous Task 19-01: Contractor Payment Public Key Storage](19-01-contractor-payment-key-storage.md)
- [Previous Task 19-04: Debt Receipt Signature Structure Update](19-04-debt-receipt-signature-structure.md)

# Description
This task implements payment public key verification during payment flow and updates all related transactions that use the debt receipt structure. This includes:

1. **Payment Public Key Verification**: Adding verification that public keys received from the coordinator match the payment public keys stored in local `Contractor` entities
2. **CycleCloserIntermediateNodeTransaction**: Updating to use new receipt structure
3. **CycleCloserInitiatorTransaction**: Updating to use new receipt structure
4. **ConflictResolverContractorTransaction**: Updating to use new receipt structure for verification

**Context from PRD:**
- During payment, nodes receive public keys from the coordinator for path participants
- For neighbors (participants with reservations in this payment), verify that coordinator's provided key matches the stored `paymentPublicKey` in `Contractor`
- If keys don't match, return same error as hash mismatch
- If stored `paymentPublicKey` is NULL for an existing channel, verification fails and payment is rejected

# Requirements and DOD

## Functional Requirements

### 1. Payment Public Key Verification in BaseExchangePaymentTransaction
- Add verification logic to check coordinator-provided keys against stored channel keys
- Verification applies only to neighbors with reservations (incoming or outgoing)
- Location: likely in or near `checkPublicKeysAppropriate()` method
- Error handling:
  - If coordinator's key doesn't match stored `paymentPublicKey`: return same error as hash mismatch
  - If stored `paymentPublicKey` is NULL: reject payment with same error

### 1.1 Coordinator Final Amounts Confirmation Key Verification
- In coordinator flow, verify `FinalAmountsConfigurationResponseMessage` public keys for neighbors with reservations
- If a neighbor's message key doesn't match stored `paymentPublicKey` or is absent: reject payment
- Location: `runFinalAmountsConfigurationConfirmation()` in coordinator transactions

### 2. Update CycleCloserIntermediateNodeTransaction
- Update calls to `getSerializedReceipt()` with correct parameters
- Retrieve `recipientPaymentPublicKey` from appropriate `Contractor`
- Pass `equivalent` parameter
- Ensure receipt creation works with new structure

### 3. Update CycleCloserInitiatorTransaction
- Update calls to `getSerializedReceipt()` with correct parameters
- Retrieve `recipientPaymentPublicKey` from appropriate `Contractor`
- Pass `equivalent` parameter
- Ensure receipt creation works with new structure

### 4. Update ConflictResolverContractorTransaction
- Update receipt verification to work with new structure
- Ensure `getSerializedReceipt()` calls use correct parameters
- Receipt verification must handle new signature structure

## Definition of Done
- [ ] Payment public key verification implemented for neighbor nodes
- [ ] Verification returns same error as hash mismatch on key mismatch
- [ ] Verification fails if stored `paymentPublicKey` is NULL
- [ ] Coordinator verifies final amounts confirmation public keys for neighbors with reservations
- [ ] `CycleCloserIntermediateNodeTransaction` uses new receipt structure
- [ ] `CycleCloserInitiatorTransaction` uses new receipt structure
- [ ] `ConflictResolverContractorTransaction` uses new receipt structure
- [ ] All transactions retrieve `recipientPaymentPublicKey` from `Contractor`
- [ ] Code compiles without errors
- [ ] Existing tests pass (no regression)

# Implementation Plan

## Step 1: Analyze Current Verification Logic
**Files:**
- `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.cpp`

**Actions:**
1. Locate `checkPublicKeysAppropriate()` method
2. Understand current hash-based verification logic
3. Identify where neighbor relationship is determined
4. Find the error return for hash mismatch to reuse

## Step 2: Implement Payment Key Verification
**File:** `BaseExchangePaymentTransaction.cpp`

**Changes:**
Add verification logic (either in `checkPublicKeysAppropriate()` or new method):

```cpp
// For each path participant that is a neighbor (has TrustLine with us)
for (const auto& participant : pathParticipants) {
    if (isNeighbor(participant.contractorID)) {
        auto contractor = mContractorsHandler->getContractor(participant.contractorID);

        // Check if payment public key exists
        auto storedPaymentKey = contractor->paymentPublicKey();
        if (storedPaymentKey == nullptr) {
            warning() << "Neighbor has no stored payment public key";
            return resultPublicKeyHashMismatch();  // Same error as hash mismatch
        }

        // Compare with coordinator-provided key
        auto coordinatorProvidedKey = participant.publicKey;  // From coordinator message
        if (*storedPaymentKey != *coordinatorProvidedKey) {
            warning() << "Payment public key mismatch for neighbor";
            return resultPublicKeyHashMismatch();  // Same error as hash mismatch
        }
    }
}
```

**Note:** Actual implementation depends on:
- How coordinator-provided keys are stored/accessed
- How neighbor relationship is determined
- Existing error return patterns

## Step 2.1: Add Coordinator Final Amounts Key Verification
**Files:**
- `src/core/transactions/transactions/regular/payments/CoordinatorPaymentTransaction.cpp`
- `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

**Actions:**
1. Locate `runFinalAmountsConfigurationConfirmation()` in each coordinator transaction
2. When the sender is a neighbor with reservations, compare `kMessage->publicKey()` to stored `Contractor::paymentPublicKey()`
3. If the key is absent or mismatched, reject the transaction

## Step 3: Update CycleCloserIntermediateNodeTransaction
**Files:**
- `src/core/transactions/transactions/regular/payments/CycleCloserIntermediateNodeTransaction.h`
- `src/core/transactions/transactions/regular/payments/CycleCloserIntermediateNodeTransaction.cpp`

**Actions:**
1. Locate all calls to `getSerializedReceipt()`
2. For each call:
   - Identify the recipient contractor
   - Retrieve their `paymentPublicKey`
   - Update call with new parameters:
     ```cpp
     auto recipientPaymentKey = recipientContractor->paymentPublicKey();
     auto receipt = getSerializedReceipt(
         recipientPaymentKey,
         amount,
         mEquivalent
     );
     ```
3. Handle case where `paymentPublicKey` might be null (shouldn't happen in normal flow)

## Step 4: Update CycleCloserInitiatorTransaction
**Files:**
- `src/core/transactions/transactions/regular/payments/CycleCloserInitiatorTransaction.h`
- `src/core/transactions/transactions/regular/payments/CycleCloserInitiatorTransaction.cpp`

**Actions:**
Apply same pattern as Step 3:
1. Locate all calls to `getSerializedReceipt()`
2. Update each call with new parameters
3. Ensure `recipientPaymentPublicKey` is retrieved from correct `Contractor`

## Step 5: Update ConflictResolverContractorTransaction
**Files:**
- `src/core/transactions/transactions/regular/payments/ConflictResolverContractorTransaction.h`
- `src/core/transactions/transactions/regular/payments/ConflictResolverContractorTransaction.cpp`

**Actions:**
1. Locate calls to `getSerializedReceipt()` used for verification
2. Update calls with new parameters
3. Ensure verification logic works with new receipt structure
4. Verify that signature verification still functions correctly

## Step 6: Verification
1. Compile the project
2. Run all existing payment and cycle closer tests
3. Manual verification of key verification logic

# Test Plan

**Complexity Level:** Moderate

This task's E2E tests will be implemented in Task 19-06. For this task, verification focuses on:

1. **Compilation Verification**
   - Project compiles without errors
   - No new warnings

2. **Regression Testing**
   - All existing payment transaction tests pass
   - All existing cycle closer tests pass
   - All existing conflict resolver tests pass

3. **Manual Verification (Demo)**

   **Key Verification Tests:**
   - Execute payment between nodes with valid matching keys → payment succeeds
   - (If possible to simulate) Coordinator provides mismatched key → payment fails with appropriate error
   - (If possible to simulate) Coordinator receives mismatched/absent key in final amounts confirmation from neighbor → payment fails
   - (If possible to simulate) Neighbor has NULL `paymentPublicKey` → payment fails

   **Cycle Closer Tests:**
   - Execute cycle closing transaction
   - Verify receipts are created with new structure (debug logging)

   **Conflict Resolver Tests:**
   - Trigger conflict resolution scenario
   - Verify receipt verification works with new structure

# Verification and Validation
_To be completed by Architect after implementation_

## Architecture integrity
- [ ] Verification logic follows existing patterns
- [ ] All related transactions consistently use new receipt structure
- [ ] Error handling is consistent across transaction types

## Security
- [ ] Key verification prevents MITM attacks with mismatched keys
- [ ] NULL key handling prevents use of uninitialized channels
- [ ] Error messages don't leak sensitive information

## Performance
- [ ] Key comparison is efficient (direct byte comparison)
- [ ] Contractor lookup is optimized (likely already cached)

## Scalability
- N/A for this task

## Reliability
- [ ] Verification failure results in clean transaction rejection
- [ ] All error paths are properly handled
- [ ] No partial state on verification failure

## Maintainability
- [ ] Verification logic is centralized and reusable
- [ ] Code follows existing transaction patterns
- [ ] Changes are well-documented

## Cost
- N/A for this task

## Compliance
- N/A for this task

# Restrictions
- Commit changes only after successfully executing the demo or after successfully passing the tests
- Do not modify Contractor or ContractorsHandler in this task (completed in Task 19-01)
- Do not modify `getSerializedReceipt()` signature in this task (completed in Task 19-04)
- Do not implement tests in this task (that's Task 19-06)
