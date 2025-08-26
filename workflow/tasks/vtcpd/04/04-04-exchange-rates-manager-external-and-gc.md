# 04-04 – ExchangeRatesManager: external rates + TTL GC + tests

# Links
- [PRD](../../../prd/vtcpd/04-exchange-flow-calculation.md)

# Description
Implement storage of external exchange rates in `ExchangeRatesManager` (`mExternalExchangeRates`) and a TTL cleanup mechanism with nearest-event rescheduling. Provide unit coverage.

# Requirements and DOD
- Add `mExternalExchangeRates`: `map<pair<SerializedEquivalent, SerializedEquivalent>, vector<ExchangeRate>>`.
- Add cleanup timer: finds nearest `expiresAt`, sleeps until due, removes expired entries, erases empty vectors, and reschedules on insertion.
- Public API for inserting/updating external rates.
- Unit tests for insert/update/cleanup, including edge cases.
- Build and run in `build-tests`.

# Implementation Plan
1. Extend `ExchangeRatesManager` with the new map and API.
2. Implement a dedicated TTL cleanup timer/scheduler for external rates.
3. Integrate with TailManager processing of `ExchangeRatesMessage` (writes into external rates storage).
4. Document rescheduling behavior on new rate insert.

# Test Plan
- Unit tests:
  - Add one/multiple `ExchangeRate` with different `expiresAt`.
  - Trigger cleanup and verify removal of expired items and erasure of empty keys.
  - Reschedule timer after inserting a nearer expiration rate.
- Execute in `build-tests`.

# Verification and Validation
Architect validates timer behavior and data structure per PRD.

## Architecture integrity
Clear separation of local/external rates and their timers.

## Security
No authenticity checks (explicitly out of scope in PRD).

## Performance
Cleanup runs in O(total_entries) per event without excessive activity.

## Scalability
Stable under PRD’s NFR bounds.

## Reliability
Expired values are removed; no memory leaks.

## Maintainability
Simple API, tests, and clear invariants.

## Cost
No additional costs.

## Compliance
Complies with PRD.

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)


