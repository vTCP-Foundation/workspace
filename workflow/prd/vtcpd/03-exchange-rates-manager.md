## [03] Exchange Rates Manager and Commands

## Document Information
- **Project Name**: vtcpd
- **Phase/Iteration**: PRD 03
- **Document Version**: 1.0
- **Date**: 2025-08-15
- **Author(s)**: Mykola Ilashchuk
- **Stakeholders**: Mykola Ilashchuk, Dima Chizhevsky
- **Status**: Active
- **Previous PRD**: 02 Sphincs+ Cryptography Implementation
- **Related Documents**: []

## Executive Summary
This iteration introduces directional exchange-rate management as a foundation for a future exchange operation. It adds an in-memory, TTL-based `ExchangeRatesManager`, command and transaction interfaces, and integration with core components. It enables nodes to publish and query rates, with automatic expiry and cleanup.

## Iteration Context
### Previous Iterations Summary
- Completed Features: Core payments, trust lines, history, and cryptography updates
- Lessons Learned: Follow existing patterns for timers and ResultsInterface formatting
- Technical Debt: None directly impacted
- User Feedback: Need for cross-equivalent exchanges

### Current State Analysis
- What's working well: Transaction/command infrastructure; timer-based delayed tasks
- Pain points identified: No mechanism to manage exchange rates
- Performance metrics: Not applicable yet for rates

## Problem Statement

### Background
Current system supports payments within single equivalents but lacks cross-equivalent exchange capability. Nodes need to declare and manage exchange rates as foundation for future exchange operations.

### Problem Description
- No mechanism for nodes to publish exchange rates between different equivalents
- Missing standardized interface for rate management and querying
- Need ephemeral rate storage with automatic cleanup to ensure freshness

### Overview
Introduce exchange-rate support as a foundational step toward a future "exchange" operation. This iteration delivers an in-memory exchange rates store with TTL-based expiry, CRUD commands and transactions, and wiring into the core system. It enables nodes to declare directional exchange rates between equivalents and provides read/write access via commands.

### Business Justification
- Enable a forthcoming cross-equivalent exchange operation by allowing nodes to publish and manage directional exchange rates.
- Provide a standard interface and serialization format so clients can set, query, and maintain rates.
- Ensure rates are ephemeral and refreshed frequently through TTL and auto-cleanup.

### Conditions of Satisfaction (CoS)
- A new `ExchangeRate` class exists in `src/core/rates/` with required fields, accessors, `update(...)`, TTL handling, and `Shared` typedef.
- A new `ExchangeRatesManager` exists in `src/core/rates/manager/` with:
  - Add-or-update, get-by-pair, list-all, delete-by-pair, delete-all APIs
  - Conversion function that multiplies by rate and applies base-10 shift with truncation for negative shifts, throwing `NotFoundError` if no rate, and `OverflowError` on overflow
  - Background expiry using a timer that deletes stale rates and reschedules next wakeup per earliest expiry
- Commands implemented under `src/core/interface/commands_interface/commands/rates/` to:
  - Set/update a rate
  - Get a rate by pair
  - List all rates
  - Delete a rate by pair
  - Clear all rates
- Transactions implemented under `src/core/transactions/transactions/rates/` and bound in `TransactionsManager`:
  - Set/delete commands return only `transactionUUID` and status code
  - Get/list commands serialize data to `ResultsInterface` using the standard tokens format
- `ExchangeRatesManager` is initialized in `Core` and injected into `TransactionsManager` via constructor (same pattern as other components).
- Unit tests for `ExchangeRate` and `ExchangeRatesManager` cover creation/update, TTL/expiry cleanup, CRUD, conversion logic, and overflow/NotFound behavior.

## Project Scope
### This Iteration's Scope
#### New Features/Enhancements
- Data model: directional exchange rates with TTL
- Manager with timer-driven expiry
- Commands, transactions, serialization via `ResultsInterface`
- Wiring into `Core` and `TransactionsManager`
- Unit tests (no integration/E2E in this iteration)

#### Bug Fixes & Technical Improvements
- None

