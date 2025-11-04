# 09-02 - Intermediate Node Exchange Rate and Commission Validation

# Links
- [PRD](../../../prd/vtcpd/09-exchange-rate-commission-change-handling.md)
- [Previous task: 09-01](09-01-message-protocol-extensions.md)

# Description
Implement comprehensive validation of exchange rates and commissions on intermediate nodes during exchange payment processing. This task adds context storage for conditions, validates coordinator requests against current node conditions, implements the "charge commission once per payment" logic, and validates outgoing reservations against incoming reservations after applying conditions.

When an intermediate node receives a CoordinatorReservationRequestMessage, it must verify that the coordinator's expected conditions match the node's actual conditions, reject mismatches with detailed information about actual values, and maintain consistent conditions throughout the payment lifecycle.

# Requirements and DOD

## Requirements

### R1: Transaction Context Storage
- Add `map<pair<SerializedEquivalent, SerializedEquivalent>, ExchangeRate> mContextExchangeRates` to IntermediateNodeExchangePaymentTransaction
- Add `map<SerializedEquivalent, Commission> mContextCommissions` to IntermediateNodeExchangePaymentTransaction
- Context is written only once per equivalent pair (exchange rate) or equivalent (commission)
- Context is used for all subsequent validations in the same payment

### R2: Protocol Violation Check
- In `runCoordinatorRequestProcessingStage()`, check if both expectedExchangeRate and expectedCommission are populated
- If both present: reject with Rejected status (not RejectedDueConditionsChanged)
- Log error: "Protocol violation: both exchange rate and commission provided in request"
- Do not process request further

### R3: Exchange Rate Validation
- Extract expectedExchangeRate from CoordinatorReservationRequestMessage
- Check if exchange rate already in mContextExchangeRates for this equivalent pair
- If in context: use context rate for validation
- If not in context:
  - Get current rate from ExchangeRatesManager
  - Store in mContextExchangeRates (first path only)
  - Use this rate for validation
- If current rate doesn't match expected:
  - Drop incoming reservation on this path
  - Send RejectedDueConditionsChanged with actualExchangeRate
  - Log info about mismatch
- If expected rate provided but node doesn't have this rate:
  - Drop incoming reservation
  - Send RejectedDueConditionsChanged with empty actualExchangeRate

### R4: Commission Validation
- Extract expectedCommission from CoordinatorReservationRequestMessage
- Check if commission already in mContextCommissions for this equivalent
- If in context: use context commission for validation
- If not in context:
  - Get current commission from CommissionsManager
  - Store in mContextCommissions (first path only - "charge once")
  - Use this commission for validation
- If current commission doesn't match expected:
  - Drop incoming reservation on this path
  - Send RejectedDueConditionsChanged with actualCommission
  - Log info about mismatch
- If expected commission provided but node no longer has commission:
  - Drop incoming reservation
  - Send RejectedDueConditionsChanged with actualCommission = TrustLineAmount(0)

### R5: Condition Existence Check
- If coordinator didn't send expectedExchangeRate or expectedCommission, but node has one:
  - Check if node has exchange rate for different equivalents
  - Check if node has commission for same equivalent
  - If found: send RejectedDueConditionsChanged with actual values

### R6: Outgoing/Incoming Reservation Validation
- After condition validation passes, validate outgoing reservation against incoming
- Find incoming reservation by same PathID
- If no exchange rate and no commission:
  - Outgoing amount must equal incoming amount
  - Equivalents must match
- If commission exists (same equivalent):
  - Outgoing amount must equal incoming amount minus commission
  - Equivalents must match
- If exchange rate exists (different equivalents):
  - Outgoing amount must equal incoming amount after applying exchange rate
  - Equivalents must match exchange rate pair
- If validation fails: send CoordinatorReservationResponseMessage with Rejected status

### R7: Transaction Completion Logic Update
- Implement `canCompleteTransaction()` method
- Transaction can complete only if:
  - No reservations exist AND
  - mContextExchangeRates is empty AND
  - mContextCommissions is empty
- If any of the three conditions is not met: transaction must continue

### R8: Helper Method for Rejection
- Implement `sendRejectedDueConditionsChanged(optional<ExchangeRate>, optional<TrustLineAmount>)` method
- Creates CoordinatorReservationResponseMessage with RejectedDueConditionsChanged status
- Populates actualExchangeRate and/or actualCommission if provided
- Sends to coordinator address

