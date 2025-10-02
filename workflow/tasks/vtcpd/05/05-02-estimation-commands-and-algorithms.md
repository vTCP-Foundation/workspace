# 05-02 - Estimation Commands and Algorithms

# Links
- [PRD](/home/mc/Personal/vtcp/vtcpd/workspace/workflow/prd/vtcpd/05-payment-estimation.md)
- [Previous task: 05-01 Exchange Paths Manager Implementation](/home/mc/Personal/vtcp/vtcpd/workspace/workflow/tasks/vtcpd/05/05-01-exchange-paths-manager-implementation.md)

# Description
Implement bidirectional payment estimation algorithms (`forwardSimulatePath` and `inverseSimulatePath`) with accurate commission "charge once" semantics, exchange limit validation, and edge capacity tracking. Create command and transaction classes for both estimation directions: `EstimateReceiveForPaymentAmountCommand/Transaction` (payment → receive) and `EstimatePaymentForReceiveAmountCommand/Transaction` (receive → payment). Implement error handling for all edge cases with appropriate error codes (401, 412, 462).

# Requirements and DOD

## Path Simulation Algorithms
1. Implement `forwardSimulatePath()` helper function:
   - **Input**: `ExchangePath path`, `double inputAmount`, `set<pair<ContractorID, SerializedEquivalent>>& appliedCommissions`, `map<EdgeKey, double>& edgeRemainingCapacity`
   - **Process**:
     - Iterate through path segments (node to node)
     - For each segment: apply exchange rate if applicable, deduct commission if not already charged (check `appliedCommissions` set), validate edge capacity
     - Track cumulative flow amount through path
     - Update `edgeRemainingCapacity` map for each consumed edge
     - Add `(nodeID, equivalent)` to `appliedCommissions` when commission deducted
   - **Output**: `double deliveredAmount` at receiver end
   - **Exchange Rate Application**: `outputAmount = inputAmount * (exchangeRate / pow(10, exchangeRateShift))`
   - **Commission Deduction**: If `(nodeID, equivalent)` not in `appliedCommissions`, subtract commission from flow
   - **Capacity Constraint**: Reduce flow if edge capacity insufficient, update remaining capacity
2. Implement `inverseSimulatePath()` helper function:
   - **Input**: `ExchangePath path`, `double targetOutputAmount`, `set<pair<ContractorID, SerializedEquivalent>>& appliedCommissions`, `map<EdgeKey, double>& edgeRemainingCapacity`
   - **Process**:
     - Work backwards through path from receiver to sender
     - For each segment: apply inverse exchange rate, add commission if not already charged, validate edge capacity
     - Calculate required input amount at sender to achieve target output at receiver
     - Update `edgeRemainingCapacity` and `appliedCommissions` same as forward simulation
   - **Output**: `double requiredInputAmount` at sender end
   - **Inverse Exchange Rate**: `inputAmount = outputAmount / (exchangeRate / pow(10, exchangeRateShift))`
   - **Commission Addition**: If `(nodeID, equivalent)` not in `appliedCommissions`, add commission to required input
3. Exchange limit validation in both algorithms:
   - Extract `minExchangeAmount` and `maxExchangeAmount` from `ExchangeStep`
   - Check flow amount against limits: `amount >= minExchangeAmount && amount <= maxExchangeAmount`
   - If below min: skip path (return 0.0)
   - If above max: cap flow at max
   - If within range: proceed normally
4. Edge capacity tracking:
   - Use `EdgeKey{from, to, equivalent}` as map key
   - Initialize capacity from path's edge data or topology manager
   - Deduct consumed flow from `edgeRemainingCapacity[key]`
   - Check capacity before simulating flow: if insufficient, reduce flow amount

## EstimateReceiveForPaymentAmountCommand and Transaction
1. Create `EstimateReceiveForPaymentAmountCommand` class:
   - Inherit from `BaseUserCommand`
   - Identifier: `GET:contractors/transactions/estimate/receive`
   - Parse format: `GET:contractors/transactions/estimate/receive:<address_type>:<address>:<payment_amount>:<sender_equivalent>:<receiver_equivalent>`
   - Validation: contractor address valid, payment_amount > 0, equivalents valid
   - Output: initiate `EstimateReceiveForPaymentAmountTransaction`
