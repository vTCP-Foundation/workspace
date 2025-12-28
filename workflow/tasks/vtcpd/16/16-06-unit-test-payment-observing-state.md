# 16-06 - Unit Test for PaymentObservingState

# Links
- [PRD](../../prd/vtcpd/16-payment-transaction-observing-states.md)
- [Previous task: 16-01-enum-stages-constant](16-01-enum-stages-constant.md)

# Description

This task creates a unit test file for the `PaymentObservingState` enum to verify that all enum values are correct and distinct. This ensures that the enum values remain stable across builds and that the new `Conflicted` state has the expected value.

The test verifies:
- `Init = 0`
- `Committed = 1`
- `ParticipantsVotesPresent = 2`
- `RejectedByObserving = 3`
- `Conflicted = 4`
- All values are distinct

# Requirements and DOD

## Requirements

1. **Create Test File**
   - Create `tests/unit/sqlite/PaymentObservingStateTest.cpp`
   - Use Google Test framework (consistent with existing tests)

2. **Test Cases**
   - Test that each enum value has the expected integer value
   - Test that all enum values are distinct

3. **CMake Integration**
   - Add test file to `tests/unit/CMakeLists.txt`

## Definition of Done

- [ ] Test file created at `tests/unit/sqlite/PaymentObservingStateTest.cpp`
- [ ] Test verifies `Init = 0`
- [ ] Test verifies `Committed = 1`
- [ ] Test verifies `ParticipantsVotesPresent = 2`
- [ ] Test verifies `RejectedByObserving = 3`
- [ ] Test verifies `Conflicted = 4`
- [ ] Test verifies all values are distinct
- [ ] Test added to CMakeLists.txt
- [ ] All tests pass

# Implementation Plan

## Step 1: Create Test File

File: `tests/unit/sqlite/PaymentObservingStateTest.cpp`

```cpp
#include <gtest/gtest.h>
#include <set>

#include "core/io/storage/interfaces/PaymentTransactionsHandler.h"

// Task 16-06: Unit tests for PaymentObservingState enum

class PaymentObservingStateTest : public ::testing::Test
{
protected:
    void SetUp() override {}
    void TearDown() override {}
};

// Test: Init enum value equals 0
TEST_F(PaymentObservingStateTest, InitValueEquals0)
{
    EXPECT_EQ(static_cast<int>(PaymentObservingState::Init), 0);
}

// Test: Committed enum value equals 1
TEST_F(PaymentObservingStateTest, CommittedValueEquals1)
{
    EXPECT_EQ(static_cast<int>(PaymentObservingState::Committed), 1);
}

// Test: ParticipantsVotesPresent enum value equals 2
TEST_F(PaymentObservingStateTest, ParticipantsVotesPresentValueEquals2)
{
    EXPECT_EQ(static_cast<int>(PaymentObservingState::ParticipantsVotesPresent), 2);
}

// Test: RejectedByObserving enum value equals 3
TEST_F(PaymentObservingStateTest, RejectedByObservingValueEquals3)
{
    EXPECT_EQ(static_cast<int>(PaymentObservingState::RejectedByObserving), 3);
}

// Test: Conflicted enum value equals 4
TEST_F(PaymentObservingStateTest, ConflictedValueEquals4)
{
    EXPECT_EQ(static_cast<int>(PaymentObservingState::Conflicted), 4);
}

// Test: All enum values are distinct
TEST_F(PaymentObservingStateTest, AllValuesAreDistinct)
{
    std::set<int> values;

    values.insert(static_cast<int>(PaymentObservingState::Init));
    values.insert(static_cast<int>(PaymentObservingState::Committed));
    values.insert(static_cast<int>(PaymentObservingState::ParticipantsVotesPresent));
    values.insert(static_cast<int>(PaymentObservingState::RejectedByObserving));
    values.insert(static_cast<int>(PaymentObservingState::Conflicted));

    // If all values are distinct, set size should equal number of enum values
    EXPECT_EQ(values.size(), 5);
}

// Test: Enum values form a continuous sequence from 0 to 4
TEST_F(PaymentObservingStateTest, ValuesFormContinuousSequence)
{
    EXPECT_EQ(static_cast<int>(PaymentObservingState::Init), 0);
    EXPECT_EQ(static_cast<int>(PaymentObservingState::Committed),
              static_cast<int>(PaymentObservingState::Init) + 1);
    EXPECT_EQ(static_cast<int>(PaymentObservingState::ParticipantsVotesPresent),
              static_cast<int>(PaymentObservingState::Committed) + 1);
    EXPECT_EQ(static_cast<int>(PaymentObservingState::RejectedByObserving),
              static_cast<int>(PaymentObservingState::ParticipantsVotesPresent) + 1);
    EXPECT_EQ(static_cast<int>(PaymentObservingState::Conflicted),
              static_cast<int>(PaymentObservingState::RejectedByObserving) + 1);
}
```