### R9: Drop Incoming Reservation Helper
- Implement `dropIncomingReservationOnPath(PathID)` method
- Finds incoming reservation by PathID
- Drops the reservation
- Logs the action

## Definition of Done

1. Protocol violation (both fields present) correctly detected and rejected with Rejected status

2. Exchange rate validation works for all scenarios:
   - Matching rate: validation passes, rate stored in context
   - Mismatching rate: RejectedDueConditionsChanged sent with actual rate
   - Rate in context: context value used for validation
   - Rate not found: RejectedDueConditionsChanged sent
   - Rate exists but not expected: RejectedDueConditionsChanged sent

3. Commission validation works for all scenarios:
   - Matching commission: validation passes, commission stored in context
   - Mismatching commission: RejectedDueConditionsChanged sent with actual commission
   - Commission in context: context value used (charge once logic)
   - Commission removed: RejectedDueConditionsChanged sent with commission = 0
   - Commission exists but not expected: RejectedDueConditionsChanged sent

4. Commission "charge once per payment" logic working correctly:
   - Commission charged on first path
   - Commission NOT charged on subsequent paths (already in context)

5. Outgoing/incoming reservation validation working for all cases:
   - Same equivalent without commission: amounts equal
   - Same equivalent with commission: outgoing = incoming - commission
   - Different equivalents with exchange: outgoing = incoming * rate

6. Transaction completion correctly blocked when context has data

7. Incoming reservations dropped when rejection occurs

8. Unit tests pass for all validation scenarios (Tests 0, 1a, 1-20, 29-31 from PRD)

9. Code compiles without warnings

# Implementation Plan

## Step 1: Add Context Storage Fields

**File**: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.h`

### 1.1: Add private fields
```cpp
private:
    // Existing fields...

    // Context storage for condition validation
    map<pair<SerializedEquivalent, SerializedEquivalent>, ExchangeRate> mContextExchangeRates;
    map<SerializedEquivalent, Commission> mContextCommissions;