2. Create `EstimateReceiveForPaymentAmountTransaction` class:
   - Inherit from `BaseTransaction`
   - Dependencies: `ExchangePathsManager*`, `TopologyTrustLinesManager*`, `EquivalentsSubsystemsRouter*`
   - Implement `run()` method:
     - Construct `PathCacheKey{contractorID, senderEquivalent, receiverEquivalent}`
     - Call `mExchangePathsManager->retrievePaths(key)`
     - If no paths: return error 462
     - Distribute `paymentAmount` across cached paths using forward simulation
     - Return delivered receive amount (200:<receive_amount>)
   - Implement `estimateReceive()` helper:
     - Initialize `remainingPayment = paymentAmount`, `totalReceive = 0`
     - Initialize `appliedCommissions` set and `edgeRemainingCapacity` map
     - For each `OptimalPathResult` in cached paths (already sorted by efficiency):
       - Determine `pathInput = min(remainingPayment, path.optimal_flow)`
       - Call `forwardSimulatePath(path, pathInput, appliedCommissions, edgeRemainingCapacity)`
       - Add result to `totalReceive`, subtract `pathInput` from `remainingPayment`
       - Break if `remainingPayment == 0`
     - Return `totalReceive`
   - Error handling: catch exceptions, return error 401 for unexpected errors
3. Response format:
   - Success: `200:<total_receive_amount>`
   - Error 401: `401` (unexpected error)
   - Error 462: `462` (no cached paths)

## EstimatePaymentForReceiveAmountCommand and Transaction
1. Create `EstimatePaymentForReceiveAmountCommand` class:
   - Inherit from `BaseUserCommand`
   - Identifier: `GET:contractors/transactions/estimate/payment`
   - Parse format: `GET:contractors/transactions/estimate/payment:<address_type>:<address>:<receive_amount>:<receiver_equivalent>:<sender_equivalent>`
   - Validation: contractor address valid, receive_amount > 0, equivalents valid
   - Output: initiate `EstimatePaymentForReceiveAmountTransaction`
2. Create `EstimatePaymentForReceiveAmountTransaction` class:
   - Inherit from `BaseTransaction`
   - Dependencies: `ExchangePathsManager*`, `TopologyTrustLinesManager*`, `EquivalentsSubsystemsRouter*`
   - Implement `run()` method:
     - Construct `PathCacheKey{contractorID, senderEquivalent, receiverEquivalent}`
     - Call `mExchangePathsManager->retrievePaths(key)`
     - If no paths: return error 462
     - Iterate through cached paths using inverse simulation to achieve `receiveAmount`
     - If insufficient: return error 412
     - Return required payment amount (200:<payment_amount>)
   - Implement `estimatePayment()` helper:
     - Initialize `remainingReceive = receiveAmount`, `totalPayment = 0`
     - Initialize `appliedCommissions` set and `edgeRemainingCapacity` map
     - For each `OptimalPathResult` in cached paths:
       - Determine `targetForPath = min(remainingReceive, path.received_amount)`
       - Call `inverseSimulatePath(path, targetForPath, appliedCommissions, edgeRemainingCapacity)`
       - Add result to `totalPayment`, subtract `targetForPath` from `remainingReceive`
       - Break if `remainingReceive == 0`
     - If `remainingReceive > 0`: throw `InsufficientPathsError(412)`
     - Return `totalPayment`
   - Error handling: catch exceptions, return error 401 for unexpected errors
3. Response format:
   - Success: `200:<total_payment_amount>`
   - Error 401: `401` (unexpected error)
   - Error 412: `412` (insufficient paths to deliver requested amount)
   - Error 462: `462` (no cached paths)

## Commission Lookup Integration
1. Both transactions fetch current commission values from `TopologyTrustLinesManager`:
   - For each intermediate node in path, query commission for `(nodeID, equivalent)`
   - Use current commission values (may differ from values at max flow calculation time)
   - Apply "charge once" semantics via `appliedCommissions` set across all paths
2. Commission refresh ensures estimation reflects current network state within cache TTL

## DOD Criteria
- `forwardSimulatePath()` and `inverseSimulatePath()` correctly simulate flow with exchange rates, commissions, and capacity constraints
- Commission "charge once" semantics verified: same intermediate node does not charge twice across multiple paths
- Exchange limit validation correctly skips or caps flows based on min/max constraints
- Edge capacity tracking correctly reduces available capacity after each path simulation
- Both command classes successfully parse input parameters and initiate corresponding transactions
- Both transaction classes return correct results for valid inputs
- Error codes 401, 412, 462 returned correctly for respective error conditions
- All error cases logged with appropriate severity and detail

# Implementation Plan

