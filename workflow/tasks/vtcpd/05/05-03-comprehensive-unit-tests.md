# 05-03 - Comprehensive Unit Tests

# Links
- [PRD](/home/mc/Personal/vtcp/vtcpd/workspace/workflow/prd/vtcpd/05-payment-estimation.md)
- [Previous task 1: 05-01 Exchange Paths Manager Implementation](/home/mc/Personal/vtcp/vtcpd/workspace/workflow/tasks/vtcpd/05/05-01-exchange-paths-manager-implementation.md)
- [Previous task 2: 05-02 Estimation Commands and Algorithms](/home/mc/Personal/vtcp/vtcpd/workspace/workflow/tasks/vtcpd/05/05-02-estimation-commands-and-algorithms.md)

# Description
Create comprehensive unit test suites for `ExchangePathsManager` (storage, retrieval, TTL expiration, thread safety) and both estimation transactions (`EstimatePaymentForReceiveAmountTransaction`, `EstimateReceiveForPaymentAmountTransaction`) using four detailed test topologies specified in PRD (single-equivalent, cross-equivalent, multi-path commissions, exchange limits). All tests execute in `build-tests` with mocked dependencies and no Docker usage.

# Requirements and DOD

## ExchangePathsManager Test Suite
1. **Store and Retrieve Test**:
   - Create multiple `PathCacheKey` instances with different contractor/equivalent combinations
   - Store `OptimalPathResult` vectors for each key
   - Retrieve by exact key match, verify paths identical to stored
   - Retrieve non-existent key, verify returns nullopt
2. **TTL Expiration Test**:
   - Mock `DateTime::now()` to control time progression
   - Store paths with current timestamp
   - Advance mock time by 601 seconds (beyond 600s TTL)
   - Manually trigger `removeExpiredPaths()` or wait for timer callback
   - Verify expired paths removed, retrieval returns nullopt
3. **Invalidate by Key Test**:
   - Store paths for keys K1, K2, K3
   - Call `invalidatePaths(K2)`
   - Verify K1 and K3 still retrievable, K2 returns nullopt
4. **Invalidate by Contractor Test**:
   - Store paths for contractor C with keys: (C, eq1, eq2), (C, eq1, eq3), (C, eq2, eq3)
   - Store paths for contractor D with keys: (D, eq1, eq2)
   - Call `invalidatePathsForContractor(C)`
   - Verify all C keys return nullopt, D keys still retrievable
5. **Invalidate by Equivalent Test**:
   - Store paths with mixed sender/receiver equivalents: (C1, eq1, eq2), (C1, eq2, eq3), (C2, eq1, eq3)
   - Call `invalidatePathsForEquivalent(eq1)` (matches sender in first/third, receiver in none)
   - Verify keys with eq1 as sender or receiver removed
6. **Thread Safety Test**:
   - Create 10 threads concurrently storing different keys
   - Create 10 threads concurrently retrieving various keys
   - Run for 5 seconds
   - Verify no crashes, no data corruption, all stored keys retrievable
7. **Edge Cases**:
   - Store empty path vector, verify retrieval returns empty vector (not nullopt)
   - Store paths with same sender and receiver equivalent (eq1 = eq2), verify storage and retrieval work
   - Retrieve immediately after TTL expiration boundary, verify nullopt returned

## Path Simulation Algorithm Test Suite
1. **Forward Simulation - Simple Path** (no exchange):
   - Path: A → B → C in equivalent 1
   - Capacities: A→B = 1000, B→C = 800
   - Commission: B has 10 units in eq 1
   - Input: 500 units
   - Expected output: 490 (500 - 10 commission)
2. **Forward Simulation - Exchange Path**:
   - Path: A (eq 1) → B (eq 1) → X [1→2, rate=2.0] → C (eq 2)
   - Input: 200 units in eq 1
   - Expected output: ~400 units in eq 2 (200 * 2.0)
3. **Forward Simulation - Exchange Min/Max Limits**:
   - Path: A (eq 1) → X [1→2, rate=1.5, min=100, max=500] → B (eq 2)
   - Test 1 (below min): Input 50, expected output 0 (below min exchange)
   - Test 2 (above max): Input 600, expected output 750 (capped at max: 500 * 1.5)
   - Test 3 (within range): Input 200, expected output 300 (200 * 1.5)