#### Modifications to Existing Features
- Extend `TransactionsManager` constructor and processing to wire new commands/transactions

### Explicitly Out of Scope
- Actual "exchange" operation and flow coordination
- Enforcement of `min/max` amounts during conversion (stored only)
- Persistence to storage (rates are in-memory ephemeral)

### Dependencies from Previous Iterations
- Reuse command/transaction infrastructure
- Use existing time utilities and delayed-task timer patterns

### Future Roadmap Impact
- Provides the prerequisite for implementing cross-equivalent exchange flows in a future iteration

### Dependencies
- Internal time utilities: `src/core/common/time/TimeUtils.h` (`DateTime`, `utc_now()`, `Duration`)
- Boost.Asio timers (`as::steady_timer`) – follow project’s delayed task style
- `ResultsInterface` for command results formatting and FIFO writing
- Exceptions: `OverflowError`, `NotFoundError` (existing exception types)

### Assumptions and Constraints
- Directional rates: A→B is distinct from B→A.
- TTL: 300000 ms (5 minutes) as a static constant in `ExchangeRatesManager`.
- Expiry: `mExpiresAt` is calculated by `ExchangeRatesManager` as `utc_now() + TTL` on create and on update.
- Shift semantics: base-10 shift; negative shift uses truncation (no rounding).
- Overflow: multiplication or shifting overflow throws `OverflowError`.
- `mMinExchangeAmount`/`mMaxExchangeAmount` are stored only; not enforced in this iteration.
- Manager operates within the existing single-threaded `io_context` model; no additional threading.

### Task List
- Refer to: `workspace/workflow/tasks/vtcpd/03/` (to be created). Each task will follow repository policy and include validation and test plans.

## Technical Specifications

### Architecture Evolution
- Add `src/core/rates/` and `src/core/rates/manager/` modules.
- Initialize `ExchangeRatesManager` in `Core`, inject into `TransactionsManager` via constructor parameter and member.
- Add command classes under `src/core/interface/commands_interface/commands/rates/`.
- Add transactions under `src/core/transactions/transactions/rates/` and bind them in `TransactionsManager::processCommand` similarly to existing patterns.

### Data Requirements
#### Data Models
- `ExchangeRate` (in `src/core/rates/`):
  - Private fields:
    - `SerializedEquivalent mEquivalentFrom`
    - `SerializedEquivalent mEquivalentTo`
    - `TrustLineAmount mExchangeRate`
    - `int16_t mExchangeRateShift`
    - `DateTime mExpiresAt`
    - `TrustLineAmount mMinExchangeAmount`
    - `TrustLineAmount mMaxExchangeAmount`
  - Constructor: takes all fields, including `mExpiresAt`.
  - Accessors: getters for all fields (style consistent with `TrustLine`)
  - `update(exchangeRate, exchangeRateShift, minExchangeAmount, maxExchangeAmount, expiresAt)`: updates fields, including `mExpiresAt`.
  - `typedef shared_ptr<ExchangeRate> Shared`

### Manager
- `ExchangeRatesManager` (in `src/core/rates/manager/`):
  - Ctor: `(as::io_context &ioCtx, Logger &logger)`
  - Static constant: `kTTLMilliseconds = 300000` (5 minutes)
  - Private map: `map<pair<SerializedEquivalent, SerializedEquivalent>, ExchangeRate::Shared> mExchangeRates`
  - Timer: `unique_ptr<as::steady_timer> mExpiryTimer`
  - Methods:
    - `addOrUpdate(equivFrom, equivTo, exchangeRate, exchangeRateShift, minExchangeAmount, maxExchangeAmount)`
      - If pair exists, call `update()` with new values and a refreshed expiry timestamp; otherwise, create a new `ExchangeRate` instance and insert it. In both cases, update the expiry schedule.
    - `get(equivFrom, equivTo) -> ExchangeRate::Shared` (may return `nullptr` if absent)
    - `list() -> vector<ExchangeRate::Shared>` (copy values)
    - `remove(equivFrom, equivTo)` (no-op if absent)
    - `clear()`
    - `calculateConvertedAmount(equivFrom, equivTo, amountInEquivFrom) -> TrustLineAmount`
      - Lookup rate or throw `NotFoundError`
      - Compute: `amountInEquivFrom * mExchangeRate` then apply base-10 shift `mExchangeRateShift` (truncate toward zero for negative); throw `OverflowError` on overflow
    - Expiry maintenance:
      - On first insertion or after cleanup, set timer to the earliest `mExpiresAt`
      - On timer fire: remove all expired entries (`utc_now() >= mExpiresAt`), then reschedule to next soonest expiry if any
  - Timer style: use `expires_after(chrono::milliseconds(delay))` and `async_wait(...)` as in `TopologyCacheUpdateDelayedTask` and `GatewayNotificationAndRoutingTablesDelayedTask`.