## Step 1: Implement Path Simulation Algorithms
1. Create helper file (e.g., `/home/mc/Personal/vtcp/vtcpd/src/core/paths/lib/PathSimulation.h` and `.cpp`):
   - Include necessary headers: `ExchangePath.h`, `OptimalPathResult.h`, `EdgeKey.h`, `TopologyTrustLinesManager.h`
2. Implement `forwardSimulatePath()`:
   - Signature: `double forwardSimulatePath(const ExchangePath& path, double inputAmount, set<pair<ContractorID, SerializedEquivalent>>& appliedCommissions, map<EdgeKey, double>& edgeRemainingCapacity, TopologyTrustLinesManager* topologyManager)`
   - Initialize `currentAmount = inputAmount`
   - Iterate through path nodes (i = 0 to nodes.size()-1):
     - Create `EdgeKey{nodes[i], nodes[i+1], equivalents[i]}`
     - Check edge capacity: if `currentAmount > edgeRemainingCapacity[key]`, reduce `currentAmount`
     - Update `edgeRemainingCapacity[key] -= currentAmount`
     - If exchange step at node i+1: apply exchange rate, validate min/max limits
     - If commission at node i+1: check `appliedCommissions`, deduct if not present, add to set
   - Return `currentAmount` as delivered amount
3. Implement `inverseSimulatePath()`:
   - Signature: `double inverseSimulatePath(const ExchangePath& path, double targetOutputAmount, set<pair<ContractorID, SerializedEquivalent>>& appliedCommissions, map<EdgeKey, double>& edgeRemainingCapacity, TopologyTrustLinesManager* topologyManager)`
   - Initialize `requiredAmount = targetOutputAmount`
   - Iterate backwards through path nodes (i = nodes.size()-1 to 0):
     - If commission at node i: check `appliedCommissions`, add to `requiredAmount` if not present, add to set
     - If exchange step at node i: apply inverse exchange rate, validate min/max limits
     - Create `EdgeKey{nodes[i-1], nodes[i], equivalents[i-1]}`
     - Check edge capacity: if `requiredAmount > edgeRemainingCapacity[key]`, return 0.0 (path cannot satisfy)
     - Update `edgeRemainingCapacity[key] -= requiredAmount`
   - Return `requiredAmount` as required input at sender
4. Add unit tests for both functions (defer to task 05-03)

## Step 2: Implement EstimateReceiveForPaymentAmountCommand and Transaction
1. Create `EstimateReceiveForPaymentAmountCommand.h` and `.cpp` in `/home/mc/Personal/vtcp/vtcpd/src/core/interface/commands_interface/commands/max_flow_calculation/`:
   - Inherit from `BaseUserCommand`
   - Implement constructor parsing command string
   - Validate parameters: address, payment_amount, sender_equivalent, receiver_equivalent
   - Store parsed values as member variables
2. Create `EstimateReceiveForPaymentAmountTransaction.h` and `.cpp` in `/home/mc/Personal/vtcp/vtcpd/src/core/transactions/transactions/max_flow_calculation/`:
   - Inherit from `BaseTransaction`
   - Add dependencies: `ExchangePathsManager*`, `TopologyTrustLinesManager*`
   - Implement `run()`: retrieve paths, call `estimateReceive()`, format response
   - Implement `estimateReceive()`: forward simulation loop as specified in requirements
   - Implement error handling: try-catch around estimation logic, return error codes
3. Register command in command factory
4. Register transaction in transaction factory

## Step 3: Implement EstimatePaymentForReceiveAmountCommand and Transaction
1. Create `EstimatePaymentForReceiveAmountCommand.h` and `.cpp`:
   - Similar structure to receive estimation command
   - Parse `receive_amount`, `receiver_equivalent`, `sender_equivalent` parameters
2. Create `EstimatePaymentForReceiveAmountTransaction.h` and `.cpp`:
   - Similar structure to receive estimation transaction
   - Implement `estimatePayment()`: inverse simulation loop as specified in requirements
   - Implement error 412 handling: if `remainingReceive > 0` after all paths, throw exception
3. Register command in command factory
4. Register transaction in transaction factory

## Step 4: Commission Lookup Integration
1. In both transactions, after retrieving cached paths:
   - For each path, iterate through nodes
   - For each intermediate node, query `mTopologyManager->getCommission(nodeID, equivalent)`
   - Use current commission value in simulation algorithms
2. Pass `TopologyTrustLinesManager*` to simulation functions for real-time commission lookup

