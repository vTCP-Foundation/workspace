# 10-01 - Command Extension and Result Code 415

# Links
- [PRD](../../prd/vtcpd/10-allowable-payment-amount-control.md)

# Description
Extend `CreditUsageExchangeCommand` with optional `maxAllowablePaymentAmount` parameter to allow users to specify maximum payment amount limit. Add new result code 415 "Allowable payment amount has been exceeded" for returning specific error when limit is violated. This provides the foundation for user-controlled spending limits in exchange payments.

# Requirements and DOD

## Functional Requirements
1. Add `optional<TrustLineAmount> mMaxAllowablePaymentAmount` field to `CreditUsageExchangeCommand` class
2. Add getter method `const optional<TrustLineAmount>& maxAllowablePaymentAmount() const`
3. Extend command parsing to accept optional parameter after `exchangeEquivalents`:
   - Command format with parameter: `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1:1500`
   - Command format without parameter (legacy): `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1`
4. Validate parameter during parsing: if provided, must be positive `TrustLineAmount`
5. Add method `responseAllowablePaymentAmountExceeded()` to `CreditUsageExchangeCommand` returning `CommandResult` with code 415
6. Add method `resultAllowablePaymentAmountExceeded()` to `CoordinatorExchangePaymentTransaction`
7. Implementation must follow existing pattern from line 2590-2591 in `CoordinatorExchangePaymentTransaction.cpp`:
   ```cpp
   reject("Allowable payment amount exceeded");
   return resultAllowablePaymentAmountExceeded();
   ```

## Definition of Done
- [ ] `CreditUsageExchangeCommand.h` updated with new field and getter
- [ ] `CreditUsageExchangeCommand.cpp` parsing logic handles optional parameter
- [ ] Both command formats (with and without parameter) parse correctly
- [ ] Invalid parameter (negative, zero) causes parsing error
- [ ] Method `responseAllowablePaymentAmountExceeded()` implemented in `CreditUsageExchangeCommand`
- [ ] Method `resultAllowablePaymentAmountExceeded()` implemented in `CoordinatorExchangePaymentTransaction`
- [ ] Result code 415 with message "Allowable payment amount has been exceeded" returned correctly
- [ ] All code compiles without errors or warnings
- [ ] Code follows existing patterns and naming conventions
- [ ] Backward compatibility maintained (commands without parameter work unchanged)

# Implementation Plan

## Step 1: Extend CreditUsageExchangeCommand Header
**File**: `src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.h`

**Changes**:
1. Add private field after existing fields:
   ```cpp
   optional<TrustLineAmount> mMaxAllowablePaymentAmount;
   ```
2. Add public getter method after existing getters:
   ```cpp
   const optional<TrustLineAmount>& maxAllowablePaymentAmount() const;
   ```
3. Add public response method:
   ```cpp
   CommandResult::SharedConst responseAllowablePaymentAmountExceeded() const;
   ```

## Step 2: Implement Command Parsing
**File**: `src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.cpp`

**Changes in constructor**:
1. Add lambda for parsing max allowable amount:
   ```cpp
   auto maxAllowableAmountAddNumber = [&](auto &ctx) {
       maxAllowableAmount += _attr(ctx);
       maxAllowableAmountDigitsCounter++;
       if (maxAllowableAmountDigitsCounter == 1 && _attr(ctx) == '0') {
           throw ValueError("CreditUsageExchangeCommand: maxAllowablePaymentAmount contains leading zero.");
       }
   };
   ```

2. Update parsing grammar to accept optional parameter after `exchangeEquivalents` and before `payload`:
   ```cpp
   parse(
       commandTail.begin(),
       commandTail.end(),
       (
           *(digit[amountAddNumber] > !alpha > !punct)
           > char_(kTokensSeparator)
           > +(int_[equivalentParse])
           > *(char_(kTokensSeparator) > int_[exchangeEquivalentParse])
           > -(char_(kTokensSeparator) > +(digit[maxAllowableAmountAddNumber])) // NEW: optional
           > -(char_(kTokensSeparator) > *(char_[payloadParse] - eol))
           > eol > eoi));
   ```

