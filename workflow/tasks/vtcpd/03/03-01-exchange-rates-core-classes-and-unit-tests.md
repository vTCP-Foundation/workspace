# 03-01 - Implement ExchangeRate and ExchangeRatesManager classes with unit tests

# Links
- [PRD](../../../prd/vtcpd/03-exchange-rates-manager.md)

# Description
Implement exchange rate management modules: the `ExchangeRate` class and the `ExchangeRatesManager` with TTL-based expiry and background cleanup, and cover them with unit tests as specified in the PRD.

# Requirements and DOD
- **Architecture and files**:
  - Create directories `src/core/rates/` and `src/core/rates/manager/`.
  - Implement `src/core/rates/ExchangeRate.{h,cpp}` according to the PRD.
  - Implement `src/core/rates/manager/ExchangeRatesManager.{h,cpp}` according to the PRD.
- **ExchangeRate**:
  - Fields: `SerializedEquivalent mEquivalentFrom`, `SerializedEquivalent mEquivalentTo`, `TrustLineAmount mExchangeRate`, `int16_t mExchangeRateShift`, `DateTime mExpiresAt`, `TrustLineAmount mMinExchangeAmount`, `TrustLineAmount mMaxExchangeAmount`.
  - Constructor accepts all fields (including `mExpiresAt`).
  - Getters for all fields (consistent with existing classes, e.g., `TrustLine`).
  - Method `update(exchangeRate, exchangeRateShift, minExchangeAmount, maxExchangeAmount, expiresAt)` updates the respective fields.
  - `typedef shared_ptr<ExchangeRate> Shared`.
- **ExchangeRatesManager**:
  - Constructor: `(as::io_context &ioCtx, Logger &logger)`.
  - Constant: `kTTLMilliseconds = 300000` (5 minutes) as `static const`.
  - Storage: `map<pair<SerializedEquivalent, SerializedEquivalent>, ExchangeRate::Shared> mExchangeRates`.
  - Timer: `unique_ptr<as::steady_timer> mExpiryTimer` scheduled to the earliest expiry.
  - Methods: `addOrUpdate(...)`, `get(...)`, `list()`, `remove(...)`, `clear()`, `calculateConvertedAmount(...)`.
  - Conversion semantics: multiply `amount * rate` then apply base-10 `shift` (negative shift truncates toward zero), throw `OverflowError` on overflow, `NotFoundError` if rate is absent.
  - TTL maintenance: on insert/update refresh `mExpiresAt` (`utc_now() + TTL`); on timer fire remove expired entries and reschedule to the earliest remaining `mExpiresAt`.
  - Timer style mirrors existing delayed tasks (`expires_after(...)`, `async_wait(...)`).
- **Logging**: add debug logs for add/update/remove/clear and timer rescheduling.
- **Unit tests**:
  - Add tests under `tests/unit/rates/` for `ExchangeRate` and `ExchangeRatesManager`.
  - Cover scenarios from the PRD “Testing Strategy”, including TTL cleanup, CRUD, conversion (positive/negative/zero shift), `NotFoundError`, `OverflowError`, edge cases, and timer rescheduling.
- **Definition of Done**:
  - All new tests pass in the `build-tests` environment.
  - Code compiles cleanly with no new warnings/errors and matches project style.
  - No architectural regressions; changes localized to new modules.

# Implementation Plan
1. Create directories `src/core/rates/` and `src/core/rates/manager/`.
2. Implement `ExchangeRate` per PRD: fields, getters, constructor, `update(...)`, and `Shared` typedef.
3. Implement `ExchangeRatesManager`:
   - Store in `std::map` keyed by (`equivFrom`, `equivTo`).
   - TTL: in `addOrUpdate` compute `expiresAt = utc_now() + TTL`.
   - Timer: schedule to the earliest `expiresAt`; in `async_wait` remove expired and reschedule.
   - Conversion: multiply then apply decimal shift with truncation for negatives; ensure overflow checks.
   - Logging: key actions and rescheduling.
4. Add unit tests:
   - `tests/unit/rates/ExchangeRateTest.cpp` — constructor/getters/`update`.
   - `tests/unit/rates/ExchangeRatesManagerTest.cpp` — CRUD, TTL, conversion, errors, rescheduling.
5. Ensure tests are isolated, no external IO, and run within a single `io_context`.

# Test Plan
- Build and run tests strictly in `build-tests` (project convention):
  - `mkdir -p build-tests && cd build-tests`
  - `cmake .. -DCMAKE_BUILD_TYPE=Debug`
  - `make -j`
  - `ctest -V` or with filter `-R rates`
- Verify all new tests pass and cover the PRD scenarios.
- Verify no new linter/compiler warnings.

# Verification and Validation
## Architecture integrity
- New modules (`src/core/rates/`, `src/core/rates/manager/`) do not violate existing dependencies; operate within the single-threaded `io_context`.
- Timer is implemented in the style of existing delayed tasks.

## Security
- In-memory only; no external IO; no access policy changes.

## Performance
- Lookup/update operations are amortized O(1) (map insert/find); TTL maintenance scales linearly with number of pairs.

## Scalability
- Straightforward future swap to `unordered_map` or specialized structures if needed.

## Reliability
- TTL cleanup ensures data freshness; overflow/missing-rate conditions are signaled via standard exceptions.

## Maintainability
- Code follows existing manager/timer patterns; constants centralized.

## Cost
- Minimal; memory only and lightweight background timer maintenance.

## Compliance
- Conforms to the PRD and project policies; unit tests only, no Docker.

# Restrictions
- make commits only after all unit tests pass in `build-tests` and all checks in this document are satisfied.