4. **Forward Simulation - Capacity Constraint**:
   - Path: A → B → C
   - Capacities: A→B = 1000, B→C = 500
   - Input: 800 units
   - Expected output: 500 (limited by B→C capacity)
   - Verify `edgeRemainingCapacity[B→C]` reduced to 0
5. **Inverse Simulation - Simple Path** (reverse of forward test 1):
   - Target output: 490 units at C
   - Expected input: 500 units at A (490 + 10 commission)
6. **Inverse Simulation - Exchange Path** (reverse of forward test 2):
   - Target output: 400 units in eq 2 at C
   - Expected input: ~200 units in eq 1 at A (400 / 2.0)
7. **Inverse Simulation - Exchange Min/Max Limits**:
   - Same path as forward test 3
   - Test 1 (below min): Target output 50, expected input 0 (would require exchange below min)
   - Test 2 (above max): Target output 800, expected input based on max exchange capacity
   - Test 3 (within range): Target output 300, expected input 200 (300 / 1.5)
8. **Inverse Simulation - Capacity Constraint**:
   - Path: A → B → C with B→C capacity = 500
   - Target output: 600 units at C
   - Expected input: 0 (cannot achieve target due to capacity constraint)
9. **Commission Charge Once - Multi-Path**:
   - Path 1: A → B → C (commission at B = 10)
   - Path 2: A → B → D (shares node B)
   - Simulate path 1 with input 100, verify commission deducted
   - Simulate path 2 with input 100, verify commission NOT deducted (already in `appliedCommissions` set)
   - Verify total output matches expectation

## EstimateReceiveForPaymentAmountTransaction Test Suite

### Test Topology 1: Single-Equivalent Path (No Exchange)
- **Setup**:
  - Path: A → B → C in equivalent 1
  - Capacities: A→B = 1000, B→C = 800
  - Commission: B = 10 units in eq 1
  - Mock `retrievePaths()` to return this single path with `optimal_flow = 800`
- **Test 1**: Payment amount 510
  - Expected result: `200:500` (510 - 10 commission)
- **Test 2**: Payment amount 2000
  - Expected result: `200:790` (limited by B→C capacity 800, minus commission 10)
- **Test 3**: Payment amount 100
  - Expected result: `200:90` (100 - 10 commission)

### Test Topology 2: Cross-Equivalent Path with Exchange
- **Setup**:
  - Path: A (eq 1) → B (eq 1) → X [1→2, rate=2.0] → C (eq 2)
  - Capacities: A→B = 1000, B→X = 800, X→C = 1500
  - No commissions
  - Mock key: (C, sender_eq=1, receiver_eq=2)
  - Mock path with `optimal_flow = 800`
- **Test 1**: Payment amount 200 in eq 1
  - Expected result: `200:400` (200 * 2.0 exchange rate)
- **Test 2**: Payment amount 1000 in eq 1
  - Expected result: `200:1600` (limited by B→X capacity 800, then 800 * 2.0 = 1600)
- **Test 3**: Payment amount 400 in eq 1
  - Expected result: `200:800` (400 * 2.0)

### Test Topology 3: Multi-Path with Commissions
- **Setup** (from PRD 04 example):
  - Path 1: A → B → F (capacity: min(A→B=1000, B→F=500) = 500)
  - Path 2: A → B → C → F (capacity: min(A→B=1000, B→C=800, C→F=200) = 200)
  - Path 3: A → B → D → E → F (capacity: min(A→B=1000, B→D=700, D→E=300, E→F=900) = 300)
  - Commission at B: 10 units in equivalent 2002
  - All paths in same equivalent 2002 (sender_eq = receiver_eq = 2002)
  - Mock key: (F, sender_eq=2002, receiver_eq=2002)
  - Mock paths sorted by efficiency (path 1 first, path 3 last)
- **Test 1**: Payment amount 1000
  - Expected result: `200:990` (1000 input, commission 10 charged once across all paths)
  - Verify: `appliedCommissions` set contains only (B, 2002)
- **Test 2**: Payment amount 500
  - Expected result: `200:490` (500 input flows through path 1, commission 10 deducted once)
- **Test 3**: Payment amount 510
  - Expected result: `200:500` (510 - 10 commission on path 1 = 500)

### Test Topology 4: Exchange Min/Max Limits
- **Setup**:
  - Path: A (eq 1) → X [1→2, rate=1.5, min=100, max=500] → B (eq 2)
  - Capacity: A→X = 1000
  - Mock key: (B, sender_eq=1, receiver_eq=2)
  - Mock path with `optimal_flow = 1000`
