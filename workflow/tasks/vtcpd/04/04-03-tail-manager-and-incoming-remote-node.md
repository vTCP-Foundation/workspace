# 04-03 – TailManager/IncomingRemoteNode (new exchange rates tail) + tests

# Links
- [PRD](../../../prd/vtcpd/04-exchange-flow-calculation.md)

# Description
Add a new `mExchangeRatesTail` in `TailManager` for `ExchangeRatesMessage`, integrate routing in `IncomingRemoteNode::tryCollectNextPacket`, ensure backward compatibility, and cover with unit tests.

# Requirements and DOD
- `mExchangeRatesTail` created and documented in `TailManager`.
- `IncomingRemoteNode::tryCollectNextPacket` enqueues `ExchangeRatesMessage` into `mExchangeRatesTail`.
- Existing tails remain unaffected (no regressions).
- Define queue limits/TTL behavior per PRD (at minimum: no overflow under nominal load).
- Unit tests for routing and isolation of the new tail.
- Build and run tests in `build-tests`.

# Implementation Plan
1. Add `mExchangeRatesTail` field in `TailManager`.
2. Extend `IncomingRemoteNode::tryCollectNextPacket` to recognize `ExchangeRatesMessage`.
3. Ensure `IncomingMessagesHandler` reads from the new tail where required by transactions.
4. Update TailManager comments/documentation.

# Test Plan
- Unit tests:
  - Insert/read `ExchangeRatesMessage` to/from `mExchangeRatesTail`.
  - Routing from `IncomingRemoteNode::tryCollectNextPacket`.
  - Verify no side effects on other tails.
- Execute in `build-tests`.

# Verification and Validation
Architect verifies no regressions and correct routing per PRD.

## Architecture integrity
New tail integrated into TailManager’s queue model without breaking contracts.

## Security
No changes to security protocols.

## Performance
Minimal overhead compared to existing tails.

## Scalability
Stable behavior as message volume grows.

## Reliability
Queues remain functional under mixed workloads.

## Maintainability
Code and tests follow existing patterns.

## Cost
No additional costs.

## Compliance
Complies with PRD and repository policy.

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)


