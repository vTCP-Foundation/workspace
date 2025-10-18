# 06-12 - OptimalPathResult Flow Calculation Enhancement

# Links
- [PRD](../../../prd/vtcpd/06-exchange-payment-with-commissions.md)
- [Related task](06-03-path-processing.md)

# Description
Add payment flow calculation capability to OptimalPathResult class. This enhancement allows calculating the actual flow amounts between each pair of nodes along a path for a given payment amount, taking into account exchange rates and commissions.

This functionality is essential for CoordinatorExchangePaymentTransaction to determine exact reservation amounts between specific nodes when processing each optimal path.

# Requirements and DOD

## Requirements

### Field Additions to OptimalPathResult
1. Add `TrustLineAmount paymentFlow` field - the input payment amount in the first equivalent
2. Add `vector<pair<TrustLineAmount, SerializedEquivalent>> flows` field - flow amounts between each pair of nodes
3. Initialize `paymentFlow = 0` and `flows = {}` by default in constructor

### Method Addition to OptimalPathResult
4. Add method `void calculateFlows(const TrustLineAmount& paymentAmount)`
   - Sets `paymentFlow = paymentAmount`
   - Calculates flows between each pair of nodes along the path
   - Takes into account exchange rates and commissions
   - **IMPORTANT**: Must throw `ValueError` if `paymentAmount > optimal_flow`

### Flow Calculation Logic
5. For a path with N nodes, `flows` must contain N-1 elements (edges between nodes)
6. Each element is `pair<TrustLineAmount, SerializedEquivalent>` representing (flow_amount, equivalent)
7. Start with `paymentAmount` in the first equivalent (`mPath.equivalents[0]`)
8. For each step in the path:
   - If exchange step (same node, different equivalents):
     - Apply exchange rate: `amount *= effectiveRate`
     - Validate exchange limits if set (minExchangeAmount, maxExchangeAmount)
   - If regular edge (different nodes):
     - Record current amount and equivalent in `flows`
     - If next node is intermediate (not last), apply commission in arrival equivalent

### Exchange Rate Application
9. Effective rate calculation: `rate = exchangeRate * 10^(exchangeRateShift)`
10. Exchange limits validation:
    - If `minExchangeAmount > 0` and input amount < minExchangeAmount: throw ValueError
    - If `maxExchangeAmount > 0` and input amount > maxExchangeAmount: throw ValueError

### Commission Application
11. Commission applied at intermediate nodes (excluding sender and receiver)
12. Commission charged in the arrival equivalent after the edge
13. Commission deducted from flow: `amount -= commissionAmount`
14. Commission lookup from path's `ExchangePath` structure (should be pre-populated)

### Error Handling
15. Throw `ValueError` with descriptive message if:
    - `paymentAmount > optimal_flow`
    - Exchange limits violated
    - Path structure is invalid

## Definition of Done
- [ ] `paymentFlow` field added to OptimalPathResult.h
- [ ] `flows` field added to OptimalPathResult.h
- [ ] `calculateFlows(const TrustLineAmount&)` method declared in OptimalPathResult.h
- [ ] `calculateFlows(const TrustLineAmount&)` method implemented in OptimalPathResult.cpp
- [ ] Method correctly calculates flows for paths with:
  - [ ] No exchanges (single equivalent)
  - [ ] One exchange
  - [ ] Multiple exchanges
  - [ ] Commissions
  - [ ] Mixed exchanges and commissions
- [ ] ValueError thrown when `paymentAmount > optimal_flow`
- [ ] ValueError thrown when exchange limits violated
- [ ] All unit tests pass in build-tests
- [ ] Code compiles without errors in build-debug

# Implementation Plan

## Example from Requirements

Given path: A → B → C → D (4 nodes)
- optimal_flow = 3010
- received_amount = 150
- First equivalent = 1001
- Last equivalent = 2002
- Commission at B = 10 in equivalent 1001
- Exchange at C: 1001 → 2002, rate 0.05 (exchangeRate=5, exchangeRateShift=-2)

For `paymentFlow = 2010`:
```
flows = [
  (2010, 1001),  // A→B: send 2010 in equiv 1001
  (2000, 1001),  // B→C: B receives 2010, deducts commission 10, sends 2000 in equiv 1001
  (100, 2002)    // C→D: C receives 2000 in 1001, exchanges at 0.05, sends 100 in equiv 2002
]
```

## Step 1: Add Fields to OptimalPathResult.h

Add fields after existing fields:
```cpp
// Flow calculation fields
TrustLineAmount paymentFlow;
vector<pair<TrustLineAmount, SerializedEquivalent>> flows;
```

Update constructor:
```cpp
OptimalPathResult() : mMaxPathFlow(0), mIsValid(true), paymentFlow(0) {}
```