- **Test 1**: Payment amount 50 in eq 1
  - Expected result: `200:0` (50 < min exchange amount 100, exchange skipped)
- **Test 2**: Payment amount 600 in eq 1
  - Expected result: `200:750` (capped at max exchange 500, output = 500 * 1.5 = 750)
- **Test 3**: Payment amount 200 in eq 1
  - Expected result: `200:300` (200 * 1.5, within min/max)

### Error Case Tests
- **No Cached Paths**: Mock `retrievePaths()` returns nullopt, verify response `462`
- **Expired Paths**: Mock TTL exceeded in manager, verify response `462`

## EstimatePaymentForReceiveAmountTransaction Test Suite

### Test Topology 1: Single-Equivalent Path (No Exchange)
- **Setup**: Same as receive estimation topology 1
- **Test 1**: Receive amount 500
  - Expected result: `200:510` (500 + 10 commission required)
- **Test 2**: Receive amount 900
  - Expected result: `412` (max deliverable ~790, insufficient for 900)
- **Test 3**: Receive amount 790
  - Expected result: `200:800` (790 + 10 commission, exactly at capacity)

### Test Topology 2: Cross-Equivalent Path with Exchange
- **Setup**: Same as receive estimation topology 2
- **Test 1**: Receive amount 400 in eq 2
  - Expected result: `200:200` (400 / 2.0 inverse exchange rate)
- **Test 2**: Receive amount 2000 in eq 2
  - Expected result: `412` (max deliverable 1600, insufficient for 2000)
- **Test 3**: Receive amount 1600 in eq 2
  - Expected result: `200:800` (1600 / 2.0 = 800, exactly at capacity)

### Test Topology 3: Multi-Path with Commissions
- **Setup**: Same as receive estimation topology 3
- **Test 1**: Receive amount 990
  - Expected result: `200:1000` (990 + 10 commission charged once)
  - Verify: `appliedCommissions` set contains only (B, 2002)
- **Test 2**: Receive amount 500
  - Expected result: `200:510` (500 + 10 commission, flows through path 1)
- **Test 3**: Receive amount 1000
  - Expected result: `412` (max deliverable 990, insufficient for 1000)

### Test Topology 4: Exchange Min/Max Limits
- **Setup**: Same as receive estimation topology 4
- **Test 1**: Receive amount 50 in eq 2
  - Expected result: `412` (50 / 1.5 ≈ 33 < min exchange 100, cannot deliver)
- **Test 2**: Receive amount 600 in eq 2
  - Expected result: `200:400` (600 / 1.5 = 400, within max exchange limit)
- **Test 3**: Receive amount 750 in eq 2
  - Expected result: `200:500` (750 / 1.5 = 500, exactly at max exchange)
- **Test 4**: Receive amount 800 in eq 2
  - Expected result: `412` (800 / 1.5 ≈ 533 > max exchange 500, cannot deliver)

### Error Case Tests
- **No Cached Paths**: Mock `retrievePaths()` returns nullopt, verify response `462`
- **Insufficient Paths**: Receive amount exceeds all path capacity, verify response `412`
- **Unexpected Error**: Force exception in estimation logic, verify response `401`

## Command Parsing Tests
1. **Valid Receive Estimation Command**:
   - Input: `GET:contractors/transactions/estimate/receive:12:127.0.0.1:2003:500:1:2`
   - Verify: contractor address = 127.0.0.1:2003, payment_amount = 500, sender_eq = 1, receiver_eq = 2
2. **Valid Payment Estimation Command**:
   - Input: `GET:contractors/transactions/estimate/payment:12:127.0.0.1:2003:1000:2:1`
   - Verify: contractor address = 127.0.0.1:2003, receive_amount = 1000, receiver_eq = 2, sender_eq = 1
3. **Invalid Command**: Malformed string, verify exception thrown or error handled gracefully

## DOD Criteria
- All ExchangePathsManager tests pass (store, retrieve, TTL, invalidation, thread safety, edge cases)
- All path simulation algorithm tests pass (forward, inverse, exchange limits, capacity constraints, commission charge once)
- All EstimateReceiveForPaymentAmountTransaction tests pass with 4 test topologies
- All EstimatePaymentForReceiveAmountTransaction tests pass with 4 test topologies
- All error case tests return correct error codes (401, 412, 462)
- Command parsing tests validate correct parameter extraction
- All tests execute successfully in `build-tests` without Docker dependencies
- Test coverage includes all edge cases mentioned in PRD
- No memory leaks detected during test execution

