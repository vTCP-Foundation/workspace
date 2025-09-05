# 04-08 – OR-Tools integration and LP-based flow/paths computation

# Links
- [PRD](../../../prd/vtcpd/04-exchange-flow-calculation.md)

# Description
Integrate OR-Tools (LinearSolver), build an LP model per PRD with a path-based approach, compute maximum receivable amount in `mEquivalent`, and persist detailed path information.

# Requirements and DOD
- CMake: `find_package(ortools REQUIRED)`, link `ortools::ortools`.
- Node startup verification: check OR-Tools availability and minimum version (>= 9.0), terminate with error if unavailable or insufficient version.
- LP solver: create `MPSolver` (GLOP or equivalent), path variables, objective, constraints per PRD.
- Implement stages: path enumeration, LP build, optimization, results stored in `mMaxFlows` and `mOptimalPathResults`.
- Path logs in the new specified format: F(nodes: [addr1 -> addr2]; ids [id1 -> id2]; flow: amount; eq: equivalent) and E(node: addr; id: nodeId; eqs [fromEq->toEq]; flows:[inputFlow->outputFlow]).
- Runtime error handling for individual calculation failures.
- Unit tests for components (enumeration, model build, result interpretation) with mocks.
- Build and run tests in `build-tests`.

## Additional context and constraints from PRD (must-have for this task)
- Conditional compilation: define `WITH_ORTOOLS` when `ortools_FOUND`; gate LP code with this define.
- LP model (path-based) must include the following constraints and objective explicitly:
  - Capacity constraints: `flow[path_i] ≤ min_capacity_along_path_i`.
  - Exchange limits: for each exchange step j, `min_exchange ≤ exchange_amount[j, from, to] ≤ max_exchange`.
  - Balance constraints at nodes: input flows = output flows for each intermediate node (respecting exchanges).
  - Sender balance per equivalent: total outflow per payer equivalent ≤ available balance in that equivalent.
  - Non-negativity: all flow variables ≥ 0.
- Solver status handling: map statuses to actions
  - OPTIMAL → store `objective->Value()` in `mMaxFlows[contractorID]`, extract per-path flows into `mOptimalPathResults[contractorID]`.
  - INFEASIBLE → set zero result for contractor and log warning.
  - UNBOUNDED → throw runtime error (invalid model constraints).
  - ABNORMAL → throw runtime error (numerical issues).
  - Other → throw runtime error with status code.
- Results storage: primary map `mMaxFlows` (ContractorID → `TrustLineAmount`) and detailed `mOptimalPathResults` (ContractorID → `vector<OptimalPathResult>`).

## PRD prerequisites (dependencies to be present; implement here only if included in scope)
- Extended message protocol: all max-flow messages include `vector<SerializedEquivalent> exchangeEquivalents` (empty vector keeps legacy behavior).
  - Affected messages: `MaxFlowCalculationSourceFstLevelMessage`, `MaxFlowCalculationSourceSndLevelMessage`, `MaxFlowCalculationTargetFstLevelMessage`, `MaxFlowCalculationTargetSndLevelMessage`, base `MaxFlowCalculationMessage`.
- New command and transactions for exchanges:
  - `InitiateMaxFlowExchangeCalculationCommand` (validates that `exchangeEquivalents.size() ≤ 5`, else error 401 `responseProtocolError`).
  - `InitiateMaxFlowExchangeCalculationTransaction` (LP optimization entry point; `applyCustomLogic()` builds and solves LP; stores results in `mMaxFlows`/`mOptimalPathResults`).
  - `CollectTopologyForExchangeTransaction` and `ReceiveMaxFlowCalculationForExchangeOnTargetTransaction` (topology and rate-aware collection/forwarding).
- Exchange rates messaging and tails:
  - `ExchangeRatesMessage` carrying `vector<ExchangeRate>`.
  - `TailManager`: add `mExchangeRatesTail`; `IncomingRemoteNode::tryCollectNextPacket()` routes `ExchangeRatesMessage` there.
