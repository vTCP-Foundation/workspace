# 12-08 - Unit Tests: Core Components

# Links
- [PRD](../../../prd/vtcpd/12-observer-ambiguous-transaction-handling.md)
- [Implementation task 1: 12-01](12-01-observing-payment-claim-class.md)
- [Implementation task 2: 12-02](12-02-claim-storage-map.md)
- [Implementation task 3: 12-03](12-03-periodic-timer-processing.md)

# Description
Implement unit tests for ObservingPaymentClaim class. These tests verify the fundamental data structure, constructor initialization, status management, and getter/setter methods.

Tests follow project patterns using GoogleTest framework, built in `build-tests` directory, located in `tests/unit/observing/`.

**Note**: Tests for ObservingHandler (claim map, timer, processing) require integration testing infrastructure (IOCtx, StorageHandler, RPC communication) and are not suitable for unit testing. These will be covered in separate integration test tasks.

# Requirements and DOD

## Functional Requirements
1. Create test file: `tests/unit/observing/ObservingPaymentClaimTest.cpp`
2. Update `tests/unit/CMakeLists.txt` to include new test file
3. All tests must use GoogleTest (gtest)
4. All tests must compile and run in `build-tests`
5. All tests must pass (100% pass rate)

## Test Coverage Requirements

### ObservingPaymentClaimTest.cpp
- Test: Constructor initializes all fields correctly
  - Create claim with all parameters
  - Verify transactionUUID matches
  - Verify maxBlockNumberForClaiming matches
  - Verify participantsPublicKeys map size and contents
  - Verify publicKey pointer matches
  - Verify signature pointer matches
- Test: Constructor sets status to NoInfo
  - Create claim
  - Verify status() returns ClaimStatus::NoInfo
- Test: Status enum values correct
  - Verify NoInfo == 0
  - Verify Observing == 1
  - Verify ParticipantsVotesPresent == 2
  - Verify RejectedByObserving == 3
  - Verify Done == 4
- Test: Getters return correct values
  - Create claim with known values
  - Call each getter
  - Verify returned values match constructor inputs
- Test: setStatus updates status correctly
  - Create claim (initial status NoInfo)
  - Call setStatus(Observing)
  - Verify status() returns Observing
  - Call setStatus(Done)
  - Verify status() returns Done

## Definition of Done
- [ ] Test file created: tests/unit/observing/ObservingPaymentClaimTest.cpp
- [ ] ObservingPaymentClaimTest.cpp with 5 test cases
- [ ] All tests use GoogleTest framework (TEST/TEST_F macros)
- [ ] All tests compile without errors or warnings
- [ ] All tests pass when executed (100% pass rate)
- [ ] Tests added to build system (CMakeLists.txt updated)
- [ ] Tests can be run via: `./build-tests/bin/unit_tests --gtest_filter=ObservingPaymentClaim*`

# Implementation Plan

## Step 1: Create ObservingPaymentClaimTest.cpp
1. Create directory: `tests/unit/observing/`
2. Create file: `tests/unit/observing/ObservingPaymentClaimTest.cpp`
3. Add includes:
   - `#include <gtest/gtest.h>`
   - `#include "../../../src/core/observing/ObservingPaymentClaim.h"`
   - `#include "../../../src/core/crypto/sphincskeys.h"`
   - `#include <boost/uuid/uuid_generators.hpp>`
4. Create test fixture: `class ObservingPaymentClaimTest : public ::testing::Test`
5. Implement 5 test cases per requirements

## Step 2: Update Build System
1. Update `tests/unit/CMakeLists.txt` to include ObservingPaymentClaimTest.cpp
2. Verify test file added to unit_tests executable

## Step 3: Build and Run Tests
1. Configure and build project in debug mode (build-debug)
2. Configure and build tests (build-tests)
3. Run tests: `./build-tests/bin/unit_tests --gtest_filter=ObservingPaymentClaim*`
4. Verify all 5 tests pass
5. Fix any failures if needed

## Step 4: Document Test Results
1. Capture test output
2. Verify 100% pass rate (5/5 tests passing)

# Test Plan

**Test Type**: Simple task - unit test implementation for ObservingPaymentClaim class

**Test Scope**: ObservingPaymentClaim class only (basic data structure with getters/setters).

**Validation Approach**:
- All tests compile successfully
- All tests execute without crashes
- All tests pass (green status)
- Test coverage confirmed via test case count

**Success Criteria**:
- 5 test cases implemented
- 100% pass rate
- No compilation warnings
- Tests run in < 5 seconds

# Verification and Validation

## Architecture integrity
- Tests follow project GoogleTest patterns
- Test files located in appropriate directory structure
- Tests use real objects (no unnecessary mocking)
- Test fixtures encapsulate setup logic

## Security
- No security validation required for unit tests

## Performance
- Tests execute quickly (< 30s total)
- Timer tests use shorter period (10s) to avoid long waits
- No performance-critical code in test assertions

## Scalability
- Tests scale to additional test cases easily
- Fixture-based design enables test expansion

## Reliability
- Tests deterministic (repeatable results)
- No external dependencies (network, filesystem)
- Exception tests verify error handling

## Maintainability
- Clear test names describe what is tested
- Test fixtures reduce code duplication
- Comments explain non-obvious test logic
- Consistent style across test files

## Cost
- No cost validation required

## Compliance
- Tests follow project testing guidelines
- GoogleTest framework standard
- Located in tests/unit/ per project structure

# Restrictions
- Commit changes only after all tests pass
- Do not test RPC methods (covered in 12-09)
- Do not test signal integration (covered in 12-10)
- Keep tests focused on core components only
- Use real ObservingPaymentClaim and ObservingHandler objects (no mocks)