# Implementation Plan

## Step 1: Create Test Infrastructure
1. Create test file structure in `/home/mc/Personal/vtcp/vtcpd/test/`:
   - `test_exchange_paths_manager.cpp` for manager tests
   - `test_path_simulation.cpp` for algorithm tests
   - `test_estimation_transactions.cpp` for transaction tests
2. Set up mocking framework:
   - Mock `DateTime::now()` for TTL testing
   - Mock `ExchangePathsManager` for transaction tests
   - Mock `TopologyTrustLinesManager` for commission lookup
   - Create test fixture base class with common setup

## Step 2: Implement ExchangePathsManager Tests
1. **Store and Retrieve Test**:
   - Create 3 different `PathCacheKey` instances
   - Create mock `OptimalPathResult` vectors for each
   - Call `storePaths()` for each key
   - Call `retrievePaths()` for each key, assert results match
   - Call `retrievePaths()` for non-existent key, assert nullopt
2. **TTL Expiration Test**:
   - Mock `DateTime::now()` to return fixed timestamp T0
   - Store paths at T0
   - Advance mock to T0 + 601 seconds
   - Call `retrievePaths()`, assert nullopt (expired)
3. **Invalidate by Key Test**:
   - Store paths for keys K1, K2, K3
   - Call `invalidatePaths(K2)`
   - Retrieve K1 and K3, assert successful
   - Retrieve K2, assert nullopt
4. **Invalidate by Contractor Test**:
   - Store paths for contractor C with 3 different equivalent combinations
   - Store paths for contractor D
   - Call `invalidatePathsForContractor(C)`
   - Retrieve all C keys, assert nullopt
   - Retrieve D keys, assert successful
5. **Invalidate by Equivalent Test**:
   - Store paths with eq1 as sender, eq2 as receiver
   - Store paths with eq2 as sender, eq3 as receiver
   - Call `invalidatePathsForEquivalent(eq1)`
   - Retrieve paths with eq1, assert nullopt
   - Retrieve paths without eq1, assert successful
6. **Thread Safety Test**:
   - Create thread pool with 20 threads
   - 10 threads concurrently call `storePaths()` with unique keys
   - 10 threads concurrently call `retrievePaths()` with random keys
   - Run for 5 seconds
   - Assert no crashes, verify all stored keys retrievable after completion
7. **Edge Cases**:
   - Store empty path vector, retrieve, assert returns empty vector
   - Store with sender_eq == receiver_eq, retrieve, assert successful
   - Store at T0, retrieve at T0 + 600 (exactly at TTL), assert successful or expired (boundary condition)

## Step 3: Implement Path Simulation Algorithm Tests
1. **Forward Simulation Tests**:
   - For each test case (simple, exchange, limits, capacity):
     - Create mock `ExchangePath` with specified nodes, equivalents, exchange steps
     - Create mock `edgeRemainingCapacity` map initialized with path capacities
     - Create empty `appliedCommissions` set
     - Call `forwardSimulatePath()` with test input
     - Assert output matches expected value
     - Assert `edgeRemainingCapacity` updated correctly
     - Assert `appliedCommissions` updated correctly
2. **Inverse Simulation Tests**:
   - Mirror structure of forward tests
   - Use same mock paths but call `inverseSimulatePath()` with target output
   - Assert required input matches expected value
3. **Commission Charge Once Test**:
   - Create 2 paths sharing intermediate node B with commission
   - Initialize shared `appliedCommissions` set
   - Simulate path 1, assert commission deducted, set updated
   - Simulate path 2, assert commission NOT deducted, set unchanged
   - Verify total output across both paths

## Step 4: Implement EstimateReceiveForPaymentAmountTransaction Tests
1. **Topology Setup Helper**:
   - Create helper function `createMockPath()` to generate `OptimalPathResult` from topology specification
   - Create helper function `mockRetrievePaths()` to inject mock paths into transaction
2. **Test Topology 1 (Single-Equivalent)**:
   - Create mock path: A → B → C with specified capacities and commission
   - Mock `retrievePaths()` to return this path
   - Create transaction instance with payment amounts: 510, 2000, 100
   - Execute transaction `run()` method
   - Assert response matches expected: `200:500`, `200:790`, `200:90`
3. **Test Topology 2 (Cross-Equivalent)**:
   - Create mock path with exchange step [1→2, rate=2.0]
   - Mock `retrievePaths()` to return this path
   - Execute with payment amounts: 200, 1000, 400
   - Assert responses: `200:400`, `200:1600`, `200:800`