- External exchange rates storage (in `ExchangeRatesManager`):
  - `mExternalExchangeRates: map<pair<SerializedEquivalent, SerializedEquivalent>, vector<pair<ContractorID, ExchangeRate>>>` with TTL cleanup timer: remove expired entries, erase empty vectors, reschedule on insert.
  - Each stored external rate is coupled with sender `ContractorID` via `EquivalentsSubsystemsRouter::getOrCreateParticipantID`.
- Unified contractor ID management:
  - Move `mParticipantsAddresses` to `EquivalentsSubsystemsRouter` for consistent IDs across equivalents; all topology managers use unified IDs.
- Enhanced transaction behavior when `exchangeEquivalents` is non-empty:
  - Source-side (`MaxFlowCalculationSourceFstLevelTransaction`/`SndLevel`): seek local rates for pairs `mEquivalent / exchangeEquivalents[i]` and send via `ExchangeRatesMessage` with topology.
  - Target-side (`MaxFlowCalculationTargetFstLevelTransaction`/`SndLevel`): seek local rates for `exchangeEquivalents[i] / mEquivalent` and send via `ExchangeRatesMessage` with topology.
  - Optional enhancement: send additional topology per equivalent where exchange is possible (see PRD 4.834–4.844).

If any prerequisite above is not yet implemented in the repository, stub/mock them for unit tests of this task and add TODOs for their implementation in corresponding tasks of iteration 04.

# Implementation Plan
1. Integrate OR-Tools in CMake, add `WITH_ORTOOLS` define.
2. Add startup verification logic to check OR-Tools availability and version.
3. Implement path enumeration (length cap, pruning non-viable paths).
4. Build LP: variables per path, objective coefficients = effective rates.
5. Add full constraints per PRD: capacity per path, exchange min/max limits per step, node balance equalities, sender balance per payer equivalent, non-negativity.
6. Optimize, collect results, log paths in new format.
7. Handle solver statuses/errors and startup failures.

# Test Plan
- Unit tests on small synthetic graphs:
  - Single path, single exchange – validate result.
  - Multiple alternative paths – pick the best.
  - No paths – zero result.
  - Path logging format validation for F and E elements.
  - Startup verification tests (mock OR-Tools availability/version checks).
- Execute in `build-tests`.

Additional unit tests aligning with PRD (implement here if in scope; otherwise ensure covered in adjacent tasks):
- Regression: empty `exchangeEquivalents` preserves legacy single-equivalent behavior.
- Command validation: reject `exchangeEquivalents` with more than 5 items (401 `responseProtocolError`).
- Solver status handling branches (OPTIMAL/INFEASIBLE/UNBOUNDED/ABNORMAL) produce expected outcomes.
- Serialization/deserialization of `exchangeEquivalents` in max-flow messages.

## LP model details (reference)
- Solver creation: `MPSolver::CreateSolver("GLOP")`; throw runtime error if unavailable.
- Path enumeration yields `vector<ExchangePath>`; create one variable per path with upper bound = path min capacity.
- Objective: maximize Σ(flow_path_i × effective_exchange_rate_i).
- Constraints: see full list above; sender balance constraints are created per payer equivalent and include coefficients only for paths starting in that equivalent.
- Solution extraction: for each path variable `x_i`, if `x_i.solution_value() > ε`, compute received amount = `x_i * effective_rate_i` and persist in `mOptimalPathResults`.

## CMake integration pattern (reference)
```cmake
find_package(ortools REQUIRED)

target_link_libraries(vtcpd_core
    ortools::ortools
    # existing libraries...
)

if(ortools_FOUND)
    target_compile_definitions(vtcpd_core PRIVATE WITH_ORTOOLS)
endif()
```

# Verification and Validation
Architect validates model correctness and results against PRD baselines.

## Architecture integrity
LP computation isolated within the transaction; protocols unaffected.

## Security
No changes.

## Performance
Within NFR for typical network sizes.

## Scalability
Length/paths-count limits to control complexity.

## Reliability
Proper handling of INFEASIBLE/ABNORMAL statuses.

## Maintainability
Structured code, explicit interfaces, tests.

## Cost
No additional costs.

## Compliance
Complies with PRD and repository policy.

# Restrictions
- Commit changes only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
