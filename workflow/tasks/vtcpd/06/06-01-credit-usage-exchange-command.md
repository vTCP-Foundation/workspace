# 06-01 - CreditUsageExchangeCommand

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)

# Description
Implement CreditUsageExchangeCommand to initiate multi-equivalent payment transactions. This command extends the functionality of CreditUsageCommand by adding support for exchangeEquivalents parameter, allowing senders to specify multiple equivalents they can pay with while the receiver accepts funds in their preferred equivalent.

The command follows the pattern established by InitiateMaxFlowExchangeCalculationCommand for parsing exchangeEquivalents and enforces a maximum limit of 5 exchange equivalents per payment.

# Requirements and DOD

## Requirements
1. Create CreditUsageExchangeCommand class analogous to CreditUsageCommand
2. Add `vector<SerializedEquivalent> exchangeEquivalents` field
3. Implement command parsing following InitiateMaxFlowExchangeCalculationCommand pattern for exchangeEquivalents
4. Validate exchangeEquivalents limit (maximum 5 elements, error 401 if violated)
5. Support optional payload parameter (max 65535 characters)
6. Implement responseOK() method returning status 201 with transactionUUID
7. Command identifier: `"CREATE:contractors/transactions/exchange"`
8. Command format: `CREATE:contractors/transactions/exchange:<contractor_addresses_count>:<address_type>:<address>:<amount>:<receiver_equivalent>:<exchange_equivalent_1>[:<exchange_equivalent_2>:...][:<payload>]`

## Definition of Done
- [x] CreditUsageExchangeCommand.h created in `src/core/interface/commands_interface/commands/payments/`
- [x] CreditUsageExchangeCommand.cpp created with full implementation
- [x] Constructor parses all parameters correctly (contractor address, amount, receiver_equivalent, exchangeEquivalents, optional payload)
- [x] exchangeEquivalents parsed as vector with validation after parsing (≤5 elements)
- [x] Error 401 thrown when exchangeEquivalents > 5 elements
- [x] Error 401 thrown for invalid command format or parameters
- [x] responseOK() returns CommandResult with status 201 and transactionUUID
- [x] identifier() returns `"CREATE:contractors/transactions/exchange"`
- [x] Payload length validation (≤65535 characters) with appropriate error
- [x] All getter methods implemented: contractorAddresses(), amount(), equivalent(), exchangeEquivalents(), payload()
- [x] Code compiles without errors or warnings

# Implementation Plan

## Step 1: Create Header File
**File**: `src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.h`

```cpp
#ifndef VTCPD_CREDITUSAGEEXCHANGECOMMAND_H
#define VTCPD_CREDITUSAGEEXCHANGECOMMAND_H

#include "../BaseUserCommand.h"
#include "../../../../common/multiprecision/MultiprecisionUtils.h"
#include "../../../../common/exceptions/MemoryError.h"

class CreditUsageExchangeCommand : public BaseUserCommand
{
public:
    typedef shared_ptr<CreditUsageExchangeCommand> Shared;

public:
    CreditUsageExchangeCommand(
        const CommandUUID &uuid,
        const string &commandBuffer);

    static const string &identifier();

    const TrustLineAmount& amount() const;
    vector<BaseAddress::Shared> contractorAddresses() const;
    const SerializedEquivalent equivalent() const;
    const vector<SerializedEquivalent> &exchangeEquivalents() const;
    const std::string payload() const;

public:
    // Results handlers
    CommandResult::SharedConst responseOK(
        string &transactionUUID) const;

private:
    vector<BaseAddress::Shared> mContractorAddresses;
    TrustLineAmount mAmount;
    SerializedEquivalent mEquivalent;
    vector<SerializedEquivalent> mExchangeEquivalents;
    std::string mPayload;
};

#endif //VTCPD_CREDITUSAGEEXCHANGECOMMAND_H
```

## Step 2: Implement Parsing Logic
**File**: `src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.cpp`

### Constructor Implementation
1. Parse contractor addresses count (similar to CreditUsageCommand)
2. Parse address type and address using addressLexeme
3. Parse amount with leading zero validation
4. Parse receiver_equivalent (mEquivalent)
5. Parse exchangeEquivalents vector (similar to InitiateMaxFlowExchangeCalculationCommand)
6. Parse optional payload
7. Validate exchangeEquivalents.size() ≤ 5 after parsing
8. Validate payload.length() ≤ 65535

### Key Parsing Lambdas
- `addressesCountParse`: parse contractor addresses count
- `addressTypeParse`: parse address type
- `addressAddChar`/`addressAddNumber`: build address string
- `addressAddToVector`: create BaseAddress and add to vector
- `amountAddNumber`: build amount string with leading zero check
- `equivalentParse`: parse receiver equivalent
- `exchangeEquivalentParse`: push to mExchangeEquivalents vector
- `payloadParse`: build payload string