### Integration Requirements
#### New Integrations
- New Rates commands under `src/core/interface/commands_interface/commands/rates/`
- New Rates transactions under `src/core/transactions/transactions/rates/`
- Integration with `TransactionsManager` and `ResultsInterface`

### Commands
- Directory: `src/core/interface/commands_interface/commands/rates/`
- Identifiers:
  - Set/Update: `SET:RATE`
  - Get one: `GET:RATE`
  - List all: `LIST:RATES`
  - Delete one: `DEL:RATE`
  - Delete all: `CLEAR:RATES`
- Payloads:
  - `SET:RATE`: `equivalentFrom, equivalentTo, exchangeRate, exchangeRateShift, minExchangeAmount, maxExchangeAmount`
  - `GET:RATE`: `equivalentFrom, equivalentTo`
  - `LIST:RATES`: none
  - `DEL:RATE`: `equivalentFrom, equivalentTo`
  - `CLEAR:RATES`: none
- Follow existing command parsing style (see trust lines commands) and reuse `kTokensSeparator`/`kCommandsSeparator` for serialization.

### Transactions
- Directory: `src/core/transactions/transactions/rates/`
- Transactions and behavior:
  - `SetExchangeRateTransaction`: validates inputs, invokes manager add/update; returns only `transactionUUID` and status code
  - `GetExchangeRateTransaction`: returns serialized single-rate tuple via `ResultsInterface`
  - `ListExchangeRatesTransaction`: returns count and repeated tuples
  - `RemoveExchangeRateTransaction`: removes pair; returns only `transactionUUID` and status code
  - `ClearExchangeRatesTransaction`: clears all; returns only `transactionUUID` and status code
- Serialization format for one rate tuple (order and tokens consistent across get/list):
  - `equivalentFrom`
  - `equivalentTo`
  - `exchangeRate`
  - `exchangeRateShift`
  - `minExchangeAmount`
  - `maxExchangeAmount`
  - `expiresAtUnixMicroseconds` (from `DateTime` to Unix epoch microseconds; match history serialization approach)

### Integration and Wiring
- `Core`:
  - Initialize `unique_ptr<ExchangeRatesManager>` (pass `mIOCtx` and `*mLog`)
  - Pass reference/pointer to `TransactionsManager` in its constructor
- `TransactionsManager`:
  - Extend constructor signature to accept `ExchangeRatesManager *`
  - Bind new rates commands in `processCommand(...)` following existing patterns
  - For get/list, use `mResultsInterface->writeResult(...)` with serialized payload

### Non-Functional Requirements
- Performance: O(1) average for lookup/update; timer maintenance proportional to number of distinct pairs
- Security: no external IO; in-memory only
- Reliability: timer-driven expiry ensures stale rates are removed; errors reported via existing exception/result patterns
- Maintainability: code and directory structure mirrors existing managers/transactions/commands patterns

### Migration Requirements
- None (new, isolated functionality)

## Implementation Plan

### This Iteration Timeline
- 1 iteration (single sprint)

### Iteration Milestones
| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Data model & manager | TBD | Implement `ExchangeRate` and `ExchangeRatesManager` with expiry | Time utils | Low |
| Commands & transactions | TBD | Implement commands and transactions, bind in manager | ResultsInterface | Medium |
| Wiring | TBD | Initialize in `Core`, inject into `TransactionsManager` | Prior steps | Low |
| Unit tests | TBD | Cover classes and expiry | Prior steps | Low |

