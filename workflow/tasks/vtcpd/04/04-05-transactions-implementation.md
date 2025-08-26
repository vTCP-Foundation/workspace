# 04-05 – Transactions: new and modified

# Links
- [PRD](../../../prd/vtcpd/04-exchange-flow-calculation.md)

# Description
Implement new transactions: `BaseCollectTopologyForExchangeTransaction`, `InitiateMaxFlowExchangeCalculationTransaction` (+ `InitiateMaxFlowExchangeCalculationCommand` and wiring), `CollectTopologyForExchangeTransaction`, `ReceiveMaxFlowCalculationForExchangeOnTargetTransaction`. Modify: `MaxFlowCalculationSourceFstLevelTransaction`, `MaxFlowCalculationSourceSndLevelTransaction`, `MaxFlowCalculationTargetFstLevelTransaction`, `MaxFlowCalculationTargetSndLevelTransaction`.

# Requirements and DOD
- New transactions respect semantics of `mEquivalent` (receiver currency) and `exchangeEquivalents` (payer set).
- `InitiateMaxFlowExchangeCalculationCommand` implemented and registered with the command interface.
- Forwarding of `exchangeEquivalents` in Source/Target Fst/Snd Level transactions per PRD.
- `ExchangeRatesMessage` processing added via `fillRates()` in the exchange-aware base class.
- Backward compatibility: legacy single-equivalent transactions send empty `exchangeEquivalents`.
- Unit tests for behavioral scenarios with mocked network/managers.
- Build and run tests in `build-tests`.

# Implementation Plan
1. Create `BaseCollectTopologyForExchangeTransaction` (use `mEquivalentsSubsystemsRouter`, add `fillRates()`).
2. Implement `InitiateMaxFlowExchangeCalculationTransaction` to orchestrate topology collection requests and flow solving.
3. Implement `InitiateMaxFlowExchangeCalculationCommand` and wire it in command/transaction factories.
4. Implement `CollectTopologyForExchangeTransaction` to send messages per PRD semantics.
5. Implement `ReceiveMaxFlowCalculationForExchangeOnTargetTransaction` (first level for `mEquivalent`, preserving exchange set).
6. Modify existing Level transactions to forward `exchangeEquivalents` and send `ExchangeRatesMessage`.

# Test Plan
- Unit tests with mocks:
  - Source Fst Level dispatch: `equivalent = exchangeEquivalents[i]`, `exchangeEquivalents = { mEquivalent }`.
  - Target Fst Level dispatch: `equivalent = mEquivalent`, `exchangeEquivalents = <payer-set>`.
  - Rate collection/accumulation via `fillRates()`.
- Execute in `build-tests`.

# Verification and Validation
Confirmed PRD semantics and backward compatibility.

## Architecture integrity
Aligned with existing transaction architecture and equivalents router.

## Security
No changes.

## Performance
Minor overhead to collection logic; within NFR.

## Scalability
Supports up to 10 payer equivalents.

## Reliability
Critical scenarios covered by tests.

## Maintainability
Reuses existing patterns and base classes.

## Cost
No additional costs.

## Compliance
Complies with PRD and repository policy.

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)