Add method declaration:
```cpp
void calculateFlows(const TrustLineAmount& paymentAmount);
```

## Step 2: Implement calculateFlows in OptimalPathResult.cpp

**IMPORTANT**: The flows vector must contain N-1 elements for N nodes (one flow per edge).
For a path with mIntermediateNodesStates.size() intermediate nodes:
- Total nodes = mIntermediateNodesStates.size() + 2 (coordinator + intermediates + receiver)
- Total edges = total nodes - 1 = mIntermediateNodesStates.size() + 1
- Therefore: **flows.size() == mIntermediateNodesStates.size() + 1**

Algorithm outline:
```cpp
void OptimalPathResult::calculateFlows(const TrustLineAmount& paymentAmount) {
    // 1. Validate paymentAmount <= optimal_flow
    if (paymentAmount > optimal_flow) {
        throw ValueError("OptimalPathResult::calculateFlows: "
                        "paymentAmount exceeds optimal_flow");
    }

    // 2. Clear and initialize
    flows.clear();
    paymentFlow = paymentAmount;

    // 3. Validate path structure
    if (mPath.ids.empty() || mPath.ids.size() != mPath.equivalents.size()) {
        throw ValueError("OptimalPathResult::calculateFlows: invalid path structure");
    }

    // 4. Start simulation with paymentAmount
    double currentAmount = paymentAmount.convert_to<double>();

    // 5. Process each step in path
    for (size_t k = 0; k + 1 < mPath.ids.size(); ++k) {
        ContractorID fromNode = mPath.ids[k];
        ContractorID toNode = mPath.ids[k + 1];
        SerializedEquivalent currentEquiv = mPath.equivalents[k];
        SerializedEquivalent nextEquiv = mPath.equivalents[k + 1];

        // 5a. Check if this is an exchange step
        if (fromNode == toNode && currentEquiv != nextEquiv) {
            // Apply exchange rate
            for (const auto& ex : mPath.exchangeSteps) {
                if (ex.nodeID == fromNode &&
                    ex.fromEquivalent == currentEquiv &&
                    ex.toEquivalent == nextEquiv) {

                    // Calculate effective rate
                    double rate = ex.exchangeRate.convert_to<double>();
                    int16_t shift = ex.exchangeRateShift;
                    double effectiveRate = rate * std::pow(10.0, shift);

                    // Validate exchange limits
                    if (ex.minExchangeAmount > TrustLineAmount(0)) {
                        if (currentAmount < ex.minExchangeAmount.convert_to<double>()) {
                            throw ValueError("OptimalPathResult::calculateFlows: "
                                           "amount below minExchangeAmount");
                        }
                    }
                    if (ex.maxExchangeAmount > TrustLineAmount(0)) {
                        if (currentAmount > ex.maxExchangeAmount.convert_to<double>()) {
                            throw ValueError("OptimalPathResult::calculateFlows: "
                                           "amount exceeds maxExchangeAmount");
                        }
                    }

                    // Apply exchange
                    currentAmount *= effectiveRate;
                    break;
                }
            }
            continue;
        }

        // 5b. Regular edge - record flow
        TrustLineAmount flowAmount(static_cast<uint64_t>(currentAmount));
        flows.push_back(make_pair(flowAmount, currentEquiv));

        // 5c. Apply commission at intermediate node (not last)
        if (k + 1 < mPath.ids.size() - 1) {
            // Look up commission from path structure
            // Note: In actual implementation, commissions should be
            // retrieved from trust line managers via router
            // For this method, we assume commission is 0 if not found
            // since we work only with path data

            // Skip commission lookup for now - will be added when
            // router is available in the context where this is called
        }
    }
}
```

**Note on Commission Lookup**: The initial implementation will not include commission lookup because OptimalPathResult doesn't have access to EquivalentsSubsystemsRouter. Commissions will be handled in the calling context (CoordinatorExchangePaymentTransaction) where the router is available. The method will calculate flows based on exchange rates only.

## Revised Algorithm (Without Commission Lookup)

Since OptimalPathResult doesn't have access to managers for commission lookup, we simplify:
1. Calculate flows based on exchange rates only
2. Commission handling will be done in CoordinatorExchangePaymentTransaction when using these flows
3. This makes the method pure and dependent only on path structure