```

## Step 2: Implement Protocol Violation Check

**File**: `src/core/transactions/transactions/regular/payments/IntermediateNodeExchangePaymentTransaction.cpp`

### 2.1: Add check at start of runCoordinatorRequestProcessingStage()
```cpp
TransactionResult::SharedConst
IntermediateNodeExchangePaymentTransaction::runCoordinatorRequestProcessingStage()
{
    // Extract message
    auto request = /* get CoordinatorReservationRequestMessage */;
    auto expectedExchangeRate = request->expectedExchangeRate();
    auto expectedCommission = request->expectedCommission();

    // Protocol violation check
    if (expectedExchangeRate.has_value() && expectedCommission.has_value()) {
        error() << "Protocol violation: both exchange rate and commission provided in request";
        return sendErrorMessageOnCoordinatorRequest(ResponseMessage::Rejected);
    }

    // Continue with validation...
}
```

## Step 3: Implement Exchange Rate Validation

### 3.1: Add validation logic after protocol check
```cpp
// Validate exchange rate if provided
if (expectedExchangeRate.has_value()) {
    auto incomingEquiv = /* get from incoming reservation */;
    auto outgoingEquiv = /* get from request */;
    auto ratePair = make_pair(incomingEquiv, outgoingEquiv);

    ExchangeRate actualRate;
    auto contextIt = mContextExchangeRates.find(ratePair);

    if (contextIt != mContextExchangeRates.end()) {
        // Use rate from context
        actualRate = contextIt->second;
    } else {
        // Get from ExchangeRatesManager
        auto rateOpt = mExchangeRatesManager->get(incomingEquiv, outgoingEquiv);

        if (!rateOpt) {
            warning() << "Exchange rate not found for pair ("
                      << incomingEquiv << ", " << outgoingEquiv << ")";
            dropIncomingReservationOnPath(pathID);
            return sendRejectedDueConditionsChanged(nullopt, nullopt);
        }

        actualRate = *rateOpt;
        mContextExchangeRates[ratePair] = actualRate;
    }

    // Compare rates
    if (actualRate != *expectedExchangeRate) {
        info() << "Exchange rate mismatch: expected " << *expectedExchangeRate
               << ", actual " << actualRate;
        dropIncomingReservationOnPath(pathID);
        return sendRejectedDueConditionsChanged(actualRate, nullopt);
    }
}
```

## Step 4: Implement Commission Validation

### 4.1: Add validation logic after exchange rate validation
```cpp
// Validate commission if provided
if (expectedCommission.has_value()) {
    auto equivalent = /* get equivalent */;

    Commission actualCommission;
    auto contextIt = mContextCommissions.find(equivalent);

    if (contextIt != mContextCommissions.end()) {
        // Use commission from context (charge once logic)
        actualCommission = contextIt->second;
    } else {
        // Get from CommissionsManager
        auto commissionOpt = mCommissionsManager->get(equivalent);

        if (!commissionOpt) {
            info() << "Commission no longer exists for equivalent " << equivalent;
            dropIncomingReservationOnPath(pathID);
            return sendRejectedDueConditionsChanged(nullopt, TrustLineAmount(0));
        }

        actualCommission = *commissionOpt;
        mContextCommissions[equivalent] = actualCommission;  // Store for future paths
    }

    // Compare commissions
    if (actualCommission.amount() != *expectedCommission) {
        info() << "Commission mismatch: expected " << *expectedCommission
               << ", actual " << actualCommission.amount();
        dropIncomingReservationOnPath(pathID);
        return sendRejectedDueConditionsChanged(nullopt, actualCommission.amount());
    }
}
```

## Step 5: Implement Condition Existence Check

### 5.1: Handle case where coordinator didn't send condition but node has one
```cpp
// Check if coordinator missed a condition
if (!expectedExchangeRate.has_value() && !expectedCommission.has_value()) {
    auto incomingEquiv = /* incoming equivalent */;
    auto outgoingEquiv = /* outgoing equivalent */;

    if (incomingEquiv != outgoingEquiv) {
        // Should have exchange rate
        auto rateOpt = mExchangeRatesManager->get(incomingEquiv, outgoingEquiv);
        if (rateOpt) {
            info() << "Exchange rate exists but not provided in request";
            dropIncomingReservationOnPath(pathID);
            return sendRejectedDueConditionsChanged(*rateOpt, nullopt);
        }
    } else {
        // Check commission
        auto commissionOpt = mCommissionsManager->get(incomingEquiv);
        if (commissionOpt) {
            info() << "Commission exists but not provided in request";
            dropIncomingReservationOnPath(pathID);
            return sendRejectedDueConditionsChanged(nullopt, commissionOpt->amount());
        }
    }
}
```

## Step 6: Implement Outgoing/Incoming Reservation Validation

### 6.1: Add validation after condition checks pass
```cpp
// Find incoming reservation by PathID
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
    return sendErrorMessageOnCoordinatorRequest(ResponseMessage::Rejected);
}

TrustLineAmount requestedOutgoingAmount = request->amount();
SerializedEquivalent outgoingEquiv = /* from request */;

bool validationPassed = false;

if (incomingEquiv == outgoingEquiv) {
    // Same equivalent - check commission
    auto commissionIt = mContextCommissions.find(incomingEquiv);

    if (commissionIt != mContextCommissions.end()) {
        TrustLineAmount expectedOutgoing =
            incomingReservation->amount() - commissionIt->second.amount();
        validationPassed = (requestedOutgoingAmount == expectedOutgoing);
    } else {
        validationPassed = (requestedOutgoingAmount == incomingReservation->amount());
    }
} else {
    // Different equivalents - check exchange rate
    auto ratePair = make_pair(incomingEquiv, outgoingEquiv);
    auto rateIt = mContextExchangeRates.find(ratePair);

    if (rateIt != mContextExchangeRates.end()) {
        TrustLineAmount expectedOutgoing = applyExchangeRate(
            incomingReservation->amount(),
            rateIt->second);
        validationPassed = (requestedOutgoingAmount == expectedOutgoing);
    } else {
        error() << "Exchange rate not found in context for validation";
        validationPassed = false;
    }
}

if (!validationPassed) {
    warning() << "Outgoing/incoming reservation validation failed";
    return sendErrorMessageOnCoordinatorRequest(ResponseMessage::Rejected);
}

