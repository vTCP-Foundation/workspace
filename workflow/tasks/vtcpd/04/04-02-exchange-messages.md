# 04-02 – Exchange messages (new and modified)

# Links
- [PRD](../../../prd/vtcpd/04-exchange-flow-calculation.md)

# Description
Add new messages `ExchangeRatesMessage`, `InitiateMaxFlowForExchangeCalculationMessage` and modify existing max-flow messages by adding `vector<SerializedEquivalent> exchangeEquivalents` with backward compatibility.

# Requirements and DOD
- New messages implemented with full serialization/deserialization.
- Modified: `MaxFlowCalculationSourceFstLevelMessage`, `MaxFlowCalculationSourceSndLevelMessage`, `MaxFlowCalculationTargetFstLevelMessage`, `MaxFlowCalculationTargetSndLevelMessage` (new `exchangeEquivalents` field).
- Backward compatibility: empty `exchangeEquivalents` reproduces legacy behavior.
- `ExchangeRatesMessage` carries `vector<ExchangeRate>` (see `src/core/rates/ExchangeRate.h`).
- Related transactions updated to pass/forward the field consistently.
- Unit tests for (de)serialization and backward compatibility.
- Build and run tests in `build-tests`.

# Implementation Plan
1. Add message classes under `src/core/network/messages/max_flow_calculation/`.
2. Update base class (`MaxFlowCalculationMessage`) and descendants.
3. Implement (de)serialization of the new field and messages.
4. Verify integration with existing transactions (forwarding `exchangeEquivalents`).

# Test Plan
- Unit tests for encoding/decoding `exchangeEquivalents` (empty/non-empty).
- Unit tests for `ExchangeRatesMessage` with a vector of `ExchangeRate`.
- Backward compatibility tests: empty `exchangeEquivalents` keeps legacy behavior unchanged.
- Execute in `build-tests`.

# Verification and Validation
Confirm all documented fields exist and work per PRD; verified forwarding across related transactions.

## Architecture integrity
Protocol extended without breaking compatibility; fields match PRD.

## Security
No changes to transport security; message size remains controlled.

## Performance
Minimal overhead for size/parsing; validated by unit tests.

## Scalability
Support up to 10 equivalents in `exchangeEquivalents` per NFR.

## Reliability
Tests cover critical scenarios (empty/filled vector).

## Maintainability
Code follows existing message patterns.

## Cost
No additional costs.

## Compliance
Complies with PRD and repository policy.

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)