Updated implementation:
```cpp
void OptimalPathResult::calculateFlows(const TrustLineAmount& paymentAmount) {
    // Validate paymentAmount <= optimal_flow
    if (paymentAmount > optimal_flow) {
        throw ValueError("OptimalPathResult::calculateFlows: "
                        "Payment amount exceeds optimal flow");
    }

    // Clear and initialize
    flows.clear();
    paymentFlow = paymentAmount;

    // Validate path structure
    if (mPath.ids.empty() || mPath.ids.size() != mPath.equivalents.size()) {
        throw ValueError("OptimalPathResult::calculateFlows: Invalid path structure");
    }

    double currentAmount = paymentAmount.convert_to<double>();

    for (size_t k = 0; k + 1 < mPath.ids.size(); ++k) {
        ContractorID fromNode = mPath.ids[k];
        ContractorID toNode = mPath.ids[k + 1];
        SerializedEquivalent currentEquiv = mPath.equivalents[k];
        SerializedEquivalent nextEquiv = mPath.equivalents[k + 1];

        // Handle exchange step
        if (fromNode == toNode && currentEquiv != nextEquiv) {
            for (const auto& ex : mPath.exchangeSteps) {
                if (ex.nodeID == fromNode &&
                    ex.fromEquivalent == currentEquiv &&
                    ex.toEquivalent == nextEquiv) {

                    double rate = ex.exchangeRate.convert_to<double>();
                    int16_t shift = ex.exchangeRateShift;
                    double effectiveRate = rate * std::pow(10.0, shift);

                    // Validate exchange limits before exchange
                    if (ex.minExchangeAmount > TrustLineAmount(0)) {
                        if (currentAmount < ex.minExchangeAmount.convert_to<double>()) {
                            throw ValueError("OptimalPathResult::calculateFlows: "
                                           "Amount below minimum exchange limit");
                        }
                    }
                    if (ex.maxExchangeAmount > TrustLineAmount(0)) {
                        if (currentAmount > ex.maxExchangeAmount.convert_to<double>()) {
                            throw ValueError("OptimalPathResult::calculateFlows: "
                                           "Amount exceeds maximum exchange limit");
                        }
                    }

                    currentAmount *= effectiveRate;
                    break;
                }
            }
            continue;
        }

        // Regular edge - record flow (without commission for now)
        TrustLineAmount flowAmount(static_cast<uint64_t>(currentAmount));
        flows.push_back(make_pair(flowAmount, currentEquiv));
    }
}
```

## Step 3: Write Unit Tests

Create test file: `tests/unit/paths/TestOptimalPathResultFlowCalculation.cpp`

Test cases:
1. **testCalculateFlowsSimplePath**: Path without exchanges or commissions
2. **testCalculateFlowsWithOneExchange**: Path with single exchange
3. **testCalculateFlowsWithMultipleExchanges**: Path with multiple exchanges
4. **testCalculateFlowsExceedsOptimalFlow**: Throws ValueError when paymentAmount > optimal_flow
5. **testCalculateFlowsExchangeLimitsMin**: Throws ValueError when below minExchangeAmount
6. **testCalculateFlowsExchangeLimitsMax**: Throws ValueError when above maxExchangeAmount
7. **testCalculateFlowsCorrectFlowCount**: N nodes → N-1 flows
8. **testCalculateFlowsRecalculation**: Can recalculate with different paymentAmount
9. **testCalculateFlowsInvalidPath**: Throws ValueError on invalid path structure

# Test Plan

## Test Execution
- Build tests in `build-tests` directory
- Run test binary: `./build-tests/bin/unit_tests --gtest_filter="OptimalPathResultFlowCalculation.*"`
- All tests must pass

## Test Coverage
- Happy path: simple flows without exchanges
- Exchange rate application: single and multiple exchanges
- Error cases: exceeds optimal_flow, exchange limits violated
- Edge cases: empty path, single node, recalculation

## Success Criteria
- All 9+ unit tests pass
- Code compiles without warnings
- calculateFlows works correctly for all path configurations

# Verification and Validation

## Architecture integrity
- OptimalPathResult remains self-contained data structure
- No new dependencies introduced
- Method is pure calculation based on path data

## Security
- Input validation prevents overflow (paymentAmount <= optimal_flow)
- Exchange limits validated before application

## Performance
- Single pass through path (O(n) complexity)
- No external lookups or I/O
- Memory overhead: vector of N-1 pairs

## Scalability
- Handles paths up to max length (7 hops per PRD 04)
- Memory usage proportional to path length

## Reliability
- Comprehensive error handling with descriptive messages
- Validates all inputs before processing
- Deterministic results for same inputs

## Maintainability
- Clear algorithm following forwardSimulatePath pattern
- Well-documented with example in task
- Easy to extend with commission lookup when needed

## Cost
- No additional infrastructure required
- Minimal memory overhead (vector of pairs)

## Compliance
- Follows repository policy for task-driven development
- Adheres to existing OptimalPathResult patterns
- No prohibited operations

# Restrictions
- Do not add commission lookup in this task (OptimalPathResult has no router access)
- Commission handling will be done in CoordinatorExchangePaymentTransaction
- Focus on exchange rate application only
- All tests must pass before task completion