## Step 2: Update CMakeLists.txt

File: `tests/unit/CMakeLists.txt`

Find the section where test source files are listed and add:

```cmake
sqlite/PaymentObservingStateTest.cpp
```

The exact location depends on how the CMakeLists.txt is structured. Look for patterns like:
```cmake
set(UNIT_TEST_SOURCES
    # ... existing sources ...
    sqlite/PaymentObservingStateTest.cpp
)
```

Or if tests are added individually:
```cmake
add_executable(unit_tests
    # ... existing sources ...
    sqlite/PaymentObservingStateTest.cpp
)
```

## Step 3: Verify Include Path

Ensure the include path `"core/io/storage/interfaces/PaymentTransactionsHandler.h"` is correct relative to the test's include directories. Check existing tests for include patterns.

## Step 4: Build and Run Tests

```bash
# Build
make -j$(nproc)

# Run unit tests
./bin/unit_tests --gtest_filter="PaymentObservingState*"
```

## Step 5: Verify All Tests Pass

Expected output:
```
[==========] Running 7 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 7 tests from PaymentObservingStateTest
[ RUN      ] PaymentObservingStateTest.InitValueEquals0
[       OK ] PaymentObservingStateTest.InitValueEquals0
[ RUN      ] PaymentObservingStateTest.CommittedValueEquals1
[       OK ] PaymentObservingStateTest.CommittedValueEquals1
[ RUN      ] PaymentObservingStateTest.ParticipantsVotesPresentValueEquals2
[       OK ] PaymentObservingStateTest.ParticipantsVotesPresentValueEquals2
[ RUN      ] PaymentObservingStateTest.RejectedByObservingValueEquals3
[       OK ] PaymentObservingStateTest.RejectedByObservingValueEquals3
[ RUN      ] PaymentObservingStateTest.ConflictedValueEquals4
[       OK ] PaymentObservingStateTest.ConflictedValueEquals4
[ RUN      ] PaymentObservingStateTest.AllValuesAreDistinct
[       OK ] PaymentObservingStateTest.AllValuesAreDistinct
[ RUN      ] PaymentObservingStateTest.ValuesFormContinuousSequence
[       OK ] PaymentObservingStateTest.ValuesFormContinuousSequence
[----------] 7 tests from PaymentObservingStateTest
[==========] 7 tests from 1 test suite ran.
[  PASSED  ] 7 tests.
```

# Test Plan

**Complexity**: Simple

This task creates unit tests for enum values.

## Validation Approach

1. **Compilation Test**: Test file must compile without warnings
2. **Test Execution**: All tests must pass
3. **Code Review**: Verify test coverage is complete

## Test Coverage

| Enum Value | Test Coverage |
|------------|---------------|
| Init | Value equals 0 |
| Committed | Value equals 1 |
| ParticipantsVotesPresent | Value equals 2 |
| RejectedByObserving | Value equals 3 |
| Conflicted | Value equals 4 |
| All values | Distinctness check |
| All values | Continuous sequence check |

# Verification and Validation

## Architecture integrity
- Test follows existing unit test patterns in the project
- Uses Google Test framework consistently
- File placed in appropriate directory (sqlite/)

## Security
- N/A - Unit test only

## Performance
- N/A - Unit test with trivial runtime

## Scalability
- N/A - Unit test only

## Reliability
- Tests verify enum stability for database compatibility
- Catches accidental enum value changes

## Maintainability
- Clear test naming convention
- Each test focuses on single assertion
- Easy to extend for new enum values

## Cost
- N/A - No cost implications

## Compliance
- Follows existing test file conventions
- Uses established testing patterns

# Restrictions
- Commit changes only after all tests pass
- Do not modify the PaymentObservingState enum in this task (it should already be modified by Task 16-01)