### Dependencies on Other Teams/Projects
- None

### Integration Points with Previous Work
- Reuse of `ResultsInterface` and FIFO protocol
- Reuse of time and timer patterns from delayed tasks

### Resource Requirements
- Developer time for implementation and tests

## Risk Management

### Technical Risks
| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| Timer scheduling drift | Low | Medium | Use `utc_now()` and earliest expiry rescheduling pattern |
| Overflow in arithmetic | Medium | Medium | Validate and throw `OverflowError`; unit test extreme cases |
| Serialization mismatches | Medium | Low | Mirror existing tokens format and add tests |

### Business Risks
- None significant; feature is additive and internal until used by higher-level exchange flow

## Testing Strategy

### Testing Approach for This Iteration
- Unit tests only for this iteration (per project conventions). No Docker.

#### New Feature Testing (Unit)
- `ExchangeRateTest`:
  - `testConstructorSetsAllFieldsCorrectly()`: Verify all fields, including mExpiresAt, are set from constructor arguments.
  - `testUpdateSetsAllFieldsCorrectly()`: Verify update() correctly changes all fields, including mExpiresAt.
  - `testAccessorsReturnCorrectValues()`: Verify all getter methods return current values.
- `ExchangeRatesManagerTest`:
  - `testAddOrUpdateCreatesNewRateWithCorrectExpiry()`: Add new rate, verify it exists and its mExpiresAt is correctly set to utc_now() + TTL.
  - `testAddOrUpdateUpdatesExistingRateAndRefreshesExpiry()`: Update a rate, verify values changed and mExpiresAt is refreshed.
  - `testGetReturnsCorrectRate()`: Retrieve rate by pair, verify fields match
  - `testGetReturnsNullptrForMissingPair()`: Query non-existent pair, verify nullptr
  - `testListReturnsAllRates()`: Add multiple rates, verify list() returns all
  - `testRemoveDeletesSpecificPair()`: Remove one pair, verify others remain
  - `testClearRemovesAllRates()`: Clear all, verify empty map
  - `testCalculateConvertedAmountPositiveShift()`: 1000 * 1.5 with shift +2 = 150000
  - `testCalculateConvertedAmountNegativeShift()`: 123456 * 1.0 with shift -2 = 1234 (truncated)
  - `testCalculateConvertedAmountZeroShift()`: 1000 * 2.5 with shift 0 = 2500
  - `testCalculateThrowsNotFoundError()`: Query missing pair, verify NotFoundError thrown
  - `testCalculateThrowsOverflowError()`: MAX_TRUSTLINE_AMOUNT * 2 throws OverflowError
  - `testTimerExpiryRemovesStaleRates()`: Add rate, advance time, verify removal
  - `testTimerReschedulesAfterCleanup()`: Multiple rates with different expiry, verify timer reschedules to earliest

### Quality Gates
- All new unit tests must pass in `build-tests` environment
- Linter passes with no new warnings/errors

## Deployment & Release Strategy
- Minor feature release behind command interface; no config migration
- Rollout: standard deployment; safe to include as it is inactive until commands are used

## Success Metrics & Monitoring
### Iteration-Specific KPIs
- Command execution success rate > 99%
- Rate expiry cleanup completes within 1 second of TTL
- Lookup latency < 10ms for 1000 rate pairs

### Monitoring Plan
- Add debug logs for add/update/delete and timer rescheduling
- Track command result statuses via existing logging
- Successful execution of new commands and transactions in dev/test
- No crashes due to overflow or timer logic

## Appendices
### Glossary
- TTL: Time to Live for an exchange rate

### References
- Time utilities: `src/core/common/time/TimeUtils.h`
- Delayed tasks examples: `src/core/delayed_tasks/`
- Results formatting examples: trust lines and history transactions

### Document History
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | TBD | TBD | Initial draft for exchange rates manager | PRD 03 |


