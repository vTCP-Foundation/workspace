# 18-01 - Audit Messages Extension

# Links
- [PRD-18: Audit Mechanism Based on Finalized Transactions](../../prd/vtcpd/18-audit-mechanism-finalized-transactions.md)

# Description

This task extends `AuditMessage` and `AuditResponseMessage` classes to include a list of transaction UUIDs. This enables both audit parties to agree on the exact set of transactions included in each audit.

The transaction UUID list must be:
- Lexicographically sorted by raw UUID bytes (same ordering as `compareTransactionUUID` in `CompletedPaymentsObserverMonitoringTransaction::runGetClaimStatusesStage`)
- Hashed using SHA-256 for inclusion in the audit signature payload

This is a foundational task required by all subsequent audit transaction modifications.

# Requirements and DOD

## Requirements

1. **AuditMessage Extension**
   - Add `vector<TransactionUUID> mTransactionUUIDs` member field
   - Update constructor(s) to accept transaction UUID list
   - Implement getter method `transactionUUIDs()`
   - Update `serializeToBytes()` to include transaction list
   - Update deserialization constructor to parse transaction list
   - Implement static method for sorting UUIDs lexicographically by raw bytes
   - Implement static method for computing SHA-256 hash of sorted transaction list
   - Ensure the transaction UUID list contains no duplicates (deduplicate after sorting)
   - Hash format: SHA-256 over `uint32_t count` (same byte order as `serializeGetClaimStatusesForSigning`) + concatenated `TransactionUUID::data` in sorted order

2. **AuditResponseMessage Extension**
   - Add `vector<TransactionUUID> mTransactionUUIDs` member field
   - Update constructor(s) to optionally accept transaction UUID list (used with `Audit_UpdateTransactionsList` status)
   - Implement getter method `transactionUUIDs()`
   - Update `serializeToBytes()` to include transaction list
   - Update deserialization constructor to parse transaction list

3. **Sorting Implementation**
   - Use same comparison logic as `compareTransactionUUID` in `CompletedPaymentsObserverMonitoringTransaction::runGetClaimStatusesStage`
   - Sorting must be deterministic across all nodes

4. **Hash Implementation**
   - SHA-256 algorithm
   - Input format: `uint32_t count` (same byte order as `serializeGetClaimStatusesForSigning`) followed by concatenated UUID data in sorted order
   - Deduplicate the list before hashing to ensure uniqueness
   - Output: 32-byte hash to be included in audit signature payload after balance and before `equivalentRegistryAddress`

## Definition of Done

- [ ] `AuditMessage` class extended with transaction UUID vector
- [ ] `AuditResponseMessage` class extended with transaction UUID vector
- [ ] Sorting function implemented and matches `compareTransactionUUID` behavior
- [ ] Hash function implemented according to specification
- [ ] Transaction list is deduplicated before serialization and hashing
- [ ] Serialization/deserialization works correctly for both message types
- [ ] Code compiles without errors
- [ ] Existing functionality not broken (backward compatibility not required per PRD)

# Implementation Plan

## Step 1: Analyze Existing Code

1. Read `src/core/network/messages/trust_lines/AuditMessage.h` and `.cpp`
2. Read `src/core/network/messages/trust_lines/AuditResponseMessage.h` and `.cpp`
3. Read `CompletedPaymentsObserverMonitoringTransaction` to understand `compareTransactionUUID` implementation
4. Read `serializeGetClaimStatusesForSigning` to understand byte order for count serialization

## Step 2: Implement UUID Sorting

1. Create or reuse a comparison function for `TransactionUUID` that compares raw bytes lexicographically
2. Ensure the function signature allows use with `std::sort`
3. Consider placing in a utility location if reusable (or inline in message class)

## Step 3: Implement Hash Function

1. Create a static method `computeTransactionListHash(const vector<TransactionUUID>& sortedUUIDs)`
2. Serialize `uint32_t` count in the same byte order as `serializeGetClaimStatusesForSigning`
3. Concatenate all UUID data bytes in order
4. Compute SHA-256 hash
5. Return hash as appropriate type (e.g., `BytesShared` or fixed-size array)

## Step 4: Extend AuditMessage

1. Add `vector<TransactionUUID> mTransactionUUIDs` member
2. Update constructor to accept transaction list parameter
3. Add getter `const vector<TransactionUUID>& transactionUUIDs() const`
4. Update `serializeToBytes()`:
   - Serialize count of UUIDs
   - Serialize each UUID
5. Update deserialization constructor:
   - Read count
   - Read each UUID into vector

## Step 5: Extend AuditResponseMessage

1. Add `vector<TransactionUUID> mTransactionUUIDs` member
2. Update constructor to optionally accept transaction list (default empty)
3. Add getter `const vector<TransactionUUID>& transactionUUIDs() const`
4. Update `serializeToBytes()`:
   - Serialize count of UUIDs
   - Serialize each UUID (may be 0 for most responses)
5. Update deserialization constructor:
   - Read count
   - Read each UUID into vector

## Step 6: Verification

1. Write simple test code to verify serialization round-trip
2. Verify sorting produces consistent results
3. Verify hash computation matches expected format

## Files to Modify

| File | Changes |
|------|---------|
| `src/core/network/messages/trust_lines/AuditMessage.h` | Add member, update declarations |
| `src/core/network/messages/trust_lines/AuditMessage.cpp` | Implement new functionality |
| `src/core/network/messages/trust_lines/AuditResponseMessage.h` | Add member, update declarations |
| `src/core/network/messages/trust_lines/AuditResponseMessage.cpp` | Implement new functionality |

# Test Plan

**Complexity Level:** Simple

## Functional Validation
- Verify `AuditMessage` can be constructed with transaction UUID list
- Verify `AuditMessage` serialization includes transaction list
- Verify `AuditMessage` deserialization correctly reconstructs transaction list
- Verify `AuditResponseMessage` can be constructed with and without transaction list
- Verify `AuditResponseMessage` serialization/deserialization works correctly
- Verify sorting function produces lexicographically sorted output by raw UUID bytes
- Verify hash function produces consistent 32-byte SHA-256 output

## Integration Validation
- Verify modified messages integrate with existing message handling infrastructure
- Verify message size calculations are updated correctly

# Verification and Validation

## Architecture integrity
- Messages follow existing patterns in codebase
- Sorting and hashing logic consistent with `CompletedPaymentsObserverMonitoringTransaction`
- No new external dependencies introduced

## Security
- Hash algorithm is SHA-256 (cryptographically secure)
- No sensitive data exposure in message serialization

## Performance
- Sorting is O(n log n) which is acceptable for expected transaction counts (<100)
- Hash computation is single-pass O(n)
- No performance regression in existing message handling

## Scalability
- N/A for this task (message format change)

## Reliability
- Deterministic sorting ensures all nodes produce identical results
- Hash computation is deterministic

## Maintainability
- Code follows existing patterns in message classes
- Clear separation of sorting and hashing logic

## Cost
- N/A (no infrastructure changes)

## Compliance
- N/A (internal protocol change)

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify audit transaction logic in this task (handled in subsequent tasks)
- Do not modify receipt handlers in this task
