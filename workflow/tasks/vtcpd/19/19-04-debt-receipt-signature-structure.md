# 19-04 - Debt Receipt Signature Structure Update

# Links
- [PRD-19: Debt Receipt Signature Structure Update](../../prd/vtcpd/19-debt-receipt-signature-structure-update.md)
- [Previous Task 19-01: Contractor Payment Public Key Storage](19-01-contractor-payment-key-storage.md)
- [Previous Task 19-03: Channel Transactions Payment Key Exchange](19-03-channel-transactions-key-exchange.md)

# Description
This task updates the debt receipt signature structure in `BaseExchangePaymentTransaction::getSerializedReceipt()`. The debt receipt is a cryptographic proof that one node owes a specific amount to another. This change replaces local ContractorIDs with cryptographically verifiable payment public keys, removes the audit number (which is unknown at receipt creation time), and adds the equivalent (currency type) to the signed data.

**Context from PRD:**

Current signature structure (to be replaced):
```
ContractorID source                    // Local ID - NOT cryptographically verifiable
ContractorID target                    // Local ID - NOT cryptographically verifiable
BlockNumber mMaximalClaimingBlockNumber
TransactionUUID mTransactionUUID
TrustLineAmount amount
AuditNumber currentAuditNumber         // Unknown at receipt creation time
```

New signature structure:
```
sphincs::PublicKey recipientPaymentPublicKey  // Cryptographically verifiable
BlockNumber mMaximalClaimingBlockNumber
TransactionUUID mTransactionUUID
TrustLineAmount amount
SerializedEquivalent equivalent               // Currency type
```

**Serialization Note from PRD:**
- `recipientPaymentPublicKey` is serialized using the same format and endianness as in payment flow messages `ParticipantVoteMessage` and `ParticipantsVotesMessage`

**Key Design Decision:**
- The sender is identified through the signature itself (each trust line uses unique key pairs), so source identification in the data is redundant
- The recipient is identified by their payment public key in the signed data

# Requirements and DOD

## Functional Requirements

