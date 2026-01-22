# 18-03 - TrustLinesManager Receipts Amount Preservation

# Links
- [PRD-18: Audit Mechanism Based on Finalized Transactions](../../prd/vtcpd/18-audit-mechanism-finalized-transactions.md)
- [Previous task: 18-02-receipt-handlers-and-zero-audit-number](18-02-receipt-handlers-and-zero-audit-number.md)

# Description

This task modifies `TrustLinesManager` to support preservation of unrealized receipt amounts after audit completion. Currently, `resetTrustLineTotalReceiptsAmounts()` resets `mTotalIncomingReceiptsAmount` and `mTotalOutgoingReceiptsAmount` to zero. The new mechanism requires preserving the amounts of excluded (unrealized) receipts so they can be included in future audits.

**Example from PRD:**
- Initial state: receipts `uuid1: +1000`, `uuid2: -500`, `uuid3: -300`
- `mTotalIncomingReceiptsAmount=1000`, `mTotalOutgoingReceiptsAmount=800`, `balance=2000`
- Exclude `uuid1` and `uuid3` from audit:
  - New balance = `1300` (apply `-1000 + 300`)
  - Included receipts: `mTotalIncomingReceiptsAmount=0`, `mTotalOutgoingReceiptsAmount=500`
  - After audit, preserved unrealized: `mTotalIncomingReceiptsAmount=1000`, `mTotalOutgoingReceiptsAmount=300`

# Requirements and DOD

## Requirements

1. **New Method in TrustLinesManager**
   - Add method: `void updateTrustLineTotalReceiptsAmounts(ContractorID contractorID, TrustLineAmount includedIncoming, TrustLineAmount includedOutgoing, TrustLineAmount excludedIncoming, TrustLineAmount excludedOutgoing)`
   - This method should:
     - Update trust line balance based on included receipts
     - Set `mTotalIncomingReceiptsAmount` to `excludedIncoming` (preserved for next audit)
     - Set `mTotalOutgoingReceiptsAmount` to `excludedOutgoing` (preserved for next audit)
     - Persist changes to database

2. **TrustLine Class Modifications** (if needed)
   - Add method to support partial update of receipt amounts
   - Or modify existing `resetTotalReceiptsAmounts()` to accept new values

3. **Balance Calculation Logic**
   - New balance = old balance + includedIncoming - includedOutgoing
   - After audit: `mTotalIncomingReceiptsAmount` = excludedIncoming
   - After audit: `mTotalOutgoingReceiptsAmount` = excludedOutgoing

4. **Preserve Existing Method**
   - Keep `resetTrustLineTotalReceiptsAmounts()` for backward compatibility (may be used elsewhere)
   - Or refactor callers to use the new method

## Definition of Done

- [ ] New method added to `TrustLinesManager` for updating receipt amounts with preservation
- [ ] `TrustLine` class updated if necessary
- [ ] Balance calculation is correct per the formula
- [ ] Excluded receipt amounts are preserved (not reset to zero)
- [ ] Database persistence works correctly
- [ ] Code compiles without errors
- [ ] Existing callers of `resetTrustLineTotalReceiptsAmounts` identified and documented

# Implementation Plan

## Step 1: Analyze Existing Code

1. Read `src/core/trust_lines/manager/TrustLinesManager.h` and `.cpp`
2. Read `src/core/trust_lines/TrustLine.h` and `.cpp`
3. Understand `resetTrustLineTotalReceiptsAmounts()` implementation
4. Understand `resetTotalReceiptsAmounts()` in TrustLine class
5. Identify all callers of `resetTrustLineTotalReceiptsAmounts()`
6. Understand how balance and receipt amounts are persisted

## Step 2: Design New Method Signature

1. Define parameters for the new method:
   - `ContractorID contractorID` - identifies the trust line
   - `TrustLineAmount includedIncoming` - sum of incoming receipts included in audit
   - `TrustLineAmount includedOutgoing` - sum of outgoing receipts included in audit
   - `TrustLineAmount excludedIncoming` - sum of incoming receipts excluded from audit
   - `TrustLineAmount excludedOutgoing` - sum of outgoing receipts excluded from audit