## Step 5: Error Handling and Logging
1. Define custom exception classes if needed: `InsufficientPathsError(412)`, `NoPathsError(462)`
2. In transaction `run()` method:
   - Wrap estimation logic in try-catch
   - Catch `NoPathsError`: return `resultError(462)`
   - Catch `InsufficientPathsError`: return `resultError(412)`
   - Catch generic exceptions: log error, return `resultError(401)`
3. Add logging:
   - Error log for 401: `error() << "Estimation failed with unexpected error: " << exception.what()`
   - Warning log for 412: `warning() << "Insufficient paths to deliver " << receiveAmount`
   - Warning log for 462: `warning() << "No cached optimal paths for key"`

## Step 6: Verification
1. Build project and verify no compilation errors
2. Manual testing with mocked `ExchangePathsManager` data
3. Verify command parsing for various input formats
4. Verify error codes returned correctly for edge cases

# Test Plan

## Complexity Level
Complex

## Test Scope
- Path simulation algorithms: forward and inverse with all constraint types
- Commission "charge once" semantics across multiple paths
- Exchange limit validation (min/max)
- Edge capacity tracking
- Command parsing and transaction execution
- Error handling for all error codes (401, 412, 462)

## Test Approach (Unit Tests Only)
All tests execute in `build-tests` with no Docker dependencies. Mock `ExchangePathsManager`, `TopologyTrustLinesManager`, and topology data.

### Path Simulation Algorithm Tests
1. **Forward Simulation - Simple Path**: Single-equivalent path A→B→C, no exchanges, verify commission deduction
2. **Forward Simulation - Exchange Path**: Path with exchange step, verify exchange rate application
3. **Forward Simulation - Min/Max Limits**: Exchange with min=100, max=500, verify flow capping
4. **Forward Simulation - Capacity Constraint**: Edge capacity < input flow, verify flow reduction
5. **Inverse Simulation - Simple Path**: Reverse of forward test 1, verify commission addition
6. **Inverse Simulation - Exchange Path**: Reverse of forward test 2, verify inverse exchange rate
7. **Inverse Simulation - Min/Max Limits**: Same limits as forward test 3, verify rejection or capping
8. **Inverse Simulation - Capacity Constraint**: Edge capacity < required input, verify path skipped (returns 0.0)
9. **Commission Charge Once**: Two paths sharing intermediate node B, verify commission deducted only on first path

### EstimateReceiveForPaymentAmountTransaction Tests

#### Test Topology 1: Single-Equivalent Path (No Exchange)
- **Setup**: A → B → C in equivalent 1, capacities A→B=1000, B→C=800, commission at B=10
- **Test 1**: Payment amount 510
  - Expected: Receive ~500 (510 - 10 commission)
- **Test 2**: Payment amount 2000
  - Expected: Receive ~790 (limited by B→C capacity)

#### Test Topology 2: Cross-Equivalent Path with Exchange
- **Setup**: A (eq 1) → B → X [1→2, rate=2.0] → C (eq 2), capacities A→B=1000, B→X=800, X→C=1500
- **Test 1**: Payment amount 200 in eq 1
  - Expected: Receive ~400 in eq 2 (200 * 2.0)
- **Test 2**: Payment amount 1000 in eq 1
  - Expected: Receive ~1600 in eq 2 (limited by B→X capacity 800 * 2.0)

#### Test Topology 3: Multi-Path with Commissions
- **Setup**: Same as PRD 04 commission example (3 paths through intermediate node B with commission=10)
- **Test**: Payment amount 1000
  - Expected: Receive ~990 (commission charged once across all paths)
  - Verify: `appliedCommissions` set contains only one entry for (B, eq)

#### Test Topology 4: Exchange Min/Max Limits
- **Setup**: A (eq 1) → X [1→2, rate=1.5, min=100, max=500] → B (eq 2)
- **Test 1**: Payment amount 50 in eq 1
  - Expected: Receive 0 (below min exchange)
- **Test 2**: Payment amount 600 in eq 1
  - Expected: Receive ~750 in eq 2 (capped at max exchange: 500 * 1.5)

#### Error Case Tests
- **No Cached Paths**: Mock `retrievePaths()` returns nullopt, verify error 462
- **Expired Paths**: Mock TTL exceeded, verify error 462

### EstimatePaymentForReceiveAmountTransaction Tests

#### Test Topology 1: Single-Equivalent Path (No Exchange)
- **Setup**: Same as receive estimation test 1
- **Test 1**: Receive amount 500
  - Expected: Payment ~510 (500 + 10 commission)