4. **Test Topology 3 (Multi-Path Commissions)**:
   - Create 3 mock paths sharing node B with commission
   - Mock `retrievePaths()` to return all 3 paths sorted by efficiency
   - Execute with payment amounts: 1000, 500, 510
   - Assert responses: `200:990`, `200:490`, `200:500`
   - Instrument code to verify `appliedCommissions` set contains only (B, 2002)
5. **Test Topology 4 (Exchange Limits)**:
   - Create mock path with exchange min/max limits
   - Execute with payment amounts: 50, 600, 200
   - Assert responses: `200:0`, `200:750`, `200:300`
6. **Error Case Tests**:
   - Mock `retrievePaths()` to return nullopt, assert response `462`

## Step 5: Implement EstimatePaymentForReceiveAmountTransaction Tests
1. **Test Topology 1-4**: Mirror structure of receive estimation tests
   - Use same mock path setups
   - Execute with receive amounts instead of payment amounts
   - Assert responses match expected results
   - For insufficient cases, assert response `412`
2. **Error Case Tests**:
   - Mock `retrievePaths()` to return nullopt, assert response `462`
   - Mock receive amount exceeding all path capacity, assert response `412`
   - Force exception in estimation logic, assert response `401`

## Step 6: Implement Command Parsing Tests
1. **Valid Receive Command Test**:
   - Create `EstimateReceiveForPaymentAmountCommand` with valid input string
   - Assert all parameters extracted correctly
2. **Valid Payment Command Test**:
   - Create `EstimatePaymentForReceiveAmountCommand` with valid input string
   - Assert all parameters extracted correctly
3. **Invalid Command Test**:
   - Create command with malformed input
   - Assert exception thrown or error handled

## Step 7: Build and Execute Tests
1. Add test files to `build-tests` CMake configuration
2. Build: `cmake --build build-tests`
3. Execute: Run test binaries produced in `build-tests/`
4. Verify all tests pass
5. Check for memory leaks using valgrind or similar tool (if available)

# Test Plan

## Complexity Level
Moderate

## Test Scope
This task IS the test implementation, so the "Test Plan" section describes validation of the tests themselves.

## Validation of Tests
- All test files compile successfully in `build-tests`
- All test cases execute and pass
- Test coverage includes all requirements from tasks 05-01 and 05-02
- Test outputs clearly indicate pass/fail status
- Test execution completes within reasonable time (< 5 minutes for full suite)

## Success Criteria
- 100% of implemented test cases pass
- No memory leaks detected in any test
- Test output clearly documents what each test validates
- All edge cases from PRD covered by at least one test
- Thread safety tests complete without crashes or data corruption

# Verification and Validation

## Architecture integrity
- Test suite validates architectural compliance of ExchangePathsManager (TTL pattern, thread safety)
- Tests verify simulation algorithms follow expected flow calculation semantics
- Transaction tests ensure proper integration with ExchangePathsManager
- Command tests validate adherence to command parsing conventions

## Security
- Tests verify no information leakage via error messages
- Thread safety tests ensure no race conditions under concurrent access
- Input validation tests ensure malformed commands handled gracefully

## Performance
- Test execution time provides baseline for performance expectations
- Thread safety tests validate concurrent access scalability
- Algorithm tests verify simulation complexity matches design (linear with path length)

## Scalability
- Multi-path tests verify handling of multiple paths (up to 100 per PRD target)
- Thread safety tests validate concurrent access patterns
- TTL tests verify cleanup scales with number of cached entries

## Reliability
- Error case tests verify all error codes returned correctly
- Edge case tests ensure no undefined behavior for boundary conditions
- Regression tests ensure existing max flow calculation behavior unchanged

## Maintainability
- Comprehensive test coverage documents expected behavior for future maintainers
- Test topology helpers enable easy creation of new test scenarios
- Clear test naming and structure facilitate debugging

## Cost
- Test execution in `build-tests` minimizes infrastructure costs (no Docker)
- Automated tests reduce manual testing effort
- Early bug detection via comprehensive tests reduces downstream costs

## Compliance
- Follows project policy: unit tests only, no Docker, build-tests execution
- Adheres to PRD test specifications (all 4 topologies implemented)
- Test structure follows existing test conventions in codebase

# Restrictions
- Commit changes only after successfully passing the tests (if they are provided for by the task)
