# 08-02 - Incoming Reservation Calculation in Intermediate Node

# Links
- [PRD](../../../prd/vtcpd/08-path-capacity-adjustment.md)

# Description
Implement TODO at line 491 in `IntermediateNodeExchangePaymentTransaction::runNextNeighborResponseProcessingStage()` to calculate required incoming reservation amount when the outgoing reservation is reduced by the next node.

When an intermediate node receives a response from the next node with a reduced reservation amount, it must:
1. Find the incoming reservation by the same PathID
2. Determine if equivalents match (incoming vs outgoing)
3. Calculate required incoming amount:
   - **Same equivalent**: Add commission (if exists) to outgoing amount
   - **Different equivalents**: Invert exchange rate to calculate incoming from outgoing (using ceiling division)
4. Call `shortageIncomingReservationsOnPath` with calculated amount
5. Handle errors: exchange rate not found → send error to coordinator and terminate

This ensures reservation balance is maintained across the intermediate node when capacity changes occur.

# Requirements and DOD

## Functional Requirements
1. **Location**: Replace TODO comment at line ~491 in `IntermediateNodeExchangePaymentTransaction.cpp`
2. **Get outgoing reservation details**:
   - Outgoing amount from message: `message->amount()`
   - Outgoing equivalent from outgoing reservation
   - PathID from context
3. **Find incoming reservation**:
   - Search through `mReservations` for reservation with same PathID and direction == `Incoming`
   - Extract `incomingEquivalent` from found reservation
   - If not found: send error to coordinator, terminate transaction
4. **Calculate incoming amount based on equivalents**:
   - **If `incomingEquiv == outgoingEquiv`**:
     - Check for commission: `mCommissionsManager->get(incomingEquiv)`
     - If no commission: `incomingAmount = outgoingAmount`
     - If commission exists: `incomingAmount = outgoingAmount + commission->amount()`
   - **If `incomingEquiv != outgoingEquiv`**:
     - Look up exchange rate: `mExchangeRatesManager->get(incomingEquiv, outgoingEquiv)`
     - If rate not found: send error to coordinator (`RejectedDueOtherTransaction`), terminate
     - Invert exchange: calculate incoming from outgoing using ceiling division
     - Use helper `invertExchangeForRequiredInput(outgoingAmount, rate, shift)`
5. **Update incoming reservation**:
   - Call `shortageIncomingReservationsOnPath(pathID, incomingEquivalent, incomingAmount)`
6. **Error handling**:
   - Incoming reservation not found → error to coordinator
   - Exchange rate not found → error to coordinator
   - Calculation exception → error to coordinator
7. **Logging**:
   - Debug log for same equivalent (with/without commission)
   - Debug log for exchange calculation with rate details
   - Error logs for failures