- **Test 2**: Receive amount 900
  - Expected: Error 412 (max deliverable ~790)

#### Test Topology 2: Cross-Equivalent Path with Exchange
- **Setup**: Same as receive estimation test 2
- **Test 1**: Receive amount 400 in eq 2
  - Expected: Payment ~200 in eq 1 (400 / 2.0)
- **Test 2**: Receive amount 2000 in eq 2
  - Expected: Error 412 (max deliverable ~1600)

#### Test Topology 3: Multi-Path with Commissions
- **Setup**: Same as receive estimation test 3
- **Test**: Receive amount 990
  - Expected: Payment ~1000 (commission charged once)

#### Test Topology 4: Exchange Min/Max Limits
- **Setup**: Same as receive estimation test 4
- **Test 1**: Receive amount 50 in eq 2
  - Expected: Error 412 (50/1.5 ≈ 33 < min exchange 100)
- **Test 2**: Receive amount 600 in eq 2
  - Expected: Payment ~400 in eq 1 (600/1.5 = 400, within max)

#### Error Case Tests
- **No Cached Paths**: Error 462
- **Insufficient Paths**: Receive amount exceeds all path capacity, error 412

### Command Parsing Tests
1. **Valid Input - Receive Estimation**: Parse `GET:contractors/transactions/estimate/receive:12:127.0.0.1:2003:500:1:2`, verify all parameters extracted correctly
2. **Valid Input - Payment Estimation**: Parse `GET:contractors/transactions/estimate/payment:12:127.0.0.1:2003:1000:2:1`, verify all parameters extracted correctly
3. **Invalid Input**: Malformed command string, verify exception or error handling

### Integration with ExchangePathsManager
1. **Successful Retrieval**: Mock cached paths, verify transaction retrieves and uses them
2. **Cache Miss**: Mock `retrievePaths()` returns nullopt, verify error 462
3. **Multiple Keys**: Verify correct `PathCacheKey` constructed for different contractor/equivalent combinations

## Success Criteria
- All unit tests pass in `build-tests`
- Estimation results match expected values for all test topologies
- Commission "charge once" verified across multi-path scenarios
- Exchange limit validation correctly skips or caps flows
- Edge capacity tracking reduces available capacity correctly
- Error codes 401, 412, 462 returned correctly in all edge cases
- Command parsing handles valid and invalid inputs gracefully

# Verification and Validation

## Architecture integrity
- Simulation algorithms reuse existing commission and exchange rate structures from PRD 04
- Transactions follow standard `BaseTransaction` pattern
- Commands follow standard `BaseUserCommand` pattern
- Integration with `ExchangePathsManager` via dependency injection
- Clear separation of concerns: algorithms, commands, transactions

## Security
- No cryptographic operations required
- Input validation prevents malformed commands from crashing system
- Error handling prevents information leakage via error messages
- Access control inherited from existing transaction framework

## Performance
- Forward simulation: O(path_length) per path, O(num_paths * path_length) per estimation
- Inverse simulation: same complexity as forward
- Typical estimation completes within 100ms for ≤50 paths (per PRD target)
- Commission lookup optimized via `TopologyTrustLinesManager` cache
- Edge capacity tracking: O(log(num_edges)) map operations per segment

## Scalability
- Algorithms handle up to 100 paths per estimation (per PRD target)
- Memory usage proportional to number of paths and path length
- No blocking operations; suitable for concurrent estimation requests
- Commission "charge once" set size bounded by number of unique (node, equivalent) pairs

## Reliability
- All error conditions handled with appropriate error codes
- Simulation algorithms validated against known topologies
- Commission and exchange logic reuses proven patterns from PRD 04
- Logging provides audit trail for debugging estimation failures
- No undefined behavior for edge cases (zero flow, missing paths, capacity exceeded)

## Maintainability
- Simulation algorithms extracted to separate helper functions for testability
- Clear separation between forward and inverse logic
- Extensive unit tests document expected behavior
- Error handling centralized in transaction `run()` method
- Code follows existing transaction and command patterns

## Cost
- Minimal CPU overhead: simulation algorithms efficient (linear complexity)
- No persistent storage costs
- Commission lookup reuses existing cache (no additional infrastructure)

## Compliance
- Follows project policy: task-driven development, explicit DOD criteria
- Adheres to existing code conventions and architectural patterns
- No external dependencies introduced
- Builds upon PRD 04 infrastructure without breaking changes

# Restrictions
- Commit changes only after successfully passing the tests (if they are provided for by the task)
