# 04-07 - Commissions on Intermediate Nodes

# Links
- [PRD](../../../prd/vtcpd/04-exchange-flow-calculation.md)
- [Topology Collection Protocol](../../../architecture/vtcpd/protocols/topology-collection-protocol.md)

# Description
Introduce fixed per-node commissions during topology collection and max-flow/LP calculation for exchange-enabled flows. Each node may define a fixed commission per equivalent in `conf.json`. Nodes include their commission (for the specific equivalent of the topology snapshot) in topology result messages. The payer (S) caches received commissions with TTL and uses them in flow computation.

# Requirements and DOD
- Commission definition:
  - Fixed fee per transit, independent of payment amount.
  - Represented by `Commission { amount: uint64 }` per `SerializedEquivalent`.
- New manager:
  - `CommissionsManager` holds `map<SerializedEquivalent, Commission>`.
  - Reads from `conf.json` on node startup.
  - Provides:
    - `getCommission(SerializedEquivalent) -> optional<Commission>`
    - `applyCommission(TrustLineAmount in) -> TrustLineAmount out` subtracting fixed fee; clamp to zero if `in < fee`.
  - TTL constant: `kCommissionsTTLSeconds = 300` (for doc symmetry; node-held commissions are static until config reload, payer-side cached entries use TTL; see below).
- Message protocol updates:
  - `ResultMaxFlowCalculationMessage` and `ResultMaxFlowCalculationGatewayMessage` include optional commission for the message's `equivalent` when present on the sending node.
  - Only the following transactions attach commissions: `MaxFlowCalculationSourceFstLevelTransaction`, `MaxFlowCalculationSourceSndLevelTransaction`, `MaxFlowCalculationTargetFstLevelTransaction`, `MaxFlowCalculationTargetSndLevelTransaction`.
- Payer-side caching (S):
  - Extend `TopologyTrustLinesManager` with a map keyed by `<ContractorID, SerializedEquivalent>` to value `<Commission, expiredAt>`.
  - On receiving a commission with `amount > 0`:
    - If entry absent, insert and set `expiredAt = now() + 300s`.
    - If entry exists, update `Commission` and set `expiredAt = now() + 300s`.
  - Add TTL cleanup for commissions mirroring `ExchangeRatesManager`’s cleanup of `mExchangeRates`/`mExternalExchangeRates` (remove expired entries; compute nearest expiration to schedule cleanup).
  - TTL constant: `kCommissionsTTLSeconds = 300` in `TopologyTrustLinesManager`.
- OR-Tools integration impact:
  - Incorporate fixed per-hop commission when modeling capacities/effective flow:
    - Effective deliverable on an arc that passes through a node with commission `c` reduces by `c` once per traversal of that node.
    - Path-based LP: adjust path objective/constraints to account for the sum of commissions along the path (payer+receiver side and bridge segments as traversed).
  - Maintain feasibility by clamping when cumulative commissions exceed available path capacity.
- Config:
  - Use existing `conf.json` to define per-equivalent commissions for the local node. Exact schema to be finalized with Architect; examples included in PRD.
- Documentation:
  - Update PRD and topology protocol docs to reflect commission behavior, messaging, storage, and TTL.

- Unit tests:
  - Implement unit tests for `CommissionsManager`, covering the commission application method with multiple scenarios, including edge cases.

Definition of Done:
- PRD updated per policy with commissions feature and OR-Tools considerations.
- Protocol document updated: messages carry commission, sequence updated.
- Task created (this file) with implementation plan and test plan.
- `04-07-ortools-integration-and-lp.md` renamed to `04-08-ortools-integration-and-lp.md`, all references updated.

# Implementation Plan
1. Define `Commission` and `CommissionsManager` interfaces in docs/code locations; load from `conf.json` at startup.
2. Extend `ResultMaxFlowCalculationMessage` and `ResultMaxFlowCalculationGatewayMessage` to include optional `Commission` for their `equivalent`.
3. Update the four topology transactions to attach commission when present on node.
4. Extend `TopologyTrustLinesManager` with commissions cache map and TTL cleanup logic analogous to `ExchangeRatesManager` external rates cleanup.
5. On sender (S), store commissions only when `amount > 0` and apply TTL refresh semantics.
6. Update OR-Tools LP integration (path-based model) to subtract cumulative fixed commissions along each feasible path when computing objective and feasibility.
7. Update PRD and protocol documentation; provide config examples in PRD.
8. Provide a small demo plan (log-driven) to verify commission propagation and TTL cleanup.

# Test Plan
- Unit-level (design-level) scenarios:
  - `CommissionsManager::applyCommission` clamps to zero when `amount < fee`.
  - `TopologyTrustLinesManager` commissions map: insert, refresh, TTL expiry and cleanup.
  - Message serialization/deserialization includes optional commission when present.

  - `CommissionsManager` unit tests (explicit cases):
    - No commission for equivalent: input amount returns unchanged.
    - Fee = 0: input 0 → 0; input N → N.
    - Fee > 0: input 0 → 0; input < fee → 0; input = fee → 0; input > fee → input - fee.
    - Large values close to type limits (where applicable) still clamp correctly and do not underflow.
    - Idempotence for same input/fee (no internal state mutation in apply routine).
- Integration-level (mocked network):
  - Nodes with configured commission send it in result messages for the specific equivalent.
  - Payer (S) receives and caches commission only when `amount > 0` and refreshes TTL on update.
  - TTL cleanup removes expired entries without affecting active ones.
- OR-Tools modeling (logic verification):
  - For a simple path S→X→R with single commission `c` at X, verify optimal path result equals `min(capacities) - c` (clamped at zero) in target equivalent (ignoring exchange rate for the scenario).
  - Multi-hop path cumulative commissions reduce effective objective contribution accordingly.

# Verification and Validation
<Architect approval that validation criteria appropriate to task complexity have been met>

## Architecture integrity
- Single-source-of-truth for commissions in `conf.json` and `CommissionsManager`.
- Backward-compatible: absence of commission implies zero fee; messages optionalize the field.

## Security
- No new attack surface; validate inputs from config; ignore malformed/negative values.

## Performance
- Minimal overhead: small optional payload in topology results; TTL cleanup mirrors existing patterns.

## Scalability
- Scales with number of contractors and equivalents similarly to exchange rate caches.

## Reliability
- TTL cleanup ensures stale commissions do not skew optimization.

## Maintainability
- Mirrors existing ExchangeRatesManager cleanup patterns; consistent code paths in transactions and managers.

## Cost
- Negligible memory/time impact for caches and optional payloads.

## Compliance
- Follows project policy and templates; PRD and protocol updated accordingly.

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