### 1. Update getSerializedReceipt() Method Signature
- Modify method to accept `sphincs::PublicKey::Shared recipientPaymentPublicKey` parameter
- Modify method to accept `SerializedEquivalent equivalent` parameter (or ensure it's accessible)
- Remove `ContractorID source` and `ContractorID target` parameters if they were explicit
- Remove `AuditNumber` parameter

### 2. Update Serialization Logic
- Serialize `recipientPaymentPublicKey` using same format as `ParticipantVoteMessage`
- Serialize `mMaximalClaimingBlockNumber`
- Serialize `mTransactionUUID`
- Serialize `amount`
- Serialize `equivalent`
- Ensure correct byte order (endianness) consistent with existing payment messages

### 3. Update All Callers in BaseExchangePaymentTransaction
- Update all calls to `getSerializedReceipt()` within `BaseExchangePaymentTransaction`
- Retrieve `recipientPaymentPublicKey` from the appropriate `Contractor` via `ContractorsHandler`
- Pass correct parameters to the updated method

### 4. Ensure Equivalent Access
- Verify that `equivalent` is accessible where `getSerializedReceipt()` is called
- May need to pass it as parameter or access from transaction state

## Definition of Done
- [ ] `getSerializedReceipt()` method signature updated with new parameters
- [ ] `source` (ContractorID) removed from serialized data
- [ ] `target` (ContractorID) removed from serialized data
- [ ] `currentAuditNumber` removed from serialized data
- [ ] `recipientPaymentPublicKey` (sphincs::PublicKey) added to serialized data
- [ ] `equivalent` (SerializedEquivalent) added to serialized data
- [ ] Serialization format matches `ParticipantVoteMessage` pattern for public key
- [ ] All callers within `BaseExchangePaymentTransaction` updated
- [ ] `recipientPaymentPublicKey` is retrieved from `Contractor` via `ContractorsHandler`
- [ ] Code compiles without errors
- [ ] Existing tests pass (no regression)

# Implementation Plan

## Step 1: Analyze Current Implementation
**Files:**
- `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.h`
- `src/core/transactions/transactions/regular/payments/base/BaseExchangePaymentTransaction.cpp`

**Actions:**
1. Locate `getSerializedReceipt()` method (around line 1759-1819 based on PRD analysis)
2. Understand current serialization order and format
3. Identify all call sites within the class
4. Understand how `equivalent` is accessed in the transaction

## Step 2: Study Serialization Pattern
**Reference Files:**
- `src/core/network/messages/payments/ParticipantVoteMessage.h/.cpp`

**Actions:**
1. Understand how `sphincs::PublicKey` is serialized
2. Note byte order and size handling
3. Ensure consistency with existing patterns

## Step 3: Update Method Signature
**File:** `BaseExchangePaymentTransaction.h`

**Changes:**
```cpp
// Old (conceptual):
BytesShared getSerializedReceipt(
    ContractorID source,
    ContractorID target,
    const TrustLineAmount &amount,
    AuditNumber currentAuditNumber);

// New:
BytesShared getSerializedReceipt(
    sphincs::PublicKey::Shared recipientPaymentPublicKey,
    const TrustLineAmount &amount,
    SerializedEquivalent equivalent);
```

Note: `mMaximalClaimingBlockNumber` and `mTransactionUUID` are likely member variables, not parameters.

## Step 4: Update Serialization Implementation
**File:** `BaseExchangePaymentTransaction.cpp`

**Changes to getSerializedReceipt():**

```cpp
BytesShared BaseExchangePaymentTransaction::getSerializedReceipt(
    sphincs::PublicKey::Shared recipientPaymentPublicKey,
    const TrustLineAmount &amount,
    SerializedEquivalent equivalent)
{
    // Calculate total size
    size_t dataSize =
        recipientPaymentPublicKey->size() +  // or fixed SPHINCS+ key size
        sizeof(BlockNumber) +                 // mMaximalClaimingBlockNumber
        TransactionUUID::size() +             // mTransactionUUID
        kTrustLineAmountBytesCount +          // amount
        sizeof(SerializedEquivalent);         // equivalent

    BytesShared result = make_shared<vector<byte>>(dataSize);
    size_t offset = 0;

    // Serialize recipientPaymentPublicKey (same format as ParticipantVoteMessage)
    // ... copy key bytes ...

    // Serialize mMaximalClaimingBlockNumber
    // ... existing pattern ...

    // Serialize mTransactionUUID
    // ... existing pattern ...

    // Serialize amount
    // ... existing pattern ...

    // Serialize equivalent
    memcpy(result->data() + offset, &equivalent, sizeof(SerializedEquivalent));

    return result;
}
```

## Step 5: Update All Call Sites
**File:** `BaseExchangePaymentTransaction.cpp`

For each call to `getSerializedReceipt()`:

1. Identify the contractor (recipient of the receipt)
2. Retrieve their payment public key:
   ```cpp
   auto recipientContractor = mContractorsHandler->getContractor(recipientContractorID);
   auto recipientPaymentKey = recipientContractor->paymentPublicKey();
   ```
3. Update the call:
   ```cpp
   auto serializedReceipt = getSerializedReceipt(
       recipientPaymentKey,
       amount,
       mEquivalent  // or however equivalent is accessed
   );
   ```

## Step 6: Handle Contractor Access
- Ensure `ContractorsHandler` is accessible where needed
- May need to add method to get contractor by ID or use existing mechanism
- Consider caching contractors involved in the payment path

## Step 7: Verification
1. Compile the project
2. Run existing tests
3. Verify receipt structure through debug logging or test

# Test Plan

**Complexity Level:** Moderate

This task's testing will be covered in Task 19-06 (E2E tests). For this task, verification focuses on:

1. **Compilation Verification**
   - Project compiles without errors
   - No new warnings

2. **Regression Testing**
   - All existing payment transaction tests pass

3. **Manual Verification (Demo)**
   - Enable debug logging for receipt serialization
   - Execute a test payment transaction
   - Verify serialized receipt contains:
     - Recipient's payment public key (correct size and format)
     - Maximal claiming block number
     - Transaction UUID
     - Amount
     - Equivalent
   - Verify serialized receipt does NOT contain:
     - Source ContractorID
     - Target ContractorID
     - Audit number

# Verification and Validation
_To be completed by Architect after implementation_

## Architecture integrity
- [ ] Serialization format is consistent with existing patterns
- [ ] Receipt structure change is complete (no partial migration)
- [ ] All payment transaction types use updated receipt format

## Security
- [ ] Receipt includes cryptographically verifiable recipient identity
- [ ] No sensitive data inadvertently included or excluded
- [ ] Signature verification will work with new structure

## Performance
- [ ] Serialization performance is comparable to previous implementation
- [ ] Contractor lookup is efficient

## Scalability
- N/A for this task

## Reliability
- [ ] Error handling for missing payment public key
- [ ] Consistent behavior across all transaction types

## Maintainability
- [ ] Code is clear and well-documented
- [ ] Serialization logic follows existing patterns

## Cost
- N/A for this task

## Compliance
- N/A for this task

# Restrictions
- Commit changes only after successfully executing the demo or after successfully passing the tests
- Do not modify Contractor or ContractorsHandler in this task (completed in Task 19-01)
- Do not modify channel messages or transactions in this task (completed in Tasks 19-02, 19-03)
- Do not modify CycleCloser or ConflictResolver transactions in this task (that's Task 19-05)
- Do not implement tests in this task (that's Task 19-06)