2. Alternatively, consider a simpler signature if balance update is done separately

## Step 3: Modify TrustLine Class (if needed)

1. Add method `setTotalReceiptsAmounts(TrustLineAmount incoming, TrustLineAmount outgoing)`
2. Or modify `resetTotalReceiptsAmounts()` to accept optional parameters

## Step 4: Implement New TrustLinesManager Method

1. Add method declaration to `TrustLinesManager.h`
2. Implement in `TrustLinesManager.cpp`:
   ```cpp
   void TrustLinesManager::updateTrustLineTotalReceiptsAmounts(
       ContractorID contractorID,
       TrustLineAmount includedIncoming,
       TrustLineAmount includedOutgoing,
       TrustLineAmount excludedIncoming,
       TrustLineAmount excludedOutgoing)
   {
       // 1. Verify trust line exists
       // 2. Get trust line
       // 3. Calculate new balance: balance += includedIncoming - includedOutgoing
       // 4. Set mTotalIncomingReceiptsAmount = excludedIncoming
       // 5. Set mTotalOutgoingReceiptsAmount = excludedOutgoing
       // 6. Persist to database
   }
   ```

## Step 5: Verification

1. Compile the project
2. Verify no regressions in existing functionality
3. Document which audit transactions will use the new method

## Files to Modify

| File | Changes |
|------|---------|
| `src/core/trust_lines/manager/TrustLinesManager.h` | Add new method declaration |
| `src/core/trust_lines/manager/TrustLinesManager.cpp` | Implement new method |
| `src/core/trust_lines/TrustLine.h` | Add setter method if needed |
| `src/core/trust_lines/TrustLine.cpp` | Implement setter method if needed |

# Test Plan

**Complexity Level:** Simple

## Functional Validation
- Verify new method correctly calculates balance: `new_balance = old_balance + includedIncoming - includedOutgoing`
- Verify `mTotalIncomingReceiptsAmount` is set to `excludedIncoming` after method call
- Verify `mTotalOutgoingReceiptsAmount` is set to `excludedOutgoing` after method call
- Verify changes are persisted to database
- Verify error handling when trust line doesn't exist

## Integration Validation
- Verify method integrates with existing TrustLine persistence mechanism
- Verify no side effects on other trust line properties

# Verification and Validation

## Architecture integrity
- Method follows existing patterns in TrustLinesManager
- No new dependencies introduced
- Maintains separation of concerns

## Security
- N/A (internal trust line management)

## Performance
- Single trust line update operation
- No performance impact compared to existing reset method

## Scalability
- N/A (operates on single trust line)

## Reliability
- Proper error handling for non-existent trust lines
- Atomic update of balance and receipt amounts

## Maintainability
- Clear method name indicates purpose
- Well-documented parameters
- Follows existing code patterns

## Cost
- N/A (no infrastructure changes)

## Compliance
- N/A (internal data management)

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not modify audit transaction logic in this task (handled in subsequent tasks)
- Keep existing `resetTrustLineTotalReceiptsAmounts()` method intact for backward compatibility
- Do not change behavior of other TrustLinesManager methods

# Implementation Notes

Task ID: 18-03

Balance adjustment uses excluded receipts because the stored trust line balance
already reflects all receipts before audit completion.

Existing callers of `resetTrustLineTotalReceiptsAmounts()`:
- `src/core/transactions/transactions/trust_lines/AuditSourceTransaction.cpp`
- `src/core/transactions/transactions/trust_lines/AuditTargetTransaction.cpp`
- `src/core/transactions/transactions/trust_lines/CloseIncomingTrustLineTransaction.cpp`
- `src/core/transactions/transactions/trust_lines/SetOutgoingTrustLineTransaction.cpp`
