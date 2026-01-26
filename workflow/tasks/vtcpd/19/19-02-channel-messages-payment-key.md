# 19-02 - Channel Messages Payment Public Key Extension

# Links
- [PRD-19: Debt Receipt Signature Structure Update](../../prd/vtcpd/19-debt-receipt-signature-structure-update.md)
- [Previous Task 19-01: Contractor Payment Public Key Storage](19-01-contractor-payment-key-storage.md)

# Description
This task extends the channel creation messages (`InitChannelMessage` and `ConfirmChannelMessage`) to include payment public keys. These messages are used during the channel establishment protocol, and adding payment public key exchange at this stage ensures that both parties have each other's payment keys before any payment transactions occur.

**Context from PRD:**
- Currently, payment public keys are only exchanged during payment flow, which is too late for debt receipt creation
- Channel creation is the appropriate time to exchange payment keys
- Both `InitChannelMessage` (sent by initiator) and `ConfirmChannelMessage` (sent by responder) need to include the sender's payment public key

**Serialization Note from PRD:**
- `recipientPaymentPublicKey` is serialized using the same format and endianness as in payment flow messages `ParticipantVoteMessage` and `ParticipantsVotesMessage`

# Requirements and DOD

## Functional Requirements

### 1. InitChannelMessage Extension
- Add `sphincs::PublicKey::Shared mPaymentPublicKey` field
- Update constructors to accept payment public key parameter
- Update `serializeToBytes()` to serialize the payment public key
- Update deserialization constructor to extract payment public key from buffer
- Add accessor method `sphincs::PublicKey::Shared paymentPublicKey() const`

### 2. ConfirmChannelMessage Extension
- Add `sphincs::PublicKey::Shared mPaymentPublicKey` field
- Update constructors to accept payment public key parameter
- Update `serializeToBytes()` to serialize the payment public key
- Update deserialization constructor to extract payment public key from buffer
- Add accessor method `sphincs::PublicKey::Shared paymentPublicKey() const`

### 3. Serialization Compatibility
- Use the same serialization format as `ParticipantVoteMessage` and `ParticipantsVotesMessage` for SPHINCS+ public keys
- Ensure endianness is consistent with existing payment flow messages

## Definition of Done
- [ ] `InitChannelMessage` has `mPaymentPublicKey` field with getter
- [ ] `InitChannelMessage` serializes payment public key in `serializeToBytes()`
- [ ] `InitChannelMessage` deserializes payment public key in constructor
- [ ] `ConfirmChannelMessage` has `mPaymentPublicKey` field with getter
- [ ] `ConfirmChannelMessage` serializes payment public key in `serializeToBytes()`
- [ ] `ConfirmChannelMessage` deserializes payment public key in constructor
- [ ] Serialization format matches `ParticipantVoteMessage` pattern
- [ ] Code compiles without errors
- [ ] Existing tests pass (no regression)

# Implementation Plan

## Step 1: Analyze Existing Serialization Pattern
**Reference Files:**
- `src/core/network/messages/payments/ParticipantVoteMessage.h/.cpp`
- `src/core/network/messages/payments/ParticipantsVotesMessage.h/.cpp`

**Actions:**
1. Study how `sphincs::PublicKey` is serialized in these messages
2. Note the serialization method (likely `serialize()` to vector<byte_t>)
3. Note how size is handled (fixed size or length-prefixed)
4. Note byte order/endianness

## Step 2: Update InitChannelMessage
**Files:**
- `src/core/network/messages/trust_line_channels/InitChannelMessage.h`
- `src/core/network/messages/trust_line_channels/InitChannelMessage.cpp`

**Changes to Header:**
1. Add include for SPHINCS+ public key type
2. Add private field: `sphincs::PublicKey::Shared mPaymentPublicKey;`
3. Update constructor signature to include payment public key parameter
4. Add accessor: `sphincs::PublicKey::Shared paymentPublicKey() const;`

**Changes to Implementation:**
1. Update sending constructor to accept and store payment public key
2. Update `serializeToBytes()`:
   - Serialize payment public key after existing fields
   - Use same pattern as `ParticipantVoteMessage`
3. Update deserialization constructor:
   - Extract payment public key from buffer
   - Create `sphincs::PublicKey::Shared` from deserialized data

## Step 3: Update ConfirmChannelMessage
**Files:**
- `src/core/network/messages/trust_line_channels/ConfirmChannelMessage.h`
- `src/core/network/messages/trust_line_channels/ConfirmChannelMessage.cpp`

**Changes:**
Apply same pattern as InitChannelMessage:
1. Add `mPaymentPublicKey` field
2. Update constructors
3. Update serialization/deserialization
4. Add accessor method

## Step 4: Update Message Size Constants (if applicable)
If the message classes use size constants, update them to account for the new field.

## Step 5: Verification
1. Compile the project
2. Verify no existing tests break
3. Create a simple serialization round-trip test (can be temporary/debug code)

# Test Plan

**Complexity Level:** Simple

This task's unit tests will be implemented in Task 19-06. For this task, verification focuses on:

1. **Compilation Verification**
   - Project compiles without errors
   - No new warnings

2. **Manual Serialization Verification (Demo)**
   - Create temporary test code that:
     - Creates `InitChannelMessage` with a payment public key
     - Serializes to bytes
     - Deserializes back to message object
     - Verifies payment public key matches original
   - Repeat for `ConfirmChannelMessage`

3. **Pattern Consistency Check**
   - Verify serialization code matches pattern used in `ParticipantVoteMessage`

# Verification and Validation
_To be completed by Architect after implementation_

## Architecture integrity
- [ ] Serialization format is consistent with existing SPHINCS+ key serialization in payment messages
- [ ] Message structure follows existing patterns in the codebase
- [ ] Message format change is expected; mixed versions are not supported

## Security
- [ ] Public key is properly validated during deserialization
- [ ] No buffer overflow risks in serialization/deserialization

## Performance
- [ ] Serialization overhead is minimal (single key per message)
- [ ] No unnecessary copies of key data

## Scalability
- N/A for this task

## Reliability
- [ ] Deserialization handles malformed data gracefully
- [ ] Error handling follows existing message patterns

## Maintainability
- [ ] Code follows existing message class patterns
- [ ] Serialization logic is clear and documented

## Cost
- N/A for this task

## Compliance
- N/A for this task

# Restrictions
- Commit changes only after successfully executing the demo or after successfully passing the tests
- Do not modify any files outside the scope defined in Implementation Plan
- Do not modify transaction classes in this task (that's Task 19-03)
- Do not implement unit tests in this task (that's Task 19-06)