## Definition of Done
- [ ] TODO comment at line 491 replaced with implementation
- [ ] Incoming reservation found by PathID with correct direction check
- [ ] Same equivalent path: commission correctly added (if exists)
- [ ] Different equivalent path: exchange rate correctly inverted with ceiling division
- [ ] `shortageIncomingReservationsOnPath` called with correct parameters
- [ ] Exchange rate not found: error sent to coordinator, transaction terminates
- [ ] Incoming reservation not found: error sent to coordinator, transaction terminates
- [ ] All paths logged (debug for success, error for failures)
- [ ] Helper function `invertExchangeForRequiredInput` implemented (if doesn't exist)

# Implementation Plan

## Step 1: Locate TODO and understand context
- File: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.cpp`
- Method: `runNextNeighborResponseProcessingStage()`
- Line ~491: TODO comment about incoming reservation calculation
- Understand existing outgoing reservation handling above TODO

## Step 2: Get outgoing reservation details
```cpp
// This code replaces the TODO comment at line 491

// Get outgoing reservation details
const TrustLineAmount outgoingAmount = message->amount();  // from response message
// Get outgoing equivalent from the outgoing reservation (adjust based on actual structure)
const SerializedEquivalent outgoingEquiv = /* extract from outgoing reservation */;
const PathID pathID = /* get from message or context */;
```

## Step 3: Find incoming reservation
```cpp
// Find incoming reservation by same PathID
AmountReservation::ConstShared incomingReservation = nullptr;
SerializedEquivalent incomingEquiv;

for (const auto &[contractorID, reservationsMap] : mReservations) {
    for (const auto &[resPathID, reservation] : reservationsMap) {
        if (resPathID == pathID &&
            reservation->direction() == AmountReservation::Incoming) {
            incomingReservation = reservation;
            incomingEquiv = reservation->equivalent();
            break;
        }
    }
    if (incomingReservation) break;
}

if (!incomingReservation) {
    error() << "Incoming reservation not found for pathID=" << pathID;
    return sendErrorMessageOnNextNodeResponse(
        ResponseMessage::RejectedDueOtherTransaction);
}
```

## Step 4: Calculate incoming amount
```cpp
// Calculate required incoming amount
TrustLineAmount incomingAmount;

if (incomingEquiv == outgoingEquiv) {
    // Same equivalent: check for commission
    auto commission = mCommissionsManager->get(incomingEquiv);

    if (commission && commission->amount() > TrustLineAmount(0)) {
        // Add commission to outgoing to get incoming
        incomingAmount = outgoingAmount + commission->amount();

        debug() << "Incoming amount with commission: "
                << "outgoing=" << outgoingAmount
                << ", commission=" << commission->amount()
                << ", incoming=" << incomingAmount;
    } else {
        // No commission: amounts equal
        incomingAmount = outgoingAmount;

        debug() << "Incoming amount (no commission): " << incomingAmount;
    }
} else {
    // Different equivalents: need exchange rate
    auto exchangeRate = mExchangeRatesManager->get(incomingEquiv, outgoingEquiv);

    if (!exchangeRate) {
        error() << "Exchange rate not found: "
                << incomingEquiv << " -> " << outgoingEquiv;
        return sendErrorMessageOnNextNodeResponse(
            ResponseMessage::RejectedDueOtherTransaction);
    }

    // Calculate incoming amount using inverse exchange
    // outgoingAmount = incomingAmount * rate * 10^shift
    // incomingAmount = outgoingAmount / (rate * 10^shift)
    // Use ceiling division to favor higher incoming

    try {
        incomingAmount = invertExchangeForRequiredInput(
            outgoingAmount,
            exchangeRate->exchangeRate(),
            exchangeRate->exchangeRateShift());

        debug() << "Incoming amount via exchange: "
                << "outgoing=" << outgoingAmount << " (eq " << outgoingEquiv << ")"
                << ", incoming=" << incomingAmount << " (eq " << incomingEquiv << ")"
                << ", rate=" << exchangeRate->exchangeRate()
                << ", shift=" << exchangeRate->exchangeRateShift();

    } catch (const std::exception &e) {
        error() << "Error calculating incoming amount: " << e.what();
        return sendErrorMessageOnNextNodeResponse(
            ResponseMessage::RejectedDueOtherTransaction);
    }
}
```

## Step 5: Update incoming reservation
```cpp
// Update incoming reservation
shortageIncomingReservationsOnPath(pathID, incomingEquiv, incomingAmount);

info() << "Incoming reservation adjusted: pathID=" << pathID
       << ", incomingAmount=" << incomingAmount
       << ", incomingEquiv=" << incomingEquiv;
```

## Step 6: Create helper function if needed
If `invertExchangeForRequiredInput` doesn't exist, create it (likely in the same file or as a helper):

```cpp
TrustLineAmount invertExchangeForRequiredInput(
    const TrustLineAmount &outputAmount,
    const TrustLineAmount &exchangeRate,
    int16_t exchangeRateShift)
{
    if (exchangeRate == TrustLineAmount(0)) {
        throw ValueError("Zero exchange rate");
    }

    // outputAmount = inputAmount * rate * 10^shift
    // inputAmount = outputAmount / (rate * 10^shift)

    cpp_int numerator = cpp_int(outputAmount);
    cpp_int denominator = cpp_int(exchangeRate);

    if (exchangeRateShift >= 0) {
        denominator *= pow10(static_cast<size_t>(exchangeRateShift));
    } else {
        numerator *= pow10(static_cast<size_t>(-exchangeRateShift));
    }

    // Ceiling division (favor higher incoming to ensure outgoing covered)
    return ceilDivideToAmount(numerator, denominator);
}
```

Note: Check if similar helper exists in CoordinatorExchangePaymentTransaction.cpp and reuse if possible.

# Test Plan
Tests will be implemented in a separate task (08-05). This task focuses on implementation only.

Expected test scenarios (for reference):
- Same equivalent without commission
- Same equivalent with commission
- Different equivalents with exchange rate
- Exchange rate not found (error case)
- Incoming reservation not found (error case)

# Verification and Validation

## Architecture integrity
- **Moderate Task Validation**: Verify integration with existing reservation system
- Uses existing `mCommissionsManager` and `mExchangeRatesManager` correctly
- Calls existing `shortageIncomingReservationsOnPath` (no modification needed)
- Calls existing `sendErrorMessageOnNextNodeResponse` for errors
- Helper function (if new) follows existing patterns (e.g., `ceilDivideToAmount`)

## Security
- **Moderate Task**: No security implications (internal calculation)
- Error handling prevents information leakage (generic error messages)
- No exposure of exchange rate details beyond logging

## Performance
- **Moderate Task**: Calculation overhead < 10ms
- Commission lookup: O(1) hash map access
- Exchange rate lookup: O(1) hash map access
- Ceiling division: constant time for typical amounts

## Scalability
- **Moderate Task**: Handles any valid exchange rate and commission values
- No loops or scaling issues
- Reservation search: O(n) where n = number of reservations (typically < 10)

## Reliability
- **Moderate Task**: Comprehensive error handling
- Exchange rate not found: graceful termination with error to coordinator
- Incoming reservation not found: graceful termination
- Calculation exceptions: caught and handled
- Ceiling division ensures sufficient incoming (conservative approach)

## Maintainability
- **Moderate Task**: Clear code structure with comments
- Separate paths for same/different equivalents (readable)
- Logging provides debugging information
- Helper function (if created) is reusable

## Cost
- **Moderate Task**: No additional resource usage
- Reuses existing managers

## Compliance
- **Moderate Task**: Follows PRD 08 specification
- Adheres to project coding standards
- Maintains transaction error handling patterns

# Restrictions
- Commit changes only after successfully passing the tests (tests will be created in task 08-05)
- Do not modify `shortageIncomingReservationsOnPath` or `shortageOutgoingReservationsOnPath` methods
- Do not modify `CommissionsManager` or `ExchangeRatesManager` interfaces
- Use ceiling division for exchange inversion (favor higher incoming)