### Parser Structure
```cpp
parse(command, *(int_[addressesCountParse] - char_(kTokensSeparator)) > char_(kTokensSeparator));
parse(command, *(int_) > char_(kTokensSeparator) > addressLexeme<...>(...) > *(char_[scommand]));
parse(scommand, *(digit[amountAddNumber]) > char_(kTokensSeparator) > int_[equivalentParse]
    > *(char_(kTokensSeparator) > int_[exchangeEquivalentParse])
    > -(char_(kTokensSeparator) > *(char_[payloadParse] - eol)) > eol > eoi);

// Validation after parsing
if (mExchangeEquivalents.size() > 5) {
    throw ValueError("CreditUsageExchangeCommand: exchangeEquivalents limit exceeded (maximum 5 elements).");
}
if (mPayload.length() > std::numeric_limits<PayloadLength>::max()) {
    throw ValueError("Payload length is too big");
}
```

## Step 3: Implement Methods
1. `identifier()`: return `"CREATE:contractors/transactions/exchange"`
2. `contractorAddresses()`: return mContractorAddresses
3. `amount()`: return mAmount
4. `equivalent()`: return mEquivalent (receiver equivalent)
5. `exchangeEquivalents()`: return mExchangeEquivalents
6. `payload()`: return mPayload
7. `responseOK(transactionUUID)`: return CommandResult with status 201 and transactionUUID

## Step 4: Integration
1. Register command in CommandsInterface (if not done automatically)
2. Ensure TransactionsManager can handle this command type
3. Verify command identifier is unique

# Test Plan

## Unit Tests
Since this is a Simple task, focused unit tests on core functionality:

### Test Category: Command Parsing
1. **testParseValidCommandSingleExchangeEquivalent**: Parse command with 1 exchange equivalent
   - Input: `"1:12:127.0.0.1:2003:1000:2:1"`
   - Expected: amount=1000, equivalent=2, exchangeEquivalents=[1]

2. **testParseValidCommandMultipleExchangeEquivalents**: Parse command with 3 exchange equivalents
   - Input: `"1:12:127.0.0.1:2004:500:3:1:2:5"`
   - Expected: amount=500, equivalent=3, exchangeEquivalents=[1,2,5]

3. **testParseValidCommandWithPayload**: Parse command with payload
   - Input: `"1:12:127.0.0.1:2005:2000:1:2:Invoice-12345"`
   - Expected: amount=2000, equivalent=1, exchangeEquivalents=[2], payload="Invoice-12345"

4. **testParseValidCommandMaxExchangeEquivalents**: Parse command with exactly 5 exchange equivalents
   - Input: `"1:12:127.0.0.1:2006:100:1:1:2:3:4:5"`
   - Expected: exchangeEquivalents.size()=5, no error

### Test Category: Validation
5. **testExchangeEquivalentsLimitExceeded**: Parse command with 6 exchange equivalents
   - Expected: throw ValueError with message about exceeding limit

6. **testPayloadLengthExceeded**: Parse command with payload > 65535 characters
   - Expected: throw ValueError about payload length

7. **testInvalidAmountLeadingZero**: Parse command with amount starting with zero
   - Input: amount="0123"
   - Expected: throw ValueError

8. **testInvalidAddressType**: Parse command with unsupported address type
   - Expected: throw ValueError

### Test Category: Response
9. **testResponseOK**: Call responseOK() with valid transactionUUID
   - Expected: CommandResult with status 201, correct identifier, and transactionUUID in body

### Test Category: Getters
10. **testGettersReturnCorrectValues**: Verify all getters return parsed values
    - Test contractorAddresses(), amount(), equivalent(), exchangeEquivalents(), payload()

## Success Criteria
- All 10 unit tests pass
- Code compiles without warnings
- Command successfully parsed by CommandsInterface
- TransactionsManager can process the command

# Verification and Validation

## Architecture integrity
- Command follows established pattern from CreditUsageCommand and InitiateMaxFlowExchangeCalculationCommand
- Integrates with existing CommandsInterface and BaseUserCommand infrastructure
- Uses standard parsing lambdas and error handling

## Security
- Input validation for all parameters (address, amount, equivalents, payload)
- Payload length limit enforced (≤65535 characters)
- exchangeEquivalents limit enforced (≤5 elements)
- No buffer overflows in string operations

## Performance
- Parsing complexity: O(n) where n is command string length
- Memory: O(k) where k is number of exchange equivalents (max 5)
- No performance concerns for typical command sizes

## Scalability
- Maximum 5 exchange equivalents per command (as per PRD 04 design)
- Suitable for expected usage patterns

## Reliability
- Comprehensive error handling for malformed commands
- Clear error messages for validation failures
- Graceful handling of edge cases (empty payload, single equivalent, max equivalents)

## Maintainability
- Clear code structure following existing command patterns
- Well-documented parsing logic with lambda functions
- Consistent naming conventions
- Easy to extend if needed in future

## Cost
- No additional infrastructure required
- Minimal computational overhead (parsing only)

## Compliance
- Follows repository policy for task-driven development
- Adheres to existing code style and patterns
- No prohibited operations or scope creep

# Restrictions
- Commit changes only after successfully passing all unit tests