// Validation passed - continue with outgoing reservation...
```

## Step 7: Update Transaction Completion Logic

### 7.1: Implement canCompleteTransaction() method
```cpp
bool IntermediateNodeExchangePaymentTransaction::canCompleteTransaction() const
{
    bool hasReservations = !mReservations.empty();
    bool hasContextRates = !mContextExchangeRates.empty();
    bool hasContextCommissions = !mContextCommissions.empty();

    bool canComplete = !hasReservations && !hasContextRates && !hasContextCommissions;

    if (!canComplete) {
        debug() << "Cannot complete transaction: "
                << "hasReservations=" << hasReservations
                << ", hasContextRates=" << hasContextRates
                << ", hasContextCommissions=" << hasContextCommissions;
    }

    return canComplete;
}
```

### 7.2: Use in transaction completion check
```cpp
// In appropriate stage method
if (canCompleteTransaction()) {
    return resultDone();
}
// else continue transaction
```

## Step 8: Implement Helper Methods

### 8.1: sendRejectedDueConditionsChanged()
```cpp
TransactionResult::SharedConst
IntermediateNodeExchangePaymentTransaction::sendRejectedDueConditionsChanged(
    optional<ExchangeRate> actualRate,
    optional<TrustLineAmount> actualCommission)
{
    auto response = make_shared<CoordinatorReservationResponseMessage>(
        currentTransactionUUID(),
        pathID,
        ResponseMessage::RejectedDueConditionsChanged,
        TrustLineAmount(0));

    if (actualRate) {
        response->setActualExchangeRate(*actualRate);
    }
    if (actualCommission) {
        response->setActualCommission(*actualCommission);
    }

    sendMessage(response, coordinatorAddress());
    return resultDone();
}
```

### 8.2: dropIncomingReservationOnPath()
```cpp
void IntermediateNodeExchangePaymentTransaction::dropIncomingReservationOnPath(
    const PathID &pathID)
{
    for (auto &[contractorID, reservationsMap] : mReservations) {
        auto it = reservationsMap.find(pathID);
        if (it != reservationsMap.end() &&
            it->second->direction() == AmountReservation::Incoming) {

            info() << "Dropping incoming reservation on path " << pathID;
            reservationsMap.erase(it);
            return;
        }
    }

    warning() << "Incoming reservation not found for path " << pathID;
}
```

# Test Plan

## Test Scope
Comprehensive validation of all condition checking logic on intermediate nodes, including context storage, protocol violations, exchange rate validation, commission validation, and reservation amount validation.

## Unit Tests

### Protocol Violation Tests
**Test 0**: Both exchange rate and commission in request
- Setup: CoordinatorReservationRequestMessage with both expectedExchangeRate and expectedCommission
- Expected: Rejected status, error log about protocol violation

### Exchange Rate Validation Tests
**Test 1**: Exchange rate matches
- Setup: Expected rate = actual rate
- Expected: Validation passes, rate stored in context

**Test 2**: Exchange rate mismatch
- Setup: Expected rate ≠ actual rate
- Expected: RejectedDueConditionsChanged, actualExchangeRate in response, incoming reservation dropped

**Test 3**: Exchange rate from context
- Setup: Second path, rate already in context
- Expected: Uses context rate for validation

**Test 4**: Exchange rate not found
- Setup: Expected rate provided but not in ExchangeRatesManager
- Expected: RejectedDueConditionsChanged with empty rate

**Test 5**: Exchange rate exists but not expected
- Setup: Node has rate but coordinator didn't send it
- Expected: RejectedDueConditionsChanged with actual rate

### Commission Validation Tests
**Test 6**: Commission matches
- Setup: Expected commission = actual commission
- Expected: Validation passes, commission stored in context

**Test 7**: Commission mismatch
- Setup: Expected commission ≠ actual commission
- Expected: RejectedDueConditionsChanged, actualCommission in response, incoming reservation dropped

**Test 8**: Commission from context (second path)
- Setup: Second path, commission already in context
- Expected: Uses context commission, does NOT charge again

**Test 9**: Commission removed
- Setup: Expected commission but node no longer has it
- Expected: RejectedDueConditionsChanged with commission = 0

**Test 10**: Commission exists but not expected
- Setup: Node has commission but coordinator didn't send it
- Expected: RejectedDueConditionsChanged with actual commission

### Context Storage Tests
**Test 11**: Exchange rate stored on first path
- Setup: First CoordinatorReservationRequestMessage with rate
- Expected: Rate stored in mContextExchangeRates

**Test 12**: Commission stored on first path
- Setup: First CoordinatorReservationRequestMessage with commission
- Expected: Commission stored in mContextCommissions

**Test 13**: Context not overwritten on subsequent paths
- Setup: Second path with different rate/commission
- Expected: Context unchanged, uses original values

**Test 14**: Transaction completion blocked by context
- Setup: No reservations but context has rates/commissions
- Expected: canCompleteTransaction() returns false

**Test 15**: Transaction completes when all clear
- Setup: No reservations, no context rates, no context commissions
- Expected: canCompleteTransaction() returns true

### Outgoing/Incoming Validation Tests
**Test 16**: Same equivalent without commission
- Setup: Incoming = outgoing = 100, no commission
- Expected: Validation passes

**Test 17**: Same equivalent with commission
- Setup: Incoming = 110, outgoing = 100, commission = 10
- Expected: Validation passes

**Test 18**: Same equivalent with commission mismatch
- Setup: Incoming = 110, outgoing = 105, commission = 10
- Expected: Validation fails, Rejected sent

**Test 19**: Different equivalents with exchange
- Setup: Incoming = 100 (eq 1), outgoing = 200 (eq 2), rate = 2.0
- Expected: Validation passes

**Test 20**: Different equivalents with exchange mismatch
- Setup: Incoming = 100 (eq 1), outgoing = 201 (eq 2), rate = 2.0
- Expected: Validation fails, Rejected sent

### Commission "Charge Once" Tests
**Test 29**: Commission charged on first path
- Setup: Node participates in first path with commission
- Expected: Commission stored in context, deducted from amount

**Test 30**: Commission NOT charged on second path
- Setup: Same node in second path, commission already in context
- Expected: Commission not deducted again

**Test 31**: Different equivalent commissions charged separately
- Setup: Node charges commission in eq 1 and eq 2 in different paths
- Expected: Both commissions charged (different equivalents)

## Success Criteria
- All 25 unit tests pass (Tests 0, 1a, 1-20, 29-31)
- 100% coverage of validation logic
- No memory leaks in context storage
- No compilation warnings

# Verification and Validation

## Architecture integrity
**Validation Level**: Complex task
- Context storage properly encapsulated in transaction class
- Validation logic follows existing transaction processing patterns
- Uses existing managers (ExchangeRatesManager, CommissionsManager) correctly
- Message sending follows existing patterns
- No circular dependencies introduced

## Security
**Validation Level**: Complex task
- Incoming reservations properly dropped on rejection
- No data leaks through response messages (only actual conditions sent)
- Context cleared on transaction completion
- No buffer overflows in map storage
- Proper bounds checking on all amounts

## Performance
**Validation Level**: Complex task
- Context lookups: O(log n) for map access
- Validation overhead: < 5ms per reservation request
- Memory usage: bounded by number of equivalents/pairs in payment
- No performance regression in happy path (conditions match)

## Scalability
**Validation Level**: Complex task
- Context size scales linearly with number of unique equivalent pairs
- Supports up to 5 exchange equivalents (PRD 06 limit)
- Map storage efficient for expected payment sizes

## Reliability
**Validation Level**: Complex task
- Validation failures don't crash transaction
- Missing data handled gracefully (returns appropriate rejection)
- Context consistency maintained throughout payment lifecycle
- Transaction completion logic prevents premature termination

## Maintainability
**Validation Level**: Complex task
- Clear separation of validation concerns (rate, commission, amounts)
- Helper methods improve code clarity
- Logging at appropriate levels (info for mismatches, error for failures)
- Well-commented validation logic

## Cost
**Validation Level**: Complex task
- Minimal additional memory (maps bounded by equivalents)
- No infrastructure cost changes

## Compliance
**Validation Level**: Complex task
- Follows project policy for transaction validation
- Adheres to "charge commission once" requirement from PRD 06
- Implements all acceptance criteria from PRD

# Restrictions
- Commit changes only after successfully passing all unit tests
- Do not modify coordinator transaction logic (Task 09-03)
- Ensure all validation errors are logged appropriately
- Maintain transaction state consistency
