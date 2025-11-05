# Project Requirements Document (PRD)

## Document Information
- **Project Name**: Allowable Payment Amount Control for Exchange Payments
- **PRD ID**: 10
- **Phase/Iteration**: Phase 1, Initial Implementation
- **Document Version**: 1.0
- **Date**: 2025-11-04
- **Author(s)**: Claude Code, based on Architect's requirements
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **PRD Status**: 1.1 - PRD file created
- **Last Status Update**: 2025-11-04
- **Previous PRD**: [09-exchange-rate-commission-change-handling.md](09-exchange-rate-commission-change-handling.md)
- **Related Documents**:
  - [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
  - [Exchange Payment Topology Collection PRD](07-exchange-payment-topology-collection.md)
  - [Path Capacity Adjustment PRD](08-path-capacity-adjustment.md)
  - [Exchange Rate and Commission Change Handling PRD](09-exchange-rate-commission-change-handling.md)
  - [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)

> **Workflow Reference**: For complete phase descriptions and status transitions, see **"Feature Development Workflow"** section in [policy.md](../../../policy.md). This document follows the 9-phase workflow with numbered steps (1.1 through 9.3) for precise status tracking.

## Executive Summary
This PRD introduces controllability and predictability to exchange payment execution by adding a maximum allowable payment amount parameter to `CreditUsageExchangeCommand`. This parameter allows users to specify the maximum amount they are willing to pay (in the payment equivalent) to deliver the desired amount to the receiver (in the receiver equivalent). The system validates this constraint both during initial path evaluation and after each path reservation, ensuring that condition changes during payment execution don't result in unexpectedly high payment amounts.

### Current project state
- Exchange Payment with Commissions (PRD 06) implements multi-equivalent payment execution
- Exchange Payment Topology Collection (PRD 07) adds automatic path availability checking
- Path Capacity Adjustment (PRD 08) handles dynamic capacity changes during reservation
- Exchange Rate and Commission Change Handling (PRD 09) adapts to condition changes during payment
- Current implementation: users see estimated payment amount via `EstimatePaymentForReceiveAmountTransaction` but have no control over actual payment amount if conditions change

### This iteration's focus
- Add optional `maxAllowablePaymentAmount` parameter to `CreditUsageExchangeCommand`
- Implement validation in `runPathsResourceProcessingStage()` before starting reservations
- Implement validation after each path reservation completion (in `processRemoteNodeResponse` and `processNeighborFurtherReservationResponse`)
- Calculate total reserved payment amount on coordinator side
- Add new result code 415 "Allowable payment amount has been exceeded"
- Ensure all reservations are dropped when constraint violated after reservation
- Make validation optional (skip if parameter not provided)

### Connection to overall vision
This completes the user control layer for exchange payments, enabling predictable payment costs even when network conditions (exchange rates, commissions, capacities) change during payment execution. Users can now set spending limits and be confident their payments won't exceed specified amounts.

## Iteration Context
### Previous Iterations Summary
- **PRD 04**: Exchange Flow Calculation implements topology collection and OR-Tools-based max flow computation
- **PRD 05**: Payment Estimation provides bidirectional estimation using cached paths
- **PRD 06**: Exchange Payment with Commissions implements multi-equivalent payment execution
- **PRD 07**: Exchange Payment Topology Collection adds automatic path collection and moves `mExchangeAmount` calculation to path processing stage
- **PRD 08**: Path Capacity Adjustment handles dynamic capacity changes during reservation
- **PRD 09**: Exchange Rate and Commission Change Handling adapts to condition changes during payment
- **Completed Features**: Exchange rate management, optimal path calculation, automatic topology collection, capacity adjustment, condition change handling

### Lessons Learned
- Conditions (exchange rates, commissions, capacities) can change between estimation and execution
- Users need predictability and control over payment costs
- Validation must happen at multiple stages due to dynamic condition changes
- Coordinator must track total reserved amounts across all paths

### Current State Analysis
- **What's working well**: Exchange payments execute successfully with dynamic adaptation to condition changes
- **Pain points identified**:
  - Users have no control over maximum payment amount
  - Estimated payment amount may differ significantly from actual due to condition changes
  - No mechanism to abort payment if cost exceeds acceptable threshold
  - Users cannot set spending limits for exchange payments
- **Performance metrics**: Exchange payments succeed but may result in unexpected costs

## Problem Statement
### Background
When users initiate exchange payments, they see an estimated payment amount from `EstimatePaymentForReceiveAmountTransaction`. However, during payment execution, conditions may change:

1. **Exchange rates change**: Rates become less favorable, requiring more sender currency
2. **Commissions increase**: Intermediate nodes increase fees
3. **Capacity constraints**: Optimal paths unavailable, forcing use of less efficient paths
4. **Path recalculation**: PRD 09 recalculates paths when conditions change, potentially increasing cost

Without a control mechanism, users have no way to limit the payment amount they're willing to spend. This creates unpredictability and prevents users from managing their spending.

### Problem Description
**Who is affected**: Users initiating exchange payments who want cost predictability

**When and where**:
- During payment initialization: when user wants to specify maximum acceptable payment amount
- During path processing: when system needs to validate payment cost before starting reservations
- During reservation: when conditions change and reserved amount may exceed user's limit

**Current limitations**:
- `CreditUsageExchangeCommand` doesn't accept maximum allowable payment amount parameter
- No validation of payment amount against user's limit
- No mechanism to abort payment when cost exceeds limit
- Coordinator doesn't track total reserved payment amount across paths

### Impact of not solving this problem
- Users cannot control spending on exchange payments
- Unpredictable costs reduce user trust in exchange payment feature
- Users may avoid exchange payments due to cost uncertainty
- No protection against extreme condition changes causing excessive costs

### Success Metrics
**Primary KPIs**:
- Successful payment abortion when constraint violated (target: 100% detection)
- Correct total reserved amount calculation (target: 100% accuracy)
- Proper reservation cleanup when aborting (target: 100% cleanup)
- User spending limits respected (target: 0 violations)

**Target Values**:
- 100% validation accuracy in both `runPathsResourceProcessingStage` and after reservation
- 100% correct total reserved amount calculation
- 0 payments executing beyond user's specified limit
- 100% proper reservation cleanup on constraint violation

## Goals
The primary goals of this iteration are to enable user control over maximum payment amount in exchange payments.

*   **Goal 1: Enable user specification of maximum allowable payment amount.**
    *   **Description:** Add optional `maxAllowablePaymentAmount` parameter to `CreditUsageExchangeCommand`, allowing users to specify spending limit in payment equivalent.
    *   **Success Metric:** Users can successfully specify maximum payment amount; parameter correctly parsed and stored.

*   **Goal 2: Validate payment amount before starting reservations.**
    *   **Description:** In `runPathsResourceProcessingStage()`, validate that calculated `mExchangeAmount` doesn't exceed `maxAllowablePaymentAmount` before initiating reservations.
    *   **Success Metric:** Payments correctly abort with code 415 when estimated cost exceeds limit before any reservations made.

*   **Goal 3: Validate total reserved amount after each path reservation.**
    *   **Description:** After completing each path reservation, calculate total reserved payment amount and validate against `maxAllowablePaymentAmount`, aborting with cleanup if exceeded.
    *   **Success Metric:** Payments correctly abort with code 415 when reserved amount exceeds limit; all reservations properly cleaned up.

*   **Goal 4: Ensure correct total reserved amount calculation.**
    *   **Description:** Calculate total reserved payment amount on coordinator side by summing reserved amounts across all paths in payment equivalent.
    *   **Success Metric:** Total reserved amount calculation 100% accurate across all test scenarios.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
1. **Optional maxAllowablePaymentAmount parameter in CreditUsageExchangeCommand**: Add parsing, validation, and storage
2. **Validation in runPathsResourceProcessingStage()**: Check `mExchangeAmount` against limit before reservations
3. **Validation after path reservation**: Check total reserved amount in `processRemoteNodeResponse` and `processNeighborFurtherReservationResponse`
4. **Total reserved amount calculation**: Method to calculate total reserved payment amount on coordinator
5. **New result code 415**: "Allowable payment amount has been exceeded"
6. **Reservation cleanup on violation**: Drop all reservations when constraint violated after reservation

#### Technical Infrastructure
- Extended `CreditUsageExchangeCommand` with optional parameter
- New validation logic in coordinator transaction
- Total reserved amount calculation method
- New result method `resultAllowablePaymentAmountExceeded()`
- Reservation cleanup integration with existing drop logic

#### Integration Points
- Integration with command parsing infrastructure
- Integration with existing validation stages
- Integration with reservation management
- Integration with result code system

### Explicitly Out of Scope
- Automatic adjustment of `maxAllowablePaymentAmount` based on market conditions
- Dynamic limit calculation based on historical data
- UI/API for limit recommendation to users
- Multi-payment budget management
- Limit enforcement at intermediate nodes (coordinator-only control)

### Dependencies from Previous Iterations
- **PRD 06**: `CoordinatorExchangePaymentTransaction` implementation, `mExchangeAmount` calculation
- **PRD 07**: `mExchangeAmount` calculation in `runPathsResourceProcessingStage()`
- **PRD 08**: Reservation drop mechanisms
- **PRD 09**: Condition change handling and path recalculation

### Future Roadmap Impact
This iteration establishes foundation for:
- **Smart limit suggestions**: ML-based recommendation of appropriate limits
- **Budget management**: Multi-payment spending limits and tracking
- **Limit presets**: User-defined default limits for different scenarios
- **Limit alerts**: Notifications when approaching limits during estimation

## User Stories & Requirements

### User Personas
#### Primary User: Cautious Payment Initiator
- **Role**: User wanting cost control for exchange payments
- **Goals**: Ensure payment cost doesn't exceed acceptable amount; protect against unexpected cost increases
- **Pain Points**: Cannot control maximum payment amount; surprised by high costs when conditions change
- **Technical Proficiency**: Intermediate

#### Secondary User: Budget-Conscious User
- **Role**: User managing spending across multiple payments
- **Goals**: Stay within budget; predictable payment costs
- **Pain Points**: Unpredictable exchange payment costs; no spending limit mechanism
- **Technical Proficiency**: Intermediate

### Functional Requirements
#### New Features for This Iteration

1. **Optional maxAllowablePaymentAmount Parameter in CreditUsageExchangeCommand**
   - **Description**: Add optional parameter allowing users to specify maximum payment amount in payment equivalent
   - **User Story**: As a user, I want to specify the maximum amount I'm willing to pay so that my payment costs are predictable and controlled
   - **Rationale**: Provides user control over payment spending; prevents unexpected high costs
   - **Builds Upon**: Existing `CreditUsageExchangeCommand` structure
   - **Acceptance Criteria**:
     - Add `optional<TrustLineAmount> mMaxAllowablePaymentAmount` field to `CreditUsageExchangeCommand`
     - Add getter method: `const optional<TrustLineAmount>& maxAllowablePaymentAmount() const`
     - Extend command parsing to accept optional parameter after `exchangeEquivalents`
     - Command format: `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1:1500` (1500 = max allowable amount, optional)
     - Command format without limit: `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1` (existing format)
     - Validation: if provided, must be positive TrustLineAmount
     - Parameter stored in coordinator transaction via command
     - If parameter not provided, all validation checks skipped
     - Located in: `src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.h/.cpp`
   - **Priority**: High
   - **Dependencies**: None (extension of existing command)

2. **Validation in runPathsResourceProcessingStage()**
   - **Description**: Validate calculated `mExchangeAmount` against `maxAllowablePaymentAmount` before starting reservations
   - **User Story**: As a coordinator, I need to abort payment before making any reservations if estimated cost exceeds user's limit
   - **Rationale**: Early validation prevents wasted reservation attempts; fails fast
   - **Builds Upon**: Existing `runPathsResourceProcessingStage()` implementation (PRD 07)
   - **Acceptance Criteria**:
     - After calculating `mExchangeAmount` in `runPathsResourceProcessingStage()` (before "Step 0.5: Check kTotalOutgoingPossibilities")
     - Add validation check:
       ```cpp
       // Step 0.4: Check maxAllowablePaymentAmount (if provided)
       if (mCommand->maxAllowablePaymentAmount().has_value()) {
           if (mExchangeAmount > *mCommand->maxAllowablePaymentAmount()) {
               warning() << "Calculated exchange amount " << mExchangeAmount
                         << " exceeds maximum allowable payment amount "
                         << *mCommand->maxAllowablePaymentAmount();
               return resultAllowablePaymentAmountExceeded();
           }
       }
       ```
     - If validation fails: return `resultAllowablePaymentAmountExceeded()` immediately
     - No reservations have been made at this point, so no cleanup needed
     - Validation skipped if `maxAllowablePaymentAmount` not provided
     - Log warning with both amounts when limit exceeded
   - **Priority**: High
   - **Dependencies**: `mExchangeAmount` calculation in `runPathsResourceProcessingStage()` (PRD 07)

3. **Total Reserved Payment Amount Calculation Method**
   - **Description**: Calculate total amount reserved for payment across all paths in payment equivalent
   - **User Story**: As a coordinator, I need to know the total amount reserved for payment to validate against user's limit
   - **Rationale**: Accurate tracking of reserved amounts essential for limit enforcement
   - **Builds Upon**: Existing reservation tracking in coordinator transaction
   - **Acceptance Criteria**:
     - Add method to `CoordinatorExchangePaymentTransaction`:
       ```cpp
       TrustLineAmount calculateTotalReservedPaymentAmount() const;
       ```
     - Метод ітерується по значеннях `exchangeEquivalents()` з `mCommand`
     - Для кожного еквівалента викликається існуючий API `totalReservedAmount(AmountReservation::Outgoing, equivalent)`
     - Отримані значення (в еквіваленті платника) сумуються в загальну величину без додаткових перерахунків
     - **Важливо**: не обходити вручну `mPathsStats` та не виконувати конвертацій; використовувати агреговані дані резервів
     - Метод викликається після завершення кожного резервування шляху
     - За відсутності зарезервованих сум метод повертає `TrustLineAmount(0)`
     - Located in: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h/.cpp`
   - **Priority**: High
   - **Dependencies**: Existing reservation tracking

4. **Validation After Path Reservation in processRemoteNodeResponse**
   - **Description**: Validate total reserved amount after remote node response indicates path completion
   - **User Story**: As a coordinator, I need to abort payment if total reserved amount exceeds user's limit after processing a path
   - **Rationale**: Conditions may change during reservation, causing reserved amount to exceed limit
   - **Builds Upon**: Existing `processRemoteNodeResponse` implementation
   - **Acceptance Criteria**:
     - In `processRemoteNodeResponse()`, find condition: `if (path->isLastIntermediateNodeProcessed())`
     - After existing processing in this condition, add validation:
       ```cpp
       // Check maxAllowablePaymentAmount after path completion
       if (mCommand->maxAllowablePaymentAmount().has_value()) {
           TrustLineAmount totalReserved = calculateTotalReservedPaymentAmount();
           if (totalReserved > *mCommand->maxAllowablePaymentAmount()) {
               warning() << "Total reserved payment amount " << totalReserved
                         << " exceeds maximum allowable payment amount "
                         << *mCommand->maxAllowablePaymentAmount()
                         << " after processing path " << pathID;

               reject("Allowable payment amount exceeded");
               return resultAllowablePaymentAmountExceeded();
           }
       }
       ```
     - Validation happens even if target `mAmount` for receiver achieved
     - If validation fails:
       - Log warning with both amounts and pathID
       - Call `reject("Allowable payment amount exceeded")` without return - triggers `rollBack()` in base class
       - Return `resultAllowablePaymentAmountExceeded()` with code 415
     - Validation skipped if `maxAllowablePaymentAmount` not provided
     - Transaction terminates with proper cleanup via existing `rollBack()` mechanism
   - **Priority**: High
   - **Dependencies**: `calculateTotalReservedPaymentAmount()` method, existing `reject()` and `rollBack()` infrastructure

5. **Validation After Path Reservation in processNeighborFurtherReservationResponse**
   - **Description**: Validate total reserved amount after neighbor response indicates path completion
   - **User Story**: As a coordinator, I need to abort payment if total reserved amount exceeds user's limit after processing a path via neighbor
   - **Rationale**: Same validation needed for paths going through neighbor node
   - **Builds Upon**: Existing `processNeighborFurtherReservationResponse` implementation
   - **Acceptance Criteria**:
     - In `processNeighborFurtherReservationResponse()`, find condition: `if (path->isLastIntermediateNodeProcessed())`
     - Add identical validation as in `processRemoteNodeResponse`:
       ```cpp
       // Check maxAllowablePaymentAmount after path completion
       if (mCommand->maxAllowablePaymentAmount().has_value()) {
           TrustLineAmount totalReserved = calculateTotalReservedPaymentAmount();
           if (totalReserved > *mCommand->maxAllowablePaymentAmount()) {
               warning() << "Total reserved payment amount " << totalReserved
                         << " exceeds maximum allowable payment amount "
                         << *mCommand->maxAllowablePaymentAmount()
                         << " after processing path " << pathID << " via neighbor";

               reject("Allowable payment amount exceeded");
               return resultAllowablePaymentAmountExceeded();
           }
       }
       ```
     - Same behavior as in `processRemoteNodeResponse`
     - Transaction terminates with proper cleanup via existing `rollBack()` mechanism
   - **Priority**: High
   - **Dependencies**: `calculateTotalReservedPaymentAmount()` method, existing `reject()` and `rollBack()` infrastructure

6. **New Result Code 415 - Allowable Payment Amount Exceeded**
   - **Description**: New error code indicating payment amount exceeded user's specified limit
   - **User Story**: As a user, I need clear error message when payment fails due to exceeding my specified limit
   - **Rationale**: Distinguishes limit violation from other payment failures
   - **Builds Upon**: Existing result code system and existing pattern (like line 2590-2591 in CoordinatorExchangePaymentTransaction.cpp)
   - **Acceptance Criteria**:
     - Add method to `CoordinatorExchangePaymentTransaction`:
       ```cpp
       TransactionResult::SharedConst resultAllowablePaymentAmountExceeded();
       ```
     - Method implementation:
       ```cpp
       TransactionResult::SharedConst
       CoordinatorExchangePaymentTransaction::resultAllowablePaymentAmountExceeded()
       {
           return transactionResultFromCommand(
               mCommand->responseAllowablePaymentAmountExceeded());
       }
       ```
     - Add method to `CreditUsageExchangeCommand`:
       ```cpp
       CommandResult::SharedConst responseAllowablePaymentAmountExceeded() const;
       ```
     - Method implementation:
       ```cpp
       CommandResult::SharedConst
       CreditUsageExchangeCommand::responseAllowablePaymentAmountExceeded() const
       {
           return CommandResult::SharedConst(
               new CommandResult(
                   identifier(),
                   UUID(),
                   415,
                   "Allowable payment amount has been exceeded"));
       }
       ```
     - Usage pattern (following existing pattern at line 2590-2591):
       ```cpp
       // When limit exceeded:
       reject("Allowable payment amount exceeded");
       return resultAllowablePaymentAmountExceeded();
       ```
     - Result code 415 specifically for this error condition
     - Clear message for users
     - Reuses existing `rollBack()` mechanism from base class
     - No need to override `reject()` method
     - No need for member variables to track rejection reason
     - Located in: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h/.cpp` and `src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.h/.cpp`
   - **Priority**: High
   - **Dependencies**: None (uses existing pattern)

#### Enhancements to Existing Features

1. **CreditUsageExchangeCommand Command Format**
   - **Current State**: `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1` (without limit)
   - **Proposed Changes**: Add optional `maxAllowablePaymentAmount` parameter after `exchangeEquivalents`
   - **New Format**: `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1:1500` (1500 = max allowable amount)
   - **Impact Assessment**: Backward compatible (parameter optional)
   - **Migration Strategy**: Existing commands without parameter continue working

2. **CoordinatorExchangePaymentTransaction Result Methods**
   - **Current State**: Has result methods for various error conditions
   - **Proposed Changes**: Add `resultAllowablePaymentAmountExceeded()`
   - **Impact Assessment**: Pure addition, no impact on existing methods
   - **Migration Strategy**: Direct addition

### Non-Functional Requirements
#### Performance
- Validation overhead < 5ms per check
- Total reserved amount calculation < 10ms
- No impact on reservation performance
- No additional memory overhead (parameter stored in command)

#### Security
- No security implications (internal validation)
- Parameter validated at parsing time (positive amount)
- No exposure of internal state to external parties

#### Scalability
- Support validation for up to 50 paths per payment
- Efficient total amount calculation (O(n) where n = path count)
- No scalability concerns

#### Reliability
- Validation failures don't crash transaction (clean termination)
- All reservations properly cleaned up on validation failure
- Transaction state consistent after abort
- Proper error reporting to user

## Technical Specifications
### Architecture Evolution
- **Current Architecture**: Exchange payments execute without user-specified limit; cost unpredictable when conditions change
- **Proposed Changes**: Add optional limit parameter, validation at multiple stages, proper cleanup on violation
- **Backwards Compatibility**: Fully backward compatible (parameter optional)
- **Migration Requirements**: None (new optional feature)

### Technology Stack Updates
#### New Technologies/Libraries
- No new external libraries required
- Reuses existing infrastructure: command parsing, validation, reservation management

#### Version Updates
- No version updates required

### Integration Requirements
#### New Integrations
- Command parsing extended with optional parameter
- Validation integrated into existing transaction stages
- Cleanup integrated with existing reservation drop logic

#### Modified Integrations
- `CreditUsageExchangeCommand` parsing extended
- `runPathsResourceProcessingStage()` validation extended
- `processRemoteNodeResponse` validation extended
- `processNeighborFurtherReservationResponse` validation extended

### Data Requirements
#### Data Models

No new data models. Enhanced structures:

##### Enhanced CreditUsageExchangeCommand
**Purpose**: Add optional maximum allowable payment amount parameter

**Changes**:
```cpp
class CreditUsageExchangeCommand : public BaseUserCommand {
    // Existing fields...

    // NEW field:
    optional<TrustLineAmount> mMaxAllowablePaymentAmount;

    // NEW getter:
    const optional<TrustLineAmount>& maxAllowablePaymentAmount() const;

    // NEW response method:
    CommandResult::SharedConst responseAllowablePaymentAmountExceeded() const;
};
```

#### Data Storage
- No persistent storage changes
- Parameter stored in command object (runtime only)
- No transaction state serialization changes needed

#### Data Migration
- No migration needed (optional parameter)

### Algorithm Specifications

#### Total Reserved Payment Amount Calculation

**Purpose**: Calculate total amount reserved by coordinator for payment across all paths

**Algorithm**:
```cpp
TrustLineAmount CoordinatorExchangePaymentTransaction::calculateTotalReservedPaymentAmount() const
{
    TrustLineAmount totalReserved = TrustLineAmount(0);

    for (const auto equivalent : mCommand->exchangeEquivalents()) {
        const auto reservedForEquivalent = totalReservedAmount(
            AmountReservation::Outgoing,
            equivalent);

        totalReserved = totalReserved + reservedForEquivalent;
    }

    return totalReserved;
}
```

**Key Points**:
- Пере використовує існуючі агрегати резервів через `totalReservedAmount`
- Достатньо обійти список еквівалентів, у яких координатор платить
- Повертає суму у «платіжному» еквіваленті, готову до порівняння з лімітом
- Алгоритм не залежить від структури `mPathsStats` і не виконує ручних конвертацій

#### Validation in runPathsResourceProcessingStage()

**Purpose**: Validate estimated payment amount before starting reservations

**Algorithm Location**: `runPathsResourceProcessingStage()` in `CoordinatorExchangePaymentTransaction`

**Integration Point**: After Step 0 (mExchangeAmount calculation) and before Step 0.5 (kTotalOutgoingPossibilities check)

**Algorithm**:
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::runPathsResourceProcessingStage()
{
    // Step 0: Calculate mExchangeAmount (existing code from PRD 07)
    try {
        TrustLineAmount remainingReceive = mAmount;
        TrustLineAmount totalPayment = TrustLineAmount(0);

        // ... [existing calculation logic] ...

        mExchangeAmount = totalPayment;
        info() << "Calculated exchange amount: " << mExchangeAmount;

    } catch (const exception &e) {
        error() << "Error calculating exchange amount: " << e.what();
        return resultProtocolError();
    }

    // Step 0.4: Check maxAllowablePaymentAmount (if provided) - NEW
    if (mCommand->maxAllowablePaymentAmount().has_value()) {
        if (mExchangeAmount > *mCommand->maxAllowablePaymentAmount()) {
            warning() << "Calculated exchange amount " << mExchangeAmount
                      << " exceeds maximum allowable payment amount "
                      << *mCommand->maxAllowablePaymentAmount();
            return resultAllowablePaymentAmountExceeded();
        }

        info() << "Exchange amount " << mExchangeAmount
               << " within allowable limit " << *mCommand->maxAllowablePaymentAmount();
    }

    // Step 0.5: Check kTotalOutgoingPossibilities (existing code)
    TrustLineAmount totalOutgoingAmount = TrustLineAmount(0);
    // ... [existing code] ...

    // Step 1: Initialize total flow counter (existing code)
    // ... [rest of method] ...
}
```

**Key Points**:
- Validation only if `maxAllowablePaymentAmount` provided
- Happens after `mExchangeAmount` calculated
- Happens before any reservations made (no cleanup needed)
- Returns `resultAllowablePaymentAmountExceeded()` immediately if exceeded
- Logs both amounts for debugging

#### Validation After Path Reservation

**Purpose**: Validate total reserved amount after each path completion

**Algorithm Location 1**: `processRemoteNodeResponse()` in `CoordinatorExchangePaymentTransaction`

**Integration Point**: Inside `if (path->isLastIntermediateNodeProcessed())` condition, after existing processing

**Algorithm**:
```cpp
TransactionResult::SharedConst
CoordinatorExchangePaymentTransaction::processRemoteNodeResponse(
    /* parameters */)
{
    // ... [existing code for finding path, processing response] ...

    if (path->isLastIntermediateNodeProcessed()) {
        // ... [existing code for updating reserved amount, checking if done] ...

        // NEW: Check maxAllowablePaymentAmount after path completion
        if (mCommand->maxAllowablePaymentAmount().has_value()) {
            TrustLineAmount totalReserved = calculateTotalReservedPaymentAmount();

            if (totalReserved > *mCommand->maxAllowablePaymentAmount()) {
                warning() << "Total reserved payment amount " << totalReserved
                          << " exceeds maximum allowable payment amount "
                          << *mCommand->maxAllowablePaymentAmount()
                          << " after processing path " << pathID;

                // Call reject() without return - triggers rollBack() in base class
                reject("Allowable payment amount exceeded");

                // Return specific error code
                return resultAllowablePaymentAmountExceeded();
            }

            debug() << "Total reserved amount " << totalReserved
                    << " within allowable limit " << *mCommand->maxAllowablePaymentAmount();
        }

        // ... [existing code continues] ...
    }

    // ... [rest of method] ...
}
```

**Algorithm Location 2**: `processNeighborFurtherReservationResponse()` in `CoordinatorExchangePaymentTransaction`

**Integration Point**: Inside `if (path->isLastIntermediateNodeProcessed())` condition, after existing processing

**Algorithm**: Identical to `processRemoteNodeResponse`, only log message differs (adds "via neighbor")

**Key Points**:
- Validation only if `maxAllowablePaymentAmount` provided
- Happens after path processing complete (`isLastIntermediateNodeProcessed()`)
- Calculates total reserved amount across all processed paths
- Calls `reject("Allowable payment amount exceeded")` without return - triggers `rollBack()` in base class
- Returns `resultAllowablePaymentAmountExceeded()` with code 415
- Pattern follows existing code at line 2590-2591 in CoordinatorExchangePaymentTransaction.cpp
- Validation happens even if target `mAmount` already achieved
- Ensures user's limit always respected
- Reuses existing infrastructure (no override of `reject()` needed, no flags needed)

### Error Handling Specifications

#### Error Conditions

1. **Limit exceeded in runPathsResourceProcessingStage() (before reservations)**:
   - Log: `warning() << "Calculated exchange amount X exceeds maximum allowable payment amount Y"`
   - Action: Return `resultAllowablePaymentAmountExceeded()` immediately
   - Cleanup: None needed (no reservations yet)
   - Result: Command result 415 returned to user

2. **Limit exceeded after path reservation**:
   - Log: `warning() << "Total reserved payment amount X exceeds maximum allowable payment amount Y after processing path Z"`
   - Action: Call `reject("Allowable payment amount exceeded")` without return, then return `resultAllowablePaymentAmountExceeded()`
   - Cleanup: All reservations dropped automatically via `rollBack()` called by `BaseExchangePaymentTransaction::reject()`
   - Result: Command result 415 returned to user

3. **Calculation error in calculateTotalReservedPaymentAmount()**:
   - Should not throw exceptions
   - Uses existing `totalReservedAmount()` API which handles errors internally
   - Returns 0 if no reservations found
   - Worst case: returns lower amount than actual (safe - allows payment to continue)

4. **Cleanup error in rollBack()**:
   - Handled by existing `BaseExchangePaymentTransaction::rollBack()` implementation
   - Existing error handling already in place
   - Transaction terminates with result 415

5. **Invalid maxAllowablePaymentAmount parameter**:
   - Validation during command parsing
   - Must be positive TrustLineAmount
   - Parsing error returns command error to user
   - Transaction never created

## Implementation Plan
### This Iteration Timeline
- **Duration**: 1-2 weeks implementation + 1 week testing
- **Sprint Breakdown**:
  - Sprint 1 (Week 1): Command parameter extension and parsing
  - Sprint 2 (Week 1): Total reserved amount calculation method
  - Sprint 3 (Week 1-2): Validation in `runPathsResourceProcessingStage()`
  - Sprint 4 (Week 2): Validation after path reservation
  - Sprint 5 (Week 2): Reservation cleanup method and integration
  - Sprint 6 (Week 2): Unit testing

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Command Extension Complete | Week 1 | Parameter added, parsing working | None | Low |
| Calculation Method Complete | Week 1 | Total reserved amount calculation working | None | Low |
| Early Validation Complete | Week 1 | Validation in runPathsResourceProcessingStage() working | Command extension | Low |
| Late Validation Complete | Week 2 | Validation after reservation working | Calculation method | Medium |
| Cleanup Integration Complete | Week 2 | Reservation cleanup working | All previous | Low |
| Unit Testing Complete | Week 2 | All components unit tested | All previous | Low |

### Dependencies on Other Teams/Projects
- No external team dependencies identified

### Integration Points with Previous Work
- Builds upon `CreditUsageExchangeCommand` structure (PRD 06)
- Uses `mExchangeAmount` calculation from `runPathsResourceProcessingStage()` (PRD 07)
- Uses reservation tracking from coordinator transaction (PRD 06)
- Uses reservation drop mechanisms (PRD 08)

### Resource Requirements
#### Team Structure
- **Technical Lead**: 1 developer with C++ and payment transaction experience
- **Developers**: 0-1 developer for implementation support
- **QA Engineers**: 1 engineer for unit testing

## Risk Management
### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Incorrect total reserved amount calculation | High | Low | Thorough unit testing with diverse scenarios; validate against manual calculations |
| Reservation cleanup failures | Medium | Low | Best-effort cleanup with error handling; log all failures |
| Validation logic errors | Medium | Low | Clear validation logic; extensive unit testing |
| Performance impact | Low | Low | Efficient calculation (O(n)); measure in tests |

### Business Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Users set limits too low | Low | Medium | Clear error message explaining limit exceeded; suggest running estimation first |

## Testing Strategy
### Testing Approach (Unit-Only)
- All testing is unit-only (no integration/E2E tests)
- Tests are built and executed exclusively in `build-tests`
- Use real objects following existing test patterns
- Test files located in `tests/unit/` subdirectories

### Testing Best Practices
- **Real Objects Over Mocks**: Use real instances where possible
- **Exception Testing**: Test error paths using EXPECT_THROW
- **Edge Case Coverage**: Test boundary values, missing parameter scenarios
- **Parameterized Tests**: Use for testing different limit values

#### Unit Tests: New/Modified Components

**1. CreditUsageExchangeCommand Tests**

*Test 1: Parse command with maxAllowablePaymentAmount parameter*
- **Setup**: Command string `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1:1500`
- **Expected**:
  - `maxAllowablePaymentAmount()` returns `optional<TrustLineAmount>(1500)`
  - All other parameters parsed correctly
- **Assertions**:
  - `maxAllowablePaymentAmount().has_value() == true`
  - `*maxAllowablePaymentAmount() == TrustLineAmount(1500)`

*Test 2: Parse command without maxAllowablePaymentAmount parameter*
- **Setup**: Command string `CREATE:contractors/transactions/exchange:1:12:127.0.0.1:2003:1000:2:1`
- **Expected**:
  - `maxAllowablePaymentAmount()` returns `nullopt`
  - All other parameters parsed correctly
- **Assertions**:
  - `maxAllowablePaymentAmount().has_value() == false`

*Test 3: Parse command with invalid maxAllowablePaymentAmount (negative)*
- **Setup**: Command string with negative value (if possible given parsing rules)
- **Expected**: Parsing fails with ValueError
- **Assertions**: EXPECT_THROW(ValueError)

*Test 4: responseAllowablePaymentAmountExceeded() returns correct result*
- **Setup**: Valid command object
- **Execution**: Call `responseAllowablePaymentAmountExceeded()`
- **Expected**: CommandResult with code 415 and message "Allowable payment amount has been exceeded"
- **Assertions**:
  - Result code == 415
  - Result message contains "Allowable payment amount has been exceeded"

**2. CoordinatorExchangePaymentTransaction::calculateTotalReservedPaymentAmount() Tests**

*Test 5: Calculate total with no outgoing reservations*
- **Setup**: Transaction без вихідних резервів у всіх еквівалентах
- **Execution**: Call `calculateTotalReservedPaymentAmount()`
- **Expected**: Returns TrustLineAmount(0)
- **Assertions**: `result == TrustLineAmount(0)`

*Test 6: Aggregate total по одному еквіваленту*
- **Setup**:
  - У `exchangeEquivalents()` лише один еквівалент
  - `totalReservedAmount(Outgoing, equivalent)` повертає 1000
- **Execution**: Call `calculateTotalReservedPaymentAmount()`
- **Expected**: Returns TrustLineAmount(1000)
- **Assertions**: `result == TrustLineAmount(1000)`

*Test 7: Aggregate total по кількох еквівалентах*
- **Setup**:
  - У `exchangeEquivalents()` два еквіваленти
  - `totalReservedAmount` повертає 1200 для першого і 600 для другого
- **Execution**: Call `calculateTotalReservedPaymentAmount()`
- **Expected**: Returns TrustLineAmount(1800)
- **Assertions**: `result == TrustLineAmount(1800)`

*Test 8: totalReservedAmount не враховує незавершені резерви*
- **Setup**:
  - Один з еквівалентів має частковий резерв, який ще не враховується агрегатором
- **Execution**: Call `calculateTotalReservedPaymentAmount()`
- **Expected**: Повертається сума лише завершених резервів згідно з поведінкою `totalReservedAmount`
- **Assertions**: Значення збігається з очікуваним результатом агрегатора

**3. Validation in runPathsResourceProcessingStage() Tests**

*Test 9: Validation passes when mExchangeAmount within limit*
- **Setup**:
  - `maxAllowablePaymentAmount = 2000`
  - Calculated `mExchangeAmount = 1800`
- **Expected**: Validation passes, method continues to next step
- **Assertions**: Transaction continues (no early return)

*Test 10: Validation fails when mExchangeAmount exceeds limit*
- **Setup**:
  - `maxAllowablePaymentAmount = 1500`
  - Calculated `mExchangeAmount = 1800`
- **Expected**: Returns `resultAllowablePaymentAmountExceeded()`
- **Assertions**:
  - Result code == 415
  - Warning logged with both amounts

*Test 11: Validation skipped when maxAllowablePaymentAmount not provided*
- **Setup**:
  - `maxAllowablePaymentAmount = nullopt`
  - Calculated `mExchangeAmount = 10000` (very high)
- **Expected**: Validation skipped, method continues
- **Assertions**: Transaction continues (no early return)

**4. Validation After Path Reservation Tests**

*Test 12: Validation passes when total reserved within limit (processRemoteNodeResponse)*
- **Setup**:
  - `maxAllowablePaymentAmount = 2000`
  - Path completed with `isLastIntermediateNodeProcessed() == true`
  - `calculateTotalReservedPaymentAmount()` returns 1800
- **Expected**: Validation passes, method continues
- **Assertions**: Transaction continues

*Test 13: Validation fails when total reserved exceeds limit (processRemoteNodeResponse)*
- **Setup**:
  - `maxAllowablePaymentAmount = 1500`
  - Path completed with `isLastIntermediateNodeProcessed() == true`
  - `calculateTotalReservedPaymentAmount()` returns 1800
- **Expected**:
  - `reject()` called without return (triggers `rollBack()` from base class)
  - Returns `resultAllowablePaymentAmountExceeded()`
- **Assertions**:
  - Result code == 415
  - Warning logged
  - All reservations dropped (via checking reservation state after)

*Test 14: Validation fails even if mAmount target achieved (processRemoteNodeResponse)*
- **Setup**:
  - `maxAllowablePaymentAmount = 1500`
  - `mAmount` target already achieved (receiver got desired amount)
  - Total reserved = 1800 (exceeds limit)
- **Expected**: Transaction still aborts with result 415
- **Assertions**:
  - Result code == 415
  - Transaction terminated despite achieving mAmount

*Test 15: Validation skipped when maxAllowablePaymentAmount not provided (processRemoteNodeResponse)*
- **Setup**:
  - `maxAllowablePaymentAmount = nullopt`
  - Total reserved = 10000 (very high)
- **Expected**: Validation skipped, method continues
- **Assertions**: Transaction continues

*Test 16: Validation in processNeighborFurtherReservationResponse (mirror of Test 13)*
- **Setup**: Same as Test 13 but in `processNeighborFurtherReservationResponse`
- **Expected**: Same behavior
- **Assertions**: Same as Test 13

**5. Result Code Tests**

*Test 17: resultAllowablePaymentAmountExceeded() returns correct result*
- **Setup**: Coordinator transaction object
- **Execution**: Call `resultAllowablePaymentAmountExceeded()`
- **Expected**: TransactionResult with code 415
- **Assertions**: Result code == 415

**6. End-to-End Scenario Tests**

*Test 20: Payment succeeds when conditions stable and within limit*
- **Setup**:
  - `maxAllowablePaymentAmount = 2000`
  - Estimated `mExchangeAmount = 1800`
  - No condition changes during execution
  - Final total reserved = 1800
- **Expected**: Payment succeeds
- **Assertions**:
  - Result code == 201 (success)
  - Payment completed

*Test 22: Payment fails early when estimated amount exceeds limit*
- **Setup**:
  - `maxAllowablePaymentAmount = 1500`
  - Estimated `mExchangeAmount = 1800`
- **Expected**: Payment fails in `runPathsResourceProcessingStage()`
- **Assertions**:
  - Result code == 415
  - No reservations made

*Test 21: Payment fails late when conditions change causing limit violation*
- **Setup**:
  - `maxAllowablePaymentAmount = 1800`
  - Estimated `mExchangeAmount = 1700`
  - First path reserved: 900
  - Second path reserved: 950 (due to condition change)
  - Total = 1850 (exceeds 1800)
- **Expected**:
  - Payment fails after second path reservation
  - `reject()` called without return, triggering `rollBack()`
  - Returns `resultAllowablePaymentAmountExceeded()`
- **Assertions**:
  - Result code == 415
  - All reservations dropped via rollBack()

*Test 22: Payment succeeds without limit parameter (legacy behavior)*
- **Setup**:
  - `maxAllowablePaymentAmount = nullopt`
  - Estimated `mExchangeAmount = 10000` (very high)
  - Final total reserved = 10000
- **Expected**: Payment succeeds (no limit checks)
- **Assertions**:
  - Result code == 201 (success)
  - Payment completed despite high cost

#### Regression Testing (Unit)
- Scope: Ensure changes don't break existing exchange payment execution
- Verify payments without `maxAllowablePaymentAmount` work unchanged
- Validate single-equivalent payments remain unaffected
- Confirm existing validation logic unaffected

#### Execution in CI/Locally
- Build tests in `build-tests` and run the produced binaries
- All unit tests must pass before PRD completion

### Quality Gates
- All unit tests pass in `build-tests`
- Total reserved amount calculation 100% accurate in all test scenarios
- Validation correctly detects limit violations in all test cases
- Cleanup properly drops all reservations in all test cases
- Backward compatibility maintained (parameter optional)
- No regressions in existing exchange payment behavior

## Deployment & Release Strategy
### Release Approach
- **Release Type**: Feature addition (backward compatible enhancement)
- **Rollout Strategy**: Direct deployment; optional parameter ensures backward compatibility
- **Rollback Plan**: Revert to previous version if critical issues found; no data migration concerns

### Database Migrations
- No database migrations required (runtime parameter only)

### Communication Plan
- **Internal**: Technical documentation for development team
- **External**: User documentation for new parameter and result code
- **Documentation Updates**:
  - Command documentation with new parameter format
  - Error code documentation with 415 explanation
  - Usage examples and recommendations

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- **Primary Metrics**:
  - Validation accuracy (target: 100% correct detection)
  - Total reserved amount calculation accuracy (target: 100%)
  - Cleanup success rate (target: 100% when limit exceeded)
  - User limit violations (target: 0)
- **Leading Indicators**: Unit test pass rate
- **Baseline Values**: N/A (new feature)
- **Target Values**:
  - 100% unit test pass rate
  - 100% validation accuracy
  - 0 payments executing beyond user's specified limit
  - 100% proper cleanup on limit violation

### Monitoring Plan
- **New Dashboards/Alerts**: Not applicable (internal logic enhancement)
- **Enhanced Monitoring**: Extended logging for limit checks and violations
- **A/B Testing**: Not applicable

### Review Schedule
- **Daily**: Development progress and unit test status
- **Weekly**: Code review and validation logic verification
- **Post-Implementation Review**: User feedback on feature utility

## Appendices
### Glossary
- **Maximum Allowable Payment Amount**: User-specified limit on payment amount in payment equivalent
- **Total Reserved Payment Amount**: Сума вихідних резервів координатора, повернена `totalReservedAmount(AmountReservation::Outgoing, equivalent)` для всіх еквівалентів платежу
- **Early Validation**: Check before making reservations (in `runPathsResourceProcessingStage`)
- **Late Validation**: Check after each path reservation completion
- **Result Code 415**: "Allowable payment amount has been exceeded"

### References
- [Exchange Payment with Commissions PRD](06-exchange-payment-with-commissions.md)
- [Exchange Payment Topology Collection PRD](07-exchange-payment-topology-collection.md)
- [Path Capacity Adjustment PRD](08-path-capacity-adjustment.md)
- [Exchange Rate and Commission Change Handling PRD](09-exchange-rate-commission-change-handling.md)
- [Payment Protocol](../../../architecture/vtcpd/protocols/payment-protocol.md)
- [CoordinatorExchangePaymentTransaction Implementation](../../../src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h)
- [CreditUsageExchangeCommand Implementation](../../../src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.h)

### Detailed Component Specifications

#### Modified Classes

##### CreditUsageExchangeCommand
- **Location**: `src/core/interface/commands_interface/commands/payments/CreditUsageExchangeCommand.h/.cpp`
- **New Field**: `optional<TrustLineAmount> mMaxAllowablePaymentAmount`
- **New Getter**: `const optional<TrustLineAmount>& maxAllowablePaymentAmount() const`
- **New Response Method**: `CommandResult::SharedConst responseAllowablePaymentAmountExceeded() const`
- **Parsing Changes**:
  - Accept optional parameter after `exchangeEquivalents`
  - Format: `...:exchangeEquivalent1:exchangeEquivalent2:maxAllowablePaymentAmount` (optional last)
  - Validate if provided: must be positive

##### CoordinatorExchangePaymentTransaction
- **Location**: `src/core/transactions/transactions/regular/payments/CoordinatorExchangePaymentTransaction.h/.cpp`
- **New Methods**:
  ```cpp
  TrustLineAmount calculateTotalReservedPaymentAmount() const;
  void dropReservationsOnAllPaths();
  TransactionResult::SharedConst resultAllowablePaymentAmountExceeded();
  ```
- **Modified Methods**:
  - `runPathsResourceProcessingStage()`: Add validation after `mExchangeAmount` calculation
  - `processRemoteNodeResponse()`: Add validation in `if (path->isLastIntermediateNodeProcessed())` block
  - `processNeighborFurtherReservationResponse()`: Add validation in `if (path->isLastIntermediateNodeProcessed())` block

#### Algorithm Implementation Examples

**Command Parsing Extension**:
```cpp
// In CreditUsageExchangeCommand constructor
parse(
    commandTail.begin(),
    commandTail.end(),
    (
        *(digit[amountAddNumber] > !alpha > !punct)
        > char_(kTokensSeparator)
        > +(int_[equivalentParse])
        > *(char_(kTokensSeparator) > int_[exchangeEquivalentParse])
        > -(char_(kTokensSeparator) > +(digit[maxAllowableAmountParse])) // NEW: optional
        > -(char_(kTokensSeparator) > *(char_[payloadParse] - eol))
        > eol > eoi));
```

**Validation Integration Example**:
```cpp
// In runPathsResourceProcessingStage()
if (mCommand->maxAllowablePaymentAmount().has_value()) {
    if (mExchangeAmount > *mCommand->maxAllowablePaymentAmount()) {
        warning() << "Exchange amount " << mExchangeAmount
                  << " exceeds limit " << *mCommand->maxAllowablePaymentAmount();
        return resultAllowablePaymentAmountExceeded();
    }
}
```

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | 2025-11-04 | Claude Code | Initial draft for allowable payment amount control | Phase 1 |

**Related Documents**
- **Master Project Vision**: vTCP Decentralized Payment Network
- **Previous Iteration PRD**: [09-exchange-rate-commission-change-handling.md](09-exchange-rate-commission-change-handling.md)
- **Technical Architecture**: [vTCP Network Architecture](../../../architecture/vtcpd/)
- **Payment Protocol**: [payment-protocol.md](../../../architecture/vtcpd/protocols/payment-protocol.md)
