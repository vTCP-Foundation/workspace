# 15-01 - GetClaimStatusesRpcRequest Signing Fields

# Links
- [PRD](../../prd/vtcpd/15-completed-payments-observer-monitoring.md)

# Description
Add public key and signature fields to `GetClaimStatusesRpcRequest` class to support authenticated requests to the Observer RPC service. The Observer's `GetClaimStatuses` method now requires authentication via public key and signature fields in the request.

This task updates the existing RPC request class to include storage for these cryptographic fields and ensures they are properly serialized in the JSON output. The actual signing logic (serialization for signing) will be implemented in the transaction class (Task 04).

# Requirements and DOD

## Functional Requirements
1. Add private member `mPublicKey` of type `PublicKey::Shared` to `GetClaimStatusesRpcRequest`
2. Add private member `mSignature` of type `Signature::Shared` to `GetClaimStatusesRpcRequest`
3. Add getter method `publicKey()` returning `PublicKey::Shared`
4. Add getter method `signature()` returning `Signature::Shared`
5. Add setter method `setPublicKey(PublicKey::Shared publicKey)`
6. Add setter method `setSignature(Signature::Shared signature)`
7. Update `toJson()` method to include `public_key` field (base64-encoded)
8. Update `toJson()` method to include `signature` field (base64-encoded)

## Definition of Done
- [ ] All getter/setter methods implemented and accessible
- [ ] JSON serialization includes `public_key` and `signature` fields
- [ ] Code compiles without warnings
- [ ] Existing functionality preserved (claims serialization unchanged)
- [ ] Include statements added for PublicKey and Signature types

# Implementation Plan

## Step 1: Review existing GetClaimStatusesRpcRequest
- File: `src/core/network/rpc/requests/GetClaimStatusesRpcRequest.h`
- File: `src/core/network/rpc/requests/GetClaimStatusesRpcRequest.cpp`
- Understand current structure, constructor, and toJson() implementation

## Step 2: Add member variables
In `GetClaimStatusesRpcRequest.h`:
```cpp
private:
    // ... existing members ...
    PublicKey::Shared mPublicKey;
    Signature::Shared mSignature;
```

## Step 3: Add getter methods
```cpp
public:
    PublicKey::Shared publicKey() const;
    Signature::Shared signature() const;
```

## Step 4: Add setter methods
```cpp
public:
    void setPublicKey(PublicKey::Shared publicKey);
    void setSignature(Signature::Shared signature);
```

## Step 5: Update toJson() method
In `GetClaimStatusesRpcRequest.cpp`, add to JSON output:
- `public_key`: base64-encoded public key bytes (empty string if null)
- `signature`: base64-encoded signature bytes (empty string if null)

Reference existing hex encoding patterns in the codebase (e.g., how TransactionUUID is encoded).

## Step 6: Add necessary includes
- Include `crypto/sphincskeys.h` for PublicKey
- Include `crypto/sphincsscheme.h` for Signature (if not already included)

# Test Plan

**Complexity**: Simple

Unit tests will be implemented in Task 06 (Unit Tests). This task focuses on implementation only.

Expected test coverage (to be implemented in Task 06):
- Public key getter/setter verification
- Signature getter/setter verification
- JSON serialization includes both new fields
- Null handling for optional fields

# Verification and Validation

## Architecture integrity
- Follows existing RPC request patterns in the codebase
- No changes to class hierarchy or interfaces
- Getter/setter pattern consistent with other RPC classes

## Security
- Public key and signature are stored as shared pointers (no raw memory management)
- Hex encoding prevents injection in JSON output
- No validation logic here (validation done at signing time in transaction)

## Performance
- Minimal overhead: two additional shared pointer members
- Lazy initialization (null until set)
- No impact on existing claims serialization performance

## Scalability
- N/A - simple data storage extension

## Reliability
- Null-safe getters (return nullptr if not set)
- Existing functionality unchanged

## Maintainability
- Clear naming convention matching Observer RPC spec
- Getter/setter pattern consistent with codebase style
- Well-documented purpose in code comments

## Cost
- N/A

## Compliance
- Follows Observer RPC protocol specification from `observer/README.md`

# Restrictions
- Do not implement signing logic (done in Transaction class)
- Do not modify claims serialization format
- Commit changes only after code compiles without warnings