3. After parsing, if max allowable amount string is not empty:
   ```cpp
   if (!maxAllowableAmount.empty()) {
       mMaxAllowablePaymentAmount = TrustLineAmount(maxAllowableAmount);
   }
   ```

## Step 3: Implement Getter and Response Methods
**File**: `src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.cpp`

**Getter implementation**:
```cpp
const optional<TrustLineAmount>& CreditUsageExchangeCommand::maxAllowablePaymentAmount() const
{
    return mMaxAllowablePaymentAmount;
}
```

**Response method implementation**:
```cpp
CommandResult::SharedConst CreditUsageExchangeCommand::responseAllowablePaymentAmountExceeded() const
{
    return CommandResult::SharedConst(
        new CommandResult(
            identifier(),
            UUID(),
            415,
            "Allowable payment amount has been exceeded"));
}
```

## Step 4: Add Result Method to CoordinatorExchangePaymentTransaction
**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h`

Add protected method declaration:
```cpp
TransactionResult::SharedConst resultAllowablePaymentAmountExceeded();
```

**File**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.cpp`

Implementation:
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::resultAllowablePaymentAmountExceeded()
{
    return transactionResultFromCommand(
        mCommand->responseAllowablePaymentAmountExceeded());
}
```

# Test Plan

## Manual Testing
1. **Test command parsing with parameter**:
   - Input: `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1:1500`
   - Verify: `maxAllowablePaymentAmount()` returns `optional<TrustLineAmount>(1500)`
   - Verify: All other parameters parsed correctly

2. **Test command parsing without parameter (legacy)**:
   - Input: `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1`
   - Verify: `maxAllowablePaymentAmount()` returns `nullopt`
   - Verify: All other parameters parsed correctly

3. **Test command parsing with payload**:
   - Input: `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1:1500:payload_data`
   - Verify: Both max allowable amount and payload parsed correctly

4. **Test invalid parameter (leading zero)**:
   - Input with leading zero in max allowable amount
   - Verify: Parsing throws ValueError

5. **Test result code 415**:
   - Call `responseAllowablePaymentAmountExceeded()`
   - Verify: Returns CommandResult with code 415
   - Verify: Message is "Allowable payment amount has been exceeded"

6. **Test result method in transaction**:
   - Call `resultAllowablePaymentAmountExceeded()` in coordinator transaction
   - Verify: Returns TransactionResult with code 415

## Complexity
**Moderate** - Requires command parsing extension and new result code, but follows existing patterns

# Verification and Validation

## Architecture integrity
- Command structure extended following existing optional parameter patterns
- Result code added using existing result code infrastructure
- No changes to transaction flow or state management
- Backward compatible with existing command format

## Security
- Parameter validation prevents negative or zero values
- No exposure of internal state
- No new attack vectors introduced

## Performance
- Minimal overhead: single optional field added to command
- Parsing overhead negligible (one additional optional parse step)
- No impact on transaction performance

## Scalability
- No scalability concerns
- Optional parameter adds no complexity to scaling

## Reliability
- Parsing errors handled gracefully with ValueError
- Optional parameter ensures backward compatibility
- No impact on existing command processing

## Maintainability
- Code follows existing patterns (similar to other optional parameters)
- Clear naming conventions
- Self-documenting through method names
- Minimal code changes isolated to command and result handling

## Cost
- Development effort: 1-2 days
- No infrastructure cost changes
- No additional runtime costs

## Compliance
- Follows project coding standards
- Adheres to existing command extension patterns
- Maintains API backward compatibility
- No breaking changes to existing functionality

# Restrictions
- Commit changes only after successfully passing manual tests
- Do not modify transaction logic in this task (only command and result code)
- Do not implement validation logic (handled in task 10-02)
